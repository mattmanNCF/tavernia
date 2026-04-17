# Technology Stack

**Project:** Tavernia — Modular Open-Source Hospitality OS
**Researched:** 2026-04-17
**Overall confidence:** HIGH (versions verified against PyPI; architecture patterns verified against current docs)

---

## Recommended Stack

### Core Backend

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Python | 3.12+ | Backend runtime | OCR, PDF parsing, and QBO integration all have substantially stronger Python ecosystems than Node. pdfplumber, surya-ocr, python-quickbooks are Python-only or Python-primary. Node alternatives for document AI are thin. |
| FastAPI | 0.136.0 | REST API framework | Native async, automatic OpenAPI docs, dependency injection pattern maps cleanly to plugin module boundaries, Pydantic v2 validation built-in. Faster than Django; more opinionated structure than Flask. |
| SQLAlchemy | 2.0.49 | ORM | 2.x async-first API with native asyncio support. Use `postgresql+asyncpg` URL. Single-tenant one-DB-per-install model fits cleanly — no row-level tenant filtering needed. |
| Alembic | 1.18.4 | Database migrations | Standard companion to SQLAlchemy. Use from day 1; run `alembic upgrade head` in Docker entrypoint before server start. |
| PostgreSQL | 16+ | Primary database | Single-tenant install, one DB per deployment. Postgres handles JSONB (useful for invoice line-item storage before structured extraction), full-text search, and transactional integrity. No shared schema complexity. |

### Document Processing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| surya-ocr | 0.17.1 | OCR for scanned/photo invoices | Transformer-based, handles multi-column layouts and table detection natively — critical for invoice line items. Outperforms Tesseract on structured documents. GPU recommended in production; CPU-only workable for low volume. 90+ language support. Self-hostable, no cloud dependency. |
| pdfplumber | 0.11.9 | Native PDF text/table extraction | For machine-generated vendor PDFs (the most common email invoice format). Use `page.extract_tables()` for line-item tables before falling back to OCR. Pixel-level table boundary control with visual debugging. Built on pdfminer.six. |
| pdf2image | 1.17+ | PDF-to-image conversion | Converts scanned PDF pages to images for OCR pipeline. Required when pdfplumber finds insufficient text (scanned invoice detection). Uses poppler under the hood — include in Docker image. |
| python-magic | 0.4.27+ | File type detection | Detect whether incoming file is native PDF, scanned PDF, or image before routing to correct pipeline. Avoids trust-the-extension bugs. |
| imapclient | 3.0.3+ | IMAP inbox polling | Vendors email PDFs directly to a monitored inbox — this is the second core invoice ingestion path (alongside photo upload). imapclient is higher-level than Python's built-in imaplib, handles TLS/OAuth2 SASL, and supports IDLE push notifications. Celery Beat runs the polling schedule. |

**Extraction pipeline logic:**
1. Receive file — either photo upload via web UI OR PDF fetched from IMAP inbox by Celery Beat
2. Detect type with python-magic
3. If native PDF → pdfplumber table extraction → structured output
4. If scanned PDF or image → pdf2image → surya-ocr → structured output
5. Post-process with LLM extraction prompt (optional, local Ollama or API) for unstructured layouts

**Email ingestion notes:**
- Celery Beat polls the configured inbox on a schedule (e.g., every 5 minutes)
- Store raw attachment bytes to file storage immediately on receipt; process async
- Configure a dedicated forwarding inbox per install (e.g., `invoices@operator-domain.com`) — do not share inboxes across installs
- OAuth2 SASL (Gmail, Outlook) is the modern auth path; imapclient supports it via the `oauth2` package

**Note on vision-LLM extraction (LOW confidence):** End-to-end vision-language models (Chandra, olmOCR) scored highest on benchmarks as of late 2025 but require 9B+ parameter models and significant GPU resources. Not recommended for v1 self-hosted deployment targeting restaurants without dedicated GPU hardware. Surya + pdfplumber covers 90%+ of real-world invoice formats and runs on CPU.

### QuickBooks Integration

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| python-quickbooks | 0.9.12 | QBO API client | The only practical Python option. Intuit does NOT publish an official Python SDK — only PHP, .NET, and Java SDKs are officially maintained. python-quickbooks wraps the QBO v3 REST API with OAuth2 via `intuitlib`. Last release April 16, 2025; active but self-described "alpha." |
| intuitlib | latest | OAuth2 flow for QBO | Intuit's official Python OAuth2 client. Used by python-quickbooks. Install alongside it. |

