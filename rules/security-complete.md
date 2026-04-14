---
description: Security rules — OWASP API Top 10, injection, auth, headers, secrets, uploads, multi-tenant. Loads for all projects.
paths: ["**"]
---

# Security Rules — Complete (extends security.md)

> This file EXTENDS `security.md`. Do not repeat rules already there.
> Ownership: BOLA/IDOR, JWT algorithm attacks, SSRF deep, CORS deep, BFLA, API inventory,
>   security headers complete, secret management, prompt injection (direct + indirect),
>   rate limiting, file uploads, dependency security, audit logging, data encryption,
>   PostgreSQL RLS, DoS protection, auth edge cases.

---

## BOLA / IDOR — Object Ownership

1. NEVER check existence without checking ownership — a record that exists but belongs to someone else must be treated identically to a missing record.

2. ALWAYS filter queries with `owner_id = current_user.id` (or `tenant_id`) as a WHERE clause — never fetch by ID alone and check ownership afterward in Python; the DB must do the filtering.

3. ALWAYS return `404 Not Found` when a user accesses a resource they do not own — returning `403 Forbidden` tells the attacker the record exists and they should keep probing.

4. NEVER construct the ownership check as a post-fetch Python `if` — if you fetch the record first and then check in code, the unauthorized read has already happened.

5. ALWAYS implement ownership verification as a dependency (or middleware equivalent) so it cannot be accidentally omitted from new routes.

---

## Mass Assignment

6. ALWAYS define `extra="forbid"` (Pydantic) or equivalent on every input schema — this rejects unknown fields with a 422 rather than silently ignoring them.

7. NEVER accept these fields from user input under any circumstances: `id`, `owner_id`, `user_id`, `tenant_id`, `role`, `is_admin`, `is_superuser`, `created_at`, `updated_at`, `deleted_at`.

8. ALWAYS use a dedicated `Create` schema and a dedicated `Update` schema — the `Update` schema allows a strict subset of fields compared to the `Create` schema, and neither includes system-controlled fields.

9. NEVER use bulk-update patterns that map all input keys directly to model attributes (e.g. `model.update(**request.model_dump())`).

---

## SQL Injection — Deep Patterns

10. ALWAYS use an explicit allowlist for sort column names — store valid columns in a `frozenset`, reject anything not in it with a 422 before it reaches the ORM.

11. NEVER pass sort direction (ASC/DESC) as a raw string — map the user's input to an ORM attribute or literal after allowlist validation, never string-interpolate it into SQL.

12. ALWAYS parameterize queries even when the input came from your own database — second-order injection stores a payload safely and injects it at query time later.

13. NEVER use raw SQL string builders with f-strings or `.format()` — use parameterized queries (`text("... :param").bindparams(param=value)`) exclusively.

14. NEVER dynamically construct `IN (...)` clauses with string joining — use your ORM's `column.in_(list_of_values)` which generates parameterized placeholders.

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

23. NEVER trust the `kid` (Key ID) JWT header parameter as safe input — validate it against a hardcoded allowlist of known key IDs before any lookup; never pass the raw `kid` value into a file path or SQL query (path traversal and SQL injection via `kid` are a documented attack class).

24. NEVER use the `jku` (JSON Web Key Set URL) or `x5u` (X.509 URL) JWT header parameters unless you enforce a strict per-service allowlist of permitted key-hosting domains validated at startup — an attacker can replace these headers to point to their own JWKS endpoint, causing your server to verify tokens signed with an attacker-controlled key (active CVEs in 2025: CVE-2025-4692, CVE-2025-30144).

---

## SSRF Prevention — Deep

25. ALWAYS resolve the hostname to an IP and check the IP against RFC1918 ranges — checking only the domain name is bypassable with DNS rebinding.

26. NEVER follow redirects in server-side HTTP requests to user-supplied URLs — an attacker can redirect through a public URL to an internal address. Set `follow_redirects=False`.

27. ALWAYS block the AWS instance metadata endpoint `169.254.169.254` — add it to your private IP check; it is a link-local address (`ip.is_link_local`) and RFC1918 checks may miss it depending on the library.

28. ALWAYS check `ip.is_private`, `ip.is_loopback`, `ip.is_link_local`, and `ip.is_reserved` — all four conditions must be blocked, not just `is_private`.

29. ALWAYS validate the URL scheme before resolving — reject anything that is not `https`; block `file://`, `ftp://`, `gopher://`, and `dict://`.

30. PREFER an explicit domain allowlist over a blocklist — blocklists have gaps; allowlists are closed by default.

31. ALWAYS strip these headers from any outbound request your server makes on behalf of user-supplied input: `X-aws-ec2-metadata-token-ttl-seconds`, `X-aws-ec2-metadata-token`, and `Metadata: true` (Azure IMDS) — an attacker who injects these headers can defeat IMDSv2 protections even when `169.254.169.254` is blocked by IP.

---

## CORS — Deep Configuration

