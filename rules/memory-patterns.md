---
description: AI memory architecture rules — Mem0, Qdrant, layered memory (short-term/long-term/episodic/procedural), injection, extraction, privacy.
paths: ["*.py", "app/**/*.py", "memory/**/*.py"]
---

# AI Memory Patterns (L18)

> Qdrant vector DB rules → see `rag-patterns.md` for pgvector alternative
> Async patterns → see `async-patterns.md`
> Privacy / GDPR erasure obligations → see `security-complete.md`
> Observability (latency histograms, trace binding) → see `observability-patterns.md`

---

## Memory Layer Architecture

1. ALWAYS treat short-term memory as session-scoped LangGraph `messages` state only — NEVER persist raw short-term content directly to a vector database.

2. ALWAYS store long-term semantic memory as extracted fact embeddings in Qdrant or pgvector — NEVER store full conversation text as long-term memory entries.

3. ALWAYS store episodic memory as compressed session summaries (text + embedding) — NEVER treat episodic records as short-term state.

4. ALWAYS treat procedural memory (system prompts, rules files) as static configuration loaded at startup — NEVER write procedural rules to a runtime vector database.

5. NEVER mix memory layers: short-term `messages` state must not be written to the long-term fact store, and long-term facts must not be injected back into the `messages` list as if they were prior turns.

6. ALWAYS write to long-term memory asynchronously after the HTTP response is sent — NEVER block the response path on a memory write operation.

7. ALWAYS cap the total token count of injected memories at 15% of the available context window — NEVER allow memory injection to crowd out the current conversation turns.

8. ALWAYS read memory in this order: short-term first, long-term semantic second, episodic summaries third — NEVER treat all layers as equal-priority sources.

9. ALWAYS extract new facts after each complete user-assistant turn — NEVER run extraction mid-turn while the LLM is streaming.

10. ALWAYS compute cosine similarity between a new candidate fact and existing memories before writing, and skip the write when similarity exceeds 0.95 — NEVER write a duplicate memory entry.

11. ALWAYS tag every memory entry with one of these categories: `preference`, `fact`, `constraint`, or `goal` — NEVER write an untagged memory record.

12. NEVER use short-term session memory as the source for long-term writes without an extraction step — raw dialogue contains noise, filler, and retracted statements that must not be persisted verbatim.

---

## Mem0 Integration

13. ALWAYS initialise the Mem0 `AsyncMemory` client once inside the FastAPI lifespan startup block and inject it via `Depends()` — NEVER construct a new `AsyncMemory` instance per request.

14. ALWAYS pass `user_id` to every `m.add()` and `m.search()` call — NEVER call either method without scoping to a specific user or tenant.

15. ALWAYS use `AsyncMemory` (not the synchronous `Memory` class) in any `async def` route handler or background task — NEVER call blocking Mem0 methods inside an async context without `run_in_executor`.

16. ALWAYS pass the full message list (system + user + assistant turns) to `m.add()` — NEVER pass only the latest user message, because extraction requires conversational context.

17. ALWAYS check the `score` field on every result returned by `m.search()` before injecting into the prompt and discard results with score below 0.7 — NEVER blindly inject all returned memories regardless of relevance.

18. ALWAYS implement the GDPR erasure endpoint by calling `m.delete_all(user_id=uid)` and verifying the return count — NEVER consider erasure complete until the call succeeds and the count matches stored records.

19. NEVER call `m.search()` inside the LLM token-generation stream — ALWAYS pre-fetch memories before initiating the LLM call.

20. ALWAYS configure `memory_config` with an explicit cheap model (e.g., `claude-haiku-4-5`) for the extraction step — NEVER let Mem0 default to the same model used for generating responses.

21. ALWAYS use a different, cheaper model for memory extraction than the model used for the primary response — NEVER use the same model for both to avoid cost amplification on every turn.

22. NEVER store raw conversation text in long-term Mem0 memory — ALWAYS configure the extraction prompt to produce structured facts only.

23. ALWAYS include a `created_at` ISO timestamp and a `confidence` float (0.0–1.0) in every memory payload — NEVER write a memory entry without these two fields.

24. ALWAYS write a test that calls `m.add()` twice with identical content and asserts that `m.search()` returns exactly one result — NEVER ship without verifying Mem0's built-in deduplication is active.

25. NEVER use the Mem0 in-memory backend in any environment outside local development or unit tests — ALWAYS use a persistent Qdrant or pgvector backend in production.

---

## Qdrant Configuration

