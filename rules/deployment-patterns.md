---
description: Production deployment rules — Docker, health checks, Gunicorn, migrations, config, CI/CD.
paths: ["Dockerfile", "docker-compose*.yml", "*.py", "*.toml", "*.yml", "gunicorn.conf.py"]
---

# Production Deployment Patterns

> This file owns: Docker multi-stage builds, health checks, process management (Gunicorn),
>   zero-downtime migrations, configuration management, CI/CD pipeline, observability,
>   graceful shutdown.
> Security rules → security.md / security-complete.md
> Performance rules → performance-patterns.md
> Database rules → database-patterns.md

---

## Docker Multi-Stage Builds

1. ALWAYS use a multi-stage Dockerfile with at least two stages: `builder` (install deps with all build tools) and `production` (copy compiled artifacts only) — a single-stage build from `python:3.12` produces 800 MB+ images; multi-stage targets under 200 MB.

2. ALWAYS copy dependency manifests (`pyproject.toml`, `uv.lock` or `requirements.txt`) before copying application source code — this ensures the dependency installation layer is cached independently and is not invalidated on every code change.

3. NEVER install build tools (`gcc`, `build-essential`, `git`, `pip`, `uv`) in the production stage — they increase image size, expand the attack surface, and allow an attacker with code execution to download and compile additional tools.

4. ALWAYS run the final container process as a non-root user — create the user with `useradd --uid 1000 --no-create-home --shell /sbin/nologin appuser` and switch with `USER appuser` before the `CMD` directive.

5. ALWAYS set `COPY --chown=appuser:appuser` when copying application files in the production stage — files owned by root cannot be read or modified by the non-root user at runtime.

6. ALWAYS set `ENV PYTHONUNBUFFERED=1` in production images — without it, Python buffers stdout/stderr and log lines are held in memory until the buffer flushes, making real-time log streaming unreliable.

7. ALWAYS set `ENV PYTHONDONTWRITEBYTECODE=1` in both builder and production stages — `.pyc` files add unnecessary image size and are not needed when the container's filesystem is read-only.

8. NEVER put secrets (`API keys`, passwords, tokens, `DATABASE_URL` with credentials) in `ARG` or `ENV` directives — they appear in `docker history` output and `docker inspect` and persist in every layer of the image. Secrets belong in environment variables injected at container startup by your platform.

9. ALWAYS include a `.dockerignore` file that excludes at minimum: `.git/`, `.env*`, `__pycache__/`, `*.pyc`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `tests/`, `docs/`, `.venv/`, and `node_modules/` — without `.dockerignore`, `COPY . .` sends the full working tree to the daemon, inflating build context size and potentially leaking secrets.

10. ALWAYS install `uv` or `pip` in the builder stage using a pinned version — use `COPY --from=ghcr.io/astral-sh/uv:0.5.14 /uv /usr/local/bin/uv` for uv to avoid a pip dependency; pin versions to ensure reproducible builds.

11. ALWAYS use `RUN uv sync --frozen --no-dev` (or `pip install --no-cache-dir --require-hashes`) in the builder stage — `--frozen` ensures the lock file is the source of truth; `--no-dev` excludes test dependencies from the production image.

12. ALWAYS install a `HEALTHCHECK` instruction in the production Dockerfile specifying at minimum `--interval`, `--timeout`, `--start-period`, and `--retries` — Railway and other platforms use this to gate traffic routing.

13. DO use `ARG BUILD_DATE` and `ARG GIT_SHA` with `LABEL` to embed build metadata (commit SHA, build timestamp) into the image — this enables you to determine exactly which commit is running in production by inspecting the image.

14. NEVER use `latest` as the base image tag in a production Dockerfile — `FROM python:3.12-slim` is acceptable; prefer `FROM python:3.12.3-slim` (exact patch version) or a digest-pinned reference to guarantee reproducible builds and prevent silent security regressions from upstream tag updates.

15. DO configure Docker Compose for local development with bind mounts (`./src:/app/src`) and `--reload` for hot-reloading, while production images are baked (no bind mounts) — never use bind mounts in production as they couple the container to the host filesystem.

