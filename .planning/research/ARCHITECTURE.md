# Architecture Patterns

**Domain:** Modular open-source hospitality operating system (plugin/marketplace model)
**Researched:** 2026-04-17
**Overall confidence:** MEDIUM-HIGH (plugin patterns from Backstage/VSCode ecosystem; OCR pipeline from current open-source landscape; Docker patterns from stable current documentation)

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     CORE PLATFORM                        │
│  Plugin Registry │ Event Bus │ Auth/RBAC │ Shared Schema │
└────────┬─────────────────────────────────────────┬───────┘
         │ Plugin API (REST + event subscriptions)  │
         ▼                                          ▼
┌─────────────────────┐              ┌──────────────────────┐
│  INVOICE MODULE      │              │  (Future Modules)    │
│  (Plugin v1)         │              │  Tips, POS, Cost Mgmt│
│                      │              └──────────────────────┘
│  Ingestion Service   │
│  OCR Pipeline        │
│  Extraction Engine   │
│  Review UI           │
│  Export Service      │
└──────────┬───────────┘
           │
           ▼
┌────────────────────────┐
│  EXTERNAL INTEGRATIONS  │
│  QBO API │ CSV/IIF sink │
└────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│                SELF-HOSTED INFRASTRUCTURE                │
│  Docker Compose │ PostgreSQL │ File Storage │ Backups    │
└─────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| **Plugin Registry** | Discovers, validates, and loads installed modules from filesystem; enforces manifest contracts | Core Platform (internal), all plugins |
| **Event Bus** | In-process pub/sub (Node.js EventEmitter or Redis pub/sub for multi-process); decouples plugins from core | All plugins; core services |
| **Auth / RBAC** | Session management, role enforcement (chef/receiver, GM/ops, accountant); JWT or session tokens | All modules; every HTTP route guard |
| **Shared Schema** | Canonical tables that all plugins read/write: `vendors`, `items`, `gl_accounts`, `users`, `roles`, `audit_log` | Core Platform (owns migrations); plugins (read/write via service layer) |
| **Ingestion Service** | Accepts uploads (multipart photo) and inbound emails (IMAP/SMTP webhook); normalizes to raw document record | OCR Pipeline (emits `document.received` event) |
| **OCR Pipeline** | Image preprocessing → OCR engine → layout-aware text extraction; outputs raw text + bounding boxes | Extraction Engine (writes to `ocr_results`) |
| **Extraction Engine** | Maps raw OCR output to structured invoice fields (vendor, date, line items, amounts); applies validation rules | Review UI (exposes pending extractions); Shared Schema |
| **Review / Correction UI** | Role-specific views for staff to confirm or correct extracted fields; chef sees receive quantities, GM sees totals, accountant sees GL mapping | Extraction Engine; Export Service |
| **Export Service** | Pushes approved invoices to QBO API or generates CSV/IIF; tracks export status | QBO API (external); filesystem for CSV |
| **Docker Compose Stack** | Orchestrates all services; handles startup ordering with health checks; named volumes for data persistence | All services via internal Docker network |

---

## Data Flow

```
INBOUND DOCUMENTS
     │
     │  photo upload (multipart HTTP)
     │  vendor email (IMAP listener → webhook)
     ▼
Ingestion Service
     │  stores raw file → emits document.received
     ▼
OCR Pipeline
  [1] Preprocessing: deskew, denoise, binarize (ImageMagick / PIL)
  [2] OCR Engine:    Tesseract 5 (local-first, CPU-only)
                     ↳ Cloud OCR (Google Vision / AWS Textract) as opt-in override
  [3] Layout Analysis: bounding boxes, table detection (Tesseract HOCR or PaddleOCR PP-Structure)
     │  writes to ocr_results table
     ▼
Extraction Engine
  [4] Field extraction: invoice_number, vendor_id, date, line_items[], amounts, tax
  [5] Vendor matching: fuzzy match against vendors table
  [6] GL suggestion: map items to gl_accounts by prior history
     │  writes to pending_invoices table → emits invoice.extracted
     ▼
Review UI
  [7] Chef/Receiver view: received qty vs invoiced qty
  [8] GM view: total cost, food cost %, approval
  [9] Accountant view: GL account mapping, AP close
     │  user approves → emits invoice.approved
     ▼
Export Service
  [10a] QBO API: create Bill + BillLineItems, push vendor payment terms
  [10b] CSV/IIF: format to chart-of-accounts mapping, write file
     │  marks invoice as exported
     ▼
EXTERNAL ACCOUNTING SYSTEM (QBO / generic GL)
```

