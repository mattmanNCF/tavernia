# Project Research Summary

**Project:** Tavernia — Modular Open-Source Hospitality OS
**Domain:** Hospitality back-office / AP automation / agentic platform
**Researched:** 2026-04-17
**Confidence:** HIGH

## Executive Summary

Tavernia is a modular, self-hosted hospitality operating system built on two interlocking premises: single-tenant data sovereignty (no shared cloud surface, operator-owned infrastructure) and a natively agentic architecture (domain-scoped context files, model-agnostic agent abstraction, persistent wiki KB). The first shipped module is invoice processing — the highest-pain workflow for independent restaurants, replacing tools like MarginEdge (~$350/month/location SaaS). The recommended technical approach is a Python/FastAPI backend with pluggy-based plugin system, surya-ocr for document extraction, Celery for async OCR jobs, and a React/TypeScript/shadcn frontend — all containerized in Docker Compose for single-command operator deployment.

The agentic framework is a first-class architectural constraint, not a feature. Each installed module carries domain-scoped context files (analogous to CLAUDE.md) that orient any AI agent to its domain. The platform layer provides a model-agnostic agent abstraction so operators can swap Anthropic/OpenAI/Gemini/Ollama via config without behavior regression. A portable markdown wiki KB accumulates agent task outcomes and provides continuity guarantees across model switches. This architecture must be designed into the core platform in Phase 1 — both the plugin system and the agentic layer share the same "no hardcoded coupling" principle and reinforce each other.

The top risks are: (1) QBO integration correctness — using `Bill` endpoint, not `JournalEntry`, and building atomic OAuth token refresh from day one; (2) invoice OCR accuracy on catch-weight and handwritten-annotation invoices, which require a data model designed for catch-weight and a required (not optional) human confirmation step; (3) plugin schema coupling, where the first-party invoice module must itself obey the "no mutations to core tables" contract or every future community plugin will violate it; (4) missing an update mechanism and backup tooling at launch, which rot the system's security value proposition immediately. All four are Phase 1 design constraints, not implementation details to defer.

---

## Key Findings

### Recommended Stack

Python/FastAPI is the clear backend choice — the OCR (surya-ocr, pdfplumber), QBO (python-quickbooks), and async processing (Celery) ecosystems are all Python-primary or Python-only. SQLAlchemy 2.x async + PostgreSQL (one DB per install, never multi-tenant) provides the data layer. The plugin system is built on pluggy (status: Mature; used by pytest/tox). The frontend is React 19 + Vite + TypeScript + shadcn/ui.

| Component | Choice | Version | Confidence |
|-----------|--------|---------|------------|
| Backend | FastAPI | 0.136.0 | HIGH |
| ORM | SQLAlchemy async + asyncpg | 2.x | HIGH |
| Database | PostgreSQL | 16 | HIGH |
| Migrations | Alembic | 1.18.4 | HIGH |
| OCR (scanned) | surya-ocr | 0.17.1 | MEDIUM-HIGH |
| OCR (native PDF) | pdfplumber | 0.11.9 | HIGH |
| Email ingestion | imapclient | 3.0.3+ | HIGH |
| Async workers | Celery + Redis | 5.6.3 / 7.4.0 | HIGH |
| Plugin system | pluggy | 1.6.0 | HIGH |
| QBO integration | python-quickbooks + intuitlib | 0.9.12 | MEDIUM |
| Auth | PyJWT + passlib[bcrypt] | 2.12.1 | HIGH |
| Frontend | React 19 + Vite + TypeScript + shadcn/ui + Tailwind 4 | latest | HIGH |

**Notable decisions:**
- python-jose is abandoned/insecure — PyJWT is the correct replacement
- IIF (QuickBooks Desktop) export deferred — separate product from QBO, different operator profile
- Vision-LLM extraction (olmOCR) deferred — requires 9B+ GPU, not viable for v1 self-hosted
- Docker Compose v2 + nginx for TLS termination + named volumes for file persistence

### Expected Features