**QBO integration notes:**
- API is QBO v3 REST with minor versions. Intuit deprecated minor versions 1–74 as of August 1, 2025. Target minorVersion=75+ in all API calls.
- python-quickbooks covers Vendor, Bill, JournalEntry, Account objects — sufficient for AP workflow (invoice → bill → journal entry push).
- OAuth2 tokens expire; implement token refresh storage in the DB (per-install credentials, never shared).
- Build a thin abstraction layer over python-quickbooks so the community library can be replaced if it stagnates without rewriting business logic.

**Accounting export tiers (distinct targets, distinct complexity):**
- **QBO (primary):** python-quickbooks + intuitlib, OAuth2 connected app. Covers most restaurant operators.
- **CSV (universal fallback):** Standard Python `csv` module. Works with any accounting system that accepts CSV import. This is the v1 fallback for non-QBO shops.
- **IIF (QuickBooks Desktop — lower priority):** IIF is a legacy format for the standalone QuickBooks Desktop product, not QuickBooks Online. These are different products with different operators. IIF support is a separate, lower-priority feature deferred to a later milestone; do not conflate with the CSV fallback.

### Background Jobs

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Celery | 5.6.3 | Async task queue | Invoice OCR processing must be async — user uploads, gets job ID, polls for result. Celery handles retries, failure states, and scheduled jobs (IMAP polling). RQ is simpler but has worse reliability on worker crash (task loss). Celery's added complexity is justified by the retry/reliability requirements of invoice processing. |
| Redis | 7.4.0 (redis-py) | Celery broker + result backend | Standard Celery broker for single-server deployments. Also used for caching API responses. Run as a Docker service. |

### Authentication & Authorization

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PyJWT | 2.12.1 | JWT token encoding/decoding | FastAPI documentation previously recommended python-jose, which is now near-abandoned (last release 2021, known security issues). FastAPI project updated its docs to recommend PyJWT instead. Drop-in replacement with active maintenance. |
| passlib[bcrypt] | 1.7.4+ | Password hashing | bcrypt is the standard. Use `CryptContext` from passlib. |

**Role model:** Three roles (chef/receiver, GM/ops, accountant) map to FastAPI dependency-injected role checks. No external RBAC library needed for v1 — simple enum-based role on the User model.

### Plugin Architecture

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pluggy | 1.6.0 | Plugin hook system | Mature (status: "6 - Mature" on PyPI), battle-tested in pytest and tox. Define hook specs in core; modules register implementations. Lifecycle hooks: `on_startup`, `on_invoice_received`, `on_export_requested`, etc. |

**Plugin loading strategy:** Python `importlib.metadata` entry points for installed plugins (pip-installable), plus a local `plugins/` directory scan for development plugins. Pluggy manages hook dispatch. FastAPI `include_router()` used by each plugin to mount its routes under `/api/modules/{module_name}/`.

**Plugin interface contract:**
- Each plugin is a Python package with a `plugin.py` exposing a pluggy hookimpl
- Plugins declare metadata: name, version, required_permissions, menu_items
- Core provides: DB session factory, current user context, event bus hooks
- Plugins cannot import from other plugins directly (event bus only)

### Frontend

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| React | 19+ | UI framework | Dominant ecosystem, largest component library availability, shadcn/ui is built for it. Role-based routing is well-supported. |
| Vite | 6+ | Build tool | Standard for React in 2025. Fast HMR, simple config, better than CRA (deprecated). |
| TypeScript | 5.x | Type safety | Non-optional at this scale — three role views with distinct data models benefit from compile-time safety. |
| shadcn/ui | latest | Component library | Copy-in components (not a dependency), built on Radix UI + Tailwind. Zero lock-in, fully customizable, production-grade accessibility. The right choice over MUI for a self-hosted operator tool that needs to feel custom, not corporate. |
| Tailwind CSS | 4.x | Styling | Pairs with shadcn/ui. Utility-first prevents CSS sprawl in a multi-role app with distinct view layouts. |
| TanStack Query | 5.x | Server state | Standard for async data fetching in React. Handles invoice job polling, QBO sync status, optimistic updates. |
| React Router | 7.x | Client routing | Role-based route guards. Chef sees receiving views; accountant sees AP/export views. One SPA, different route trees per role. |
| Zustand | 5.x | Client state | Lightweight global state for auth context, user role, module registry. Redux is overkill for this scope. |