32. NEVER echo back the request `Origin` header as `Access-Control-Allow-Origin` — dynamic origin reflection grants the same access as a wildcard while appearing specific. Validate against a hardcoded allowlist.

33. NEVER load the CORS allowlist from a database at request time — an attacker who controls the DB controls your CORS policy. The allowlist must be in environment config at startup.

34. NEVER set `allow_credentials=True` with `allow_origins=["*"]` — browsers block this combination but some server implementations bypass it via reflection; defense must be server-side.

35. ALWAYS set `max_age` on your CORS middleware to reduce preflight request volume — 600 seconds (10 minutes) is a reasonable production default.

36. ALWAYS keep separate `ALLOWED_ORIGINS` lists for development and production — never add `localhost` origins to the production list.

---

## Broken Function Level Authorization (BFLA)

37. ALWAYS attach an explicit role/permission check to every non-GET endpoint — never rely on URL path prefix (`/admin/`) as the sole access control; prefix obscurity is not authorization.

38. ALWAYS reject `X-HTTP-Method-Override` headers at the middleware layer unless your application has a documented reason to support them — these headers allow a POST to masquerade as a DELETE or PUT, bypassing method-level access controls.

39. NEVER grant access based on a field in the request body that should be admin-only — use server-side role checks, not client-supplied role flags.

---

## API Inventory and Lifecycle

40. ALWAYS generate an OpenAPI schema at startup and fail CI if it diverges from the committed spec — shadow APIs (undocumented endpoints deployed without security review) are OWASP API9:2023 and go unmonitored and unpatched.

41. ALWAYS set a `Sunset` response header (RFC 8594) on any deprecated endpoint, indicating the removal date — clients must be informed before removal; this forces you to track which endpoints are scheduled for decommissioning.

42. ALWAYS remove deprecated API versions from routing when a new major version ships — not just from documentation; zombie endpoints (old versions still reachable in production) are unmonitored and unpatched.

---

## Unsafe Third-Party API Consumption

43. ALWAYS validate third-party API responses with Pydantic schemas before using the data — external API responses are untrusted input; treat them identically to user input from the web.

44. NEVER propagate a redirect from a third-party API response directly to your client — validate any URL in a third-party response body against the same SSRF blocklist you apply to user input.

45. ALWAYS set both connect and read timeouts on third-party API calls — a slow or malicious upstream that never responds will hold your connection pool; missing timeouts cause cascading failures.

---

## Security Headers — Complete Set

> Note: `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY` are in `security.md` rules 24-25.
> This section covers the remaining required headers.

46. ALWAYS set `Strict-Transport-Security: max-age=31536000; includeSubDomains` — 1 year is the minimum; add `preload` only after verifying all subdomains support HTTPS.

47. ALWAYS set a `Content-Security-Policy` header — start with `default-src 'self'` and open up only what your frontend requires; CSP is the primary defense against XSS after input validation fails.

48. NEVER set `X-XSS-Protection: 1` or `X-XSS-Protection: 1; mode=block` — set `0` instead; the old browser XSS auditor has known bypass exploits and has been removed from modern browsers; your CSP handles this now.

49. ALWAYS set `Referrer-Policy: strict-origin-when-cross-origin` — this sends the full path on same-origin requests (for analytics) but only the origin on cross-origin requests (no path leakage).

50. ALWAYS set `Permissions-Policy` to disable browser features your app does not use — at minimum disable `geolocation=(), microphone=(), camera=(), payment=()`.

51. NEVER allow the `Server` header to reveal your server software version — strip or replace it in your reverse proxy configuration.

---

## Secret Management — Deep

52. ALWAYS use `pydantic.SecretStr` or equivalent for any field that holds a credential — `SecretStr` renders as `**********` in `str()`, `repr()`, and logging; the actual value requires `.get_secret_value()`.

53. NEVER call `.get_secret_value()` outside the specific function that needs the credential — do not unwrap and store it in a variable that gets passed around.

54. ALWAYS generate secrets with `secrets.token_urlsafe(32)` — never use `random`, `uuid4`, or `os.urandom` for security tokens; `secrets` is the stdlib module designed for this.

55. ALWAYS have a key rotation procedure documented and tested before production — rotation without a plan means a compromised key cannot be invalidated quickly.

56. NEVER rotate a secret by replacing it immediately — use the `CURRENT` + `PREVIOUS` pattern: deploy with both, wait for old token TTL, then remove the old key.

57. ALWAYS use a secret manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) in production for secrets that change — do not manage rotation manually via `.env` files for production systems.

58. ALWAYS run `gitleaks` or `trufflehog` as a pre-commit hook AND as a CI step that blocks merge — scan full git history (`--all`) on first install; 23 million hardcoded secrets were added to public GitHub repos in 2024 (+25% YoY).

---

## Prompt Injection

59. ALWAYS wrap user-supplied content in explicit delimiters when injecting into LLM prompts — use `<user_message>...</user_message>` and instruct the model that content inside those tags is data, not instructions.

60. NEVER inject raw user input directly into an LLM prompt at any position — always delimit, always instruct the model about the boundary.

