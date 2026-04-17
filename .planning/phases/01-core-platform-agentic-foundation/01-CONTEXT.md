# Phase 1: Core Platform + Agentic Foundation - Context

**Gathered:** 2026-04-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Build the foundational scaffold — Docker skeleton, plugin registry, event bus, auth/RBAC, shared schema, AgentProvider abstraction, and wiki KB — so any developer can boot Tavernia locally, install a hello-world plugin, fire an event, and verify the agentic framework is wired end-to-end.

Creating/editing invoices, OCR, review UI, and QBO export are future phases. Phase 1 proves the contracts work, not that the features work.

</domain>

<decisions>
## Implementation Decisions

### AgentProvider interface
- Async method: `await agent.run(task, domain)` on an `AgentProvider` instance
- Provider loads `modules/{domain}/context.md` and the relevant `wiki/{domain}/` section on each call — no hardcoded context, no hardcoded provider SDK
- Returns an `AgentResult` dataclass: `content` (str), `domain` (str), `timestamp` (datetime), `metadata` (dict)
- `AgentProvider` is configured via `.env` (provider name + model); no module imports a provider SDK directly

### Wiki KB structure
- Flat files: `wiki/{domain}/{topic}.md` (e.g., `wiki/invoicing/vendor-profiles.md`, `wiki/invoicing/gl-patterns.md`)
- Every wiki file has YAML frontmatter with `created`, `last_updated`, and `domain` fields — date metadata is mandatory, not optional
- Each domain folder has an `index.md` that lists topics in the domain
- After each agent task, the outcome is appended as a structured markdown entry:
  ```
  ## [2026-04-17T14:32:00] [task summary]
  ### What was done
  ### What was learned
  ### Confidence
  high / medium / low
  ### Edge cases
  ```
- Date metadata on files + chronological entries within files gives agents a "recently touched" signal and temporal proximity lookup

### Plugin contract
- Phase 1 defines two pluggy hookspecs: `on_startup(registry)` and `on_event(event_name, payload)`
- Domain-specific hooks (`on_invoice_received`, `on_export_requested`, etc.) are added in the phases that introduce those domains
- Plugin manifest (declared in `plugin.py`): `name`, `version`, `description` — minimal for Phase 1; permissions and menu_items deferred to when plugins need them
- Hello-world plugin demonstrates the full loop: registers in `plugin_registry` table on startup → emits `hello.world.v1` event via event bus → a second listener receives and logs the delivery; this is the Phase 1 integration test

### Docker compose topology
- Four services in `docker compose up`: `postgres`, `redis`, `api` (FastAPI), `celery-worker`
- React frontend scaffolded in repo (Vite + shadcn/ui) but runs via `npm run dev` in development — not built into a Docker service in Phase 1
- No Celery Beat in Phase 1; it's added in Phase 2 when IMAP polling needs scheduling
- All four compose services must pass health checks for Phase 1 success criteria 1

### Auth / RBAC
- JWT stored in an `httpOnly`, `SameSite=Strict` cookie (not localStorage — protects against XSS on shared terminals)
- 15-minute access token expiry; no refresh token in Phase 1 — user re-logs in on expiry (refresh token flow deferred to Phase 7 hardening)
- Backend enforces RBAC via `user_roles` join table (`user_id`, `role_id`, `location_id NULLABLE`); no boolean role flags
- Frontend: login page → JWT cookie → `ProtectedRoute` components check role claim → redirect to role-gated shell route (`/chef`, `/gm`, `/accountant`) with placeholder content
- Proves CORE-05 (auth) and CORE-06 (role-appropriate routes) end-to-end

### Claude's Discretion
- Exact Alembic migration naming convention and directory structure
- Specific pluggy `hookspec` + `hookimpl` file layout within the `core/` module
- FastAPI dependency injection pattern for DB session and current-user context
- React route tree structure within each role shell
- YAML frontmatter field names beyond the mandatory date fields

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Tech stack (all Phase 1 technology decisions)
- `CLAUDE.md` (project root) — Full recommended stack with versions, what NOT to use, and alternatives considered. All technology decisions for Phase 1 are locked here. Read before writing any code.

### Requirements
- `.planning/REQUIREMENTS.md` — CORE-01 through CORE-07 and AGENT-01 through AGENT-06 are the Phase 1 requirements. Read for exact acceptance criteria.
- `.planning/ROADMAP.md` — Phase 1 success criteria (5 items) are the integration test checklist.

### No external specs yet
No external ADRs or design docs exist beyond the planning files above. All Phase 1 constraints are in CLAUDE.md, REQUIREMENTS.md, and decisions captured above.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — project is a blank repo (only CLAUDE.md exists). Phase 1 creates the foundation.

### Established Patterns
- None yet — Phase 1 establishes all patterns.

### Integration Points
- Phase 1 establishes the integration points that all future phases connect to:
  - `plugin_registry` table → Phase 2+ plugins register here
  - Event bus (`on_event` hookspec) → Phase 2 invoice ingestion fires `invoice.received.v1`
  - `AgentProvider.run(task, domain)` → Phase 3+ extraction, GL suggestion, vendor matching all call this
  - `wiki/{domain}/` KB → Phase 3+ agents read/write vendor profiles, GL patterns, extraction notes
  - Auth JWT cookie + RBAC → Phase 4 role-specific review views extend these route guards

</code_context>

<specifics>
## Specific Ideas

- Wiki date metadata: user specifically wants date info "even just as retrievable metadata" because temporal proximity is a useful fallback for finding recently relevant entries. YAML frontmatter on every wiki file is mandatory, not optional.
- Hello-world plugin must prove the *full* register → emit → deliver loop (not just register or emit), so Phase 1 integration test is unambiguous.
- Shared terminals context: httpOnly cookie choice is informed by restaurant environments where kitchen staff share workstations — XSS protection on the token matters more than API client convenience.

</specifics>

<deferred>
## Deferred Ideas

- Refresh token / silent refresh flow — Phase 7 production hardening
- Celery Beat (IMAP scheduler) — Phase 2
- nginx + React production build in Docker Compose — Phase 7 hardening
- Plugin permissions and menu_items manifest fields — Phase 2+ when plugins need them
- Domain-specific hookspecs (`on_invoice_received`, `on_export_requested`) — added in the phases that introduce those domains
- Vision-LLM extraction (GPU) — v2 requirement (out of scope for v1)

</deferred>

---

*Phase: 01-core-platform-agentic-foundation*
*Context gathered: 2026-04-17*