Direction rule: data always flows forward through stages. Correction in Review UI writes back to `pending_invoices` (same table), never backward to OCR results — corrections are recorded as deltas, not re-runs.

---

## Plugin System Architecture

### Module Manifest (JSON)

Each plugin ships a `plugin.json` at its package root:

```json
{
  "id": "invoice-processing",
  "version": "1.0.0",
  "displayName": "Invoice Processing",
  "entrypoint": "dist/index.js",
  "permissions": ["vendors:read", "gl_accounts:read", "invoices:write"],
  "extensionPoints": ["ingestion.sources", "export.targets"],
  "subscribes": ["document.received"],
  "emits": ["invoice.extracted", "invoice.approved"]
}
```

### Discovery and Loading (v1: Filesystem)

At startup, the Plugin Registry:
1. Scans `$DATA_DIR/plugins/` for subdirectories containing `plugin.json`
2. Validates manifest schema and declared permissions
3. Imports the entrypoint module
4. Registers extension point implementations
5. Wires event subscriptions

No dynamic marketplace registry in v1 — operators install plugins by dropping directories (or running `tavernia plugin install <name>`). This eliminates runtime loading complexity until the pattern is proven.

### Extension Points

Plugins declare contribution points they expose. Other plugins implement them. Example: the Export Service exposes an `export.targets` extension point; a future Xero plugin implements it without touching Export Service code.

Pattern sourced from Backstage's backend extension point system: plugins create a shared interface, modules register implementations into it, and the plugin consumes all registered implementations at initialization time.

### Event Bus

In-process Node.js `EventEmitter` for v1 (single process). Upgrade path to Redis pub/sub when the deployment needs multiple processes or worker scaling.

Rules:
- Events are typed by topic string: `domain.action` (e.g., `invoice.approved`)
- Plugins publish and subscribe; core platform never subscribes to plugin events
- Event payloads are plain JSON objects — no class instances, no circular refs
- Plugins must not call each other's APIs directly — only through events or shared schema

---

## Shared Schema Primitives

These tables are owned by core and must exist before any plugin can initialize.

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `vendors` | Canonical vendor list shared across modules | `id`, `name`, `aliases[]`, `default_gl_account_id`, `tax_id` |
| `items` | Product/ingredient catalog | `id`, `name`, `unit`, `vendor_id`, `current_price` |
| `gl_accounts` | Chart of accounts (imported from QBO or manually seeded) | `id`, `code`, `name`, `type` (asset/expense/liability) |
| `users` | Platform user accounts | `id`, `email`, `hashed_password`, `role_id` |
| `roles` | Named role definitions | `id`, `name` (chef, gm, accountant, admin), `permissions[]` |
| `audit_log` | Append-only change log | `id`, `user_id`, `entity_type`, `entity_id`, `action`, `delta_json`, `ts` |
| `plugin_registry` | Installed plugin records | `id`, `plugin_id`, `version`, `enabled`, `config_json` |

Invoice module adds:

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `raw_documents` | Raw ingested files before OCR | `id`, `source` (upload/email), `file_path`, `mime_type`, `received_at` |
| `ocr_results` | OCR output per document | `id`, `document_id`, `engine`, `raw_text`, `hocr_xml`, `confidence`, `processed_at` |
| `pending_invoices` | Extracted fields awaiting review | `id`, `ocr_result_id`, `vendor_id`, `invoice_number`, `invoice_date`, `status` (pending/approved/rejected) |
| `invoice_line_items` | Line-by-line extracted rows | `id`, `invoice_id`, `item_id`, `description`, `qty`, `unit_price`, `total`, `gl_account_id` |
| `export_records` | Export history and status | `id`, `invoice_id`, `target` (qbo/csv), `external_id`, `exported_at`, `status` |

