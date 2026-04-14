---
description: Multi-tenant, family/group data sharing, and RLS rules.
paths: ["*.py", "app/**/*.py", "models/**/*.py", "routes/**/*.py"]
---

# Tenancy Patterns — Production Rules

> Covers: tenancy model selection, PostgreSQL RLS, family data models, RBAC, invite systems, shared AI memory, audit logging, cache isolation, and anti-patterns.
> Security rules → security.md / security-complete.md
> Auth rules → auth-patterns.md

---

## 1. Tenancy Model Selection

1. NEVER use application-level filtering (WHERE clause in every query) as the sole tenancy enforcement mechanism for any data classified as sensitive, PII, or regulated — a single developer error, a raw SQL hotfix, or an ORM bypass will silently expose all tenants' data with no database-layer safety net.

2. ALWAYS choose the tenancy model based on compliance requirement first, then cost, then tenant count — regulatory requirements (HIPAA, PCI-DSS enterprise tier, GDPR data residency) mandate database-per-tenant or schema-per-tenant regardless of cost; overriding compliance for cost savings is a liability, not an optimization.

3. USE Row-Level Security (RLS) as the default tenancy mechanism for SaaS apps with thousands of small tenants on a shared PostgreSQL instance — it enforces tenant isolation at the database layer, requires no schema changes per tenant, and adds minimal overhead when `tenant_id` columns are properly indexed.

4. USE schema-per-tenant (PostgreSQL `search_path`) only when tenants number in the hundreds and require different schema evolution timelines or per-tenant Postgres permissions — beyond ~2,000 tenants, migration orchestration across schemas becomes a production hazard.

5. USE database-per-tenant only when enterprise contracts mandate physical data isolation, when tenants require region-specific data residency, or when per-tenant backup and restore are a contractual SLA — never use it as a default for free-tier users.

6. ALWAYS implement a hybrid model for multi-tier SaaS: shared schema + RLS for free/pro tenants, dedicated database for enterprise tenants — route requests to the correct engine via a `TenantRouter` class that checks `tenant.tier` before returning a session; never mix shared-session and dedicated-session logic in route handlers.

7. NEVER run application code under the database migration role (the role used by Alembic or similar) — migration roles typically have BYPASSRLS or table ownership; any bug that causes a request to execute under the migration role bypasses all tenant isolation.

8. DOCUMENT the chosen tenancy model in SPEC.md and in a comment at the top of the database session factory — include the rationale and the compliance requirement it satisfies; future engineers must understand why changing the model requires a compliance review, not just a refactor.

9. ALWAYS test tenancy boundaries with integration tests that run as the actual application database role (not a superuser) with RLS active — tests that run as a superuser or as the table owner silently bypass RLS and will not catch policy regressions.

10. WHEN migrating from application-level filtering to RLS, run both mechanisms in parallel for at least one release cycle — keep all WHERE clauses while RLS is deployed; remove the WHERE clauses only after verifying RLS catches the same queries in integration tests.

---

## 2. PostgreSQL RLS