**Table stakes (must have — complete invoice pipeline):**
- Photo invoice capture (mobile web, dock-side — no native app required)
- Email-to-invoice ingestion via IMAP (vendor PDF auto-ingest by Celery Beat)
- Line-item OCR extraction with per-field confidence scoring
- Vendor auto-recognition via fuzzy match
- GL code assignment (rule-based, user-trainable per vendor + item)
- Required human review gate (not optional) before any export
- Multi-role approval workflow: chef confirms received goods → GM approves cost → accountant exports
- QBO Bill API export (Bills, not JournalEntries — critical distinction)
- CSV export fallback for non-QBO operators
- Vendor management, price history, duplicate detection, audit trail

**Differentiators:**
- Local-first data sovereignty + raw data portability (operator owns the DB directly)
- Price deviation alerts (flag when vendor charges above historical price)
- Role-specific UX: chef/receiver view, GM cost dashboard, accountant AP queue
- Offline-capable capture (PWA + service worker) — dock connectivity is unreliable
- Natively agentic framework: context files + model-agnostic abstraction + wiki KB

**Defer to v2+:**
- Cross-vendor price comparison, recipe costing, inventory counting, demand forecasting
- Native mobile apps (PWA covers v1)
- EDI integrations, menu engineering, IIF export

**Critical adoption insight:** QBO export accuracy (Bills with correct GL codes + complete line items) is the single thing that determines whether accountants accept or reject the tool at first use.

### Architecture Approach

Tavernia is a modular monolith with pluggy-based plugin boundaries. Core Platform owns: auth/RBAC, plugin registry, event bus, shared schema, AgentProvider abstraction, context file loader, wiki KB. Everything else is a plugin. The first-party invoice module is itself a plugin and must obey the plugin contract.

**Shared schema primitives (core owns):** `vendors`, `gl_accounts`, `users`, `roles`, `user_roles`, `audit_log`, `plugin_registry`

**Invoice module schema (plugin owns):** `raw_documents`, `ocr_results`, `pending_invoices`, `invoice_line_items`, `export_records`

**Agentic framework (co-located in core):**
- Module context files at `modules/{name}/context.md` — loaded on agent initialization for that domain
- `AgentProvider` abstraction — single interface wrapping Anthropic/OpenAI/Gemini/Ollama; swap via config
- Wiki KB at `wiki/` — structured markdown; agents write outcome entries after each task completion
- Context file loader reads relevant `context.md` + wiki KB section into agent context window

**Data flow:** Capture → Extract → Code → Approve → Export (forward-only; corrections are deltas, not re-runs)

**Plugin contract (enforced from day 1):**
- Plugins own their own tables (prefixed `{plugin_name}_`)
- Plugins may never ALTER core tables
- Plugin-to-plugin communication only via event bus or shared service layer
- All events are versioned: `domain.action.v1`

### Critical Pitfalls

1. **Catch-weight items silently corrupt food cost %** — Produce/meat invoices price by actual delivery weight, breaking `unit_price × qty = line_total`. Model `pricing_type` field (`unit`/`catch_weight`/`case_price`) in schema before writing any extraction code. Post-extraction validation: flag lines where formula fails within ±2%.

2. **QBO: JournalEntry instead of Bill breaks AP entirely** — Bills appear in Pay Bills queue, age properly, and clear on payment. JournalEntries do not. Test the full cycle (create Bill → pay in QBO → verify cleared) before declaring integration complete.

3. **QBO OAuth refresh tokens are rolling single-use** — Every refresh returns a new token and invalidates the old. Store in transactional DB row; refresh must be atomic. As of November 2025, tokens have a 5-year hard maximum with required reauthorization. Build a "QBO connection health" indicator visible to GM role; build `invalid_grant` recovery flow.

4. **Plugin schema coupling defeats the installable-module architecture** — Once one plugin writes to a core table, the contract is broken and all future plugins will follow. The first-party invoice module must serve as the positive example.

5. **No update mechanism = security value proposition destroyed at first install** — Ship `./tavernia update` with v1. An operator running unpatched 6-month-old code is more exposed than a SaaS customer.

6. **Boolean role flags can't handle real restaurant org charts** — Build `user_roles` as join table `(user_id, role_id, location_id NULLABLE)` from day one. Multi-unit operators hit boolean flags at first deployment.

---

## Implications for Roadmap

### Suggested Phase Structure (6 phases, fine granularity)

**Phase 1: Core Platform + Agentic Foundation**
Plugin registry (pluggy hookspecs + filesystem discovery), event bus with versioned schemas, auth/RBAC (PyJWT + `user_roles` join table), shared schema primitives, Docker Compose skeleton (all health-checked), update + backup tooling, AgentProvider abstraction, context file loader convention, wiki KB schema, "hello world" plugin validating the full cycle.

