# Performance — Study Notes (L13: Query Optimization, Caching, Profiling)

> Written from first principles. Every concept explained with analogies before code.

---

## Part 1: Profiling — Finding the Real Bottleneck

### The Doctor's Diagnosis Analogy

Before a doctor prescribes treatment, they run tests to find what is actually wrong.
Prescribing medicine for a broken bone is not just useless — it delays real treatment.

Performance optimization works the same way:
- **Guessing** where the bottleneck is = prescribing without diagnosis
- **Profiling** = running the blood tests first
- **Optimizing** = prescribing based on results

The single most common mistake senior engineers see in junior code: optimizing the wrong function because it "looked slow."

---

### Concept 1: Sampling vs Instrumentation Profilers

**The traffic camera analogy:**

Two ways to measure highway congestion:
1. **Sampling**: A helicopter flies over every 5 seconds and photographs who is on the road. You get a statistical picture — not every car, but accurate patterns. Low overhead, non-invasive.
2. **Instrumentation**: You install a sensor at every on-ramp and off-ramp. You see every car, exact times. But those sensors slow down traffic slightly.

**py-spy** = the helicopter (sampling profiler)
**cProfile** = the sensors at every ramp (instrumentation profiler)

---

### py-spy — The Production-Safe Profiler

py-spy attaches to a running Python process and samples the call stack at regular intervals (default: 100 samples/second). It never modifies the process — it reads memory directly using OS ptrace APIs.

**Why this matters for production:**
- Zero code changes required
- No import overhead
- Attach to a live process that is already struggling
- Does not change the behavior you are trying to measure

```bash
# Install
pip install py-spy

# Profile a running process (PID from ps aux)
sudo py-spy top --pid 12345

# Record a flamegraph (visualize time per function)
sudo py-spy record -o flamegraph.svg --pid 12345 --duration 60

# Profile a script from start
py-spy record -o flamegraph.svg -- python my_script.py
```

**Reading a flamegraph:**
- X-axis = time (wider = more time spent)
- Y-axis = call stack depth (bottom = entry point, top = leaf function)
- Wide bars near the top of the stack = where time is actually spent
- A wide bar at the bottom but narrow at the top = overhead distributed across many leaf functions

**When to use py-spy:**
- Production performance incident (attach without restart)
- Long-running workers (Celery, background tasks)
- Any time you cannot afford instrumentation overhead

---

### pyinstrument — The Human-Readable Dev Profiler

pyinstrument is also a sampling profiler but wraps around code with a decorator or context manager. Designed for development: produces a clean tree output showing which function chains consumed the most wall time.

```python
from pyinstrument import Profiler

profiler = Profiler()
profiler.start()

# ... code you want to profile ...
result = expensive_operation()

profiler.stop()
profiler.print()
```

Output example:
```
  _     ._   __/__   _ _  _  _ _/_
 /_//_/// /_\ / //_// / //_'/ //

0.532 expensive_operation  my_module.py:45
  0.421 _fetch_items  my_module.py:62
    0.398 execute  sqlalchemy/engine/base.py:1414
      0.395 <many small functions>
  0.099 _serialize_results  my_module.py:88
```

This immediately shows: the database call is 75% of your time. The serialization is 18%. There is no point optimizing serialization until you fix the query.

**When to use pyinstrument:**
- Local development and debugging
- CI performance regression tests
- Profiling specific code paths in tests

---

### cProfile — The Standard Library Profiler

cProfile is Python's built-in instrumentation profiler. It counts every function call and measures cumulative time. More precise than sampling but adds overhead (5-20% slowdown).

```python
import cProfile
import pstats
import io

pr = cProfile.Profile()
pr.enable()

# ... code to profile ...
result = do_work()

pr.disable()

s = io.StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
ps.print_stats(20)  # top 20 functions
print(s.getvalue())
```

Key columns in cProfile output:
- `ncalls` — number of calls (high ncalls on slow functions = N+1 problem)
- `tottime` — time in THIS function, excluding subcalls
- `cumtime` — time in this function INCLUDING all subcalls
- `percall` — cumtime / ncalls (average cost per call)

**Pattern to spot N+1:**
If `ncalls` for a database function is 200 and you only called a list endpoint once, you have an N+1 problem. You fetched a list of 200 items and then queried the database once per item.

---

### asyncio Debug Mode — Finding Slow Coroutines

**The slow elevator analogy:**

In an async system, all coroutines share the same event loop thread. If one coroutine takes 200ms without yielding (awaiting), every other coroutine waits. This is like a slow person holding up an elevator — everyone queues.

asyncio debug mode logs any coroutine that blocks the event loop for more than 100ms.

```bash
# Enable via environment variable (production debugging)
PYTHONASYNCIODEBUG=1 python -m uvicorn app.main:app

# Enable programmatically
import asyncio
asyncio.get_event_loop().set_debug(True)
```

What it catches:
- CPU-bound code running in an async function (should use `run_in_executor`)
- Blocking I/O calls (file reads with `open()`, `time.sleep()`)
- Long-running synchronous database calls without async driver

Example log output:
```
Executing <Task 'get_user_profile'> took 0.342 seconds
```

That 342ms pause blocked every other request on that worker.

**Fix: Offload blocking work to thread pool:**
```python
import asyncio

async def get_user_profile(user_id: int) -> dict:
    # WRONG: blocking call on event loop
    # result = blocking_db_call(user_id)

    # CORRECT: run blocking call in thread pool
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_db_call, user_id)
    return result
```

---

### pg_stat_statements — Query-Level PostgreSQL Statistics

**The restaurant ticket analogy:**

A restaurant kitchen keeps every order ticket. At the end of the night you can sort by: which dish took longest, which dish was ordered most, which dish cost the most prep time in total. pg_stat_statements does this for SQL queries.

It tracks every unique query shape (normalized — parameters replaced with `$1`, `$2`) and records:
- `calls` — how many times this query ran
- `total_exec_time` — cumulative milliseconds
- `mean_exec_time` — average milliseconds per call
- `rows` — average rows returned
- `shared_blks_hit` / `shared_blks_read` — cache hits vs disk reads

```sql
-- Enable (requires superuser, restart not needed with shared_preload_libraries)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries by total time
SELECT
    query,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find queries with poor cache hit ratio (hitting disk)
SELECT
    query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(
        shared_blks_hit::numeric /
        NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 1
    ) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 100
ORDER BY shared_blks_read DESC
LIMIT 20;

-- Reset stats (do this after tuning to measure impact)
SELECT pg_stat_statements_reset();
```

**What to look for:**
- `mean_exec_time > 100ms` on frequently-called queries = index candidate
- `calls` very high with low `mean_exec_time` = N+1 victim (many small queries)
- Low `cache_hit_pct` = working set larger than `shared_buffers` (memory tuning needed)

