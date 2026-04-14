---
description: Authentication + RBAC rules. Loads for all FastAPI projects with auth.
paths: ["*.py", "main.py", "auth.py", "app/**/*.py"]
---

# Auth Rules

> Credentials/secrets from env → see `security.md`
> Logging what not to log → see `security.md`
> 401 vs 403 status codes → see `http-contracts.md`
> 201 + Location on register, 409 on duplicate → see `http-contracts.md`

---

## Passwords

1. Hash passwords with bcrypt, work factor >= 12 — lower is too fast to crack
2. NEVER store plain passwords — NEVER use MD5 or SHA1 for passwords
3. Always return the same error for wrong username AND wrong password
   — NEVER reveal which one failed (attacker learns valid usernames)
4. Use `hmac.compare_digest(a, b)` for comparing secrets — NOT `==` (timing-safe comparison prevents timing attacks)
5. Lock account temporarily after 5 consecutive failed login attempts — return 429, not 401 (attacker gets no info)
6. Logout MUST invalidate BOTH the access token (add `jti` to blocklist in Redis) AND the refresh token (delete from DB)

## JWT Tokens

7. Set access token expiry to 15–30 minutes maximum
8. Never put sensitive data in JWT payload — only `sub` (user id), `role`, `exp`, `iat`
9. `sub` claim must be the user's **ID** (stable, immutable) — NEVER the username (can change)
10. Always include `exp` claim — NEVER skip token expiry check
11. Use HS256 for single-service, RS256 for multi-service (asymmetric — public key verifies, private key signs)
12. NEVER accept tokens signed with `alg: none` — verify algorithm explicitly
13. Validate `iat` (issued at) claim — reject tokens with `iat` in the future (clock skew attacks)

## Token Extraction

14. Use `HTTPBearer(auto_error=False)` — NEVER `auto_error=True`
    — `auto_error=True` returns 403 for missing token, which is wrong (should be 401)
15. NEVER parse `Authorization` header manually
16. Raise `HTTPException(401)` manually in `get_current_user` when token is absent or invalid

## Dependency Chain (this is the source of truth for the chain)

17. Dependency order: `get_db` → `get_current_user` → `require_role`
18. `get_current_user` depends on `get_db` — always inject via `Depends(get_db)`
19. Use `require_role()` as a dependency factory — NEVER check role inside the route body

## Data Isolation

20. Filter data by `user_id` on every query — regardless of role
    — DB-level enforcement in `database-patterns.md`
21. NEVER return another user's private data even to admin role

## Role Management

22. NEVER allow role to be changed via public API — admin-only, server-side only
23. NEVER expose a role-change endpoint (e.g. `POST /users/{id}/role`)

## OAuth2 Form-Based Auth (first-party clients only)

24. Use `OAuth2PasswordRequestForm` when login form sends `application/x-www-form-urlencoded`
    — this is the OAuth2 spec format, not JSON
    ```python
    from fastapi.security import OAuth2PasswordRequestForm

    @router.post("/auth/token")
    async def login(form: OAuth2PasswordRequestForm = Depends()):
        # form.username, form.password
    ```
25. NEVER use `OAuth2PasswordRequestForm` for third-party clients — use JSON body with `LoginRequest` Pydantic model instead
26. `OAuth2PasswordRequestForm` is for first-party apps (your own frontend) only

## Refresh Tokens (when implemented)

27. Rotate refresh token on every use — issue new, invalidate old
28. Store revoked `jti` in Redis with TTL = token expiry duration
29. Check Redis for revoked `jti` on EVERY protected request — not just at logout
30. Refresh tokens are stateful (stored in DB) — access tokens are stateless (verify signature only)

## API Key Auth (machine-to-machine)

31. For server-to-server integrations, use static API keys via `X-API-Key` header — not JWT
32. Store API keys as bcrypt hash in DB — NEVER store plaintext
33. Scope API keys to specific permissions — NEVER issue full-access keys
34. Show the key only once at creation — never retrievable again (user must rotate if lost)

## Session Security

35. Invalidate all sessions on password change — user may have been compromised
36. NEVER reuse a JWT `jti` — it must be unique per token (use uuid4)
37. Add `iss` (issuer) claim to JWT and validate it — prevents tokens from other services being accepted
38. On account email change: re-verify the new email AND invalidate all existing tokens

## JWT aud Claim

39. Add `aud` (audience) claim to all JWTs and validate it on decode — prevents a token issued for Service A from being accepted by Service B:
    ```python
    # Encode
    payload = {"sub": user_id, "iss": "my-api", "aud": "my-api", "exp": ...}

    # Decode — rejects token if aud doesn't match
    decoded = jwt.decode(token, SECRET, algorithms=["HS256"], audience="my-api")
    ```
40. NEVER skip `aud` validation in multi-service architectures — without it, a compromised token from any service works everywhere

## Concurrent Session Limits

41. Enforce a maximum number of active refresh tokens per user — revoke the oldest when the limit is exceeded:
    ```python
    MAX_SESSIONS = 5

    async def create_session(user_id: str, db: AsyncSession) -> str:
        count = await db.scalar(
            select(func.count()).where(RefreshToken.user_id == user_id, RefreshToken.revoked == False)
        )
        if count >= MAX_SESSIONS:
            # Revoke the oldest
            oldest = await db.scalar(
                select(RefreshToken)
                .where(RefreshToken.user_id == user_id, RefreshToken.revoked == False)
                .order_by(RefreshToken.created_at.asc())
            )
            oldest.revoked = True
        new_token = RefreshToken(user_id=user_id, ...)
        db.add(new_token)
        return new_token.token
    ```
42. NEVER allow unlimited concurrent sessions — an attacker who obtains a refresh token can maintain access indefinitely across all sessions