26. ALWAYS create Qdrant collections with HNSW parameters `m=16` and `ef_construction=200` — NEVER use the default `ef_construction=100` for production workloads where recall matters.

27. ALWAYS set `ef` (search-time `ef_search`) to a minimum of 100 when querying — NEVER reduce below 100 to chase latency at the cost of recall accuracy.

28. ALWAYS create collections with named vectors (`vectors_config` dict) when storing more than one embedding type — NEVER store mixed-dimension embeddings in an unnamed single-vector collection.

29. ALWAYS set `on_disk=True` for vector storage when a collection is expected to exceed 1 million points — NEVER rely on in-memory vector storage at that scale or OOM events will terminate the node.

30. ALWAYS enable scalar quantization (`ScalarQuantizationConfig(type=ScalarType.INT8)`) on collections larger than 500K points to reduce memory footprint by 4x — NEVER enable quantization without first verifying recall does not drop below your SLO in staging.

31. NEVER insert vectors from different embedding models (different dimensions) into the same Qdrant collection — ALWAYS create a separate collection per embedding model.

32. ALWAYS include `user_id` and `tenant_id` as payload fields on every point upserted into Qdrant — NEVER store a point without these two isolation identifiers.

33. ALWAYS create payload indexes on `user_id`, `category`, and `created_at` before going to production — NEVER rely on full payload scan for filtered searches on large collections.

34. ALWAYS include a `must` filter condition on `user_id` in every Qdrant `search` call — NEVER issue an unfiltered vector search that could return another user's memories.

35. ALWAYS take Qdrant collection snapshots via `POST /collections/{name}/snapshots` as part of the daily backup schedule — NEVER rely solely on disk-level backups.

36. ALWAYS use Qdrant Cloud for managed deployments and self-hosted Docker for development — NEVER use the Python in-memory Qdrant client in production.

37. NEVER hardcode Qdrant collection names as string literals scattered through application code — ALWAYS define them as named constants in a settings module.

38. ALWAYS set an explicit timeout (10 seconds default) on all Qdrant client calls — NEVER leave the timeout unbounded.

39. ALWAYS use batch upsert (`client.upsert(collection_name, points=[...])`) when writing more than 10 points — NEVER loop over individual `upsert` calls for bulk memory writes.

40. ALWAYS monitor `qdrant_collection_vectors_count` and alert when growth rate exceeds baseline expectations — NEVER let a runaway extraction loop silently inflate the collection until disk is exhausted.

---

## Memory Injection

41. ALWAYS inject retrieved memories into the system prompt — NEVER append retrieved memories to the user message turn, which contaminates the user's stated intent with prior context.

42. ALWAYS format injected memories as a numbered bullet list with each item prefixed by its category tag (`[preference]`, `[goal]`, etc.) — NEVER inject memories as unstructured prose blocks.

43. ALWAYS include the retrieval score alongside each injected memory so the LLM can weight relevance — NEVER strip scores before injection.

44. ALWAYS cap injected memories at the top-5 results ranked by combined relevance-freshness score — NEVER inject more than 5 memories per request.

45. NEVER inject a memory with a cosine similarity score below 0.7 — ALWAYS filter low-confidence retrievals before injection.

46. ALWAYS apply a freshness decay weight using `score * exp(-days_since_created / 30)` when ranking candidates — NEVER rank by semantic score alone without factoring in recency.

47. ALWAYS pre-fetch memories before initiating the LLM API call — NEVER trigger memory retrieval after streaming has started.

48. ALWAYS cache memory retrieval results keyed by `(user_id, sha256(query))` with a 60-second TTL — NEVER issue repeated Qdrant searches for the same user and query within a session retry window.

49. ALWAYS implement a circuit breaker on the memory retrieval path so a Qdrant outage causes the LLM call to proceed without memories — NEVER block or fail the primary response due to memory store unavailability.

50. ALWAYS ensure that identical `(user_id, query, session_id)` inputs produce the same injected memory set — NEVER produce non-deterministic injection sets that cause inconsistent LLM behaviour across retries.

51. ALWAYS instrument `memory_hits_total` and `memory_miss_total` as Prometheus counters — NEVER operate memory injection without these two observability signals.

52. NEVER expose raw Qdrant point IDs, vector scores, or internal memory UUIDs in any API response body — ALWAYS treat memory store internals as implementation details.

---

## Memory Extraction

53. ALWAYS run memory extraction as a `BackgroundTask` dispatched after `return response` — NEVER run extraction synchronously in the request path where it adds 500–1500ms to perceived latency.

