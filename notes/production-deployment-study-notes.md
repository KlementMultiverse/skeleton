# Level 15: Production Deployment

> Prerequisites: Docker basics, FastAPI, SQLAlchemy, Alembic, basic Linux commands.
> Goal: Ship a Python/FastAPI backend to production with zero-downtime deploys, proper health
> checks, secret management, observability, and a CI/CD pipeline that protects main.

---

## Part 1 — Docker Multi-Stage Builds

### The Analogy

Think of building a house. During construction you need scaffolding, power tools, cranes, and a
crew of workers. Once the house is finished, none of that stays inside. You hand over a clean,
finished building — no tools left behind.

A multi-stage Docker build works the same way. **Stage 1 (builder)** has all the tools: pip, uv,
gcc, git, build headers. **Stage 2 (production)** is the finished house — just your application
and its runtime, nothing used to build it.

The result: images that are 3–10x smaller, with a dramatically reduced attack surface.

---

### Why Image Size Matters

A 1.2 GB image with gcc, pip, build-essential, and git contains:
- Compilers an attacker can use to compile exploits
- Package managers that can install new software at runtime
- Source code and build artifacts that leak implementation details
- Slower deploys (pulling a 1.2 GB image over the network takes time)

A 120 MB production image contains only what runs your app.

---

### The Basic Pattern

```dockerfile
# ── Stage 1: builder ────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

# Install uv (fast Python package installer)
RUN pip install --no-cache-dir uv==0.5.14

WORKDIR /build

# Copy dependency files FIRST (layer cache optimization — explained below)
COPY pyproject.toml uv.lock ./

# Install into an explicit prefix so we can copy the whole venv to stage 2
RUN uv sync --frozen --no-dev --prefix /install

# ── Stage 2: production ──────────────────────────────────────────────────────
FROM python:3.12-slim AS production

# Copy the installed packages from builder — no pip, no uv, no build tools
COPY --from=builder /install /usr/local

# Create a non-root user (security hardening — explained below)
RUN useradd --uid 1000 --no-create-home --shell /sbin/nologin appuser

WORKDIR /app

# Copy application source — owned by appuser
COPY --chown=appuser:appuser src/ ./src/

# Switch to non-root before running anything
USER appuser

# Health check — Kubernetes / Railway will call this
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/live')"

EXPOSE 8000

CMD ["gunicorn", "src.main:app", "-c", "gunicorn.conf.py"]
```

---

### Layer Caching: Why Copy Requirements Before Code

Docker builds images in layers. Each instruction is a layer. If a layer's inputs haven't changed,
Docker reuses the cached layer — no rebuild needed.

The trick: **your dependencies change rarely, your code changes constantly**. If you copy everything
at once, every code change invalidates the dependency layer and triggers a full `pip install` on
every build.

```dockerfile
# BAD — copies everything at once, invalidates dep cache on every code change
COPY . .
RUN uv sync --frozen

# GOOD — copy deps first, install, then copy code
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev          # ← cached unless pyproject.toml changes
COPY --chown=appuser:appuser src/ ./src/ # ← code changes don't bust dep cache
```

Build time comparison on a typical project:
- Without layer caching: 90 seconds (full dep install every time)
- With layer caching (deps unchanged): 8 seconds (just copies new code)

---

### Non-Root User: Why It Matters

By default, Docker containers run as root (UID 0). If an attacker exploits your application and
gets code execution, they're root inside the container. With some container escape vulnerabilities,
that means root on the host.

Running as a non-root user (UID 1000) limits the blast radius:

```dockerfile
# Create the user with a specific UID, no home directory, no login shell
RUN useradd --uid 1000 --no-create-home --shell /sbin/nologin appuser

# Give the app directory to that user
COPY --chown=appuser:appuser src/ ./src/

# Switch — everything after this runs as appuser, including CMD
USER appuser
```

The `--no-create-home` and `--shell /sbin/nologin` flags prevent the user from logging in
interactively, which reduces attack surface further.

---

### ARG vs ENV: Build-Time vs Runtime Config

`ARG` values exist only during the build — they're not in the final image and not visible at runtime.
`ENV` values persist into the final image and are visible at runtime.

```dockerfile
# ARG — used during build, gone afterward (safe for build-time tokens)
ARG BUILD_DATE
ARG GIT_SHA
LABEL build.date="${BUILD_DATE}" build.sha="${GIT_SHA}"

# ENV — baked into the image, visible at runtime
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONFAULTHANDLER=1 \
    PORT=8000
```

**Critical rule**: NEVER put secrets (API keys, passwords, tokens) in `ARG` or `ENV` — they appear
in `docker inspect` and image layer history. Secrets belong in environment variables injected at
container startup by your platform (Railway Variables, Fly.io secrets, Kubernetes Secrets).

The `PYTHONUNBUFFERED=1` setting ensures stdout/stderr output is not buffered — logs appear
immediately instead of being held in a buffer until the process exits.

---

### .dockerignore

The `.dockerignore` file prevents unwanted files from entering the build context. Without it,
`COPY . .` sends your entire working directory (including `.git`, `__pycache__`, `.env`, test
fixtures, and node_modules) to the Docker daemon.

```
# .dockerignore
.git
.gitignore
.env
.env.*
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
.mypy_cache
.ruff_cache
.coverage
coverage.xml
htmlcov/
dist/
build/
*.egg-info/
.venv/
venv/
node_modules/
docs/
*.md
tests/
Dockerfile
docker-compose*.yml
.github/
```

A well-crafted `.dockerignore` reduces build context from hundreds of MB to a few KB, which
meaningfully speeds up builds.

---

### Docker Compose for Local Development

Production uses a baked image. Local development wants a bind mount — so code changes appear
instantly without rebuilding.

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      target: production       # build only to production stage (skip builder for local)
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src          # bind mount for hot-reload
    env_file:
      - .env                    # local secrets, never committed
    environment:
      - DEBUG=true
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/appdb
    depends_on:
      db:
        condition: service_healthy
    command: uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

The `condition: service_healthy` dependency means the API container won't start until the DB
container passes its health check — preventing "database refused connection" errors on startup.

---

### Complete Multi-Stage Example with uv