61. ALWAYS validate LLM output with Pydantic (or equivalent schema) before saving it to a database or taking any action based on it — LLM output is untrusted input.

62. NEVER expose the full LLM error or raw output in API responses — log it with `exc_info=True`, return a generic error message.

63. ALWAYS dispatch tool calls through an explicit allowlist — never use `getattr(module, tool_name)()` or `eval()` to dispatch LLM-requested tool calls; match against a known set of safe callables.

64. ALWAYS validate tool arguments with Pydantic schemas before executing — an injection can manipulate tool arguments even if the tool name is on the allowlist.

65. ALWAYS delimit and label retrieved content separately from user instructions in RAG pipelines — use `<retrieved_document source="...">...</retrieved_document>` tags; retrieved documents can contain indirect injection payloads (PoisonedRAG attack: 5 poisoned docs in a corpus of millions can control LLM output 90% of the time for targeted queries). NEVER allow retrieved content to appear in the system prompt position.

---

## Rate Limiting — Layered

66. ALWAYS implement at least two layers of rate limiting: edge (CDN/nginx) and application — edge stops volumetric attacks before they reach your app; app-level stops authenticated abuse.

67. ALWAYS add a third business-logic rate limit for expensive operations (LLM calls, password resets, account creation, data exports) separately from the general API rate limit.

68. ALWAYS return `Retry-After`, `X-RateLimit-Limit`, and `X-RateLimit-Remaining` headers with `429` responses — clients need to know the limit, how much remains, and when to retry.

69. NEVER rate limit only by IP address for authenticated endpoints — rate limit by `user_id` for authenticated routes; IP-based limits are trivially bypassed with proxies and hurt shared NAT users.

70. ALWAYS use a sliding window algorithm (not fixed window) for rate limiting, and always set a TTL on every Redis rate limit key — fixed windows allow 2x burst attacks at window boundaries; missing TTLs cause unbounded memory growth.

---

## File Upload Security

71. NEVER trust file extensions to determine file type — extensions are user-controlled; always read the first 2048 bytes and use a magic bytes library (`python-magic`) to identify the actual MIME type, then enforce an explicit MIME type allowlist.

72. ALWAYS rename uploaded files to a UUID before storage and enforce a per-file size limit before buffering — the original filename can contain path traversal sequences, null bytes, or shell metacharacters; checking `Content-Length` before buffering prevents memory exhaustion.

73. NEVER store uploaded files on the application server's local filesystem — use object storage (S3, GCS, Azure Blob); local storage does not scale and the app server should be stateless.

74. ALWAYS serve uploaded files via presigned URLs with short TTLs — never construct a direct public URL to a file; presigned URLs enforce authentication and expiry at the storage layer.

75. NEVER serve user-uploaded files from the same origin as your API — a malicious HTML or JavaScript file served from your origin has the same-origin permissions as your API.

76. ALWAYS quarantine uploads before serving them — upload to a quarantine prefix, run virus scan (ClamAV, AWS GuardDuty Malware Protection), move to the clean prefix only on pass.

77. ALWAYS check the compression ratio and member count before extracting any archive — reject files where (uncompressed / compressed size) > 100x or member count > a hardcoded limit; decompression bombs (e.g. 42 KB zip expanding to 4.5 GB) bypass file size checks on the uploaded file. Use Python 3.12.3+ which patches CVE-2024-0450.

---

## Dependency Security

78. ALWAYS pin exact versions in application lock files — `fastapi==0.115.6` not `fastapi>=0.115`; unpinned deps silently pick up malicious updates.

79. ALWAYS generate hash-pinned requirements for production deploys — `pip-compile --generate-hashes` produces a `requirements.txt` where each package hash is verified at install time.

80. ALWAYS run `pip-audit` and `safety` in CI — they check different vulnerability databases (OSV and Safety DB respectively); use both.

81. NEVER install packages with names that differ by one character from well-known packages without verifying the author — typosquatting attacks use `requets`, `fastap1`, `sqlachemy`.

82. ALWAYS use Renovate Bot or Dependabot with `pip-audit` on the PR — automated dependency updates must include an automated vulnerability check on the update itself.

---

## Audit Logging — Complete

83. ALWAYS log audit events in the same database transaction as the action they record — if the action rolls back, the audit entry rolls back too; never have phantom audit entries for actions that did not complete.

84. ALWAYS make audit tables append-only at the ORM level — add `before_update` and `before_delete` listeners that raise a `RuntimeError` to prevent any modification.

85. NEVER log these in any log entry or audit record: plaintext passwords, password hashes, JWT tokens, session IDs, API keys, full credit card numbers, SSNs, `Authorization` headers, request bodies for `/login` or `/register` endpoints.

86. ALWAYS include these contextual identifiers in every log entry: `user_id`, `tenant_id` (if multi-tenant), `resource_type`, `resource_id`, `ip_address`, and `event` enum value.

87. ALWAYS use a typed `SecurityEvent` enum (not free-form strings) for audit event types — enumerated events can be searched, graphed, and alerted on reliably.

