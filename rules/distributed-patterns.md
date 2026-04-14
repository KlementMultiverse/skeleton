---
description: Distributed systems and Matrix/Synapse integration rules.
paths: ["*.py", "app/**/*.py", "matrix/**/*.py"]
---

# Distributed Systems and Matrix/Synapse Rules

> Owns: CAP/PACELC decisions, distributed transactions, Saga pattern, idempotency,
>        Matrix protocol, Application Service integration, event-driven architecture,
>        eventual consistency, CRDT, AI+Matrix patterns.
> Performance rules → performance-patterns.md
> Security rules → security-complete.md

---

## Section 1: CAP Theorem Applied

1. ALWAYS explicitly choose CP or AP for each data store in your system — document the choice and the reasoning; "we use Postgres" is not an answer because Postgres with synchronous replication is CP and with async replication is effectively AP during partition.

2. NEVER assume partition tolerance is optional — in any system with more than one process connected over a network, partitions happen; design for them rather than hoping they don't.

3. ALWAYS design user-facing reads on AP datastores to display a staleness indicator when the returned data may be stale — users can tolerate slightly old data if they know it; users cannot tolerate silent lies.

4. NEVER make a CP system the sole dependency in the write path of a high-availability endpoint — if the CP store is unavailable (and correctly refusing to serve), the entire write path fails; buffer writes or degrade gracefully.

5. ALWAYS pick consistency level per query in Cassandra (or equivalent AP stores) — use `QUORUM` for writes and reads of financial data; use `ONE` for analytics or presence data; never set a global default and forget it.

6. ALWAYS document the consistency level used for every Redis operation that affects user-visible state — Redis Cluster during partition can silently return stale data or route to a wrong slot; this must be in comments, not assumed.

7. NEVER use `origin_server_ts` from Matrix events as the authoritative ordering field — servers set their own clocks; a malicious or misconfigured server can lie; use `depth` and DAG position for causal ordering.

8. ALWAYS implement a conflict resolution strategy before deploying any AP system that accepts writes from multiple nodes — "we'll figure it out later" is how you lose data in production.

9. ALWAYS apply the PACELC lens to infrastructure decisions, not just the CAP lens — even without a partition, synchronous replication adds latency (EC choice); cache-aside adds latency reduction at the cost of potential staleness (EL choice); make this tradeoff explicit.

---

## Section 2: Distributed Transactions

10. NEVER use two-phase commit (2PC) across services owned by different teams or across datacenters — 2PC is a blocking protocol; a coordinator crash between Phase 1 and Phase 2 leaves all participants locked indefinitely; use Saga instead.

11. ALWAYS implement a Saga with compensating transactions for every distributed multi-step workflow — every step that has a side effect (charge card, send email, deduct inventory) must have a documented and tested compensation (refund, send cancellation, release inventory).

12. NEVER write compensating transactions as afterthoughts — design the compensation when you design the forward step; a compensation that is untested is no better than no compensation.

13. ALWAYS use idempotency keys on every step that calls an external service — the key must be scoped to: `{resource_type}:{resource_id}:{operation}:{idempotency_uuid}`; reusing the same key for a different operation on the same resource will cause incorrect deduplication.

14. ALWAYS store idempotency key results in the same database transaction as the business data write — if you store the key in Redis and the business data in PostgreSQL, a crash between the two writes produces an inconsistent state.

15. NEVER retry a Saga step more than 3 times without exponential backoff (base 1s, max 60s with jitter) — tight retry loops during a downstream outage amplify the load and extend the outage; implement circuit breakers at the Saga orchestrator level.

16. ALWAYS use orchestration-based Sagas when the workflow has more than 3 steps or contains conditional branching — choreography becomes unreadable beyond 3 services; the event chain is impossible to trace without a central state log.

17. ALWAYS persist the Saga orchestrator's state to durable storage (PostgreSQL) after each step — if the orchestrator restarts, it must resume from the last committed step, not from the beginning.

18. NEVER let a Saga compensate a step that never executed — track which steps actually completed before attempting compensation; compensating a step that didn't run causes phantom operations (e.g., refunding a payment that was never charged).

---

## Section 3: Matrix Protocol

19. ALWAYS verify the Ed25519 signature on incoming federation PDUs before applying them to the DAG — an unverified event can contain arbitrary state_key values; accepting it without signature verification allows any server to impersonate any user.

20. NEVER trust `origin_server_ts` for business logic ordering — use the event's `depth` field (which reflects DAG position) for causal ordering; `origin_server_ts` is informational and not cryptographically protected.

21. ALWAYS reference the exact `auth_events` that authorize a new state event when constructing PDUs — `auth_events` must include the most recent `m.room.create`, the sender's `m.room.member` with `membership: join`, and the most recent `m.room.power_levels`; missing auth events will cause the event to be rejected by other homeservers during state resolution.

22. NEVER mutate the `signatures` or `hashes` fields on a PDU — these are computed over the event content; any modification invalidates the signature and causes remote homeservers to reject the event.

