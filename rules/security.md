---
description: Security rules for all projects with auth, APIs, or data storage.
paths: ["**"]
---

# Security Rules

> This file owns: credentials, logging, CORS, SQL injection, SSRF, prompt injection, mass assignment
> Auth-specific security (JWT, bcrypt, 401/403) → see `auth-patterns.md`
> DB-level isolation (user_id filtering) → see `database-patterns.md`
> sort_by SQL injection → see `api-design.md`

---

## Credentials and Secrets

1. All credentials from environment variables — NEVER hardcoded anywhere
2. Use `pydantic-settings` + `BaseSettings` — raises error at startup if required var is missing
3. NEVER commit `.env` files — only commit `.env.example` with placeholder values
4. Run `pip audit` in CI pipeline before deploying — checks for known package vulnerabilities

## CORS

5. Whitelist specific origins only — NEVER `allow_origins=["*"]` in production
   — CORSMiddleware implementation in `fastapi-patterns.md`

## SQL Injection

6. NEVER pass user input directly into SQL strings — always use parameterized queries or ORM
7. NEVER use f-strings or `.format()` to build SQL queries
8. Raw SQL only via `text("...").bindparams(name=name)` — NEVER string interpolation
9. sort_by parameter requires allowlist — see `api-design.md`

## Input Validation

10. Validate all input at system boundaries — user input, external APIs, LLM output
11. LLM output is untrusted — always validate with Pydantic before saving to database
12. Wrap user content in explicit delimiters before injecting into LLM prompts
    ```python
    f"<user_message>{user_input}</user_message>"
    ```
    — prevents prompt injection attacks

## SSRF Prevention

13. NEVER make server-side HTTP requests to user-supplied URLs without validation
14. Validate that resolved IPs are not in private ranges (RFC1918: 10.x, 172.16.x, 192.168.x)
15. Use an allowlist of permitted external domains when possible

## Mass Assignment Prevention

16. Use separate Pydantic schemas for request input vs DB model — never pass request body directly to ORM
17. Never accept `role`, `is_admin`, `id`, or `created_at` as user input fields

## Session and Cookies

18. Session cookies: `HttpOnly=True, SameSite=Lax, Secure=True` in production
19. CSRF protection on all state-changing endpoints when using cookie-based sessions

## File Access

20. Use presigned URLs for file access — NEVER expose storage keys or bucket names
21. NEVER serve files directly from the application server — use CDN or presigned URLs
22. Validate file uploads by magic bytes (file header), not just file extension — extensions are trivially faked
23. NEVER allow user-supplied file paths to reach the filesystem — validate against a fixed directory and reject paths containing `..`, `/`, or `\` (path traversal)

## Security Response Headers

24. Set `X-Content-Type-Options: nosniff` on all responses — prevents MIME-type sniffing attacks
25. Set `X-Frame-Options: DENY` on all responses — prevents clickjacking
26. NEVER expose server version in `Server` header — configure middleware to strip or replace it
27. Enforce HTTPS — redirect all HTTP requests to HTTPS; never serve sensitive APIs over HTTP

## Open Redirect Prevention

28. NEVER redirect to a URL supplied by the client without validation — open redirect enables phishing
29. Validate redirect URLs are on your own domain — check against allowlist of permitted domains, never just check prefix

## Audit and Logging (this is the source of truth for what not to log)

30. Audit log must be immutable — save raises error if pk exists, delete raises error
31. Log security events: login, failed auth, access denied, role change
32. NEVER log: passwords, tokens, API keys, PII, session IDs, JWT payloads, Authorization headers

## Data Isolation

33. Tenant/user data isolation: filter every query by `user_id` or `tenant_id` — full rule in `database-patterns.md` rule 9
34. Role-based access: check permissions on every protected endpoint — full rules in `auth-patterns.md`
35. Rate limiting implementation (slowapi, Redis sliding window, headers) — full rules in `api-design.md`