---

## Health Checks

16. ALWAYS implement three distinct health endpoints: `/health/live` (liveness), `/health/ready` (readiness), and `/health/startup` (startup) — using a single `/health` endpoint for all three purposes conflates probes with different failure semantics and different recovery actions.

17. ALWAYS return HTTP 200 from `/health/live` with zero latency and zero external calls — liveness only answers "is this process alive and not deadlocked?"; if it calls the database and the database is down, Kubernetes will restart a perfectly healthy pod unnecessarily.

18. ALWAYS check the database connection pool from `/health/ready` by executing a trivial query (`SELECT 1`) inside the connection pool's `connect()` context — a readiness failure removes the pod from the load balancer rotation without restarting it.

19. NEVER check third-party APIs (Stripe, SendGrid, OpenAI, external payment processors) in `/health/ready` — if a third-party API is down, your pod should not be removed from rotation; your application should degrade gracefully, not stop serving traffic.

20. ALWAYS return HTTP 503 with a JSON body listing which checks failed from `/health/ready` when any check fails — the body must contain `{"status": "not ready", "failed": ["database"], "checks": {"database": "fail", "redis": "ok"}}` so that automated systems and humans can diagnose the cause without reading logs.

21. ALWAYS use `/health/startup` for containers that require significant initialization time (loading ML models, warming caches, running initial data validation) — configure the startup probe with a higher `failureThreshold × periodSeconds` total than the maximum expected startup time.

22. NEVER mark startup as complete in the startup probe until all mandatory initialization tasks have finished — if the startup probe passes before the database connection pool is warmed, readiness checks may pass while the application is still half-initialized.

23. ALWAYS set `initialDelaySeconds` on Kubernetes liveness and readiness probes — without it, Kubernetes probes the container immediately after it starts, before the application server is listening, producing spurious failures that restart or exclude healthy pods.

24. ALWAYS configure the HEALTHCHECK Dockerfile instruction with `--start-period` equal to or greater than your expected worst-case startup time — Railway and Fly.io use this; a `start-period` that is too short causes health check failures to count during legitimate startup.

25. NEVER make the health check HTTP client perform DNS resolution or TCP connection establishment on every check — use `urllib.request` (stdlib, no dependency) or a pre-established connection; a health check that opens a new TCP connection on every probe adds unnecessary overhead at scale.

26. ALWAYS protect the `/health/ready` endpoint from returning stale data by running each check with a timeout — wrap each database/Redis check in `asyncio.wait_for(..., timeout=3.0)` and treat a timeout as a failure, not a pass.

27. ALWAYS return the same content type and schema on both pass and fail responses from health endpoints — load balancers and monitoring systems parse the response body; an inconsistent schema breaks automated alerting.

28. DO include a version field in health responses (`{"status": "ready", "version": "1.2.3", "checks": {...}}`) — this confirms which version is serving traffic without accessing deployment logs.

---

## Process Management

29. ALWAYS use Gunicorn with `UvicornWorker` in production for ASGI applications — never use `uvicorn` directly in production; Gunicorn provides worker auto-restart on crash, graceful SIGHUP reload, and a managed process group.

30. ALWAYS calculate worker count as `workers = (2 × cpu_count) + 1` — this formula leaves headroom for one worker handling a slow blocking operation while the rest serve requests; do not use more workers than this formula produces without load-testing to confirm improvement.

31. ALWAYS calculate connection pool size per worker as `pool_size = floor((max_db_connections × 0.8) / num_workers)` — each Gunicorn worker is a separate OS process with its own pool; 20% of connections must be reserved for migrations, monitoring tools, and admin access.

32. ALWAYS set `timeout = 120` in `gunicorn.conf.py` — this hard-kills a worker if it fails to respond for 120 seconds; lower values cause spurious kills for legitimately slow operations (LLM calls, large file processing); raise it only with a corresponding increase in the Kubernetes `terminationGracePeriodSeconds`.