23. ALWAYS use room version 11 (or the latest stable version) when creating new rooms — older room versions have known security issues (e.g., v1 event IDs are human-readable and guessable; v4+ event IDs are content hashes).

24. ALWAYS treat state resolution v2 as a black box when implementing federation — do not attempt to short-circuit it or apply heuristics; all homeservers must run the identical algorithm to converge on the same state.

25. NEVER attempt to resolve state forks by timestamp alone — two simultaneous power level changes from servers with synchronized clocks would be indistinguishable; use the full state resolution v2 algorithm with its topological sort and lexicographic tie-breaking.

26. ALWAYS handle the `limited` flag in the sync timeline response — `limited: true` means there are events you missed; fetch the gap using `prev_batch` before processing the returned events, otherwise your client will have holes in its event history.

27. ALWAYS use the `user_id` query parameter (not a separate access token) when acting as virtual AS users via the Client-Server API — the AS presents its `as_token` in the Authorization header and specifies the target user in the query parameter; creating individual access tokens for each virtual user is unnecessary and unscalable.

28. NEVER access the `unsigned` field of an event for security decisions — it is not covered by the event signature and can be modified in transit by any intermediate server.

---

## Section 4: Application Service Integration

29. ALWAYS return HTTP 200 from `PUT /_matrix/app/v1/transactions/{txnId}` within 5 seconds — Synapse has a configurable timeout (default varies); if the AS exceeds it, Synapse marks the transaction as failed and retries; returning 200 after a timeout causes the AS to process the transaction twice while Synapse believes it failed.

30. NEVER process events synchronously inside the transaction endpoint handler — the handler must only: (1) verify the hs_token, (2) deduplicate by txn_id, (3) enqueue events, (4) return 200; all LLM calls, database writes, and external API calls happen in a background worker.

31. ALWAYS deduplicate transaction IDs in a persistent store (PostgreSQL, not in-memory) — an in-memory dedup set is lost on process restart; after restart, Synapse retries pending transactions and the AS must still recognize them as duplicates.

32. ALWAYS use `INSERT INTO seen_transactions (txn_id) ON CONFLICT DO NOTHING RETURNING txn_id` for atomic deduplication — a SELECT then INSERT has a TOCTOU race condition when multiple AS instances run concurrently; the atomic insert eliminates it.

33. NEVER use a single shared asyncio.Queue between the transaction endpoint and the worker if the AS runs multiple processes — use Redis Streams or a database-backed queue for multi-process deployments so events are not silently dropped when a process restarts.

34. ALWAYS verify the `Authorization: Bearer {hs_token}` header on every request from Synapse — AS endpoints are publicly reachable HTTP endpoints; any system can POST to them; the hs_token is the only authentication; missing verification allows unauthorized event injection.

35. NEVER re-use the same txn_id for different outgoing API calls from the AS — when the AS sends a reply via the Client-Server API, it must provide a client-side txn_id; make this deterministic based on the input event_id (`sha256(f"reply:{event_id}")[:16]`) so that retries produce the same txn_id and Synapse deduplicates the reply.

36. ALWAYS return 404 from `GET /_matrix/app/v1/users/{userId}` for users outside the AS namespace — returning 200 tells Synapse the AS can provision that user, which creates phantom users the AS cannot actually control.

37. ALWAYS filter out events sent by the AS's own virtual users before processing — failing to do so creates echo loops: the bot responds to its own message, which triggers another response, until rate limits kick in.

38. ALWAYS set `rate_limited: false` in the AS registration YAML only for the AS user namespace, not globally — rate limiting prevents accidental loops and abuse; disable it surgically per namespace.

39. NEVER expose the `as_token` in logs, error messages, or API responses — it grants administrative access to your homeserver namespace; rotate it immediately if exposed.

40. ALWAYS use the `m.notice` msgtype for automated bot messages that are status updates, not conversational replies — clients display `m.notice` with reduced visual prominence; using `m.text` for system notifications creates visual noise indistinguishable from human messages.

41. ALWAYS implement a dead letter queue for events that fail processing after 3 retries — silently dropping events makes debugging impossible; store failed events with error context in a `failed_events` table for manual inspection.

42. NEVER assume the `events` array in a transaction contains exactly one event — Synapse batches multiple events into one transaction; handle all of them, and process them in the order they appear.

---

## Section 5: Event-Driven Architecture

43. ALWAYS use the Outbox pattern when publishing events to a message broker from an application that also writes to a database — a direct publish-then-write or write-then-publish creates a window where the application can crash and produce either an event without data or data without an event.

44. ALWAYS write the outbox event in the same database transaction as the business data — if the transaction rolls back, the outbox event rolls back with it; this is the entire value of the pattern.

45. NEVER use the same consumer group ID for two logically independent services reading from the same Kafka topic — Kafka assigns each partition to exactly one consumer per group; two services sharing a group will steal each other's messages, resulting in incomplete processing.

