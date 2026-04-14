# L20: Multi-User Family/Group Tenancy

> Audience: knows authentication, RBAC, PostgreSQL, SQLAlchemy async. This level extends single-user data isolation to groups of users who share a data space while retaining individual privacy within it.

---

## The Core Problem

Single-user systems are easy: every row has `owner_id = current_user.id`, every query filters on it, done.

Multi-user tenancy is harder because the same row now needs to answer three questions:
1. Who created it?
2. Who is allowed to read it?
3. Who is allowed to modify or delete it?

The answers can differ. A meal plan entry was created by the parent (`owner_id`), visible to every family member (`family_id`), but only the parent can delete it (`role = 'owner'`).

**Analogy:** Think of an apartment building. Each unit (family) has its own locked door. Inside the unit, family members move freely. The building super (database superuser) can enter any unit. A guest with an invite code gets a temporary key. Nobody from unit 5B can walk into unit 5A — even though the building is the same physical structure (same database server).

---

## Part 1: Tenancy Models

### The Four Tenancy Strategies

There is a spectrum from complete isolation to maximum sharing. Where you sit on that spectrum is determined by compliance, cost, and scale — not personal preference.

```
Full Isolation ◄────────────────────────────────────────► Full Sharing
  DB-per-tenant   Schema-per-tenant   RLS (shared table)   App-level filter
```

#### Strategy 1: Database-per-Tenant

Every tenant gets their own PostgreSQL database instance (or at minimum, their own database within a server).

```
Tenant A → postgresql://db-a.internal:5432/tenant_a
Tenant B → postgresql://db-b.internal:5432/tenant_b
```