---

### EXPLAIN ANALYZE — Reading the Query Plan

**The GPS navigation analogy:**

EXPLAIN shows the database's planned route before it drives. EXPLAIN ANALYZE actually drives the route and reports back real vs estimated times. BUFFERS shows how much was read from memory vs disk.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT u.id, u.email, p.bio
FROM users u
JOIN profiles p ON p.user_id = u.id
WHERE u.is_active = true
  AND u.created_at > '2024-01-01'
ORDER BY u.created_at DESC
LIMIT 50;
```

**Reading the output — key node types:**

| Node Type | Meaning | When It Appears |
|---|---|---|
| Seq Scan | Full table scan, reads every row | No usable index, or planner prefers it for small tables |
| Index Scan | Uses B-tree index, fetches heap rows | Index exists and is selective enough |
| Index Only Scan | Reads only the index, never touches heap | Covering index satisfies all columns |
| Bitmap Heap Scan | Uses index to build row bitmap, then scans heap | Multi-condition queries with partial overlap |
| Hash Join | Hashes the smaller table, probes with larger | Large table joins where nested loop would be N*M |
| Nested Loop | For each row in outer, scan inner | Small tables or index-backed inner side |
| Merge Join | Requires both inputs pre-sorted | Sort-heavy, rarely faster than Hash Join |

**The cost field:**

```
Seq Scan on users  (cost=0.00..4521.30 rows=98432 width=64)
                          ↑         ↑       ↑           ↑
                     startup  total cost  est rows    bytes/row
```

- `cost` = planner's estimated unit of work (not milliseconds)
- Compare `rows=98432` (estimated) to `actual rows=312` (real) — large gap = stale statistics, run `ANALYZE`

**The buffers field (with BUFFERS option):**

```
Buffers: shared hit=4521 read=8932
```
- `hit` = read from PostgreSQL shared_buffers (RAM) — fast
- `read` = read from OS page cache or disk — slower
- High `read` count = query is I/O bound, not CPU bound

**When Nested Loop = N+1:**

```
Nested Loop  (cost=... rows=50 ...)
  ->  Index Scan on users  (actual rows=50 loops=1)
  ->  Index Scan on profiles  (actual rows=1 loops=50)
                                                   ↑
                          50 loops = 50 separate index lookups
```

This is N+1 at the database level. The ORM executed one query to get 50 users, then 50 individual queries for profiles. Fix: use a JOIN in the ORM query or `selectinload` in SQLAlchemy.

**When Hash Join is better:**

For large tables where the inner side is not indexed or selectivity is low, a Hash Join pre-builds a hash table from the smaller relation and probes it once per row of the larger relation — O(N+M) instead of O(N*M).

```sql
-- Force Hash Join for testing (pg_hint_plan or disable nested loop)
SET enable_nestloop = off;
EXPLAIN ANALYZE SELECT ...;
SET enable_nestloop = on;
```

---

## Part 2: Database Optimization

### Concept 2: Index Types — The Phonebook Metaphor

A database index is like a phonebook organized for a specific kind of lookup. Different lookups need different organizations.

---

### B-tree Index — The Default, Most Versatile

**Analogy:** A sorted filing cabinet. You can find an exact drawer (equality) or find a range of drawers (range scan). Works for anything you can sort.

Supports: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`, `ORDER BY`
Does NOT support: `LIKE '%suffix'`, array containment, full-text search

```sql
-- Default index type (B-tree implied)
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_orders_created ON orders (created_at);

-- Composite B-tree — order matters
-- This index helps: WHERE tenant_id = $1 AND status = $2
-- This index helps: WHERE tenant_id = $1 (leading column)
-- This index does NOT help: WHERE status = $2 (non-leading column alone)
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
```

**Rule for composite index column order:**
1. Equality columns first (`WHERE tenant_id = $1`)
2. Range columns last (`AND created_at > $2`)
3. Sort columns at the end if frequently used (`ORDER BY created_at`)

---

### Hash Index — Equality Only, Slightly Faster Lookups

**Analogy:** A locker room with numbered lockers. You know exactly which locker to open for any key. No concept of "nearby" lockers. Cannot scan a range.

Supports: `=` only
Does NOT support: `<`, `>`, ordering, ranges

```sql
CREATE INDEX idx_sessions_token ON sessions USING hash (token);
```

In practice: B-tree equality lookups are nearly as fast as Hash. Hash indexes have fewer use cases. Use Hash only when the column is hash-partitioned and you need pure equality with no range operations ever.

---

### GIN Index — The Inverted Index (Arrays, JSONB, Full-Text)

**Analogy:** A book's index at the back. "Kubernetes" appears on pages 14, 87, 203. The index maps terms to page numbers. GIN maps element values to row IDs.

Supports: `@>` (contains), `<@` (contained by), `&&` (overlap), `@@` (full-text match), `?` (JSONB key exists)
Use cases: array columns, JSONB columns, tsvector columns

```sql
-- Array containment
CREATE INDEX idx_posts_tags ON posts USING gin (tags);
-- Query: SELECT * FROM posts WHERE tags @> ARRAY['python', 'fastapi'];

-- JSONB key/value search
CREATE INDEX idx_events_metadata ON events USING gin (metadata);
-- Query: SELECT * FROM events WHERE metadata @> '{"source": "api"}';

-- Full-text search
CREATE INDEX idx_articles_tsv ON articles USING gin (to_tsvector('english', body));
-- Query: SELECT * FROM articles WHERE to_tsvector('english', body) @@ to_tsquery('fastapi & async');
```

**GIN trade-off:** Slower to build and update than B-tree. Excellent for read-heavy workloads with array/JSONB data.

---

### GiST Index — Geometric and Range Types

**Analogy:** A map where you can find everything within a geographic bounding box. GiST supports "proximity" queries that B-tree cannot express.

Supports: geometric operators, range type operators (`&&`, `@>`, `-|-`), nearest-neighbor searches
Use cases: PostGIS geometry, `tsrange`, `daterange`, `int4range`, ip address ranges

```sql
-- Range type: find bookings that overlap a given date range
CREATE INDEX idx_bookings_period ON bookings USING gist (period);
-- Query: SELECT * FROM bookings WHERE period && '[2024-01-01, 2024-01-07)'::daterange;

-- PostGIS: find locations within a radius
CREATE INDEX idx_locations_geom ON locations USING gist (geom);
-- Query: SELECT * FROM locations WHERE ST_DWithin(geom, ST_MakePoint(-122.4, 37.8)::geography, 1000);
```

---