33. ALWAYS set `graceful_timeout` to a value less than `terminationGracePeriodSeconds` in Kubernetes — if `graceful_timeout = 20` and `terminationGracePeriodSeconds = 30`, workers get 20 seconds to finish in-flight requests before Gunicorn kills them, and Kubernetes gives a total of 30 seconds before sending SIGKILL.

34. ALWAYS set `preload_app = True` in `gunicorn.conf.py` when the application uses significant memory at import time — preloading loads the application before forking workers so workers share memory pages via copy-on-write, reducing total RSS by 50–70 MB per worker.

35. NEVER open database connections or instantiate SQLAlchemy engines at module import level when `preload_app = True` — connections opened before the fork are shared across workers, causing race conditions; use `post_fork` hook or lazy initialization to open connections after forking.

36. ALWAYS use a `gunicorn.conf.py` file rather than passing all settings as CLI flags — a config file is version-controlled, reviewable, and prevents the `CMD` in the Dockerfile from becoming an unreadable wall of flags.

37. ALWAYS expose the Gunicorn PID file with `pidfile = "/tmp/gunicorn.pid"` — this enables zero-downtime reloads via `kill -HUP $(cat /tmp/gunicorn.pid)` which spawns new workers with new code before killing old ones.

38. NEVER set `worker_class = "sync"` (the default) for ASGI applications — the sync worker cannot handle `async def` route handlers and will silently block the process; always set `worker_class = "uvicorn.workers.UvicornWorker"`.

39. DO set `keepalive = 5` — this keeps HTTP connections alive for 5 seconds, reducing TCP handshake overhead for clients making multiple requests; values above 75 seconds can tie up worker slots.

40. ALWAYS configure Gunicorn logging to write to stdout and stderr (`accesslog = "-"`, `errorlog = "-"`) — the platform captures these streams for log aggregation; never configure Gunicorn to write to log files inside the container.

---

## Zero-Downtime Migrations

41. NEVER run `alembic upgrade head` inside the FastAPI lifespan startup hook — with multiple workers or replicas, all instances attempt to run the same migration simultaneously, causing race conditions and failed deployments; run migrations as a one-off process before the application starts.

42. ALWAYS run migrations as a separate step before starting the application — in Railway, use `startCommand: "alembic upgrade head && gunicorn ..."`, or in Kubernetes use an init container that runs migrations to completion before the main container starts.

43. ALWAYS use the expand/migrate/contract pattern for schema changes that affect running application instances — Phase 1 (expand): add new column/table without removing old; Phase 2 (migrate): backfill data; Phase 3 (contract): remove old column/table after all instances use the new schema.

44. NEVER rename a column in a single migration — a rename is invisible to old application instances that still reference the old name; instead, add the new column, dual-write, backfill, switch reads, then drop the old column across 3–4 separate deploys.

45. ALWAYS add new columns as `nullable=True` in expand migrations — if a column is `NOT NULL` without a `DEFAULT`, inserting a row (which old application code will do) will fail until all rows are backfilled; nullable columns allow old and new code to coexist.

46. ALWAYS use `CREATE INDEX CONCURRENTLY` for index creation in production migrations — a standard `CREATE INDEX` takes an exclusive write lock that blocks all DML for the entire index build duration; `CONCURRENTLY` builds without blocking at the cost of a longer build time.

47. NEVER run `CREATE INDEX CONCURRENTLY` inside a transaction — PostgreSQL explicitly rejects it with an error; the migration must end the auto-started transaction before issuing the command: `op.execute("COMMIT")` before the `CREATE INDEX CONCURRENTLY` statement.

48. ALWAYS check for `INVALID` indexes after a migration run that includes `CREATE INDEX CONCURRENTLY` — a failed concurrent build leaves an invalid index that is invisible to queries but occupies space and can prevent re-creation; query `pg_indexes` joined to `pg_index WHERE indisvalid = false` before marking a migration successful.

49. NEVER use `LOCK TABLE` or schema changes that require table rewrites (type changes on large tables, adding NOT NULL without a DEFAULT) in a migration that runs against a live production database — these operations hold exclusive locks and cause downtime equal to the operation duration.