---

## Data Encryption — Field-Level and Envelope

88. ALWAYS use `cryptography.fernet.Fernet` for field-level encryption — do not implement your own encryption; Fernet provides AES-128-CBC with HMAC-SHA256 and handles IV generation.

89. ALWAYS use `MultiFernet` with two keys during rotation — `MultiFernet([new_key, old_key])` encrypts with `new_key` and decrypts with either; this enables zero-downtime key rotation.

90. NEVER store encryption keys in the database alongside the encrypted data — the key and the ciphertext must be separated; store keys in a secret manager or environment variable.

91. ALWAYS use envelope encryption for large datasets or KMS integration — generate a DEK (data encryption key) per record or per batch, encrypt the DEK with a KEK (key encryption key) from KMS, store the encrypted DEK with the data.

92. NEVER encrypt columns that are used as WHERE clause filters without a separate lookup index — Fernet ciphertext is not searchable; encrypt only columns that are retrieved but not filtered on, or maintain a separate HMAC index for searchable encryption.

---

## PostgreSQL Row-Level Security

93. ALWAYS use `ALTER TABLE ... FORCE ROW LEVEL SECURITY` in addition to `ENABLE ROW LEVEL SECURITY` — without `FORCE`, the table owner bypasses RLS; with `FORCE`, even the owner is subject to the policy.

94. ALWAYS set the per-transaction tenant context via `set_config('app.current_tenant_id', :tid, true)` — the third argument `true` means local to the transaction; it resets automatically on commit or rollback.

95. NEVER set `set_config` with `false` (session-scoped) in a connection pool — session-scoped settings persist across requests in a pooled connection; always use transaction-scoped (`true`).

96. ALWAYS namespace every shared cache (Redis, Memcached) key with `tenant_id` and `user_id` — a cache miss that falls through to the DB is safe; a cache hit that returns another tenant's data is a breach.

97. NEVER cache data under a key derived from user-supplied input alone — always include server-side identifiers (tenant_id, user_id) to prevent cross-tenant cache poisoning.

---

## DoS Protection

98. ALWAYS set a maximum request body size and enforce a per-request timeout in middleware — check `Content-Length` before buffering to reject oversized requests; wrap each request handler in an async timeout so slow DB queries or external calls cannot hold a connection open indefinitely.

99. NEVER run your application server directly in production without a reverse proxy — nginx or Caddy handles TLS termination, Slowloris defense, and connection timeouts; configure keep-alive and concurrency limits at both layers.

100. ALWAYS set `client_header_timeout`, `client_body_timeout`, and `send_timeout` in nginx — these values prevent Slowloris attacks by closing connections that do not complete their HTTP exchange within the time limits.

---

## Authentication Edge Cases

101. ALWAYS generate password reset tokens with `secrets.token_urlsafe(32)`, set expiry to ≤15 minutes, invalidate the token on first use (single-use), and compare with `hmac.compare_digest()` — NEVER reuse the same token if the user requests a second reset; invalidate the prior token before issuing a new one.

102. ALWAYS return identical response bodies and HTTP status codes for "user not found" vs "wrong password" on login endpoints — different messages or timing reveals valid usernames to attackers. Perform the same computational work (e.g., a bcrypt dummy hash) regardless of whether the account exists.

103. NEVER return `404` for "email not registered" on password reset endpoints — always return a generic "if this email exists, you'll receive a link" message to prevent account enumeration.

104. NEVER issue a full session token until ALL authentication factors are verified — use a short-lived, capability-limited "pending MFA" token between credential verification and OTP verification, and exchange it for a full session only on OTP success; pre-issuing a full session before 2FA verification allows bypassing the second factor entirely.

105. ALWAYS use `SameSite=Strict` for session cookies on APIs serving a decoupled SPA on a different origin — `SameSite=Lax` still sends the cookie on top-level navigations (e.g., cross-site link clicks that trigger GET requests); `Strict` prevents this; reserve `Lax` only when the application requires cookie delivery on incoming navigations from external links, and document the reason.

---

## Password Hashing

106. ALWAYS use Argon2id for new password hashing implementations — minimum config: 64 MiB memory, 3 iterations, 1 degree of parallelism (`argon2-cffi` in Python). bcrypt silently truncates passwords beyond 72 bytes, making it unsuitable for modern auth; use Argon2id as the OWASP 2025 mandatory default.

107. IF bcrypt is already deployed and migration is not feasible, enforce a 72-byte maximum password length at the input validation layer and set work factor >= 12 — the 72-byte truncation is silent and allows passwords like "correct horse battery staple..." to match any suffix.

---

## OAuth2 / PKCE Security

108. ALWAYS require PKCE (RFC 7636) for all authorization code flows — use `code_challenge_method=S256` only; never accept `plain`. Never use the implicit flow; it exposes tokens in the browser URL fragment.

109. ALWAYS validate `redirect_uri` with exact string matching against a pre-registered allowlist — never use prefix matching or wildcard subdomains; one open redirect in the allowlist compromises the entire OAuth flow (RFC 9700).