**When to use:**
- Compliance requirements mandate data residency (each tenant's data in a specific region)
- Enterprise customers demand data isolation guarantees in their contracts
- Tenant-specific schema evolution (each tenant on a different migration version)
- High write volume per tenant that would create contention in a shared DB

**When NOT to use:**
- You have thousands of tenants — connection overhead and DB management becomes untenable
- Tenants are small (a family of 5) — a full DB per family is massive overengineering
- You need cross-tenant analytics — you would need a data warehouse aggregation layer

**Cost:** High. Each DB requires its own connection pool, its own backup process, its own monitoring, its own migration run. Operational complexity scales linearly with tenant count.

**Migration complexity:** Every schema change must be applied to every tenant DB, either sequentially or via a parallel migration orchestrator. One failed migration on tenant #847 while #1-846 are already migrated is a real operational problem.

#### Strategy 2: Schema-per-Tenant

All tenants on the same PostgreSQL server and database, but each tenant gets their own schema (namespace):

```sql
-- Schema = namespace within a database
CREATE SCHEMA tenant_abc;
CREATE SCHEMA tenant_xyz;

-- Each tenant has their own copy of every table
tenant_abc.items
tenant_xyz.items

-- switch context via search_path
SET search_path = tenant_abc;
SELECT * FROM items;  -- reads tenant_abc.items
```

**When to use:**
- Moderate isolation requirement (tenant data must be physically separate but not in separate DBs)
- Tenant count in the hundreds (not thousands)
- You need per-tenant Postgres permissions (DB roles scoped to a schema)

**Migration complexity:** You must run `alembic upgrade head` once per schema, or use a multi-schema migration tool. Adding a column to `items` means: `ALTER TABLE tenant_abc.items ADD COLUMN ...`, `ALTER TABLE tenant_xyz.items ADD COLUMN ...`, ... for every tenant. With 500 tenants this is a 500-step migration.

**PostgreSQL search_path trick:**

```python
# SQLAlchemy: set search_path per connection
from sqlalchemy import event

@event.listens_for(engine, "connect")
def set_search_path(dbapi_conn, connection_record):
    cursor = dbapi_conn.cursor()
    cursor.execute(f"SET search_path = {tenant_schema}, public")
    cursor.close()
```

**Performance:** Schema-level partitioning means each tenant's tables can have independent statistics and vacuum runs. This is actually better than RLS for tenants with very different data distributions.

#### Strategy 3: Row-Level Security (RLS)

All tenants share the same tables. A `tenant_id` column on every table. PostgreSQL enforces a WHERE clause on every query at the database layer — the application cannot accidentally bypass it.

```sql
-- One table for all tenants
CREATE TABLE items (
    id          UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    owner_id    UUID NOT NULL,
    content     TEXT
);

-- RLS policy
ALTER TABLE items ENABLE ROW LEVEL SECURITY;
ALTER TABLE items FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON items
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**The key insight:** The policy lives in the database. No matter how the application queries — raw SQL, ORM, stored procedure — the policy is always applied. You cannot forget to add the WHERE clause because the database adds it for you.

**When to use:**
- Thousands of small tenants (SaaS with free tier, family apps)
- Shared infrastructure is acceptable
- You trust your RLS policy implementation (see Part 2 for the details to get right)

**When NOT to use:**
- Enterprise compliance requires physical isolation
- Tenants have radically different schema needs
- You need to run per-tenant backups/restores

#### Strategy 4: Application-Level Filtering

No database enforcement. The application adds `WHERE tenant_id = :tid` to every query. The simplest approach — and the most error-prone.

```python
# Every single query must include this. Forget it once = data leak.
result = await db.execute(
    select(Item).where(
        Item.tenant_id == current_user.tenant_id,  # developer must remember this
        Item.id == item_id,
    )
)
```

**The problem:** It only takes one developer, one tired night, one complex join, one raw SQL query in a special-case handler to forget the WHERE clause and expose another tenant's data. There is no safety net.

**When to use:** Only when RLS is genuinely not available (non-PostgreSQL databases) or when you have a very simple app and can enforce it via code review + testing.

**The defense:** If you use application-level filtering, enforce it with a base query class that always appends the tenant filter:

```python
class TenantScopedQuery:
    """Every ORM query goes through this. Never call select(Model) directly."""
    def __init__(self, model, tenant_id: UUID):
        self._model = model
        self._tenant_id = tenant_id

    def base(self):
        return select(self._model).where(
            self._model.tenant_id == self._tenant_id
        )
```

---

### Decision Matrix

| Criterion | DB-per-Tenant | Schema-per-Tenant | RLS | App Filter |
|-----------|--------------|-------------------|-----|------------|
| **Data isolation** | Complete (physical) | Strong (namespace) | Strong (policy) | Weak (trust the code) |
| **Compliance fit** | HIPAA, PCI, SOC2 enterprise | SOC2, light GDPR | GDPR, SOC2 shared | Not recommended for regulated data |
| **Tenant count** | < 100 | 100–2000 | 2000–millions | Any (but risky) |
| **Cost at 10k tenants** | Very high (infra per tenant) | High (migration ops) | Low (one DB) | Low (one DB) |
| **Migration complexity** | Very high | High | Low (single schema) | Low |
| **Cross-tenant queries** | Requires data warehouse | Possible (cross-schema query) | Easy (no JOIN needed) | Easy |
| **Operational overhead** | Very high | Medium | Low | Low |
| **Risk of data leak** | Very low | Low | Low (if RLS correct) | High (developer error) |
| **Team skill needed** | DevOps | Postgres advanced | Postgres RLS | Basic SQL |

---

### Hybrid Models

Real-world SaaS products frequently combine strategies:

**Small tenant → shared schema with RLS. Enterprise tenant → dedicated database.**

```python
class TenantRouter:
    """Route queries to the right database based on tenant tier."""

    def get_engine(self, tenant: Tenant) -> AsyncEngine:
        if tenant.tier == "enterprise" and tenant.dedicated_db_url:
            return create_async_engine(tenant.dedicated_db_url)
        else:
            return self._shared_engine

    async def get_session(self, tenant: Tenant) -> AsyncSession:
        engine = self.get_engine(tenant)
        async with AsyncSession(engine) as session:
            if tenant.tier != "enterprise":
                # RLS for shared tenants
                await session.execute(
                    text("SELECT set_config('app.current_tenant_id', :tid, true)"),
                    {"tid": str(tenant.id)}
                )
            yield session
```

This is how Notion, Linear, and similar products handle enterprise customers — the free/pro tier is on shared infrastructure, enterprise pays for a dedicated instance.

---

## Part 2: PostgreSQL Row-Level Security (RLS) Deep Dive

### The Library Stacks Analogy

Imagine a library where the librarian has a policy: "You can only check out books that belong to your library card." The librarian does not check IDs manually for every request — the system does it automatically at the checkout desk. Even if a librarian forgets to check, the desk terminal refuses.

That terminal is RLS. It operates at the database level, below the application.

### Enabling RLS — Two Required Commands

```sql
-- Step 1: Enable RLS on the table
ALTER TABLE items ENABLE ROW LEVEL SECURITY;

-- Step 2: FORCE RLS even for the table owner
ALTER TABLE items FORCE ROW LEVEL SECURITY;
```

**Why two steps?** `ENABLE ROW LEVEL SECURITY` alone does not apply RLS to the table owner (the role that created the table, typically your migration user). The table owner can bypass RLS by default. `FORCE ROW LEVEL SECURITY` removes that bypass — even the owner is subject to policies.

**What happens with ENABLE only (without FORCE)?**
```
Migration role = table owner
RLS enabled, FORCE not set
→ Migration role runs SELECT * FROM items
→ Returns ALL rows (policy ignored)
→ If your app user = migration user, RLS is silently bypassed
```

Always use both commands together.

### Creating Policies: USING vs WITH CHECK

A policy has two clauses:
- `USING` — applies to SELECT, UPDATE, DELETE. "Which rows can I see/touch?"
- `WITH CHECK` — applies to INSERT, UPDATE. "Can I write this new value?"

```sql
-- Basic tenant isolation policy
CREATE POLICY tenant_isolation ON items
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**USING clause — governs reads and deletes:**
```sql
-- User can only SELECT rows where tenant_id matches
-- User can only DELETE rows where tenant_id matches
USING (tenant_id = current_setting('app.current_tenant_id')::UUID)
```

**WITH CHECK clause — governs writes:**
```sql
-- User cannot INSERT a row with a different tenant_id
-- User cannot UPDATE a row to change tenant_id
WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::UUID)
```

**If you omit WITH CHECK:** A malicious user can INSERT a row with `tenant_id = 'competitor_tenant'`, then that row is owned by the competitor. They just planted data in another tenant's space.

### The Transaction-Scoped Config Variable

The database needs to know which tenant the current query belongs to. You pass this via a session-level config variable:

```sql
-- Set the tenant ID for the current transaction
-- Third argument 'true' = LOCAL = reset at transaction boundary
SELECT set_config('app.current_tenant_id', 'abc123', true);

-- Read it in a policy
USING (tenant_id = current_setting('app.current_tenant_id')::UUID)
```

**The critical detail — `true` vs `false`:**

```sql
-- true = transaction-scoped (SAFE for connection pools)
SELECT set_config('app.current_tenant_id', 'abc123', true);
-- Value resets after COMMIT or ROLLBACK

-- false = session-scoped (DANGEROUS with connection pools)
SELECT set_config('app.current_tenant_id', 'abc123', false);
-- Value persists on the connection until explicitly changed
-- If connection is returned to pool and reused, new request
-- inherits PREVIOUS tenant's ID → data leak
```

**Always use `true`.** Connection pools reuse connections across requests. A session-scoped variable set for tenant A persists when that connection is reused for tenant B.

### Setting the Config Variable in SQLAlchemy

```python
from sqlalchemy import event, text
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(DATABASE_URL)

# Option 1: On connection checkout (fires when pool gives out a connection)
@event.listens_for(engine.sync_engine, "connect")
def set_tenant_on_connect(dbapi_conn, connection_record):
    """
    Called when a new connection is created.
    At connect time we don't know the tenant yet.
    We set a safe default; the actual value is set per-request.
    """
    pass  # Don't set here — no tenant context yet

# Option 2: In the request dependency (correct approach)
async def get_db_for_tenant(
    tenant: Tenant = Depends(get_current_tenant),
) -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(engine) as session:
        # Set tenant ID at the start of every request
        await session.execute(
            text("SELECT set_config('app.current_tenant_id', :tid, true)"),
            {"tid": str(tenant.id)}
        )
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

**Why not set it in the connection event?** At connection creation time, you do not have a request context — no authenticated user, no tenant. The tenant is known only once the JWT is decoded inside the request lifecycle. Set it explicitly in the request dependency after authentication resolves.

### Policy Types: PERMISSIVE vs RESTRICTIVE

**PERMISSIVE (default):** Multiple permissive policies combine with OR logic. If ANY permissive policy grants access, the row is visible.

```sql
-- Policy 1: owner can see their own rows
CREATE POLICY owner_access ON items
    AS PERMISSIVE FOR SELECT
    USING (owner_id = current_user_id());

-- Policy 2: family members can see family rows
CREATE POLICY family_access ON items
    AS PERMISSIVE FOR SELECT
    USING (
        family_id IS NOT NULL
        AND family_id = current_setting('app.current_family_id')::UUID
        AND visibility = 'family'
    );

-- Result: A row is visible if EITHER policy passes
-- owner_access OR family_access
```

**RESTRICTIVE:** Multiple restrictive policies combine with AND logic. ALL restrictive policies must pass.

```sql
-- Even if owner_access passes, this must ALSO pass
CREATE POLICY tenant_boundary ON items
    AS RESTRICTIVE FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Result: Row visible ONLY if (owner_access OR family_access) AND tenant_boundary
```

**Pattern:** Use PERMISSIVE for the "what you're allowed to see" logic (owner access, family access, public access). Use RESTRICTIVE for hard boundaries that can never be crossed (tenant isolation). The RESTRICTIVE policy is a guard rail — even if a PERMISSIVE policy accidentally grants too much, the RESTRICTIVE policy enforces the outer boundary.

### Bypassing RLS: Superuser and BYPASSRLS

Two ways to bypass RLS:
1. Connect as a PostgreSQL superuser (roles with `superuser = true`)
2. Use a role with `BYPASSRLS` attribute: `ALTER ROLE migration_user BYPASSRLS;`

**When to use BYPASSRLS:**
- Database migrations (you need to alter tables, create indexes — across all tenants)
- Background jobs that need to query all tenants (nightly reports, billing jobs)
- Admin tools (customer support viewing a specific tenant's data)

**The danger:** Any connection that bypasses RLS is a loaded gun. If an application bug causes user A's request to run under the migration role, they see all data. Keep BYPASSRLS roles separate from the application role.

```python
# Two separate engines: one for the app, one for migrations
app_engine = create_async_engine(
    "postgresql://app_user:pass@db/mydb"  # no BYPASSRLS
)

migration_engine = create_async_engine(
    "postgresql://migration_user:pass@db/mydb"  # BYPASSRLS
)
# Never use migration_engine in request handlers
```

### RLS Performance

RLS adds a predicate to every query. PostgreSQL's planner sees the expanded query:

```sql
-- What you write
SELECT * FROM items WHERE id = $1;

-- What PostgreSQL actually runs
SELECT * FROM items
WHERE id = $1
AND tenant_id = current_setting('app.current_tenant_id')::UUID;
```

If `tenant_id` has no index, this becomes a Seq Scan on the full table filtered by tenant — expensive as the table grows.

**Fix: partial index per tenant_id**

```sql
-- Composite index covering tenant_id first (matches the RLS predicate)
CREATE INDEX CONCURRENTLY idx_items_tenant_id
    ON items (tenant_id, created_at DESC);

-- For family context queries
CREATE INDEX CONCURRENTLY idx_items_family_id
    ON items (family_id, visibility)
    WHERE family_id IS NOT NULL;
```

With this index, every query with the RLS tenant predicate does an Index Scan on just that tenant's rows — O(log N) even on a 100 million row table.

**Check that RLS does not break index usage:**

```sql
-- Test as the app user with RLS active
SET ROLE app_user;
SELECT set_config('app.current_tenant_id', 'abc123', true);
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM items ORDER BY created_at DESC LIMIT 20;
-- Should show: Index Scan on idx_items_tenant_id
-- Should NOT show: Seq Scan on items
```

### Testing RLS in CI

```sql
-- Test: User A cannot read User B's private data even with direct SQL
BEGIN;

SET ROLE app_user;  -- switch to the application database role
SELECT set_config('app.current_tenant_id', 'tenant-a-uuid', true);

-- Should return 0 rows (tenant B's data is invisible)
SELECT COUNT(*) FROM items WHERE tenant_id = 'tenant-b-uuid';
-- Expected: 0

-- Should return only tenant A's rows
SELECT COUNT(*) FROM items;
-- Expected: (number of rows for tenant A only)

ROLLBACK;
```

```python
# pytest integration test
@pytest.mark.asyncio
async def test_rls_tenant_isolation(db: AsyncSession):
    """Tenant A cannot see Tenant B's items even with direct query."""
    # Create two tenants
    tenant_a = await create_tenant(db, name="Family A")
    tenant_b = await create_tenant(db, name="Family B")

    # Create an item for tenant B
    item_b = await create_item(db, tenant_id=tenant_b.id, content="Family B secret")

    # Set context as Tenant A
    await db.execute(
        text("SELECT set_config('app.current_tenant_id', :tid, true)"),
        {"tid": str(tenant_a.id)}
    )

    # Attempt to read tenant B's item — should be invisible
    result = await db.execute(
        select(Item).where(Item.id == item_b.id)
    )
    assert result.scalar_one_or_none() is None, "RLS failed: Tenant A can see Tenant B's data"
```

---

## Part 3: Family/Group Data Model

### The Family as a Tenant

In a family app, a "family" is a tenant. Multiple users belong to the same family. Data can be:
- **Private to the creator** (`owner_id` only, `visibility = 'private'`)
- **Shared with the family** (`family_id` + `visibility = 'family'`)
- **Public** (visible to everyone, `visibility = 'public'`)

Think of it like a shared Google Drive folder:
- Some files are in "My Drive" (private)
- Some files are in the shared "Family" folder (family-scoped)
- Some files have a public share link (public)

### The Core Schema

```sql
-- Users
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    family_id   UUID REFERENCES families(id) ON DELETE SET NULL,  -- nullable!
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Families (the tenant entity)
CREATE TABLE families (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    owner_id    UUID NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Family membership with roles
CREATE TABLE family_members (
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id   UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    role        TEXT NOT NULL CHECK (role IN ('owner', 'member', 'viewer')),
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, family_id)
);

-- Items with visibility
CREATE TABLE items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id   UUID REFERENCES families(id) ON DELETE SET NULL,  -- nullable!
    visibility  TEXT NOT NULL DEFAULT 'private'
                    CHECK (visibility IN ('private', 'family', 'public')),
    content     TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Why `family_id` is Nullable

A user who has not joined a family is a solo user. They can still use the app; they just cannot share data with a family. Making `family_id NOT NULL` would force every user to belong to a family before they can create any data — a terrible onboarding experience.

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    email: Mapped[str] = mapped_column(String, unique=True, nullable=False)
    family_id: Mapped[UUID | None] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("families.id", ondelete="SET NULL"),
        nullable=True  # Explicit: solo users have no family
    )
    family: Mapped["Family | None"] = relationship("Family", back_populates="users")
```

### Why a Separate `family_members` Table

Why not just `users.family_id` and `users.role`? Because:

1. **Roles need history.** If you track `joined_at` and potentially `left_at`, you need a row per membership period.
2. **Auditability.** Who invited whom, when they accepted, when they left — all stored as rows in `family_members`.
3. **Easy membership queries.** "List all members of family X with their roles" is one query on `family_members`, not a self-join on `users`.
4. **Future flexibility.** A user could eventually be in multiple families (extended family scenarios, step-parents). A single `family_id` column on users can only model one family per user.

```python
class FamilyMember(Base):
    __tablename__ = "family_members"

    user_id: Mapped[UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"),
        primary_key=True
    )
    family_id: Mapped[UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("families.id", ondelete="CASCADE"),
        primary_key=True
    )
    role: Mapped[str] = mapped_column(
        String, nullable=False,
        # Enforce at ORM level too (DB CHECK constraint is the real guard)
    )
    joined_at: Mapped[datetime] = mapped_column(
        TIMESTAMPTZ, nullable=False, default=lambda: datetime.now(UTC)
    )
```

### The Visibility Enum

```python
from enum import StrEnum

class Visibility(StrEnum):
    PRIVATE = "private"    # Only the owner
    FAMILY = "family"      # All family members
    PUBLIC = "public"      # Anyone (no auth required)
```

**Access logic:**
```python
def can_read_item(item: Item, user: User) -> bool:
    if item.visibility == Visibility.PUBLIC:
        return True
    if item.owner_id == user.id:
        return True
    if item.visibility == Visibility.FAMILY:
        # user must be in the same family as the item
        return (
            item.family_id is not None
            and user.family_id is not None
            and item.family_id == user.family_id
        )
    return False
```

### RLS Policy for Family Items

This is the most important policy — it encodes the full access logic at the database layer:

```sql
-- Enable + Force on items table
ALTER TABLE items ENABLE ROW LEVEL SECURITY;
ALTER TABLE items FORCE ROW LEVEL SECURITY;

-- Read policy: owner OR family member (when family-scoped) OR public
CREATE POLICY items_read_policy ON items
    AS PERMISSIVE FOR SELECT
    TO app_user
    USING (
        -- Case 1: Public items visible to everyone
        visibility = 'public'
        OR
        -- Case 2: Owner always sees their own items
        owner_id = current_setting('app.current_user_id')::UUID
        OR
        -- Case 3: Family member sees family-scoped items
        (
            visibility = 'family'
            AND family_id IS NOT NULL
            AND family_id = current_setting('app.current_family_id')::UUID
        )
    );

-- Write policy: owner only (or family member for family-scoped)
CREATE POLICY items_write_policy ON items
    AS PERMISSIVE FOR INSERT
    TO app_user
    WITH CHECK (
        -- Creator must be the current user
        owner_id = current_setting('app.current_user_id')::UUID
        AND
        -- If claiming a family_id, it must be the user's actual family
        (
            family_id IS NULL
            OR family_id = current_setting('app.current_family_id')::UUID
        )
    );

-- Delete policy: only owner can delete
CREATE POLICY items_delete_policy ON items
    AS PERMISSIVE FOR DELETE
    TO app_user
    USING (owner_id = current_setting('app.current_user_id')::UUID);
```

Now the application sets both `app.current_user_id` and `app.current_family_id` on every request:

```python
async def get_db(
    current_user: User = Depends(get_current_user),
) -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(engine) as session:
        await session.execute(
            text("""
                SELECT
                    set_config('app.current_user_id', :uid, true),
                    set_config('app.current_family_id', :fid, true)
            """),
            {
                "uid": str(current_user.id),
                "fid": str(current_user.family_id) if current_user.family_id else ""
            }
        )
        yield session
```

---

## Part 4: RBAC for Family Context

### Role Hierarchy

```
owner → member → viewer
  │         │        │
  │         │        └─ can: read family data
  │         └─ can: read + create items in family scope
  └─ can: read + create + delete + manage members + delete family
```

This is a simple linear hierarchy. A member can do everything a viewer can, plus more. An owner can do everything a member can, plus more.

```python
from enum import IntEnum

class FamilyRole(IntEnum):
    """Higher number = more privilege. Comparison works: MEMBER >= VIEWER."""
    VIEWER = 1
    MEMBER = 2
    OWNER = 3

    @classmethod
    def from_str(cls, role: str) -> "FamilyRole":
        return cls[role.upper()]
```

### The Dependency Chain

FastAPI dependency injection makes role checking composable. Each layer depends on the layer below:

```
HTTP Request
    │
    ▼
get_current_user (verify JWT, load user from DB)
    │
    ▼
require_family_member (user must be in family, any role)
    │
    ▼
require_family_role("member") (user must have at least 'member' role)
    │
    ▼
Route handler (business logic)
```

```python
from fastapi import Depends, HTTPException, Path
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

async def get_family_membership(
    family_id: UUID = Path(...),
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> FamilyMember:
    """Verify the current user belongs to the specified family."""
    result = await db.execute(
        select(FamilyMember).where(
            FamilyMember.family_id == family_id,
            FamilyMember.user_id == current_user.id,
        )
    )
    membership = result.scalar_one_or_none()
    if not membership:
        # 404, not 403: don't reveal the family exists
        raise HTTPException(status_code=404, detail="Family not found")
    return membership


def require_family_role(min_role: str):
    """
    Factory that returns a dependency requiring at least min_role.

    Usage:
        @router.delete("/families/{family_id}")
        async def delete_family(
            membership: FamilyMember = Depends(require_family_role("owner"))
        ):
            ...
    """
    required = FamilyRole.from_str(min_role)

    async def _check(
        membership: FamilyMember = Depends(get_family_membership),
    ) -> FamilyMember:
        if FamilyRole.from_str(membership.role) < required:
            raise HTTPException(
                status_code=403,
                detail=f"This action requires {min_role} role"
            )
        return membership

    return _check
```

### Using the Dependencies

```python
router = APIRouter(prefix="/families/{family_id}")

@router.get("/items")
async def list_family_items(
    # Any family member can list items
    membership: FamilyMember = Depends(require_family_role("viewer")),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Item).where(
            Item.family_id == membership.family_id,
            Item.visibility == "family",
        ).order_by(Item.created_at.desc())
    )
    return result.scalars().all()


@router.post("/items")
async def create_family_item(
    body: ItemCreate,
    # Must be at least 'member' to create
    membership: FamilyMember = Depends(require_family_role("member")),
    db: AsyncSession = Depends(get_db),
):
    item = Item(
        owner_id=membership.user_id,
        family_id=membership.family_id,
        visibility="family",
        content=body.content,
    )
    db.add(item)
    return item


@router.delete("")
async def delete_family(
    # Only owner can delete the entire family
    membership: FamilyMember = Depends(require_family_role("owner")),
    db: AsyncSession = Depends(get_db),
):
    family = await db.get(Family, membership.family_id)
    await db.delete(family)
    return {"deleted": True}
```

### JWT with Family Claims vs DB Lookup

Two options for passing family context in a JWT:

**Option A: Store `family_id` and `role` in the JWT payload**
```python
# JWT payload
{"sub": "user-uuid", "family_id": "family-uuid", "family_role": "member", "exp": ...}
```
Pros: No DB lookup per request for family context.
Cons: If the user's role changes (owner kicks member), the old JWT is still valid until expiry. Requires short JWT TTL (15 minutes) or token revocation.

**Option B: DB lookup per request**
```python
async def get_current_user_with_family(
    token_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> User:
    # Reload user from DB to get current family_id
    # Also validates the user still exists and is active
    user = await db.get(User, token_user.id)
    if not user:
        raise HTTPException(status_code=401)
    return user
```
Pros: Always reflects current state. Revocation works instantly.
Cons: One extra DB query per request.

**Recommendation:** Use the DB lookup approach for family role checks. The DB query is one indexed lookup (milliseconds). The cost is negligible; the correctness benefit is real. Cache the result in the request state if you need the family context in multiple dependencies.

---

## Part 5: Invite System

### The Security Requirements

An invite system has several failure modes:
1. **Token guessing** — attacker tries random tokens to join a family
2. **Token reuse** — same invite link used by multiple people
3. **Token exposure in DB** — if DB is stolen, all pending invites are compromised
4. **Expired links still working** — invite sent 6 months ago still works

The fix for all four: store a SHA-256 hash of the token (not the raw token), set an expiry, mark used on first use.

### The Invite Schema

```sql
CREATE TABLE family_invites (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id   UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL UNIQUE,  -- SHA-256 of the raw token, NOT the token itself
    created_by  UUID NOT NULL REFERENCES users(id),
    expires_at  TIMESTAMPTZ NOT NULL,
    used_by     UUID REFERENCES users(id),      -- NULL until used
    used_at     TIMESTAMPTZ,                    -- NULL until used
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_family_invites_token_hash ON family_invites(token_hash);
CREATE INDEX idx_family_invites_family_id ON family_invites(family_id);
```

```python
class FamilyInvite(Base):
    __tablename__ = "family_invites"

    id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    family_id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("families.id"))
    token_hash: Mapped[str] = mapped_column(String, unique=True, nullable=False)
    created_by: Mapped[UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    expires_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, nullable=False)
    used_by: Mapped[UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    used_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=lambda: datetime.now(UTC))
```

### Generating an Invite

```python
import secrets
import hashlib
from datetime import UTC, datetime, timedelta

def hash_token(raw_token: str) -> str:
    """SHA-256 of the token. Store this; compare against this. Never store raw."""
    return hashlib.sha256(raw_token.encode()).hexdigest()


@router.post("/families/{family_id}/invites")
async def create_invite(
    membership: FamilyMember = Depends(require_family_role("owner")),
    db: AsyncSession = Depends(get_db),
) -> dict:
    # Generate a cryptographically random token (256 bits)
    raw_token = secrets.token_urlsafe(32)  # 43-character URL-safe base64 string

    invite = FamilyInvite(
        family_id=membership.family_id,
        token_hash=hash_token(raw_token),
        created_by=membership.user_id,
        expires_at=datetime.now(UTC) + timedelta(days=7),  # 7 days to accept
    )
    db.add(invite)
    await db.flush()  # get the ID

    return {
        "invite_url": f"https://app.example.com/join?token={raw_token}",
        "expires_at": invite.expires_at.isoformat(),
        # Note: raw_token is returned ONCE and never stored
    }
```

### Accepting an Invite

```python
class AcceptInviteRequest(BaseModel):
    token: str


@router.post("/join")
async def accept_invite(
    body: AcceptInviteRequest,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> dict:
    # Hash the supplied token to look it up
    token_hash = hash_token(body.token)

    result = await db.execute(
        select(FamilyInvite).where(
            FamilyInvite.token_hash == token_hash,
        )
    )
    invite = result.scalar_one_or_none()

    # Always same error message regardless of which check fails
    # (prevents enumeration: "is this token valid?", "is it expired?")
    invalid_msg = "Invalid or expired invite"

    if not invite:
        raise HTTPException(status_code=400, detail=invalid_msg)

    if invite.used_by is not None:
        # Already used — single-use token
        raise HTTPException(status_code=400, detail=invalid_msg)

    if invite.expires_at < datetime.now(UTC):
        # Expired
        raise HTTPException(status_code=400, detail=invalid_msg)

    if current_user.family_id is not None:
        # Already in a family — must leave first
        raise HTTPException(
            status_code=409,
            detail="You are already in a family. Leave your current family first."
        )

    # Accept: create membership, mark invite used, update user's family_id
    # All three writes in one transaction
    async with db.begin_nested():  # savepoint within the outer transaction
        db.add(FamilyMember(
            user_id=current_user.id,
            family_id=invite.family_id,
            role="member",
        ))

        current_user.family_id = invite.family_id

        invite.used_by = current_user.id
        invite.used_at = datetime.now(UTC)

    return {"joined_family_id": str(invite.family_id)}
```

### Token Security Details

**Why `secrets.token_urlsafe(32)` and not UUID?**
`uuid4()` generates 122 bits of randomness. `secrets.token_urlsafe(32)` generates 256 bits — larger search space, harder to guess. Both are fine for invite tokens in practice, but `secrets` is the correct module for security tokens.

**Why SHA-256 and not HMAC?**
For invite tokens, SHA-256 is sufficient. HMAC adds a server-side secret key, which means:
- If the key rotates, old token hashes are permanently invalid
- There is no additional security benefit for invite tokens (you are not verifying the token was *your* server that created it — any path to the token_hash lookup confirms authenticity)

Use HMAC-SHA256 when you need to verify both token content AND server origin (e.g., signed cookies, webhook signatures). Use SHA-256 alone when the token is random and you are just using the hash as a lookup key.

**Why `hmac.compare_digest` for comparisons?**
If you ever compare token hashes as strings (not SQL lookups), use `hmac.compare_digest(a, b)` to prevent timing attacks:

```python
import hmac

# WRONG — timing attack: short-circuits on first differing byte
if stored_hash == supplied_hash:
    ...

# CORRECT — constant-time comparison
if hmac.compare_digest(stored_hash, supplied_hash):
    ...
```

In the DB lookup path above, the comparison happens in SQL (`WHERE token_hash = :hash`), which is effectively constant-time due to the index lookup. But any Python-level comparison of security tokens must use `hmac.compare_digest`.

### Revocation

```python
@router.delete("/families/{family_id}/invites/{invite_id}")
async def revoke_invite(
    invite_id: UUID,
    membership: FamilyMember = Depends(require_family_role("owner")),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(FamilyInvite).where(
            FamilyInvite.id == invite_id,
            FamilyInvite.family_id == membership.family_id,  # ownership check
        )
    )
    invite = result.scalar_one_or_none()
    if not invite:
        raise HTTPException(status_code=404)

    await db.delete(invite)
    return {"revoked": True}
```

---

## Part 6: Shared AI Context (Family Memory)

### The Dual-Scope Problem

In an AI assistant app, memories power personalization. But in a family app, some memories are personal ("I prefer spicy food") and some are shared ("we are vegetarian as a family").

The query that builds context for an LLM call must return:
1. All of the **current user's private memories**
2. All of the **family's shared memories** (if the user is in a family)

But it must NOT return:
1. Other family members' private memories
2. Another family's shared memories

```
User A (family_id = F1)
├── Private memories (user_id = A, scope = 'user')     ← visible to A only
└── Family memories (family_id = F1, scope = 'family') ← visible to all F1 members

User B (family_id = F1)
├── Private memories (user_id = B, scope = 'user')     ← visible to B only
└── Reads the SAME family memories as A               ← shared F1 memories

User C (family_id = F2)
├── Cannot see F1's family memories                    ← hard boundary
└── Cannot see A or B's private memories               ← hard boundary
```

### The Memory Schema

```sql
CREATE TABLE memories (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id   UUID REFERENCES families(id) ON DELETE SET NULL,
    scope       TEXT NOT NULL CHECK (scope IN ('user', 'family')),
    content     TEXT NOT NULL,
    embedding   vector(1536),  -- pgvector for semantic search
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Constraint: family-scoped memories must have a family_id
    CONSTRAINT family_scope_requires_family_id
        CHECK (scope = 'user' OR family_id IS NOT NULL)
);
```

```python
class MemoryScope(StrEnum):
    USER = "user"      # Private to the creating user
    FAMILY = "family"  # Shared with all members of family_id
```

### The Memory Query

```python
async def get_relevant_memories(
    user: User,
    query_embedding: list[float],
    limit: int = 10,
    db: AsyncSession = Depends(get_db),
) -> list[Memory]:
    """
    Return memories relevant to the current user:
    - All of the user's own private memories
    - All family-shared memories (if user is in a family)

    Ordered by cosine similarity to the query embedding,
    with user-private memories given higher priority.
    """
    # Base condition: user's own memories
    user_condition = Memory.user_id == user.id

    # Family condition: family-scoped memories from user's family
    if user.family_id:
        family_condition = and_(
            Memory.scope == MemoryScope.FAMILY,
            Memory.family_id == user.family_id,
        )
        combined_condition = or_(user_condition, family_condition)
    else:
        # Solo user: only their own memories
        combined_condition = user_condition

    result = await db.execute(
        select(Memory)
        .where(combined_condition)
        .order_by(
            # pgvector cosine distance (lower = more similar)
            Memory.embedding.cosine_distance(query_embedding)
        )
        .limit(limit)
    )
    return result.scalars().all()
```

### Injection Priority: User-Private Over Family-Shared

When building context for an LLM, the user's specific preferences should override general family preferences. If the family memory says "we prefer chicken" but the user's private memory says "I am vegan since January", the personal preference wins.

```python
def build_memory_context(
    private_memories: list[Memory],
    family_memories: list[Memory],
) -> str:
    """
    Build the memory injection block for the LLM system prompt.
    Private memories are listed first and labeled as higher priority.
    """
    parts = []

    if private_memories:
        parts.append("## Personal Context (highest priority):")
        for m in private_memories:
            parts.append(f"- {m.content}")

    if family_memories:
        parts.append("\n## Family Context (lower priority, defer to personal if conflict):")
        for m in family_memories:
            parts.append(f"- {m.content}")

    return "\n".join(parts)
```

### Sharing a Memory to Family Scope

A user explicitly opts in — private memories are never automatically visible to the family.

```python
@router.post("/memories/{memory_id}/share")
async def share_memory_with_family(
    memory_id: UUID,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    if not current_user.family_id:
        raise HTTPException(status_code=400, detail="You are not in a family")

    result = await db.execute(
        select(Memory).where(
            Memory.id == memory_id,
            Memory.user_id == current_user.id,  # ownership check
        )
    )
    memory = result.scalar_one_or_none()
    if not memory:
        raise HTTPException(status_code=404)

    if memory.scope == MemoryScope.FAMILY:
        raise HTTPException(status_code=400, detail="Memory is already shared")

    # Create a NEW family-scoped copy (do not modify the original)
    family_copy = Memory(
        user_id=current_user.id,          # still owned by the creator
        family_id=current_user.family_id, # but scoped to the family
        scope=MemoryScope.FAMILY,
        content=memory.content,
        embedding=memory.embedding,       # reuse the embedding
    )
    db.add(family_copy)
    return {"shared": True, "family_memory_id": str(family_copy.id)}
```

### GDPR: Leaving the Family vs Deleting the Account

**Leaving the family:**
- Remove the user from `family_members`
- Set `user.family_id = NULL`
- Family-scoped memories (`scope = 'family'`, `family_id = F1`) remain — they are owned by the family, not the user personally. The user loses access (they are no longer a member), but the data is not deleted.
- User-private memories are unchanged.

```python
@router.post("/families/{family_id}/leave")
async def leave_family(
    membership: FamilyMember = Depends(get_family_membership),
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    if membership.role == "owner":
        raise HTTPException(
            status_code=400,
            detail="Owner cannot leave. Transfer ownership or delete the family."
        )

    # Remove membership
    await db.delete(membership)

    # Dissociate user from family (but do not delete family data)
    current_user.family_id = None

    # Family-scoped memories created by this user remain in the family
    # The family "owns" the shared memories; leaving removes access, not data
    return {"left": True}
```

**Deleting the account:**
- All user-private memories are deleted (cascade via `ON DELETE CASCADE` on `user_id`)
- Family-scoped memories created by this user: depends on your GDPR policy. Options:
  - Delete them (RIGHT TO ERASURE — strictest interpretation)
  - Anonymize them (`user_id = NULL`, content retained)
  - Keep them (content is "contributed to the family" and now belongs to it)

Document your policy explicitly. The most defensible GDPR approach is to delete all memories where `user_id = deleted_user_id`.

```python
async def delete_user_account(
    user: User,
    db: AsyncSession = Depends(get_db),
):
    # GDPR erasure: explicitly delete user-private memories
    await db.execute(
        delete(Memory).where(
            Memory.user_id == user.id,
            Memory.scope == MemoryScope.USER,
        )
    )

    # Family-scoped memories: anonymize (retain content, remove user attribution)
    await db.execute(
        update(Memory)
        .where(Memory.user_id == user.id, Memory.scope == MemoryScope.FAMILY)
        .values(user_id=None)  # requires nullable user_id or a "deleted" sentinel user
    )

    await db.delete(user)
```

---

## Part 7: Cross-User Data Access Audit

### Why Audit Cross-User Reads

In a family app, it is legitimate for member B to read member A's family-scoped data. But you still need a record of it — for GDPR data access requests ("show me everyone who has seen my data"), for security investigations, and for compliance.

The audit log is not a bug — it is a feature. GDPR Article 15 gives users the right to know who has accessed their data.

### The Audit Schema

```sql
CREATE TABLE family_data_access_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    accessor_user_id UUID NOT NULL REFERENCES users(id),  -- who accessed
    owner_user_id    UUID NOT NULL REFERENCES users(id),  -- whose data was accessed
    resource_type    TEXT NOT NULL,   -- 'item', 'memory', 'note', etc.
    resource_id      UUID NOT NULL,
    action           TEXT NOT NULL CHECK (action IN ('read', 'update', 'delete')),
    ip_address       INET,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No UPDATE or DELETE allowed: enforced at ORM level
);

-- Fast lookup: "who accessed user A's data?"
CREATE INDEX idx_access_log_owner ON family_data_access_log(owner_user_id, created_at DESC);
-- Fast lookup: "what has user B accessed?"
CREATE INDEX idx_access_log_accessor ON family_data_access_log(accessor_user_id, created_at DESC);
```

### Append-Only Enforcement at ORM Level

```python
from sqlalchemy import event

class FamilyDataAccessLog(Base):
    __tablename__ = "family_data_access_log"

    id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    accessor_user_id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    owner_user_id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    resource_type: Mapped[str] = mapped_column(String, nullable=False)
    resource_id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    action: Mapped[str] = mapped_column(String, nullable=False)
    ip_address: Mapped[str | None] = mapped_column(String)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=lambda: datetime.now(UTC))


# Block UPDATE and DELETE at the ORM event level
@event.listens_for(FamilyDataAccessLog, "before_update")
def prevent_audit_update(mapper, connection, target):
    raise RuntimeError("Audit log entries are immutable — UPDATE is forbidden")

@event.listens_for(FamilyDataAccessLog, "before_delete")
def prevent_audit_delete(mapper, connection, target):
    raise RuntimeError("Audit log entries are immutable — DELETE is forbidden")
```

**Also enforce at the database level:**

```sql
-- PostgreSQL trigger for belt-and-suspenders immutability
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'family_data_access_log is append-only. UPDATE and DELETE are forbidden.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_log_immutable
    BEFORE UPDATE OR DELETE ON family_data_access_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
```

### Logging Cross-User Reads

```python
async def log_cross_user_access(
    accessor_user_id: UUID,
    owner_user_id: UUID,
    resource_type: str,
    resource_id: UUID,
    action: str,
    ip_address: str | None,
    db: AsyncSession,
) -> None:
    """
    Log whenever one user accesses another user's data.
    Only log cross-user access (skip self-access: accessor == owner).
    """
    if accessor_user_id == owner_user_id:
        return  # No audit needed for self-access

    log_entry = FamilyDataAccessLog(
        accessor_user_id=accessor_user_id,
        owner_user_id=owner_user_id,
        resource_type=resource_type,
        resource_id=resource_id,
        action=action,
        ip_address=ip_address,
    )
    db.add(log_entry)
    # Does NOT flush immediately — let the calling code commit the transaction
    # This ensures the audit entry only persists if the actual access committed


# Usage in a route
@router.get("/items/{item_id}")
async def get_item(
    item_id: UUID,
    request: Request,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Item).where(Item.id == item_id)
        # RLS handles the visibility check
    )
    item = result.scalar_one_or_none()
    if not item:
        raise HTTPException(status_code=404)

    # Audit if this is a cross-user family read
    if item.owner_id != current_user.id and item.visibility == "family":
        await log_cross_user_access(
            accessor_user_id=current_user.id,
            owner_user_id=item.owner_id,
            resource_type="item",
            resource_id=item.id,
            action="read",
            ip_address=request.client.host if request.client else None,
            db=db,
        )

    return item
```

### GDPR Access Request: Compile Audit Log

```python
@router.get("/users/me/access-log")
async def get_my_access_log(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    limit: int = 100,
    before: datetime | None = None,
) -> list[AccessLogEntry]:
    """
    GDPR Article 15: show the user who has accessed their data.
    Returns cross-user access events where owner_user_id = current user.
    """
    query = (
        select(FamilyDataAccessLog)
        .where(FamilyDataAccessLog.owner_user_id == current_user.id)
        .order_by(FamilyDataAccessLog.created_at.desc())
        .limit(limit)
    )
    if before:
        query = query.where(FamilyDataAccessLog.created_at < before)

    result = await db.execute(query)
    return result.scalars().all()
```

---

## Part 8: Practical Code — Full Working Examples

### Complete RLS Policy for Family Items

```sql
-- 1. Enable and force RLS
ALTER TABLE items ENABLE ROW LEVEL SECURITY;
ALTER TABLE items FORCE ROW LEVEL SECURITY;

-- 2. Drop any existing policies
DROP POLICY IF EXISTS items_select ON items;
DROP POLICY IF EXISTS items_insert ON items;
DROP POLICY IF EXISTS items_update ON items;
DROP POLICY IF EXISTS items_delete ON items;

-- 3. SELECT: owner sees own items; family members see family-scoped items; public is open
CREATE POLICY items_select ON items
    AS PERMISSIVE FOR SELECT
    TO app_user
    USING (
        visibility = 'public'
        OR owner_id = current_setting('app.current_user_id', true)::UUID
        OR (
            visibility = 'family'
            AND family_id IS NOT NULL
            AND family_id = current_setting('app.current_family_id', true)::UUID
        )
    );

-- 4. INSERT: must be creating as yourself; if assigning a family, it must be your family
CREATE POLICY items_insert ON items
    AS PERMISSIVE FOR INSERT
    TO app_user
    WITH CHECK (
        owner_id = current_setting('app.current_user_id', true)::UUID
        AND (
            family_id IS NULL
            OR family_id = current_setting('app.current_family_id', true)::UUID
        )
    );

-- 5. UPDATE: only owner can update
CREATE POLICY items_update ON items
    AS PERMISSIVE FOR UPDATE
    TO app_user
    USING (owner_id = current_setting('app.current_user_id', true)::UUID)
    WITH CHECK (owner_id = current_setting('app.current_user_id', true)::UUID);

-- 6. DELETE: only owner can delete
CREATE POLICY items_delete ON items
    AS PERMISSIVE FOR DELETE
    TO app_user
    USING (owner_id = current_setting('app.current_user_id', true)::UUID);

-- 7. RESTRICTIVE tenant boundary (outer guard, never bypassed)
CREATE POLICY items_tenant_boundary ON items
    AS RESTRICTIVE FOR ALL
    TO app_user
    USING (
        -- Items with a family_id must belong to the current user's family
        family_id IS NULL
        OR family_id = current_setting('app.current_family_id', true)::UUID
        OR owner_id = current_setting('app.current_user_id', true)::UUID
    );
```

### FastAPI Dependency: `require_family_role` (Full Version)

```python
from typing import Callable
from uuid import UUID
from fastapi import Depends, HTTPException, Path
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models import FamilyMember, User
from app.auth import get_current_user
from app.db import get_db


class FamilyRole:
    RANK = {"viewer": 1, "member": 2, "owner": 3}

    @staticmethod
    def rank(role: str) -> int:
        return FamilyRole.RANK.get(role, 0)


async def get_family_membership(
    family_id: UUID = Path(..., description="Family UUID from URL path"),
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> FamilyMember:
    result = await db.execute(
        select(FamilyMember).where(
            FamilyMember.family_id == family_id,
            FamilyMember.user_id == current_user.id,
        )
    )
    membership = result.scalar_one_or_none()
    if not membership:
        raise HTTPException(status_code=404, detail="Family not found")
    return membership


def require_family_role(min_role: str = "viewer") -> Callable:
    """
    Dependency factory. Returns a FastAPI dependency that enforces a minimum role.

    Example:
        @router.delete("/families/{family_id}")
        async def delete_family(
            _: FamilyMember = Depends(require_family_role("owner"))
        ): ...
    """
    required_rank = FamilyRole.rank(min_role)

    async def _dependency(
        membership: FamilyMember = Depends(get_family_membership),
    ) -> FamilyMember:
        if FamilyRole.rank(membership.role) < required_rank:
            raise HTTPException(
                status_code=403,
                detail=f"Requires {min_role} role or higher"
            )
        return membership

    # Preserve the function name for FastAPI's OpenAPI schema
    _dependency.__name__ = f"require_{min_role}_role"
    return _dependency
```

### Invite Generation + Acceptance (Full Endpoints)

```python
import secrets
import hashlib
from datetime import UTC, datetime, timedelta
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models import Family, FamilyInvite, FamilyMember, User
from app.auth import get_current_user
from app.db import get_db
from app.families.deps import require_family_role

router = APIRouter()


def _hash_token(raw_token: str) -> str:
    return hashlib.sha256(raw_token.encode("utf-8")).hexdigest()


@router.post("/families/{family_id}/invites", status_code=201)
async def create_invite(
    membership: FamilyMember = Depends(require_family_role("owner")),
    db: AsyncSession = Depends(get_db),
) -> dict:
    raw_token = secrets.token_urlsafe(32)
    invite = FamilyInvite(
        family_id=membership.family_id,
        token_hash=_hash_token(raw_token),
        created_by=membership.user_id,
        expires_at=datetime.now(UTC) + timedelta(days=7),
    )
    db.add(invite)
    await db.flush()
    return {
        "invite_url": f"https://app.example.com/join?token={raw_token}",
        "expires_at": invite.expires_at.isoformat(),
        "invite_id": str(invite.id),
    }


class AcceptInviteBody(BaseModel):
    token: str


@router.post("/invites/accept", status_code=200)
async def accept_invite(
    body: AcceptInviteBody,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> dict:
    token_hash = _hash_token(body.token)

    result = await db.execute(
        select(FamilyInvite).where(FamilyInvite.token_hash == token_hash)
    )
    invite = result.scalar_one_or_none()

    # Generic error: do not reveal whether token exists, is expired, or was used
    err = HTTPException(status_code=400, detail="Invalid or expired invite link")

    if not invite:
        raise err
    if invite.used_by is not None:
        raise err
    if invite.expires_at < datetime.now(UTC):
        raise err

    if current_user.family_id is not None:
        raise HTTPException(
            status_code=409,
            detail="You are already in a family. Leave first to accept this invite."
        )

    # Atomic: membership + user update + invite mark-used
    db.add(FamilyMember(
        user_id=current_user.id,
        family_id=invite.family_id,
        role="member",
    ))
    current_user.family_id = invite.family_id
    invite.used_by = current_user.id
    invite.used_at = datetime.now(UTC)

    return {"joined_family_id": str(invite.family_id)}
```

### Family Memory Query: Private + Shared Scope

```python
from sqlalchemy import or_, and_, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Memory, MemoryScope, User


async def get_contextual_memories(
    user: User,
    query_embedding: list[float],
    db: AsyncSession,
    limit: int = 10,
) -> tuple[list[Memory], list[Memory]]:
    """
    Returns (private_memories, family_memories) for building LLM context.
    Separated so the caller can control injection priority.
    """
    # Private memories: only the current user's
    private_result = await db.execute(
        select(Memory)
        .where(
            Memory.user_id == user.id,
            Memory.scope == MemoryScope.USER,
        )
        .order_by(Memory.embedding.cosine_distance(query_embedding))
        .limit(limit)
    )
    private = private_result.scalars().all()

    # Family memories: shared within the user's family
    family: list[Memory] = []
    if user.family_id:
        family_result = await db.execute(
            select(Memory)
            .where(
                Memory.family_id == user.family_id,
                Memory.scope == MemoryScope.FAMILY,
            )
            .order_by(Memory.embedding.cosine_distance(query_embedding))
            .limit(limit)
        )
        family = family_result.scalars().all()

    return private, family
```

### Connection Checkout: Setting `app.current_tenant_id`

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from app.models import User

engine = create_async_engine(
    "postgresql+asyncpg://app_user:pass@localhost/mydb",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_db_for_user(
    current_user: User,
) -> AsyncGenerator[AsyncSession, None]:
    """
    Session dependency that sets RLS context variables for the current user.
    Always use transaction-scoped (true) set_config.
    """
    async with SessionLocal() as session:
        await session.execute(
            text("""
                SELECT
                    set_config('app.current_user_id', :uid, true),
                    set_config('app.current_family_id', :fid, true)
            """),
            {
                "uid": str(current_user.id),
                "fid": str(current_user.family_id) if current_user.family_id else "",
            }
        )
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Integration Test: RLS Prevents Cross-User Reads

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Family, FamilyMember, Item, User
from tests.factories import make_user, make_family, make_item


@pytest.mark.asyncio
async def test_user_cannot_read_family_members_private_items(
    client: AsyncClient,
    db: AsyncSession,
):
    """
    User B is in the same family as User A.
    User A has a private item.
    User B should NOT be able to read it.
    """
    # Setup
    family = make_family(name="Test Family")
    user_a = make_user(email="a@test.com", family=family)
    user_b = make_user(email="b@test.com", family=family)
    db.add_all([family, user_a, user_b])

    # Both are members of the family
    db.add(FamilyMember(user_id=user_a.id, family_id=family.id, role="member"))
    db.add(FamilyMember(user_id=user_b.id, family_id=family.id, role="member"))

    # User A creates a PRIVATE item
    private_item = make_item(
        owner=user_a,
        family=family,
        visibility="private",  # NOT family-scoped
        content="User A's private note"
    )
    db.add(private_item)
    await db.commit()

    # Authenticate as User B
    login_response = await client.post("/auth/login", json={
        "email": "b@test.com",
        "password": "testpass"
    })
    token_b = login_response.json()["access_token"]

    # Attempt to read User A's private item as User B
    response = await client.get(
        f"/items/{private_item.id}",
        headers={"Authorization": f"Bearer {token_b}"}
    )

    assert response.status_code == 404, (
        f"Expected 404 (not visible), got {response.status_code}. "
        "RLS or application filtering failed: User B can read User A's private item."
    )


@pytest.mark.asyncio
async def test_user_can_read_family_scoped_items(
    client: AsyncClient,
    db: AsyncSession,
):
    """
    User B is in the same family as User A.
    User A has a family-scoped item.
    User B SHOULD be able to read it.
    """
    family = make_family(name="Test Family")
    user_a = make_user(email="a2@test.com", family=family)
    user_b = make_user(email="b2@test.com", family=family)
    db.add_all([family, user_a, user_b])

    db.add(FamilyMember(user_id=user_a.id, family_id=family.id, role="member"))
    db.add(FamilyMember(user_id=user_b.id, family_id=family.id, role="member"))

    family_item = make_item(
        owner=user_a,
        family=family,
        visibility="family",  # Family-scoped
        content="Shared meal plan"
    )
    db.add(family_item)
    await db.commit()

    login_response = await client.post("/auth/login", json={
        "email": "b2@test.com",
        "password": "testpass"
    })
    token_b = login_response.json()["access_token"]

    response = await client.get(
        f"/items/{family_item.id}",
        headers={"Authorization": f"Bearer {token_b}"}
    )

    assert response.status_code == 200, (
        f"Expected 200, got {response.status_code}. "
        "User B should be able to read a family-scoped item."
    )
    assert response.json()["content"] == "Shared meal plan"


@pytest.mark.asyncio
async def test_cross_tenant_rls_isolation(
    db: AsyncSession,
):
    """
    Direct DB test: switching tenant context hides the other tenant's rows.
    """
    from sqlalchemy import text, select

    # Create two families with items
    family_a = make_family(name="A")
    family_b = make_family(name="B")
    user_a = make_user(family=family_a)
    user_b = make_user(family=family_b)
    db.add_all([family_a, family_b, user_a, user_b])
    item_a = make_item(owner=user_a, family=family_a, visibility="family")
    item_b = make_item(owner=user_b, family=family_b, visibility="family")
    db.add_all([item_a, item_b])
    await db.commit()

    # Set context to Family A
    await db.execute(
        text("SELECT set_config('app.current_user_id', :uid, true), "
             "set_config('app.current_family_id', :fid, true)"),
        {"uid": str(user_a.id), "fid": str(family_a.id)}
    )

    result = await db.execute(select(Item))
    visible_ids = {row.id for row in result.scalars().all()}

    assert item_a.id in visible_ids, "Family A item should be visible"
    assert item_b.id not in visible_ids, "Family B item should NOT be visible under Family A context"
```

---

## Summary: The Design Principles

1. **Push enforcement down.** RLS at the database is safer than WHERE clauses in application code. The database cannot forget to add the filter.

2. **Transaction-scoped config, always.** `set_config(..., true)` resets at transaction end. Connection pools reuse connections; session-scoped config leaks to the next request.

3. **FORCE RLS, not just ENABLE.** Table owners bypass RLS by default. FORCE removes that escape hatch.

4. **Separate roles for different access levels.** App user has RLS enforced. Migration user has BYPASSRLS. Never mix them.

5. **Hash invite tokens, not store them.** The raw token goes to the user; the hash goes to the database. DB theft does not compromise pending invites.

6. **GDPR: leaving ≠ deleting.** When a user leaves the family, they lose access to family data. When they delete their account, their data must be erased. These are different actions with different outcomes.

7. **Audit cross-user reads.** Legitimate family access is audited in append-only logs. Users can request their access history. Audit entries are immutable — no UPDATE or DELETE.

8. **Memory scope is explicit opt-in.** Private memories never automatically become family memories. Users explicitly share memories; the default is private.

9. **Role hierarchy via integer comparison.** `OWNER (3) >= MEMBER (2) >= VIEWER (1)`. A dependency factory that takes a minimum role integer is composable and avoids string comparison bugs.

10. **Cache keys must include tenant and family scope.** A cached item fetched under family F1 must never be served to a request from family F2.