54. ALWAYS use a dedicated extraction prompt: "List new facts about the user from this conversation turn. Facts only. No reasoning." — NEVER add extraction instructions to the response generation prompt.

55. ALWAYS validate every extraction result with a Pydantic schema before writing (`fact: str`, `category: Literal[...]`, `confidence: float`) — NEVER store unvalidated LLM output directly.

56. NEVER store an extracted fact with confidence below 0.7 — ALWAYS discard low-confidence extractions at the validation layer.

57. NEVER extract and store: session identifiers, authentication tokens, passwords, raw payment card data, or medical lab values without clinical context — ALWAYS run a pre-write guard rejecting these patterns.

58. ALWAYS check for semantic duplicates by embedding the new fact, searching existing memories with a `user_id` filter, and skipping the write when cosine similarity exceeds 0.95 — NEVER write a fact already effectively represented in the store.

59. ALWAYS issue memory extraction as a separate LLM API call — NEVER concatenate extraction instructions onto the response generation prompt.

60. ALWAYS limit extraction output to a maximum of 5 new facts per turn — NEVER accept more than 5, as that indicates an overly broad extraction prompt capturing noise.

61. ALWAYS attach `source_turn_id` to every extracted memory payload — NEVER write a memory entry without a reference to the conversation turn that produced it.

62. ALWAYS log extraction failures at ERROR level with exception type and `user_id` — NEVER surface extraction errors to the end-user.

---

## Privacy and Multi-Tenancy

63. ALWAYS include a `must` filter on `user_id = current_user.id` in every Qdrant search call — NEVER issue a memory search without this user-isolation filter, even in admin or debugging contexts.

64. ALWAYS add `family_id` or `group_id` to memory payload and use a `should` filter alongside the `must user_id` filter when group-shared memories are a product requirement — NEVER expose group memories to users outside the authorised group.

65. NEVER allow memory search results from one user to appear in another user's retrieval results — ALWAYS verify cross-user isolation with a dedicated integration test.

66. ALWAYS implement GDPR right-to-erasure by deleting all Qdrant points filtered by `user_id`, then PostgreSQL, then Redis — ALWAYS perform in that order and NEVER consider erasure complete until all three stores confirm deletion.

67. ALWAYS run a regex and NER pass on extracted text before any memory write and reject when SSN patterns, credit card numbers, or medical diagnosis terms are detected — NEVER store bare PII in the memory store.

68. ALWAYS write an audit log entry containing `user_id`, `operation` (`write` or `read`), `memory_id`, and `timestamp` for every memory write and retrieval — NEVER operate the memory store without this audit trail.

69. ALWAYS execute memory deletion synchronously (not via an async queue) to satisfy GDPR 30-day erasure SLAs — NEVER queue erasure requests.

70. NEVER use a shared or global Qdrant collection where points from multiple users coexist without payload-level `user_id` isolation.

71. ALWAYS write an integration test asserting user A's memories return zero results when searched under user B's credentials — NEVER ship a memory feature without this passing in CI.

72. ALWAYS document the memory data retention period in the privacy policy and enforce it with an automated expiry job — NEVER rely on manual cleanup to meet retention obligations.

---

## Production Operations

73. ALWAYS pre-warm a user's memory context by fetching their top-10 memories before the first LLM call of a new session — NEVER wait for an explicit query to trigger memory retrieval.

74. ALWAYS configure the Qdrant client to use gRPC transport (`grpc_port=6334`) in production for lower serialisation overhead — NEVER use HTTP/REST transport for latency-sensitive high-throughput memory operations.

75. ALWAYS monitor Qdrant search latency at P99 and alert when it exceeds 100ms — HNSW searches should complete in under 20ms on a properly indexed collection.

76. ALWAYS monitor the memory extraction background task queue depth and alert when it exceeds 100 pending items — NEVER let backlog grow unbounded.

77. ALWAYS run a monthly compaction job that summarises episodic memory entries older than 90 days into shorter summaries and deletes the originals — NEVER allow unbounded growth of verbatim episodic records.

78. NEVER schedule the memory compaction job during peak traffic hours — ALWAYS run it during off-peak hours to avoid competing with real-time search and write workloads.

79. ALWAYS use Qdrant collection aliases for zero-downtime re-indexing: build the new collection, switch the alias atomically with `PUT /aliases`, then delete the old collection — NEVER re-index in place.

