# Phase 1: Core Platform + Agentic Foundation - Research

**Researched:** 2026-04-17
**Domain:** FastAPI / pluggy / SQLAlchemy async / PyJWT / Docker Compose / AgentProvider abstraction
**Confidence:** HIGH (stack locked in CLAUDE.md with verified versions; research fills in wiring patterns)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**AgentProvider interface**
- Async method: `await agent.run(task, domain)` on an `AgentProvider` instance
- Provider loads `modules/{domain}/context.md` and the relevant `wiki/{domain}/` section on each call — no hardcoded context, no hardcoded provider SDK
- Returns an `AgentResult` dataclass: `content` (str), `domain` (str), `timestamp` (datetime), `metadata` (dict)
- `AgentProvider` is configured via `.env` (provider name + model); no module imports a provider SDK directly

**Wiki KB structure**
- Flat files: `wiki/{domain}/{topic}.md` (e.g., `wiki/invoicing/vendor-profiles.md`, `wiki/invoicing/gl-patterns.md`)
- Every wiki file has YAML frontmatter with `created`, `last_updated`, and `domain` fields — date metadata is mandatory, not optional
- Each domain folder has an `index.md` that lists topics in the domain
- After each agent task, the outcome is appended as a structured markdown entry: `## [timestamp] [task summary]` / `### What was done` / `### What was learned` / `### Confidence (high/medium/low)` / `### Edge cases`
- Date metadata on files + chronological entries within files gives agents a "recently touched" signal

**Plugin contract**
- Phase 1 defines two pluggy hookspecs: `on_startup(registry)` and `on_event(event_name, payload)`
- Domain-specific hooks deferred to the phases that introduce those domains
- Plugin manifest declared in `plugin.py`: `name`, `version`, `description` (minimal for Phase 1)
- Hello-world plugin must prove the FULL loop: registers in `plugin_registry` on startup → emits `hello.world.v1` → second listener receives and logs delivery

**Docker compose topology**
- Four services: `postgres`, `redis`, `api` (FastAPI), `celery-worker`
- React frontend scaffolded (Vite + shadcn/ui) runs via `npm run dev` in dev — NOT a Docker service in Phase 1
- No Celery Beat in Phase 1 (added Phase 2)
- All four compose services must pass health checks

**Auth / RBAC**
- JWT stored in `httpOnly`, `SameSite=Strict` cookie (not localStorage)
- 15-minute access token expiry; NO refresh token in Phase 1
- Backend enforces RBAC via `user_roles` join table (`user_id`, `role_id`, `location_id NULLABLE`); no boolean role flags
- Frontend: login page → JWT cookie → `ProtectedRoute` checks role claim → redirect to role-gated shell route (`/chef`, `/gm`, `/accountant`) with placeholder content

### Claude's Discretion
- Exact Alembic migration naming convention and directory structure
- Specific pluggy `hookspec` + `hookimpl` file layout within the `core/` module
- FastAPI dependency injection pattern for DB session and current-user context
- React route tree structure within each role shell
- YAML frontmatter field names beyond the mandatory date fields (`created`, `last_updated`, `domain`)

### Deferred Ideas (OUT OF SCOPE)
- Refresh token / silent refresh flow — Phase 7 production hardening
- Celery Beat (IMAP scheduler) — Phase 2
- nginx + React production build in Docker Compose — Phase 7 hardening
- Plugin permissions and menu_items manifest fields — Phase 2+ when plugins need them
- Domain-specific hookspecs (`on_invoice_received`, `on_export_requested`) — added in phases that introduce those domains
- Vision-LLM extraction (GPU) — v2 requirement
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| CORE-01 | `docker compose up` starts all services healthy | Docker Compose healthcheck patterns for postgres/redis/FastAPI/celery; `depends_on` with `condition: service_healthy` |
| CORE-02 | Plugin registry discovers and loads modules from `plugins/` via pluggy hookspecs | pluggy `PluginManager`, `HookspecMarker`, `HookimplMarker`, `add_hookspecs`, `register`; filesystem discovery via importlib |
| CORE-03 | Event bus emits and receives versioned events; no plugin subscribes directly to another plugin | pluggy `on_event` hookspec; event naming convention `domain.action.v1`; plugin isolation via bus |
| CORE-04 | Plugin schema isolation enforced: plugins own `{plugin_name}_*` tables; no ALTER on core tables | Alembic migration conventions; SQLAlchemy `MetaData` naming; plugin table prefix enforcement |
| CORE-05 | Auth: email + password login; JWT session via cookie; `user_roles` join table | PyJWT encode/decode; FastAPI `response.set_cookie(httponly=True, samesite="strict")`; passlib bcrypt |
| CORE-06 | Three built-in roles: `chef`, `gm`, `accountant`; role-appropriate route enforcement | FastAPI RBAC dependency; React `ProtectedRoute` component; JWT role claim |
| CORE-07 | Shared schema primitives: `vendors`, `gl_accounts`, `users`, `roles`, `user_roles`, `audit_log`, `plugin_registry` | SQLAlchemy 2.x declarative models; Alembic initial migration |
| AGENT-01 | `AgentProvider` abstraction: single interface; provider + model configurable via `.env`; no SDK imported in modules | Abstract base class pattern; `importlib` dynamic provider loading; `python-dotenv` / Pydantic settings |
| AGENT-02 | Domain context file convention: `modules/{name}/context.md`; loaded on each agent task | File I/O in AgentProvider; context.md format spec |
| AGENT-03 | Wiki KB at `wiki/{domain}/{topic}.md`; agents read relevant KB section on invocation | Markdown file I/O; YAML frontmatter parsing (`python-frontmatter` or manual regex); `index.md` per domain |
| AGENT-04 | Write-after-task protocol: structured outcome appended to `wiki/{domain}/` after each task | File append logic; structured markdown format; timestamp generation |
| AGENT-05 | `wiki/users/{username}.md` profile per user; updated by agents over time | User profile creation on first agent interaction; profile update logic |
| AGENT-06 | Multiple users per role share role but have distinct USER.md profiles | Profile keyed by username, not role; roles table separate from profiles |
</phase_requirements>