11. ALWAYS run both `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and `ALTER TABLE ... FORCE ROW LEVEL SECURITY` on every tenanted table — `ENABLE` alone does not apply RLS to the table owner; `FORCE` removes the owner bypass; omitting `FORCE` silently grants full table access to the migration role.

12. ALWAYS use `set_config('app.current_tenant_id', :tid, true)` with the third argument set to `true` (transaction-scoped) — never `false` (session-scoped); connection pools reuse connections across requests; a session-scoped config set for tenant A persists when that connection is reused for tenant B, causing a cross-tenant data leak.

13. ALWAYS set RLS context variables (`app.current_user_id`, `app.current_family_id`, etc.) inside the per-request database session dependency, after authentication resolves — never set them in a connection pool `connect` event because tenant context is not available at connection creation time.

14. ALWAYS include both a USING clause (governs SELECT/UPDATE/DELETE row visibility) and a WITH CHECK clause (governs INSERT/UPDATE write validation) in every RLS policy — a policy with only USING allows inserting rows with an attacker-controlled `tenant_id`; WITH CHECK prevents writing outside the current tenant's boundary.

15. USE PERMISSIVE policies (OR logic) for "what the user is allowed to see" — owner access, family member access, public visibility; use RESTRICTIVE policies (AND logic) for hard outer boundaries that must never be crossed — `tenant_id` must always match, regardless of what PERMISSIVE policies grant.

16. ALWAYS create a composite index with `tenant_id` as the leading column on every RLS-enforced table — without it, every query with the RLS predicate performs a Seq Scan on the full table; the index reduces this to an O(log N) Index Scan on only that tenant's rows.

17. ALWAYS verify that RLS policies use the index by running `EXPLAIN (ANALYZE, BUFFERS)` as the application database role with `set_config` active — the plan should show `Index Scan` on the tenant_id index, not `Seq Scan` on the full table; a Seq Scan means the policy predicate is not being pushed down correctly.

18. NEVER use `current_user` (the PostgreSQL session role name) as the tenant identifier in RLS policies — `current_user` requires one database role per tenant to distinguish tenants; use `current_setting('app.current_tenant_id')` with per-request config instead, which works correctly with a shared application role and a connection pool.

19. ALWAYS test RLS in CI by running `SET ROLE app_user; SELECT set_config(...)` before each isolation assertion — tests run as a superuser bypass RLS entirely; the test must simulate the exact role and config state that production queries run under.

20. NEVER grant `BYPASSRLS` to the application database role — `BYPASSRLS` is for migration roles, reporting roles, and admin tools only; any application path that runs under a BYPASSRLS role is a tenant isolation hole.

21. WHEN `set_config` is called with an empty string for `app.current_family_id` (because the user has no family), ensure the RLS policy handles this gracefully — either treat empty string as NULL in the policy (`NULLIF(current_setting('app.current_family_id', true), '')::UUID`) or return no rows for family-scoped conditions when the value is empty.

22. ALWAYS include RLS migration tests in CI as a dedicated test suite separate from unit tests — run them against a real PostgreSQL instance (not SQLite), as a role with the same privileges as the production application role; SQLite does not support RLS.

23. ALWAYS use `current_setting('app.current_tenant_id', true)` (with the second `true` argument = missing_ok) in policies rather than `current_setting('app.current_tenant_id')` — if the config variable is not set (e.g., a background job forgot to set it), the missing_ok variant returns NULL and the policy denies access; without it, PostgreSQL raises an error and the query fails with a 500.

24. WHEN a background job or admin tool needs to query across all tenants, run it under the BYPASSRLS role, not under the app role — log the reason in a comment in the job code; any query that bypasses RLS must be explicitly justified and audited.

25. NEVER place the `set_config` call inside an ORM `before_execute` event that fires on every statement — this causes the config to be reset mid-transaction on sub-queries; set it exactly once per request at the start of the session dependency.

---

## 3. Family Data Model

26. ALWAYS make `family_id` nullable on both the `users` table and on content tables (`items`, `memories`, etc.) — solo users who have not joined a family must be able to use the app; a NOT NULL constraint forces every user into a family before they can create any data, breaking onboarding.

27. ALWAYS maintain a separate `family_members` join table with a `role` column and a `joined_at` timestamp rather than storing `family_id` and `role` directly on the `users` table — the join table supports audit history, eventual multi-family membership, role changes without user table writes, and a clean query for "list all members of family X".

28. ALWAYS enforce the role column value with a PostgreSQL CHECK constraint in addition to an application-level enum — `CHECK (role IN ('owner', 'member', 'viewer'))` — the database constraint is the last line of defense if a code path bypasses the ORM enum.

29. ALWAYS define visibility as a PostgreSQL CHECK-constrained TEXT column (`CHECK (visibility IN ('private', 'family', 'public'))`) rather than a boolean `is_public` flag — a boolean cannot express the three-state private/family/public model; adding a fourth visibility level later (e.g., "friends") requires a column migration instead of just adding an enum value.

30. ALWAYS enforce the invariant "family-scoped content must have a family_id" at the database level with a CHECK constraint — `CHECK (visibility = 'user' OR family_id IS NOT NULL)` — without this, a `visibility = 'family'` row with `family_id = NULL` is orphaned and no RLS policy can correctly gate access to it.

31. NEVER allow a user to set `family_id` on a content row to a family they are not a member of — validate in the application that `body.family_id == current_user.family_id` before insert; also enforce in the RLS WITH CHECK clause as a database-layer backstop.

32. ALWAYS cascade deletes from `families` to `family_members` (`ON DELETE CASCADE`) and set `family_id` to NULL on content tables (`ON DELETE SET NULL`) — when a family is deleted, membership rows must be cleaned up automatically; content rows should be orphaned (not deleted) so users retain their data.

33. ALWAYS cascade deletes from `users` to `family_members` and to all user-owned content (`ON DELETE CASCADE`) — when a user account is deleted, all their family memberships and private content must be removed; leaving orphaned rows with a deleted `user_id` FK violates referential integrity and breaks GDPR erasure guarantees.

34. WHEN designing queries that combine ownership and family access, keep the OR logic explicit in application code rather than relying on the RLS policy alone — this makes the access semantics visible in the query, aids code review, and remains correct even if the underlying RLS policy changes.

35. ALWAYS add a unique constraint on `(user_id, family_id)` in the `family_members` table to prevent duplicate memberships — a user should appear at most once per family; duplicate rows cause ambiguous role lookups and double-counting in member queries.

---

## 4. RBAC and Permission Checks

36. ALWAYS implement family role checks as FastAPI dependency factories that take a `min_role` string parameter — `require_family_role("owner")` vs `require_family_role("member")` — never inline role string comparisons in route handlers; inline comparisons are untestable and easy to skip in a copy-paste error.

37. ALWAYS represent the role hierarchy as integer ranks (`VIEWER=1, MEMBER=2, OWNER=3`) and compare with `>=` — string equality (`role == "owner"`) breaks the moment you need to check "at least member"; integer comparison makes the hierarchy unambiguous.

38. ALWAYS return HTTP 404 (not 403) when the current user is not a member of the requested family — 403 Forbidden reveals that the family ID exists; 404 reveals nothing about whether the family exists or the user simply lacks access.

39. NEVER check family membership after fetching family-owned content — fetch the membership first (as a dependency), verify the role, then proceed to the content query; if the membership check is a side effect of content access, it is easy to accidentally skip.

40. ALWAYS pass the validated `FamilyMember` object (not just the role string) as the return value of the role dependency — downstream handlers need `membership.family_id` and `membership.user_id` without additional DB queries; returning the full object avoids a second lookup.

41. ALWAYS use a separate `require_family_owner()` dependency (not an inline check) for destructive operations — delete family, remove a member, revoke all invites — explicit named dependencies make the intent obvious in code review and are harder to accidentally remove.

42. NEVER store role information in the JWT and use it as the sole authorization check for family operations — role changes (member promoted to owner, member removed) must take effect immediately; JWT-cached roles are stale until token expiry; always re-validate the role from the database for family authorization checks.

43. WHEN a user has no family (`user.family_id IS NULL`), reject any request to family-scoped endpoints with 400 ("You are not in a family") rather than 403 — the user is not unauthorized; they simply have no family context; the distinction matters for client-side UX (show "join a family" CTA vs "access denied" message).

44. ALWAYS include the `family_id` in the family role dependency signature (from the URL path parameter) rather than inferring it from `current_user.family_id` — a user could theoretically be a member of multiple families in the future; binding to the path parameter makes the check robust to schema evolution.

45. NEVER skip the role dependency for GET (read) endpoints on family resources — `require_family_role("viewer")` should be the minimum for any family data endpoint, even read-only ones; unauthenticated reads of family data are a BOLA vulnerability.

---

## 5. Invite System

46. NEVER store the raw invite token in the database — store `hashlib.sha256(raw_token.encode()).hexdigest()` only; if the database is compromised, the attacker gets only the hash; the raw token (which appears in the invite URL) cannot be reconstructed from its SHA-256 hash.

47. ALWAYS generate invite tokens with `secrets.token_urlsafe(32)` — this produces 256 bits of entropy encoded as 43 URL-safe characters; do not use `uuid4()` (122 bits) or `random.token_hex()` (non-cryptographic PRNG) for security tokens.

48. ALWAYS set an expiry on every invite token and enforce it at acceptance time — a link sent six months ago should not still be valid; 7 days is a reasonable default; check `invite.expires_at < datetime.now(UTC)` before accepting.

49. ALWAYS mark invite tokens as single-use by setting `used_by` and `used_at` on acceptance and checking `used_by IS NOT NULL` before processing — an invite link that can be shared and reused by multiple people defeats the purpose of a controlled family membership system.

50. ALWAYS return the same generic error message for all invite validation failures (not found, expired, already used) — `"Invalid or expired invite link"` — differentiating the error leaks information: "token expired" tells an attacker a token matching that hash exists; "already used" tells them when it was accepted.

51. ALWAYS perform the three-step acceptance atomically in a single database transaction: create the `family_members` row, update `user.family_id`, and mark the invite as used — if any step fails, all three roll back; a partial state (user has family_id but no FamilyMember row) breaks every downstream authorization check.

52. NEVER allow a user who is already a member of a family to accept an invite without first leaving their current family — check `current_user.family_id IS NOT NULL` before processing; return 409 Conflict with a clear message; silently overwriting `family_id` would orphan the old FamilyMember row.

53. ALWAYS verify the invite's `family_id` matches the family the owner intended before creating the membership — the lookup is by token hash, which is unique; but include a `WHERE family_id = :expected_family_id` if the invite URL also encodes the family ID to prevent invite replay across families.

54. ALWAYS allow the family owner to revoke (delete) pending invite records — implement `DELETE /families/{id}/invites/{invite_id}` gated by `require_family_role("owner")`; deleted invite rows make the token hash unfindable, instantly invalidating the link.

55. NEVER expose the `token_hash` in any API response — it is internal state; the only time the raw token is transmitted is at creation time in the `invite_url` field; never return it again from any endpoint.

---

## 6. Shared AI Memory

56. ALWAYS define memory scope as an explicit enum (`scope = 'user' | 'family'`) rather than a boolean `is_shared` flag — the enum is self-documenting, handles future scope additions (e.g., `scope = 'public'`), and makes the intent of every memory record unambiguous in queries and audit logs.

57. NEVER automatically promote a user's private memory to family scope — require an explicit user action (a "share with family" endpoint); the default for any new memory must be `scope = 'user'`; implicit sharing violates the principle of least disclosure and can cause GDPR issues.

58. ALWAYS query memories with explicit scope conditions rather than fetching all memories for the user and filtering in Python — `WHERE (user_id = :uid AND scope = 'user') OR (family_id = :fid AND scope = 'family')` is one indexed DB query; filtering 10,000 rows in Python is a latency and memory problem.

59. ALWAYS inject user-private memories before family-shared memories when building LLM context — personal preferences and personal history take precedence over shared family context; explicitly label each block (`## Personal Context` / `## Family Context`) and instruct the model that personal context is higher priority.

