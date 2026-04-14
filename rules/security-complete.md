---
description: Security rules — OWASP API Top 10, injection, auth, headers, secrets, uploads, multi-tenant. Loads for all projects.
paths: ["**"]
---

# Security Rules — Complete (extends security.md)

> This file EXTENDS `security.md`. Do not repeat rules already there.
> Ownership: BOLA/IDOR, JWT algorithm, SSRF deep, CORS deep, security headers complete,
>   secret management, prompt injection, rate limiting, file uploads, dependency security,
>   audit logging complete, data encryption, PostgreSQL RLS, DoS protection.

---

## BOLA / IDOR — Object Ownership

1. NEVER check existence without checking ownership — a record that exists but belongs to someone else must be treated identically to a missing record.

2. ALWAYS filter queries with `owner_id = current_user.id` (or `tenant_id`) as a WHERE clause — never fetch by ID alone and check ownership afterward in Python; the DB must do the filtering.

3. ALWAYS return `404 Not Found` when a user accesses a resource they do not own — returning `403 Forbidden` tells the attacker the record exists and they should keep probing.

4. NEVER construct the ownership check as a post-fetch Python `if` — if you fetch the record first and then check in code, the unauthorized read has already happened.

5. ALWAYS implement ownership verification as a FastAPI dependency (or middleware equivalent) so it cannot be accidentally omitted from new routes.

---

## Mass Assignment

6. ALWAYS define `model_config = ConfigDict(extra="forbid")` on every Pydantic input schema — this rejects unknown fields with a 422 rather than silently ignoring them.

7. NEVER accept these fields from user input under any circumstances: `id`, `owner_id`, `user_id`, `tenant_id`, `role`, `is_admin`, `is_superuser`, `created_at`, `updated_at`, `deleted_at`.

8. ALWAYS use a dedicated `Create` schema and a dedicated `Update` schema — the `Update` schema allows a strict subset of fields compared to the `Create` schema, and neither includes system-controlled fields.

9. NEVER use `model.update(**request.model_dump())` or equivalent ORM bulk-update patterns that map all input keys directly to model attributes.

---

## SQL Injection — Deep Patterns

10. ALWAYS use an explicit allowlist for sort column names — store valid columns in a `frozenset`, reject anything not in it with a 422 before it reaches the ORM.

11. NEVER pass sort direction (ASC/DESC) as a raw string — map the user's input to an ORM attribute or a `literal_column` after allowlist validation, never string-interpolate it into SQL.

12. ALWAYS parameterize queries even when the input came from your own database — second-order injection stores a payload safely and injects it at query time later. Parameterize every query regardless of data source.

13. NEVER use `text()` with f-strings or `.format()` — use `text("... :param").bindparams(param=value)` exclusively.

14. NEVER dynamically construct `IN (...)` clauses with string joining — use SQLAlchemy's `column.in_(list_of_values)` which generates parameterized placeholders.

---

## JWT Algorithm Security

15. ALWAYS specify the algorithm explicitly in `jwt.decode(..., algorithms=["RS256"])` — never omit the `algorithms` parameter; libraries that accept all algorithms by default will accept `alg:none`.

16. NEVER accept more than one algorithm family — pick RS256 (asymmetric) or HS256 (symmetric) for your service and whitelist only that one. Do not accept both.

17. NEVER accept `"alg": "none"` — this must be explicitly rejected; some libraries require `options={"verify_signature": True}` to be set; check your library's defaults.

18. NEVER use `python-jose` without auditing CVE-2024-37568 — prefer `PyJWT` which provides explicit algorithm whitelisting by default.

19. ALWAYS use `SecretStr` or equivalent to store JWT signing secrets — the secret must never appear in logs, `repr()`, or `str()` output.

20. ALWAYS require `exp`, `sub`, and `iat` claims in decoded tokens — use `options={"require": ["exp", "sub", "iat"]}` in PyJWT.

