# Requirements: Tavernia

**Defined:** 2026-04-17
**Core Value:** A hospitality operator's data stays on their own infrastructure, visible only to them, and the tools that process it are open-source, auditable, and natively agentic.

---

## v1 Requirements

### Core Platform

- [ ] **CORE-01**: Operator can install Tavernia with a single `docker compose up` command
- [ ] **CORE-02**: Plugin registry discovers and loads modules from the `plugins/` directory via pluggy hookspecs
- [ ] **CORE-03**: Event bus emits and receives versioned events (`domain.action.v1` naming convention); no plugin may subscribe to another plugin's internal events directly
- [ ] **CORE-04**: Plugin schema isolation enforced: plugins own their own tables (`{plugin_name}_*`); no plugin may ALTER core tables
- [ ] **CORE-05**: Auth/RBAC: user can log in with email + password; sessions persist via JWT; `user_roles` join table with `(user_id, role_id, location_id NULLABLE)` — no boolean role flags
- [ ] **CORE-06**: Three built-in roles: `chef`, `gm`, `accountant`; each restricts access to role-appropriate views and actions
- [ ] **CORE-07**: Shared schema primitives exist in core: `vendors`, `gl_accounts`, `users`, `roles`, `user_roles`, `audit_log`, `plugin_registry`
- [ ] **CORE-08**: Operator can run `./tavernia update` to pull latest versioned Docker images and restart data-safely
- [ ] **CORE-09**: Operator can run `./tavernia backup` to produce a timestamped pg_dump; backup-cron container runs daily automatic backups with 7-day retention
- [ ] **CORE-10**: Admin UI shows "Last backup: [date]" with a warning banner if backup is >7 days old
- [ ] **CORE-11**: Installation tested and documented for Linux VPS (DigitalOcean/Hetzner), Windows Docker Desktop, and Mac Docker Desktop
- [ ] **CORE-12**: Managed single-tenant hosting path available (same Docker stack deployed to cloud VM per operator)

### Agentic Framework

- [ ] **AGENT-01**: `AgentProvider` abstraction in core — single interface wrapping AI calls; provider (Anthropic/OpenAI/Gemini/Ollama) and model configurable via `.env`; no module imports a provider SDK directly
- [ ] **AGENT-02**: Domain context file convention: each module ships a `modules/{name}/context.md` describing the domain, its data model, workflow, and boundaries; `AgentProvider` loads the relevant context file on initialization for any agent task in that domain
- [ ] **AGENT-03**: Wiki KB at `wiki/` — structured markdown hierarchy (`wiki/{domain}/{topic}.md`); agents read the relevant KB section as part of their context window on each invocation
- [ ] **AGENT-04**: Write-after-task protocol: each agent task completion triggers a structured outcome write to the relevant `wiki/{domain}/` section (what was done, what was learned, confidence, edge cases encountered)
- [ ] **AGENT-05**: Machine-bound USER.md profiles: each named user gets a `wiki/users/{username}.md` profile on their installation; profile is updated by agents as the user interacts with the system (preferences, workflow patterns, recurring corrections)
- [ ] **AGENT-06**: Multiple users per role supported: exec chef and sous chef both have `chef` role but each has a distinct USER.md profile; profiles are installation-specific (not synced across locations by default)
- [ ] **AGENT-07**: KB continuity test: after swapping AI provider via config, agent behavior (GL code suggestions, vendor matching, extraction confidence) is equivalent to pre-swap behavior when bootstrapped from the KB alone — no manual reorientation required
- [ ] **AGENT-08**: Self-improvement loop: GL code suggestion accuracy improves over time as agents accumulate correction history in the KB; vendor-specific extraction notes accumulate in `wiki/invoicing/vendor-profiles.md`

### Invoice Ingestion

- [ ] **INGEST-01**: Chef can photograph a paper invoice on a mobile device and upload it via the web UI without installing a native app (mobile-optimized web, PWA-capable)
- [ ] **INGEST-02**: System polls a configured IMAP inbox via Celery Beat; vendor PDF email attachments are auto-ingested without manual intervention
- [ ] **INGEST-03**: Each ingested document is stored as a raw file on a named Docker volume with a corresponding `raw_documents` database record (source, file_path, mime_type, ingestion_timestamp)
- [ ] **INGEST-04**: System routes each document to the correct processing path (native PDF extraction vs. OCR) based on file type detection (python-magic), not file extension

### OCR & Extraction