---

## Summary

Phase 1 creates the skeleton that all future phases build on. The stack is already locked in CLAUDE.md with verified package versions — no re-research needed there. This research fills in three gaps: (1) how to wire pluggy into a FastAPI app with proper file layout and hookspec/hookimpl patterns; (2) how to set up SQLAlchemy 2.x async sessions and JWT cookie auth in FastAPI using the correct patterns; and (3) how to structure the AgentProvider abstraction, which has no open-source reference (STATE.md flags this as a design spike).

The project has no existing code — Phase 1 creates everything from scratch. The key insight is that the integration test is the hello-world plugin: it must prove register → emit → deliver as a single verifiable loop. All four Docker services must pass health checks. The AgentProvider must load context files at runtime and write outcomes after every task — but must not import any provider SDK at the module level.

**Primary recommendation:** Build the directory scaffold and all core abstractions (pluggy hookspecs, AgentProvider ABC, AsyncSession dependency, Alembic initial migration) before any feature code. The scaffold is the deliverable; hello-world plugin validates the scaffold is correct.

---

## Standard Stack

The full stack is documented in `CLAUDE.md` (root). Below is the Phase 1 relevant subset with verified versions from CLAUDE.md sources.

### Core (Phase 1 subset)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| FastAPI | 0.136.0 | REST API, routing, dependency injection | Native async, Pydantic v2 built-in, OpenAPI docs auto |
| SQLAlchemy | 2.0.49 | Async ORM | `async_sessionmaker` + `AsyncSession` native asyncio; `postgresql+asyncpg` URL |
| Alembic | 1.18.4 | DB migrations | Standard SQLAlchemy companion; run `alembic upgrade head` at container startup |
| PostgreSQL | 16+ (Alpine) | Primary database | Single-tenant; JSONB; transactional integrity |
| Redis | 7.x (Alpine) | Celery broker | Docker service; broker + result backend |
| Celery | 5.6.3 | Async task queue | Phase 1 worker (no Beat yet); reliable retry |
| pluggy | 1.6.0 | Plugin hook system | Mature (status: "Mature" on PyPI); used by pytest, tox |
| PyJWT | 2.12.1 | JWT encode/decode | FastAPI docs migrated from python-jose; actively maintained |
| passlib[bcrypt] | 1.7.4+ | Password hashing | `CryptContext` with bcrypt |
| python-dotenv / Pydantic Settings | included in FastAPI | `.env` loading | Provider + model configuration for AgentProvider |

### Frontend (Phase 1 scaffold only — runs via `npm run dev`)
| Library | Version | Purpose |
|---------|---------|---------|
| React | 19+ | UI framework |
| Vite | 6+ | Build tool |
| TypeScript | 5.x | Type safety |
| shadcn/ui | latest | Components |
| Tailwind CSS | 4.x | Styling |
| React Router | 7.x | Role-based routing |
| Zustand | 5.x | Auth context state |

### Installation (backend)
```bash
pip install fastapi==0.136.0 uvicorn[standard] \
    sqlalchemy==2.0.49 asyncpg alembic==1.18.4 \
    celery==5.6.3 redis==7.4.0 \
    PyJWT==2.12.1 "passlib[bcrypt]" \
    pluggy==1.6.0 \
    python-dotenv pydantic-settings \
    python-frontmatter
```

---

## Architecture Patterns