80. ALWAYS wrap all Qdrant client calls in a circuit breaker with a 5-second open window so a Qdrant outage produces graceful memory-less degradation — NEVER propagate Qdrant connection errors into the LLM call path.

81. ALWAYS benchmark memory retrieval with at least 1 million points in staging before production launch — NEVER assume HNSW recall and latency will hold at scale without empirical validation.

82. ALWAYS record `memory_extraction_latency_ms` and `memory_search_latency_ms` as Prometheus histograms with buckets covering 5ms to 2000ms — NEVER rely on average latency alone.

---

## Anti-Patterns

83. NEVER store full conversation text as a long-term memory entry — ALWAYS extract discrete facts through a dedicated extraction call before writing.

84. NEVER use the same LLM model to both generate the primary response and verify the quality of extracted memories — ALWAYS use a separate judge model to avoid self-serving bias.

85. NEVER call the Qdrant client inside an open SQLAlchemy database transaction — ALWAYS perform vector store operations outside transaction boundaries.

86. NEVER execute memory extraction synchronously in the HTTP request path — ALWAYS offload to a `BackgroundTask` or task queue.

87. NEVER cache extracted memory payloads without an explicit TTL — ALWAYS set a TTL of no more than 24 hours on any memory-related cache entry.

88. NEVER delete memories automatically at user session end — ALWAYS retain memories across sessions and delete only on explicit GDPR erasure request or documented retention policy expiry.

89. NEVER set the cosine similarity deduplication threshold above 0.95 — thresholds above that value miss semantically near-duplicate facts that differ only in surface phrasing.

90. NEVER build or ship a memory feature without a written privacy review and a documented data retention policy specifying what is stored, for how long, and how it is deleted — ALWAYS treat memory architecture as a data-handling obligation, not a pure engineering concern.

---

## LangMem and Multi-Agent Memory

91. ALWAYS initialize the LangMem `InMemoryStore` or `PostgresStore` once at application startup and pass it as a dependency — NEVER construct a new store instance per agent invocation, because each new store instance loses all prior context.

92. ALWAYS use LangMem's `get_memories(namespace=("user", user_id))` with an explicit namespace tuple rather than a flat string key — NEVER share a single flat namespace across multiple users or multiple agent roles, because namespace collision produces cross-user memory bleed identical to a missing `user_id` filter in Qdrant.

93. ALWAYS assign each agent in a multi-agent system its own write namespace (`("agent", agent_id, user_id)`) and a separate read namespace for cross-agent shared state (`("shared", user_id)`) — NEVER let two agents write to the same namespace concurrently without a compare-and-swap or optimistic lock, because simultaneous writes cause silent fact overwrites.

94. ALWAYS designate a single writer agent per fact category in multi-agent memory — NEVER allow two agents to independently extract and write the same fact category, because divergent writes produce contradictory memories with no conflict resolution mechanism.

---

## Working Memory Overflow

95. ALWAYS truncate a LangGraph `messages` list when its token count exceeds 70% of the model's context window, replacing the oldest turns with a `SystemMessage` containing a progressive summary generated by a cheap model — NEVER silently drop messages without summarizing, because silent truncation loses facts the user stated explicitly.

96. ALWAYS keep the first system message and the last two user-assistant turn pairs outside the truncation window — NEVER summarize the current unanswered turn before the LLM has responded to it.

---

## Embedding Model Selection

97. ALWAYS select an embedding model trained on short declarative sentences for personal fact storage (voyage-3, text-embedding-3-small, or nomic-embed-text) — NEVER use a code-optimized embedding model for natural-language memory, and NEVER use `ada-002` for new deployments because its recall on short personal facts is 8–12% lower than voyage-3 on MTEB personal fact benchmarks.

98. ALWAYS re-embed all existing memory points when changing the embedding model by running a background migration that reads in batches of 256, re-embeds with the new model, upserts to a new collection, and uses a Qdrant collection alias swap for cutover — NEVER overwrite vectors in-place in the existing collection, because a partially migrated collection contains mixed-dimension vectors producing undefined cosine similarity.

---

## Memory Quality Evaluation

99. ALWAYS measure memory quality with two separate weekly metrics: `memory_precision` (fraction of injected memories rated relevant by an LLM judge) and `memory_recall` (fraction of user-stated facts retrievable within 7 days) — NEVER treat Prometheus hit-rate counters alone as a quality proxy.

100. ALWAYS run a weekly A/B shadow evaluation where 5% of requests are served without memory injection and an LLM judge scores both responses — NEVER assume memory is helping without this counterfactual baseline.

---