50. ALWAYS backfill large tables in batches using `WHERE id IN (SELECT id FROM table WHERE new_col IS NULL ORDER BY id LIMIT :batch_size)` — a single `UPDATE table SET ...` on a large table holds a row-level lock on every matching row for the duration and can cause lock contention.

51. ALWAYS test migrations against a production-sized data copy in staging before applying to production — a migration that runs in 2 seconds on 1,000 rows may take 45 minutes on 50 million rows; time the migration before the production window.

52. ALWAYS keep the previous migration version deployable for at least one full deploy cycle after the contract phase — if a rollback is needed, the database must be in a state that both old and new application code can use.

---

## Configuration and Secrets

53. ALWAYS use `pydantic-settings` `BaseSettings` as the single source of configuration — type validation, `.env` file loading, and required-field enforcement come for free; raw `os.environ.get()` scattered through the codebase has no type safety and no centralized documentation.

54. ALWAYS declare required configuration fields with `Field(...)` (no default) — `pydantic_settings` raises a `ValidationError` at startup if a required variable is missing, making misconfiguration a loud startup failure rather than a silent runtime error.

55. ALWAYS use `model_config = SettingsConfigDict(extra="forbid")` — this rejects unknown environment variables with a `ValidationError` and prevents typos in variable names from silently being ignored (e.g., `DATABSE_URL` being set but never read).

56. ALWAYS decorate `get_settings()` with `@lru_cache` — settings are read from environment variables once and cached for the life of the process; without caching, every call reads and re-parses all environment variables, adding latency to every request.

57. ALWAYS use `pydantic.SecretStr` for fields that hold credentials (`secret_key`, `database_url` with password, `api_key_*`) — `SecretStr` renders as `**********` in `str()`, `repr()`, logging, and exception messages, preventing accidental credential exposure in logs.

58. NEVER call `.get_secret_value()` at module level — unwrap `SecretStr` only inside the function that needs the raw value; unwrapping at import time stores the secret as a plain string in a module-level variable, where it can be logged.

59. ALWAYS write a `@field_validator` that raises `ValueError` if `environment == "production"` and `debug == True` — debug mode in production enables the Starlette/Werkzeug interactive debugger, which allows arbitrary code execution via the browser console.

60. NEVER store secrets in `Dockerfile` `ENV` or `ARG` directives, `docker-compose.yml` `environment` blocks, `railway.json`, `fly.toml`, or any file committed to version control — secrets committed to any of these locations appear in image history, CI logs, and git history permanently.

61. ALWAYS use a `@field_validator` to normalize the `DATABASE_URL` scheme from `postgres://` to `postgresql+asyncpg://` — Railway, Heroku, and Render export the URL with the `postgres://` scheme; SQLAlchemy async requires `postgresql+asyncpg://`; silent scheme mismatch causes a runtime crash on first database query.

62. DO use `env_nested_delimiter="__"` in `SettingsConfigDict` and nested `BaseSettings` subclasses for complex configuration — `DATABASE__POOL_SIZE=20` maps to `settings.database.pool_size = 20`; this avoids a flat namespace of 50+ environment variables.

---

## CI/CD Pipeline

63. ALWAYS run the full test suite, lint, and type check as a prerequisite gate before the Docker build step — building an image for code that fails tests wastes build minutes and registry storage; fail fast before the expensive step.

64. ALWAYS run database migrations in CI against a real database service (not a mock or SQLite) — Alembic migrations contain raw SQL and PostgreSQL-specific features (`CREATE INDEX CONCURRENTLY`, custom types, triggers) that SQLite cannot validate.

65. ALWAYS pin all action versions in GitHub Actions with a full commit SHA (`uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`) — using `@v4` (a mutable tag) allows upstream compromises to inject malicious code into your CI pipeline via a tag update.

66. ALWAYS use `cache-from: type=gha` and `cache-to: type=gha,mode=max` in `docker/build-push-action` — GitHub Actions Docker layer cache reduces build time from 3–5 minutes to under 60 seconds for code-only changes where the dependency layer is cached.