### Recommended Project Structure
```
tavernia/
├── docker-compose.yml
├── .env.example
├── alembic/
│   ├── env.py
│   └── versions/
│       └── 0001_initial_schema.py
├── app/
│   ├── main.py                    # FastAPI app factory
│   ├── settings.py                # Pydantic Settings (loads .env)
│   ├── database.py                # engine, async_sessionmaker, get_db
│   ├── core/
│   │   ├── hooks.py               # HookspecMarker, hookspec classes
│   │   ├── plugin_manager.py      # PluginManager singleton, discovery
│   │   ├── event_bus.py           # on_event dispatcher wrapper
│   │   └── schema.py              # Core SQLAlchemy models (CORE-07 tables)
│   ├── api/
│   │   ├── auth.py                # /auth/login, /auth/me routes
│   │   ├── deps.py                # get_db, get_current_user, require_role
│   │   └── health.py              # /health endpoint (Docker healthcheck)
│   └── agents/
│       ├── provider.py            # AgentProvider ABC + AgentResult dataclass
│       ├── loader.py              # dynamic provider import, context/wiki loading
│       └── wiki.py                # wiki read/write/append helpers
├── modules/
│   └── hello_world/
│       └── context.md             # domain context file (AGENT-02)
├── plugins/
│   └── hello_world/
│       └── plugin.py              # hookimpl: on_startup, on_event
├── wiki/
│   ├── hello_world/
│   │   └── index.md              # domain index (AGENT-03)
│   └── users/                    # user profiles (AGENT-05/06)
└── frontend/
    ├── package.json
    ├── vite.config.ts
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── store/                 # Zustand auth store
        ├── components/
        │   └── ProtectedRoute.tsx
        └── pages/
            ├── Login.tsx
            ├── chef/Shell.tsx
            ├── gm/Shell.tsx
            └── accountant/Shell.tsx
```

### Pattern 1: pluggy Hook System

**What:** Define hookspecs in `core/hooks.py`, register a `PluginManager` singleton in `core/plugin_manager.py`, discover plugins from `plugins/` directory at startup.

**When to use:** Every plugin registration and every event dispatch goes through pluggy.

```python
# app/core/hooks.py
# Source: https://pluggy.readthedocs.io/en/latest/
import pluggy

hookspec = pluggy.HookspecMarker("tavernia")
hookimpl = pluggy.HookimplMarker("tavernia")

class TaverniaSpec:
    @hookspec
    def on_startup(self, registry) -> None:
        """Called once at app startup. Plugin registers itself in registry."""

    @hookspec
    def on_event(self, event_name: str, payload: dict) -> None:
        """Called when any versioned event is emitted on the event bus."""
```

```python
# app/core/plugin_manager.py
import importlib
import pluggy
from pathlib import Path
from app.core.hooks import TaverniaSpec

def build_plugin_manager(plugins_dir: Path) -> pluggy.PluginManager:
    pm = pluggy.PluginManager("tavernia")
    pm.add_hookspecs(TaverniaSpec)
    for plugin_path in plugins_dir.iterdir():
        if plugin_path.is_dir() and (plugin_path / "plugin.py").exists():
            spec = importlib.util.spec_from_file_location(
                f"plugins.{plugin_path.name}.plugin",
                plugin_path / "plugin.py"
            )
            mod = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(mod)
            pm.register(mod)  # registers any @hookimpl in the module
    return pm
```

```python
# plugins/hello_world/plugin.py
from app.core.hooks import hookimpl

class HelloWorldPlugin:
    @hookimpl
    def on_startup(self, registry) -> None:
        registry.register(name="hello_world", version="0.1.0")
        # Also emit the integration-test event
        from app.core.event_bus import emit_event
        emit_event("hello.world.v1", {"message": "Hello from hello_world plugin"})

    @hookimpl
    def on_event(self, event_name: str, payload: dict) -> None:
        if event_name == "hello.world.v1":
            print(f"[hello_world listener] received {event_name}: {payload}")
```

**Key non-obvious fact:** `pm.hook.on_startup(registry=reg)` calls ALL registered implementations. Results are returned as a list (LIFO order). If you want a single-result hook, use `@hookspec(firstresult=True)`.

**PluginRegistry helper class** (`app/core/plugin_manager.py`):
The `registry` argument passed to `on_startup` is a `PluginRegistryHelper` that wraps the DB session and exposes a `.register()` method that inserts/upserts a row in the `plugin_registry` table.

```python
# app/core/plugin_manager.py (PluginRegistryHelper addition)
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.schema import PluginRegistryRow  # SQLAlchemy model for plugin_registry table

class PluginRegistryHelper:
    """Passed to on_startup hooks. Plugins call self.register() to record themselves."""
    def __init__(self, session: AsyncSession):
        self._session = session

    async def register(self, name: str, version: str, description: str = "") -> None:
        """Upsert a row in plugin_registry (name is unique key)."""
        from sqlalchemy.dialects.postgresql import insert as pg_insert
        stmt = pg_insert(PluginRegistryRow).values(
            name=name, version=version, description=description
        ).on_conflict_do_update(
            index_elements=["name"],
            set_={"version": version, "description": description}
        )
        await self._session.execute(stmt)
```

Note: `on_startup` hookimpl signatures must be `async def on_startup(self, registry)` if they call `await registry.register()`. Pluggy supports async hooks — but only if the hookspec is called with `await pm.hook.on_startup(registry=reg)` (not all versions of pluggy do this natively; alternatively, keep `on_startup` synchronous and schedule `register()` via `asyncio.create_task()`). Safest Phase 1 pattern: run `on_startup` hooks synchronously; pass a sync-friendly registry that queues inserts and flushes after all hooks fire.

**Gap 2: emit_event in event_bus.py:**

