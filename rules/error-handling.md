---
description: Error handling + observability rules — exception hierarchy, structured logging, retry, circuit breaker.
paths: ["*.py", "main.py", "app/**/*.py", "routes/**/*.py"]
---

# Error Handling + Observability Rules (L7)

> RFC 7807 error shape + all 3 FastAPI handlers → see `api-design.md`
> What NOT to log (tokens, passwords, PII) → see `security.md`
> 429 + Retry-After status code rule → see `http-contracts.md`

---

## Custom Exception Hierarchy

### DO
- Root all custom exceptions in a single `AppError(Exception)` base class
- Each subclass declares its own `status_code` — mapping happens in the global handler
- Register exception handlers most-specific → least-specific (reverse registration order in FastAPI)
- Use `raise ExternalServiceError("msg") from original_exc` — `from` preserves original traceback
- Log errors ONCE — in the global handler only, never at the raise site
- Add `code` field to error responses when two errors share the same HTTP status: `"code": "NOTE_NOT_FOUND"`
- Use 502 Bad Gateway for upstream service failures — not 500 (500 = your server crashed)
- Include `request_id` in EVERY error response — the only way to correlate client error report with server logs

### NEVER
- NEVER inherit custom exceptions from `HTTPException` — domain errors must not know about HTTP
- NEVER raise 422 manually — 422 belongs to Pydantic's `RequestValidationError` handler only
- NEVER catch AppError inside route code — only the global handler catches it
- NEVER register catch-all `Exception` handler before specific handlers — it swallows everything
- NEVER swallow exceptions with bare `except: pass` — silent failures are invisible failures
- NEVER return raw stack traces in HTTP responses — rule in `http-contracts.md` rule 11
- NEVER use a different error shape on different endpoints — one shape, everywhere, always

### Exception hierarchy template
```python
class AppError(Exception):
    status_code: int = 500
    title: str = "Internal Server Error"   # fixed label per error type (RFC 7807)
    def __init__(self, detail: str, code: str | None = None):
        self.detail = detail   # specific instance message — changes per occurrence
        self.code = code
        super().__init__(detail)

class NotFoundError(AppError):
    status_code = 404
    title = "Resource Not Found"

class ConflictError(AppError):
    status_code = 409
    title = "Conflict"

class UnauthorizedError(AppError):
    status_code = 401
    title = "Unauthorized"

class ForbiddenError(AppError):
    status_code = 403
    title = "Forbidden"

class ExternalServiceError(AppError):
    status_code = 502  # upstream failed — not our fault
    title = "External Service Error"
```

### Global handler template
```python
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    logger.error(                          # log ONCE here, never at raise site
        "app_error",
        status_code=exc.status_code,
        detail=exc.detail,
        code=exc.code,
        request_id=request.headers.get("x-request-id", ""),
        path=str(request.url.path),
    )
    body = {
        "type": "about:blank",
        "title": exc.title,            # fixed label — same for all NotFoundErrors
        "status": exc.status_code,
        "detail": exc.detail,          # specific message — changes per occurrence
        "instance": str(request.url.path),
        "request_id": request.headers.get("x-request-id", ""),
    }
    if exc.code is not None:           # exclude code field entirely when not set
        body["code"] = exc.code
    return JSONResponse(
        status_code=exc.status_code,
        media_type="application/problem+json",
        content=body,
    )
```

---

## Structured Logging

### DO
- Use `structlog` — not Python's built-in `logging` (painful to get structured output from)
- Every log line is a dictionary — never an f-string sentence
- Include on every log line: `timestamp`, `level`, `event`, `request_id` — `user_id` included after auth resolves (not available on unauthenticated routes)
- Log all external API calls with duration: Anthropic, Stripe, SendGrid, S3
- Log all security events: login, failed auth, access denied, role change — see `security.md`
- Make log level configurable via env var — `DEBUG` in dev, `INFO` in production
- Propagate `request_id` via `contextvars.ContextVar` — one value per coroutine, auto-isolated

### NEVER
- NEVER log sensitive data — passwords, tokens, API keys, JWT payloads, PII → see `security.md`
- NEVER use f-strings for log messages — fields become unsearchable plain text
- NEVER hard-code `DEBUG` level in production — fills log storage in hours
- NEVER over-log — log state changes, external calls, errors, security events only

### Log levels
```
DEBUG     Dev only — detailed internals. NEVER in production.
INFO      Normal operations: user logged in, note created, job queued
WARNING   Unexpected but recovered: retry attempt 2 of 3, cache miss
ERROR     Failure needing attention: payment failed, DB connection lost
CRITICAL  System unusable: database unreachable, shutting down
```