110. ALWAYS generate the `state` parameter with `secrets.token_urlsafe(32)` and bind it to the user session before the authorization redirect — verify it on callback before exchanging the code; a missing or guessable `state` enables CSRF against the OAuth flow.

---

## WebSocket Security

111. ALWAYS validate the `Origin` header during the WebSocket upgrade handshake and reject connections from origins not on your CORS allowlist — WebSocket upgrades are sent with cookies but without CORS preflight, making them vulnerable to cross-site WebSocket hijacking (CSWSH).

112. ALWAYS validate a token (JWT or CSRF token) passed as a query parameter or `Sec-WebSocket-Protocol` subprotocol header during the upgrade — never rely on cookie authentication alone for WebSocket connections.

113. ALWAYS use `wss://` (TLS) exclusively in production; never `ws://` — unencrypted WebSocket traffic exposes session tokens and message payloads to network interception.

---

## GraphQL Security

114. ALWAYS disable GraphQL introspection in production — it exposes your full schema and field inventory to attackers; enable it only in development environments behind authentication.

115. ALWAYS enforce maximum query depth and complexity limits before execution — a single HTTP request can batch thousands of nested GraphQL operations; depth limit of 7 levels and complexity score of 1000 are common production defaults.

116. NEVER rate-limit GraphQL endpoints by HTTP request count alone — one request can contain hundreds of batched operations; rate-limit by operation count per request.

---

## Server-Side Template Injection (SSTI)

117. NEVER pass user-supplied input to `render_template_string()` or any equivalent that compiles a template at runtime from a string — always use static template files on disk; Jinja2 SSTI via `__class__.__mro__` gadget chains leads directly to RCE.

118. IF user-customizable templates are a product requirement, render them in `jinja2.sandbox.SandboxedEnvironment()` with an explicit whitelist of globals and filters — the default `Environment()` gives attackers full Python object traversal.

---

## Business Logic Race Conditions

119. ALWAYS use `SELECT ... FOR UPDATE` (pessimistic locking) or optimistic locking with a version column for any operation that reads a value and writes based on it — balance deductions, inventory decrements, coupon redemptions, and rate limit counters are all vulnerable to TOCTOU (time-of-check-time-of-use) race conditions without this.

120. NEVER check a constraint in application code and then write separately — the check and write must be atomic within a single database transaction; two concurrent requests will both pass the check before either commits.

---

## Container Security

121. ALWAYS run the application as a non-root user in production containers — a root process in a container can escape to the host if the container runtime has a vulnerability; add a non-root user in the Dockerfile and switch to it before the `CMD`.

122. ALWAYS specify a non-latest, digest-pinned base image in production Dockerfiles — `FROM python:3.12.3-slim@sha256:...`; `latest` tags silently pull new images that may introduce vulnerabilities or supply-chain compromises.

123. NEVER mount the Docker socket into a container and NEVER use `--privileged` — mounting the socket gives the container full host control equivalent to root on the host machine.

---

## HTTP Request Smuggling

124. ALWAYS configure your reverse proxy to reject or normalize requests that contain both `Content-Length` and `Transfer-Encoding` headers — this is the root cause of CL.TE and TE.CL request smuggling attacks (CVE-2025-55315).

125. PREFER HTTP/2 end-to-end between load balancer and application server — HTTP/2 does not have the CL/TE ambiguity that makes HTTP/1.1 smuggling possible.

---

## TLS Configuration

126. ALWAYS configure your reverse proxy to reject TLS 1.0 and TLS 1.1 — set `ssl_protocols TLSv1.2 TLSv1.3;` in nginx; PCI-DSS 4.0 (effective April 2024) makes TLS 1.2 the hard minimum.

127. ALWAYS use strong cipher suites: prefer ECDHE+AESGCM and CHACHA20-POLY1305; disable RC4, DES, 3DES, and export-grade ciphers. Verify quarterly with `testssl.sh` — cipher regressions frequently appear after reverse proxy upgrades.

---

## Cryptographic Strength

128. NOTE on rule 88 — Fernet uses AES-128-CBC. For environments requiring AES-256 (FIPS 140-2/3, PCI-DSS), use `cryptography.hazmat.primitives.ciphers.aead.AESGCM` with a 32-byte key instead — AES-GCM provides authenticated encryption and is FIPS-approved. NEVER use AES-CBC without an explicit HMAC; CBC without MAC is vulnerable to padding oracle attacks.

129. ALWAYS generate an SBOM (`syft` or `cyclonedx-bom`) as a CI artifact on every production build — required for EO 14028 compliance and enables rapid triage when a new CVE drops; you can instantly determine if the affected package is in your build without scanning all repos manually.

130. ALWAYS audit DNS CNAMEs for dangling records before enabling HSTS `preload` — a CNAME pointing to an unclaimed S3 bucket, GitHub Pages, or Heroku app allows subdomain takeover; under HSTS preload, the attacker's endpoint is served with your domain's HTTPS trust. Run `subjack` or `nuclei -t takeovers/` in CI against your full DNS zone.

