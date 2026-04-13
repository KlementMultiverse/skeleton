# Error Handling + Observability — Study Notes (Klement's Plain English Guide)

> Written from first principles. Every concept explained with analogies before code.
> Read this before reading the technical wiki page.

---

## Concept 1: Custom Exception Hierarchy

**The hospital alarm analogy:**

A hospital has different alarms — fire alarm, cardiac alarm, security breach. Each one tells staff what happened, how urgent, and what to do.

A bare Python `Exception("something went wrong")` is one alarm for everything. You hear it but don't know what's wrong.

A custom exception hierarchy is a family of named alarms:

```
AppError                  ← the parent
├── NotFoundError         → 404
├── ConflictError         → 409
├── UnauthorizedError     → 401
├── ForbiddenError        → 403
└── ExternalServiceError  → 502
```

One global handler catches all of them and returns the right RFC 7807 response automatically. Route code just raises the right exception — never builds HTTP responses manually.

### Edge cases

**1. Handler order matters** — register most specific first, catch-all last. If catch-all `Exception` is registered first, it swallows everything including your specific handlers.

**2. Don't inherit from HTTPException** — your exceptions are business concepts (note not found), not HTTP concepts. The handler does the HTTP mapping, not the exception itself.

**3. ExternalServiceError is 502, not 500** — 500 means YOUR server crashed. 502 Bad Gateway means an upstream service (Stripe, OpenAI) failed. Different meaning, different code.

**4. `from e` preserves traceback** — `raise ExternalServiceError("msg") from original_exc` chains exceptions. Logs show your clean message AND the original error underneath.

**5. Log once — in the handler** — never log at the raise site AND in the handler. You'll get duplicate log lines for every error.

**6. 422 belongs to Pydantic** — never raise a custom 422. Pydantic's RequestValidationError handler owns it.

**7. Add internal `code` field** — two different 404 errors can mean different things. `"code": "NOTE_NOT_FOUND"` vs `"code": "USER_NOT_FOUND"` lets the client handle them separately.

**One sentence:** A custom exception hierarchy is a family of named alarms — each one knows its own severity and what HTTP response it produces.

---

## Concept 2: Structured Logging

**The hospital records analogy:**

A hospital recording events as sentences:
```
"Patient came in. Treated. Left."
"Doctor saw someone."
```
Finding all cardiac events for Dr. Smith last Tuesday = impossible.

Same hospital using structured forms:
```
timestamp: 2026-04-12 14:32
doctor: Dr. Smith
event: cardiac_alert
patient_id: 4421
severity: high
```
Finding all cardiac events for Dr. Smith = one query, instant.

That's the difference between f-string logs and structured logging.

### The problem with normal logging

```python
print(f"User {user_id} failed to login")     # plain text — unsearchable
logger.info(f"Note {note_id} created")       # same problem
```

To find patterns you need regex. To count failures you need grep. To find all errors for one user — nearly impossible.

### Structured logging

Every log line is a **dictionary**, not a sentence:

```json
{
    "timestamp": "2026-04-12T14:32:01Z",
    "level": "error",
    "event": "login_failed",
    "user_id": 442,
    "request_id": "01HZ9K...",
    "reason": "invalid_password"
}
```

Query: "show me all `login_failed` for `user_id=442` in the last hour." One query. Instant.

### Log levels

| Level | When | Example |
|---|---|---|
| `DEBUG` | Dev only — detailed internals | "Entering function X" |
| `INFO` | Normal operations | "User logged in", "Note created" |
| `WARNING` | Unexpected but recovered | "Retry attempt 2 of 3" |
| `ERROR` | Failure needing attention | "Payment failed" |
| `CRITICAL` | System unusable | "Database unreachable" |

DEBUG never runs in production. INFO and above do.

### Request ID — the thread connecting everything

One user action = many log lines (request received, auth checked, DB queried, response sent). Without a request ID they look like unrelated events.

With a request ID, search `request_id=01HZ9K` → see only those 4 lines. The complete story of one request.

The request ID travels through every log line for that request via a `ContextVar` — a variable that's automatically different for each concurrent request.

### Edge cases

**1. Never log sensitive data** — passwords, tokens, API keys, JWT payloads, credit card numbers. Once in logs they're hard to remove.

**2. Structured ≠ verbose** — log state changes, external calls, errors, security events. Over-logging buries the signal.

**3. Log level must be configurable** — DEBUG in dev, INFO in production. Hard-coding DEBUG in production fills storage in hours.

**4. Always log external API calls** — every call to Anthropic, Stripe, SendGrid gets a log line with duration. You'll know which external service is slow or failing.