- [ ] **OCR-01**: OCR processing runs asynchronously in a Celery worker; no OCR work blocks the HTTP request thread
- [ ] **OCR-02**: Machine-generated vendor PDFs are processed with pdfplumber (no OCR engine invoked); scanned/photo invoices are processed with surya-ocr — all local, no cloud API calls
- [ ] **OCR-03**: `invoice_line_items` schema includes `pricing_type` field (`unit` / `catch_weight` / `case_price`) from day one; catch-weight items are modeled correctly before any extraction code is written
- [ ] **OCR-04**: `invoice_line_items` stores `invoiced_qty` and `received_qty` as separate columns; they start equal and diverge when chef confirms a short shipment
- [ ] **OCR-05**: Each extracted field carries a per-field confidence score; fields below threshold are flagged for human review
- [ ] **OCR-06**: Post-extraction validation: any line item where `unit_price × invoiced_qty ≠ extended_total` (outside ±2% tolerance) is automatically flagged for human review
- [ ] **OCR-07**: Vendor auto-recognition via fuzzy match against the `vendors` table; unrecognized vendors surface as a "new vendor" prompt
- [ ] **OCR-08**: GL code suggestion based on historical `(vendor_id, item_description)` → `gl_account_id` patterns from prior approved invoices

### Invoice Review & Approval

- [ ] **REVIEW-01**: Chef/receiver view displays the original invoice image alongside extracted fields; chef must confirm receipt before invoice proceeds (required step, not skippable)
- [ ] **REVIEW-02**: Chef records received quantities per line item; discrepancies between `received_qty` and `invoiced_qty` are flagged and persist to the record
- [ ] **REVIEW-03**: GM view shows total invoice cost, food cost % impact, and any price deviation alerts; GM can approve or reject
- [ ] **REVIEW-04**: Accountant view shows GL account mapping per line item, AP queue, and QBO sync status; accountant assigns/confirms GL codes before export
- [ ] **REVIEW-05**: Invoice review queue surfaces all low-confidence extractions and flagged items; no invoice can be exported without passing through the review queue
- [ ] **REVIEW-06**: Duplicate invoice detection: system flags invoices matching an existing record on (vendor_id + invoice_number + total_amount + invoice_date) hash
- [ ] **REVIEW-07**: Role handoff notifications: when chef completes confirmation, GM is notified; when GM approves, accountant queue is updated

### Accounting Export

- [ ] **EXPORT-01**: Accountant can push an approved invoice to QBO as a `Bill` (not `JournalEntry`) with correct `VendorRef` and line items via the QBO v3 API
- [ ] **EXPORT-02**: QBO OAuth2 flow implemented with atomic refresh token persistence; a crash during token refresh does not permanently disconnect the integration
- [ ] **EXPORT-03**: `invalid_grant` recovery flow: when QBO connection expires or is revoked, GM sees a clear "Reconnect to QuickBooks" prompt
- [ ] **EXPORT-04**: QBO connection health indicator (green/yellow/red) visible to GM role at all times
- [ ] **EXPORT-05**: QBO rate limit handling: requests are queued and retried with exponential backoff + jitter on 429 responses; bulk historical import does not exceed 500 req/min
- [ ] **EXPORT-06**: Accountant can export approved invoices as a CSV file with configurable column mapping
- [ ] **EXPORT-07**: Accountant can export approved invoices as an IIF file for QuickBooks Desktop operators
- [ ] **EXPORT-08**: `export_records` table tracks each export (target, external_id if QBO, status, timestamp)
- [ ] **EXPORT-09**: Price history is maintained per (vendor_id, item_description); price deviation alert is shown when a vendor charges above the rolling average
- [ ] **EXPORT-10**: Price deviation threshold is configurable per vendor (default: flag if >5% above 90-day average)

### Recipe & Food Costing Module

- [ ] **RECIPE-01**: User can create and manage an ingredient/item database linked to vendor items from processed invoices
- [ ] **RECIPE-02**: Chef can create recipes by selecting ingredients and specifying quantities; recipe cost is calculated automatically from current invoice prices
- [ ] **RECIPE-03**: GM can view actual vs theoretical food cost % per category and per period, updated as invoices are processed
- [ ] **RECIPE-04**: When a vendor's price changes (detected via new invoice), affected recipe costs are recalculated and the GM is alerted
- [ ] **RECIPE-05**: Recipe module ships as a plugin following the core plugin contract; it owns its own tables and communicates with the invoice module via the event bus

---

## v2 Requirements

### Point of Sale (Toast replacement)

- **POS-01**: Table management and order taking
- **POS-02**: Ticket printing and kitchen display integration
- **POS-03**: Payment processing integration
- **POS-04**: Sales data feeds into food cost % (closes the actual vs theoretical loop)

### Tip Reconciliation (Kickfin replacement)

- **TIP-01**: Tip pool calculation and distribution by role/hour
- **TIP-02**: Tip payout records and compliance reporting
- **TIP-03**: Integration with payroll export

### Accounting / GL (QuickBooks replacement)

- **ACCT-01**: Chart of accounts management
- **ACCT-02**: Journal entries and P&L reporting
- **ACCT-03**: Replaces QBO integration (native GL module)

### Advanced AI Features

