# API Design — Study Notes (Klement's Plain English Guide)

> Written from first principles. Every concept explained with analogies before code.
> Read this before reading the technical wiki page.

---

## Concept 1: What is an API?

**The restaurant analogy:**

| Restaurant | Software |
|---|---|
| You (hungry customer) | Mobile app, browser, another program |
| The waiter | The **API** |
| The kitchen | The **server** (FastAPI code) |
| The menu | List of things the server can do |
| Your order | The **request** |
| The plate of food | The **response** |

The API is the **contract** between the outside world and your server.

The contract says:
- Here is what you can ask for.
- Here is exactly how you must ask.
- Here is exactly what you will get back.

**One sentence:** An API is the waiter between the outside world and your server.

---

## Concept 2: Request and Response

**The letter analogy:**

| Letter | Software |
|---|---|
| You writing the letter | Making a **request** |
| The letter you send | The **request** |
| The reply you get back | The **response** |

### A request has 4 parts

```
METHOD   URL             HEADERS           BODY
------   ---             -------           ----
GET      /notes          token: abc123     (empty)
POST     /notes          token: abc123     {"title": "My note"}
```

| Part | Plain English |
|---|---|
| Method | The action — what do you want to do? |
| URL | The address — what thing are you acting on? |
| Headers | Extra info — your login token, data format |
| Body | The content you are sending (only for POST/PATCH) |

### A response has 3 parts

```
STATUS CODE     HEADERS            BODY
-----------     -------            ----
200             Content-Type:json  {"id": 42, "title": "Shopping list"}
404             Content-Type:json  {"error": "Note not found"}
```

| Part | Plain English |
|---|---|
| Status code | A number — did it work? |
| Headers | Extra info about the response |
| Body | The actual data returned |

**Rule:** 2xx = good. 4xx = your fault. 5xx = our fault.

**One sentence:** A request is "I want this." A response is "Here it is" or "Here is why I can't."

---

## Concept 3: HTTP Methods

**The library analogy:**

| Method | Library analogy | Plain English |
|---|---|---|
| `GET` | Read a book | Give me this data. Don't change anything. |
| `POST` | Add a new book | Create something new. |
| `PATCH` | Fix one page in a book | Update part of something. |
| `DELETE` | Remove a book | Delete this thing. |
| `PUT` | Replace the entire book | Replace everything with what I send. |

### PATCH vs PUT — the important difference

**PATCH** = fix only what you send. Everything else stays.

```
Send: {"name": "Klement G"}
Result: name changed. email, phone, bio all untouched.
```

**PUT** = replace the entire record. Fields not sent become null.

```
Send: {"name": "Klement G"}
Result: name changed. email = null. phone = null. bio = null. ALL GONE.
```

**Use PATCH almost always. Use PUT only when replacing a complete document.**

### Idempotent — important word

**Idempotent** = do it once or ten times, the result is the same.

**Light switch analogy:** Flip ON once = light on. Flip ON ten times = light still on. Same result.

| Method | Idempotent? | Risk |
|---|---|---|
| `GET` | Yes | Zero — only reads |
| `PATCH` | Yes | Low — partial update |
| `PUT` | Yes | Medium — replaces all |
| `DELETE` | Yes | High — permanent |
| `POST` | **No** | Highest — creates duplicates on retry |

**Why it matters:** Networks fail. Apps retry. POST retry = duplicate created. Payment POST retry = double charge.

### All 9 methods

```
Daily use:       GET, POST, PATCH, DELETE
Occasional:      PUT (full replacement)
Automatic:       OPTIONS (browser handles for CORS)
Debugging only:  HEAD (check existence without downloading)
Disable this:    TRACE (security risk)
```

**One sentence:** GET reads, POST creates, PATCH updates part, DELETE removes — POST is the only one dangerous to retry.

---

## Concept 4: Status Codes

**The envelope sticker analogy:**

- Green sticker = good news (2xx)
- Yellow sticker = go elsewhere (3xx)
- Red sticker = problem (4xx or 5xx)

