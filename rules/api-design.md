---
description: API Design rules — pagination, sorting, errors, idempotency, rate limiting.
paths: ["*.py", "main.py", "app/**/*.py", "routes/**/*.py"]
---

# API Design Rules (L6)

> HTTP Methods (GET/POST/PATCH/PUT/DELETE) → see `http-contracts.md`
> Status codes (201, 204, 401, 403, 409…) → see `http-contracts.md`
> URL design (nouns, hyphens, versioning) → see `http-contracts.md`
> CORS → see `security.md`

---

## Richardson Maturity Model

Target **Level 2** for all production APIs.

```
Level 0 — single endpoint RPC:     POST /api  {"action": "getUser", "id": 42}
Level 1 — resource URLs:           POST /users/42  {"action": "get"}
Level 2 — HTTP methods + codes:    GET /users/42 → 200, POST /users → 201   ← TARGET
Level 3 — HATEOAS (links in body): rarely worth the complexity, skip
```

Rule: NEVER ship Level 0 or Level 1. NEVER add action verbs to URLs.

---

## Filtering

Use typed Pydantic models with `Depends()` for query filter parameters — not individual `Query()` params.

```python
from pydantic import BaseModel
from fastapi import Depends, Query

class NoteFilter(BaseModel):
    tag: str | None = None
    is_pinned: bool | None = None
    created_after: datetime | None = None

@router.get("/notes")
async def list_notes(
    filters: NoteFilter = Depends(),   # Pydantic validates all filter params
    db: AsyncSession = Depends(get_db),
):
    query = select(Note).where(Note.user_id == current_user.id)

    if filters.tag:
        query = query.where(Note.tag == filters.tag)
    if filters.is_pinned is not None:
        query = query.where(Note.is_pinned == filters.is_pinned)
    if filters.created_after:
        query = query.where(Note.created_at > filters.created_after)
```

Rule: NEVER accept filter params as raw strings without Pydantic validation.

---

## Pagination

### DO
- Use cursor pagination for all user-facing lists, feeds, timelines, infinite scroll
- Return `next_cursor` and `has_more` in every paginated response
- Encode cursors as `base64(json)` — opaque to clients, stable if internals change
- Default page size to 20, cap at 100 via `Query(le=100)`
- Fetch `limit + 1` rows to detect if more pages exist — return only `limit` rows

### NEVER
- NEVER use offset pagination for user-facing lists — O(n) scan cost at depth
- NEVER return unlimited results without pagination — crashes server and client
- NEVER expose raw IDs as cursors — always encode them
- NEVER use offset beyond page ~100 for any high-traffic endpoint

### Cursor pagination template
```python
import base64, json

def encode_cursor(id: int) -> str:
    return base64.b64encode(json.dumps({"id": id}).encode()).decode()

def decode_cursor(cursor: str) -> int:
    return json.loads(base64.b64decode(cursor).decode())["id"]

@router.get("/notes")
async def list_notes(
    limit: int = Query(default=20, ge=1, le=100),  # ge=1 prevents limit=0 crashing items[-1]
    after: str | None = Query(default=None),
    db: AsyncSession = Depends(get_db),
):
    query = select(Note).where(Note.user_id == current_user.id).order_by(Note.id.asc())

    if after:
        after_id = decode_cursor(after)
        query = query.where(Note.id > after_id)

    rows = (await db.execute(query.limit(limit + 1))).scalars().all()
    has_more = len(rows) > limit
    items = rows[:limit]

    return {
        "items": items,
        "next_cursor": encode_cursor(items[-1].id) if has_more else None,
        "has_more": has_more,
    }
```

### Paginated response shape
```json
{
    "items": [ "...list..." ],
    "next_cursor": "eyJpZCI6IDIwfQ==",
    "has_more": true
}
```
Last page: `next_cursor: null`, `has_more: false`

---

## Sorting

### DO
- Validate `sort_by` against an explicit allowlist dict before any SQL
- Normalize `sort_by` to lowercase before allowlist lookup — reject `Created_At` and `CREATED_AT` cleanly
- Return `400 Bad Request` if `sort_by` value is not in the allowlist
- Accept `order=asc` or `order=desc` as a separate query param
- Validate `order` with `Literal["asc", "desc"]`
- Use `.nullslast()` / `.nullsfirst()` explicitly on nullable columns — never rely on database default

### NEVER
- NEVER pass client-supplied `sort_by` directly into SQL — SQL injection vector
- NEVER use `text(sort_by)` or f-strings to build ORDER BY clauses
- NEVER trust any user input in a raw SQL string
- NEVER sort by a non-id column with cursor pagination unless the cursor encodes the sort key — cursor must encode `(sort_value, id)` for stability

### Sorting template
```python
from typing import Literal
from sqlalchemy import nullslast

SORTABLE_FIELDS = {
    "created_at": Note.created_at,
    "updated_at": Note.updated_at,
    "title":      Note.title,
}

@router.get("/notes")
async def list_notes(
    sort_by: str = Query(default="created_at"),
    order: Literal["asc", "desc"] = Query(default="desc"),
):
    key = sort_by.lower()  # normalize before lookup
    column = SORTABLE_FIELDS.get(key)
    if not column:
        raise HTTPException(400, f"Cannot sort by '{sort_by}'. Valid: {list(SORTABLE_FIELDS)}")

    direction = nullslast(column.desc()) if order == "desc" else nullslast(column.asc())
    query = select(Note).order_by(direction)
```