- **AI-01**: Vision-LLM extraction (olmOCR) for operators with dedicated GPU hardware
- **AI-02**: Cross-location KB synchronization (multi-unit operators share vendor profiles)
- **AI-03**: Natural language query interface against the wiki KB

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| Multi-tenant SaaS | Defeats the security premise; single-tenant is the core value |
| Native mobile apps | PWA covers dock-side capture; native apps deferred to v2 |
| Vision-LLM extraction (GPU) | Requires 9B+ GPU models; not viable for self-hosted v1 hardware |
| Cloud OCR (Google Vision / AWS Textract) | Breaks local-first data premise; operator can configure if they choose but not shipped by default |
| EDI integrations | Enterprise-only; email PDF covers independent restaurant vendor distribution |
| Menu engineering / profitability analytics | Requires POS sales data; deferred until POS module ships |
| Bill payment execution (paying vendors) | PCI compliance scope; out of scope for AP recording tool |

---

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CORE-01 | Phase 1 | Pending |
| CORE-02 | Phase 1 | Pending |
| CORE-03 | Phase 1 | Pending |
| CORE-04 | Phase 1 | Pending |
| CORE-05 | Phase 1 | Pending |
| CORE-06 | Phase 1 | Pending |
| CORE-07 | Phase 1 | Pending |
| CORE-08 | Phase 7 | Pending |
| CORE-09 | Phase 7 | Pending |
| CORE-10 | Phase 7 | Pending |
| CORE-11 | Phase 7 | Pending |
| CORE-12 | Phase 7 | Pending |
| AGENT-01 | Phase 1 | Pending |
| AGENT-02 | Phase 1 | Pending |
| AGENT-03 | Phase 1 | Pending |
| AGENT-04 | Phase 1 | Pending |
| AGENT-05 | Phase 1 | Pending |
| AGENT-06 | Phase 1 | Pending |
| AGENT-07 | Phase 7 | Pending |
| AGENT-08 | Phase 7 | Pending |
| INGEST-01 | Phase 2 | Pending |
| INGEST-02 | Phase 2 | Pending |
| INGEST-03 | Phase 2 | Pending |
| INGEST-04 | Phase 2 | Pending |
| OCR-01 | Phase 3 | Pending |
| OCR-02 | Phase 3 | Pending |
| OCR-03 | Phase 3 | Pending |
| OCR-04 | Phase 3 | Pending |
| OCR-05 | Phase 3 | Pending |
| OCR-06 | Phase 3 | Pending |
| OCR-07 | Phase 3 | Pending |
| OCR-08 | Phase 3 | Pending |
| REVIEW-01 | Phase 4 | Pending |
| REVIEW-02 | Phase 4 | Pending |
| REVIEW-03 | Phase 4 | Pending |
| REVIEW-04 | Phase 4 | Pending |
| REVIEW-05 | Phase 4 | Pending |
| REVIEW-06 | Phase 4 | Pending |
| REVIEW-07 | Phase 4 | Pending |
| EXPORT-01 | Phase 5 | Pending |
| EXPORT-02 | Phase 5 | Pending |
| EXPORT-03 | Phase 5 | Pending |
| EXPORT-04 | Phase 5 | Pending |
| EXPORT-05 | Phase 5 | Pending |
| EXPORT-06 | Phase 5 | Pending |
| EXPORT-07 | Phase 5 | Pending |
| EXPORT-08 | Phase 5 | Pending |
| EXPORT-09 | Phase 5 | Pending |
| EXPORT-10 | Phase 5 | Pending |
| RECIPE-01 | Phase 6 | Pending |
| RECIPE-02 | Phase 6 | Pending |
| RECIPE-03 | Phase 6 | Pending |
| RECIPE-04 | Phase 6 | Pending |
| RECIPE-05 | Phase 6 | Pending |

**Coverage:**
- v1 requirements: 54 total
- Mapped to phases: 54
- Unmapped: 0 ✓

**Phase distribution:**
- Phase 1 (Core Platform + Agentic Foundation): CORE-01–07, AGENT-01–06 (13 requirements)
- Phase 2 (Invoice Ingestion + Document Store): INGEST-01–04 (4 requirements)
- Phase 3 (OCR Pipeline + Extraction Engine): OCR-01–08 (8 requirements)
- Phase 4 (Review UI — Three-Role Views): REVIEW-01–07 (7 requirements)
- Phase 5 (Accounting Export): EXPORT-01–10 (10 requirements)
- Phase 6 (Recipe & Food Costing Module): RECIPE-01–05 (5 requirements)
- Phase 7 (Production Hardening + KB Self-Improvement Loop): CORE-08–12, AGENT-07–08 (7 requirements)

---

*Requirements defined: 2026-04-17*
*Last updated: 2026-04-17 — traceability updated to 7-phase roadmap; CORE-08–12 and AGENT-07–08 deferred to Phase 7 (Production Hardening); requirement count corrected to 54*