### 2xx — Success

| Code | Name | When |
|---|---|---|
| `200` | OK | Worked. Here is the data. |
| `201` | Created | New thing made. Include Location header. |
| `202` | Accepted | Received. Still processing. Like a dry cleaner ticket. |
| `204` | No Content | Worked. Nothing to send back. Use for DELETE. |

### 4xx — Client Error (your fault)

| Code | Name | When |
|---|---|---|
| `400` | Bad Request | Malformed request, missing field |
| `401` | Unauthorized | Not logged in — no ID at the nightclub |
| `403` | Forbidden | Logged in but not allowed — not on the VIP list |
| `404` | Not Found | Does not exist (or hidden for security) |
| `409` | Conflict | Already exists — duplicate username/email |
| `422` | Unprocessable | Pydantic validation failed |
| `429` | Too Many Requests | Slow down. Add Retry-After header. |

### 401 vs 403 — the nightclub analogy

**401** = You have no ID. You are not identified at all.
**403** = You have ID. You are identified. But you are not on the list.

### 5xx — Server Error (our fault)

| Code | Name | When |
|---|---|---|
| `500` | Internal Server Error | We crashed |
| `503` | Service Unavailable | We are down |

**Iron rule for 5xx:** Never send raw error details. Never expose database names, file paths, or stack traces. Send a clean message + request_id only.

**One sentence:** Status codes are the sticker on the envelope — read the number before reading the body.

---

## Concept 5: URL Design Rules

**The good city analogy:** A good city has logical street addresses. You can find anything without a map. Good URLs are the same — predictable without documentation.

### Rule 1: Nouns not verbs

```
WRONG                   RIGHT
/getUser/42        →    GET /users/42
/createNote        →    POST /notes
/deleteNote/42     →    DELETE /notes/42
```

The HTTP method IS the verb. Don't repeat it in the URL.

### Rule 2: Plural for collections, ID for one item

```
/notes          → all notes
/notes/42       → note number 42
```

### Rule 3: Hierarchy only when things truly belong together

```
/users/42/notes       ← notes belong to user. Good.
/notes/7/comments     ← comments belong to note. Good.
```

Never go deeper than 3 levels. Flatten if too deep.

### Rule 4: Lowercase, hyphens not underscores

```
WRONG                   RIGHT
/userProfiles      →    /user-profiles
/user_profiles     →    /user-profiles
```

### Rule 5: No file extensions

```
WRONG          RIGHT
/notes.json → /notes    (use Accept header for format)
```

### Rule 6: Version in the URL

```
/api/v1/notes    ← old clients still work
/api/v2/notes    ← new behavior
```

Only create v2 when you have a real breaking change.

### Rule 7: Query params for options, not identity

```
/notes/42              ← identity (which note)
/notes?tag=work        ← option (filter by tag)
/notes?limit=20        ← option (how many)
/notes?sort_by=title   ← option (how to sort)
```

### Complete example

```
GET    /api/v1/notes              → list all notes
POST   /api/v1/notes              → create a note
GET    /api/v1/notes/42           → read note 42
PATCH  /api/v1/notes/42           → update note 42
DELETE /api/v1/notes/42           → delete note 42
GET    /api/v1/notes?limit=20&sort_by=title
GET    /api/v1/health             → server health check
```

**The test:** Cover the HTTP method. Can you tell what the URL is about? Can you guess what GET/POST/DELETE would do? If yes — good design.

**One sentence:** A URL is a predictable address — nouns not verbs, plural for collections, query params for options.

---

## Concept 6: Pagination

**The bookmark analogy:**

- **Offset** = "skip the first 40 books and give me the next 20." The library still counts through 40 books every time. Gets slow at depth. Fine for small admin reports.
- **Cursor** = "start from where this bookmark is." The library opens directly to that page. Always instant.

Use cursor for all user-facing lists. Use offset only for small bounded admin pages.