## RAG vs Personalization Injection

101. ALWAYS inject RAG-retrieved document chunks into the user message turn and inject personalization memories into the system prompt — NEVER merge both types into the same prompt position, because document chunks belong to query context while personalization facts belong to the persistent user model.

---

## Serverless and Verbatim Storage

102. ALWAYS initialize memory clients in serverless functions using a module-level singleton with a `None` guard (`if _client is None: _client = build_client()`) so the client is reused across warm invocations — NEVER initialize inside the handler function body, because Qdrant connection setup adds 200–800ms of cold-start latency.

103. ALWAYS store verbatim user utterances as episodic memory when the product requires exact-quote recall (legal, medical, customer support) — NEVER apply the extraction-only rule to domains where verbatim fidelity is a compliance requirement; in these domains, store both the verbatim episodic record AND the extracted semantic fact, then delete the verbatim record after the retention window expires.

---

## Zep Memory Patterns

104. ALWAYS store Zep fact triples as `(subject, predicate, object)` with an explicit `valid_from` timestamp and a nullable `valid_until` timestamp — NEVER overwrite an existing fact triple on contradiction; instead set `valid_until = now()` on the old triple and insert a new one, because temporal knowledge graphs derive value from full change history.

105. NEVER configure Zep to use the same Neo4j instance for both memory storage and application business-logic graph data — ALWAYS provision a dedicated Neo4j database for Zep's memory graph, because Zep's schema management assumes full namespace ownership and will conflict with application-defined node labels.

---

## Streaming, Hybrid Search, and Negative Memories

106. ALWAYS trigger memory extraction only after the full LLM response stream is complete — attach the extraction background task to the `on_complete` callback of the streaming generator, not on first token or at an arbitrary chunk boundary; extraction requires the complete assistant turn as input, and a partial turn produces incoherent or false memories.

107. ALWAYS use hybrid search (vector similarity + BM25 keyword) when the user's message contains proper nouns, technical identifiers, or quoted strings — pure vector similarity degrades on rare terms like library names, people's names, and version strings; route to Qdrant's sparse-dense hybrid mode (`SparseVector` + `NamedVector`) for exact-term recall; fall back to vector-only when the query contains no capitalised tokens or quoted terms.

108. ALWAYS store negative preferences as first-class procedural memory entries with a `polarity: "negative"` payload field, and render them in the system prompt as explicit prohibitions (`"NEVER mention: [item]"`) — NEVER store a negative preference as a standard vector fact, because it will match on topic similarity and surface at the wrong time.

---

## Concurrent Writes and Cold Start

109. ALWAYS resolve concurrent writes to the same user's memory using a per-user write mutex — use Redis `SET NX PX 5000 user_memory_write_lock:{user_id}` to prevent two app instances from both passing the 0.95 deduplication check simultaneously and writing near-duplicate memories; retry once on lock contention.

110. ALWAYS bootstrap a new user's memory with an onboarding extraction pass immediately after account creation — format the onboarding form fields as a synthetic `"user"` message and run the extraction pipeline on them; tag bootstrapped memories with `source: "onboarding"` to distinguish them from conversation-derived memories.

---

## Episodic Compression and Tool Results

111. ALWAYS cap episodic memory accumulation at a configurable `max_episodes` limit (default: 200) and run hierarchical compression when the limit is reached — batch the oldest 50 episodes, compress them into 5 meta-summaries with a cheap model, write the meta-summaries with `source: "compressed"` and `covers_episode_ids`, then delete the 50 source rows; NEVER compress the 20 most-recent episodes regardless of limit.

112. ALWAYS gate tool call results before storing in long-term memory using a `memory_worthy` binary classifier — store only results containing durable user-specific facts; NEVER store transient results (weather lookups, stock prices, throwaway search results); cap tool-result memories at 20% of a user's total memory budget to prevent bloat displacing conversational facts.

---

## User Transparency and Capacity Planning

113. ALWAYS expose a `GET /users/{user_id}/memories` endpoint returning the user's full memory set in plain-language format, paginated at 50 items, with `memory_id`, `text`, `created_at`, `source`, and `confidence` on each item — ALWAYS provide `DELETE /users/{user_id}/memories/{memory_id}` for individual deletion; without this, users cannot exercise GDPR right-of-access and the system is unauditable.

114. ALWAYS estimate memory storage growth at capacity planning time using `users × sessions_per_month × facts_per_session × vector_bytes_per_fact` and set a `max_memories_per_user` hard cap (default: 2000 semantic facts) enforced at write time — a 1536-dim float32 vector consumes ~6 KB after payload overhead; apply INT8 scalar quantisation to stay within budget at scale.

