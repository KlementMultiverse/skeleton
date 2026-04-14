---
description: Performance rules — profiling, query optimization, caching, LLM cost. Loads for all projects.
paths: ["**"]
---

# Performance Rules (L13)

> This file owns: profiling, database indexes, connection pools, caching patterns, Python performance, LLM cost management.
> Security rules → see `security.md`. DB query patterns → see `database-patterns.md`.

---

## Profiling

1. NEVER optimize code without profiling first — guessing the bottleneck is the most common performance mistake.

2. ALWAYS profile in production conditions (production data size, production concurrency) — benchmarks on toy datasets produce misleading results.

3. USE py-spy to profile live production processes — it attaches via ptrace, requires no code changes, and does not alter observed behavior. Run `py-spy record -o flamegraph.svg --pid <pid> --duration 60` to capture a 60-second flamegraph.

4. USE pyinstrument for development profiling — wrap suspect code in a `Profiler()` context manager; it produces a human-readable tree showing wall time per call chain.

5. USE cProfile only when you need exact call counts — it instruments every function call, which adds 5-20% overhead and can change timing characteristics. Do not run cProfile in production.

6. ALWAYS read flamegraphs from the top — the widest bars near the top of the stack are where time is actually spent. Wide bars at the bottom indicate overhead spread across many leaf calls.

7. ALWAYS sort cProfile output by `cumtime` first, then investigate functions where `ncalls` is anomalously high — high `ncalls` on a DB function is the signature of N+1 queries.

8. ENABLE asyncio debug mode (`PYTHONASYNCIODEBUG=1`) when debugging slow async code — it logs any coroutine that blocks the event loop for more than 100ms.

9. NEVER run CPU-bound code in an async function without `run_in_executor` — a blocking operation in an async context freezes the entire event loop, stalling every other concurrent request.

10. USE `asyncio.get_event_loop().run_in_executor(None, blocking_fn, *args)` to offload blocking calls from async contexts — `None` uses the default ThreadPoolExecutor.

---

## PostgreSQL Profiling

11. ALWAYS install and use `pg_stat_statements` — it is the only way to see query-level aggregate statistics (total time, calls, mean time, cache hit ratio) across all sessions.

12. ALWAYS query `pg_stat_statements` sorted by `total_exec_time DESC` to find the highest-impact queries — a query that takes 5ms but runs 10,000 times per minute is more impactful than one that takes 2 seconds but runs once per day.

13. ALWAYS check `shared_blks_read` in `pg_stat_statements` — high read values indicate queries hitting disk rather than PostgreSQL's shared_buffers RAM cache.

14. ALWAYS use `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` when a query is suspected slow — `ANALYZE` gives actual row counts and execution times; `BUFFERS` shows RAM vs disk reads.

15. NEVER run `EXPLAIN ANALYZE` on a destructive statement (`UPDATE`, `DELETE`, `INSERT`) in production without first wrapping it in a transaction that is immediately rolled back.

16. TREAT a gap between `rows=N` (estimated) and `actual rows=M` (real) larger than 10x as a sign of stale statistics — run `ANALYZE <tablename>` to refresh the planner's data.

17. RECOGNIZE `Seq Scan` on a large table in an EXPLAIN output as a mandatory investigation trigger — determine if an index is missing, not being used due to poor statistics, or if the planner correctly chose Seq Scan (small table or very low selectivity).

18. RECOGNIZE `Nested Loop` with `loops > 50` on an inner index scan as an N+1 pattern at the database level — the ORM issued one query to get N rows, then N individual queries for related data.

19. PREFER `Hash Join` for large table joins where the inner side is not indexed — Hash Join is O(N+M); Nested Loop without index is O(N*M).

20. RESET `pg_stat_statements` with `SELECT pg_stat_statements_reset()` after applying optimizations — compare metrics before and after to measure impact, not across sessions with mixed historical data.

---

## Index Design

21. NEVER add an index without first confirming the query plan shows a Seq Scan and the table has more than ~10,000 rows — small tables are faster to Seq Scan than to use an index.