```dockerfile
# ── Stage 1: builder ────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1

# Install uv (binary installer, no pip dependency)
COPY --from=ghcr.io/astral-sh/uv:0.5.14 /uv /usr/local/bin/uv

WORKDIR /build

# Dependency files only — cache this layer aggressively
COPY pyproject.toml uv.lock ./

# Install prod deps to a virtual environment
RUN uv venv /opt/venv && \
    . /opt/venv/bin/activate && \
    uv sync --frozen --no-dev

# ── Stage 2: production ──────────────────────────────────────────────────────
FROM python:3.12-slim AS production

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONFAULTHANDLER=1 \
    PATH="/opt/venv/bin:$PATH"

# Security: non-root user
RUN useradd --uid 1000 --no-create-home --shell /sbin/nologin appuser

# Copy virtualenv from builder (no pip, no uv in prod image)
COPY --from=builder /opt/venv /opt/venv

WORKDIR /app

COPY --chown=appuser:appuser src/ ./src/
COPY --chown=appuser:appuser gunicorn.conf.py ./

USER appuser

HEALTHCHECK \
    --interval=30s \
    --timeout=5s \
    --start-period=15s \
    --retries=3 \
    CMD python -c \
        "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/live')" \
        || exit 1

EXPOSE 8000

CMD ["gunicorn", "src.main:app", "-c", "gunicorn.conf.py"]
```

Target image size with this approach: 130–180 MB for a typical FastAPI app (compared to 1.2 GB
with a naive single-stage build from `python:3.12`).

---

## Part 2 — Health Checks

### The Analogy

A hospital triage system has three levels of assessment:

1. **Is the patient breathing?** (alive at all)
2. **Is the patient stable enough to operate on?** (ready to receive work)
3. **Has the patient recovered from surgery?** (startup complete)

Health checks in Kubernetes (and Railway/Fly.io) use the same three levels:
- **Liveness probe** — is the process alive?
- **Readiness probe** — can it serve traffic?
- **Startup probe** — has it finished initializing?

---

### The Three Endpoints

```python
# src/api/health.py
from __future__ import annotations

import asyncio
import logging
import time
from typing import Any

from fastapi import APIRouter, HTTPException, status
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from src.database import engine
from src.cache import redis_client

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/health", tags=["health"])


@router.get("/live", status_code=status.HTTP_200_OK)
async def liveness() -> dict[str, str]:
    """
    Liveness probe: is the process alive?

    Returns 200 immediately. The only thing we check is that the process
    itself is running and the event loop is not blocked.

    Kubernetes: if this fails N times → restart the container.
    """
    return {"status": "alive"}


@router.get("/ready", status_code=status.HTTP_200_OK)
async def readiness() -> dict[str, Any]:
    """
    Readiness probe: can this instance serve traffic?

    Checks everything this instance needs to do its job:
    - Database connection pool has a live connection
    - Redis is reachable

    Kubernetes: if this fails N times → remove from load balancer rotation.
    The pod stays alive (liveness still passes) but gets no traffic.
    """
    checks: dict[str, str] = {}
    failed: list[str] = []

    # Check database
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception:
        logger.warning("Readiness: database check failed", exc_info=True)
        checks["database"] = "fail"
        failed.append("database")

    # Check Redis
    try:
        await redis_client.ping()
        checks["redis"] = "ok"
    except Exception:
        logger.warning("Readiness: redis check failed", exc_info=True)
        checks["redis"] = "fail"
        failed.append("redis")

    if failed:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail={"status": "not ready", "failed": failed, "checks": checks},
        )

    return {"status": "ready", "checks": checks}


@router.get("/startup", status_code=status.HTTP_200_OK)
async def startup_check() -> dict[str, Any]:
    """
    Startup probe: has the application finished initializing?

    Used for containers that need extra time before they're ready.
    Example: loading a large ML model, running initial cache warm-up,
    waiting for migrations to complete.

    Kubernetes: if this keeps failing → eventually kill and restart.
    Only runs during startup; once it succeeds once, readiness probe takes over.
    """
    from src.state import startup_complete

    if not startup_complete.is_set():
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail={"status": "starting", "message": "startup tasks not complete"},
        )

    return {"status": "started"}
```

---

### What to Check and What NOT to Check