60. NEVER return another family member's private memories even when building a family LLM context — the query for LLM context must always filter by `user_id = current_user.id` for user-scoped memories; only family-scoped memories (`scope = 'family'`) are visible to all family members.

61. WHEN a user leaves a family, immediately invalidate any cached family memory context for that user — if the LLM context is cached by user_id, a cache entry built before the user left the family will still include family-scoped memories that the user no longer has access to.

62. WHEN a user account is deleted, delete all their user-scoped memories immediately via DB cascade (ON DELETE CASCADE on user_id) — for family-scoped memories they created, either delete them or anonymize them (set user_id = NULL) depending on your documented GDPR policy; document the policy in SPEC.md before implementing.

63. WHEN a user leaves a family, retain family-scoped memories they contributed — those memories belong to the family, not the individual; the user loses access to them by losing family membership, but the content remains for the other family members.

64. ALWAYS validate that a user is a current family member before allowing them to create a family-scoped memory — check the `family_members` table at write time, not just `user.family_id`; the join table is the authoritative source of membership.

65. NEVER store family-scoped memories without a `family_id` — enforce `CHECK (scope = 'user' OR family_id IS NOT NULL)` at the database level; a family-scoped memory with no family_id is unreachable by any correct query.

---

## 7. Cross-User Audit Logging