22. ALWAYS create indexes with `CREATE INDEX CONCURRENTLY` in production — non-concurrent index creation takes an exclusive write lock that blocks all writes for the duration of the build.

23. NEVER create a `CREATE INDEX CONCURRENTLY` inside an explicit transaction — PostgreSQL rejects it. Run it as a standalone statement outside any `BEGIN` block.

24. ALWAYS check for `INVALID` indexes after a failed concurrent index build — `SELECT indexname FROM pg_indexes WHERE schemaname = 'public'` and verify none are invalid. Drop invalid indexes before retrying.

25. DEFAULT to B-tree index for all equality and range queries — it is the correct choice for `=`, `<`, `>`, `BETWEEN`, `LIKE 'prefix%'`, and `ORDER BY` on any sortable type.

26. USE GIN index for array columns, JSONB columns, and full-text search tsvector columns — GIN is an inverted index mapping element values to row IDs; B-tree cannot search inside arrays or JSONB.

27. USE GiST index for geometric types (PostGIS), range types (`daterange`, `tsrange`, `int4range`), and nearest-neighbor queries.

28. USE BRIN index only on append-only tables where rows are physically ordered by the indexed column (e.g., `created_at` on an events log table) — BRIN is 1000x smaller than B-tree but only works when data is physically ordered on disk.

29. NEVER use a Hash index as the default for equality columns — B-tree equality lookups are equally fast in practice, and B-tree additionally supports range queries and ordering.

30. ORDER composite index columns as: equality columns first, range columns second, sort columns last — `(tenant_id, status, created_at)` serves `WHERE tenant_id = $1 AND status = $2 ORDER BY created_at`.

31. ADD `INCLUDE (col1, col2)` to a B-tree index when the query's SELECT list is a strict superset of the index key columns — this enables Index Only Scan, which never touches the heap.

32. NEVER include frequently-updated columns in the INCLUDE list of a covering index — every write to those columns invalidates index entries.

33. CREATE partial indexes with `WHERE <condition>` when a query always filters on a fixed condition — `CREATE INDEX ... WHERE is_active = true` is smaller and faster than a full index when only 5% of rows are active.

34. CREATE expression indexes that exactly mirror the query's WHERE clause — `CREATE INDEX ON users (lower(email))` only benefits queries that use `WHERE lower(email) = lower($1)`.

35. NEVER rely on an expression index if the query uses a different expression — `lower(email)` and `LOWER(email)` are identical, but `trim(lower(email))` does not use a `lower(email)` index.

---

## Connection Pooling

36. ALWAYS configure connection pool size with the formula `(max_connections * 0.8) / num_instances` — reserve 20% of PostgreSQL's `max_connections` for admin tools and migrations.

37. NEVER set `pool_size` larger than the PostgreSQL instance can support — opening more connections than `max_connections` causes all new connection attempts to fail.

38. ALWAYS set `pool_pre_ping=True` (SQLAlchemy) or equivalent — this detects dropped connections (after network interruption or PostgreSQL restart) before handing them to the application.

39. ALWAYS set `pool_recycle` to less than PostgreSQL's `idle_in_transaction_session_timeout` and `tcp_keepalives_idle` — prevents stale connection reuse.

40. NEVER use a thread-based connection pool with an async application — use an async-native pool (`asyncpg`, `aiopg`, SQLAlchemy async engine).

41. ALWAYS monitor connection pool wait time in production — if `pool_timeout` exceptions appear, increase `pool_size` or reduce application concurrency.

42. PREFER asyncpg's pipelining for sequences of queries inside a transaction — pipeline multiple `execute()` calls to send them in one network round trip rather than request-response for each.

---

## VACUUM and Statistics

43. NEVER rely solely on autovacuum for high-write tables — tune `autovacuum_vacuum_scale_factor` per table using `ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.01)` to trigger VACUUM when 1% of rows are dead rather than the default 20%.