```python
# app/core/event_bus.py
# emit_event is just a thin wrapper that calls pm.hook.on_event(...)
# pm is the PluginManager singleton built at app import time

from app.core.plugin_manager import _pm  # module-level singleton

def emit_event(event_name: str, payload: dict) -> None:
    """
    Dispatch a versioned event to all registered on_event hookimpls.
    Naming convention: domain.action.v1 (e.g. hello.world.v1)
    Plugins must NOT call emit_event on another plugin's internal events.
    """
    _pm.hook.on_event(event_name=event_name, payload=payload)
```

`_pm` is set in `plugin_manager.py` as a module-level variable after `build_plugin_manager()` is called:
```python
# app/core/plugin_manager.py (bottom of file)
_pm: pluggy.PluginManager | None = None

def init_plugin_manager(plugins_dir: Path) -> pluggy.PluginManager:
    global _pm
    _pm = build_plugin_manager(plugins_dir)
    return _pm
```


### Pattern 2: SQLAlchemy 2.x Async Session

**What:** `create_async_engine` + `async_sessionmaker` + a `get_db` dependency generator. Must use `postgresql+asyncpg` URL (not `postgresql://`).

**When to use:** Every database access in FastAPI route handlers.

```python
# app/database.py
# Source: https://leapcell.io/blog/building-high-performance-async-apis-with-fastapi-sqlalchemy-2-0-and-asyncpg
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.settings import settings

engine = create_async_engine(
    settings.database_url,   # must be postgresql+asyncpg://...
    echo=False,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,   # prevents DetachedInstanceError after commit
)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db

DBSession = Annotated[AsyncSession, Depends(get_db)]
```

### Pattern 3: JWT Cookie Auth

**What:** Login endpoint creates JWT, sets it as `httpOnly + SameSite=Strict` cookie on the response. Protected endpoints read cookie, verify JWT, return current user.

```python
# app/api/auth.py (login route)
import jwt
from datetime import datetime, timedelta, timezone
from fastapi import APIRouter, Depends, Response, HTTPException, status, Cookie
from fastapi.security import OAuth2PasswordRequestForm
from passlib.context import CryptContext
from app.settings import settings

router = APIRouter(prefix="/auth")
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
ACCESS_TOKEN_EXPIRE_MINUTES = 15

def create_access_token(data: dict) -> str:
    payload = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    payload["exp"] = expire
    return jwt.encode(payload, settings.secret_key, algorithm="HS256")

@router.post("/login")
async def login(
    response: Response,
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: DBSession = ...,
):
    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    token = create_access_token({"sub": user.username, "role": user.role})
    response.set_cookie(
        key="access_token",
        value=token,
        httponly=True,
        samesite="strict",
        max_age=ACCESS_TOKEN_EXPIRE_MINUTES * 60,
        secure=False,   # set True in production behind HTTPS
    )
    return {"message": "logged in"}
```

```python
# app/api/deps.py (current user dependency)
from fastapi import Cookie, HTTPException, status
import jwt

async def get_current_user(access_token: str = Cookie(None)):
    if not access_token:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    try:
        payload = jwt.decode(access_token, settings.secret_key, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

def require_role(*roles: str):
    async def checker(user = Depends(get_current_user)):
        if user.get("role") not in roles:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)
        return user
    return checker
```

### Pattern 4: AgentProvider Abstraction

**What:** Abstract base class that wraps any AI provider. Concrete implementations live in `agents/providers/`. The correct provider is loaded at runtime by reading `AGENT_PROVIDER` from `.env` — no provider SDK is imported at the top-level of any core module.

**Note on confidence:** This pattern has no open-source reference (STATE.md flags it as a design spike). The pattern below is derived from the CONTEXT.md decisions and standard Python ABC patterns. Mark implementation specifics LOW confidence — validate with a simple test call before relying on it.

```python
# app/agents/provider.py
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path

@dataclass
class AgentResult:
    content: str
    domain: str
    timestamp: datetime
    metadata: dict = field(default_factory=dict)

class AgentProvider(ABC):
    """Abstract provider. Concrete implementations in agents/providers/."""

    def __init__(self, model: str):
        self.model = model

    @abstractmethod
    async def _call_provider(self, messages: list[dict]) -> str:
        """Send messages to the underlying AI provider and return raw text."""
        ...

    async def run(self, task: str, domain: str, username: str | None = None) -> AgentResult:
        """
        Load domain context + relevant wiki KB, run the task,
        write outcome to wiki, return structured result.

        Args:
            task: Natural language description of the task to perform.
            domain: Domain name (e.g. 'hello_world', 'invoicing'). Maps to modules/{domain}/context.md.
            username: Optional. If provided, agent loads wiki/users/{username}.md as additional
                      context, and updates the user profile after task completion (AGENT-05/06).
                      If None, no user-specific context is loaded or written.
        """
        from app.agents.loader import load_context, load_wiki_context
        from app.agents.wiki import append_outcome

        context_md = load_context(domain)        # modules/{domain}/context.md
        wiki_ctx = load_wiki_context(domain)      # wiki/{domain}/*.md

        messages = [
            {"role": "system", "content": context_md + "\n\n" + wiki_ctx},
            {"role": "user", "content": task},
        ]
        raw = await self._call_provider(messages)
        result = AgentResult(
            content=raw,
            domain=domain,
            timestamp=datetime.utcnow(),
            metadata={"model": self.model},
        )
        await append_outcome(domain, task, result, username=username)
        return result
```

