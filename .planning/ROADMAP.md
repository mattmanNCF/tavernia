# Roadmap: Tavernia

## Overview

Tavernia ships as seven sequential phases, each delivering a verifiable capability before the next begins. Phase 1 lays the plugin contract and agentic framework that everything else builds on. Phases 2–5 complete the full invoice pipeline — capture → extract → review → export — the primary value proposition. Phase 6 ships the Recipe module as the first community-pattern plugin, proving the plugin contract works for domain extensions beyond invoicing. Phase 7 hardens production operations: update tooling, backup automation, cross-platform deployment docs, and the KB continuity guarantee that makes the agentic framework's self-improvement loop trustworthy.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Core Platform + Agentic Foundation** - Plugin registry, event bus, auth/RBAC, shared schema, AgentProvider abstraction, wiki KB, Docker skeleton
- [ ] **Phase 2: Invoice Ingestion + Document Store** - Photo upload, IMAP polling, raw document storage, file-type routing
- [ ] **Phase 3: OCR Pipeline + Extraction Engine** - Async Celery workers, pdfplumber + surya-ocr routing, catch-weight schema, vendor fuzzy match, GL suggestion
- [ ] **Phase 4: Review UI — Three-Role Views** - Chef/receiver confirmation, GM approval, accountant GL mapping, review queue, duplicate detection
- [ ] **Phase 5: Accounting Export** - QBO Bill push, atomic OAuth refresh, CSV/IIF export, price history, rate-limit handling
- [ ] **Phase 6: Recipe & Food Costing Module** - Ingredient database, recipe builder, theoretical vs actual food cost, price-change recalculation
- [ ] **Phase 7: Production Hardening + KB Self-Improvement Loop** - Update/backup tooling, backup-cron, cross-platform docs, KB continuity test, audit log verification

## Phase Details

### Phase 1: Core Platform + Agentic Foundation
**Goal**: Any developer can boot Tavernia locally, install a "hello world" plugin, fire an event, and verify the agentic framework is wired end-to-end
**Depends on**: Nothing (first phase)
**Requirements**: CORE-01, CORE-02, CORE-03, CORE-04, CORE-05, CORE-06, CORE-07, AGENT-01, AGENT-02, AGENT-03, AGENT-04, AGENT-05, AGENT-06
**Success Criteria** (what must be TRUE):
  1. Operator runs `docker compose up` and all services start healthy (database, API, worker, frontend)
  2. A "hello world" plugin installs from `plugins/` directory, registers in the `plugin_registry` table, and emits a versioned event (`domain.action.v1`) that the event bus delivers
  3. A user can register, log in with email + password, receive a JWT, and have their role (`chef`, `gm`, or `accountant`) enforce access to role-appropriate routes
  4. An agent call via `AgentProvider` loads the domain `context.md`, reads the relevant wiki KB section, and writes a structured outcome entry to `wiki/{domain}/` after completion — all without hardcoding any provider SDK
  5. A second user sharing a role receives a distinct `wiki/users/{username}.md` profile, and agents address that user's preferences separately from the first
**Plans**: TBD

### Phase 2: Invoice Ingestion + Document Store
**Goal**: An invoice — photographed on a phone or emailed as a PDF — lands in Tavernia's document store and triggers a processing event, with no manual file transfer required
**Depends on**: Phase 1
**Requirements**: INGEST-01, INGEST-02, INGEST-03, INGEST-04
**Success Criteria** (what must be TRUE):
  1. Chef photographs a paper invoice on a mobile browser (no native app), uploads it, and sees a confirmation that the document is queued — without the page reloading or requiring a desktop browser
  2. A vendor PDF emailed to the configured inbox is automatically ingested by Celery Beat within the polling interval, no manual action required
  3. Each ingested document appears as a row in `raw_documents` with source, file path on the named Docker volume, MIME type, and ingestion timestamp
  4. The system routes the document to the correct processing path (native PDF vs. OCR) based on python-magic file type detection, not file extension
**Plans**: TBD

### Phase 3: OCR Pipeline + Extraction Engine
**Goal**: An ingested document becomes a structured, validated invoice with line items, confidence scores, vendor identity, and GL code suggestions — all computed locally without cloud API calls
**Depends on**: Phase 2
**Requirements**: OCR-01, OCR-02, OCR-03, OCR-04, OCR-05, OCR-06, OCR-07, OCR-08
**Success Criteria** (what must be TRUE):
  1. OCR processing runs in a Celery worker; uploading an invoice returns immediately and the HTTP thread is never blocked waiting for extraction
  2. A machine-generated vendor PDF is processed by pdfplumber; a photographed invoice is processed by surya-ocr — routing is automatic, and neither path makes outbound API calls
  3. Every extracted line item carries `pricing_type` (`unit`, `catch_weight`, or `case_price`), plus separate `invoiced_qty` and `received_qty` columns initialized equal
  4. Any line item where `unit_price × invoiced_qty ≠ extended_total` (outside ±2%) is automatically flagged; any field below the confidence threshold is flagged for human review
  5. An unrecognized vendor surfaces a "new vendor" prompt; a recognized vendor is matched via fuzzy lookup; GL code suggestions appear based on historical `(vendor_id, item_description)` patterns
**Plans**: TBD