### structlog setup template
```python
import os
import logging
import structlog
from uuid import uuid4

# Log level from env var — never hardcode
_log_level = getattr(logging, os.getenv("LOG_LEVEL", "INFO").upper(), logging.INFO)

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,  # pulls bound vars into every log line
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(_log_level),
)

logger = structlog.get_logger()

# In middleware — bind request_id to every log line AND return it in response header
@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request_id = request.headers.get("x-request-id", str(uuid4()))
    structlog.contextvars.bind_contextvars(request_id=request_id)
    try:
        response = await call_next(request)
        response.headers["x-request-id"] = request_id  # client can correlate their logs
        return response
    finally:
        structlog.contextvars.clear_contextvars()  # MUST be in finally — clears even on exception

# In auth dependency — bind user_id AFTER auth resolves so it appears in all subsequent log lines
async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    user = await verify_token(token, db)
    structlog.contextvars.bind_contextvars(user_id=user.id)  # all logs after this have user_id
    return user
```

### Logging external API calls
```python
import time

start = time.monotonic()
try:
    result = await anthropic_client.messages.create(...)
    logger.info("llm_call_success", duration_ms=int((time.monotonic() - start) * 1000), model=model)
except Exception as e:
    # DO NOT log here — global handler logs ExternalServiceError once
    raise ExternalServiceError("LLM call failed") from e
```

---

## Retry with Exponential Backoff

### DO
- Use `tenacity` library for all retry logic — never write manual retry loops
- Use exponential backoff with jitter — base doubles each attempt, add ~10% jitter to prevent thundering herd
- For 429s: read `Retry-After` header via custom `wait=` function — overrides your backoff calculation
- For other errors: `wait = min(10, 1 * 2^attempt) + 10% jitter` — implemented in `_wait_respecting_retry_after`
- Only retry idempotent operations — NEVER retry POST without an idempotency key
- Set `stop=stop_after_attempt(3)` — never retry forever
- Log each retry attempt at WARNING level with attempt number and error cause
- Use `retry_if_not_exception_type` carefully — it catches ALL unexcluded errors including 400/401/422; prefer `retry_if_exception_type` for explicit allowlist

### NEVER
- NEVER retry on validation errors (400, 422) — the request is broken, retrying won't fix it
- NEVER retry on auth errors (401, 403) — token is wrong, retrying won't fix it
- NEVER retry without backoff — hammering a failing service makes it worse
- NEVER retry POST endpoints without idempotency key — creates duplicates
- NEVER retry blindly in high-fanout scenarios — 3 retries × 1000 parallel requests = 3000 calls; use circuit breaker to stop the flood

### LLM-specific retry rules
```
RateLimitError (429)        → retry — read Retry-After header for exact wait time
APIConnectionError          → retry with backoff (network issue)
APITimeoutError             → retry with backoff
APIStatusError 529          → retry with longer backoff (Anthropic overloaded)
ValidationError             → DO NOT retry — fix the request
APIStatusError 400          → DO NOT retry — bad request
APIStatusError 401/403      → DO NOT retry — auth issue, fix the credentials
```

### tenacity template
```python
from anthropic import RateLimitError, APIConnectionError, APITimeoutError, APIStatusError
from tenacity import (
    retry, stop_after_attempt,
    retry_if_exception_type
)

# retry_if_not_exception_type — use when it's easier to list what NOT to retry
# WARNING: APIStatusError covers ALL 4xx/5xx — if you use this, you retry on 529 (good)
# BUT ALSO on 400/401/422 (bad). Prefer retry_if_exception_type for LLM calls.

def _wait_respecting_retry_after(retry_state) -> float:
    """Use Retry-After header value if present, otherwise exponential backoff."""
    exc = retry_state.outcome.exception()
    if isinstance(exc, RateLimitError):
        header = getattr(getattr(exc, "response", None), "headers", {})
        return float(header.get("retry-after", 10))
    # exponential backoff with jitter for other errors
    base = min(10, 1 * (2 ** (retry_state.attempt_number - 1)))
    return base + (0.1 * base)  # 10% jitter

def _log_retry(retry_state):
    logger.warning(
        "llm_retry",
        attempt=retry_state.attempt_number,
        error=str(retry_state.outcome.exception()),
    )

def _log_retry_exhausted(retry_state):
    # MUST raise — returning None would make call_llm silently return None
    logger.error(
        "llm_retry_exhausted",
        attempts=retry_state.attempt_number,
        error=str(retry_state.outcome.exception()),
    )
    raise ExternalServiceError("LLM unavailable after retries") from retry_state.outcome.exception()

@retry(
    retry=retry_if_exception_type((RateLimitError, APIConnectionError, APITimeoutError)),
    wait=_wait_respecting_retry_after,   # ← respects Retry-After on 429, backoff otherwise
    stop=stop_after_attempt(3),
    before_sleep=_log_retry,
    retry_error_callback=_log_retry_exhausted,  # MUST raise — see above
)
async def call_llm(prompt: str) -> str:
    return await anthropic_client.messages.create(...)
```

