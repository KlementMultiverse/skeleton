---
description: HTTP contract rules. Loads for all FastAPI projects.
paths: ["*.py", "main.py", "app/**/*.py", "routes/**/*.py"]
---

# HTTP Contract Rules

## Status Codes

1. Return 201 + Location header for successful POST that creates a resource
2. Return 202 Accepted + job_id for async background operations
3. Return 204 No Content for successful DELETE (no body)
4. Return 200 + updated resource for PATCH when returning the modified object
5. Return 204 No Content for PATCH when returning no body — pick one convention and be consistent across the entire API
6. Use 409 Conflict for duplicate resource (not 400)
7. Use 422 for Pydantic validation failures (FastAPI default — keep it)
8. Use 401 when user is not authenticated (no token / invalid token)
9. Use 403 when user IS authenticated but lacks permission (never 403 for unauthenticated)
10. Return 429 + Retry-After header for all rate limit responses
11. NEVER return 200 OK for a failed operation
12. NEVER return raw Python exception messages in 5xx responses
13. NEVER use 500 for validation errors — those are 4xx client errors

## Status Code Cheat Sheet
```
200  GET success — data returned
201  POST create success — include Location: /resource/{id}
202  Background job started — include job_id in body
200  PATCH success — when returning the updated resource
204  DELETE success (no body) / PATCH success (no body)

400  Malformed request, missing required field
401  Not authenticated — no token or expired token
403  Authenticated but wrong role or permission
404  Does not exist (also use to hide unauthorized resources)
409  Duplicate — already exists
422  Pydantic validation failed
429  Rate limit hit — add Retry-After: {seconds}

500  Server crashed — clean message + request_id only
502  Upstream service failed (Stripe, Anthropic, etc.) — not your fault
503  Server down or overloaded — add Retry-After header
```

## Request and Response

13. Add X-Request-ID to every response via middleware
14. Include request_id in every error response body
15. Return RFC 7807 Problem Details shape for ALL errors:
    `{"type", "title", "status", "detail", "instance", "request_id"}`
16. Return the same error shape from every endpoint — never mix formats
17. Return 404 (not 403) for resources the user shouldn't know exist — never reveal existence of unauthorized resources
18. Return `WWW-Authenticate: Bearer` header on every 401 response — RFC standard for bearer token auth
19. Return 202 responses MUST include a `job_id` AND a polling endpoint URL — client must be able to check status

## URL Design

20. Use plural nouns for collections: /notes, /users, /comments
21. Use path params for specific items: /notes/42, /users/7
22. Use ONLY lowercase letters in URLs
23. Use hyphens between words: /user-profiles not /user_profiles
24. NEVER use verbs in URLs — HTTP method is the verb
25. NEVER add file extensions to URLs: no /notes.json
26. Use /api/v1/ prefix — only create v2 when a real breaking change exists
27. Use query params for options, not identity: ?limit=20&sort_by=title

## HTTP Methods

28. GET must NEVER change server state — read only
29. HEAD returns the same headers as GET but NO body — use to check if resource exists without downloading it
30. OPTIONS returns `Allow` header listing permitted methods — CORS preflight uses it automatically (FastAPI handles it)
31. POST is NOT idempotent — never retry without an idempotency key
32. PATCH updates only fields sent — use model_dump(exclude_unset=True) — see `pydantic-patterns.md`
33. PUT replaces the entire record — all fields not sent become null
34. DELETE returns 204 No Content — no body
35. Accept Idempotency-Key header on all POST endpoints with side effects — see `api-design.md`
36. NEVER enable TRACE method in production — leaks server internals to attacker
37. NEVER use 200 with `{"success": false}` in the body — use the correct 4xx/5xx status code
38. Content-Type on all POST/PATCH/PUT requests MUST be `application/json` — reject 415 if wrong
39. Include `ETag` on GET responses for cacheable resources — enables `If-None-Match` for 304 Not Modified
40. Return 406 Not Acceptable when client sends `Accept` header your API cannot satisfy (e.g. `Accept: application/xml` on a JSON-only API)
