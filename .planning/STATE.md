# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-17)

**Core value:** A hospitality operator's data stays on their own infrastructure, visible only to them, and the tools that process it are open-source, auditable, and natively agentic.
**Current focus:** Phase 1 — Core Platform + Agentic Foundation

## Current Position

Phase: 1 of 7 (Core Platform + Agentic Foundation)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-04-17 — Roadmap created; all 54 v1 requirements mapped across 7 phases

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: —
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: —
- Trend: —

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Phase 1: Plugin architecture from day 1 — no module coupling debt
- Phase 1: Agentic framework (AGENT-01–08) co-located in core, not a separate module
- Phase 1: python-jose abandoned/insecure — use PyJWT instead
- Phase 3: Catch-weight `pricing_type` field designed into schema before any extraction code is written
- Phase 5: QBO uses `Bill` endpoint (not `JournalEntry`) — critical for AP queue correctness
- Phase 5: OAuth token refresh must be atomic in DB transaction — rolling single-use tokens
- Phase 7: CORE-08 through CORE-12 deferred to Phase 7 (hardening); AGENT-07 and AGENT-08 also deferred to Phase 7 (continuity test + self-improvement loop)

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 1: Agentic framework design spike required — context file format, wiki KB schema, AgentProvider interface surface area; no open-source reference exists
- Phase 3: Do not commit to extraction approach before collecting 20+ real invoice samples (Sysco, local farm, dot-matrix, Word template formats)
- Phase 4: REVIEW-03 forward dependency — GM price deviation alerts depend on price history from EXPORT-09/10 (Phase 5). Phase 4 plan must implement graceful empty-state handling; Phase 5 plan must wire retroactive alert computation for existing approved invoices
- Phase 5: Intuit requires connected app review for production QBO access — investigate approval timeline before scheduling Phase 5 completion
- Phase 5: python-quickbooks is MEDIUM confidence (alpha status) — validate Bill entity field coverage against QBO v3 API reference before starting Phase 5
- Phase 7: Windows Docker Desktop path must be tested before declaring Windows support in documentation

## Session Continuity

Last session: 2026-04-17
Stopped at: Roadmap created; ROADMAP.md, STATE.md, and REQUIREMENTS.md traceability written; ready for /gsd:plan-phase 1
Resume file: None