21. NEVER store sensitive data in JWT payloads — the payload is base64-encoded, not encrypted. Store only `sub`, `role`, and `exp`.

22. ALWAYS use at least 32 bytes of cryptographic randomness for HS256 secrets — generate with `secrets.token_urlsafe(32)`, never with a human-readable passphrase.

---

## SSRF Prevention — Deep

23. ALWAYS resolve the hostname to an IP and check the IP against RFC1918 ranges — checking only the domain name is bypassable with DNS rebinding.

24. NEVER follow redirects in server-side HTTP requests to user-supplied URLs — an attacker can redirect through a public URL to an internal address. Set `follow_redirects=False`.

25. ALWAYS block the AWS instance metadata endpoint `169.254.169.254` — add it to your private IP check; it is a link-local address (`ip.is_link_local`) and RFC1918 checks may miss it depending on the library.

26. ALWAYS check `ip.is_private`, `ip.is_loopback`, `ip.is_link_local`, and `ip.is_reserved` — all four conditions must be blocked, not just `is_private`.

27. ALWAYS validate the URL scheme before resolving — reject anything that is not `https`; block `file://`, `ftp://`, `gopher://`, and `dict://`.

28. PREFER an explicit domain allowlist over a blocklist — blocklists have gaps; allowlists are closed by default.

---

## CORS — Deep Configuration

29. NEVER echo back the request `Origin` header as `Access-Control-Allow-Origin` — dynamic origin reflection grants the same access as a wildcard while appearing specific. Validate against a hardcoded allowlist.

30. NEVER load the CORS allowlist from a database at request time — an attacker who controls the DB controls your CORS policy. The allowlist must be in environment config at startup.

31. NEVER set `allow_credentials=True` with `allow_origins=["*"]` — browsers block this combination but some server implementations bypass it via reflection; defense must be server-side.

32. ALWAYS set `max_age` on your CORS middleware to reduce preflight request volume — 600 seconds (10 minutes) is a reasonable production default.

33. ALWAYS keep separate `ALLOWED_ORIGINS` lists for development and production — never add `localhost` origins to the production list.

---

## Security Headers — Complete Set

34. ALWAYS set `Strict-Transport-Security: max-age=31536000; includeSubDomains` — 1 year is the minimum; add `preload` only after verifying all subdomains support HTTPS.

35. ALWAYS set a `Content-Security-Policy` header — start with `default-src 'self'` and open up only what your frontend requires; CSP is the primary defense against XSS after input validation fails.

36. NEVER set `X-XSS-Protection: 1` or `X-XSS-Protection: 1; mode=block` — set `0` instead; the old browser XSS auditor has known bypass exploits and has been removed from modern browsers; your CSP handles this now.

37. ALWAYS set `Referrer-Policy: strict-origin-when-cross-origin` — this sends the full path on same-origin requests (for analytics) but only the origin on cross-origin requests (no path leakage).

38. ALWAYS set `Permissions-Policy` to disable browser features your app does not use — at minimum disable `geolocation=(), microphone=(), camera=(), payment=()`.

39. NEVER allow the `Server` header to reveal your server software version — strip or replace it in your reverse proxy configuration.

---

## Secret Management — Deep

40. ALWAYS use `pydantic.SecretStr` for any field that holds a credential — `SecretStr` renders as `**********` in `str()`, `repr()`, `json()`, and logging; the actual value requires `.get_secret_value()`.

41. NEVER call `.get_secret_value()` outside the specific function that needs to use the credential — do not unwrap and store it in a variable that gets passed around.

42. ALWAYS generate secrets with `secrets.token_urlsafe(32)` — never use `random`, `uuid4`, or `os.urandom` for security tokens; `secrets` is the stdlib module designed for this.

43. ALWAYS have a key rotation procedure documented and tested before production — rotation without a plan means a compromised key cannot be invalidated quickly.