### BRIN Index — Bulk Range Index for Physically Ordered Data

**Analogy:** A warehouse with zones A-Z. BRIN records the min/max value stored in each zone. When you look for value 'M', it skips zones where min > 'M' or max < 'M'. Tiny index, fast exclusion — but only works when data is physically ordered.

Supports: range exclusion on monotonically-ordered data
Use cases: `created_at` on append-only tables (events, logs, time-series), `id` columns with sequential inserts

```sql
-- BRIN on an append-only events table (rows are physically ordered by time)
CREATE INDEX idx_events_created_brin ON events USING brin (created_at);
-- 128 pages per range (default). Index is tiny: ~1000x smaller than B-tree.
```

**When BRIN is NOT appropriate:**
- Data inserted out of order (UPDATE-heavy tables)
- When you need point lookups on random values
- Tables smaller than ~10 million rows (B-tree will be faster)

---

### Covering Index — Index-Only Scans

**Analogy:** A library catalog that lists not just the shelf location but also the book's title, author, and year. If you only need title/author/year, you never have to go to the shelf — the catalog has everything.

A covering index stores extra columns in the index so PostgreSQL can answer a query by reading only the index (Index Only Scan), never touching the heap (the actual table).

```sql
-- Without covering index: index scan finds row IDs, then heap fetch for email and name
CREATE INDEX idx_users_tenant ON users (tenant_id);

-- With covering index: Index Only Scan returns all needed columns from index alone
CREATE INDEX idx_users_tenant_cover ON users (tenant_id) INCLUDE (email, name, is_active);

-- Query that benefits:
SELECT email, name, is_active FROM users WHERE tenant_id = $1 ORDER BY email;
-- EXPLAIN will show: Index Only Scan (no heap fetches)
```

**Important:** The INCLUDE columns are not sorted in the index — they cannot be used for filtering or ordering. Only the key columns (before INCLUDE) can be used in WHERE or ORDER BY.

---

### Partial Index — Smaller, Faster, More Selective

**Analogy:** Instead of indexing all 1 million employees, only index the 5,000 currently active ones. Any query that filters by `is_active = true` can use this tiny index.

```sql
-- Only index active users (5% of table)
CREATE INDEX idx_users_active_email ON users (email) WHERE is_active = true;

-- Only index unprocessed jobs
CREATE INDEX idx_jobs_pending ON jobs (created_at, priority) WHERE status = 'pending';

-- Only index non-null values
CREATE INDEX idx_users_deleted_at ON users (deleted_at) WHERE deleted_at IS NOT NULL;
```

**The optimizer will only use a partial index if the query's WHERE clause implies the index condition.** The query `SELECT * FROM users WHERE email = $1 AND is_active = true` can use the partial index above. The query `SELECT * FROM users WHERE email = $1` cannot (it might return inactive users).

---

### Expression Index — Index a Transformed Value

**Analogy:** Instead of a phonebook sorted by original surnames, sort by lowercase surnames. Now case-insensitive searches work with binary search, not full scan.

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_lower_email ON users (lower(email));
-- Query MUST use same expression:
SELECT * FROM users WHERE lower(email) = lower($1);

-- Extract year from timestamp for fast year-based filtering
CREATE INDEX idx_orders_year ON orders (EXTRACT(year FROM created_at));
-- Query:
SELECT * FROM orders WHERE EXTRACT(year FROM created_at) = 2024;

-- JSONB field extraction
CREATE INDEX idx_events_source ON events ((metadata->>'source'));
-- Query:
SELECT * FROM events WHERE metadata->>'source' = 'api';
```

---

### CREATE INDEX CONCURRENTLY — Non-Blocking Index Creation

**Analogy:** Building a new road lane while traffic still flows on the existing lanes. Takes longer but no road closure.

Regular `CREATE INDEX` takes an exclusive lock — all writes to the table block until the index is built. On a large production table, this can mean minutes of downtime.

`CREATE INDEX CONCURRENTLY` builds the index in the background:
1. First pass: scans existing data while allowing writes
2. Records changes during the scan
3. Second pass: applies changes
4. Marks index valid only when fully consistent

```sql
-- Takes a lock that blocks only other DDL, not DML
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);

-- Check progress
SELECT phase, blocks_done, blocks_total, tuples_done, tuples_total
FROM pg_stat_progress_create_index
WHERE relid = 'orders'::regclass;
```

**Caveats:**
- Cannot be run inside a transaction (`BEGIN ... CREATE INDEX CONCURRENTLY` fails)
- If it fails, it leaves an INVALID index — drop it before retrying
- Takes 2-3x longer than non-concurrent build
- Check for INVALID indexes after: `SELECT indexname FROM pg_indexes WHERE ... AND pg_get_indexdef(indexname::regclass) LIKE '%INVALID%'`

---

### Connection Pool Sizing

**The toll booth analogy:**

A highway has 100 toll booths (PostgreSQL `max_connections`). If you open 200 cars at once, 100 must wait. The solution is not to send 200 cars — it is to queue them in a staging area (connection pool) and send batches of 80.

PostgreSQL's `max_connections` defaults to 100. Each connection uses ~5-10MB of RAM. For a 4GB RAM server, ~400 connections is the ceiling before memory pressure causes slowdowns.

**Formula:**

```
pool_size = (max_connections * 0.8) / num_app_instances
```

Example: 100 max_connections, 4 app instances, 20% reserved for admin tools:
```
pool_size = (100 * 0.8) / 4 = 20 connections per instance
```

**SQLAlchemy async pool configuration:**

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,           # Persistent connections maintained
    max_overflow=10,        # Extra connections allowed briefly above pool_size
    pool_timeout=30,        # Seconds to wait for a connection from pool
    pool_recycle=3600,      # Recycle connections older than 1 hour (prevents stale connections)
    pool_pre_ping=True,     # Send a ping before using a connection to detect dropped connections
)
```

**asyncpg pipelining:**

asyncpg (PostgreSQL async driver) supports pipelining — sending multiple queries over a single connection without waiting for each response. This is like sending 10 letters at once instead of waiting for each reply.

```python
# asyncpg pipelining: execute multiple queries in pipeline
async with conn.transaction():
    # These three queries are pipelined — sent in one network round trip
    await asyncio.gather(
        conn.execute("UPDATE users SET last_seen = NOW() WHERE id = $1", user_id),
        conn.execute("INSERT INTO events (user_id, type) VALUES ($1, $2)", user_id, "login"),
        conn.execute("UPDATE stats SET login_count = login_count + 1 WHERE user_id = $1", user_id),
    )
```

---

### VACUUM ANALYZE — Keeping Statistics Fresh

**The garbage truck analogy:**