```python
# app/agents/loader.py
import importlib
from app.settings import settings

def get_agent_provider() -> AgentProvider:
    """
    Dynamically load provider from AGENT_PROVIDER env var.
    Example: AGENT_PROVIDER=anthropic loads agents/providers/anthropic.py
    No provider SDK is imported until this function runs.
    """
    provider_name = settings.agent_provider   # e.g., "anthropic", "openai", "ollama"
    module = importlib.import_module(f"app.agents.providers.{provider_name}")
    cls = getattr(module, "Provider")
    return cls(model=settings.agent_model)
```

### Pattern 5: Docker Compose Health Checks

**What:** All four services must pass health checks. API depends on postgres+redis being healthy; celery-worker also depends on both.

```yaml
# docker-compose.yml (Phase 1 skeleton)
# Source: https://oneuptime.com/blog/post/2026-02-08-how-to-set-up-a-fastapi-postgresql-celery-stack-with-docker-compose/view

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - tavernia_postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    volumes:
      - tavernia_redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build: .
    command: >
      sh -c "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  celery-worker:
    build: .
    command: celery -A app.celery_app worker --loglevel=info
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "celery", "-A", "app.celery_app", "inspect", "active"]
      interval: 15s
      timeout: 10s
      retries: 5

volumes:
  tavernia_postgres:
  tavernia_redis:
```

**REQUIRED:** `api` service needs a `GET /health` endpoint that returns 200. This is how the Docker healthcheck confirms the FastAPI app is ready.

### Pattern 6: Wiki KB File Format

**What:** Every wiki file has mandatory YAML frontmatter. Outcome entries are appended in chronological order.

```markdown
---
domain: hello_world
created: 2026-04-17T00:00:00
last_updated: 2026-04-17T14:32:00
---

# Hello World Wiki

## [2026-04-17T14:32:00] hello world integration test

### What was done
Fired hello.world.v1 event; second listener confirmed delivery.

### What was learned
pluggy calls all hookimpl implementations for on_event; LIFO order.

### Confidence
high

### Edge cases
Plugin must be registered before on_startup fires or it misses the event.
```

### Pattern 7: Alembic Migration Conventions (Claude's Discretion)

**Recommendation:** Use descriptive snake_case names with sequential numbering.

```
alembic/versions/
  0001_initial_schema.py      # CORE-07 tables: users, roles, user_roles, vendors, gl_accounts, audit_log, plugin_registry
  0002_hello_world_plugin.py  # hello_world plugin tables (if any)
```

`env.py` must import all SQLAlchemy models before `target_metadata = Base.metadata` so Alembic detects them. Run in entrypoint: `alembic upgrade head`.

### Anti-Patterns to Avoid
- **Using `sessionmaker` instead of `async_sessionmaker`:** Will raise `MissingGreenlet` at runtime (not at startup). FastAPI + asyncpg requires the async session factory — the error is silent until a route is hit.
- **Importing provider SDK at module top-level:** Violates AGENT-01. Any `import anthropic` or `import openai` must live only inside the concrete provider class file, never in `app/agents/provider.py` or any core module.
- **`expire_on_commit=True` (default):** Accessing model attributes after `await session.commit()` will raise `DetachedInstanceError`. Always pass `expire_on_commit=False` to `async_sessionmaker`.
- **Storing JWT in `localStorage`:** Decided against in CONTEXT.md. `httpOnly` cookie required. Do not implement a bearer token Authorization header path.
- **Hardcoding roles as boolean fields:** CONTEXT.md locked decision. Use `user_roles` join table with `(user_id, role_id, location_id NULLABLE)`.
- **Plugin importing from another plugin:** Violates CORE-03/04. All cross-plugin communication goes through the event bus.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Plugin hook discovery and ordering | Custom ABC + importlib registry | pluggy | pluggy handles call ordering, `tryfirst`/`trylast`, `firstresult` semantics, registration errors — all edge cases solved |
| Password hashing | Custom bcrypt wrapper | passlib `CryptContext` | Timing-safe comparison, upgrade-able schemes, deprecation handling |
| JWT encode/decode | Manual HMAC | PyJWT | Expiry checking, algorithm validation, error hierarchy |
| Async DB session lifecycle | Context manager inline in each route | `get_db` dependency with `yield` | FastAPI lifecycle guarantees cleanup even on exception |
| DB migration tracking | Manual SQL version table | Alembic | Dependency graph, downgrade support, autogenerate from model changes |
| YAML frontmatter parsing | Manual regex | `python-frontmatter` library | Handles edge cases in YAML block parsing, round-trips metadata without destroying content |

**Key insight:** pluggy in particular solves the hard problem of plugin ordering and isolation; building a custom hook system is reinventing it incorrectly.

---

## Common Pitfalls