---

## Insecure Deserialization

131. NEVER call `pickle.loads()`, `marshal.loads()`, or `yaml.load()` on any data that crossed a trust boundary (network, file upload, user-supplied cache hit) — these execute arbitrary Python code on attacker-controlled input.

132. ALWAYS use `yaml.safe_load()` — never bare `yaml.load()`; the unsafe loader evaluates arbitrary Python constructors.

133. NEVER serialize Python objects to Redis or any shared cache with `pickle` — use JSON with an explicit schema instead; if pickle is unavoidable (e.g. scikit-learn models), load only from paths under version control, never from user-supplied input.

134. NEVER use `jsonpickle`, `dill`, or `shelve` with untrusted input — they are `pickle` wrappers with the same RCE attack surface.

---

## Code Injection

135. NEVER call `eval()`, `exec()`, `compile()`, or `__import__()` with any value derived from user input, environment variables from untrusted sources, or LLM output — use `ast.literal_eval()` for safe evaluation of Python literals (strings, numbers, lists, dicts, booleans, None) when you need to parse structured data.

---

## Logging Injection

136. ALWAYS sanitize user-supplied values before interpolating them into log messages — strip or replace `\n`, `\r`, and ANSI escape sequences (`\x1b[...m`) before logging; newline injection allows attackers to forge entirely new log entries.

137. ALWAYS use structured logging with separate key-value fields for user data — user input isolated to a single field value cannot forge additional fields; never build log messages by f-string concatenation with user data.

---

## Error Message Information Disclosure

138. ALWAYS register a global exception handler that catches all unhandled exceptions and returns only a generic error message with HTTP 500 — never propagate `traceback.format_exc()` or raw `str(exception)` to the client in production.

139. NEVER expose SQLAlchemy, asyncpg, or database error messages directly to API clients — catch DB exceptions at the boundary, log detail server-side with `exc_info=True`, return a generic message to the client; DB errors leak table names, column names, and query fragments.

140. ALWAYS set `debug=False` in production — `DEBUG=True` enables the interactive Starlette/Werkzeug debugger which allows arbitrary code execution via the debug console PIN.

141. NEVER include internal filesystem paths, module names, or line numbers in error responses — attackers use these to target specific vulnerable library versions.

---

## Security Misconfiguration

142. ALWAYS verify `DEBUG=False`, `TESTING=False`, and `ENVIRONMENT=production` in a startup health check that raises `RuntimeError` if misconfigured in a non-local environment — misconfigured debug mode is an RCE vector, not just information disclosure.

143. NEVER deploy with default credentials on any component: PostgreSQL (`postgres/postgres`), Redis (no auth), pgAdmin, Grafana (`admin/admin`), RabbitMQ (`guest/guest`) — automate a CI check that asserts connection failure with default credentials.

144. NEVER expose admin panels (`/admin`, `/flower`, `/pgadmin`, `/grafana`) on the public internet without a network-layer restriction (VPN, IP allowlist) — admin UIs bypass application-layer controls.

145. ALWAYS remove or gate `/docs` and `/redoc` FastAPI endpoints in production — interactive API docs allow unauthenticated attackers to explore and test your full API surface; disable with `app = FastAPI(docs_url=None, redoc_url=None)` or protect behind authentication middleware.

---

## Open Redirect — Deep Implementation

146. ALWAYS parse redirect URLs with `urllib.parse.urlparse()` and validate that `scheme` is `https` and `netloc` exactly matches an entry in your allowlist — never do string prefix matching.

147. ALWAYS normalize before validating: call `urllib.parse.unquote()` and `unicodedata.normalize('NFKC', url)` — double-encoded and Unicode-normalized variants bypass string-level allowlist checks.

148. ALWAYS reject redirect URLs where `netloc` contains `@` (authority confusion: `https://yoursite.com@evil.com`) or where the scheme is empty (protocol-relative: `//evil.com`).

---

## Cache Poisoning via Unkeyed Headers

149. NEVER use `X-Forwarded-Host`, `X-Original-Host`, or `X-Host` headers as inputs to cache keys, URL generation, or response bodies — attackers who control these headers control URL generation in your app; normalize to `SITE_URL` from environment config.

150. ALWAYS configure your reverse proxy to strip `X-Forwarded-Host` and `X-Original-Host` from incoming requests before they reach your application — reconstruct protocol from TLS state, not from `X-Forwarded-Proto`.

151. NEVER generate password reset links, OAuth `redirect_uri` values, or email confirmation links from `request.headers["host"]` — always use the `SITE_URL` env var; host header injection here produces phishing links pointing to an attacker's domain.

---

## Clickjacking — Complete

152. ALWAYS include `frame-ancestors 'none'` in your CSP header — this supersedes `X-Frame-Options: DENY` in modern browsers; keep both for legacy compatibility.

153. NEVER rely on `X-Frame-Options` alone — it is not honored consistently across all user agents; use `frame-ancestors` in CSP as the primary control.

---

## Mass Enumeration Protection