67. ALWAYS tag production images with the git SHA (`type=sha,prefix=sha-`) in addition to the branch name — branch tags are mutable (a new push moves `main` to a new commit); SHA tags are immutable and uniquely identify the deployed commit.

68. ALWAYS run `pip-audit` or `safety check` in CI as a mandatory step that fails the pipeline — a passing test suite does not mean safe dependencies; automated vulnerability scanning catches known CVEs in transitive dependencies.

69. NEVER deploy without first running migrations in a separate step — always structure the deploy command as `migrate-then-start` (Railway `startCommand`, Kubernetes init container, Fly.io `release_command`); a new application version that runs against an un-migrated schema causes runtime errors.

70. ALWAYS use GitHub Actions `environment:` blocks with required reviewers for production deployments — this adds a manual approval gate so that a `git push` to `main` triggers tests and staging deploys automatically, but production requires explicit human sign-off.

71. ALWAYS store the Docker registry credentials in CI secrets, not in the workflow YAML — use `secrets.GITHUB_TOKEN` for GHCR (GitHub Container Registry) which is automatically available; never hardcode registry passwords.

72. DO implement a rollback step in your CI/CD pipeline that can be triggered manually from GitHub Actions UI — a rollback should re-deploy the previous image SHA without rebuilding; implement as a separate workflow that accepts the target SHA as an input.

---

## Observability in Production

73. ALWAYS write logs to stdout, never to files — containers are ephemeral; files inside a container are lost when it restarts, and log rotation inside a container conflicts with the platform's log capture; stdout is captured automatically by Docker, Kubernetes, Railway, and Fly.io.

74. ALWAYS use structured JSON logging in production — log aggregators (Datadog, CloudWatch, Railway Logs, Papertrail) parse JSON fields for filtering and alerting; a human-readable log line `2024-01-01 ERROR user not found` cannot be queried by `user_id = 42`.

75. ALWAYS attach a `request_id` (correlation ID) to every log entry via a `ContextVar` set in middleware — without a correlation ID, tracing a single failed request through 50 log lines requires matching on timestamp, which breaks under concurrent load.

76. ALWAYS generate the `request_id` in middleware from the incoming `X-Request-ID` header if present, or from `uuid.uuid4()` if absent — reflecting the client's request ID enables end-to-end tracing from browser to backend to database.

77. ALWAYS return the `X-Request-ID` header in every HTTP response — clients and load balancers log this header; it enables support teams to correlate a user-reported error with the exact backend log entries.

78. ALWAYS initialize Sentry in the application lifespan startup hook before any other initialization — errors during startup (database connection failures, misconfiguration) must be captured; initializing Sentry after these steps means startup errors are not reported.

79. ALWAYS set `send_default_pii=False` in `sentry_sdk.init()` — by default Sentry captures IP addresses and user agent strings; disabling PII collection is required for GDPR compliance and prevents accidental PII capture in exception context.

80. ALWAYS expose a `/metrics` endpoint in the Prometheus text format for all production services — Prometheus scrapes this endpoint; without it, you have no CPU, memory, request rate, error rate, or latency data in Grafana.

81. ALWAYS call `sentry_sdk.flush(timeout=5.0)` in the lifespan shutdown handler — Sentry's SDK batches events asynchronously; without an explicit flush, the last batch of errors before a shutdown (e.g., a crash) is silently dropped.

82. NEVER log secrets, passwords, tokens, API keys, JWT payloads, or Authorization header values at any log level — even `DEBUG`-level credential logging creates a path for secrets to appear in log aggregators where they are stored for weeks or months.

---

## Graceful Shutdown

83. ALWAYS implement the graceful shutdown sequence in the FastAPI lifespan context manager's post-yield block: (1) stop accepting connections (handled by Gunicorn), (2) wait for in-flight requests (handled by `graceful_timeout`), (3) dispose the SQLAlchemy engine, (4) close Redis connections, (5) flush Sentry — in this exact order.

84. ALWAYS dispose the SQLAlchemy async engine with `await engine.dispose()` in the shutdown block — this cleanly closes all connections in the pool and sends the TCP FIN to the database server; a missing dispose causes "too many connections" errors after repeated deploys.