66. ALWAYS log to the audit table within the same database transaction as the data access that triggered the log entry — if the content read commits, the audit entry commits; if the read rolls back (error), the audit entry rolls back too; a separate async audit write can miss entries on failure.

67. ALWAYS make audit tables append-only at both the ORM layer (raise RuntimeError in before_update and before_delete SQLAlchemy events) and the database layer (PostgreSQL trigger that raises an exception on UPDATE or DELETE) — belt-and-suspenders; either layer alone can be bypassed.

68. ONLY log cross-user data access to the family audit table — skip logging when `accessor_user_id == owner_user_id`; self-access audit spam bloats the table and makes it harder to find real cross-user access events.

69. ALWAYS include `accessor_user_id`, `owner_user_id`, `resource_type`, `resource_id`, `action`, and `ip_address` in every audit log entry — an audit entry missing any of these fields is unusable for a GDPR access request or a security investigation; do not make any field optional.

70. ALWAYS create indexes on `owner_user_id` and `accessor_user_id` on the audit log table — GDPR access requests ("who has seen my data") query by `owner_user_id`; security investigations ("what has this user accessed") query by `accessor_user_id`; without indexes, both queries are full table scans on a potentially billion-row table.

71. WHEN implementing a GDPR Article 15 data access response, query the audit log for all entries where `owner_user_id = requesting_user.id` within the relevant retention period — return the accessor's identity, the resource type and ID, the action, and the timestamp; this is the minimum required by law.

72. NEVER expose raw `resource_id` values in audit log API responses without first verifying that the resource still exists and the requesting user is its owner — a user querying their access log should not be able to enumerate deleted resources via their audit entries.

---

## 8. Multi-Tenancy Cache Isolation

73. ALWAYS prefix every cache key with `tenant:{tenant_id}:` or `family:{family_id}:` before adding any other key components — a cache miss falls through to the database (safe); a cache hit that returns another tenant's data is a data breach; namespace collision is the single most common multi-tenant cache bug.

74. NEVER cache a response that was built from a multi-tenant database query without including the tenant ID in the cache key — the output of `SELECT * FROM items WHERE family_id = 'F1'` must be cached under a key that includes `F1`; caching it under a generic key causes family F2 to receive family F1's data on a cache hit.

75. ALWAYS include `user_id` in addition to `family_id` when caching per-user views of family data — two members of the same family may see different subsets (e.g., filtered by their own item visibility); the cache must distinguish them.

76. NEVER cache family membership or role information with a TTL longer than the expected latency for role changes to take effect — if a user is removed from a family and the membership cache TTL is 1 hour, that user retains access for up to 1 hour; use a short TTL (60 seconds) or invalidate explicitly on membership changes.

77. ALWAYS invalidate family-scoped cache entries when a member joins or leaves the family — a new member should immediately see current family data, not stale pre-join cache; a leaving member should immediately lose access to family cache entries.

78. ALWAYS use Redis key tagging or a separate invalidation set to group all cache keys for a family under a single invalidation operation — when family data changes, one `DELETE family:{family_id}:*` scan (or tag-based invalidation) clears all dependent cache entries; never manually track individual keys to invalidate.