44. NEVER rotate a secret by replacing it immediately — use the `CURRENT` + `PREVIOUS` pattern: deploy with both, wait for old token TTL, then remove the old key.

45. ALWAYS use a secret manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) in production for secrets that change — do not manage rotation manually via `.env` files for production systems.

---

## Prompt Injection

46. ALWAYS wrap user-supplied content in explicit delimiters when injecting into LLM prompts — use `<user_message>...</user_message>` and instruct the model that content inside those tags is data, not instructions.

47. NEVER inject raw user input directly into an LLM prompt at any position — always delimit, always instruct the model about the boundary.

48. ALWAYS validate LLM output with Pydantic (or equivalent schema) before saving it to a database or taking any action based on it — LLM output is untrusted input.

49. NEVER expose the full LLM error or raw output in API responses — log it with `exc_info=True`, return a generic error message.

50. ALWAYS dispatch tool calls through an explicit allowlist — never use `getattr(module, tool_name)()` or `eval()` to dispatch LLM-requested tool calls; match against a known set of safe callables.

51. ALWAYS validate tool arguments with Pydantic schemas before executing — an injection can manipulate tool arguments even if the tool name is on the allowlist.

---

## Rate Limiting — Layered

52. ALWAYS implement at least two layers of rate limiting: edge (CDN/nginx) and application — edge stops volumetric attacks before they reach your app; app-level stops authenticated abuse.

53. ALWAYS add a third business-logic rate limit for expensive operations (LLM calls, password resets, account creation, data exports) separately from the general API rate limit.

54. ALWAYS return `Retry-After`, `X-RateLimit-Limit`, and `X-RateLimit-Remaining` headers with `429` responses — clients need to know the limit, how much remains, and when to retry.

55. NEVER rate limit only by IP address for authenticated endpoints — rate limit by `user_id` for authenticated routes; IP-based limits are trivially bypassed with proxies and hurt shared NAT users.

56. ALWAYS use a sliding window algorithm (not fixed window) for rate limiting, and always set a TTL on every Redis rate limit key — fixed windows allow 2x burst attacks at window boundaries; missing TTLs cause unbounded memory growth.

---

## File Upload Security

59. NEVER trust file extensions to determine file type — extensions are user-controlled; always read the first 2048 bytes and use a magic bytes library (`python-magic`) to identify the actual MIME type, then enforce an explicit MIME type allowlist.

60. ALWAYS rename uploaded files to a UUID before storage and enforce a per-file size limit before buffering — the original filename can contain path traversal sequences, null bytes, or shell metacharacters; checking `Content-Length` before buffering prevents memory exhaustion.

61. NEVER store uploaded files on the application server's local filesystem — use object storage (S3, GCS, Azure Blob); local storage does not scale and the app server should be stateless.

62. ALWAYS serve uploaded files via presigned URLs with short TTLs — never construct a direct public URL to a file; presigned URLs enforce authentication and expiry at the storage layer.

63. NEVER serve user-uploaded files from the same origin as your API — a malicious HTML or JavaScript file served from your origin has the same-origin permissions as your API.

64. ALWAYS quarantine uploads before serving them — upload to a quarantine prefix, run virus scan (ClamAV, AWS GuardDuty Malware Protection), move to the clean prefix only on pass.

---

## Dependency Security

66. ALWAYS pin exact versions in application `requirements.txt` or `pyproject.toml` — `fastapi==0.115.6` not `fastapi>=0.115`; unpinned deps silently pick up malicious updates.

67. ALWAYS generate hash-pinned requirements for production deploys — `pip-compile --generate-hashes` produces a `requirements.txt` where each package hash is verified at install time.

68. ALWAYS run `pip-audit` and `safety` in CI — they check different vulnerability databases (OSV and Safety DB respectively); use both.

69. NEVER install packages with names that differ by one character from well-known packages without verifying the author — typosquatting attacks use `requets`, `fastap1`, `sqlachemy`.