44. ALWAYS run `ANALYZE <tablename>` after a bulk import or bulk delete that significantly changes data distribution — autovacuum may not run immediately and stale statistics will cause wrong query plans.

45. TREAT a `n_dead_tup / n_live_tup` ratio above 20% (from `pg_stat_user_tables`) as a bloat alert requiring manual `VACUUM ANALYZE`.

46. NEVER run `VACUUM FULL` during business hours — it takes an exclusive lock and blocks all reads and writes for the duration.

47. ALWAYS use `REFRESH MATERIALIZED VIEW CONCURRENTLY` instead of `REFRESH MATERIALIZED VIEW` — the non-concurrent form locks the view for reads during refresh. Requires a unique index on the materialized view.

---

## Materialized Views and Partitioning

48. USE materialized views for queries that aggregate more than ~100,000 rows and are read more often than they change — pre-computed results serve in milliseconds instead of seconds.

49. ALWAYS create at least one index on a materialized view — the view is a physical table; without an index, every query against it is a Seq Scan.

50. USE range partitioning on `created_at` for append-only time-series tables exceeding ~50 million rows — PostgreSQL's partition pruning eliminates irrelevant partitions at plan time.

51. ALWAYS create indexes on each partition individually after creating the partition — indexes created on the parent before partitioning do not automatically propagate to new child partitions in all PostgreSQL versions.

52. VERIFY partition pruning is working with `EXPLAIN` — the output should show `Partitions selected: N of M` where N is much smaller than M.

53. NEVER use list partitioning for high-cardinality columns (user_id, UUID) — use hash partitioning when you need even distribution across a fixed number of partitions.

---

## Caching — General

54. ALWAYS measure cache hit rate in production — a cache hit rate below 80% means the cache is adding latency (network round trip to Redis) without sufficient benefit.

55. NEVER cache without a TTL — unbounded cache growth exhausts memory. Every cache entry must have an expiry.

56. NEVER use in-process cache (TTLCache) for per-user or per-tenant data — each app instance has its own cache; users hitting different instances see different data, causing consistency bugs.

57. USE in-process TTLCache only for global reference data that is identical across all users and all instances — config values, feature flags, translation strings.

58. ALWAYS make TTLCache instances thread-safe by passing a `threading.Lock()` instance to the `lock` parameter of `@cached`.

59. ALWAYS namespace Redis cache keys with the entity type and ID — `user:42:profile` not `42` or `profile_42`. Prevents accidental collision between different data types.

60. ALWAYS namespace Redis cache keys with `tenant_id` in multi-tenant applications — `tenant:{tid}:user:{uid}:profile`. Without namespace, one tenant's cache can return another tenant's data.

61. NEVER derive a Redis cache key from user-supplied input alone — always combine with server-side identifiers to prevent cache poisoning.

---

## Caching — Patterns

62. PREFER cache-aside (lazy loading) as the default pattern — it is the simplest, only caches data that is actually read, and cache misses always return correct data from the source of truth.

63. USE write-through caching when reads far outnumber writes and stale data is unacceptable — every write pays the cost of updating both DB and cache.

64. AVOID write-behind (write-back) caching for user data — if the cache fails before the async DB write completes, that data is permanently lost.

65. ALWAYS invalidate related cache keys on write — if updating `user:42` also affects `team:7:members`, invalidate both keys in the same write path.

66. ALWAYS use Redis pipelines when invalidating multiple keys in one operation — a pipeline sends all commands in one network round trip.

67. PREFER event-driven invalidation over pure TTL for data that is written from multiple places — TTL-only caching causes inconsistency windows; explicit deletion on write keeps cache consistent.

---

## Cache Stampede Prevention

68. ALWAYS implement stampede protection for any cache entry that is expensive to rebuild and has high concurrent read traffic.

69. USE probabilistic early expiry (XFetch algorithm) as the default stampede protection — it prevents the thundering herd without coordination overhead of distributed locks.

70. USE distributed Redis locks (SET NX EX) when you must guarantee exactly one rebuilder — implement with a short lock TTL (10 seconds) to prevent deadlock if the rebuilder crashes.