46. ALWAYS use Redis Streams (not Redis Pub/Sub) when message durability is required — Pub/Sub messages are lost if no consumer is connected; Streams persist messages until explicitly acknowledged or trimmed; use Pub/Sub only for ephemeral signals (cache invalidation, presence updates).

47. ALWAYS call `XACK` on a Redis Stream message only after successful processing, not before — early acknowledgment means a crash between ack and processing permanently loses the message; deferred ack means the message is reprocessed on retry (implement idempotency in the handler).

48. ALWAYS configure a dead letter exchange (DLX) on RabbitMQ queues for production workloads — without DLX, messages that are nacked or expire are silently discarded; DLX routes them to a separate queue for inspection.

49. NEVER set `requeue=True` on a failed RabbitMQ message without a maximum retry count — unconditional requeue creates an infinite retry loop on a permanently broken message (poison pill), consuming 100% of worker capacity.

50. ALWAYS use `SKIP LOCKED` in the outbox publisher query — `SELECT ... FOR UPDATE SKIP LOCKED` allows multiple publisher instances to run concurrently without competing for the same rows; without it, only one publisher can run at a time (row-level lock contention).

51. ALWAYS partition Kafka topics by the natural aggregation key of your domain (customer_id, order_id, user_id) — this ensures all events for a given entity land in the same partition, preserving per-entity ordering without global ordering (which is unscalable).

52. NEVER rely on Kafka Pub/Sub at-least-once delivery for financial operations without idempotency on the consumer — at-least-once means duplicates happen (network retries, rebalances); double-charging or double-crediting is a direct financial loss.

53. ALWAYS set a maximum length (`MAXLEN ~`) on Redis Streams used as job queues — without a cap, a slow consumer causes unbounded memory growth in Redis; approximate trimming (`~`) is 10x faster than exact trimming.

54. ALWAYS implement `pg_notify` event listeners on a DEDICATED connection, never from the connection pool — `LISTEN` state is per-connection; a pooled connection used for queries may lose the listener state when the connection is recycled; always create a single persistent connection for notifications.

---

## Section 6: Eventual Consistency Patterns

55. ALWAYS return the idempotency key result from cache before checking if an operation already completed — the cache check is the fast path; the DB check is the fallback; this two-tier pattern keeps idempotency overhead sub-millisecond.

56. NEVER use a PN-Counter (CRDT) to enforce a hard constraint (e.g., "never go below zero") — CRDTs guarantee convergence, not safety invariants; a CRDT counter can temporarily show a negative value across replicas if concurrent decrements occur; use optimistic locking with a database constraint for hard lower bounds.

57. ALWAYS implement merge functions for CRDTs with all three properties — **commutative** (merge(A, B) = merge(B, A)), **associative** (merge(merge(A, B), C) = merge(A, merge(B, C))), **idempotent** (merge(A, A) = A) — a merge function that violates any property will not converge.

58. NEVER use LWW (Last Write Wins) for data where concurrent writes should be preserved — LWW silently discards the losing write with no notification to the writer; use a CRDT that preserves all concurrent values (e.g., MV-Register) and present the conflict to the user.

59. ALWAYS use vector clocks (or Hybrid Logical Clocks) to detect concurrent writes rather than wall clock timestamps — wall clocks across different servers can differ by seconds; vector clocks track causality exactly without clock synchronization.

60. ALWAYS implement read repair at the application layer for any AP datastore that does not do it automatically — a stale replica that is never repaired accumulates drift; application-layer read repair corrects this lazily when the stale data is read.

61. ALWAYS document the convergence time for your eventual consistency model — "eventually consistent" with a 5-second convergence time is a very different SLA than "eventually consistent" with a 5-minute convergence time; users need to know what to expect.

62. NEVER expose eventual consistency behavior to end users without a visual indicator — show "last updated 3 seconds ago" or "syncing..." rather than silently serving stale data that the user might act on.

63. ALWAYS model compensating reads (anti-entropy) as background tasks with bounded jitter — running all anti-entropy scans simultaneously (e.g., at midnight) creates a stampede; add random offsets per partition.

---

## Section 7: AI + Matrix Patterns

64. ALWAYS key LangGraph checkpoints by `room_id`, not by user_id — one room = one conversation thread = one LangGraph thread; using user_id collapses all rooms for a user into one context, conflating separate conversations.

65. ALWAYS key user memory (Mem0, Qdrant, or equivalent) by the sender's full MXID (`@user:server`), not just the localpart — MXIDs are globally unique across federated servers; localparts are only unique per server; `alice` on `matrix.org` is not the same person as `alice` on `example.com`.

66. ALWAYS post interim agent activity as `m.notice` events while a long-running tool call is executing — silence during a 30-second tool call makes users think the bot crashed; `m.notice "Searching the web..."` sets correct expectations.

67. NEVER invoke an AI agent synchronously inside the Matrix AS transaction endpoint handler — AI completions take 1-30 seconds; the transaction endpoint must return within 5 seconds; always queue AI work and process asynchronously.