**Check in /ready:**
- Your own database (you own the connection pool, failures here mean YOU can't serve)
- Your own Redis/cache (same logic)
- Any internal service you absolutely cannot function without

**Do NOT check in /ready:**
- Third-party APIs (Stripe, Sendgrid, OpenAI) — if they're down, your pod shouldn't be removed from the load balancer
- Other microservices that have fallback behavior (circuit breaker open = degraded, not down)
- Slow external health checks (if a check takes 3 seconds, your 5-second readiness timeout will fail)

The principle: only check things where failure means **this specific instance** cannot serve requests.

---

### Kubernetes Probe Configuration

```yaml
# kubernetes deployment excerpt
containers:
  - name: api
    image: ghcr.io/myorg/myapp:1.2.3
    ports:
      - containerPort: 8000

    # Liveness: is it alive? Restart if it dies.
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8000
      initialDelaySeconds: 5     # wait before first probe
      periodSeconds: 10          # probe every 10 seconds
      timeoutSeconds: 3          # fail if no response in 3s
      failureThreshold: 3        # restart after 3 consecutive failures

    # Readiness: can it serve traffic? Remove from LB if not.
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

    # Startup: only used during initial startup for slow-starting containers
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8000
      initialDelaySeconds: 10    # give it 10s before first check
      periodSeconds: 5           # then check every 5 seconds
      failureThreshold: 24       # fail after 24 failures = 120s total startup window
```

---

### Railway and Fly.io Equivalents

Railway uses the HEALTHCHECK Dockerfile instruction directly:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/live')" \
    || exit 1
```

Fly.io uses `fly.toml`:

```toml
# fly.toml
[http_service]
  internal_port = 8000
  force_https = true

  [[http_service.checks]]
    grace_period = "10s"       # wait before first check
    interval = "15s"
    method = "GET"
    path = "/health/live"
    port = 8000
    timeout = "5s"
    tls_skip_verify = false
```

---

### Graceful Shutdown

The analogy: when a restaurant closes at midnight, they don't throw out customers mid-meal. They
stop seating new tables (stop accepting requests), let current diners finish (drain in-flight
requests), then close.

When Kubernetes wants to shut down a pod, it sends SIGTERM. Your app should:
1. Stop accepting new connections
2. Wait for in-flight requests to complete (up to a deadline)
3. Close database connections and flush buffers
4. Exit cleanly

FastAPI's lifespan context manager handles this:

```python
# src/main.py
from __future__ import annotations

import asyncio
import logging
import signal
import sys
from contextlib import asynccontextmanager
from typing import AsyncGenerator

import sentry_sdk
from fastapi import FastAPI

from src.database import engine, SessionLocal
from src.cache import redis_client
from src.config import get_settings
from src.state import startup_complete

logger = logging.getLogger(__name__)
settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """
    Lifespan context manager:
    - Code before yield: runs on startup
    - Code after yield: runs on shutdown (even if startup raised an exception)
    """
    # ── Startup ──────────────────────────────────────────────────────────────
    logger.info("Starting application", extra={"version": settings.app_version})

    # Initialize Sentry before anything else so startup errors are captured
    if settings.sentry_dsn:
        sentry_sdk.init(
            dsn=settings.sentry_dsn.get_secret_value(),
            traces_sample_rate=settings.sentry_traces_sample_rate,
            environment=settings.environment,
            release=settings.app_version,
        )

    # Warm up the database connection pool
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        logger.info("Database connection pool initialized")
    except Exception:
        logger.error("Failed to connect to database on startup", exc_info=True)
        # Let startup fail loudly — don't start a broken app
        raise

    # Mark startup as complete (startup probe will now return 200)
    startup_complete.set()
    logger.info("Application startup complete")

    yield  # ← Application runs here

    # ── Shutdown ──────────────────────────────────────────────────────────────
    logger.info("Shutting down application")

    # Dispose the SQLAlchemy connection pool (closes all DB connections gracefully)
    await engine.dispose()
    logger.info("Database connection pool disposed")

    # Close Redis connection
    await redis_client.aclose()
    logger.info("Redis connection closed")

    # Flush Sentry buffer (background traces may not have been sent)
    if settings.sentry_dsn:
        sentry_sdk.flush(timeout=5.0)

    logger.info("Application shutdown complete")


app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    lifespan=lifespan,
    # Disable docs in production
    docs_url="/docs" if settings.environment != "production" else None,
    redoc_url="/redoc" if settings.environment != "production" else None,
)
```

The state singleton for the startup probe:

```python
# src/state.py
import asyncio

# A thread-safe event that the startup probe checks
startup_complete = asyncio.Event()
```

---

### Graceful Shutdown Timeout

When Kubernetes sends SIGTERM, it also starts a timer (`terminationGracePeriodSeconds`, default 30s).
If your app hasn't exited when the timer expires, it sends SIGKILL — which kills the process
immediately regardless of in-flight requests.

Gunicorn's graceful shutdown settings:

```python
# gunicorn.conf.py (excerpt — full file in Part 3)
graceful_timeout = 20   # workers get 20s to finish in-flight requests after SIGTERM
timeout = 120           # hard kill worker if it doesn't respond for 120s
```

Set `terminationGracePeriodSeconds` in Kubernetes to match:
```yaml
spec:
  terminationGracePeriodSeconds: 30   # must be > graceful_timeout
```

---

## Part 3 — Gunicorn + UvicornWorker

### The Analogy

Gunicorn is the restaurant manager. It keeps track of all the waiters (workers), restarts them if
they get overwhelmed, and makes sure there's always someone to take orders. Each waiter (UvicornWorker)
is a full Python process with its own event loop — they don't share memory or state. They just all
work the same kitchen (database).

---

### Worker Count Formula

The rule of thumb: `workers = (2 × CPU cores) + 1`

Why the +1? It accounts for one worker being occupied with a blocking task (a rare slow DB query,
for example) while the others handle new requests.

```
1 CPU core  → 3 workers
2 CPU cores → 5 workers
4 CPU cores → 9 workers
8 CPU cores → 17 workers
```

In practice: most production PaaS containers have 1–2 vCPUs, so you'll typically run 3–5 workers.

---

### Each Worker Has Its Own Event Loop

This is the critical thing to understand about Gunicorn + UvicornWorker:

```
Process: gunicorn (manager)
├── Worker 1 (UvicornWorker, PID 1234) — event loop, handles concurrent requests
├── Worker 2 (UvicornWorker, PID 1235) — separate event loop
├── Worker 3 (UvicornWorker, PID 1236) — separate event loop
└── Worker 4 (UvicornWorker, PID 1237) — separate event loop
```

Each worker is a completely separate OS process. They do not share:
- Memory (no shared Python objects)
- Connection pools (each worker has its own SQLAlchemy pool)
- Redis connections (each worker has its own)

This has a critical implication for **pool sizing**:

```
Total DB connections = num_workers × pool_size_per_worker

If max_connections = 100 and num_workers = 5:
  pool_size_per_worker = (100 × 0.8) / 5 = 16 connections per worker
  (0.8 = leave 20% for migrations, admin tools, monitoring)
```

---

### gunicorn.conf.py

```python
# gunicorn.conf.py
from __future__ import annotations

import multiprocessing
import os

# ── Binding ──────────────────────────────────────────────────────────────────
host = os.environ.get("HOST", "0.0.0.0")
port = os.environ.get("PORT", "8000")
bind = f"{host}:{port}"

# ── Worker count ─────────────────────────────────────────────────────────────
workers = int(os.environ.get("GUNICORN_WORKERS", (2 * multiprocessing.cpu_count()) + 1))
worker_class = "uvicorn.workers.UvicornWorker"

# ── Timeouts ─────────────────────────────────────────────────────────────────
# Hard kill a worker if it doesn't respond for 120s
# (increase for endpoints that legitimately take long, e.g., LLM calls)
timeout = int(os.environ.get("GUNICORN_TIMEOUT", "120"))

# How long to wait for workers to finish in-flight requests on SIGTERM
graceful_timeout = int(os.environ.get("GUNICORN_GRACEFUL_TIMEOUT", "20"))

# How long to keep an idle HTTP connection alive
keepalive = 5

# ── Memory optimization ───────────────────────────────────────────────────────
# Load the application before forking workers — workers share the loaded
# module memory (copy-on-write). Reduces per-worker RSS by 50-70MB typically.
# Caveat: database connections opened before fork are shared — only open
# connections in @app.on_event("startup") or lifespan, not at module level.
preload_app = True