**5. Use `structlog` not built-in `logging`** — Python's built-in logging can do structured output but it's painful. `structlog` makes every log line a clean dictionary out of the box.

**One sentence:** Structured logging means every log line is a searchable dictionary — not a sentence — so you can find any event across millions of lines in one query.

---

## Concept 3: Retry with Exponential Backoff

**The phone call analogy:**

You call a friend. No answer. You wait 1 minute, call again. No answer. You wait 2 minutes. Call again. Wait 4 minutes. Call again.

You don't call back immediately — you give them time. And you wait longer each time.

That's exponential backoff. Each retry waits **twice as long** as the previous one.

### Why not retry immediately?

1000 users all hit your API. LLM provider goes down for 2 seconds. All 1000 fail. All 1000 retry instantly. Provider just recovered — and gets slammed again and goes down again.

This is the **thundering herd problem**. Everyone retries at the same moment and kills the recovering service.

Two fixes:
- **Exponential backoff** — wait 1s → 2s → 4s between retries
- **Jitter** — add a small random extra wait so not everyone retries at the exact same millisecond

### What to retry vs what NOT to retry

| Error | Retry? | Why |
|---|---|---|
| Network timeout | ✅ | Transient — might work next time |
| Rate limit 429 | ✅ | Wait and retry — server will recover |
| Server overloaded 529 | ✅ | Wait longer |
| Validation error 400/422 | ❌ | Request is broken — retrying won't fix it |
| Auth error 401/403 | ❌ | Token is wrong — retrying won't fix it |
| POST without idempotency key | ❌ | Creates duplicates |

### Edge cases

**1. Never retry forever** — max 3 attempts is standard. After that, raise an error.

**2. `Retry-After` header overrides your backoff** — 429 responses often include `Retry-After: 43`. Use that exact number, not your own backoff calculation.

**3. Only retry idempotent operations** — GET is always safe. POST without an idempotency key creates duplicates on retry.

**4. Log every retry** — each retry is a warning. Log attempt number + what error caused it.

**5. Retry budget at scale** — 3 retries × 1000 parallel requests = 3000 calls. In high-traffic systems, retries multiply load. Use a circuit breaker to stop retrying when a service is truly down.

**6. Circuit breaker is the outer layer** — retry handles "maybe it'll work in 2 seconds." Circuit breaker handles "this has been failing for 5 minutes, stop trying completely."

**One sentence:** Exponential backoff means waiting twice as long after each failed attempt, plus random jitter, so retries don't all hammer a recovering service at the same time.

---

## Concept 4: Circuit Breaker

**The electrical breaker analogy:**

Your house has a circuit breaker box. When a wire overloads, the breaker **trips** — it cuts power completely. It doesn't keep pushing electricity through a broken wire. That would start a fire.

You go to the box, flip it back, and it tries again. Fixed? Power flows. Not fixed? Trips again.

Software circuit breaker does the same with failing external services.

### Three states

```
CLOSED → everything normal, requests pass through
   ↓ (5 consecutive failures)
OPEN → service is broken, reject ALL requests immediately — don't even try
   ↓ (after 60 seconds)
HALF-OPEN → send ONE test request
   ↓ success          ↓ failure
CLOSED again       OPEN again
```

### Why not just keep retrying?

**Retry** = "maybe it'll work in 2 seconds."
**Circuit breaker** = "this has been failing for 5 minutes — stop wasting resources."

Without circuit breaker: every request waits 30 seconds for a timeout. 100 users × 30 seconds = your entire app frozen.

With circuit breaker: after 5 failures, requests are rejected **instantly**. Users get an immediate error. The failing service gets breathing room.

### Edge cases

**1. Don't open on 4xx errors** — 404 or 400 means the REQUEST is wrong, not the SERVICE is down. Only open on 5xx, timeouts, and connection errors.

**2. HALF-OPEN test must be a real request** — not a ping or health check. Send the actual type of request the service handles. Health check passing doesn't mean your use case works.

**3. One breaker per service** — if Stripe is down, only open the Stripe breaker. Don't stop calling Anthropic too.

**4. Log every state transition** — CLOSED→OPEN means a service went down (alert worthy). OPEN→CLOSED means it recovered (log at info level).

**5. Return a meaningful fallback when OPEN** — don't just return 500. Return cached data, a queue-for-later response, or a clear "service temporarily unavailable" message.

**One sentence:** A circuit breaker stops all calls to a failing service after a threshold of failures, rejects them instantly instead of waiting for timeouts, and automatically tests recovery after a cooldown.