**Phase 2: Invoice Ingestion + Document Store**
Photo upload endpoint, IMAP inbox polling via Celery Beat, PDF attachment extraction, `raw_documents` table, `document.received.v1` event, python-magic file type routing (native PDF vs. scanned).

**Phase 3: OCR Pipeline + Extraction Engine**
Celery async OCR worker (pdfplumber → surya-ocr routing), `ocr_results` and `pending_invoices` tables with `pricing_type` + catch-weight schema, vendor fuzzy matching, GL suggestion by history, per-field confidence scoring, post-extraction validation, `invoice.extracted.v1` event, agent writes outcomes to wiki KB.

**Phase 4: Review UI (Three-Role Views)**
Chef/receiver view (required confirmation step, received qty input), GM view (cost %, approvals), accountant view (GL mapping, AP queue, QBO sync status), invoice review queue, duplicate detection, role handoff notifications.

**Phase 5: Export Service (QBO + CSV)**
CSV export first (validates pipeline), then QBO Bill push (VendorRef matching, BillLineItems), atomic OAuth token refresh (intuitlib), `invalid_grant` recovery, QBO connection health indicator, rate limit handling (429 backoff), `export_records` table, price history tracking, full pay-cycle integration test.

**Phase 6: Production Hardening + Agentic Self-Improvement Loop**
`./tavernia update` + `./tavernia backup` commands, backup-cron container, admin backup status indicator, operator docs (Linux VPS / Windows Docker Desktop / Mac), hardened KB write-after-task protocol, KB continuity test (swap provider → verify equivalent behavior), audit log verification.

### Phase Ordering Rationale

- Phase 1 before everything: plugin contract + agentic framework conventions are foundational; retrofitting after first module exists requires rewriting all integration points
- Phase 2 before 3: stable document store and event contract before OCR; validates plugin-to-core interface cheaply
- Phase 3 data model before Phase 4 views: catch-weight schema must exist before any review UI screen is designed
- Phase 5 after Phase 4: export requires approved invoices; QBO OAuth needs stable redirect URI
- Phase 6 last: hardening requires stable schema and working application

### Research Flags (for phase planning)

| Phase | Flag | Priority |
|-------|------|----------|
| 1 | Agentic framework design spike (context file format, wiki KB schema, AgentProvider interface) — no open-source reference | CRITICAL |
| 3 | Collect 20+ real invoice samples before committing to extraction approach | HIGH |
| 5 | QBO sandbox setup; validate python-quickbooks Bill entity field coverage | HIGH |
| 2 | IMAP OAuth2 SASL (Gmail/Outlook) app registration complexity | MEDIUM |
| 4 | Informal user interviews (one chef, one accountant) before finalizing UI copy | MEDIUM |
| 6 | Test Windows Docker Desktop path before declaring Windows support | MEDIUM |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Versions verified on PyPI; python-quickbooks MEDIUM (alpha status) |
| Features | HIGH | Derived from MarginEdge, xtraCHEF, Restaurant365 feature sets |
| Architecture | MEDIUM-HIGH | Plugin patterns from Backstage/VSCode; agentic framework is novel (no open-source reference) |
| Pitfalls | HIGH | QBO specifics verified against official Intuit docs including Nov 2025 token policy change |

**Overall confidence:** HIGH

---

## Open Questions

1. **Agentic framework design**: Context file schema, wiki KB indexing strategy, AgentProvider interface surface area, write-after-task protocol enforcement — design spike required in Phase 1.
2. **python-quickbooks alpha status**: Validate Bill entity field coverage against QBO v3 API reference before Phase 5.
3. **Real invoice sample diversity**: Do not commit extraction approach before collecting 20+ samples across Sysco, local farm, dot-matrix output, Word template formats.
4. **Windows Docker Desktop path**: Test before Phase 6 documentation is written.
5. **IIF (QuickBooks Desktop)**: Deferred and low confidence — do not conflate with CSV fallback; assess operator demand before scheduling.
6. **QBO OAuth connected app approval**: Intuit requires app review for production access — investigate timeline before scheduling Phase 5 completion.

---

*Research completed: 2026-04-17*
*Ready for roadmap: yes*