---

## Circuit Breaker

### DO
- Use circuit breaker for all external service calls (LLM, payment, email)
- CLOSED state: normal operation, requests pass through
- OPEN state: service is failing, reject requests immediately without calling the service
- HALF-OPEN state: test recovery with one REAL request — not a ping or health check
- Set failure threshold: open after 5 consecutive failures
- Set recovery timeout: try again after 60 seconds in OPEN state
- Create one circuit breaker PER external service — Stripe down must not affect Anthropic calls
- Log every state transition: CLOSED→OPEN (alert — service down), OPEN→CLOSED (info — service recovered)
- Attach listener via `listeners=[_BreakerListener()]` — this is the only hook into internal breaker events (state changes, failures, successes) that you don't control
- Use multiple listeners when needed: one for logging, one for Prometheus metrics, one for PagerDuty alerts — all receive every event
- Override `failure(cb, exc)` in listener to track error types; `success(cb)` to track recovery rate; `state_change` for transitions
- Return a meaningful fallback when circuit is OPEN: cached data, queued for later, or clear "unavailable" message

### NEVER
- NEVER call a known-failing external service without a circuit breaker — cascades failures
- NEVER open circuit on client errors (4xx) — only open on server/network errors (5xx, timeouts, connection errors)
- NEVER use a single global circuit breaker for all services — one failure opens all
- NEVER return raw 500 when circuit is OPEN — return a degraded but meaningful response

### pybreaker template
```python
import pybreaker  # requires pybreaker >= 1.0 for async support

class _BreakerListener(pybreaker.CircuitBreakerListener):
    def state_change(self, cb, old_state, new_state):
        # CLOSED→OPEN = service down (warning), OPEN→CLOSED = recovered (info)
        log_fn = logger.warning if new_state.name == "open" else logger.info
        log_fn(
            "circuit_breaker_state_change",
            breaker=cb.name,
            old=old_state.name,
            new=new_state.name,
        )

# One breaker per external service
llm_breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=60,
    name="anthropic",
    listeners=[_BreakerListener()],
)
stripe_breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=60, name="stripe", listeners=[_BreakerListener()])

# Circuit breaker WRAPS retry — breaker is on the outside
# If retry were on the outside, retries would happen even when circuit is OPEN
@llm_breaker                       # ← outer: stops all calls when circuit is OPEN
async def call_llm_with_breaker(prompt: str):
    return await call_llm(prompt)  # ← inner: tenacity retry handles transient failures

# Always handle CircuitBreakerError — graceful fallback, not a raw 500
async def call_llm_guarded(prompt: str) -> str:
    try:
        return await call_llm_with_breaker(prompt)
    except pybreaker.CircuitBreakerError:
        logger.warning("llm_circuit_open")
        raise ExternalServiceError("LLM service temporarily unavailable")
```

## Sentry Integration

- Install `sentry-sdk[fastapi]` and initialise in lifespan before anything else runs:
    ```python
    import sentry_sdk
    from sentry_sdk.integrations.fastapi import FastApiIntegration
    from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

    sentry_sdk.init(
        dsn=os.environ["SENTRY_DSN"],
        integrations=[FastApiIntegration(), SqlalchemyIntegration()],
        traces_sample_rate=0.1,   # 10% of requests — adjust by traffic volume
        environment=os.environ.get("ENV", "development"),
    )
    ```
- Sentry auto-captures unhandled exceptions and FastAPI request context — no manual `capture_exception()` needed for HTTP errors
- Set `user` context in auth dependency so Sentry errors show which user was affected:
    ```python
    sentry_sdk.set_user({"id": str(user.id), "email": user.email})
    ```
- NEVER set `traces_sample_rate=1.0` in production for high-traffic services — Sentry charges per event; 0.05–0.1 is typical

## Alerting Thresholds

- Alert PagerDuty/Slack when: error rate (5xx / total) exceeds 1% over 5 minutes
- Alert when any `CRITICAL` log line appears — these should be rare and always actionable
- Alert when circuit breaker transitions to OPEN — means an external service is down
- Alert when P99 response latency exceeds 10s for 2 consecutive minutes
- NEVER alert on every 4xx — client errors are noise; only alert on patterns (e.g. sudden spike in 401s suggesting token issue)