71. ALWAYS set a short TTL on distributed rebuild locks — if the process holding the lock crashes, the lock must expire so another process can take over.

72. NEVER implement a rebuild lock retry as a tight polling loop — use `asyncio.sleep(0.1)` between retries to yield the event loop.

73. USE background refresh for caches where slightly stale data is acceptable and minimizing P99 latency is critical — serve the cached value immediately, schedule refresh as a background task when TTL is below a threshold.

---

## Serialization

74. ALWAYS use `ORJSONResponse` as the default response class for FastAPI applications — it is 10-50x faster than the stdlib `JSONResponse` and handles Python types (datetime, UUID, bytes, dataclass) natively.

75. SET `default_response_class=ORJSONResponse` at the `FastAPI()` constructor level — do not override per-route unless a specific route requires standard JSON behavior.

76. USE `orjson.dumps()` / `orjson.loads()` for all Redis serialization — pass the result bytes directly to Redis without decoding. Set `decode_responses=False` on the Redis client.

77. NEVER use `json.dumps()` for caching or response serialization in hot paths — it is 10x slower than orjson and requires custom encoders for datetime and UUID.

78. NEVER use `.format()` or string concatenation to build serialized data — use orjson or f-strings.

---

## Python Async Performance

79. ALWAYS install uvloop in production deployments — it replaces the default asyncio event loop with a libuv-based implementation that delivers 2-4x throughput improvement for I/O-bound workloads.

80. CALL `uvloop.install()` before any `asyncio.run()` call or server startup — or set `--loop uvloop` in the Uvicorn configuration.

81. NEVER mix synchronous blocking calls into async functions — file reads with `open()`, `time.sleep()`, synchronous ORM calls, and CPU-bound computations all block the event loop.

82. USE `asyncio.TaskGroup` (Python 3.11+) instead of `asyncio.gather` for parallel coroutines — TaskGroup cancels all remaining tasks when any task raises, preventing zombie coroutines.

83. USE `asyncio.gather` when you need `return_exceptions=True` behavior (collecting results and errors together) — TaskGroup always raises on first exception.

84. NEVER create `asyncio.Task` objects without attaching them to a TaskGroup or collecting them with `asyncio.gather` — detached tasks that raise exceptions are silently dropped.

85. USE `asyncio.Semaphore` to cap concurrency of external API calls — without a semaphore, `asyncio.gather` on a large list fires all coroutines simultaneously and may hit rate limits or exhaust connection pools.

---

## Batch Operations

86. ALWAYS use bulk INSERT (`insert().values(list)`) instead of individual `db.add()` in a loop when inserting more than 10 rows — individual inserts make one network round trip per row; bulk insert makes one.

87. USE the PostgreSQL COPY protocol (asyncpg `copy_records_to_table`) for importing more than 10,000 rows — it is 10-100x faster than bulk INSERT for large datasets.

88. ALWAYS batch Redis operations with pipelines when performing more than 3 operations in sequence — `pipeline.setex(...); pipeline.delete(...); await pipeline.execute()` sends all commands in one round trip.

89. NEVER loop over a list and call `await redis.set(key, value)` individually for each item — use a pipeline.

---

## Streaming

90. USE `StreamingResponse` for any response payload exceeding 10MB — buffering large responses in memory causes high memory pressure and increases TTFB.

91. ALWAYS use `StreamingResponse` for CSV and JSON exports that iterate database rows — yield rows as they come from the database cursor instead of fetching all rows into memory.

92. NEVER use `StreamingResponse` for normal API responses under 1MB — it adds complexity and omits `Content-Length`, which clients and proxies use for progress tracking.

---

## LLM Performance

93. ALWAYS count tokens with `client.messages.count_tokens()` before making LLM API calls that accept user-supplied content — implement a hard limit to reject inputs that would exceed your cost or context budget.