68. ALWAYS deduplicate AI responses using the input event_id as the idempotency key — if the AS crashes after completing the LLM call but before posting the reply, Synapse retries the transaction, the LLM runs again, and a duplicate reply is posted; check for an existing reply event before calling the LLM.

69. ALWAYS filter incoming events for the bot's own MXID before routing to AI — the bot's replies are delivered back to the AS (because the AS receives all events in its namespaced rooms); if the bot processes its own messages, it will reply to itself, creating an infinite loop that runs until rate limits engage.

70. ALWAYS store tool call results as `m.notice` events in the room alongside the final response — the DAG then contains a complete audit trail of what the AI searched, computed, or retrieved; this is essential for debugging and explainability.

71. ALWAYS handle `m.room.member` events to detect when the bot is invited to a new room — auto-join on invitation by calling `POST /_matrix/client/v3/join/{roomId}` from the bot's MXID; a bot that does not auto-join never receives messages from that room.

72. NEVER store the entire Matrix sync response in the AI context window — the raw sync response contains hundreds of fields, member lists, and metadata; extract only the relevant `m.room.message` events and pass those to the LLM.

73. ALWAYS scope rate limiting for AI invocations per MXID + room combination — a single user flooding one room should not exhaust the AI budget for all other rooms and users; use a Redis counter keyed by `{user_mxid}:{room_id}`.

---

## Section 8: Anti-Patterns (NEVER do these)

74. NEVER block the AS transaction endpoint handler on any I/O operation — no database queries, no HTTP calls, no LLM invocations; the only acceptable operations are: hs_token check, dedup check against an in-memory set (with DB fallback), queue.put_nowait(), return {}; anything else risks a Synapse timeout and transaction retry storm.

75. NEVER trust federation events without verifying the Ed25519 signature against the originating server's published public key — an attacker who compromises DNS can send unsigned or incorrectly-signed events; verification requires fetching the key from `/_matrix/key/v2/server` on the origin and comparing.

76. NEVER share a Kafka consumer group ID across multiple deployments of the same service in different environments (staging, production) — if both environments consume from the same Kafka cluster, they will steal each other's messages; namespace consumer group IDs: `{service_name}:{environment}`.

77. NEVER use global state (module-level mutable variables, class-level singletons) to store per-request or per-event data in an async application — global state is shared across all concurrent coroutines; use `contextvars.ContextVar` for request-scoped state and pass room/event state explicitly as function arguments.

78. NEVER implement Saga compensations that modify state that was not written by the Saga — compensating transactions are reversals of specific Saga steps; a compensation that deletes records created by a different workflow causes data corruption.

79. NEVER use Redis Pub/Sub for Matrix AS event delivery between the transaction endpoint and the worker — Pub/Sub is ephemeral; if the worker is momentarily disconnected, published messages are lost, creating invisible event gaps; use Redis Streams with consumer groups.

80. NEVER assume Matrix event delivery is ordered — federation means events from the same room can arrive at different homeservers in different orders; `depth` tells you the causal position in the DAG, but event delivery is not guaranteed to be in depth order; always use `prev_events` to reconstruct ordering.

81. NEVER hardcode Matrix room IDs in application code — room IDs are opaque identifiers (`!random:server`) and are not stable across server migrations or room upgrades; store them in configuration or resolve them from aliases (`#channel-name:server`) at startup.

82. NEVER process Matrix transactions in parallel across the same room without a room-level mutex — two concurrent handlers for the same room will race when reading and writing room state; use `asyncio.Lock` keyed by `room_id` to serialize per-room processing.

83. NEVER use `time.sleep()` in a Saga compensation loop — synchronous sleep blocks the event loop in an async application; use `await asyncio.sleep(delay)` with exponential backoff to retry failed compensation steps.

84. NEVER conflate `as_token` and `hs_token` in configuration or code — `as_token` authenticates the AS to Synapse; `hs_token` authenticates Synapse to the AS; swapping them means AS requests to Synapse are rejected and Synapse requests to the AS are accepted from anyone who guesses the as_token.

85. NEVER use `IN MEMORY` deduplication for Matrix AS transaction IDs in a production system — process restarts are normal (deployments, crashes, scaling); in-memory state is lost on restart; the 5-minute startup window where all recent transaction IDs are forgotten will cause duplicate event processing.

86. NEVER publish Kafka messages inside a database transaction without the Outbox pattern — if the transaction rolls back, the Kafka message is already published and cannot be recalled; downstream consumers will process an event for data that no longer exists.

87. NEVER run Raft with an even number of nodes — with 4 nodes and a 2-2 split, no majority is possible and the cluster cannot elect a leader; always use odd numbers: 3 (tolerates 1 failure), 5 (tolerates 2 failures), 7 (tolerates 3 failures).

88. NEVER use wall clock timestamps as the sole mechanism for "latest write wins" conflict resolution in distributed systems where clocks are not synchronized — two nodes writing 1 microsecond apart with unsynchronized clocks can produce arbitrary orderings; use Hybrid Logical Clocks (HLC) or vector clocks when causal ordering matters.
---