# ── Logging ───────────────────────────────────────────────────────────────────
# Gunicorn's access log format — JSON-compatible tokens
access_log_format = (
    '{"time": "%(t)s", "method": "%(m)s", "path": "%(U)s", '
    '"status": %(s)s, "duration_ms": %(D)s, "ip": "%(h)s"}'
)
accesslog = "-"   # stdout
errorlog = "-"    # stderr
loglevel = os.environ.get("LOG_LEVEL", "info")

# ── Worker lifecycle hooks ────────────────────────────────────────────────────
def on_starting(server):  # type: ignore[no-untyped-def]
    """Called just before the master process is initialized."""
    pass


def worker_init(arbiter, worker):  # type: ignore[no-untyped-def]
    """Called just after a worker has been forked."""
    pass


def worker_exit(arbiter, worker):  # type: ignore[no-untyped-def]
    """Called just after a worker has been exited."""
    pass


def post_fork(server, worker):  # type: ignore[no-untyped-def]
    """
    Called after a worker has been forked.
    Use this to close any file descriptors or connections opened in preload_app
    that should not be shared between workers.
    """
    pass
```

---

### UvicornWorker vs Uvicorn Direct

| | `uvicorn` directly | `gunicorn` + `UvicornWorker` |
|---|---|---|
| Multiple workers | No (single process) | Yes (multi-process) |
| Auto-restart crashed worker | No | Yes |
| Graceful reload | No | Yes (`kill -HUP $GUNICORN_PID`) |
| Memory isolation per worker | N/A | Yes |
| Use case | Development | Production |

For production: always use Gunicorn. For development: `uvicorn --reload` is fine.

---

### Zero-Downtime Reload

When you deploy a new version, you want to reload workers without dropping requests. Gunicorn
supports this with `SIGHUP`:

```bash
# Send SIGHUP to the Gunicorn master process
kill -HUP $(cat /tmp/gunicorn.pid)
```

Gunicorn will:
1. Spin up new workers with the new code
2. Wait for old workers to finish in-flight requests (up to `graceful_timeout`)
3. Kill old workers after they drain

Zero requests dropped. The PID file path is configured in `gunicorn.conf.py`:
```python
pidfile = "/tmp/gunicorn.pid"
```

---

## Part 4 — Zero-Downtime Migrations

### The Analogy

Imagine you're renovating a live restaurant kitchen. You can't close it — the restaurant must
keep serving food. So you:
1. Build the new prep station without tearing down the old one (**expand**)
2. Train the chefs to use both stations (**dual-read period**)
3. Move all the prep work to the new station (**migrate data**)
4. Tear down the old station once nothing uses it (**contract**)

You never have a moment where customers can't eat. This is the **expand/migrate/contract** pattern.

---

### The Expand / Migrate / Contract Pattern

**Phase 1: Expand — add new things, don't remove old things**

Old schema: `users.name VARCHAR(255)`
New schema: we want `users.first_name` and `users.last_name` separately.

Deploy 1 — add new columns (nullable!):
```sql
ALTER TABLE users ADD COLUMN first_name VARCHAR(255);
ALTER TABLE users ADD COLUMN last_name VARCHAR(255);
```

Application code in this deploy: reads from `name`, writes to both `name` AND `first_name`/`last_name`.

```python
# Alembic migration for Phase 1 expand
def upgrade() -> None:
    op.add_column(
        "users",
        sa.Column("first_name", sa.String(255), nullable=True),
    )
    op.add_column(
        "users",
        sa.Column("last_name", sa.String(255), nullable=True),
    )
```

**Phase 2: Backfill — migrate existing data**

```python
# Alembic data migration
def upgrade() -> None:
    # Split existing name into first_name + last_name
    op.execute("""
        UPDATE users
        SET
            first_name = split_part(name, ' ', 1),
            last_name = nullif(substring(name from position(' ' in name) + 1), '')
        WHERE first_name IS NULL
    """)
```

For large tables, backfill in batches to avoid locking:

```python
def upgrade() -> None:
    conn = op.get_bind()
    # Batch by ID to avoid full table lock
    batch_size = 1000
    offset = 0
    while True:
        result = conn.execute(
            text("""
                UPDATE users
                SET first_name = split_part(name, ' ', 1),
                    last_name = nullif(substring(name from position(' ' in name) + 1), '')
                WHERE id IN (
                    SELECT id FROM users
                    WHERE first_name IS NULL
                    ORDER BY id
                    LIMIT :batch_size OFFSET :offset
                )
            """),
            {"batch_size": batch_size, "offset": offset},
        )
        if result.rowcount == 0:
            break
        offset += batch_size
```

**Phase 3: Contract — remove old things once nothing uses them**

Deploy 3 — application no longer reads/writes `name`:
```python
def upgrade() -> None:
    op.drop_column("users", "name")
```

---

### NEVER Rename a Column in One Migration

Renaming requires a multi-deploy sequence:
1. Add the new column (both old and new exist)
2. Dual-write to old and new in application code
3. Backfill: copy old to new for existing rows
4. Switch application to read from new only
5. Remove old column

This takes 3–4 deploys. The alternative — `ALTER TABLE users RENAME COLUMN name TO full_name` in
one migration — will break any running instance that still references `name`. In a rolling deploy
with 5 pods, 3 old pods and 2 new pods run simultaneously for 30 seconds. If the old pods try to
`SELECT name FROM users` after the rename, they get a database error.

---

### CREATE INDEX CONCURRENTLY

A regular `CREATE INDEX` takes an `AccessShareLock` that blocks all writes for the duration of the
index build. On a large table, this can take minutes.

`CREATE INDEX CONCURRENTLY` builds the index without blocking writes — it does it in multiple
passes, allowing normal read/write operations throughout.

```python
# Alembic migration with concurrent index creation
def upgrade() -> None:
    # Must run outside a transaction — concurrent index creation cannot be
    # done inside a transaction block
    op.execute("COMMIT")  # end the auto-started transaction
    op.execute(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_author_id "
        "ON posts (author_id)"
    )

def downgrade() -> None:
    op.execute("COMMIT")
    op.execute("DROP INDEX CONCURRENTLY IF EXISTS idx_posts_author_id")