### Phase 4: Review UI — Three-Role Views
**Goal**: An invoice moves from "extracted" to "approved" through a structured three-role handoff — chef confirms receipt, GM approves cost, accountant maps GL codes — and no invoice can reach export without passing every gate
**Depends on**: Phase 3
**Requirements**: REVIEW-01, REVIEW-02, REVIEW-03, REVIEW-04, REVIEW-05, REVIEW-06, REVIEW-07
**Note**: REVIEW-03 (GM price deviation alerts) has a forward dependency on EXPORT-09/10 (Phase 5 price history). Phase 4 implementation must render the price deviation panel in an empty/graceful state when no price history exists yet; Phase 5 must backfill alerts for already-approved invoices when price history is first populated.
**Success Criteria** (what must be TRUE):
  1. Chef sees the original invoice image alongside extracted fields, records received quantities per line item, and cannot skip confirmation — the invoice does not advance without this step
  2. GM sees total invoice cost and food cost % impact, receives notification when chef completes confirmation, and can approve or reject; price deviation alerts surface on the GM view (empty state shown when no price history exists)
  3. Accountant sees GL account mapping per line item and the AP queue, receives queue update when GM approves, and confirms GL codes before export
  4. The review queue surfaces all low-confidence extractions and flagged items; no invoice is exportable until it clears the queue
  5. Duplicate invoices (matching vendor + invoice number + total + date hash) are flagged automatically before reaching the review queue
**Plans**: TBD

### Phase 5: Accounting Export
**Goal**: An approved invoice is pushed to QuickBooks Online as a correctly structured Bill (not a JournalEntry), and the integration survives OAuth token rotation, rate limits, and connection expiry without data loss
**Depends on**: Phase 4
**Requirements**: EXPORT-01, EXPORT-02, EXPORT-03, EXPORT-04, EXPORT-05, EXPORT-06, EXPORT-07, EXPORT-08, EXPORT-09, EXPORT-10
**Success Criteria** (what must be TRUE):
  1. Accountant pushes an approved invoice to QBO; it appears in Pay Bills queue as a `Bill` with correct `VendorRef` and line items — not as a JournalEntry
  2. A simulated crash during OAuth token refresh does not permanently disconnect QBO; the next refresh attempt succeeds using the persisted token; `invalid_grant` triggers a clear "Reconnect to QuickBooks" prompt visible to GM
  3. QBO connection health (green/yellow/red) is visible on the GM dashboard at all times
  4. Bulk historical import does not exceed QBO rate limits; 429 responses trigger exponential backoff with jitter; the queue drains without manual intervention
  5. Accountant can export approved invoices as CSV with configurable column mapping and as IIF for QuickBooks Desktop; every export is recorded in `export_records`; price deviation alerts fire when a vendor charges above the configured threshold
**Plans**: TBD

### Phase 6: Recipe & Food Costing Module
**Goal**: A chef can build recipes from invoice-sourced ingredients and a GM can see theoretical vs actual food cost — all delivered as a plugin that obeys the Phase 1 plugin contract without touching core tables
**Depends on**: Phase 5
**Requirements**: RECIPE-01, RECIPE-02, RECIPE-03, RECIPE-04, RECIPE-05
**Success Criteria** (what must be TRUE):
  1. Chef creates an ingredient linked to a vendor item from a processed invoice; ingredient cost updates automatically when a new invoice for that vendor is processed
  2. Chef builds a recipe by selecting ingredients and quantities; recipe cost is calculated automatically from current invoice prices and displayed in the UI
  3. GM views actual vs theoretical food cost % per category and per period, updated as invoices are processed
  4. When a vendor's price changes (detected via new invoice), affected recipe costs recalculate and GM receives an alert
  5. Recipe module's tables are all prefixed `recipe_`; it communicates with the invoice module only via the event bus; the plugin can be installed or removed without touching core or invoice tables
**Plans**: TBD

### Phase 7: Production Hardening + KB Self-Improvement Loop
**Goal**: An operator running Tavernia in production can update safely, recover from backup, and trust that swapping AI providers does not reset agent institutional knowledge
**Depends on**: Phase 6
**Requirements**: CORE-08, CORE-09, CORE-10, CORE-11, CORE-12, AGENT-07, AGENT-08
**Success Criteria** (what must be TRUE):
  1. Operator runs `./tavernia update`; latest versioned Docker images are pulled and services restart without data loss or migration failure
  2. Operator runs `./tavernia backup`; a timestamped pg_dump is produced; backup-cron container runs automatically daily; Admin UI shows "Last backup: [date]" with a warning if backup is >7 days old
  3. Installation is tested and documented for Linux VPS (DigitalOcean/Hetzner), Windows Docker Desktop, and Mac Docker Desktop — all three paths produce a working system
  4. After swapping AI provider via `.env` config, agent behavior (GL code suggestions, vendor matching, extraction confidence) is equivalent to pre-swap behavior when bootstrapped from the KB — no manual reorientation required
  5. GL code suggestion accuracy demonstrably improves after 20+ approved invoices compared to a fresh install; vendor-specific extraction notes accumulate in `wiki/invoicing/vendor-profiles.md`
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Platform + Agentic Foundation | 0/TBD | Not started | - |
| 2. Invoice Ingestion + Document Store | 0/TBD | Not started | - |
| 3. OCR Pipeline + Extraction Engine | 0/TBD | Not started | - |
| 4. Review UI — Three-Role Views | 0/TBD | Not started | - |
| 5. Accounting Export | 0/TBD | Not started | - |
| 6. Recipe & Food Costing Module | 0/TBD | Not started | - |
| 7. Production Hardening + KB Self-Improvement Loop | 0/TBD | Not started | - |