## Federation Security and Synapse Operations

89. ALWAYS fetch and cache the originating server's signing keys from `/_matrix/key/v2/server` before verifying federation PDU signatures, and re-fetch when verification fails with the cached key — keys are rotated; a stale cached key causes legitimate events to be rejected; never skip verification because the key fetch itself failed (fail closed, not open).

90. ALWAYS validate that the `sender` MXID domain in a PDU matches the `origin` field and the sending server's TLS certificate subject — a federated server claiming to send events from a different server's users is a spoofing attack; all three must agree before the event is accepted.

91. NEVER allow federation connections from servers not present in the room's `m.room.member` state at the time of the event — a server with no members in the room has no standing to send events to it; reject at the federation layer before signature verification to avoid wasted compute on attack traffic.

92. ALWAYS configure Synapse workers with a dedicated `federation_reader` worker and route `/_matrix/federation/` to it — the main Synapse process has a single asyncio event loop; federation ingest on the main process starves client requests during federation storms; separate workers have independent event loops.

93. ALWAYS tune PostgreSQL `work_mem` per Synapse workload to at least 32MB and set `max_connections` to `(synapse_workers * 10) + 20` — Synapse's state resolution queries perform large in-memory sorts; insufficient `work_mem` causes spill to disk, turning millisecond state resolution into seconds.

94. ALWAYS implement poison pill detection in AS event processing by counting consecutive failures per `event_id` before moving to the dead letter queue — a single permanently unprocessable event will block the entire queue if retried indefinitely; after 3 consecutive failures on the same `event_id`, route to DLQ and advance the queue.

95. ALWAYS use fencing tokens when implementing distributed locks with Redis SET NX EX — the token is a monotonically increasing value from `INCR`; before any locked operation executes, verify the fencing token still matches the stored value; a stale lock holder with an expired token will detect the mismatch and abort.

96. ALWAYS configure Redis Streams consumer groups with `XGROUP CREATE ... MKSTREAM` and monitor the Pending Entry List (PEL) via `XPENDING` — a growing PEL means consumers are claiming messages but not ACKing; PEL entries older than 2x expected processing time indicate crashed consumers; use `XCLAIM` to reassign stale entries to healthy consumers.

97. ALWAYS set Kafka `max.poll.interval.ms` to at least 3x your worst-case processing time and `session.timeout.ms` to at most `max.poll.interval.ms / 3` — if a consumer stalls on a slow record beyond `max.poll.interval.ms`, the broker evicts it and triggers a rebalance that stalls ALL partitions in the group.

98. ALWAYS validate the bot's namespace regex from the AS registration YAML against incoming event `sender` and `room_alias` fields in application code — Synapse namespace filtering is advisory; the AS still receives out-of-namespace events during room upgrades and federation edge cases; re-validate in the handler to prevent acting on events the bot should not process.

99. ALWAYS use OR-Set (Observed-Remove Set) CRDT instead of a G-Set when elements can be removed — G-Sets are add-only; attempting removal with a separate tombstone set produces incorrect results under concurrent add+remove; OR-Set tracks unique add-tags per element, allowing re-addition after removal without resurrection of prior removals.

100. ALWAYS implement health checks as three distinct probes — liveness (is the process alive), readiness (can it accept traffic: DB pool, broker connected), and dependency (are downstream services reachable) — a single 200/500 for all three causes Kubernetes to kill pods that are merely waiting for a dependency to recover, amplifying cascading failures.

101. ALWAYS implement a circuit breaker on outbound federation HTTP requests with a half-open probe interval of 30 seconds — without it, a slow or adversarial federation target fills your worker's connection pool with stalled requests, starving all room processing; break at 5 consecutive timeouts.

102. NEVER commit Kafka consumer offsets before processing completes when using manual offset management (`enable.auto.commit=false`) — auto-commit advances the offset on a schedule regardless of processing status, causing permanent message loss on consumer restart after a crash mid-processing.

103. ALWAYS set `power_level_content_override` when creating a Matrix room for a bot-managed space — default power levels grant all members equal power; bots require at minimum PL 50 to invite/kick and PL 100 for themselves to avoid being removed by a later-joining member.

104. NEVER use `join_rule: public` for rooms containing sensitive context (LLM conversation history, PII, internal tooling state) — use `invite` or `restricted` (MSC3083); restricted rooms require membership in a specified space, providing access control without per-user invites at scale.

105. ALWAYS send `m.room.power_levels` state before sending any other state events during room creation via the AS API — out-of-order state can leave the room in a state where the bot lacks power to correct it.

106. ALWAYS use `depth` for causal ordering of events within a room, never `origin_server_ts` — `origin_server_ts` is set by the originating server and can be spoofed or skewed by clock drift; `depth` is computed by Synapse from the DAG and reflects topological order; treat `origin_server_ts` as display metadata only.