94. NEVER pass unbounded user content (form fields, file uploads, document pastes) directly to an LLM API call without a token count gate — users can craft inputs that consume hundreds of thousands of tokens.

95. ALWAYS add `cache_control: {"type": "ephemeral"}` to prompt prefixes exceeding 1024 tokens that are reused across requests — prompt caching reduces input token cost by ~90% on cache hits, with a 5-minute TTL.

96. NEVER place the `cache_control` breakpoint inside the dynamic portion of a prompt — the cache key is the exact byte sequence before the breakpoint; any change invalidates the cache.

97. ALWAYS stream LLM responses for output exceeding 500 tokens — without streaming, the client waits for the full response; with streaming, the first token appears in ~200-800ms, dramatically improving perceived performance.

98. USE `asyncio.Semaphore` to cap concurrent LLM API calls — set the limit based on your API tier's requests-per-minute limit: `Semaphore(RPM_LIMIT // 60 * MAX_LATENCY_SECONDS)`.

99. ALWAYS cache embedding API responses by input hash — embeddings are deterministic; the same input always produces the same output. Cache with a long TTL (days, not minutes).

100. SELECT the smallest model capable of the task — use claude-haiku-4-5 for classification, extraction, and routing tasks; claude-sonnet-4-6 for general generation; reserve claude-opus-4-5 for complex reasoning only.

101. NEVER use a large model for tasks that are reliably handled by a smaller model — model selection has a 5-20x impact on cost and a 3-10x impact on latency.

102. LOG `cache_read_input_tokens` and `cache_creation_input_tokens` from every LLM API response — these fields tell you whether prompt caching is working; a `cache_read` of 0 on repeated calls means the cache is not being hit.

---

## Measurement

103. ALWAYS measure P95 and P99 latency — mean latency hides the experience of the slowest 1-5% of requests, which are often the users most likely to churn.

104. NEVER report average (mean) latency as the primary performance metric — averages are distorted by outliers and mask tail latency problems.

105. ALWAYS establish a latency baseline before any optimization — without a before measurement, you cannot claim the after measurement is an improvement.

106. ALWAYS measure TTFB (Time to First Byte) separately from TTLB (Time to Last Byte) — high TTFB indicates slow server processing; high TTLB minus low TTFB indicates large response body or slow streaming.

107. USE Prometheus histograms (not gauges or counters) for latency — histograms support percentile calculation in Grafana/PromQL. Use buckets covering 5ms to 10s.

108. ALWAYS run load tests before launching a new endpoint in production — identify the saturation point (requests per second at which P99 starts degrading) before real traffic hits it.

109. ALWAYS load test with realistic data distribution — a load test with a single cached user_id is not representative of real traffic.

110. NEVER claim a query optimization is an improvement without running EXPLAIN on the exact production query with production data volume — optimizer behavior changes with data size.

---

## PostgreSQL Profiling — Setup and Advanced Nodes

111. ALWAYS enable `pg_stat_statements` by adding it to `shared_preload_libraries` in `postgresql.conf` and running `CREATE EXTENSION pg_stat_statements` after restart — set `pg_stat_statements.max = 10000` and `pg_stat_statements.track = all`. Without these settings the extension collects no data.

112. ALWAYS set `log_min_duration_statement = 1000` in `postgresql.conf` as a baseline slow query threshold — lower to 100ms in staging; never set to 0 in production (logs every query).

113. USE `nplusone` or a SQLAlchemy `after_cursor_execute` event listener during development to detect N+1 queries before production — assert `len(captured_queries) <= EXPECTED_MAX` per test.

114. RECOGNIZE `Bitmap Index Scan` + `Bitmap Heap Scan` as the planner combining multiple index conditions via an in-memory bitmap — investigate only when `Rows Removed by Recheck` is high (bitmap was lossy due to low `work_mem`); increase `work_mem` for that session to fix it.

115. RECOGNIZE `Gather`/`Gather Merge` nodes as parallel query execution — if `Workers Launched < Workers Planned`, increase `max_parallel_workers` and `max_parallel_workers_per_gather` in `postgresql.conf`.