PostgreSQL uses MVCC (Multi-Version Concurrency Control) — instead of overwriting rows, it creates new versions and marks old ones dead. VACUUM removes the dead rows and reclaims space. Without VACUUM, dead rows accumulate and queries get slower (table bloat).

**ANALYZE** updates the planner's statistics about data distribution. Stale statistics cause the query planner to choose wrong plans (e.g., Seq Scan instead of Index Scan because it thinks there are 1000 rows when there are 1 million).

```sql
-- Run both in one command (safe to run during normal operation)
VACUUM ANALYZE users;

-- Full VACUUM (reclaims disk space back to OS, requires exclusive lock)
-- Only use when table is very bloated and disk is critical
VACUUM FULL users;

-- Check table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**Autovacuum tuning for high-write tables:**

```sql
-- Per-table autovacuum override (for high-write tables)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- Trigger when 1% of rows are dead (default 20%)
    autovacuum_analyze_scale_factor = 0.005, -- Trigger ANALYZE when 0.5% changed
    autovacuum_vacuum_cost_delay = 2         -- Reduce cost delay for faster vacuuming
);
```

---

### Materialized Views — Pre-Computing Expensive Aggregates

**The dashboard analogy:**

A company dashboard shows total revenue, active users, and top products. Recomputing these on every page load would join 10 million order rows. Instead: compute it once per hour and store the result. That is a materialized view.

```sql
-- Create the materialized view
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
    date_trunc('day', created_at) AS day,
    SUM(amount) AS total_revenue,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY date_trunc('day', created_at)
ORDER BY day DESC
WITH DATA;  -- Populate immediately

-- Index on the materialized view
CREATE UNIQUE INDEX idx_mv_daily_revenue_day ON mv_daily_revenue (day);

-- Refresh (blocking — locks the view during refresh)
REFRESH MATERIALIZED VIEW mv_daily_revenue;