107. ALWAYS handle out-of-order event delivery in AS transaction handlers by storing events with their `event_id` and processing them in `depth` order within a room — federation means events can arrive in any order; assuming arrival order equals causal order produces incorrect state for multi-user rooms.

108. ALWAYS implement outbox polling with exponential backoff starting at 100ms and capping at 30 seconds, with jitter (`random.uniform(0, interval * 0.25)`) — a fixed 1-second interval wastes CPU at idle and creates latency spikes; jitter prevents thundering herd when multiple publishers restart simultaneously.

109. ALWAYS batch outbox events by partition key when publishing to Kafka — `SELECT ... ORDER BY partition_key, sequence_num LIMIT 500` ensures events for the same entity are published in order; a single `producer.send()` per partition key preserves per-entity ordering.

110. ALWAYS delete or archive outbox rows within 24 hours of successful publish — leaving processed rows causes table bloat that degrades the `SELECT ... FOR UPDATE SKIP LOCKED` scan; use a partial index on `(status, published_at)` for the cleanup job.

111. ALWAYS propagate W3C `traceparent` through Matrix custom event content for AI-generated actions — embed `"traceparent": trace_context` in the event's `content`; downstream AS instances that process the event can extract and continue the trace, making the full causal chain visible in Jaeger without relying on HTTP headers that Matrix federation strips.

112. ALWAYS implement backpressure in the AS worker by tracking in-flight event count — when it reaches the limit, stop calling `XREADGROUP` (Redis) or `consumer.pause()` (Kafka) until count drops below 80%; without backpressure a slow LLM response causes unbounded queue growth and OOM.

113. NEVER implement E2EE (Olm/Megolm) in Application Service bots unless the use case explicitly requires it — AS bots use server-side tokens and Synapse reads plaintext; E2EE in a bot requires managing per-device session state, key rotation, and `m.room_key` loss recovery, all significant operational burden.

114. ALWAYS detect split-brain by requiring a quorum write (majority of nodes must acknowledge) for globally unique operations — detecting a partition via heartbeat timeout is insufficient because the window between partition start and detection allows both sides to accept writes; use Raft-based consensus (etcd, Consul) or PostgreSQL advisory locks with lease TTL.

115. ALWAYS run Synapse's `purge_history` Admin API on a maintenance schedule for high-volume rooms — Matrix event DAGs are append-only; without periodic purges the `events` table grows without bound; retain the last 90 days for active rooms and run `VACUUM ANALYZE events` afterward; log `room_id`, `events_deleted`, and `timestamp` for compliance tracking.

116. ALWAYS set `stream_writers` and `instance_map` in Synapse's `homeserver.yaml` when running multiple workers — without explicit stream writer assignment, all workers contend on the same stream locks; assign `events` to a dedicated writer worker and route federation inbound to a dedicated federation reader worker.

117. NEVER send typing notifications (`m.typing`), presence updates (`m.presence`), or read receipts (`m.receipt`) from Application Service bot users unless the bot is explicitly representing a human session — these ephemeral events generate fanout to every room member and increase Synapse load.

118. ALWAYS suppress read receipts for AS virtual users by omitting `m.receipt` sends and setting `"presence": "offline"` in the AS registration — a bot that joins 10,000 rooms and sends per-event read receipts creates O(members × events) fanout that saturates the Synapse notifier.

119. ALWAYS implement the transactional outbox with asyncpg using a single `BEGIN`/`COMMIT` that writes both the business row and the outbox row atomically — if the publisher crashes after `COMMIT`, the outbox row persists and the relay picks it up on restart; if before `COMMIT`, neither row exists, preventing dual-write inconsistency.

120. NEVER use a separate transaction for the outbox INSERT — splitting the business write and outbox write across two transactions creates a window where the business row commits but the outbox row does not, silently dropping the event.

121. ALWAYS implement the outbox relay with `SELECT ... WHERE processed_at IS NULL ORDER BY id FOR UPDATE SKIP LOCKED LIMIT N` — `SKIP LOCKED` enables multiple relay workers to consume in parallel without lock contention; mark rows `processed_at = NOW()` only after downstream publish is confirmed.

122. ALWAYS build event sourcing projections as separate read-model tables populated by a projection worker consuming the event log in append order — never query the raw event log for read paths; projection tables have schemas optimised for each query pattern.

123. ALWAYS implement projection rebuild by replaying the event log in batched `SELECT ... WHERE id > $last_seen ORDER BY id LIMIT 1000` pages, updating a `projection_checkpoint` row atomically with each batch — a single-transaction rebuild holds a write lock for its full duration and will timeout on large logs.

124. ALWAYS use the aggregate snapshot pattern when event history exceeds ~500 rows — write `(aggregate_id, version, snapshot_blob)` periodically; on load, fetch the latest snapshot then replay only events with `version > snapshot.version`; replaying 50,000 events per command is O(N) per write.

125. ALWAYS maintain strict separation between CQRS write model (command handlers that append events) and read model (query handlers that read projections) — a command handler that reads a projection reintroduces the consistency problems CQRS is designed to avoid.