### Pitfall 1: MissingGreenlet — Using Sync Session with asyncpg
**What goes wrong:** `sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called` at the first DB request. The app starts fine, fails only on the first route hit.
**Why it happens:** `sessionmaker` (sync) used instead of `async_sessionmaker`, or the connection URL is `postgresql://` instead of `postgresql+asyncpg://`.
**How to avoid:** Always `from sqlalchemy.ext.asyncio import async_sessionmaker`; always use `postgresql+asyncpg://` URL. Verify both in `database.py` before any other code.
**Warning signs:** App starts; first GET/POST to any DB-touching route raises `MissingGreenlet`.

### Pitfall 2: pluggy Called Before Registration
**What goes wrong:** `on_startup` fires and the plugin isn't registered yet because discovery runs after the hook call.
**Why it happens:** FastAPI `@app.on_event("startup")` fires at app start; if plugin discovery is not complete before that hook runs, plugins miss `on_startup`.
**How to avoid:** Build the `PluginManager` eagerly at module import time (before `app = FastAPI()`), not inside the startup handler. The startup handler only calls `pm.hook.on_startup(registry=reg)`.
**Warning signs:** Plugin appears in logs as discovered but `on_startup` callback never fires; `plugin_registry` table stays empty.

### Pitfall 3: Cookie SameSite / CORS Interaction
**What goes wrong:** Frontend on `localhost:5173` can't send the cookie to API on `localhost:8000` — the cookie is silently dropped.
**Why it happens:** `SameSite=Strict` blocks cross-site cookie sending. During dev, frontend and API are on different ports, which browsers treat as different origins.
**How to avoid:** In development, proxy frontend requests through Vite's `server.proxy` to the API (same origin from browser perspective). In production, nginx serves both under the same domain. Do not change cookie policy to `Lax` or `None` — fix the proxy configuration.
**Warning signs:** Login returns 200, cookie appears in DevTools, but next request to `/api/auth/me` returns 401 because cookie wasn't sent.

### Pitfall 4: DetachedInstanceError After Commit
**What goes wrong:** Accessing a model attribute after `await session.commit()` raises `sqlalchemy.orm.exc.DetachedInstanceError`.
**Why it happens:** Default `expire_on_commit=True` expires all attributes; with async sessions there's no implicit lazy load to re-fetch them.
**How to avoid:** Set `expire_on_commit=False` in `async_sessionmaker`. After insert, use `await session.refresh(obj)` if you need fresh DB values.
**Warning signs:** Attribute access on a model object immediately after commit raises `DetachedInstanceError`.

### Pitfall 5: AgentProvider SDK Import at Module Level
**What goes wrong:** Any module that imports from `app.agents.provider` triggers the AI provider SDK import, even in contexts that don't use agents.
**Why it happens:** `import anthropic` at the top of `provider.py` runs at Python module load time.
**How to avoid:** Concrete provider classes live in `app/agents/providers/{name}.py` and are loaded only via `importlib.import_module()` inside `get_agent_provider()`. The ABC in `provider.py` has zero provider imports.
**Warning signs:** `ModuleNotFoundError: anthropic` or `openai` when running tests that don't set AGENT_PROVIDER.

### Pitfall 6: Alembic env.py Missing Model Imports
**What goes wrong:** `alembic revision --autogenerate` produces an empty migration even though models changed.
**Why it happens:** `alembic/env.py` doesn't import the models before `target_metadata = Base.metadata`.
**How to avoid:** Add `import app.core.schema  # noqa` (and any plugin models) before the `target_metadata` line in `env.py`.
**Warning signs:** `autogenerate` says "No changes detected" after adding a new model.

---

## Code Examples

### FastAPI App Factory with pluggy
```python
# app/main.py
from contextlib import asynccontextmanager
from pathlib import Path
from fastapi import FastAPI
from app.core.plugin_manager import build_plugin_manager
from app.database import engine
from app.api import auth, health

PLUGINS_DIR = Path(__file__).parent.parent / "plugins"

pm = build_plugin_manager(PLUGINS_DIR)  # build eagerly at import time

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: fire on_startup hook for all plugins
    from app.core.schema import PluginRegistry
    from app.database import AsyncSessionLocal
    async with AsyncSessionLocal() as session:
        registry = PluginRegistry(session=session)
        pm.hook.on_startup(registry=registry)
        await session.commit()
    yield
    # Shutdown: (future) teardown hooks

app = FastAPI(lifespan=lifespan)
app.include_router(auth.router)
app.include_router(health.router)
```

### Health Endpoint (Required for Docker Healthcheck)
```python
# app/api/health.py
from fastapi import APIRouter
router = APIRouter()

@router.get("/health")
async def health():
    return {"status": "ok"}
```

### RBAC Route Guard (FastAPI)
```python
# app/api/example.py
from fastapi import APIRouter, Depends
from app.api.deps import require_role

router = APIRouter()

@router.get("/chef/dashboard")
async def chef_dashboard(user=Depends(require_role("chef"))):
    return {"role": "chef", "placeholder": True}
```