116. ALWAYS route read-only queries to a hot standby replica via a separate SQLAlchemy read engine — monitor replication lag with `SELECT extract(epoch FROM (now() - pg_last_xact_replay_timestamp()))` and add a circuit-breaker fallback to the primary if lag exceeds your staleness SLO.

---

## Caching — Redis Advanced Patterns

117. USE Redis `MULTI`/`EXEC` when the correctness of a command batch requires atomicity — pipelines reduce round trips but are not atomic; use `WATCH key` before `MULTI` for optimistic read-modify-write cycles (retry on `nil` return from `EXEC`).

118. ALWAYS use hash tags `{...}` in Redis Cluster key names when a Lua script or `MULTI`/`EXEC` block must operate on multiple keys — `{user:42}:profile` and `{user:42}:sessions` hash to the same slot; keys without a shared hash tag may live on different nodes and raise `CROSSSLOT`.

119. CHOOSE Redis over Memcached whenever you need persistence, pub/sub, sorted sets, streams, lists, Lua scripting, or atomic multi-key operations — choose Memcached only for pure key-value caching at extreme write throughput when none of Redis's data structures are needed.

120. ALWAYS version cache keys when the shape of cached data changes — use a version prefix (`v2:user:42:profile`); old versioned keys expire under their TTL without a cache flush. Never flush an entire cache namespace on deploy — it creates a simultaneous miss storm.

121. ALWAYS add TTL jitter (10–25% random offset) to cache entries populated in bulk — if 10,000 keys expire simultaneously they cause a synchronized miss storm; use `ttl = base_ttl + random.randint(0, base_ttl // 4)`.

122. CHECK `OBJECT ENCODING <key>` in Redis when memory usage is unexpectedly high — a hash exceeding `hash-max-listpack-entries` (default 128) promotes to a full hash table, multiplying memory by 5–10x; tune thresholds in `redis.conf` after benchmarking.

---

## Query Patterns — Pagination and Prepared Statements

123. USE prepared statements for hot query paths in asyncpg — `await conn.prepare(sql)` compiles the plan once; subsequent executions skip parse and plan phases. asyncpg's built-in `statement_cache_size=100` auto-prepares on first execution.

124. ALWAYS use cursor-based (keyset) pagination instead of `LIMIT N OFFSET M` for tables larger than ~100,000 rows — `OFFSET M` forces reading and discarding M rows; keyset `WHERE (created_at, id) < ($ts, $id)` executes in O(log N) via index regardless of page depth.

125. NEVER use `SELECT COUNT(*) FROM large_table` in a user-facing request path — use `pg_stat_user_tables.n_live_tup` or `SELECT reltuples FROM pg_class` for UI display counts; maintain an explicit counter for business-critical exact counts.

126. ALWAYS create a covering index with `INCLUDE` for frequently paginated queries — `CREATE INDEX ON posts (author_id, created_at DESC) INCLUDE (id, title)` enables Index Only Scan; the query never touches the heap.

127. TREAT any query using `OFFSET` beyond 1,000 rows as a mandatory migration trigger to keyset pagination — OFFSET cost grows linearly; this is fundamental, not a configuration problem.

128. USE `SELECT ... FOR UPDATE SKIP LOCKED` for job queue patterns in PostgreSQL — atomically locks and returns only rows not held by another worker; eliminates the need for a separate queue broker at moderate throughput (<1,000 jobs/second).

---

## LLM Performance — Advanced

129. TUNE async worker concurrency by workload type — I/O-bound async tasks: `--concurrency = 4–8x CPU_COUNT` with async pool; CPU-bound tasks: `--concurrency = CPU_COUNT` with prefork. For Celery set `worker_prefetch_multiplier = 1` for long tasks to prevent queue hoarding.

130. IMPLEMENT LLM streaming in FastAPI using `StreamingResponse` with `media_type="text/event-stream"` and set `X-Accel-Buffering: no` to prevent nginx from buffering the stream — yield each delta chunk immediately without accumulating the full response.