---

## Single-Tenant Isolation

Tavernia is single-tenant by design. One Docker Compose stack = one operator = one PostgreSQL instance. This eliminates:
- Row-level security (RLS) complexity
- Cross-tenant data leakage risks
- Shared credential surfaces

Isolation strategy:
- One `.env` file per deployment holds all secrets (DB password, QBO client secret, JWT secret)
- PostgreSQL listens only on the internal Docker network — never exposed on host ports in production config
- File storage (uploaded invoices, exported CSVs) lives in a named Docker volume on operator's hardware
- Backups: `pg_dump` via a lightweight backup container on a cron schedule, compressed to a local volume or operator-configured S3-compatible target

Optional managed hosting (one cloud VM per customer) follows the identical architecture — no architectural changes, only the target machine changes.

---

## Self-Hosting Infrastructure Patterns

### Docker Compose Service Graph

```
traefik (reverse proxy, TLS termination)
    └── app (Node.js API + UI server)
           └── postgres (PostgreSQL 16+)
           └── redis (optional, for job queuing)
           └── ocr-worker (Tesseract, isolated container)
backup-cron (pg_dump on schedule, independent)
```

### Health Check Pattern

Every service declares a health check. App container must wait for `postgres` to pass `pg_isready` before starting (via `depends_on: condition: service_healthy`). App exposes `GET /health` returning `{"status":"ok","db":"connected","version":"x.y.z"}` for external monitoring.

### Backup Strategy

```yaml
backup-cron:
  image: postgres:16-alpine
  environment:
    PGPASSWORD: ${DB_PASSWORD}
  volumes:
    - backups:/backups
  command: >
    sh -c "while true; do
      pg_dump -h postgres -U tavernia tavernia | gzip > /backups/tavernia_$(date +%Y%m%d_%H%M).sql.gz;
      find /backups -mtime +7 -delete;
      sleep 86400;
    done"
```

Operator documentation should explain: mount `/backups` to a NAS path or configure rclone to sync to cloud storage.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Shared Multi-Tenant Database
**What:** Row-level tenant isolation in a single PostgreSQL instance
**Why bad:** Defeats the security premise; adds schema complexity for zero benefit in single-operator deployments; creates migration risk as schema must coordinate across all tenants
**Instead:** One Compose stack per installation

### Anti-Pattern 2: Plugins Calling Each Other Directly
**What:** Invoice module importing and calling Export module functions directly
**Why bad:** Creates tight coupling; breaks plugin isolation; makes individual module extraction impossible
**Instead:** Publish events, share only schema; plugin-to-plugin communication always through the event bus or shared tables

### Anti-Pattern 3: Fat Core / Thin Plugins
**What:** Core platform accumulates domain logic (invoice parsing, export formatting) over time
**Why bad:** Defeats the modularity premise; "uninstalling" the invoice module becomes impossible because core depends on it
**Instead:** Core owns only: auth, plugin registry, event bus, shared schema, and web server bootstrap. Domain logic lives exclusively in plugins.

### Anti-Pattern 4: Cloud-First OCR Pipeline
**What:** Defaulting to Google Vision or AWS Textract as the primary OCR path
**Why bad:** Breaks the local-first data premise; vendor invoices (with pricing) leave the operator's infrastructure; ongoing cost per document
**Instead:** Tesseract 5 local-first, cloud as an opt-in override the operator explicitly enables with their own API key

### Anti-Pattern 5: Blocking OCR in the Request Thread
**What:** Running Tesseract synchronously during the HTTP upload request
**Why bad:** OCR on a photograph can take 2-15 seconds; blocks the server; degrades UI responsiveness
**Instead:** Ingestion Service stores the file and emits an event; OCR Pipeline processes asynchronously; Review UI polls or receives a push notification when extraction completes

---

## Suggested Build Order