---

## Multi-Modal Memory and Quality Evaluation

115. ALWAYS store image content in memory as a structured text description, not as raw bytes or embedding URLs — run the image through a vision model at extraction time and store `"User shared an image of: [structured description]"` as a standard text memory; embed with the same text embedder used for all memories so it participates in unified similarity search.

116. ALWAYS run a weekly `memory_precision` eval on a 5% random sample of stored memories using an LLM judge scoring `1-5` for "Is this specific enough to improve a future response?" and purge memories scoring 1 (generic fillers like "User seems interested in technology") — set minimum rolling threshold of 3.0; tighten the extraction prompt on drops, never lower the threshold.

---

## Agent Role Namespacing

117. NEVER use the same namespace or `user_id` scope for memories written by different agent roles — store research agent, coding agent, and scheduling agent memories in separate namespaces (`research::{user_id}`, `coding::{user_id}`, etc.) and maintain a single `shared::{user_id}` namespace only for facts explicitly promoted after cross-agent consensus.

---

## Qdrant Operational Tuning

118. ALWAYS tune `hnsw_config.ef` (ef_search) dynamically under load — set `ef = max(m * 2, 128)` at index build time for balanced recall/speed; raise to 256 when P99 recall drops below 0.95 under concurrent query load; NEVER leave ef at the default of 100 for collections exceeding 1M points.

119. ALWAYS run a Qdrant collection warm-up query immediately after a pod restart before opening the readiness probe — issue one vector search per collection using a zero vector; HNSW graph pages are cold in OS page cache after restart and the first real query takes 5–30× longer than subsequent queries; the warm-up prevents this from being user-visible.

120. ALWAYS monitor Qdrant's `segments_count` per collection and alert when it exceeds 10 unmerged segments — a high segment count degrades search latency because each query scans all segments independently; trigger a manual `POST /collections/{name}/index` optimise call to force segment merging.

---

## Extraction Scope Rules

121. ALWAYS run memory extraction over tool call arguments in addition to user message turns — user intent encoded in tool arguments (e.g. `"search_query": "best Python async patterns"`) reveals durable preferences that never appear in conversational text; tag extracted memories from tool arguments with `source: "tool_args"`.

122. NEVER extract memories from tool call return values — return values are ephemeral API responses containing transient facts (weather, prices, current time); storing them as long-term memories produces stale facts that degrade injection quality over time.

---

## Freshness Decay and Explicit Corrections

123. ALWAYS compute a composite relevance score as `semantic_score * freshness_weight * importance_weight` where `importance_weight = min(1.0, 0.5 + 0.1 * confirmation_count)` — memories that the user has re-confirmed multiple times should decay more slowly than single-mention facts.

124. ALWAYS implement user corrections as a two-phase atomic operation: (1) set `is_active = false` on the old memory point with a `corrected_by` payload reference, (2) insert the new corrected memory with `source: "user_correction"` and `replaces_id` pointing to the old point — NEVER delete-then-insert without atomicity, because a crash between the two operations leaves the memory in an inconsistent state.

---

## Agentic Loop Self-Reference Guard

125. ALWAYS attach `source_task_id` to every memory written during an agentic loop execution and reject any retrieval query issued from within the same task that would return memories written by that same `source_task_id` — a loop that reads its own prior-iteration writes creates a runaway amplification cycle where each iteration reinforces incorrect beliefs.

126. ALWAYS enforce a `max_memory_writes_per_task = 10` hard cap within a single agentic task execution — raise an `AgentMemoryBudgetExceeded` exception when the cap is hit; NEVER allow an unguarded loop to write an unbounded number of memories in one task invocation.

127. ALWAYS implement write-after-read consistency by adding a `version` integer to memory payloads and using optimistic locking (`if payload.version == expected_version`) before any update write — NEVER perform read-modify-write on a memory entry without checking that the version has not advanced since the read.

---

## Collection Sharding and Analytics

128. ALWAYS shard Qdrant collections when a single collection exceeds 10M vectors — implement consistent-hash routing using a routing table stored in Redis (`HSET memory_shard_map {user_id_hash_bucket} {shard_collection_name}`); NEVER split by arbitrary time windows, because a user's queries must route to a deterministic shard to avoid fan-out search across all shards.