85. ALWAYS close async Redis clients with `await redis_client.aclose()` in the shutdown block — open connections to Redis at process exit are treated as errors by Redis Cluster and trigger reconnect storms in the remaining nodes.

86. NEVER use `atexit` handlers or `signal.signal()` for cleanup in production ASGI applications — Gunicorn manages SIGTERM delivery to workers and calls the ASGI lifespan shutdown; competing signal handlers cause race conditions and double-disposal of resources.

87. ALWAYS set `terminationGracePeriodSeconds` in Kubernetes to at least `graceful_timeout + 10` seconds — the buffer accounts for time between SIGTERM delivery and Gunicorn beginning its graceful shutdown; if Kubernetes sends SIGKILL before Gunicorn finishes draining, in-flight requests are dropped.

88. NEVER perform slow operations (database queries, external API calls, file I/O) in the lifespan shutdown block beyond what is required to cleanly close connections — the shutdown block runs while Kubernetes is waiting; every second of shutdown delay counts against `terminationGracePeriodSeconds`.

89. ALWAYS use `asyncio.Event` (not a boolean flag) to signal startup completion between the lifespan startup block and the startup health probe — `asyncio.Event.is_set()` is thread-safe and coroutine-safe; a plain boolean is not safe under concurrent access from multiple coroutines.

90. DO log a structured startup-complete and shutdown-complete message at INFO level with the application version, environment, and timestamp — these log lines serve as deployment audit entries and allow you to confirm exactly when a new version started serving traffic and when the old version stopped.

---

## Docker/Container Hardening

91. ALWAYS set memory and CPU limits in `docker-compose.yml` and Kubernetes pod specs — in Compose use `deploy.resources.limits.memory` and `deploy.resources.limits.cpus`; in Kubernetes use `resources.limits.memory` and `resources.limits.cpu`; a container without limits can exhaust host memory and trigger the OOM killer on unrelated processes.

92. ALWAYS set `resources.requests` equal to or less than `resources.limits` in Kubernetes — requests are used by the scheduler for placement; a pod that requests 256 MiB but limits at 2 GiB may be scheduled onto a node that cannot actually serve the limit, causing OOM kills under load.

93. ALWAYS add `ulimits: nofile: soft: 65536, hard: 65536` in Docker Compose for services that hold many open file descriptors — the Linux default of 1024 open files causes "too many open files" errors under moderate concurrency long before any application-level limit is hit.

94. ALWAYS use `--mount=type=secret,id=pip_token,dst=/run/secrets/pip_token` (BuildKit secret mounts) when the build step requires credentials (private PyPI index, private Git repo) — secret mounts are never written to the image layer and do not appear in `docker history`; `ARG` and `ENV` do appear in image history and are permanently readable.

95. ALWAYS mount `/tmp` as a `tmpfs` volume for containers that write temporary files — prevents temporary file accumulation on the container overlay filesystem and limits disk I/O impact on the host.

96. ALWAYS run containers with a read-only root filesystem (`read_only: true` in Compose, `readOnlyRootFilesystem: true` in Kubernetes `securityContext`) and explicitly mount writable volumes only for paths that require writes — a read-only rootfs prevents an attacker with code execution from modifying binaries or writing persistence mechanisms.

97. ALWAYS add `cap_drop: [ALL]` and add back only required capabilities in Docker Compose and Kubernetes `securityContext.capabilities` — by default containers inherit a broad set of Linux capabilities (including `CAP_CHOWN`, `CAP_DAC_OVERRIDE`, `CAP_SETUID`); dropping all prevents privilege escalation via capability abuse.

98. ALWAYS apply a seccomp profile to production containers — use Docker's default seccomp profile or the Kubernetes `RuntimeDefault` profile; it blocks system calls that have no legitimate use in a Python application server (e.g., `ptrace`, `mount`, `kexec_load`).

99. ALWAYS run `trivy image` or `grype` against every built image in CI as a gate step that fails on CRITICAL or HIGH severity CVEs — image scanning catches vulnerable OS packages (e.g., an unpatched `libssl` in the base image) that are not detected by `pip-audit`.