154. ALWAYS use random UUIDs (v4) or `secrets.token_urlsafe()` as resource IDs in URLs — sequential integer IDs and timestamp-ordered UUIDs (v1, v7) allow full resource enumeration by incrementing; UUID v4 makes the key space infeasible to brute-force.

155. NEVER expose a total record count in list responses unless required — `"total": 48291` tells a scraper exactly how many requests to make; use `"has_more": true/false` instead.

---

## Injection — XPATH and LDAP

156. NEVER interpolate user input into XPath expressions — use an XML library's variable binding mechanism (e.g. `lxml`'s `variables` parameter) rather than string concatenation; for SAML assertions, always validate signatures before attribute extraction to prevent SAML wrapping attacks.

157. NEVER interpolate user input into LDAP filter strings — escape all input with `ldap3.utils.conv.escape_filter_chars()` before it appears in a filter; ALWAYS disable referral chasing (`AUTO_REFERRALS=False`) to prevent LDAP referral injection redirecting auth to an attacker-controlled server.

---

## Multi-Tenancy Edge Cases

158. ALWAYS configure PgBouncer (or equivalent) in transaction-pooling mode (`pool_mode=transaction`) when using PostgreSQL RLS with `set_config` for tenant isolation — session-pooling mode allows a connection used by tenant A to be reused by tenant B before the next `set_config` call; any code path that omits `set_config` (error handlers, health checks, background tasks) inherits the prior tenant's identity.

---

## ReDoS Protection

159. NEVER compile user-supplied strings as regular expressions with `re.compile()` — use the `re2`/`google-re2` library (linear-time, no backtracking) for user-supplied patterns, or wrap `re.compile()` in a subprocess with a hard timeout; catastrophic backtracking on patterns like `(a+)+$` can freeze an event loop thread for minutes.

---

## XXE Prevention

160. NEVER parse XML with `lxml.etree` without an explicitly hardened parser: `lxml.etree.XMLParser(resolve_entities=False, no_network=True, huge_tree=False)` — the default parser resolves local file entities (XXE file read) and network entities (SSRF). For stdlib `xml.etree.ElementTree`, use the `defusedxml` drop-in replacement which disables entity expansion and DTD processing.

---

## HTTP Parameter Pollution

161. ALWAYS reject requests that supply the same security-sensitive query parameter more than once — check `len(request.query_params.getlist("param")) > 1` and return 422; duplicate parameters are used to bypass WAFs that inspect only the first or last occurrence while your app reads the other. Apply this to any parameter influencing access control, filtering, or pricing.

---

## DAST in CI

162. ALWAYS run a DAST scan (OWASP ZAP `zap-api-scan.py` or equivalent) against a live staging deployment as a blocking nightly CI job — SAST cannot detect runtime auth bypass, IDOR, or session fixation; configure ZAP with your OpenAPI schema so it exercises every documented endpoint; store scan reports as CI artifacts for audit trail.

---

## Incident Response — JWT Revocation

163. ALWAYS implement a `jti` revocation path before going to production: store a `jti` claim generated with `secrets.token_urlsafe(16)` in every issued JWT, maintain a Redis sorted set of revoked JTIs keyed by expiry timestamp, and check membership on every authenticated request; use `ZREMRANGEBYSCORE` on expiry to keep the set bounded.

164. ALWAYS document and test a kill-switch runbook: rotating the signing key invalidates ALL active tokens within one access token TTL; test this in staging before production launch. Refresh tokens MUST be stored server-side (database) to enable per-user revocation without key rotation.

---

## Browser Security — Cross-Origin Isolation

165. ALWAYS set `Cross-Origin-Opener-Policy: same-origin` on all document responses — severs the opener relationship between your pages and cross-origin popups, blocking Spectre-class side-channel leaks.

166. ALWAYS set `Cross-Origin-Resource-Policy: same-origin` on API responses — prevents cross-origin reads of sensitive JSON by other origins.

167. ALWAYS add a `integrity="sha384-..."` Subresource Integrity hash and `crossorigin="anonymous"` to every `<script>` or `<link>` tag loading from a CDN — SRI blocks execution if the CDN-hosted file is modified by a supply-chain attacker; generate hashes with `openssl dgst -sha384 -binary file.js | base64`.

---

## Secrets in Environment Variables

168. NEVER store secrets in `.env` files on persistent disk in production, in CI environment variable UIs as plaintext, in container image layers (`ENV` in Dockerfile), or in ECS task definition console — these are logged, cached, and visible in `docker inspect`. Environment variables are acceptable ONLY when injected at runtime by a secrets manager (Vault Agent Injector, AWS Secrets Manager CSI, Doppler operator) that writes them ephemerally and never to disk.

---

## Cryptographic Nonces

169. ALWAYS generate CSP nonces, CSRF tokens, WebSocket handshake tokens, and anti-replay nonces with `secrets.token_urlsafe(16)` (minimum 128 bits) — NEVER use `uuid.uuid4()` for security nonces; CSP nonces MUST be unique per HTTP response (not per session) and injected into both the `Content-Security-Policy` header and the corresponding `<script nonce="...">` tag simultaneously; a reused nonce defeats the CSP nonce mechanism.