126. ALWAYS return a `X-Read-Model-Version` response header containing the highest event sequence number reflected in the projection — clients that just issued a command can poll until this version advances past their command's event sequence before refreshing, avoiding "I just did X and nothing changed" UX failures.

127. ALWAYS use `XADD mystream MAXLEN ~ 100000 * field value` (approximate trimming with `~`) for Redis Streams retention — approximate trimming batches trim to macro-node boundaries and is orders of magnitude cheaper than exact `MAXLEN` trimming on every append.

128. PREFER `MINID ~ <id>` over `MAXLEN` for time-based Redis Stream retention — `MINID` trims entries older than a given timestamp ID; `MAXLEN` retains a fixed count regardless of time, which under bursty load retains only seconds of data.

129. NEVER rely solely on `MAXLEN` trimming — also monitor `XLEN` and set `maxmemory-policy noeviction`; Redis will not evict stream entries under memory pressure unless explicitly configured, and a misconfigured trim silently allows unbounded growth.

130. NEVER configure Synapse push rules to notify on every AS bot message — set `"default_rule": "suppress"` for bot virtual users; a room with 500 members receiving 100 bot messages/minute generates 50,000 push evaluations per minute on the Synapse push evaluator.

131. ALWAYS implement cross-instance distributed rate limiting using a Redis sorted-set sliding window wrapped in a Lua script (`EVALSHA`) — `ZADD ratelimit:{id} <now_ms> <uuid>; ZREMRANGEBYSCORE ...; ZCARD ...; EXPIRE ...`; Lua atomicity prevents two concurrent requests from both reading `count = N-1` and both proceeding.

132. NEVER publish raw JSON messages to Kafka topics consumed by more than one team — without a schema registry (Confluent, AWS Glue, Apicurio) there is no contract enforcement; a producer renaming a key silently breaks all consumers.

133. ALWAYS register schemas with `BACKWARD` compatibility mode — allows consumers on the old schema to read messages from producers using the new schema; deploy consumers before producers during rolling upgrades to avoid a compatibility window.

134. ALWAYS handle `SIGTERM` in AS workers by: (1) stopping acceptance of new transactions, (2) draining the in-flight queue, (3) committing the consumer offset checkpoint, (4) calling `sys.exit(0)` — a worker killed mid-transaction re-receives the same transaction on restart (Synapse retries unacknowledged transactions).

135. ALWAYS call `consumer.commit(offsets)` synchronously inside the `SIGTERM` handler before `consumer.close()` — relying on auto-commit interval means the last processed-but-not-committed batch is redelivered after restart.

136. ALWAYS monitor Synapse room state size and alert at 5,000 state events per room — a room with 10,000 joined members has at least 10,000 `m.room.member` entries; every state resolution after a fork evaluates all of them; consider using a dedicated low-membership bot room for AS event delivery instead of large human-occupied rooms.

137. NEVER discard a `m.room.tombstone` event — mark the old room as archived, store the `replacement_room` field, and join the new room via the `via` servers before further interaction; an AS that continues sending to a tombstoned room receives 400 errors indefinitely.

138. ALWAYS migrate per-room account data to the replacement room when handling `m.room.tombstone` — `m.fully_read` markers and custom account data are not automatically ported; read from the old room before it becomes inaccessible and re-write to the new room.

139. ALWAYS use `INSERT ... ON CONFLICT DO NOTHING RETURNING id` for idempotent event ingestion — zero rows returned means "already processed, skip downstream effects"; this eliminates the read-before-write race condition in application-level deduplication.

140. PREFER `INSERT ... ON CONFLICT (key) DO UPDATE SET processed_at = EXCLUDED.processed_at RETURNING (xmax = 0) AS inserted` when you need to distinguish a genuine insert from a duplicate — `xmax = 0` is true only for the inserting transaction; the only lock-free way to branch on first-seen vs duplicate in a single round trip.

141. NEVER hold a Raft leader lease longer than `election_timeout / 2` without refreshing via heartbeat — a leader that stops sending heartbeats but does not crash retains its lease while a new leader is elected, creating a split-brain window.

142. ALWAYS treat Raft split-vote as a liveness failure, not a safety failure — split votes leave the cluster leaderless until the next randomized election timeout; retry with backoff and do NOT assume a timeout means the previous operation was not applied.

143. ALWAYS use `pg_try_advisory_lock(key)` rather than `pg_advisory_lock(key)` for background job coordination — `pg_advisory_lock` blocks indefinitely; `pg_try_advisory_lock` returns false immediately; simulate TTL with a `locked_until` timestamp column.

144. NEVER use session-level advisory locks in a connection pool — the lock persists on the session after the connection returns to the pool; always use `pg_try_advisory_xact_lock` (transaction-level), which releases automatically on COMMIT/ROLLBACK.

145. ALWAYS deduplicate federation transactions by persisting `(origin, txnId)` with `UNIQUE` constraint and `INSERT ON CONFLICT DO NOTHING` before processing — Synapse retries transactions on timeout; processing a duplicate twice corrupts room state or double-charges operations.