```
Offset URL:  /notes?limit=20&offset=40
Cursor URL:  /notes?limit=20&after=eyJpZCI6IDIwfQ==
```

Response shape:
```json
{"items": [...], "next_cursor": "eyJpZCI6IDIwfQ==", "has_more": true}
```
Last page: `next_cursor: null`, `has_more: false`

**One sentence:** Cursor pagination is a bookmark — never counts from the start, always jumps directly to where you left off.

---

## Concept 7: Sorting Safety

**The librarian analogy:**

You ask the librarian: "Sort the books by whatever field I write on this note."

Most of the time the note says "title" or "author." Fine.

But one day someone hands a note that says: `title; DELETE ALL BOOKS;`

The librarian just does it — because they follow instructions literally without checking first.

That's what happens when you pass the user's sort field directly into a database command.

### The safe pattern

Keep a **locked list** of allowed fields in your code:

```python
SORTABLE_FIELDS = {
    "created_at": Note.created_at,
    "updated_at": Note.updated_at,
    "title":      Note.title,
}
```

When user sends `sort_by=created_at`:
1. Look it up in your locked list ✅
2. Found → use the actual column object (not the user's text)
3. Not found → return 400 error immediately

The user's text **never touches SQL**. You always use your own safe column reference.

### Edge cases to know

**1. Case sensitivity** — user sends `Created_At` instead of `created_at`. Your allowlist lookup fails. Fix: always normalize to lowercase before looking up.

**2. Sorting + pagination together** — cursor pagination bookmarks by `id`. If you sort by `title`, the cursor also needs to remember the `title` value. Otherwise page 2 gives you wrong results. Rule: when sorting by anything other than id, your cursor must encode both the sort value AND the id.

**3. Nullable columns** — if a column can be empty (null), where do nulls appear when sorted? Different databases behave differently. Always explicitly say `nulls last` so behavior is consistent everywhere.

**One sentence:** Never put user-supplied sort field names into SQL — always look them up in an allowlist you control, and reject anything not on the list.

---

## Concept 8: RFC 7807 — Standard Error Shape

**The restaurant analogy:**

You order from 3 restaurants. When something goes wrong:
- Restaurant A texts you: `"nope"`
- Restaurant B sends a 6-page PDF
- Restaurant C gives a clear slip: **what went wrong, which order, why, what to do next**

RFC 7807 is the agreement that all APIs be like Restaurant C — every error looks the same shape.

### The shape

```json
{
    "type": "https://errors.myapp.com/not-found",
    "title": "Resource Not Found",
    "status": 404,
    "detail": "Note with id=42 does not exist.",
    "instance": "/api/v1/notes/42"
}
```

| Field | Job |
|---|---|
| `type` | A URL naming the category of error (permanent, never changes) |
| `title` | Human-readable name for this error type |
| `status` | The HTTP status code (404, 422, 409…) |
| `detail` | What specifically went wrong this time |
| `instance` | Which URL caused this error |

### Why it matters

Without a standard, different endpoints return errors in different shapes. Client code has to handle each one separately. With RFC 7807, one error handler covers the whole API.

### Edge cases to know

**1. Content-Type header** — RFC 7807 errors should have `Content-Type: application/problem+json`, not the usual `application/json`. It signals to the client "this is a problem details object."

**2. Three handlers, not one** — most people only write the Pydantic validation handler. But there are 3 types of errors in FastAPI:
- Pydantic failures (422) → one handler
- HTTP errors like 404, 401, 403 → second handler
- Unexpected crashes (500) → third handler
All three must return the same RFC 7807 shape.

**3. `title` stays the same, `detail` changes** — for the same type of error (e.g. "not found"), the `title` is always "Resource Not Found." The `detail` tells you what specifically — "Note 42 not found." Never put specific info in `title`.

**4. `request_id` in every error** — the error body must include the request ID so a user can report it and you can find it in your logs.

**5. `about:blank` is a valid type** — if you don't have a real error catalog website, RFC 7807 allows `"type": "about:blank"` as a fallback. Use it for generic errors.

**One sentence:** RFC 7807 is the agreement that every error from every endpoint looks the same shape — so client code only needs to handle errors in one place.

---

## Concept 9: Idempotency Keys — Preventing Double Charges

**The coffee shop analogy:**

You tap your card to pay. The machine freezes. You don't know if it went through. You tap again. Two charges appear on your statement.

The server processed the first request — but you never got confirmation. So you retried. The server had no way to know it was the same payment.

Idempotency keys fix this. Before you tap, your phone generates a unique ticket number for this specific payment: `pay-abc123`. Both taps carry the same ticket. Server sees the second one and says: "I already processed `pay-abc123`. Here's the same receipt. Not charging again."

### How it works

**First request:**
1. Client generates a unique key: `Idempotency-Key: pay-abc123`
2. Server processes the payment
3. Server saves the response in Redis: key → response (24 hour expiry)
4. Server returns the response

**Second request (retry):**
1. Client sends same key: `Idempotency-Key: pay-abc123`
2. Server checks Redis — found
3. Server returns **saved response** — does NOT charge again

### When to use

```
USE:        payments, email sends, SMS, account creation, order placement
NOT NEEDED: GET (reading never creates duplicates)
NOT NEEDED: DELETE (already idempotent — deleting twice = same result)
```

### Edge cases

**1. Same key, different body** — someone sends the same key but different request data. Return `409 Conflict`. The key is locked to the first request's data.

**2. Request still processing** — first request is still running when the retry arrives. Return `409 Conflict` with "request in progress." Never start a second execution.

**3. Client generates the key, not the server** — if the server generated it, a retry would have no key to send. Only the client knows it's about to retry.

**4. Keys are scoped per user** — stored as `idempotency:{user_id}:{key}` so user A's key can never accidentally match user B's key.

**One sentence:** An idempotency key is a unique ticket the client generates before each dangerous operation — the server uses it to return the same result on retry instead of doing the work twice.

---

## Concept 10: Rate Limiting — Protecting Your API

**The coffee shop analogy:**

A coffee shop has one barista who can make 100 coffees per hour. If 10,000 bots walk in at the same time, the barista collapses and real customers get nothing.

Rate limiting is the "max 100 per hour" sign on the door. It protects your server from being overwhelmed and ensures real users still get served.

### Two different limits

| Endpoint type | Limit | Why |
|---|---|---|
| Login / Register | **5 per minute** | Attackers try millions of passwords. Slow them down hard. |
| Reading data | **100 per minute** | Normal users read a lot. Be generous but not unlimited. |

### The headers you always send back

Every response carries these 3 headers so the client knows where they stand:

```
X-RateLimit-Limit:     100        ← total allowed this window
X-RateLimit-Remaining: 47         ← how many left right now
X-RateLimit-Reset:     1713000000 ← Unix timestamp when window resets
```

When limit is hit — return `429 Too Many Requests` plus:

```
Retry-After: 43    ← wait this many seconds before trying again
```

Without `Retry-After`, the client has no idea when to retry.

### Edge cases

**1. IP isn't always the right key** — behind a company network, 1000 employees share one IP. Rate limiting by IP blocks the whole company. For authenticated users, rate limit by `user_id` instead.

**2. Distributed servers** — 3 servers each track their own counter. User sends 5 to each = 15 total, bypassing the limit. Fix: store counters in shared Redis, not in each server's memory.

**3. Burst vs sustained** — some users send 50 requests in 2 seconds then nothing. A fixed window (100/minute) allows this. A sliding window is stricter. Use sliding window for auth endpoints.

**4. Repeated 429s = likely attack** — log every 429. Repeated hits from the same IP should escalate to a longer ban, not just the per-minute window.

**One sentence:** Rate limiting is the "max N requests per minute" rule that protects your server — auth endpoints get a strict limit, read endpoints get a generous one, and every 429 tells the client exactly when to retry.