---

## API Gateway vs Application Security Split

170. NEVER trust identity headers injected by an API gateway (`X-User-ID`, `X-Tenant-ID`, `X-Authenticated-User`) without verifying they originated from your own trusted gateway — the application MUST re-verify the JWT on every request regardless of gateway validation, OR MUST enforce network policy (firewall / Kubernetes NetworkPolicy) that allows traffic to the app port ONLY from the gateway's IP range; "gateway does auth, app trusts the header" without network-layer enforcement allows any internal-network attacker to forge identity.

---

## Timing Attacks — General

171. ALWAYS use `hmac.compare_digest(a, b)` for ALL security token comparisons — API keys, webhook signatures (`X-Hub-Signature-256`, `X-Stripe-Signature`), HMAC MACs, CSRF tokens — not only password reset tokens; Python's `==` short-circuits on the first differing byte, leaking token prefix length via timing. This is the single most commonly missed constant-time comparison case in production code.

---

## Path Traversal — Deep

172. ALWAYS call `os.path.realpath()` on any user-influenced file path and assert the resolved path starts with the expected base directory before opening the file — symlinks inside a permitted directory can point outside it, bypassing prefix checks on the original path string.

173. ALWAYS strip null bytes (`\x00`) from any user-supplied filename or path before processing — Python's `os.path` and the OS handle null bytes inconsistently across platforms, causing the apparent path and the actual path to diverge.

174. ALWAYS apply `unicodedata.normalize('NFKC', path)` and `urllib.parse.unquote(path)` before any path validation — overlong UTF-8 encodings (`..%c0%af`) and Unicode-normalized variants bypass string-level path checks.

---

## Supply Chain — Maintainer Takeover

175. ALWAYS verify package ownership hasn't changed before approving Dependabot/Renovate PRs for critical dependencies — check PyPI maintainer history for packages in your security path (auth, crypto, HTTP client); legitimate packages taken over by malicious maintainers (`event-stream`, `ua-parser-js` class) are distinct from typosquatting and not caught by name-similarity checks.

---

## Feature Flags for Security Controls

176. ALWAYS wrap new security controls (new auth middleware, rate limit changes, new encryption) in a feature flag so they can be disabled in production without a redeploy if they cause an outage — but ensure feature flag access requires the same privilege level as deploying code; a low-privilege actor disabling a security flag is a security misconfiguration.

177. ALWAYS run new security checks in logging-only (shadow) mode first — log violations without enforcing them, measure false-positive rate, then switch to enforcement; this prevents a misconfigured security control from becoming an outage.

---

## LLM Security — Model-Level Attacks

178. NEVER fine-tune a model on sensitive user data without evaluating membership inference risk first — an attacker can determine whether a specific record was in the training set by probing the model; prefer RAG over fine-tuning for sensitive data corpora.

179. ALWAYS scan LLM output for secret patterns (regex for API key formats, credit card numbers, SSNs) before returning responses to clients — models trained on internet data may verbatim reproduce memorized secrets; output scanning is a defense-in-depth layer before the response reaches the client.

---

## Content-Type and Polyglot Files

180. ALWAYS set an explicit, allowlisted `Content-Type` on every response that serves user-generated content — default to `Content-Disposition: attachment; filename="download"` and `Content-Type: application/octet-stream` for unknown types; forcing download prevents the browser from rendering attacker-controlled content as HTML or script.

181. NEVER rely on magic bytes alone to reject malicious uploads — polyglot files are simultaneously valid in two formats (e.g. a JPEG that is also valid JavaScript); run the uploaded content through a format-specific strict parser (not just a magic-byte check) and reject files that parse successfully as any non-allowed format. (`X-User-ID`, `X-Tenant-ID`, `X-Authenticated-User`) without verifying they originated from your own trusted gateway — the application MUST re-verify the JWT on every request regardless of gateway validation, OR MUST enforce network policy (firewall / Kubernetes NetworkPolicy) that allows traffic to the app port ONLY from the gateway's IP range; "gateway does auth, app trusts the header" without network-layer enforcement allows any internal-network attacker to forge identity. — escape all input with `ldap3.utils.conv.escape_filter_chars()` before it appears in a filter; ALWAYS disable referral chasing (`AUTO_REFERRALS=False`) to prevent LDAP referral injection redirecting auth to an attacker-controlled server. — a CNAME pointing to an unclaimed S3 bucket, GitHub Pages, or Heroku app allows subdomain takeover; under HSTS preload, the attacker's endpoint is served with your domain's HTTPS trust. Run `subjack` or `nuclei -t takeovers/` in CI against your full DNS zone. — `SameSite=Lax` still sends the cookie on top-level navigations (e.g., cross-site link clicks that trigger GET requests); `Strict` prevents this; reserve `Lax` only when the application requires cookie delivery on incoming navigations from external links, and document the reason.
