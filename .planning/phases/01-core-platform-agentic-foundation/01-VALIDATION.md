---
phase: 1
slug: core-platform-agentic-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-17
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest + pytest-asyncio + FastAPI TestClient (httpx) |
| **Config file** | `pyproject.toml [tool.pytest.ini_options]` — Wave 0 gap |
| **Quick run command** | `pytest tests/ -x -q` |
| **Full suite command** | `pytest tests/ -v --tb=short` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pytest tests/ -x -q`
- **After every plan wave:** Run `pytest tests/ -v --tb=short`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | CORE-07 | integration | `pytest tests/test_schema.py::test_core_tables_exist -x` | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 1 | CORE-01 | smoke | `docker compose up -d && docker compose ps` | ❌ W0 | ⬜ pending |
| 1-01-03 | 01 | 1 | CORE-02 | integration | `pytest tests/test_plugin_registry.py::test_hello_world_registered -x` | ❌ W0 | ⬜ pending |
| 1-01-04 | 01 | 1 | CORE-03 | integration | `pytest tests/test_event_bus.py::test_hello_world_event_delivered -x` | ❌ W0 | ⬜ pending |
| 1-01-05 | 01 | 1 | CORE-04 | unit | `pytest tests/test_schema_isolation.py::test_plugin_table_prefix -x` | ❌ W0 | ⬜ pending |
| 1-01-06 | 01 | 1 | CORE-05 | integration | `pytest tests/test_auth.py::test_login_sets_httponly_cookie -x` | ❌ W0 | ⬜ pending |
| 1-01-07 | 01 | 1 | CORE-06 | integration | `pytest tests/test_rbac.py::test_role_access_enforcement -x` | ❌ W0 | ⬜ pending |
| 1-01-08 | 01 | 1 | AGENT-01 | unit | `pytest tests/test_agent_provider.py::test_no_sdk_import_in_core -x` | ❌ W0 | ⬜ pending |
| 1-01-09 | 01 | 1 | AGENT-02 | unit | `pytest tests/test_agent_loader.py::test_context_file_loaded -x` | ❌ W0 | ⬜ pending |
| 1-01-10 | 01 | 1 | AGENT-03 | unit | `pytest tests/test_agent_loader.py::test_wiki_context_loaded -x` | ❌ W0 | ⬜ pending |
| 1-01-11 | 01 | 1 | AGENT-04 | integration | `pytest tests/test_wiki.py::test_outcome_appended -x` | ❌ W0 | ⬜ pending |
| 1-01-12 | 01 | 1 | AGENT-05 | integration | `pytest tests/test_wiki.py::test_user_profile_created -x` | ❌ W0 | ⬜ pending |
| 1-01-13 | 01 | 1 | AGENT-06 | integration | `pytest tests/test_wiki.py::test_distinct_user_profiles -x` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `pyproject.toml` — pytest config with `asyncio_mode = auto`
- [ ] `tests/conftest.py` — async DB session fixtures, TestClient, mock AgentProvider fixture, `AGENT_PROVIDER=mock` env override
- [ ] `app/agents/providers/mock.py` — deterministic mock provider (no real AI calls)
- [ ] `tests/test_schema.py` — CORE-07 table existence tests
- [ ] `tests/test_plugin_registry.py` — CORE-02/03/04 plugin registration + event delivery
- [ ] `tests/test_event_bus.py` — CORE-03 hello.world.v1 event delivery
- [ ] `tests/test_schema_isolation.py` — CORE-04 plugin table prefix enforcement
- [ ] `tests/test_auth.py` — CORE-05 login + httpOnly cookie
- [ ] `tests/test_rbac.py` — CORE-06 role-gated route enforcement
- [ ] `tests/test_agent_provider.py` — AGENT-01 import inspection (no SDK in core)
- [ ] `tests/test_agent_loader.py` — AGENT-02/03 context + wiki loading
- [ ] `tests/test_wiki.py` — AGENT-04/05/06 outcome append + user profile
- [ ] Framework install: `pip install pytest pytest-asyncio httpx python-frontmatter`

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `docker compose up` shows all 4 services healthy | CORE-01 | Requires running Docker daemon; CI-unsafe in some envs | Run `docker compose up -d`; run `docker compose ps`; confirm all services show `healthy` status |
| Frontend role routing (chef → /chef, gm → /gm) | CORE-06 | Browser interaction required | Log in as chef user; verify redirect to `/chef`; log in as gm user; verify redirect to `/gm`; verify cross-role route returns 403 |
| Vite dev server starts on npm run dev | CORE-01 | Frontend not in Docker Phase 1 | Run `cd frontend && npm run dev`; confirm Vite serves on localhost:5173 |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