100. ALWAYS build multi-arch images with `docker buildx build --platform linux/amd64,linux/arm64` when deploying to ARM-based infrastructure — a single-arch `linux/amd64` image runs under QEMU emulation on ARM, adding 2-3x CPU overhead.

101. NEVER use `docker login` with credentials hardcoded in CI workflow YAML — use OIDC-based keyless authentication for ECR/GCR or `secrets.GITHUB_TOKEN` for GHCR.

---

## Health and Reliability

102. ALWAYS use `depends_on: db: condition: service_healthy` in Docker Compose — the plain `depends_on: [db]` starts the dependent service as soon as the dependency container process starts, not when the database is accepting connections; use `service_healthy` to wait until the dependency's healthcheck passes.

103. ALWAYS expose circuit breaker state on the `/health/ready` endpoint — include circuit breaker states in the response; when a circuit is open, mark the service as degraded (200 with `status: degraded`) rather than removing it from rotation entirely, unless the open circuit is a mandatory dependency.

104. ALWAYS check available connection pool slots (not just a ping) from `/health/ready` — a fully saturated pool accepts a SELECT 1 query but the application cannot acquire connections for real requests; pool saturation is a different failure mode than database unavailability.

105. ALWAYS define a `PodDisruptionBudget` for every Deployment with more than one replica — set `minAvailable: 1` to prevent cluster operations (node drain, upgrades) from terminating all replicas simultaneously.

106. ALWAYS set `maxUnavailable: 0` and `maxSurge: 1` in the Kubernetes `RollingUpdateStrategy` for stateless API services — `maxUnavailable: 0` ensures no pod is removed from the load balancer before a replacement is healthy; the default `maxUnavailable: 25%` risks downtime on small deployments with 2-4 replicas.

---

## Process and Runtime

107. ALWAYS set `--limit-concurrency` on Uvicorn workers when running behind a reverse proxy — without a concurrency limit, a burst of slow requests can fill all worker connection queues causing memory exhaustion.

108. ALWAYS set `max_requests` and `max_requests_jitter` in `gunicorn.conf.py` — `max_requests = 1000` and `max_requests_jitter = 100` causes each worker to restart after handling 900–1100 requests, preventing slow memory leaks from accumulating indefinitely.

109. ALWAYS set `worker_tmp_dir = "/dev/shm"` in `gunicorn.conf.py` when using `preload_app = False` — Gunicorn uses worker heartbeat files to detect dead workers; writing these to `/dev/shm` (RAM-backed tmpfs) eliminates I/O latency on the heartbeat check and prevents false-positive worker kills under heavy disk I/O.

110. ALWAYS configure the default thread pool size for sync endpoints explicitly — the default asyncio thread pool is `min(32, os.cpu_count() + 4)`, which can be exhausted by blocking I/O under moderate concurrency, stalling the entire event loop.

111. ALWAYS set `loop = "uvloop"` in `gunicorn.conf.py` or pass `--loop uvloop` to the Uvicorn worker — uvloop is a C implementation of the asyncio event loop delivering 2-4x throughput improvement for I/O-bound workloads.

---

## Migration and Database

112. ALWAYS set `lock_timeout` and `statement_timeout` for migration sessions before running Alembic — `SET lock_timeout = '10s'; SET statement_timeout = '300s';`; without `lock_timeout`, a migration waiting for a lock held by a long-running query blocks all subsequent schema operations indefinitely.

113. ALWAYS use a synchronous SQLAlchemy engine for Alembic migrations even when the application uses an async engine — Alembic's migration context does not support `asyncpg` natively; create a sync engine in `env.py` by replacing `postgresql+asyncpg://` with `postgresql+psycopg2://`.

114. ALWAYS run `alembic upgrade --sql head` in staging before executing the migration in production — the `--sql` flag prints the raw SQL without running it; review for lock-acquiring operations that are safe on small tables but catastrophic on large ones.