79. NEVER use in-process cache (e.g., Python `functools.lru_cache` or `cachetools.TTLCache`) for family or tenant-scoped data in a multi-instance deployment — in-process caches are per-process; a write on instance A invalidates that instance's cache but leaves instances B and C serving stale data to users of the same family.

80. WHEN a user's family context changes (`user.family_id` changes from joining or leaving a family), invalidate ALL cache entries keyed by `user:{user_id}:*` in addition to family-scoped entries — the user's cached context (memory context, item lists, permissions) was built under the old family context and is now stale.

---

## 9. Anti-Patterns (NEVER do these)

81. NEVER check tenant or family ownership after fetching a record — the check must be part of the WHERE clause or RLS policy that fetches the record; a post-fetch Python check means the unauthorized read has already happened in the database, and the data is in memory even if you raise a 403 afterward.

82. NEVER use `role == "owner"` string comparison for access decisions — use integer rank comparison (`FamilyRole.rank(role) >= FamilyRole.rank("owner")`) or a dedicated `require_family_role("owner")` dependency; string comparison is not hierarchy-aware and breaks the moment you add a new role between existing ones.

83. NEVER skip the family membership DB check and rely solely on `user.family_id` from the JWT or the users table for authorization — `user.family_id` tells you the user's current family; it does not tell you their role or whether they are still an active member; always check the `family_members` table for role-dependent operations.

84. NEVER use `session_config` (false) instead of `local_config` (true) in `set_config` calls for RLS context — see rule 12; this is the most dangerous single-line mistake in a multi-tenant PostgreSQL app and does not produce an error; it silently leaks data.

85. NEVER call `set_config` once at application startup or in a middleware that runs before the connection is checked out from the pool — the config must be set per-request, on the exact connection used for that request, after the connection is checked out from the pool.

86. NEVER design a family invite system that sends the raw token as a query parameter AND stores the same raw token in the database — an attacker with DB read access has every pending invite link; store the hash, transmit the raw token once, never persist it.

87. NEVER let a user accept an invite without first verifying their current family status — silently overwriting `user.family_id` orphans the old FamilyMember row and leaves the old family with a phantom member in the membership table.

88. NEVER cache per-family AI memory context with a TTL greater than 5 minutes when the family has more than one member — any member can share or unshare memories, and the other members should see changes promptly; a 5-minute max ensures the context is never more than one cache period stale.

89. NEVER log cross-user access in a separate async fire-and-forget without handling failures — if the audit log write fails silently, the access happens but is not audited; this breaks GDPR Article 15 compliance and removes the forensic trail for security incidents; use the same transaction as the data access.

90. NEVER build a multi-tenant app without a dedicated integration test that proves RLS prevents cross-tenant data access at the database layer — unit tests mock the DB and cannot catch RLS policy bugs; if the test does not execute against a real PostgreSQL instance as the app role, you have no evidence the policy works.

91. NEVER store family role or membership status in the JWT without a short TTL and a DB validation on role-sensitive operations — stale role claims in a JWT allow a user who was just removed from a family to continue operating as a family member until token expiry; always re-validate family roles from the DB.

92. NEVER delete a user's family-scoped memories when they leave the family — those memories were contributed to the shared family context and belong to the family, not to the individual; delete user-private memories on account deletion, not on family departure; document this distinction in your GDPR policy before go-live.

93. NEVER use a single RLS policy for both SELECT and INSERT without separate USING and WITH CHECK clauses — a policy with only USING does not validate the tenant_id on INSERT; a malicious user can insert a row with another tenant's family_id, planting data in their namespace.

94. NEVER expose family member email addresses or names in invite acceptance responses — the accepting user should only learn they joined a family (by ID or name), not receive a directory of all other members; a member directory is a separate, gated endpoint.
95. NEVER call `set_config('app.current_tenant_id', ...)` inside a background task without an explicit guard — background tasks do not execute inside a request lifecycle; the config variable will be NULL and every RLS policy using it will silently return zero rows rather than raising an error, causing silent data loss.

96. ALWAYS debug RLS policy behavior by running `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` as the application role with `set_config` active, and query `pg_policies` to list active policies — `SELECT policyname, cmd, permissive, qual, with_check FROM pg_policies WHERE tablename = 'your_table'` shows exactly which policies exist and their expressions.

97. NEVER run Alembic migrations under the application database role — the application role has RLS enabled and FORCE RLS active; Alembic DDL does not set `app.current_tenant_id` and will fail; create a dedicated migration role with `BYPASSRLS` privilege and connect Alembic via a separate `MIGRATION_DATABASE_URL` environment variable.

98. ALWAYS handle the family-has-no-owner edge case at the database level with a trigger that fires AFTER DELETE ON family_members and raises an exception when `COUNT(*)` of owner-role members for the family reaches 0 — without this, removing the last owner leaves the family in an ungoverned, unrecoverable state.