131. IMPLEMENT a sliding window token budget for multi-turn LLM conversations — evict oldest turns when history exceeds 80% of context window, preserving the system prompt and the most recent user+assistant pair; log every eviction event.

132. ALWAYS batch embedding requests to the provider's maximum batch size — OpenAI up to 2,048 inputs, Voyage up to 128; batching eliminates per-request HTTP overhead; use `asyncio.gather` with a `Semaphore` to parallelize batches.

133. ALWAYS implement exponential backoff with jitter on LLM API 429 responses — base 1s, double each retry, add `random.uniform(0, delay * 0.5)` jitter, cap at 60s, stop after 5 attempts. Never retry immediately after a 429.

134. USE structured output (JSON mode or tool-use with explicit schema) for LLM tasks that require parsing — eliminates prose-wrapping tokens, makes output token count predictable, and uses 30–50% fewer output tokens than free-text with "respond in JSON" instructions.

---

## HTTP and Network Performance

135. ALWAYS reuse `httpx.AsyncClient` instances by instantiating them at application startup and injecting via dependency — creating a new client per request opens a new TCP+TLS connection each time, adding 50–200ms per external API call.

136. ALWAYS add `GZipMiddleware` with `minimum_size=1000` — responses under 1KB cost more CPU to compress than bandwidth saved; responses over 1KB typically compress to 20–30% of original size. Do not compress already-compressed formats (JPEG, PNG, binary).

137. SERVE static files via nginx using `sendfile(2)` in all production deployments where nginx is available — use WhiteNoise only on PaaS platforms without reverse proxy configuration. Set `Cache-Control: public, max-age=31536000, immutable` on versioned static assets.

138. ALWAYS project response fields to the minimum set needed by each endpoint — define separate `UserSummary` (list) and `UserDetail` (single-item) schemas; field projection reduces response size by 60–80% on list endpoints.

139. ALWAYS use `selectinload` for one-to-many and `joinedload` for many-to-one (FK) in SQLAlchemy — `joinedload` on one-to-many produces a JOIN that multiplies result rows by child count (cartesian product); `selectinload` issues one extra `WHERE id IN (...)` query with no row inflation.

140. NEVER rely on SQLAlchemy default lazy loading on relationships accessed in API paths — set `lazy="raise_on_sql"` on all relationship definitions to fail loudly during development, then use explicit `options(selectinload(...))` or `options(joinedload(...))` at query time.

---

## Python Memory and Process Tuning

141. USE `tracemalloc.start(25)` + `take_snapshot()` + `statistics("lineno")` to identify memory allocation hotspots by file and line number — compare two snapshots with `snapshot2.compare_to(snapshot1, "lineno")` to find what grew between requests.

142. USE `objgraph.show_growth(limit=10)` after processing N requests to detect reference leaks — use `show_backrefs(obj, max_depth=5)` to trace what holds the reference alive. Run in a local load test harness only; objgraph traverses the entire heap and is very slow.

143. ALWAYS set `worker_class = "uvicorn.workers.UvicornWorker"` when running FastAPI under Gunicorn — the default `sync` worker cannot execute `async def` route handlers correctly; `UvicornWorker` embeds a full uvicorn event loop per Gunicorn worker process.

144. CALCULATE Gunicorn workers as `(2 * CPU_COUNT) + 1` for I/O-bound async applications — verify that `workers * pool_size_per_worker <= db_max_connections * 0.8`.

145. INTERPRET load test results using Little's Law (`L = λ × W`) to identify the saturation point — throughput grows linearly until the server saturates; beyond saturation throughput flattens while P99 grows rapidly. Report capacity as the RPS at which P99 first exceeds your SLO, not peak RPS.

146. DISABLE Python's cyclic GC in Gunicorn workers via `gc.disable()` in a `post_fork` hook ONLY when the application creates no reference cycles — GC pauses add P99 spikes of 5–50ms; re-enable immediately if memory grows without bound after disabling.