115. ALWAYS take a `pg_dump` backup before running any migration that alters existing table structures — `pg_dump -Fc -d $DATABASE_URL -f pre_migration_$(date +%Y%m%d_%H%M%S).dump`; if the migration fails mid-execution, the only guaranteed recovery path is restore from backup.

---

## Observability and CI

116. ALWAYS set `OTEL_SERVICE_NAME` and `OTEL_SERVICE_VERSION` as environment variables on every production container — `OTEL_SERVICE_NAME` is the primary grouping dimension in every OTel backend; `OTEL_SERVICE_VERSION` maps traces to the exact git SHA deployed.

117. NEVER use high-cardinality values (user_id, request_id, session_id, IP address) as Prometheus label values — each unique label combination creates a new time series; 100,000 unique user_ids causes Prometheus memory exhaustion and query timeouts.

118. ALWAYS implement a dead-man's switch alert — configure an alert that fires if no log lines or metric scrapes are received from a service for more than 5 minutes; this catches total service silence (process crash, OOM kill, network partition) that would otherwise produce no alert at all.

119. ALWAYS annotate deployments in your metrics platform at deploy time — send a deployment event with the git SHA to Grafana, Datadog, or equivalent; deploy annotations make it visually obvious which metric changes correlate with a new deploy.

120. ALWAYS run smoke tests immediately after a production deploy, before removing the old replica set — smoke tests make authenticated API calls through the full stack and fail the deploy if any call returns an unexpected status; a passing `/health/ready` only confirms the process is running, not that application logic is correct.

121. ALWAYS define explicit rollback trigger conditions before a deploy — document conditions that automatically trigger a rollback (error rate > 1% for 5 min, P99 latency > 2x baseline, smoke test failure); a verbal agreement to "roll back if things look bad" is not a rollback plan.

122. ALWAYS run secret scanning (`gitleaks detect` or `trufflehog git`) in CI as a blocking step — run against the full commit diff; secret scanners in deployment pipelines catch secrets committed to migration files, config templates, and seed data scripts.

123. ALWAYS generate an SBOM as a CI artifact on every production build using `syft packages . -o cyclonedx-json > sbom.json` — when a new CVE is disclosed, an SBOM lets you determine in seconds whether the affected package is present without scanning every deployed image.

124. ALWAYS implement log sampling for high-volume, low-value endpoints — access logs for `/health/live`, `/metrics`, and static asset endpoints can constitute 80% of log volume; emit at most 1 log per 100 requests for these endpoints.

---

## Container Security Hardening

125. ALWAYS consider distroless base images (`gcr.io/distroless/python3-debian12`) for production stages — distroless images contain only the Python runtime and CA certificates, with no shell, no package manager, no `wget`; an attacker with RCE cannot download tools or pivot; image size is 30-50% smaller than `python:3.12-slim`.

126. NEVER use a base image with a shell in the production stage unless explicitly required for the entrypoint — override the entrypoint with `ENTRYPOINT ["python", "-m", "gunicorn"]` rather than a shell script; a shell in the production image allows an attacker with command injection to spawn an interactive shell session.

127. ALWAYS apply Kubernetes `NetworkPolicy` resources with a default-deny-all ingress rule and explicit allow rules — a cluster without NetworkPolicies allows any pod to connect to any other pod on any port; a compromised application pod can reach the database, Redis, and the Kubernetes API server directly.

128. NEVER mount a writable volume over the entire `/app` directory — if writable output is needed (cached assets, compiled templates), mount it at a specific subdirectory like `/app/cache` with an explicit `emptyDir` volume, not over the entire code tree.

129. ALWAYS set `WORKDIR /app` with explicit ownership before switching to the non-root user — run `RUN mkdir -p /app && chown appuser:appuser /app` before `USER appuser`; without explicit ownership, `WORKDIR` creates the directory owned by root and the non-root user cannot write to it at runtime.

130. ALWAYS pin production base images to a digest SHA — use `FROM python:3.12.3-slim@sha256:<digest>` rather than `FROM python:3.12.3-slim`; version tags are mutable and can be updated to point to a different image without your knowledge; digest pinning guarantees byte-for-byte reproducibility.