99. ALWAYS implement ownership transfer as a single atomic transaction — insert the new owner row, update the old owner's role to 'member', and update `families.owner_user_id` in that order with `SELECT ... FOR UPDATE` on both family_members rows; never delete the old owner row first or rely on application-level sequencing.

100. ALWAYS rate-limit invite token acceptance separately from general API rate limits — limit to 5 attempts per IP per 10 minutes with exponential backoff; return identical 400 responses for all failure modes (not found, expired, used); log every failed attempt as a `SecurityEvent.INVITE_TOKEN_INVALID` with source IP.

101. WHEN a user changes family (leaves one, joins another), flush all Redis keys matching `user:{user_id}:*` AND `family:{old_family_id}:member:{user_id}:*` in the `after_commit` hook — stale cache entries keyed by the old family context will serve incorrect data before TTL expiry.

102. ALWAYS add `AND deleted_at IS NULL` to every RLS USING clause for soft-deleted tables rather than relying on application-level filtering — a soft-deleted record without this clause remains visible via direct DB queries and raw SQL hotfixes that bypass ORM filters.

103. ALWAYS store LangGraph checkpoint data with `thread_id` encoding both `family_id` and `user_id` (`family:{fid}:user:{uid}:thread:{tid}`) and configure `AsyncPostgresSaver` with a tenant-aware connection that has `set_config('app.current_family_id', ...)` active — LangGraph's default checkpoint tables have no tenant isolation.

104. NEVER silently overwrite a family-scoped AI memory when two members set contradictory values — use last-write-wins with `last_modified_by_user_id` column and a conflict event in the audit table; a higher-fidelity strategy is per-user override that shadows (not replaces) the family-scoped value.

105. ALWAYS scope GDPR data portability exports to include family-scoped memories the user created (`created_by_user_id = current_user.id`) but exclude family-scoped memories created by other members — GDPR Article 20 covers data the user provided, not other members' contributions; implement as two separate subqueries with UNION to make the boundary explicit and auditable.

106. NEVER apply RLS policies to a regular VIEW and expect them to protect it — PostgreSQL views run with the view owner's privileges by default, bypassing the calling user's RLS; create views with `SECURITY INVOKER` (`CREATE VIEW ... WITH (security_invoker = true)` in PG15+) so the query runs as the calling user and RLS applies normally.

107. WHEN you cannot use PG15+ `security_invoker` views, define a `SECURITY DEFINER` function that enforces the same row filter explicitly — treat every SECURITY DEFINER function as a potential RLS bypass and code-review it with the same rigour as the RLS policy itself.

108. NEVER create a materialized view that aggregates family data and refresh it outside a tenant context — materialized views store a physical snapshot that ignores RLS at read time; either create one per tenant, maintain aggregates in a regular table with triggers, or treat the materialized view as admin-only and block end-user access.

109. ALWAYS verify that RLS policies apply to each side of a JOIN independently — PostgreSQL evaluates the policy per table; passing RLS on one side does not confer access to the other side.

110. NEVER expose a join result where the joined table has no RLS policy — if `note_attachments` has no policy but is joined to an RLS-protected `shared_notes`, users who can read notes implicitly read all attachments; add a matching policy to the joined table.

111. ALWAYS run Alembic migrations in schema-per-tenant mode through a `run_migrations_per_tenant()` function that iterates a tenant registry, sets `search_path` to each tenant schema, runs `alembic upgrade head`, and resets `search_path` — a single `alembic upgrade head` with unset `search_path` applies only to `public`.

112. ALWAYS maintain a `migration_log (tenant_id, revision, applied_at)` table in the shared schema — query it in deployment to detect drift where some tenants are on an older revision and alert if any tenant is more than one revision behind head.

113. ALWAYS handle family deletion as a two-phase operation: (1) mark family `status='deleting'`, (2) enqueue a background job to re-parent or delete orphaned content, notify members, then hard-delete the family row — never cascade thousands of rows in the request handler; it will timeout and leave a partially-deleted family visible to members.

114. ALWAYS give VIEWER-role members a read-only PostgreSQL role (`app_viewer`) with only `SELECT` grants and set the transaction role to `app_viewer` in the connection setup path — a viewer cannot write even if application code omits the role check, because the database will reject DML with `permission denied`.

115. WHEN bulk-inviting members, use `SELECT COUNT(*) ... FOR UPDATE` on `family_members` inside the insert transaction to enforce the per-family member cap — without the row lock, two concurrent bulk invites both read a count below the cap and together push the family over limit.

116. ALWAYS implement cross-tenant admin operations under a separate `/internal/` router that sets `SET LOCAL row_security = off` (requires `BYPASSRLS` role) and records every operation to `admin_audit_log (admin_user_id, operation, target_family_id, target_user_id, timestamp, justification_text)` — never expose admin bypass via the same router as user-facing endpoints.

117. ALWAYS store family-scoped feature flags as rows in a `family_feature_flags (family_id, flag_name, enabled, overridden_by, overridden_at)` table with its own RLS policy — never store them as a JSONB blob on the `families` row; a blob cannot be individually audited and forces full-row lock contention on every flag change.

