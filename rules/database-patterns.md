---
description: SQLAlchemy 2.0 + PostgreSQL rules. Loads for all database-related files.
paths: ["models.py", "database.py", "db/**/*.py", "app/**/*.py"]
---

# Database Rules

> PATCH model handling → see `pydantic-patterns.md` for model_dump(exclude_unset=True)
> Async session rules → also see `async-patterns.md`

---

## Session Setup

1. Always use `AsyncSession` in FastAPI — NEVER sync `Session` in async routes
2. Always set `expire_on_commit=False` on `AsyncSession`
3. Create engine once at lifespan startup — NEVER inside a route or dependency
4. Use `postgresql+asyncpg://` for PostgreSQL async — NEVER plain `postgresql://`
5. Set `pool_pre_ping=True` on async engine — validates connection before use, mandatory for long-running processes
6. Set `pool_size = (DB max_connections × 0.8) / num_app_instances`
7. Use `Depends(get_db)` to inject `AsyncSession` into routes — NEVER create session inside route

## Querying (SQLAlchemy 2.0 style)

8. NEVER use `session.query()` — use `select()` with SQLAlchemy 2.0 style
9. Filter every user-data query by `user_id` or `tenant_id` — NEVER return all rows unfiltered
   (this is the DB-level enforcement of the isolation rule in `security.md`)
10. Use `await session.execute(select(...))` then `.scalars().all()` or `.scalar_one_or_none()`
11. Use `await session.flush()` before `await session.commit()` when you need the generated id

## Relationships

12. Set `lazy="raise"` on ALL model relationships — forces explicit loading strategy, prevents silent N+1
13. Use `selectinload` for one-to-many relationships (runs one extra SELECT ... WHERE id IN (...))
14. Use `joinedload` for many-to-one (single parent) — do NOT use for one-to-many (inflates rows)
15. NEVER access relationship attributes without selectinload/joinedload — causes MissingGreenletError

## PATCH Operations

16. Use `model_dump(exclude_unset=True)` + setattr loop — see `pydantic-patterns.md` for why
    ```python
    updates = body.model_dump(exclude_unset=True)
    for field, value in updates.items():
        setattr(db_obj, field, value)
    await session.commit()
    ```

## Migrations (Alembic)

17. NEVER use `Base.metadata.create_all()` in production — use Alembic migrations
18. Always run `alembic revision --autogenerate` + `alembic upgrade head` after changing a model
19. Always add `nullable=True` when adding a new column to a live table (zero-downtime deploy)
20. NEVER add `nullable=False` column to a live table without a default value
21. Use `CREATE INDEX CONCURRENTLY` for new indexes on production tables — non-blocking
22. Run `alembic upgrade head` in deploy pipeline — NEVER in lifespan startup hook in production
23. NEVER use `DROP COLUMN` in a single migration on a live table — use multi-step deploy:
    Step 1: deploy code that stops writing to the column
    Step 2: migration removes the column
    (skipping step 1 = running code crashes trying to write a now-missing column)

## Indexes

24. Add index on any column used in `WHERE` clauses on large tables
25. Use composite indexes for multi-column filters — leftmost column filters first
26. Use partial indexes for filtered queries: `WHERE is_active = true`
27. Use `INCLUDE` columns for index-only scans when SELECT columns are predictable
28. Run `EXPLAIN ANALYZE` on any query taking > 100ms — `Seq Scan` on large table = missing index
29. Add `unique=True` to `mapped_column()` for uniqueness constraints — this is enforced at DB level, not just application level
30. Use `index=True` on foreign key columns — SQLAlchemy does NOT add these automatically

## PostgreSQL Isolation Levels

31. Default isolation: `READ COMMITTED` — each statement sees only committed data
32. Use `REPEATABLE READ` for operations that need a consistent snapshot across multiple queries
    (e.g. reading totals + row counts must see the same data snapshot)
33. Use `SERIALIZABLE` only for full correctness with conflicting concurrent writes — requires retry logic on serialization failure
34. NEVER assume `READ COMMITTED` protects against phantom reads — it does not

## Locking and Concurrency

35. Use `with_for_update()` when multiple requests might update the same row — prevents lost updates
    ```python
    # Pessimistic lock: row is locked until transaction commits/rolls back
    stmt = select(Note).where(Note.id == note_id).with_for_update()
    note = (await session.execute(stmt)).scalar_one_or_none()
    ```
    — Use when: inventory decrement, wallet balance update, any read-then-write operation
36. Use `with_for_update(skip_locked=True)` for job queue patterns — skips rows already locked by other workers
37. Use `await session.refresh(obj)` to reload an object's state from DB after an operation that may have changed it externally

## Cascade Delete

38. Use `cascade="all, delete-orphan"` on parent → child relationships
39. Use `selectinload` when deleting a parent so ORM can cascade without MissingGreenletError

## Soft Delete

40. Use `deleted_at: datetime | None` column for soft delete — NEVER physically delete user data unless legally required
41. Add `WHERE deleted_at IS NULL` to ALL queries — use a base query helper or SQLAlchemy event listener so it's never forgotten
42. NEVER show soft-deleted records to users — treat them as if they don't exist at the application layer

## returning() — Atomic Read-After-Write

43. Use `.returning(Model)` on INSERT/UPDATE to get the affected row in one round-trip — avoids a redundant SELECT:
    ```python
    from sqlalchemy import insert

    stmt = (
        insert(Note)
        .values(title=data.title, user_id=user_id)
        .returning(Note)
    )
    result = await session.execute(stmt)
    note = result.scalar_one()
    ```
44. NEVER do `session.add(obj); await session.flush(); await session.refresh(obj)` when `.returning()` achieves the same result atomically in one query

## Connection Pool Monitoring

45. Log a warning when pool utilisation exceeds 80% — silent exhaustion causes cascading timeouts:
    ```python
    from sqlalchemy import event

    @event.listens_for(engine.sync_engine, "checkout")
    def on_checkout(dbapi_conn, record, proxy):
        used = engine.pool.checkedout()
        size = engine.pool.size()
        if used / size >= 0.8:
            logger.warning("db_pool_near_exhaustion", checked_out=used, pool_size=size)
    ```
46. At 100% pool utilisation all requests queue silently — tune `pool_size` or investigate slow queries holding connections before scaling the pool