70. ALWAYS use Renovate Bot or Dependabot with `pip-audit` on the PR — automated dependency updates must include an automated vulnerability check on the update itself.

---

## Audit Logging — Complete

71. ALWAYS log audit events in the same database transaction as the action they record — if the action rolls back, the audit entry rolls back too; never have phantom audit entries for actions that did not complete.

72. ALWAYS make audit tables append-only at the ORM level — add `before_update` and `before_delete` listeners that raise a `RuntimeError` to prevent any modification.

73. NEVER log these in any log entry or audit record: plaintext passwords, password hashes, JWT tokens, session IDs, API keys, full credit card numbers, SSNs, `Authorization` headers, request bodies for `/login` or `/register` endpoints.

74. ALWAYS include these contextual identifiers in every log entry: `user_id`, `tenant_id` (if multi-tenant), `resource_type`, `resource_id`, `ip_address`, and `event` enum value.

75. ALWAYS use a typed `SecurityEvent` enum (not free-form strings) for audit event types — enumerated events can be searched, graphed, and alerted on reliably.

---

## Data Encryption — Field-Level and Envelope

76. ALWAYS use `cryptography.fernet.Fernet` for field-level encryption — do not implement your own encryption; Fernet provides AES-128-CBC with HMAC-SHA256 and handles IV generation.

77. ALWAYS use `MultiFernet` with two keys during rotation — `MultiFernet([new_key, old_key])` encrypts with `new_key` and decrypts with either; this enables zero-downtime key rotation.

78. NEVER store encryption keys in the database alongside the encrypted data — the key and the ciphertext must be separated; store keys in a secret manager or environment variable.

79. ALWAYS use envelope encryption for large datasets or KMS integration — generate a DEK (data encryption key) per record or per batch, encrypt the DEK with a KEK (key encryption key) from KMS, store the encrypted DEK with the data.

80. NEVER encrypt columns that are used as WHERE clause filters without a separate lookup index — Fernet ciphertext is not searchable; encrypt only columns that are retrieved but not filtered on, or maintain a separate HMAC index for searchable encryption.

---

## PostgreSQL Row-Level Security

81. ALWAYS use `ALTER TABLE ... FORCE ROW LEVEL SECURITY` in addition to `ENABLE ROW LEVEL SECURITY` — without `FORCE`, the table owner bypasses RLS; with `FORCE`, even the owner is subject to the policy.

82. ALWAYS set the per-transaction tenant context via `set_config('app.current_tenant_id', :tid, true)` — the third argument `true` means local to the transaction; it resets automatically on commit or rollback.

83. NEVER set `set_config` with `false` (session-scoped) in a connection pool — session-scoped settings persist across requests in a pooled connection; always use transaction-scoped (`true`).

84. ALWAYS namespace every shared cache (Redis, Memcached) key with `tenant_id` and `user_id` — a cache miss that falls through to the DB is safe; a cache hit that returns another tenant's data is a breach.

85. NEVER cache data under a key derived from user-supplied input alone — always include server-side identifiers (tenant_id, user_id) to prevent cross-tenant cache poisoning.

---

## DoS Protection

86. ALWAYS set a maximum request body size and enforce a per-request timeout in middleware — check `Content-Length` before buffering to reject oversized requests; wrap each request handler in `asyncio.wait_for(call_next(request), timeout=30.0)` so slow DB queries or external calls cannot hold a connection open indefinitely.

87. NEVER run Uvicorn directly in production without a reverse proxy — nginx or Caddy handles TLS termination, Slowloris defense, and connection timeouts; always configure `--timeout-keep-alive`, `--limit-concurrency`, and `--backlog` on Uvicorn, and set `client_header_timeout`, `client_body_timeout`, and `send_timeout` in the reverse proxy.

88. ALWAYS set `client_header_timeout`, `client_body_timeout`, and `send_timeout` in nginx — these values prevent Slowloris attacks by closing connections that do not complete their HTTP exchange within the time limits.