```

Or configure Alembic to not use transactions for this migration:

```python
# In the migration file, add at top level:
# revision: ...
# ...

# Tell Alembic this migration manages its own transactions
transaction_per_migration = False
```

**Gotcha**: after a failed `CREATE INDEX CONCURRENTLY`, you'll have an `INVALID` index in
`pg_indexes`. Always check:

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public' AND indexname LIKE '%INVALID%';

-- Or more reliably:
SELECT indexname
FROM pg_indexes
JOIN pg_class ON pg_class.relname = pg_indexes.indexname
JOIN pg_index ON pg_index.indexrelid = pg_class.oid
WHERE pg_index.indisvalid = false;
```

Drop invalid indexes before retrying: `DROP INDEX CONCURRENTLY idx_posts_author_id;`

---

### Column Type Changes

Some type changes PostgreSQL can do instantly (no table rewrite):
- VARCHAR to TEXT
- Increasing VARCHAR length
- Adding/removing NOT NULL constraint with a DEFAULT

Some require a full table rewrite (blocks reads and writes for the table size):
- Changing from VARCHAR to INTEGER
- Decreasing VARCHAR length
- Changing column type in a way that requires data conversion

For type changes requiring a rewrite, use the expand/contract pattern:
1. Add new column with correct type (nullable)
2. Backfill
3. Make NOT NULL (once all rows backfilled)
4. Drop old column

---

### Run Migrations in Deploy Pipeline, NOT in Lifespan

The temptation: run `alembic upgrade head` inside the FastAPI lifespan startup hook, so
migrations always run before the app serves traffic.

The problem:
- With 5 workers/pods, 5 instances run `alembic upgrade head` simultaneously
- Race condition: all 5 try to run the same migration at the same time
- Alembic has a lock, but this is still dangerous and slow

The correct pattern: **run migrations as a separate step before starting the app**.

Railway deploy command:
```bash
alembic upgrade head && gunicorn src.main:app -c gunicorn.conf.py
```

Kubernetes: use an init container:
```yaml
initContainers:
  - name: migrate
    image: ghcr.io/myorg/myapp:1.2.3
    command: ["alembic", "upgrade", "head"]
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: app-secrets
            key: database-url
```

The init container runs to completion before the main container starts. If migrations fail, the
pod never starts — preventing a broken deploy from reaching users.

---

### Alembic Configuration

```ini
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = %(DATABASE_URL)s   # read from env var

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

```python
# alembic/env.py
from __future__ import annotations

import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import async_engine_from_config

from src.database import Base  # your SQLAlchemy Base

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=config.get_main_option("sqlalchemy.url"),
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):  # type: ignore[no-untyped-def]
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

---

## Part 5 — Configuration and Secrets

### The Analogy

Configuration is like a restaurant's recipe book. The recipes (code) stay the same. What changes
per kitchen (environment) is the ingredient sources (database URLs, API keys, feature flags).

The Twelve-Factor App methodology says: **configuration belongs in the environment, not the code**.

---

### pydantic-settings BaseSettings

```python
# src/config.py
from __future__ import annotations

from functools import lru_cache
from typing import Literal

from pydantic import Field, PostgresDsn, RedisDsn, SecretStr, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """
    All configuration comes from environment variables or .env file.
    Fields with no default = REQUIRED — startup fails if they're missing.
    Fields with defaults = optional with sensible fallbacks.
    """

    model_config = SettingsConfigDict(
        # Load from .env file in development
        env_file=".env",
        env_file_encoding="utf-8",
        # Reject unknown environment variables — catches typos
        extra="forbid",
        # Nested settings delimiter: DATABASE__POOL_SIZE maps to database.pool_size
        env_nested_delimiter="__",
        # Don't cache between tests — each test can set its own env vars
        case_sensitive=False,
    )

    # ── Application ───────────────────────────────────────────────────────────
    app_name: str = "My API"
    app_version: str = "0.0.1"
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = False
    log_level: str = "INFO"

    # ── Server ────────────────────────────────────────────────────────────────
    host: str = "0.0.0.0"
    port: int = 8000

    # ── Database ─────────────────────────────────────────────────────────────
    # No default → REQUIRED → startup raises ValueError if DATABASE_URL missing
    database_url: PostgresDsn = Field(...)
    database_pool_size: int = Field(default=10, ge=1, le=100)
    database_max_overflow: int = Field(default=20, ge=0)
    database_pool_timeout: int = Field(default=30, ge=1)

    # ── Redis ─────────────────────────────────────────────────────────────────
    redis_url: RedisDsn = Field(...)

    # ── Security ─────────────────────────────────────────────────────────────
    # SecretStr renders as '**********' in logs and repr()
    secret_key: SecretStr = Field(...)
    jwt_algorithm: str = "RS256"
    access_token_expire_minutes: int = Field(default=30, ge=1)

    # ── External APIs ─────────────────────────────────────────────────────────
    openai_api_key: SecretStr | None = None
    stripe_secret_key: SecretStr | None = None
    sendgrid_api_key: SecretStr | None = None

    # ── Observability ─────────────────────────────────────────────────────────
    sentry_dsn: SecretStr | None = None
    sentry_traces_sample_rate: float = Field(default=0.1, ge=0.0, le=1.0)

    # ── CORS ─────────────────────────────────────────────────────────────────
    allowed_origins: list[str] = Field(default=["http://localhost:3000"])

    @field_validator("environment")
    @classmethod
    def validate_not_debug_in_production(cls, v: str, info):  # type: ignore[no-untyped-def]
        # info.data contains already-validated fields
        if v == "production" and info.data.get("debug", False):
            raise ValueError("debug=True is not allowed in production")
        return v

    @field_validator("database_url", mode="before")
    @classmethod
    def fix_postgres_scheme(cls, v: str) -> str:
        """
        Railway and Heroku export DATABASE_URL with 'postgres://' scheme.
        SQLAlchemy async requires 'postgresql+asyncpg://'.
        """
        if isinstance(v, str) and v.startswith("postgres://"):
            return v.replace("postgres://", "postgresql+asyncpg://", 1)
        return v


@lru_cache
def get_settings() -> Settings:
    """
    Returns the settings singleton.

    @lru_cache means this is computed once and cached. The same Settings
    object is returned on every call — no re-reading environment variables.

    In tests: use `app.dependency_overrides[get_settings] = lambda: test_settings`
    to inject test configuration.
    """
    return Settings()
```