-- Refresh concurrently (non-blocking — requires a unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
```

**Triggering refresh:**

```python
# Option 1: Scheduled refresh (cron or Celery beat)
# Option 2: Trigger refresh from application after bulk write
async def complete_order(order_id: int, db: AsyncSession) -> None:
    await db.execute(update(Order).where(Order.id == order_id).values(status="completed"))
    await db.execute(text("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue"))
    await db.commit()
```

---

### PostgreSQL Partitioning — Splitting Large Tables

**The filing cabinet analogy:**

A filing cabinet with 10 years of records is slow to search because everything is mixed together. Partition it: one drawer per year. Searching 2024 records only opens the 2024 drawer.

PostgreSQL partitioning physically separates a large table into smaller child tables. The parent table is a logical view — queries go to the correct partition automatically.

```sql
-- Range partitioning by month (for time-series data)
CREATE TABLE events (
    id BIGSERIAL,
    tenant_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    payload JSONB
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Each partition has its own indexes
CREATE INDEX ON events_2024_01 (tenant_id, created_at);
CREATE INDEX ON events_2024_02 (tenant_id, created_at);

-- Hash partitioning by tenant_id (distribute load across partitions)
CREATE TABLE tenant_data (
    id BIGSERIAL,
    tenant_id UUID NOT NULL,
    data JSONB
) PARTITION BY HASH (tenant_id);

CREATE TABLE tenant_data_0 PARTITION OF tenant_data FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE tenant_data_1 PARTITION OF tenant_data FOR VALUES WITH (modulus 4, remainder 1);
CREATE TABLE tenant_data_2 PARTITION OF tenant_data FOR VALUES WITH (modulus 4, remainder 2);
CREATE TABLE tenant_data_3 PARTITION OF tenant_data FOR VALUES WITH (modulus 4, remainder 3);
```

**Partition pruning:** PostgreSQL eliminates irrelevant partitions at plan time. A query with `WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'` only scans `events_2024_01`.

Enable: `SET enable_partition_pruning = on;` (default on)
Verify: EXPLAIN shows `Partitions selected: 1 of 24`

---

## Part 3: Caching

### Concept 3: The Caching Spectrum — From Memory to Network

**The photocopier analogy:**

- **In-process cache**: The photocopy on your desk. Instant to access. Only you can see it. Gone when you go home.
- **Redis**: The shared printer room. Everyone can use it. Takes a few seconds to walk there. Available after you leave.
- **CDN cache**: The regional distribution center. Serves your entire city. Takes minutes to update.

Each level is slower and more shared than the previous. Choose based on: how consistent does the data need to be, how many processes need to share it, and how fast does it need to be.

---

### In-Process TTLCache (cachetools)

```python
from cachetools import TTLCache, cached
from threading import Lock

# Thread-safe TTLCache: max 1000 items, 5-minute TTL
_cache: TTLCache = TTLCache(maxsize=1000, ttl=300)
_lock = Lock()

@cached(cache=_cache, lock=_lock)
def get_config_value(key: str) -> str | None:
    return db.query_config(key)  # Only called on cache miss
```

**NEVER use in-process cache for:**
- Per-user data (each process has its own cache — user hits different instances, gets inconsistent data)
- Data that must be consistent across multiple app instances
- Data that must be invalidated from outside the process

**Use in-process cache for:**
- Application configuration loaded from DB
- Feature flags
- Translation strings
- Rarely-changing reference data (country codes, plan tiers)

---

### Redis Caching with orjson

**orjson vs json vs ujson:**

| Library | Serialize (MB/s) | Deserialize (MB/s) | Type support |
|---|---|---|---|
| json (stdlib) | 100 | 150 | Basic |
| ujson | 400 | 500 | Basic |
| orjson | 1200 | 1400 | datetime, UUID, dataclasses |

orjson is 10-50x faster than stdlib json and handles Python types (datetime, UUID, bytes, dataclasses) natively without custom encoders.

```python
import orjson
from redis.asyncio import Redis
from typing import Any

redis: Redis = Redis.from_url("redis://localhost:6379", decode_responses=False)

async def cache_get(key: str) -> Any | None:
    value = await redis.get(key)
    if value is None:
        return None
    return orjson.loads(value)

async def cache_set(key: str, value: Any, ttl_seconds: int = 300) -> None:
    # orjson.dumps returns bytes — Redis stores bytes natively
    await redis.setex(key, ttl_seconds, orjson.dumps(value))

async def cache_delete(key: str) -> None:
    await redis.delete(key)
```

---

### Cache Patterns

**1. Cache-Aside (Lazy Loading) — The Most Common Pattern**

Application checks cache first. On miss, loads from DB and populates cache. Data is only cached when needed.

```python
async def get_user(user_id: int, db: AsyncSession, redis: Redis) -> UserSchema:
    cache_key = f"user:{user_id}"

    # Check cache first
    cached = await cache_get(cache_key)
    if cached is not None:
        return UserSchema.model_validate(cached)

    # Cache miss: load from DB
    user = await db.get(User, user_id)
    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")

    # Populate cache for next request
    await cache_set(cache_key, user.model_dump(), ttl_seconds=300)
    return UserSchema.model_validate(user)
```

**Pros:** Simple. Only caches data that is actually requested. Cache miss is always consistent (reads from DB).
**Cons:** First request after TTL expiry hits DB (cold start). Cache can hold stale data until TTL expires.

**2. Write-Through — Strong Consistency**

On every write, update both DB and cache atomically. Cache is never stale.

```python
async def update_user(user_id: int, data: UserUpdateSchema, db: AsyncSession, redis: Redis) -> UserSchema:
    user = await db.get(User, user_id)
    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")

    for field, value in data.model_dump(exclude_unset=True).items():
        setattr(user, field, value)

    await db.commit()
    await db.refresh(user)

    # Update cache immediately after DB write
    cache_key = f"user:{user_id}"
    await cache_set(cache_key, user.model_dump(), ttl_seconds=300)

    return UserSchema.model_validate(user)
```

**Pros:** Cache always consistent with DB. No stale reads.
**Cons:** Every write pays the extra cache-write cost. Cache may fill with data never read again.

**3. Write-Behind (Write-Back) — Async Consistency**

Writes go to cache immediately. DB write happens asynchronously in background. Highest write performance, eventual consistency.

```
Write → Cache (immediate, synchronous)
             ↓ (async, background worker)
           Database
```

**Use only when:** write throughput matters more than durability guarantees. If the cache crashes before the background write completes, that data is lost. Rarely appropriate for user data in web applications.

**4. Read-Through**

Application reads through the cache layer. Cache is responsible for fetching from DB on miss (rather than the application). Libraries like Memcached with a read-through wrapper, or a caching proxy layer.

```python
# Application perspective: only one call
user = await cache_layer.get_user(user_id)
# Cache layer handles miss → DB fetch → populate → return internally
```

---

### Cache Invalidation Strategies

**The newspaper analogy:**

How do you know your newspaper is outdated?
1. **TTL**: Throw it away after 24 hours whether you've read it or not.
2. **Event-driven**: When the editor prints a correction, send you a new copy.
3. **Cache tags**: Tag all sports-related articles. When sports scores update, invalidate all tagged articles at once.

```python
# TTL invalidation — simplest, set at write time
await cache_set(f"user:{user_id}", data, ttl_seconds=300)

# Event-driven invalidation — explicit delete on data change
async def on_user_updated(user_id: int, redis: Redis) -> None:
    await redis.delete(f"user:{user_id}")
    await redis.delete(f"user_profile:{user_id}")  # Related keys

# Cache tag invalidation — group related keys
async def cache_set_with_tags(key: str, value: Any, tags: list[str], ttl: int) -> None:
    pipe = redis.pipeline()
    pipe.setex(key, ttl, orjson.dumps(value))
    for tag in tags:
        pipe.sadd(f"tag:{tag}", key)      # Add key to tag set
        pipe.expire(f"tag:{tag}", ttl)    # Tag set expires with data
    await pipe.execute()

async def invalidate_tag(tag: str, redis: Redis) -> None:
    keys = await redis.smembers(f"tag:{tag}")
    if keys:
        await redis.delete(*keys)
    await redis.delete(f"tag:{tag}")

# Usage: cache user and all their posts under "user:42" tag
await cache_set_with_tags("user:42", user_data, tags=["user:42"], ttl=300)
await cache_set_with_tags("posts:user:42", posts_data, tags=["user:42"], ttl=300)
# On user update: invalidate all "user:42" tagged entries
await invalidate_tag("user:42", redis)
```

---

### Cache Stampede Prevention

**The concert ticket analogy:**

A popular concert sells out. 10,000 fans refresh the ticket page every second. When a new batch of tickets drops, all 10,000 hit the database simultaneously. This is a cache stampede — when a cached item expires and many processes simultaneously try to rebuild it.

**Three solutions:**

**1. Probabilistic Early Expiry (XFetch algorithm):**

Don't wait for expiry. Randomly expire the cache slightly before TTL. The probability of early expiry increases as TTL approaches zero. Only one request regenerates the cache.

```python
import math
import time
import random

async def get_with_early_expiry(key: str, compute_fn, ttl: int, beta: float = 1.0) -> Any:
    raw = await redis.get(f"data:{key}")
    ttl_remaining = await redis.ttl(f"data:{key}")

    if raw is not None:
        # XFetch: early expiry decision
        delta = time.time() - (ttl - ttl_remaining)  # time since cached
        if delta - beta * math.log(random.random()) < ttl_remaining:
            return orjson.loads(raw)
        # Probabilistically decided to recompute early

    # Cache miss or early expiry: recompute
    value = await compute_fn()
    await cache_set(f"data:{key}", value, ttl_seconds=ttl)
    return value
```

**2. Distributed Lock (Mutex):**

First request to detect a miss acquires a lock. Subsequent requests wait for the lock. Only one request rebuilds the cache.

```python
import asyncio

async def get_with_lock(key: str, compute_fn, ttl: int, redis: Redis) -> Any:
    cached = await cache_get(key)
    if cached is not None:
        return cached

    lock_key = f"lock:{key}"
    lock_acquired = await redis.set(lock_key, "1", nx=True, ex=10)  # 10s lock TTL

    if lock_acquired:
        try:
            value = await compute_fn()
            await cache_set(key, value, ttl_seconds=ttl)
            return value
        finally:
            await redis.delete(lock_key)
    else:
        # Wait briefly and retry (another process is rebuilding)
        await asyncio.sleep(0.1)
        return await get_with_lock(key, compute_fn, ttl, redis)
```

**3. Background Refresh:**

Before TTL expires, schedule a background task to refresh the cache. Serve stale data until refresh completes.

```python
async def get_with_background_refresh(key: str, compute_fn, ttl: int, refresh_at: int) -> Any:
    cached = await cache_get(key)
    remaining = await redis.ttl(key)

    if cached is not None:
        if remaining < refresh_at:  # Nearing expiry
            asyncio.create_task(refresh_cache(key, compute_fn, ttl))  # Background refresh
        return cached  # Serve current (possibly slightly stale) value

    # True cache miss: synchronous rebuild
    return await refresh_cache(key, compute_fn, ttl)
```

---

### Semantic Caching for LLM Responses

**The librarian analogy:**

A librarian who has answered "what are the best Python books" can recognize that "which Python books do you recommend" asks essentially the same thing and give the same answer without re-reading all the books.

Semantic caching stores LLM responses indexed by the embedding (semantic meaning) of the input. New requests are compared by cosine similarity — if similar enough, return cached response.

```python
import hashlib
import numpy as np
from anthropic import AsyncAnthropic
import orjson

client = AsyncAnthropic()

async def embed_text(text: str) -> list[float]:
    # Use any embedding model (OpenAI, Cohere, sentence-transformers, etc.)
    response = await openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

async def cosine_similarity(a: list[float], b: list[float]) -> float:
    a_np = np.array(a)
    b_np = np.array(b)
    return float(np.dot(a_np, b_np) / (np.linalg.norm(a_np) * np.linalg.norm(b_np)))

async def semantic_cache_get(query: str, redis: Redis, threshold: float = 0.95) -> str | None:
    query_embedding = await embed_text(query)

    # Check stored embeddings (in production: use pgvector or Pinecone for ANN search)
    keys = await redis.keys("semantic:embed:*")
    for key in keys:
        stored = orjson.loads(await redis.get(key))
        similarity = await cosine_similarity(query_embedding, stored["embedding"])
        if similarity >= threshold:
            return stored["response"]
    return None

async def get_llm_response_cached(query: str, redis: Redis) -> str:
    cached = await semantic_cache_get(query, redis)
    if cached:
        return cached

    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": query}]
    )
    result = response.content[0].text

    # Store embedding and response
    embedding = await embed_text(query)
    cache_key = f"semantic:embed:{hashlib.sha256(query.encode()).hexdigest()}"
    await redis.setex(cache_key, 3600, orjson.dumps({"embedding": embedding, "response": result}))

    return result
```

---

### CDN Caching Headers

**Cache-Control anatomy:**

```
Cache-Control: public, max-age=3600, stale-while-revalidate=60
               ↑       ↑              ↑
         anyone can    cache 1 hour   serve stale while fetching fresh (60s window)
         cache this
```

| Directive | Meaning | Use Case |
|---|---|---|
| `public` | CDN and browsers can cache | Static assets, public API responses |
| `private` | Only browser can cache, not CDN | Per-user responses |
| `no-store` | Never cache anywhere | Payment pages, CSRF tokens |
| `no-cache` | Cache but always revalidate | Responses that need freshness checks |
| `max-age=N` | Cache for N seconds | Any cacheable resource |
| `s-maxage=N` | CDN-specific max-age | When CDN and browser should have different TTLs |
| `stale-while-revalidate=N` | Serve stale while fetching fresh in background | Tolerates brief staleness for lower latency |
| `immutable` | Content will never change | Versioned assets (bundle.abc123.js) |

```python
from fastapi import Response

@router.get("/api/config")
async def get_config(response: Response) -> ConfigSchema:
    response.headers["Cache-Control"] = "public, max-age=300, stale-while-revalidate=60"
    return await load_config()

@router.get("/api/user/profile")
async def get_profile(response: Response, current_user: User = Depends(get_current_user)) -> UserSchema:
    response.headers["Cache-Control"] = "private, max-age=60"
    return UserSchema.from_orm(current_user)

@router.get("/static/bundle.{hash}.js")
async def get_asset(hash: str, response: Response):
    response.headers["Cache-Control"] = "public, max-age=31536000, immutable"
    return FileResponse(f"static/bundle.{hash}.js")
```

---

## Part 4: Python-Specific Performance

### Concept 4: JSON Serialization Overhead

**The translation analogy:**

Every API response is a translation from Python objects (datetimes, UUIDs, SQLAlchemy models) to JSON bytes. stdlib `json` is a careful translator who double-checks everything. `orjson` is a specialist who knows Python types natively and works 10x faster.

```python
# FastAPI: replace default JSONResponse with ORJSONResponse globally
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)

# Individual route override
from fastapi.responses import ORJSONResponse

@router.get("/items", response_class=ORJSONResponse)
async def list_items() -> list[ItemSchema]:
    return await db.fetch_items()
```

ORJSONResponse handles: `datetime`, `UUID`, `bytes`, `numpy` arrays, `dataclass`, `Enum` — without custom encoders.

---

### uvloop — 2-4x Throughput Improvement

**The highway analogy:**

Python's default asyncio event loop is like a city street — functional but not built for throughput. uvloop replaces it with a highway: same traffic, much faster movement. It is a Cython reimplementation of asyncio's event loop on top of libuv (the same C library used by Node.js).

```python
# Install
# pip install uvloop

# Method 1: Set globally at startup (recommended)
import uvloop
uvloop.install()  # Call before any asyncio.run() or uvicorn startup

# Method 2: Uvicorn --loop flag
# uvicorn app.main:app --loop uvloop

# Method 3: In pyproject.toml uvicorn config
# [tool.uvicorn]
# loop = "uvloop"
```

**When uvloop helps most:**
- High-concurrency APIs (>1000 req/s)
- Lots of small async I/O operations
- WebSocket servers

**When it helps less:**
- CPU-bound work (the bottleneck is not the event loop)
- Very low traffic applications
- Windows (uvloop does not support Windows)

---

### asyncio.gather vs asyncio.TaskGroup

**asyncio.gather — Run coroutines in parallel, collect results:**

```python
import asyncio

async def fetch_user_data(user_id: int) -> dict:
    # Run three I/O operations in parallel
    user, posts, settings = await asyncio.gather(
        get_user(user_id),
        get_user_posts(user_id),
        get_user_settings(user_id),
    )
    return {"user": user, "posts": posts, "settings": settings}
```

**asyncio.TaskGroup (Python 3.11+) — Structured concurrency, better error handling:**

```python
async def fetch_user_data_structured(user_id: int) -> dict:
    async with asyncio.TaskGroup() as tg:
        user_task = tg.create_task(get_user(user_id))
        posts_task = tg.create_task(get_user_posts(user_id))
        settings_task = tg.create_task(get_user_settings(user_id))
    # All tasks guaranteed complete here

    return {
        "user": user_task.result(),
        "posts": posts_task.result(),
        "settings": settings_task.result(),
    }
```

**Key difference:** With `asyncio.gather`, if one coroutine raises an exception, the others continue running (unless `return_exceptions=False`). With `TaskGroup`, if any task raises, all remaining tasks are cancelled immediately. TaskGroup prevents zombie coroutines.

**Performance:** Both have similar throughput. TaskGroup has slightly cleaner semantics for structured concurrency. Prefer TaskGroup in new code (Python 3.11+).

---

### Batch Database Writes

**The post office analogy:**

Sending 1000 individual letters one at a time costs 1000 trips to the post office. Putting them all in one bag costs 1 trip. Batch inserts work the same way.

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

# SLOW: 1000 individual INSERTs (1000 network round trips)
async def insert_events_slow(events: list[EventCreate], db: AsyncSession) -> None:
    for event in events:
        db.add(Event(**event.model_dump()))
    await db.commit()

# FAST: Single bulk INSERT (1 network round trip)
async def insert_events_fast(events: list[EventCreate], db: AsyncSession) -> None:
    if not events:
        return

    await db.execute(
        pg_insert(Event).values([e.model_dump() for e in events])
    )
    await db.commit()

# FASTEST: COPY protocol (for >10k rows)
# asyncpg supports copy_records_to_table for maximum throughput
async def insert_events_copy(events: list[EventCreate], conn) -> None:
    await conn.copy_records_to_table(
        "events",
        records=[(e.tenant_id, e.event_type, e.payload) for e in events],
        columns=["tenant_id", "event_type", "payload"],
    )
```

**Benchmark comparison (10,000 rows):**
- Individual inserts: ~8-12 seconds
- Bulk `INSERT ... VALUES (...)`: ~0.3-0.8 seconds
- COPY protocol: ~0.05-0.1 seconds

---

### StreamingResponse for Large Payloads

**The garden hose analogy:**

Filling a bucket vs. drinking from a hose. If you fill a bucket (buffer all results in memory), you wait for the bucket to fill and use a lot of memory. If you drink from a hose (streaming), you get data immediately and memory stays low.

```python
from fastapi.responses import StreamingResponse
import orjson

async def stream_large_dataset(db: AsyncSession) -> StreamingResponse:
    async def generate():
        yield b"["
        first = True
        async for row in await db.stream(select(Event).order_by(Event.created_at)):
            if not first:
                yield b","
            yield orjson.dumps(EventSchema.model_validate(row).model_dump())
            first = False
        yield b"]"

    return StreamingResponse(
        generate(),
        media_type="application/json",
        headers={"Content-Disposition": "attachment; filename=export.json"}
    )
```

**When to use StreamingResponse:**
- CSV/JSON exports with millions of rows
- File downloads
- Server-sent events
- Any response where the full payload would exceed ~10MB in memory

**When NOT to use StreamingResponse:**
- Normal API responses (adds complexity, no benefit for small payloads)
- When clients need the Content-Length header (streaming omits it)
- When middleware needs to inspect the full response body

---

## Part 5: LLM Performance

### Concept 5: Token Economics

**The taxi meter analogy:**

Every token is a tick of the taxi meter. Input tokens cost less than output tokens. The longer your prompt, the more the ride costs before you even start moving. The goal: minimize meter ticks without sacrificing destination quality.

---

### Token Counting Before API Calls

Count tokens before the call to implement cost gates — prevent runaway costs on user inputs that are unexpectedly large.

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic()

async def call_with_cost_gate(prompt: str, max_input_tokens: int = 10000) -> str:
    # Count tokens WITHOUT making the generation call
    token_count = await client.messages.count_tokens(
        model="claude-sonnet-4-6",
        messages=[{"role": "user", "content": prompt}]
    )

    if token_count.input_tokens > max_input_tokens:
        raise PromptTooLargeError(
            f"Prompt has {token_count.input_tokens} tokens, limit is {max_input_tokens}"
        )

    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text
```

---

### Prompt Caching — ~90% Cost Reduction on Repeated Context

**The photocopied textbook analogy:**

If every student reads the same textbook chapter before asking questions, copying the chapter once and circulating it is cheaper than printing it fresh for each student. Prompt caching works the same way: mark the stable prefix of your prompt with `cache_control`. Anthropic caches it for 5 minutes. Subsequent requests with the same prefix skip the input tokens cost (~90% reduction on cached portions).

Requirements:
- Cache breakpoint must be at a position with >1024 tokens before it
- TTL: 5 minutes (1 hour for extended thinking)
- Cache key: exact byte-for-byte match of the prefix

```python
async def analyze_document_with_caching(document: str, question: str) -> str:
    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": f"You are a document analyst. Analyze this document:\n\n{document}",
                "cache_control": {"type": "ephemeral"}  # Cache this large prefix
            }
        ],
        messages=[{"role": "user", "content": question}]  # Only this changes between calls
    )

    # Check cache performance
    usage = response.usage
    cache_read = getattr(usage, 'cache_read_input_tokens', 0)
    cache_write = getattr(usage, 'cache_creation_input_tokens', 0)
    logger.info(
        "LLM call usage",
        extra={
            "input_tokens": usage.input_tokens,
            "output_tokens": usage.output_tokens,
            "cache_read_tokens": cache_read,
            "cache_write_tokens": cache_write,
        }
    )

    return response.content[0].text
```

---

### Streaming LLM Responses

**The restaurant order analogy:**

You order a 5-course meal. Option A: waiter brings all 5 courses at once after 45 minutes. Option B: each course arrives as it is ready. Streaming is option B — users see the first token appear within ~500ms even if the full response takes 10 seconds.

```python
from fastapi.responses import StreamingResponse
import anthropic

async def stream_llm_response(prompt: str) -> StreamingResponse:
    async def generate():
        async with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}]
        ) as stream:
            async for text in stream.text_stream:
                # Server-sent events format
                yield f"data: {orjson.dumps({'text': text}).decode()}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**Time to First Byte (TTFB):**
- Without streaming: TTFB = full generation time (~5-30 seconds for long responses)
- With streaming: TTFB = ~200-800ms (time to first token)

Users tolerate latency far better when they can see progress. Streaming is critical for any LLM response over ~500 tokens.

---

### Parallel LLM Calls with Semaphore

**The library book analogy:**

The library has 10 copies of a popular book. If 100 people request it simultaneously, 90 must wait. A semaphore limits how many can "check out" at once.

Without rate limiting, parallel LLM calls can hit API rate limits (tokens per minute, requests per minute) and trigger 429 errors.

```python
import asyncio

llm_semaphore = asyncio.Semaphore(5)  # Max 5 concurrent LLM calls

async def call_llm_rate_limited(prompt: str) -> str:
    async with llm_semaphore:
        response = await client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text

async def process_batch(prompts: list[str]) -> list[str]:
    # All calls run in parallel, but at most 5 at a time
    return await asyncio.gather(*[call_llm_rate_limited(p) for p in prompts])
```

---

### Embedding Caching

Embeddings are deterministic: the same input always produces the same output. Cache them by input hash to avoid redundant API calls.

```python
import hashlib

async def get_embedding_cached(text: str, redis: Redis) -> list[float]:
    cache_key = f"embed:{hashlib.sha256(text.encode()).hexdigest()}"

    cached = await cache_get(cache_key)
    if cached is not None:
        return cached

    # Call embedding API
    response = await openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    embedding = response.data[0].embedding

    # Cache indefinitely (embeddings are deterministic, use long TTL)
    await cache_set(cache_key, embedding, ttl_seconds=86400 * 7)  # 7 days

    return embedding
```

---

### Model Selection Strategy

| Model | Use Case | Latency | Cost |
|---|---|---|---|
| claude-haiku-4-5 | Classification, extraction, routing | ~200-500ms | Lowest |
| claude-sonnet-4-6 | General reasoning, content generation | ~1-5s | Medium |
| claude-opus-4-5 | Complex reasoning, long documents | ~5-30s | Highest |

```python
from enum import StrEnum

class LLMTier(StrEnum):
    FAST = "claude-haiku-4-5"         # <500ms, low cost
    BALANCED = "claude-sonnet-4-6"    # <5s, medium cost
    SMART = "claude-opus-4-5"         # <30s, high cost

def select_model(task_type: str, max_latency_ms: int) -> LLMTier:
    if task_type in ("classify", "extract", "route") or max_latency_ms < 1000:
        return LLMTier.FAST
    if task_type in ("generate", "summarize", "analyze"):
        return LLMTier.BALANCED
    return LLMTier.SMART  # Complex reasoning, long documents
```

---

## Part 6: Measuring Performance

### Concept 6: Why Averages Lie

**The hospital analogy:**

"The average wait time in the ER is 30 minutes." If 90% of patients wait 10 minutes but 10% wait 5 hours, the average is misleading. The 10% who wait 5 hours are the problem — and they are invisible in the average.

**P50 (median):** 50% of requests complete within this time. The "typical" experience.
**P95:** 95% of requests complete within this time. The "slow but not worst" experience.
**P99:** 99% of requests complete within this time. The experience of your most frustrated users.

In a service handling 1000 req/s, P99 = 1% of 1000 = 10 requests per second are experiencing P99 latency. At P99 = 2 seconds, 10 users per second are waiting 2 seconds. That is your user experience problem, not the P50.

```python
import statistics

def compute_percentiles(latencies_ms: list[float]) -> dict[str, float]:
    if not latencies_ms:
        return {}
    sorted_latencies = sorted(latencies_ms)
    n = len(sorted_latencies)
    return {
        "p50": sorted_latencies[int(n * 0.50)],
        "p95": sorted_latencies[int(n * 0.95)],
        "p99": sorted_latencies[int(n * 0.99)],
        "p999": sorted_latencies[int(n * 0.999)],
        "max": sorted_latencies[-1],
        "mean": statistics.mean(sorted_latencies),
    }
```

**Prometheus histogram in FastAPI:**

```python
from prometheus_client import Histogram
import time

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "endpoint", "status_code"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

@app.middleware("http")
async def track_request_latency(request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path,
        status_code=response.status_code,
    ).observe(duration)

    return response
```

---

### TTFB vs TTLB

**Time to First Byte (TTFB):** Time from client sends request to client receives first byte of response.
- Measures server processing time + network
- Critical for perceived performance — users see "something happening"
- High TTFB = slow DB queries, slow LLM calls, or network latency

**Time to Last Byte (TTLB):** Time from client sends request to client receives final byte.
- Measures total transfer time including response body
- Critical for bulk data operations, file downloads
- High TTLB with low TTFB = large response body or slow streaming

```python
# Measuring TTFB in FastAPI middleware
@app.middleware("http")
async def measure_ttfb(request, call_next):
    start = time.perf_counter()

    # This captures the generator, not the actual body
    response = await call_next(request)

    # TTFB = time to generate the response object (headers ready)
    ttfb = time.perf_counter() - start
    logger.info("TTFB", extra={"path": request.url.path, "ttfb_ms": ttfb * 1000})

    response.headers["X-TTFB-Ms"] = str(round(ttfb * 1000, 2))
    return response
```

---

### Throughput vs Latency Tradeoff

**The assembly line analogy:**

A factory can optimize for:
- **Throughput**: Maximum units per hour. Accept each unit immediately, process in batches. Higher throughput but each unit waits for the batch.
- **Latency**: Minimum time per unit. Process each unit immediately upon arrival. Lower latency but lower overall throughput.

In databases:
- **Batching writes** increases throughput (one transaction for 100 rows) but adds latency to each row.
- **Writing immediately** reduces latency but increases transaction overhead.

In caching:
- **Longer TTL** increases cache hit rate (higher throughput, less DB load) but serves staler data (higher effective latency to fresh data).
- **Shorter TTL** ensures fresh data but increases cache miss rate (more DB load).

---

### Load Testing with Locust

```python
# locustfile.py
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(0.1, 0.5)  # Wait 100-500ms between requests

    def on_start(self):
        response = self.client.post("/auth/token", json={
            "username": "test@example.com",
            "password": "password"
        })
        self.token = response.json()["access_token"]

    @task(3)  # Weight 3: called 3x more than weight-1 tasks
    def list_items(self):
        self.client.get(
            "/api/items",
            headers={"Authorization": f"Bearer {self.token}"}
        )

    @task(1)
    def create_item(self):
        self.client.post(
            "/api/items",
            json={"name": "test item", "description": "load test"},
            headers={"Authorization": f"Bearer {self.token}"}
        )
```

```bash
# Run: ramp up to 100 users over 30 seconds, target host
locust -f locustfile.py --users 100 --spawn-rate 10 --host http://localhost:8000
```

**What to watch during load testing:**
- Response time P95/P99 — starts degrading before P50
- Error rate — should stay < 0.1% under expected load
- Throughput (req/s) — find the point where adding users no longer increases throughput (saturation)
- CPU and memory on the server — find resource ceiling

---

## Summary: The Performance Optimization Workflow

```
1. MEASURE baseline (py-spy, pg_stat_statements, Prometheus P99)
   ↓
2. IDENTIFY top-1 bottleneck (never optimize second before first is fixed)
   ↓
3. HYPOTHESIS: is it CPU, I/O, DB, or serialization?
   ↓
4. VERIFY with EXPLAIN ANALYZE (for DB) or profiler flamegraph (for CPU)
   ↓
5. IMPLEMENT targeted fix:
   - N+1 → add selectinload or JOIN
   - Full table scan → add index (CONCURRENTLY)
   - Repeated expensive query → add caching (Redis or in-process)
   - Slow serialization → switch to orjson/ORJSONResponse
   - Blocking async → run_in_executor or convert to async
   ↓
6. MEASURE again — verify fix improved P99, not just P50
   ↓
7. REPEAT for next bottleneck
```

Never optimize without measuring. Never measure without a specific question. Never call a change "faster" without a before/after comparison at P99.