118. ALWAYS bind an AI agent's tool-call authorization to the `(user_id, family_id, role)` triple active when the session was initiated — pass this as an immutable context object; every tool must verify the operation is permitted for that role; never allow the agent to re-derive authorization from current DB state mid-session, because a concurrent role downgrade must not be silently ignored.

119. NEVER use `COPY tablename TO` on RLS-protected tables — this form bypasses RLS entirely regardless of `ENABLE` and `FORCE` settings; always use `COPY (SELECT ... WHERE family_id = $1) TO STDOUT` to maintain row-level filtering.

120. NEVER use `pg_dump` with a superuser role on tenant-scoped tables for per-tenant exports — superuser bypasses RLS; use a dedicated export role with `NOINHERIT` and no `BYPASSRLS`, with tenant context set via `set_config`.

121. NEVER accept `family_id` or `tenant_id` from client-supplied request parameters, headers, or request body as the authoritative tenant context — derive tenant context exclusively from the authenticated JWT combined with a server-side membership lookup.

122. NEVER expose `family_id` as a URL segment without verifying membership in a FastAPI dependency — the dependency queries `family_members WHERE user_id = current_user.id AND family_id = path_family_id`; a missing row returns 403 (not 404, to avoid confirming the family exists).

123. ALWAYS cache family membership checks in `family:{family_id}:member:{user_id}:role` with a 60-second maximum TTL — on any role change or removal, immediately delete the key in the same transaction's `after_commit` hook.

124. NEVER cache family membership beyond the current request without a cache invalidation path — if the invalidation hook cannot be guaranteed, reduce TTL to 15 seconds rather than accepting stale access grants.

125. ALWAYS include `family_id` as an explicit `WHERE` clause in cross-tenant analytics queries — without it, a reporting query produces cross-tenant aggregates that leak relative usage information even when individual rows are RLS-filtered.

126. ALWAYS run cross-tenant analytics on a read replica with a dedicated reporting role — analytical queries on multi-million-row tenant tables on the primary produce lock contention that degrades write latency for all tenants.

127. ALWAYS route webhook delivery through a per-tenant endpoint registry `tenant_webhooks(family_id, event_type, endpoint_url, signing_secret)` — never route webhooks based on a `family_id` embedded in the event payload alone; derive the target endpoint from the authoritative registry.

128. ALWAYS sign outbound webhook payloads with a per-tenant `signing_secret` — never share a single signing secret across tenants; a compromised tenant's secret must be rotatable without affecting other tenants.

129. ALWAYS implement family-scoped rate limits as Redis counters keyed `family:{family_id}:quota:{resource}:{month_bucket}` — never share a global counter; a family exhausting its LLM quota must receive 429 without affecting other families.

130. ALWAYS enforce family-level LLM token budgets with an atomic Redis pipeline that checks the counter and increments in one call — checking and incrementing in two separate commands creates a TOCTOU race where two concurrent requests both pass the check.

131. ALWAYS include `family_id` as a composite index column alongside every GIN full-text search index and prefix every FTS query with `WHERE family_id = $1` — without the filter, cross-tenant document content participates in search ranking even when rows are RLS-filtered from results, leaking content similarity signals.

132. NEVER create a single GIN index over a tenant-scoped table without a `family_id` filter on every query — the GIN scan evaluates cross-tenant rows before RLS removes them, leaking relative similarity information across tenants.

133. ALWAYS scope API keys to a specific `family_id` at issuance — store `(key_hash, family_id, user_id, scopes[], expires_at)` in `api_keys`; verify `api_keys.family_id` matches the requested resource's family on every call.

134. ALWAYS model multi-family membership as a many-to-many join table `user_family_memberships(user_id, family_id, role, joined_at)` — never store `family_id` as a single FK on `users`; AI memory, preferences, and history must be scoped to `(user_id, family_id)` pairs, never to `user_id` alone.

135. ALWAYS require an explicit `active_family_id` on every authenticated session when a user belongs to multiple families — never infer the active family from the first or most-recently-joined family; a context switch must re-authenticate or exchange a context-switching token.

136. ALWAYS enforce the family member limit with `SELECT COUNT(*) FROM family_members WHERE family_id = $1 FOR UPDATE` inside the insert transaction — the `FOR UPDATE` lock prevents two concurrent join requests from both reading a count below the limit and both inserting.

137. ALWAYS make the member count check and insert atomic within a single serializable transaction or use a database-level trigger — two-step application-layer checks are vulnerable to TOCTOU even with `FOR UPDATE` if routing to different connections.

138. ALWAYS use `DECR` in Redis when a member leaves to keep the cached count consistent, and invalidate the count entirely on any membership change that cannot be guaranteed atomic — Redis is a fast pre-check only; the database `FOR UPDATE` is the authoritative gate.