### Infrastructure & Deployment

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Docker | latest | Container runtime | Non-negotiable for self-hosted path. Operators need `docker compose up` to work. |
| Docker Compose | v2 | Multi-service orchestration | Single-host deployment: app, postgres, redis, celery worker, nginx. Compose v2 (plugin syntax) not v1 (standalone binary). |
| Nginx | 1.27+ (Alpine) | Reverse proxy | TLS termination, static asset serving, routes `/api` to FastAPI and `/` to React build. Include in Docker Compose service. |
| Docker named volume | — | File storage (self-hosted path) | Uploaded invoice images and PDFs must be persisted across container restarts. A named Docker volume (`tavernia_uploads`) is the correct v1 solution — operator-controlled, on their own hardware, no cloud dependency. Bind-mount to a host directory so operators can back it up with standard tools. |
| MinIO | latest | File storage (managed hosting path) | S3-compatible self-hosted object storage. Use for the optional managed single-tenant cloud path where operators don't manage their own hardware. Not needed for v1 self-hosted; design the storage abstraction layer to support both backends. |
| Traefik | 3.x | Optional managed hosting path | For the optional managed single-tenant cloud path, Traefik handles per-customer subdomain routing and auto TLS (Let's Encrypt). Not needed for self-hosted operator path. |

**Docker Compose service layout:**
```
services:
  db:          postgres:16-alpine
  redis:       redis:7-alpine
  app:         tavernia-api (FastAPI + Alembic migrations on start)
  worker:      tavernia-worker (Celery + Celery Beat, same image as app)
  frontend:    tavernia-frontend (nginx serving React build)

volumes:
  tavernia_uploads:    # invoice images and PDFs
  tavernia_pgdata:     # postgres data
```

**Single-tenant guarantee:** One `docker-compose.yml` per operator installation. DB credentials are per-install environment variables. No shared infrastructure between installs.

---

## What NOT to Use (and Why)

| Avoid | Category | Why |
|-------|----------|-----|
| Tesseract / pytesseract | OCR | Regex-era OCR, poor on table layouts. Surya significantly outperforms on structured documents. Only viable as emergency fallback. |
| Django | Backend | Django's ORM and admin are appealing, but the monolithic structure fights the plugin architecture. FastAPI's dependency injection is a better fit for modular boundaries. Django REST Framework adds complexity that FastAPI avoids by design. |
| Node.js / Express | Backend | No compelling OCR or QBO SDK ecosystem. pdfplumber and surya-ocr are Python-only. Would require a Python sidecar anyway — just use Python throughout. |
| Multi-tenant Postgres schemas | Database | Defeats the security premise. Row-level security is complex and schema-per-tenant is operationally messy. One DB per install is simpler and more auditable. |
| Prisma (Python port) | ORM | Not mature in Python. SQLAlchemy 2.x async is the established standard. |
| Redux Toolkit | Frontend state | Overkill. Zustand handles auth context and module registry without boilerplate. |
| python-jose | JWT | Near-abandoned (2021 last release), known security vulnerabilities. FastAPI docs migrated to PyJWT. |
| RQ (Redis Queue) | Task queue | Simpler than Celery, but tasks are silently dropped on worker crash. Invoice processing requires reliable retry. |
| Microservices | Architecture | Premature for v1. Modular monolith with plugin boundaries provides the right separation without Kubernetes complexity. Independent restaurants cannot operate a service mesh. |
| imaplib (stdlib) | Email ingestion | Too low-level. imapclient provides a higher-level API with proper TLS handling and IDLE support. |

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| OCR | surya-ocr | PaddleOCR | PaddleOCR is competitive on benchmarks but has a heavier install, more complex setup, and the PaddlePaddle framework is less common in the Python data stack. Surya is cleaner to install and Docker-image. |
| OCR | surya-ocr | Vision LLM (Chandra, olmOCR) | Best benchmark scores (83.1 on olmOCR-Bench) but requires 9B parameter model and GPU. Unsuitable for operator self-hosting on standard hardware. Flag for v2 optional enhancement. |
| Backend | FastAPI | Django | Django admin is compelling but fights plugin architecture. FastAPI wins on async, dependency injection, and modular structure. |
| Frontend | React + Vite | Next.js | SSR not needed — this is a role-specific dashboard, not a public web app. Next.js adds complexity (server components, API routes) that fights the FastAPI backend model. Vite + React is simpler. |
| Task queue | Celery + Redis | RQ | RQ loses tasks on worker crash. Celery's complexity cost is worth it for invoice reliability. |
| Plugin system | pluggy | Custom ABC + importlib | pluggy is battle-tested at pytest/tox scale. Custom plugin registry is reinventing it. |
| QBO auth | Direct OAuth2 | python-quickbooks full wrapper | python-quickbooks wraps both OAuth2 and API objects. The intuitlib layer is the official Intuit piece; keep both. |
| File storage (v1) | Docker named volume | S3 / MinIO | S3 requires cloud account; MinIO adds operational complexity. Named volume is correct for v1 self-hosted — operator controls it, backs it up with `docker volume`. MinIO is the upgrade path for managed hosting. |

---

## Installation

```bash
# Python backend core
pip install fastapi==0.136.0 uvicorn[standard] sqlalchemy==2.0.49 alembic==1.18.4 asyncpg pydantic-settings

# Document processing + email ingestion
pip install surya-ocr==0.17.1 pdfplumber==0.11.9 pdf2image python-magic imapclient

# QBO integration
pip install python-quickbooks==0.9.12 intuitlib

# Background jobs
pip install celery==5.6.3 redis==7.4.0

# Auth
pip install PyJWT==2.12.1 "passlib[bcrypt]"

# Plugin system
pip install pluggy==1.6.0

# Dev dependencies
pip install pytest pytest-asyncio httpx ruff mypy
```

```bash
# Frontend
npm create vite@latest tavernia-ui -- --template react-ts
cd tavernia-ui
npx shadcn@latest init
npm install @tanstack/react-query react-router-dom zustand
```

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| FastAPI + SQLAlchemy + Alembic | HIGH | Versions verified on PyPI; well-documented patterns; no controversy |
| PostgreSQL single-tenant | HIGH | Stable choice; architectural fit is clear |
| surya-ocr for invoice OCR | MEDIUM-HIGH | Surya v0.17.1 verified on PyPI; benchmark superiority over Tesseract verified in multiple sources; production GPU recommendation is consistent. "Medium" element: real-world invoice accuracy numbers for surya vs docTR not available from a single authoritative benchmark. |
| pdfplumber for native PDFs | HIGH | Well-established, actively maintained, widely used in invoice pipelines |
| imapclient for email ingestion | HIGH | Actively maintained, widely used for IMAP in Python, supports OAuth2 SASL |
| python-quickbooks for QBO | MEDIUM | Library is active (April 2025 release) but self-described "alpha." No official Intuit Python SDK exists. QBO v3 API is stable; wrapper may lag on new minor version features. Mitigated by thin abstraction layer recommendation. |
| PyJWT for auth | HIGH | FastAPI docs migrated to PyJWT; python-jose abandonment confirmed in FastAPI GitHub discussions |
| pluggy for plugin system | HIGH | Status "Mature" on PyPI; used by pytest, tox, pip |
| Celery + Redis task queue | HIGH | Industry standard; versions verified |
| React + Vite + shadcn/ui | HIGH | shadcn/ui is the dominant component system for self-built tools in 2025 |
| Docker Compose + named volume deployment | HIGH | Standard for single-host self-hosted apps |
| CSV export (non-QBO fallback) | HIGH | stdlib csv module, no library risk |
| IIF export (QB Desktop) | LOW | Deferred; separate target from QBO/CSV; complexity unresearched |

---

## Sources

- surya-ocr PyPI: https://pypi.org/project/surya-ocr/ (v0.17.1, Jan 2026)
- python-doctr PyPI: https://pypi.org/project/python-doctr/ (v1.0.1, Feb 2026)
- pdfplumber PyPI: https://pypi.org/project/pdfplumber/ (v0.11.9, Jan 2026)
- FastAPI PyPI: https://pypi.org/project/fastapi/ (v0.136.0, Apr 2026)
- SQLAlchemy PyPI: https://pypi.org/project/sqlalchemy/ (v2.0.49, Apr 2026)
- Alembic PyPI: https://pypi.org/project/alembic/ (v1.18.4, Feb 2026)
- Celery PyPI: https://pypi.org/project/celery/ (v5.6.3, Mar 2026)
- redis-py PyPI: https://pypi.org/project/redis/ (v7.4.0, Mar 2026)
- PyJWT PyPI: https://pypi.org/project/PyJWT/ (v2.12.1, Mar 2026)
- pluggy PyPI: https://pypi.org/project/pluggy/ (v1.6.0, May 2025)
- python-quickbooks PyPI: https://pypi.org/project/python-quickbooks/ (v0.9.12, Apr 2025)
- FastAPI python-jose discussion: https://github.com/fastapi/fastapi/discussions/9587
- OCR comparison (2025): https://unstract.com/blog/best-opensource-ocr-tools-in-2025/
- Invoice OCR Python comparison: https://invoicedataextraction.com/blog/python-ocr-library-comparison-invoices
- Python task queue comparison 2025: https://devproportal.com/languages/python/python-background-tasks-celery-rq-dramatiq-comparison-2025/
- Python plugin systems guide: https://oneuptime.com/blog/post/2026-01-30-python-plugin-systems/view
- FastAPI modular monolith patterns: https://github.com/arctikant/fastapi-modular-monolith-starter-kit
- Intuit QBO Python OAuth: https://developer.intuit.com/app/developer/qbo/docs/develop/sdks-and-samples-collections/python
- Docker Compose production: https://docs.docker.com/compose/how-tos/production/