129. ALWAYS maintain a read-only Qdrant replica for analytics and dashboard queries — NEVER run analytics queries (full collection scans, histogram generation, monthly compaction) against the primary write-path collection; analytics scans block real-time HNSW searches for 100–500ms per query on large collections.

130. ALWAYS maintain a materialized summary table in PostgreSQL (`user_memory_stats`: `user_id`, `total_count`, `category_breakdown`, `oldest_memory_at`, `last_write_at`) updated on every memory write — NEVER compute these statistics on-demand from Qdrant; Qdrant is a vector store, not a reporting database, and aggregate queries degrade search performance.

---

## Language and Schema Provenance

131. ALWAYS store a `language` payload field using ISO 639-1 codes (`"en"`, `"fr"`, `"de"`) on every memory point — NEVER store memories from multilingual users without a language tag; the language tag is required for language-filtered retrieval, prevents cross-language false positives in semantic search, and enables per-language quality evaluation.

132. ALWAYS use a multilingual embedding model (e.g. `multilingual-e5-large`, `voyage-multilingual-2`) when the user base includes more than two languages — NEVER use a monolingual model and rely on cross-lingual transfer; recall degrades by 15–30% for out-of-training-language queries on monolingual models.

133. ALWAYS include an explicit `memory_type` field (e.g. `"preference"`, `"fact"`, `"goal"`, `"aversion"`) in the Qdrant payload — NEVER infer memory type from the text at retrieval time; explicit type enables type-filtered injection (e.g. "inject only `goal` memories for planning tasks") and prevents preference memories from appearing in factual recall contexts.

134. ALWAYS set a higher confidence floor (0.85 vs 0.7) for memories labelled `memory_type: "explicit"` (user directly stated a preference) vs `"implicit"` (inferred from behavior) — explicit statements carry higher epistemic weight and should only be stored when extraction confidence is high.

135. ALWAYS store provenance fields on every extracted memory: `conversation_id`, `turn_index`, `session_id`, `extraction_model_version`, and `extracted_at` — NEVER write a memory without full provenance; these fields are required for auditing, debugging extraction model regressions, and selective re-extraction when the extraction model is upgraded.

136. NEVER mutate provenance fields after a memory is written — treat `conversation_id`, `turn_index`, and `extracted_at` as immutable; if provenance data was wrong, set the point `is_active = false` and write a new corrected point rather than updating in place.

137. ALWAYS create a Qdrant payload index on `conversation_id` — retrieval by `conversation_id` is needed for per-conversation memory audits, GDPR access requests, and bulk deletion when a user deletes a specific conversation; without this index the lookup is a full collection scan.

---

## Testing and Recovery

138. ALWAYS write unit tests for the memory extraction function using a mock LLM that returns a controlled list of facts — assert that facts below `min_confidence`, facts matching PII patterns, and facts exceeding `max_facts_per_turn` are rejected before the Qdrant write call is reached.

139. ALWAYS run integration tests against a real ephemeral Qdrant instance (Docker container in CI) rather than mocking the Qdrant client — HNSW behaviour, payload filtering, and collection creation failures cannot be meaningfully reproduced with mock objects; a mock that passes CI tests is not evidence of correct Qdrant integration.

140. ALWAYS define explicit RTO (recovery time objective) = 15 minutes and RPO (recovery point objective) = 1 hour for the memory subsystem — configure Qdrant snapshots to run every 30 minutes to satisfy the RPO; document and test the restore procedure in staging before production launch.

141. ALWAYS store Qdrant snapshots in a geographically separate region from the primary cluster — a snapshot co-located with the primary cannot satisfy the recovery objective during a regional outage; name snapshots with `{collection_name}_{iso_timestamp}.snapshot` for unambiguous ordering during restore.

142. ALWAYS verify snapshot integrity after creation using Qdrant's snapshot checksum endpoint before reporting the backup as valid — NEVER assume a snapshot file is intact without checksum verification; a corrupted snapshot discovered during a recovery incident is worse than no snapshot.

143. ALWAYS implement point-in-time recovery for memory writes by appending every write operation to a PostgreSQL WAL (write-ahead log) table (`memory_write_log`: `id`, `user_id`, `operation`, `payload`, `written_at`) — NEVER rely solely on Qdrant snapshots for granular recovery; snapshots provide collection-level restore, but the WAL enables replaying individual writes from the last snapshot forward to reconstruct the exact state at any timestamp.

144. NEVER promote a memory store from staging to production without running the full snapshot/restore cycle in staging first — an untested restore procedure always fails during a real incident.

---

## Deploy Safety and Rate Limiting