This order respects hard dependencies and de-risks the plugin interface early.

### Phase 1: Core Platform
**What:** Plugin registry, event bus, auth/RBAC, shared schema, Docker Compose skeleton
**Why first:** Everything else depends on it. Building the invoice module before this means refactoring the interface later.
**Deliverable:** Empty platform with one "hello world" plugin that demonstrates discovery, registration, and event subscription

### Phase 2: Ingestion Service
**What:** File upload endpoint (multipart), inbound email listener (IMAP or SMTP webhook), raw document storage
**Why second:** Gate for the OCR pipeline; validates the plugin-to-core interface with a concrete, non-ML component
**Deliverable:** Uploaded photo persisted to volume; `document.received` event confirmed on event bus

### Phase 3: OCR Pipeline
**What:** Image preprocessing, Tesseract 5 integration, HOCR output parsing, confidence scoring
**Why third:** Depends on ingestion; isolated in its own worker container; testable with known invoice fixtures
**Deliverable:** Given a raw document, produces structured text with bounding boxes and field candidates

### Phase 4: Extraction Engine + Review UI
**What:** Field extraction rules (invoice number, vendor, date, line items), vendor fuzzy-matching, GL suggestion, role-specific review screens
**Why fourth:** Depends on OCR output schema being stable; review UI and extraction logic are tightly coupled (correction shapes the extraction model)
**Deliverable:** Staff can photograph an invoice, confirm extracted fields, and approve

### Phase 5: Export Service
**What:** QBO API integration (Bill + BillLineItem push), CSV/IIF fallback generator
**Why fifth:** Depends on approved invoices existing; QBO OAuth flow requires a stable domain/redirect URI; can be tested with CSV path first
**Deliverable:** Approved invoice appears in QBO as a Bill; CSV export works for non-QBO operators

### Phase 6: Production Hardening
**What:** Backup cron container, health checks, operator documentation, `.env` template, upgrade path (DB migration tooling)
**Why last:** Requires stable schema; hardening before schema stability wastes effort
**Deliverable:** `docker compose up` on a fresh machine produces a working system; operator can restore from backup

---

## Scalability Considerations

| Concern | At 1 restaurant | At 10 locations | At 100+ locations |
|---------|----------------|-----------------|-------------------|
| Database | Single Postgres on Docker volume | Same — one instance per location | Same — no architecture change, just more instances |
| OCR throughput | Synchronous Tesseract worker sufficient | Add Redis queue + worker pool | GPU-accelerated OCR or cloud override |
| File storage | Local Docker volume | NAS mount or local S3 (MinIO) | Same or operator-provided S3 |
| Deployment management | Manual `docker compose pull && up` | Bash script / Ansible | Managed hosting offering takes over |

Tavernia's single-tenant design means horizontal scale is "more installs," not "bigger cluster." This is a feature for the security model and a simplification for architecture.

---

## Sources

- Backstage Backend Plugin Extension Points: https://backstage.io/docs/backend-system/architecture/extension-points/ (HIGH confidence — official Backstage documentation)
- Open Source OCR Invoice Extraction Comparison: https://invoicedataextraction.com/blog/open-source-ocr-invoice-extraction (MEDIUM confidence — practitioner comparison, current 2025-2026)
- Invoice OCR Pipeline Architecture (2026): https://www.artsyltech.com/OCR-for-invoice-processing (MEDIUM confidence — industry overview)
- LLM-centric pipeline for invoice extraction: https://hal.science/hal-04772570v1/document (MEDIUM confidence — academic preprint)
- Docker Compose Health Checks: https://last9.io/blog/docker-compose-health-checks/ (HIGH confidence — well-maintained technical reference)
- Docker Compose depends_on with Health Checks: https://oneuptime.com/blog/post/2026-01-16-docker-compose-depends-on-healthcheck/view (HIGH confidence — current 2026 documentation)
- Plugin Architecture patterns: https://dev.to/ivangavlik/discovering-plug-in-play-architecture-in-java-4ibp (MEDIUM confidence — conceptual reference)