146. NEVER rely on federation retry as a signal that delivery failed — a 200 acknowledges receipt and prevents retransmission permanently; write the raw transaction payload to durable storage atomically with the 200 response, or return 500 to force a retry.

147. ALWAYS inject W3C `traceparent` into message envelope headers at publish time and extract it at consume time — for Kafka use `Headers`; for Redis Streams use a payload field; a consumer that creates a root span rather than a child span produces disconnected traces.

148. NEVER create a DLQ consumer that unconditionally requeues messages — inspect the message, log a structured error, increment a per-message-type DLQ depth counter, and require a human or automated decision gated on `max_dlq_requeue_attempts`; poison pills cycling between main queue and DLQ mask root causes.

149. ALWAYS alert when DLQ depth exceeds a per-queue threshold within 5 minutes — set at depth > 0 for queues where DLQ entries should never occur (financial writes, event sourcing); a DLQ growing silently indicates the alert was never configured.

150. NEVER read `m.direct` account data without handling the case where it is absent — `m.direct` is set by clients, not servers; treat absence as "no direct rooms known"; also note that `m.direct` values are NOT updated when a room is tombstoned.

151. NEVER modify `m.push_rules` account data via the raw `account_data` endpoint — use the dedicated `/_matrix/client/v3/pushrules/` API which validates rule structure; raw account data writes bypass validation and can silently disable all push for a user.

152. ALWAYS implement Saga compensation transactions as idempotent operations with a `compensation_idempotency_key` stored in the Saga log — derive it deterministically from the original step key (e.g. `sha256(f"compensate:{original_step_key}")`); non-idempotent compensations double-compensate on retry.

153. WHEN a Saga compensation fails after exhausting retries, NEVER silently mark the Saga as failed — record it in a `saga_stuck` table, emit a `saga.compensation.stuck` metric, and page on-call; the system is in an inconsistent state requiring human intervention.

154. ALWAYS include a `schema_version` integer (starting at 1, never 0) in every event-sourced event payload — upcasting functions at read time need this to dispatch correctly; an event without a version cannot be safely upcasted.

155. ALWAYS implement event upcasting as a pure function applied at read time, not write time — rewriting old events destroys the audit trail and invalidates snapshots; upcasters must be composable and tested with property-based tests against historical events.

156. NEVER remove an upcasting function until no events with that `schema_version` exist anywhere including cold storage and DR backups — a snapshot built from an old event using a deleted upcaster will fail to replay.

157. ALWAYS use the `from` token from the previous `/messages` response for Matrix timeline pagination, never construct tokens manually — tokens are opaque server-side cursors; stop when `end` is absent (beginning of visible history) or `chunk` is empty, not when you estimate you have enough events.

158. NEVER assume paginating `/messages` with a large `limit` returns all events in order — the homeserver may return fewer events than `limit`; always apply a `RoomEventFilter` excluding state events when building a message corpus for RAG ingestion.

159. ALWAYS record Kafka consumer lag as a Prometheus gauge per `(topic, partition, consumer_group)` triple — aggregate lag hides a single hot partition where one consumer is stuck while others are healthy; alert when per-partition lag exceeds `acceptable_reprocess_seconds * messages_per_second_per_partition`.

160. ALWAYS measure message processing latency as wall-clock time from the publication timestamp embedded in the message envelope to offset commit — broker enqueue time measures queue depth, not end-to-end latency.

161. NEVER use a high-cardinality skewed value as the sole Kafka partition key — 1% of keys producing 80% of volume routes 80% of traffic to one partition; detect via per-partition byte rate in broker JMX; mitigate by appending `random.randint(0, N-1)` to the key for high-volume producers and aggregating at the consumer.

162. NEVER repartition a Kafka topic with active consumers without first pausing all consumers and committing offsets — the safest strategy is creating a new topic with target partition count, migrating producers first, then consumers, with a dual-write window.

163. ALWAYS verify the `hs_token` on every incoming AS transaction using `hmac.compare_digest` — missing or mismatched token must return 403 immediately; never log the token value; an AS without token verification is an unauthenticated event injection endpoint.

164. ALWAYS enforce a maximum payload size (e.g. 10 MB) on incoming AS transactions at the HTTP middleware layer before `json.loads()`, and cap events per transaction at a configurable limit (default 100) — return 400 if exceeded; never process oversized transactions partially.

165. PREFER per-tenant Kafka topics over a shared topic with `tenant_id` filter when tenants have significantly different throughput or compliance isolation requirements — a shared topic requires deserializing every message to filter by tenant; switch to shared topics only above the broker's practical per-topic overhead threshold.

166. NEVER use a shared consumer group across tenants in a shared-topic architecture without partitioning by `hash(tenant_id) % num_partitions` — unaligned partitioning causes slow tenants to stall consumers for the partition they share with fast tenants, introducing cross-tenant latency coupling.