The `Field(...)` syntax means "required, no default". If `DATABASE_URL` is not set, Pydantic
raises a `ValidationError` before the app serves a single request. This is correct — you want
a loud failure at startup, not a mysterious 500 error when the first database query runs.

---

### Using Settings in FastAPI

```python
# src/database.py
from __future__ import annotations

from functools import lru_cache

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from src.config import get_settings


@lru_cache
def get_engine():  # type: ignore[no-untyped-def]
    settings = get_settings()
    return create_async_engine(
        str(settings.database_url),
        pool_size=settings.database_pool_size,
        max_overflow=settings.database_max_overflow,
        pool_timeout=settings.database_pool_timeout,
        pool_pre_ping=True,   # verify connections before use
        echo=settings.debug,  # log SQL in debug mode
    )


engine = get_engine()

SessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

```python
# FastAPI dependency injection
from fastapi import Depends
from src.config import Settings, get_settings

def require_api_key(
    api_key: str,
    settings: Settings = Depends(get_settings),
) -> None:
    if api_key != settings.secret_key.get_secret_value():
        raise HTTPException(status_code=401, detail="Invalid API key")
```

---

### Secret Management Hierarchy

From least to most secure:

1. **Hardcoded in code** — NEVER. Leaks in git history, logs, `docker inspect`.
2. **.env file committed** — NEVER. Even with `.gitignore`, one slip = breach.
3. **.env file not committed** — OK for local dev. Not for CI/CD pipelines.
4. **CI/CD environment variables** (GitHub Actions secrets, GitLab CI variables) — OK for CI. Encrypted at rest, not in image.
5. **Platform secrets** (Railway Variables, Fly.io secrets) — Good for small apps. Encrypted, injected at runtime.
6. **Secret manager** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) — Best practice for production. Rotation support, audit logs, access control.

For AWS Secrets Manager:

```python
# src/secrets.py
from __future__ import annotations

import json
import logging

import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)


def get_secret(secret_name: str, region: str = "us-east-1") -> dict[str, str]:
    """
    Fetch a secret from AWS Secrets Manager.

    In production, the ECS task role or EC2 instance profile provides
    IAM credentials — no ACCESS_KEY/SECRET_KEY needed in the container.
    """
    client = boto3.client("secretsmanager", region_name=region)
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response["SecretString"])
    except ClientError:
        logger.error(
            "Failed to retrieve secret",
            extra={"secret_name": secret_name},
            exc_info=True,
        )
        raise
```

---

### Twelve-Factor App Methodology

The Twelve-Factor App (12factor.net) is a methodology for building production-grade web apps.
The relevant factors for our purposes:

**III. Config: Store config in the environment**
- Never commit config (database URLs, API keys, feature flags) to version control
- All config differences between dev/staging/production must come from environment variables

**IV. Backing services: Treat backing services as attached resources**
- Database, cache, queue — all accessed via URLs from config
- Swapping a local Postgres for a managed RDS requires only a URL change, no code change

**XI. Logs: Treat logs as event streams**
- Don't manage log files — write to stdout
- The platform (Railway, ECS, Kubernetes) captures stdout and routes it to your log aggregator

**XII. Admin processes: Run admin/management tasks as one-off processes**
- `alembic upgrade head` is a one-off process, not something that runs in the app lifespan

---

## Part 6 — Railway / Fly.io Deployment

### Railway

Railway is a platform-as-a-service that deploys from a Dockerfile or Nixpacks automatically.

```json
// railway.json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE",
    "dockerfilePath": "./Dockerfile"
  },
  "deploy": {
    "startCommand": "alembic upgrade head && gunicorn src.main:app -c gunicorn.conf.py",
    "healthcheckPath": "/health/live",
    "healthcheckTimeout": 15,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

Key Railway concepts:
- **Variables**: Secret values set in the Railway dashboard. Injected as environment variables.
  Never commit these to `railway.json`.
- **Private networking**: Services in the same Railway project can communicate over
  `service-name.railway.internal` without going through the public internet.
- **Volumes**: Persistent storage mounted at a path. Use for SQLite databases in staging
  (use managed Postgres in production).
- **Deploy triggers**: Railway auto-deploys on push to your configured branch.

```bash
# Deploy from CLI
railway up --service api

# Open a shell in a running service
railway run --service api bash

# View logs
railway logs --service api --follow

# Set a secret
railway variables set OPENAI_API_KEY=sk-...

# Run one-off migration command
railway run --service api alembic upgrade head
```

---

### Fly.io

```toml
# fly.toml
app = "my-api-production"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  # Non-secret config (safe to commit)
  ENVIRONMENT = "production"
  PORT = "8000"
  LOG_LEVEL = "INFO"

[http_service]
  internal_port = 8000
  force_https = true
  auto_stop_machines = true      # stop idle machines to save cost
  auto_start_machines = true     # restart when traffic arrives
  min_machines_running = 1       # always keep at least one running

  [[http_service.checks]]
    grace_period = "10s"
    interval = "15s"
    method = "GET"
    path = "/health/live"
    port = 8000
    timeout = "5s"

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1

[[mounts]]
  # Persistent volume for files (SQLite in staging, logs, etc.)
  source = "data"
  destination = "/data"
```

```bash
# Set secrets (never in fly.toml)
fly secrets set DATABASE_URL="postgresql+asyncpg://..."
fly secrets set SECRET_KEY="$(openssl rand -base64 32)"

# Deploy
fly deploy

# Run one-off migration
fly ssh console --command "alembic upgrade head"
# Or as a release command:
# [deploy] release_command = "alembic upgrade head"

# Scale horizontally
fly scale count 3

# View logs
fly logs --follow
```

---

### Rolling Deploys vs Blue-Green

**Rolling deploy**: replace pods/machines one at a time. During the rollout, both old and new
versions run simultaneously. Zero downtime if health checks are configured.

```
Before:  [v1] [v1] [v1] [v1]
Step 1:  [v2] [v1] [v1] [v1]  ← v2 health check passes, takes traffic
Step 2:  [v2] [v2] [v1] [v1]
Step 3:  [v2] [v2] [v2] [v1]
After:   [v2] [v2] [v2] [v2]
```

**Blue-green deploy**: run two identical environments. Switch traffic all at once. Rollback is
instant (just switch back).

```
Blue (current):  [v1] [v1] [v1]  ← takes 100% traffic
Green (new):     [v2] [v2] [v2]  ← tested, ready

Switch:
Blue:  [v1] [v1] [v1]  ← 0% traffic (standby for rollback)
Green: [v2] [v2] [v2]  ← 100% traffic
```

Blue-green requires 2× the infrastructure cost during the transition window. Rolling deploys are
the default for most platforms (Railway, Fly.io, Kubernetes).

---

## Part 7 — Observability in Production

### The Analogy

Observability is the airplane's black box — flight data recorder. You can't see inside the plane
while it's flying, but after something goes wrong, you have the data to reconstruct exactly what
happened: what the plane was doing, when, and why.

The three pillars:
- **Logs**: what happened (events)
- **Metrics**: how much / how fast (numbers over time)
- **Traces**: why it took so long (request flow across services)

---

### Structured Logging to stdout

In production, don't write logs to files. Write to stdout — the platform (Railway, ECS,
Kubernetes) captures it and routes it to your log aggregator (Datadog, CloudWatch, Railway logs).