### React ProtectedRoute
```typescript
// frontend/src/components/ProtectedRoute.tsx
import { Navigate } from "react-router-dom";
import { useAuthStore } from "../store/auth";

interface Props {
  allowedRoles: string[];
  children: React.ReactNode;
}

export function ProtectedRoute({ allowedRoles, children }: Props) {
  const { user } = useAuthStore();
  if (!user) return <Navigate to="/login" replace />;
  if (!allowedRoles.includes(user.role)) return <Navigate to="/unauthorized" replace />;
  return <>{children}</>;
}
```

### Wiki Outcome Append
```python
# app/agents/wiki.py
from pathlib import Path
from datetime import datetime

WIKI_ROOT = Path(__file__).parent.parent.parent / "wiki"

async def append_outcome(domain: str, task: str, result) -> None:
    domain_dir = WIKI_ROOT / domain
    domain_dir.mkdir(parents=True, exist_ok=True)
    outcomes_file = domain_dir / "outcomes.md"
    entry = f"""
## [{result.timestamp.isoformat()}] {task[:80]}

### What was done
{result.content[:500]}

### What was learned
(agent-populated)

### Confidence
{result.metadata.get("confidence", "medium")}

### Edge cases
(agent-populated)

"""
    with outcomes_file.open("a", encoding="utf-8") as f:
        f.write(entry)
    # Update last_updated in index.md frontmatter
    await _update_index_timestamp(domain)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| python-jose for JWT | PyJWT | 2023–2024 | python-jose unmaintained; FastAPI docs migrated to PyJWT |
| SQLAlchemy 1.x sync sessions | SQLAlchemy 2.x `async_sessionmaker` | v2.0 (2023) | Old `Session()` pattern causes `MissingGreenlet` with asyncpg |
| CRA (Create React App) | Vite | 2022–2023 | CRA deprecated; Vite is the standard |
| `@app.on_event("startup")` | `lifespan` context manager | FastAPI 0.95+ | `on_event` deprecated; `lifespan` is the current pattern |
| `sessionmaker` sync factory | `async_sessionmaker` | SQLAlchemy 2.0 | Required for asyncio-native apps |

**Deprecated/outdated:**
- `python-jose`: Do not use. Abandoned, CVEs. Use PyJWT.
- `@app.on_event("startup")` / `@app.on_event("shutdown")`: Deprecated. Use `lifespan` context manager.
- `create_engine` (sync): Use `create_async_engine` with asyncpg for async FastAPI.
- Docker Compose v1 (`docker-compose` binary): Use v2 plugin syntax (`docker compose`).

---

## Open Questions

1. **AgentProvider concrete provider for hello-world integration test**
   - What we know: Interface is locked (ABC, `run(task, domain)`, `.env` config)
   - What's unclear: Does Phase 1 need a REAL provider call, or is a mock sufficient for the integration test?
   - Recommendation: Implement a `mock` provider (writes a deterministic fake response) so tests don't require API keys. Gate the real Anthropic/OpenAI provider behind `AGENT_PROVIDER=anthropic` in `.env`. Phase 1 integration test uses `mock` provider.

2. **Event bus: synchronous or async pluggy calls**
   - What we know: pluggy hooks are synchronous by default; FastAPI and Celery are async
   - What's unclear: Whether `on_event` hooks should be called synchronously (from an async context using `asyncio.run`) or wrapped in a thread executor
   - Recommendation: Keep `on_event` synchronous in Phase 1 (it's just logging). Phase 2+ can introduce async event handling if needed. Flag this as a design decision for the planner.

3. **python-frontmatter vs manual YAML parsing**
   - What we know: YAML frontmatter is mandatory on all wiki files (CONTEXT.md)
   - What's unclear: Whether to add `python-frontmatter` as a dependency
   - Recommendation: Add `python-frontmatter` — it's lightweight (~2KB), handles edge cases, and prevents YAML parsing bugs in wiki metadata. Alternative: manual `---` block split + `yaml.safe_load`, but this is fragile.

---

## Validation Architecture

> `nyquist_validation: true` in `.planning/config.json` — section is required.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest + pytest-asyncio + FastAPI TestClient |
| Config file | `pytest.ini` or `pyproject.toml [tool.pytest.ini_options]` — Wave 0 gap |
| Quick run command | `pytest tests/ -x -q` |
| Full suite command | `pytest tests/ -v --tb=short` |

**Additional required packages:**
```bash
pip install pytest pytest-asyncio httpx  # httpx required for FastAPI async TestClient
```

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CORE-01 | All 4 services start and pass healthchecks | smoke | `docker compose up -d && docker compose ps` (assert all healthy) | ❌ Wave 0 |
| CORE-02 | Plugin discovered from `plugins/` and registered in `plugin_registry` | integration | `pytest tests/test_plugin_registry.py::test_hello_world_registered -x` | ❌ Wave 0 |
| CORE-03 | `hello.world.v1` emitted and received by second listener | integration | `pytest tests/test_event_bus.py::test_hello_world_event_delivered -x` | ❌ Wave 0 |
| CORE-04 | Plugin table names follow `{plugin_name}_*` prefix; no ALTER on core tables | unit | `pytest tests/test_schema_isolation.py::test_plugin_table_prefix -x` | ❌ Wave 0 |
| CORE-05 | Login endpoint returns 200, cookie is set with httponly=True | integration | `pytest tests/test_auth.py::test_login_sets_httponly_cookie -x` | ❌ Wave 0 |
| CORE-06 | Chef route returns 200 for chef role, 403 for gm role | integration | `pytest tests/test_rbac.py::test_role_access_enforcement -x` | ❌ Wave 0 |
| CORE-07 | All 7 core tables exist after `alembic upgrade head` | integration | `pytest tests/test_schema.py::test_core_tables_exist -x` | ❌ Wave 0 |
| AGENT-01 | `AgentProvider.run()` called without importing any provider SDK in core modules | unit | `pytest tests/test_agent_provider.py::test_no_sdk_import_in_core -x` | ❌ Wave 0 |
| AGENT-02 | `modules/hello_world/context.md` loaded into agent context | unit | `pytest tests/test_agent_loader.py::test_context_file_loaded -x` | ❌ Wave 0 |
| AGENT-03 | `wiki/hello_world/index.md` read into agent context on invocation | unit | `pytest tests/test_agent_loader.py::test_wiki_context_loaded -x` | ❌ Wave 0 |
| AGENT-04 | Outcome entry appended to `wiki/hello_world/outcomes.md` after agent task | integration | `pytest tests/test_wiki.py::test_outcome_appended -x` | ❌ Wave 0 |
| AGENT-05 | `wiki/users/{username}.md` created on first agent interaction | integration | `pytest tests/test_wiki.py::test_user_profile_created -x` | ❌ Wave 0 |
| AGENT-06 | Two users sharing `chef` role each have distinct profile files | integration | `pytest tests/test_wiki.py::test_distinct_user_profiles -x` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `pytest tests/ -x -q` (fast subset, <30s)
- **Per wave merge:** `pytest tests/ -v --tb=short` (full suite)
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `tests/conftest.py` — async DB session fixtures, TestClient, mock AgentProvider fixture
- [ ] `tests/test_plugin_registry.py` — CORE-02/03/04
- [ ] `tests/test_event_bus.py` — CORE-03
- [ ] `tests/test_schema_isolation.py` — CORE-04
- [ ] `tests/test_auth.py` — CORE-05/06
- [ ] `tests/test_rbac.py` — CORE-06
- [ ] `tests/test_schema.py` — CORE-07
- [ ] `tests/test_agent_provider.py` — AGENT-01 (import inspection test)
- [ ] `tests/test_agent_loader.py` — AGENT-02/03
- [ ] `tests/test_wiki.py` — AGENT-04/05/06
- [ ] `pytest.ini` or `pyproject.toml` test config — async mode setting (`asyncio_mode = auto`)
- [ ] `app/agents/providers/mock.py` — mock provider for tests (no real API calls)
- [ ] Framework install: `pip install pytest pytest-asyncio httpx python-frontmatter`

**Critical:** All AGENT tests must use the `mock` AgentProvider, not a real AI provider. Tests must not make network calls to Anthropic/OpenAI. A `conftest.py` fixture that sets `AGENT_PROVIDER=mock` before any test is required.

---

## Sources

### Primary (HIGH confidence)
- CLAUDE.md (project root) — full stack with verified PyPI versions and sources; all Phase 1 technology decisions locked here
- pluggy docs: https://pluggy.readthedocs.io/en/latest/ — HookspecMarker, HookimplMarker, PluginManager API
- FastAPI + SQLAlchemy 2.0 async pattern: https://leapcell.io/blog/building-high-performance-async-apis-with-fastapi-sqlalchemy-2-0-and-asyncpg

### Secondary (MEDIUM confidence)
- Docker Compose health checks for FastAPI+Celery+Postgres+Redis stack: https://oneuptime.com/blog/post/2026-02-08-how-to-set-up-a-fastapi-postgresql-celery-stack-with-docker-compose/view
- FastAPI JWT cookie auth: https://retz.dev/blog/jwt-and-cookie-auth-in-fastapi/
- SQLAlchemy 2.0 patterns (2025): https://dev-faizan.medium.com/fastapi-sqlalchemy-2-0-modern-async-database-patterns-7879d39b6843

### Tertiary (LOW confidence)
- AgentProvider ABC design: derived from CONTEXT.md decisions + standard Python ABC patterns; no open-source reference exists (STATE.md confirms this is a design spike)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — versions locked and sourced in CLAUDE.md
- pluggy wiring: HIGH — official docs + multiple verified examples
- SQLAlchemy 2.x async: HIGH — verified pattern from current (2025/2026) sources
- JWT cookie auth: HIGH — verified against FastAPI ecosystem sources
- Docker healthchecks: HIGH — verified against official Docker + community sources
- AgentProvider design: LOW-MEDIUM — no open-source reference; derived from CONTEXT.md decisions
- Validation architecture: HIGH — standard pytest-asyncio patterns

**Research date:** 2026-04-17
**Valid until:** 2026-05-17 (stable stack; AgentProvider pattern may need revision after implementation spike)