139. ALWAYS configure PgBouncer in `pool_mode=transaction` when using RLS with `set_config` — in session-pooling mode, `set_config` with `is_local=true` resets at transaction end, but the connection is reused without re-running `set_config` if any code path skips it, causing the prior tenant's identity to persist.

140. NEVER use `set_config('app.current_family_id', $fid, false)` — the `false` argument makes the setting session-scoped; it persists across transactions on the same connection, causing the next tenant to inherit the prior tenant's `family_id` in a pooled setup.

141. ALWAYS call `set_config('app.current_family_id', $family_id, true)` as the first statement in every transaction touching RLS-protected tables — never rely on it persisting from a prior call; pool checkout order is non-deterministic.

142. ALWAYS write RLS policies to explicitly deny when `current_setting('app.current_family_id', true)` returns NULL or empty string — use `COALESCE(current_setting('app.current_family_id', true), '') != ''` as a guard; a bare `::uuid` cast on an empty string raises an error rather than denying cleanly, leaking RLS configuration details.

143. ALWAYS use event sourcing for family membership and role changes by appending to `family_events(id, family_id, event_type, actor_user_id, payload jsonb, created_at)` — never update membership status in-place without an audit trail; the `family_members` table is a projection derived from the event log.

144. ALWAYS make `family_events` append-only at the database level with a `BEFORE UPDATE OR DELETE` trigger that raises `EXCEPTION 'family_events is append-only'` — application-layer constraints are bypassed during incident response direct database access.

145. ALWAYS stream family data exports using asyncpg's server-side cursor in chunks of 500 rows yielded to a `StreamingResponse` — never collect all rows into a Python list; a family with hundreds of thousands of AI interaction records will exhaust application memory and block the event loop.

146. ALWAYS set `Content-Disposition: attachment; filename="family-export-{first_8_chars_of_family_id}.json"` on export responses — never include the full `family_id` in the filename; the filename appears in browser download history and system metadata, leaking a stable identifier.

147. NEVER include `family_id`, `user_id`, `tenant_id`, or internal row IDs in 4xx response bodies — a 403 body must say only "Access denied"; a 404 for a resource the user does not own must say only "Not found"; error bodies appear in client logs, bug reports, and error trackers.

148. NEVER log `family_id` or `user_id` at ERROR or WARNING level without confirming your logging infrastructure has access controls — use structured logging with a separate `tenant_context` field filtered from log sinks accessible to non-engineering roles.

149. ALWAYS write pytest RLS isolation tests using two separate database connections with different `set_config` values — `family_a_conn` and `family_b_conn` fixtures each set `set_config('app.current_family_id', ...)` at transaction start; assert a row inserted via `family_a_conn` is not visible via `family_b_conn`; a single-connection or superuser test does not verify RLS.

150. ALWAYS add a CI test connecting with a non-superuser `NOINHERIT`/`NOBYPASSRLS` role and verifying that `SELECT * FROM family_members` with no `set_config` returns zero rows — this catches the failure mode where a developer accidentally grants `BYPASSRLS` or disables `FORCE ROW LEVEL SECURITY` during a migration.

151. ALWAYS automate new family provisioning as a single database transaction that atomically creates the `family` row, inserts the owner into `family_members`, inserts default `feature_flags` rows, and appends a `FAMILY_CREATED` event — never provision across multiple transactions; a partial failure leaves the family permanently ownerless.

152. ALWAYS write a deferred constraint trigger (`INITIALLY DEFERRED`) that prevents a `family` row from existing with zero owner-role members — this enforces provisioning atomicity at the database level, catching bugs that application-layer tests miss.

153. ALWAYS use `CREATE POLICY ... AS PERMISSIVE` explicitly and never rely on the default — an accidental `AS RESTRICTIVE` policy that is too narrow will deny all rows for users who don't satisfy it, causing a production outage that is difficult to diagnose.

154. ALWAYS add new columns to RLS-protected tables as `NOT NULL DEFAULT value` in a single statement — an intermediate nullable state during a multi-step migration creates a window where NULL values in RLS policy expressions silently deny access to existing rows.

155. ALWAYS archive inactive families to archive schemas rather than deleting — use a scheduled job for families with no activity in 365 days; hard deletion makes GDPR data subject access requests for archived families impossible to fulfill.

156. ALWAYS enforce GDPR retention with a `data_retention_policy(family_id, delete_after_date)` table and a nightly deletion job that writes to an immutable `gdpr_deletion_log` before executing — never rely on manual deletion; the log must survive the deletion to prove compliance.

157. ALWAYS emit per-tenant metrics with `family_id` as a label on every metric — LLM token usage, API call count, error rate, and p99 latency must all be attributable to a specific family; use OpenTelemetry baggage to propagate `family_id` through async task boundaries.

158. ALWAYS record `(family_id, user_id, model, input_tokens, output_tokens, cost_usd, created_at)` in an `llm_usage_ledger` table on every LLM call completion — never aggregate costs at the application level only; the ledger enables per-family billing, cost anomaly detection, and GDPR data portability responses.