```python
# src/logging_config.py
from __future__ import annotations

import logging
import sys

from pythonjsonlogger import jsonlogger


def configure_logging(log_level: str = "INFO", json_format: bool = True) -> None:
    """
    Configure structured JSON logging.

    In development (json_format=False), logs are human-readable.
    In production (json_format=True), logs are JSON for Datadog/CloudWatch parsing.
    """
    root_logger = logging.getLogger()
    root_logger.setLevel(log_level.upper())

    # Remove existing handlers
    root_logger.handlers.clear()

    handler = logging.StreamHandler(sys.stdout)

    if json_format:
        formatter = jsonlogger.JsonFormatter(
            fmt="%(asctime)s %(name)s %(levelname)s %(message)s",
            datefmt="%Y-%m-%dT%H:%M:%S",
            rename_fields={
                "asctime": "timestamp",
                "name": "logger",
                "levelname": "level",
            },
        )
    else:
        formatter = logging.Formatter(
            fmt="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
            datefmt="%H:%M:%S",
        )

    handler.setFormatter(formatter)
    root_logger.addHandler(handler)

    # Suppress noisy libraries
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
    logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)
```

Usage in application code:
```python
# Every module gets its own logger
logger = logging.getLogger(__name__)

# Always include contextual identifiers
logger.info(
    "User created post",
    extra={
        "user_id": user.id,
        "post_id": post.id,
        "tenant_id": request.state.tenant_id,
    },
)

# ERROR level includes stack trace
try:
    result = await external_api.call()
except Exception:
    logger.error(
        "External API call failed",
        extra={"api": "openai", "user_id": user.id},
        exc_info=True,   # attaches full stack trace to log entry
    )
```

---

### Correlation ID Middleware

When a request fails, you need to find all log entries for that specific request across your logs.
A correlation ID (also called request ID or trace ID) is a unique value attached to every log
entry from a single request.

```python
# src/middleware/correlation_id.py
from __future__ import annotations

import uuid
from contextvars import ContextVar

import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

# ContextVar is async-safe — unlike threading.local(), it's per-coroutine
request_id_var: ContextVar[str] = ContextVar("request_id", default="")


class CorrelationIDMiddleware(BaseHTTPMiddleware):
    """
    Adds a unique request ID to every request.

    Priority:
    1. Use X-Request-ID header if the client sent one (e.g., from a load balancer)
    2. Generate a new UUID4 if not present

    The request ID is:
    - Stored in a ContextVar (available to all async code in this request)
    - Added to the response as X-Request-ID header (for client-side correlation)
    """

    async def dispatch(self, request: Request, call_next) -> Response:  # type: ignore[no-untyped-def]
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        token = request_id_var.set(request_id)

        # Bind to structlog context so it appears in all log entries
        structlog.contextvars.bind_contextvars(request_id=request_id)

        try:
            response = await call_next(request)
            response.headers["X-Request-ID"] = request_id
            return response
        finally:
            request_id_var.reset(token)
            structlog.contextvars.clear_contextvars()
```

Register in FastAPI:
```python
from src.middleware.correlation_id import CorrelationIDMiddleware

app.add_middleware(CorrelationIDMiddleware)
```

---

### Sentry for Error Tracking

Sentry captures unhandled exceptions, shows the full stack trace, and groups them by error type.
Without Sentry, you discover errors when users complain. With Sentry, you see them in real time.

```python
# In lifespan startup (from Part 2):
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.redis import RedisIntegration
from sentry_sdk.integrations.logging import LoggingIntegration

sentry_sdk.init(
    dsn=settings.sentry_dsn.get_secret_value(),
    # Capture 10% of transactions as performance traces
    traces_sample_rate=0.1,
    # Profile 10% of sampled transactions
    profiles_sample_rate=0.1,
    environment=settings.environment,
    release=settings.app_version,
    integrations=[
        FastApiIntegration(transaction_style="url"),
        SqlalchemyIntegration(),
        RedisIntegration(),
        LoggingIntegration(
            level=logging.WARNING,       # capture WARNING+ as breadcrumbs
            event_level=logging.ERROR,   # send ERROR+ as events
        ),
    ],
    # Never send PII to Sentry
    send_default_pii=False,
)
```

---

### Prometheus Metrics

Prometheus scrapes your `/metrics` endpoint periodically and stores time-series data. Grafana
reads from Prometheus to build dashboards.

```python
# src/metrics.py
from __future__ import annotations

from prometheus_client import Counter, Histogram, Gauge

# Track total requests by endpoint and status code
http_requests_total = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

# Track request duration distribution
http_request_duration_seconds = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# Track current database pool connections
db_pool_connections = Gauge(
    "db_pool_connections",
    "Database connection pool size",
    ["state"],  # checked_in, checked_out, overflow
)
```

The easiest way: use the `prometheus_fastapi_instrumentator` package:

```python
# src/main.py
from prometheus_fastapi_instrumentator import Instrumentator

# After app creation:
Instrumentator(
    should_group_status_codes=True,
    should_ignore_untemplated=True,
    should_group_untemplated=True,
    should_round_latency_decimals=True,
    should_respect_env_var=True,        # disable in tests
    env_var_name="ENABLE_METRICS",
    excluded_handlers=[
        "/health/live",
        "/health/ready",
        "/metrics",
    ],
).instrument(app).expose(app, endpoint="/metrics")
```

---

## Part 8 — CI/CD Pipeline

### The Analogy

CI/CD is the factory assembly line for your software. Code enters at one end (a git push), passes
through quality checks (tests, lint, security scans), gets assembled into a deployable artifact
(Docker image), and rolls out to production at the other end — automatically, reproducibly, every
time.

---

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Job 1: Test ──────────────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "0.5.14"

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run migrations
        env:
          DATABASE_URL: postgresql+asyncpg://testuser:testpassword@localhost:5432/testdb
        run: uv run alembic upgrade head

      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://testuser:testpassword@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379/0
          SECRET_KEY: test-secret-key-for-ci-only
          ENVIRONMENT: development
        run: uv run pytest -x --cov=src --cov-report=xml -q

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.xml

  # ── Job 2: Lint ───────────────────────────────────────────────────────────
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v4
        with:
          version: "0.5.14"

      - name: Install dev dependencies
        run: uv sync --frozen --group dev

      - name: Black (format check)
        run: uv run black --check src/ tests/

      - name: Ruff (lint)
        run: uv run ruff check src/ tests/

      - name: Mypy (type check)
        run: uv run mypy src/ --ignore-missing-imports

      - name: pip-audit (vulnerability check)
        run: uv run pip-audit

  # ── Job 3: Build and Push Image ───────────────────────────────────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, lint]          # only build if tests and lint pass
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    outputs:
      image-digest: ${{ steps.push.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # Cache layers between builds — much faster CI
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_SHA=${{ github.sha }}

  # ── Job 4: Deploy ─────────────────────────────────────────────────────────
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    environment:
      name: production
      url: https://api.myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Railway
        uses: railway/deploy-action@v1
        with:
          service: api
          token: ${{ secrets.RAILWAY_TOKEN }}
```

---

### Canary Deployment Pattern

A canary deploy sends a small percentage of traffic (5–10%) to the new version before rolling it
out completely. Named after the canary in a coal mine — if something's wrong, only a small portion
of users are affected.

```bash
# Kubernetes: deploy canary with 10% of pods running new version
# Assuming 10 total replicas in production

# Deploy 1 replica with new version (10% of traffic)
kubectl set image deployment/api-canary api=ghcr.io/myorg/myapp:v2.0.0

# Monitor error rates and latency for 10 minutes
# If metrics look good, proceed with full rollout
kubectl set image deployment/api api=ghcr.io/myorg/myapp:v2.0.0

# If metrics look bad, rollback canary
kubectl set image deployment/api-canary api=ghcr.io/myorg/myapp:v1.9.0
```

On Railway/Fly.io, canary deploys are implemented with feature flags rather than traffic splitting.

---

### Feature Flags for Gradual Rollout

```python
# src/feature_flags.py
from __future__ import annotations

import logging
import os

logger = logging.getLogger(__name__)


class FeatureFlags:
    """
    Simple feature flag implementation backed by environment variables.

    In production, use a proper feature flag service (LaunchDarkly, Flagsmith,
    Unleash) for gradual rollout by user segment.
    """

    def __init__(self) -> None:
        self._flags: dict[str, bool] = {}

    def is_enabled(self, flag_name: str) -> bool:
        """Check if a feature flag is enabled."""
        if flag_name not in self._flags:
            # Read from environment on first access
            env_var = f"FEATURE_{flag_name.upper()}"
            self._flags[flag_name] = os.environ.get(env_var, "false").lower() == "true"
        return self._flags[flag_name]


feature_flags = FeatureFlags()

# Usage:
# if feature_flags.is_enabled("new_search_algorithm"):
#     results = await new_search(query)
# else:
#     results = await old_search(query)
```

---

### Rollback Procedure

```bash
# Railway rollback
railway rollback --service api

# Fly.io rollback to previous version
fly releases list                          # list recent deployments
fly deploy --image registry.fly.io/myapp:v1.9.0@sha256:abc123

# Kubernetes rollback
kubectl rollout undo deployment/api
kubectl rollout status deployment/api      # watch rollout progress

# Rollback to specific version
kubectl rollout history deployment/api     # see revision history
kubectl rollout undo deployment/api --to-revision=3
```

---

## Appendix: Key Files Summary

### Project Structure

```
myproject/
├── src/
│   ├── __init__.py
│   ├── main.py              # FastAPI app, lifespan
│   ├── config.py            # Settings, get_settings()
│   ├── database.py          # Engine, SessionLocal
│   ├── state.py             # startup_complete Event
│   ├── api/
│   │   ├── health.py        # /health/live, /ready, /startup
│   │   └── ...
│   ├── middleware/
│   │   └── correlation_id.py
│   ├── models/
│   ├── schemas/
│   └── services/
├── alembic/
│   ├── env.py
│   └── versions/
├── tests/
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── gunicorn.conf.py
├── alembic.ini
├── pyproject.toml
├── uv.lock
├── railway.json
└── .github/
    └── workflows/
        └── deploy.yml
```

### Quick Reference: Common Commands

```bash
# Local development
docker compose up -d                        # start all services
docker compose logs -f api                  # follow app logs
docker compose exec api alembic upgrade head  # run migrations

# Build and test locally
docker build -t myapp:local .
docker run --rm -e DATABASE_URL=... myapp:local python -m pytest

# Check image size
docker images myapp:local
docker history myapp:local                  # see each layer's size

# Inspect running container
docker exec -it container_name bash        # interactive shell (if bash installed)
docker exec -it container_name /bin/sh     # alpine uses sh, not bash

# Railway
railway up                                  # deploy current directory
railway run python -c "from src.config import get_settings; print(get_settings())"

# Database operations
alembic revision --autogenerate -m "add users table"
alembic upgrade head
alembic downgrade -1                        # one step back
alembic history --verbose                   # show migration history
alembic current                             # show current revision in DB
```

---

*End of Level 15: Production Deployment*

---

> Next: Level 16 — Monitoring and Alerting (Prometheus, Grafana, PagerDuty alerting rules)