### Cursor + non-id sort (stable pagination)
When sorting by a non-id column, encode both sort key and id in the cursor:
```python
def encode_cursor(sort_val: str, id: int) -> str:
    return base64.b64encode(json.dumps({"sort_val": sort_val, "id": id}).encode()).decode()

# WHERE clause for stable cursor on title sort:
query = query.where(
    or_(
        Note.title > cursor_sort_val,
        and_(Note.title == cursor_sort_val, Note.id > cursor_id)
    )
)
```

---

## Error Responses — RFC 7807

The rule (return RFC 7807) lives in `http-contracts.md` rule 15.
This section owns the **implementation templates**.

### Rules
- Use `media_type="application/problem+json"` — not `application/json` (RFC 7807 spec)
- `title` must be constant for the same `type` — only `detail` and `instance` change per occurrence
- Always include `request_id` in error body — read from `request.headers.get("x-request-id")`
- Use `"about:blank"` as `type` when you don't have an error catalog URL — valid RFC 7807 fallback
- All 3 handlers must be registered: `RequestValidationError`, `HTTPException`, and bare `Exception`

### Shape
```json
{
    "type": "https://errors.myapp.com/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "One or more fields failed validation.",
    "instance": "/api/v1/notes",
    "request_id": "01HZ9K...",
    "errors": [
        {"field": "body.title", "message": "field required"}
    ]
}
```

### FastAPI — all 3 handlers (register all three, no exceptions)
```python
from fastapi import Request, HTTPException
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

# Handler 1: Pydantic validation failures (422)
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        media_type="application/problem+json",
        content={
            "type": "https://errors.myapp.com/validation-error",
            "title": "Validation Error",
            "status": 422,
            "detail": "One or more fields failed validation.",
            "instance": str(request.url.path),
            "request_id": request.headers.get("x-request-id", ""),
            "errors": [
                {
                    "field": ".".join(str(l) for l in err["loc"]),
                    "message": err["msg"],
                }
                for err in exc.errors()
            ],
        },
    )

# Handler 2: HTTPException (401, 403, 404, 409…)
# RFC 7807: title must be FIXED per status code — never use exc.detail as title
_HTTP_TITLES = {
    400: "Bad Request", 401: "Unauthorized", 403: "Forbidden",
    404: "Not Found", 409: "Conflict", 422: "Unprocessable Entity",
    429: "Too Many Requests", 500: "Internal Server Error",
    502: "Bad Gateway", 503: "Service Unavailable",
}

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        media_type="application/problem+json",
        content={
            "type": "about:blank",
            "title": _HTTP_TITLES.get(exc.status_code, "Error"),   # fixed label
            "status": exc.status_code,
            "detail": exc.detail,                                   # variable message
            "instance": str(request.url.path),
            "request_id": request.headers.get("x-request-id", ""),
        },
    )

# Handler 3: Unhandled exceptions — NEVER leak stack traces
@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        media_type="application/problem+json",
        content={
            "type": "about:blank",
            "title": "Internal Server Error",
            "status": 500,
            "detail": "An unexpected error occurred.",
            "instance": str(request.url.path),
            "request_id": request.headers.get("x-request-id", ""),
        },
    )
```

---

## Idempotency Keys

### DO
- Accept `Idempotency-Key` header on all POST endpoints with side effects
- Cache the first response in Redis with 24-hour TTL keyed by the idempotency key
- Return the exact same response on duplicate requests with the same key
- Scope keys per user: `idempotency:{user_id}:{key}` — prevents cross-user key collisions
- Client must generate the key before the request — server cannot generate it (retry would have no key)
- Return `409 Conflict` if same key arrives with different request body — key is locked to first request's data
- Return `409 Conflict` with "request in progress" if first request is still executing when retry arrives

### NEVER
- NEVER process the same idempotency key twice — check cache before doing work
- NEVER use idempotency keys as authentication — they are deduplication only
- NEVER store idempotency keys without TTL — they must expire
- NEVER apply idempotency keys to GET or DELETE — not needed (GET is safe, DELETE is already idempotent)

### When to use
```
Use for:        payments, email sends, account provisioning, SMS, order placement
Not needed for: GET (already safe), DELETE (already idempotent)
```

---

## Rate Limiting

### DO
- Rate limit auth endpoints strictly: `5 requests/minute per IP`
- Rate limit read endpoints leniently: `100 requests/minute per IP`
- Return `429 Too Many Requests` when limit is hit — rule in `http-contracts.md`
- Always include `Retry-After: {seconds}` header on 429 responses
- Include on every response:
  - `X-RateLimit-Limit` — total allowed per window
  - `X-RateLimit-Remaining` — remaining in this window
  - `X-RateLimit-Reset` — Unix timestamp when window resets
- Use `user_id` as rate limit key for authenticated endpoints — IP is wrong for shared networks (offices, universities)
- Store counters in Redis — never in-process memory (breaks on multi-server deployments)
- Use sliding window for auth endpoints — stricter than fixed window, prevents burst attacks
- Log every 429 response — repeated 429s from same IP = escalate to longer ban

### NEVER
- NEVER rate limit without `Retry-After` — clients cannot know when to retry
- NEVER apply the same limit to auth and read endpoints — auth must be stricter
- NEVER store rate limit counters in-process — they reset on restart and don't share across instances

### slowapi template
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.post("/auth/login")
@limiter.limit("5/minute")
async def login(request: Request, body: LoginRequest): ...

@router.get("/notes")
@limiter.limit("100/minute")
async def list_notes(request: Request): ...
```