145. NEVER promote memories from a staging environment to a production collection — staging memories contain synthetic test data, developer experiments, and corrupted extractions; import into production would inject malformed facts into real user contexts.

146. ALWAYS run a payload schema compatibility check before deploying a new version of the extraction model — query the live collection for a sample of 100 points and assert all required payload fields (`user_id`, `memory_type`, `language`, `confidence`, `provenance.*`) are present before the deploy proceeds; a new extraction model that drops a payload field breaks all retrieval filters that depend on it.

147. ALWAYS give system-prompt-level procedural memories the highest priority during conflict resolution — when a system-configured procedural memory (`source: "system_config"`) contradicts a user-derived procedural memory, the system-configured one takes precedence; NEVER allow user-stated preferences to override system-level constraints (e.g. safety rules, content policies).

148. ALWAYS enforce a rate limit of 100 implicit memory writes per user per hour using a Redis sliding window counter — a user whose memory write rate exceeds this threshold is triggering either a runaway extraction loop or an adversarial prompt designed to flood the memory store; raise a `MemoryRateLimitExceeded` exception and log the event at WARNING level.

149. ALWAYS apply the 0.95 deduplication check to explicitly user-submitted memories (via `PUT /users/{id}/memories`) in addition to extraction-derived memories — NEVER skip deduplication for user-submitted entries on the assumption they are authoritative; users who submit duplicate or near-duplicate facts via the API will produce retrieval noise identical to extraction duplicates.

---

## Injection Security and Offline Support

150. ALWAYS gate memory injection input on a maximum of 10,000 tokens of total conversation context per extraction call — reject with `MemoryInputTooLarge` when exceeded; an attacker can craft a long message containing adversarial instructions in the text that will be extracted as a high-confidence fact and injected into future system prompts.

151. ALWAYS assert `retrieved_point.payload["user_id"] == current_user_id` on each retrieved memory point immediately before building the injection string — NEVER rely solely on Qdrant payload filters for isolation; the client-side equality check provides defense-in-depth against misconfigured filter bugs.

152. ALWAYS wrap injected memories in explicit XML delimiters: `<memory source="system">[memory text]</memory>` — NEVER inject raw memory text into the system prompt without delimiters; a memory that contains a conflicting instruction (indirect injection via poisoned memory store) must be visually and structurally distinguishable from authoritative system instructions.

153. ALWAYS support an offline/edge memory mode using SQLite + sqlite-vec for deployments without network access — configure a `pending_sync_queue` table that stores writes made while offline; sync to Qdrant when connectivity resumes using idempotent upserts keyed by `(user_id, text_hash)`.

154. ALWAYS enforce a `max_local_memories = 500` capacity cap on the SQLite offline store — at 6 KB per vector (1536-dim float32), 500 memories consumes ~3 MB, which is within mobile device constraints; evict the lowest-scored memories when the cap is reached.

---

## Observability and Account Lifecycle

155. ALWAYS create a Langfuse span named `memory.retrieval` wrapping every Qdrant search call and record these fields: `memory.query_text` (first 200 chars), `memory.results_count`, `memory.top_score`, `memory.bottom_score`, `memory.latency_ms`, `user_id`, `session_id` — NEVER operate memory retrieval without this observability span.

156. ALWAYS create a Langfuse span named `memory.write` wrapping every Qdrant upsert and record: `memory.operation` (`insert` or `update`), `memory.type`, `memory.confidence`, `memory.source`, `user_id` — NEVER operate memory writes without tracing.

157. ALWAYS implement account deletion as a two-phase process: (1) soft-delete phase — set all user memory points `is_active = false`, revoke the session, and return 200 within 5 seconds; (2) hard-delete phase — schedule a background job to physically delete all Qdrant points, PostgreSQL rows, and Redis keys for the user within 7–30 days; NEVER perform hard deletion synchronously in the request path, as it can take minutes for users with large memory stores.

158. ALWAYS add a `memory_ready` sub-check to the application's `/health/ready` probe — issue a lightweight Qdrant `GET /collections/{name}` call and fail the readiness probe if the response is not `200 OK`; a pod that cannot reach the memory store should not receive production traffic, because it will serve memory-less responses without any client-visible error signal.

159. ALWAYS key the embedding cache by `(embedding_model_version, sha256(text))` only — NEVER include `user_id` in the embedding cache key; embeddings are model-deterministic functions of text content only; adding `user_id` to the key prevents cache hits for identical text across different users and inflates cache size by a factor equal to the user count.
