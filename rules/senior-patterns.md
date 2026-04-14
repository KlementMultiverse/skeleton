---
description: Senior engineering patterns — ADRs, system design, technical debt, LLM cost modeling, code review, event-driven architecture.
paths: ["*.py", "*.md", "docs/**", "*.toml"]
---

# Senior Engineering Patterns — Rules

## Architecture Decision Records (rules 1-10)

1. ALWAYS write an ADR when a technical decision was debated for more than 30 minutes, requires a migration to reverse, or will confuse a future engineer reading the resulting code without context — absence of ADRs is a maintenance liability, not a sign of fast iteration.

2. NEVER edit the Decision section of an accepted ADR — write a new ADR with `Supersedes: <old-id>` instead; mutating history destroys the audit trail and makes it impossible to understand what was decided at what time with what context.

3. ALWAYS include all four required sections in every ADR: Context (why this decision existed), Decision (what was chosen and why), Consequences (both positive and negative), and Alternatives Considered (what was rejected and why) — an ADR with only the decision but no rejected alternatives is not useful.

4. ALWAYS store ADRs in `docs/adr/` with sequential numeric prefixes (`0001-`, `0002-`) and descriptive slugs — never store ADRs inline in code files, wikis that can be edited without review, or in personal notes; ADRs must be version-controlled alongside the code they govern.

5. ALWAYS link the relevant ADR from code that embodies a non-obvious architectural decision — use a comment like `# ADR-0004: refresh tokens used here because long-lived JWTs are not revocable. See docs/adr/0004-...` — without this link, the code and the rationale are disconnected.

6. NEVER delete a superseded ADR — mark it `Status: Superseded by ADR-NNNN` and leave it in place; the historical record of why you made the old decision is often as valuable as the new decision itself, especially during incident postmortems.

7. ALWAYS maintain an `docs/adr/README.md` index table with one-line summaries and statuses for all ADRs — a directory of 20 files with no index requires reading every file to find the relevant decision.

8. DO write ADRs for infrastructure choices (database, cache, message broker, deployment platform), authentication mechanisms, API versioning strategies, and any choice between two legitimate technical options — these are the decisions that compound in cost when reversed.

9. NEVER write ADRs for style conventions, formatting rules, or decisions with one obvious correct answer — ADR overhead is justified only for decisions with meaningful tradeoffs; ADR fatigue from over-documentation reduces the quality of important ones.

10. ALWAYS include a date and author list in every ADR — an undated ADR cannot be evaluated against the constraints that existed at the time it was written; context changes and a 2019 decision looks different from a 2025 decision.

---

## System Design (rules 11-22)

11. ALWAYS clarify functional requirements, non-functional requirements (latency SLO, availability target, durability), and scale constraints before drawing any architecture diagram — designing first and constraining later produces over-engineered solutions that do not match the actual problem.

12. ALWAYS produce a back-of-envelope estimate before committing to an architecture: QPS (requests/day ÷ 86400), storage growth per year, and bandwidth at peak — a feature that seems simple may require sharding at 100M users; discovering this after implementation costs 10x more than designing for it upfront.

13. NEVER choose CP vs AP (CAP theorem) without explicitly stating what failure mode you prefer — returning stale data vs returning an error are both valid choices with different user impact; the default must be documented, not implicit.

14. ALWAYS apply PACELC reasoning to any system with replication: for the normal case (no partition), state the latency vs consistency tradeoff you are accepting — PostgreSQL synchronous replication is EC (consistent but higher latency); async replication is EL (lower latency but brief inconsistency window).

15. ALWAYS design APIs (method, path, request schema, response schema, error codes) before designing the database schema — the API contract defines what data must be retrievable and at what granularity; designing DB first often produces schemas optimized for storage rather than query patterns.

16. NEVER use sequential integer IDs as public API resource identifiers — use UUID v4 or `secrets.token_urlsafe(16)`; sequential IDs enable enumeration attacks, leak record counts, and reveal creation order.

17. ALWAYS design pagination as cursor-based (keyset) for any entity table expected to exceed 100K rows — OFFSET-based pagination performs O(N) full scans at deep pages; cursor-based is O(log N) via index regardless of page depth.

18. DO use read replicas for any read:write ratio greater than 5:1 — route SELECT queries to replicas via a separate SQLAlchemy async engine; monitor replication lag and add a circuit-breaker fallback to the primary if lag exceeds your staleness SLO.

19. ALWAYS choose the Saga pattern over two-phase commit (2PC) for distributed transactions across microservices — 2PC holds locks across a network partition, which is slow and fragile; Sagas trade ACID for BASE with explicit compensating transactions that are faster and partition-tolerant.

20. NEVER treat sharding as the first scaling step — exhaust read replicas, connection pooling, query optimization, and caching first; sharding adds complexity that cannot be easily undone and requires application changes for every query that crosses shard boundaries.

21. ALWAYS include a data model section in every system design that specifies the primary key strategy, indexing plan, and any denormalization decisions — schema choices made at design time are expensive to change in production; a 10-minute review catches 90% of costly mistakes.

22. DO use event sourcing only when you have a concrete requirement for audit history, time-travel queries, or multiple independent projections from the same events — event sourcing adds significant operational complexity (projection rebuilds, snapshot management) that is not justified by CRUD-level requirements.

---

## Technical Debt (rules 23-32)

23. ALWAYS classify technical debt before deciding whether to pay it: deliberate/prudent (known shortcut, tracked), deliberate/reckless (no time for design), inadvertent/prudent (learned a better way), inadvertent/reckless (did not know better) — only deliberate/prudent debt is acceptable; the others require immediate attention or scheduled paydown.

24. ALWAYS create a GitHub Issue for every piece of deliberate technical debt at the time you incur it — label it `tech-debt`, include the business reason for deferring, the estimated effort to fix, and what it blocks in the future; untracked debt is invisible and will never be paid.

25. NEVER begin refactoring a function without first writing characterization tests that document its current behavior (including its bugs) — without these tests, refactoring breaks calling code in ways that are discovered in production rather than in CI.

26. ALWAYS measure the velocity impact of technical debt before making a case for paying it — "this code is messy" is not an engineering argument; "this module takes 3x longer to modify than comparable modules, causing us to ship 2 fewer features per sprint" is an argument that resonates with product teams.

27. DO apply the Boy Scout Rule on every PR that touches existing code: improve one thing about any function you modify — fix a missing type annotation, add a missing error log, rename an opaque variable; the aggregate effect of consistent small improvements is a codebase that stays healthy.

28. NEVER apply the Strangler Fig pattern without deploying the proxy router first (before any migration) — the router is the safety net; if the new system fails, traffic returns to the old system; building the new system before the router removes this safety net.

29. ALWAYS set cyclomatic complexity and cognitive complexity gates in CI — flag functions with cyclomatic complexity > 10 as warnings, > 15 as errors; cognitive complexity > 15 as warnings; these thresholds are strong predictors of bug density.

30. NEVER pay down technical debt during a feature sprint without explicit product agreement — unannounced refactoring that delays a feature creates trust damage that is harder to repair than the technical debt itself; make the tradeoff visible and agreed.

31. DO treat high-coupling modules (imports > 10 distinct non-stdlib modules) as technical debt signals — high coupling means changes ripple unpredictably across the codebase; refactor toward dependency injection and explicit interfaces.

32. ALWAYS document the interest rate of technical debt in the tracking issue: what is the maintenance overhead per sprint, and what features does it block — debt with no stated cost is perpetually deprioritized; debt with a $X/sprint cost has a compelling payback argument.

---

## LLM Cost Modeling (rules 33-45)

33. ALWAYS calculate the estimated monthly LLM cost before launching any AI feature using the formula: `monthly_cost = QPS × 86400 × 30 × avg_tokens × price_per_token` — shipping without a cost model means you cannot price the feature, set a budget, or detect when costs are anomalous.

34. ALWAYS use `Decimal`, not `float`, for all LLM cost calculations — floating-point accumulation errors are significant at the scale of millions of requests; `Decimal("0.000003") × Decimal(input_tokens)` is exact.

35. NEVER hardcode LLM pricing in business logic — store prices in a config dict keyed by exact model string; model prices change; a configuration dict is updated without touching business logic.

36. ALWAYS check the LLM budget BEFORE making an API call, never after — the tokens are spent at call time; a post-call budget check is advisory at best.

37. ALWAYS store per-user and per-tenant LLM budgets in Redis using `INCRBYFLOAT` with a TTL set to cover the full billing period — without TTL, budget keys accumulate indefinitely; without Redis atomicity, concurrent requests can both pass the budget check and together exceed the limit.

38. ALWAYS log `cache_read_input_tokens` and `cache_creation_input_tokens` separately from `input_tokens` — Anthropic charges cache_creation at 1.25x base rate and cache_read at 0.10x base rate; conflating them into a single token count produces incorrect cost accounting and hides whether prompt caching is working.

39. ALWAYS add `cache_control: {"type": "ephemeral"}` to any prompt prefix exceeding 1024 tokens that is reused across requests — the system prompt is the most common candidate; without the cache breakpoint, every request pays full input token cost for static content.

40. NEVER place the `cache_control` breakpoint inside the dynamic portion of a prompt — the cache key is the exact byte sequence before the breakpoint; any change to the dynamic content before the breakpoint invalidates the cache.

41. ALWAYS route classification, extraction, tagging, and short summarization tasks to the cheapest available model tier (Haiku or equivalent) — the cost ratio between model tiers is 1:4:20 for Haiku:Sonnet:Opus; task difficulty rarely requires escalation, and the cost impact is immediate.

42. NEVER escalate to a more expensive model based on user-claimed complexity — users always describe their task as complex; route based on measured signals: input token count, task type enum, output format requirements.

43. ALWAYS implement a semantic response cache for deterministic LLM tasks — note summarization, entity extraction, classification on fixed inputs; cache by SHA-256 of the full prompt; set TTL to 24 hours for time-sensitive content, longer for static content; a cache hit at zero LLM cost is always preferable.

44. ALWAYS compute and log ROI for AI features: `roi_ratio = value_per_user_per_month / cost_per_user_per_month` — features with ROI > 10x are defensible; features with ROI < 2x need redesign or model downgrade before launch.

45. ALWAYS set a per-request `max_tokens` ceiling — an unset `max_tokens` allows the model to generate up to its context window; a single malformed request with no ceiling can generate 100K tokens and cost $1.50+ on Sonnet; at scale, missing `max_tokens` is a DoS vector against your own budget.

---

## Code Review (rules 46-55)

46. ALWAYS review along all 7 dimensions before approving: correctness, edge cases (null/empty/overflow/concurrent), performance at 10x load, security (injection/ownership/privilege), maintainability (naming/structure/coupling), testability (pure functions, injected dependencies), and observability (logging/metrics on failure paths).

47. ALWAYS label every review comment with its priority: `[blocking]` for correctness/security/serious-performance issues that must be fixed before merge; `[suggest]` for important improvements that are not merge blockers; `[nit]` for style preferences; `[question]` for genuine questions; `[praise]` for explicit positive reinforcement.

48. NEVER approve a PR that lacks tests for the new behavior — a PR without tests is incomplete, not "mostly done"; the test is the specification; merging without tests makes the next change dangerous.

49. ALWAYS check for N+1 queries when reviewing endpoints that return lists — look for relationship accesses inside loops, lazy-loaded ORM relationships accessed after a query, and missing `selectinload`/`joinedload` options.

50. ALWAYS verify ownership checks on every non-GET endpoint — confirm that the handler fetches by `(id, owner_id)` or equivalent, not just by `id`; post-fetch ownership checks in Python have already performed the unauthorized read.

51. NEVER write a review comment that is personal rather than technical — "This is messy" is personal; "The function has 4 single-letter parameters which makes it hard to review for correctness" is technical; personal feedback creates defensiveness; technical feedback creates learning.

52. ALWAYS review LLM-generated code with extra skepticism on these specific failure modes: hallucinated library methods (verify every API call against current docs), silent edge case failures (empty list, None, zero), security checks copied from insecure examples, and missing error handling on external calls.

53. DO use the approval-with-conditions pattern for PRs that are structurally correct but need one small fix: "Approved pending: change `except Exception: return False` on line 47 to log the exception before returning" — this unblocks the author without requiring a second review cycle for a minor fix.

54. NEVER skip architecture review for PRs that introduce new database tables, new external service dependencies, new authentication flows, or changes to core domain models — line-by-line review is insufficient for these; the entire design needs evaluation before implementation details.

55. ALWAYS verify that every new public function has: type annotations on all parameters and return type, a docstring for non-obvious behavior, and at least one test — functions without these are incomplete, regardless of whether the feature "works".

---

## Event-Driven Architecture (rules 56-70)

56. ALWAYS use the outbox pattern for any system where both a database write AND an event publication must succeed atomically — write the event to an `outbox_events` table in the same database transaction as the state change; a separate polling process publishes from the outbox; this eliminates the dual-write problem.

57. NEVER poll the outbox table with `SELECT ... WHERE published_at IS NULL` without `FOR UPDATE SKIP LOCKED` — without `SKIP LOCKED`, multiple workers contend on the same rows and block each other; `SKIP LOCKED` makes workers take non-overlapping subsets of pending events.

58. ALWAYS design event consumers to be idempotent — at-least-once delivery is the default guarantee of every durable message system; duplicate event processing must produce the same result as single processing; use a `SET NX` deduplification key in Redis keyed by `event_id` with TTL equal to your maximum retry window.

59. NEVER use PostgreSQL LISTEN/NOTIFY for events that must survive a subscriber restart — NOTIFY is fire-and-forget; if the subscriber is down, the notification is lost; use LISTEN/NOTIFY only for cache-invalidation signals and real-time UI pushes where loss is acceptable.

60. ALWAYS include `event_id` (UUID v4), `event_type` (string), `occurred_at` (UTC ISO timestamp), and `schema_version` (integer) in every event payload — `event_id` enables deduplication; `schema_version` enables consumers to handle schema evolution; `occurred_at` enables event ordering and replay.

61. NEVER allow consumers to mutate the event payload — events are immutable records of what happened; if business logic requires enrichment, enrich at consumption time and store the enriched form separately; mutating events corrupts the audit log.

62. ALWAYS set a Dead Letter Queue (DLQ) for any message queue used in production — after N failed processing attempts, messages must land in the DLQ rather than being silently dropped; unprocessed business events (order created, payment received) that disappear without trace are a data integrity failure.

63. NEVER retry a failed message without exponential backoff — immediate retry on transient failure hammers the dependency that is already stressed; use base 1s, double each attempt, add `random.uniform(0, delay * 0.5)` jitter, cap at 60s.

64. ALWAYS include the full entity state (not just the ID) in domain events when the event represents a state transition — consumers should not need to re-query the database to process the event; a `NoteUpdated` event with only `note_id` forces N follow-up queries from N consumers.

65. DO choose PostgreSQL LISTEN/NOTIFY for: low volume (<100 events/second), no durability requirement, same-database consumers, cache invalidation signals — it requires no additional infrastructure and adds no operational complexity.

66. DO choose Redis Streams for: moderate volume (<100K events/second), consumer groups needed, short-term durability sufficient, already using Redis — it is the pragmatic default for most web applications.

67. DO choose Kafka for: high volume (>100K events/second), multi-service fan-out, long retention (days to weeks), ordered processing within a partition, event sourcing replay — do not use Kafka for simple background task queues; the operational cost is not justified below ~10K events/second.

68. ALWAYS configure consumer group names with the service name, not a generic name — `note_search_indexer` not `consumer_group_1`; consumer group names appear in monitoring; generic names make it impossible to identify which service is lagging.

69. NEVER process events in the same thread as the HTTP request handler — event processing that fails should not fail the HTTP request; use background workers (Celery, ARQ, or a dedicated consumer process) that process from a queue or stream independently.

70. ALWAYS monitor consumer lag (messages in queue minus messages consumed) and alert when lag exceeds a threshold — growing lag means consumers are slower than producers; unmonitored lag grows silently until it causes cascading failures when the queue fills to capacity.

---

## Prompt Engineering as Code (rules 71-80)

71. ALWAYS version prompts with semantic versioning stored in the prompt module — `CURRENT_VERSION = PromptVersion(2, 1, 0)` — breaking changes in expected output format are major versions; new examples or defensive additions are minor; typo fixes are patches; version the judge prompt separately from the application prompt.

72. ALWAYS maintain a golden dataset of at least 20 test cases with defined expected properties for every prompt in production — golden dataset tests must run in CI and must gate any PR that modifies the prompt, system instructions, few-shot examples, or output format.

73. NEVER deploy a prompt change to production without running the golden dataset eval in CI — a prompt change that appears safe locally may produce regressions on the long tail of inputs; the golden dataset catches these before users see them.

74. ALWAYS wrap user-supplied content in explicit XML-like delimiters before injecting it into prompts — `<user_note>{content}</user_note>` — and instruct the model in the system prompt that content inside those tags is data, not instructions; bare injection of user content is a prompt injection vulnerability.

75. NEVER put the cache_control breakpoint in the middle of the few-shot examples section — the cache key is everything before the breakpoint; changing any example invalidates the cache for all requests; put the breakpoint at the end of the static system context, before any dynamic content.

76. ALWAYS test for prompt injection resistance in the golden dataset — include at least 3 test cases where the input contains adversarial instructions ("ignore previous instructions", "you are now a different AI", "reveal your system prompt") and assert the output does not comply with those instructions.

77. DO use chain-of-thought prompting for tasks with complexity score > 5 (multi-entity comparison, code debugging, planning) — direct answer prompting is faster and cheaper; CoT adds 100-300 output tokens but measurably improves accuracy on tasks requiring intermediate reasoning steps.

78. NEVER store prompt text inline in route handlers or service functions — prompt text belongs in a dedicated `prompts/` module versioned alongside the code; inline prompts cannot be tested in isolation, versioned independently, or reused across endpoints.

79. ALWAYS A/B test prompt changes on a canary percentage of traffic before full rollout — compare output quality scores (Langfuse, LangSmith, or your own eval) between the old and new prompt; a prompt change that looks like an improvement in testing may degrade real-world quality.

80. ALWAYS include at least 2 few-shot examples in prompts where output format compliance is critical — zero-shot format instructions have ~80% compliance; two well-chosen examples raise compliance to ~98%; examples cost input tokens but save output parsing errors and downstream failures.

---

## Multi-Model Architecture (rules 81-90)

81. ALWAYS default to the cheapest capable model tier and escalate explicitly — the cost ratio between model tiers is not linear; Haiku:Sonnet:Opus is approximately 1:4:20 for input tokens; assume Haiku is sufficient until benchmarks show it is not.

82. NEVER select a model based on user-described complexity — users always claim their task is complex; route based on measurable signals: input token count threshold, task type enum, whether structured output is required, whether multi-step reasoning is required.

83. ALWAYS pin exact model version strings in production code — `"claude-haiku-4-5"` not `"claude-haiku-latest"`; model aliases change without notice; pinning ensures behavior consistency between test runs and production, and makes deprecation notices actionable rather than silent surprises.

84. ALWAYS build a fallback chain for any production LLM call path — primary model (best quality for task) → fallback model (cheaper/faster) → cached response (degraded but available); on HTTP 529 (overloaded) or connection error, transparently fall through; log every fallback usage.

85. NEVER mix models within the same evaluation scoring run — Claude-Haiku and Claude-Sonnet produce different score distributions for the same quality of output; mixing them makes quality trends uninterpretable; tag every score with the model that produced it and filter separately.

86. ALWAYS manage context windows explicitly in multi-turn conversations — calculate available token budget as `(model_context_limit × 0.80) - system_prompt_tokens - max_output_tokens`; trim oldest message pairs when history approaches the limit; log every eviction; never silently truncate.

87. DO use parallel model calls for consensus on high-stakes decisions — call a primary model and a cheaper validator model concurrently; the validator checks for factual errors, safety issues, or significant omissions; the cost overhead (validator call) is justified when errors have meaningful downstream consequences.

88. NEVER fine-tune a model when RAG can satisfy the requirement — fine-tuning is appropriate for: consistent output style that RAG cannot enforce, specialized vocabulary not in base training, sub-50ms latency requirements that preclude retrieval; for knowledge retrieval and factual questions, RAG is always preferred (fresher, attributable, cheaper to update).

89. ALWAYS use `asyncio.Semaphore` to cap concurrent LLM API calls — without a semaphore, `asyncio.gather` on a large list fires all coroutines simultaneously, exhausting your API rate limit and triggering 429s; set the semaphore based on your tier's RPM: `Semaphore(RPM_LIMIT // 60 * max_expected_latency_seconds)`.

90. ALWAYS track the output/input token ratio per endpoint and alert when it exceeds 3.0 — a ratio above 3.0 indicates the model is generating verbose output that exceeds instructions, or is stuck in a repetition loop; high ratios inflate cost and latency and are a quality signal that the prompt needs a length constraint.

---

## ADR and RFC Process

91. ALWAYS use an RFC (Request for Comments) for decisions that require broad team input before any implementation — cross-team API contracts, org-wide conventions, public API surface changes, or deprecation of shared infrastructure; use an ADR to record the outcome of a decision already made. RFC is a proposal seeking consensus; ADR is a record of a concluded decision.

92. ALWAYS structure an RFC with: Problem Statement, Proposed Solution, Alternatives Rejected (with reasons), Open Questions (with owners and due dates), and a Decision Deadline — close the RFC by converting the chosen option into an ADR; an RFC that never closes and never becomes an ADR is a discussion thread, not governance.

---

## Architecture Fitness Functions

93. ALWAYS implement architecture fitness functions as automated tests in CI that verify structural constraints — examples: `assert import_depth("src/api") <= 3`, `assert no_datetime_naive_calls()`, `assert all_db_access_through_repositories()`. Fitness functions turn architecture rules into verifiable facts rather than aspirational guidelines.

94. ALWAYS assign each fitness function a severity (warning or blocking) and a team owner — a fitness function with no owner degrades silently when failing; a blocking one with no owner stops all deploys with no one to fix it.

95. DO define fitness functions for: maximum cyclomatic complexity per module, maximum fan-in/fan-out per layer, absence of direct DB access from non-repository files, absence of cross-bounded-context direct imports, and absence of hardcoded config values — these five cover the most common architecture erosion patterns.

96. NEVER treat an architecture diagram as a contract — treat it as a hypothesis; attach a fitness function to every architectural constraint you care about; a constraint enforced only by convention will be violated within 6 months.

---

## Strangler Fig Pattern

97. ALWAYS implement the Strangler Fig in three explicit phases: (1) deploy a proxy/facade routing 100% to the old system with feature flags for selective rerouting; (2) build the new system behind the flag and route a canary percentage; (3) flip to 100% new traffic and decommission only after a defined stability window — skipping to phase 2 without the proxy eliminates the ability to roll back incrementally.

98. ALWAYS instrument the Strangler Fig proxy to emit metrics for each routing decision (`old_system_hits`, `new_system_hits`, `new_system_errors`) — without these metrics you cannot verify the canary percentage, detect elevated error rates, or make a data-driven decision to proceed or roll back.

---

## Bounded Context Mapping

99. ALWAYS draw an explicit bounded context map before designing microservice or module boundaries — identify the relationship type between each pair of contexts: Partnership, Customer-Supplier, Conformist, Anti-Corruption Layer, Open-Host Service, or Published Language; the relationship type determines integration pattern and team coordination cost.

100. NEVER share a database table between two bounded contexts — shared tables create hidden coupling; a schema change in one context silently breaks another; if data must flow between contexts, use events or explicit API calls.

101. ALWAYS implement an Anti-Corruption Layer (ACL) when integrating with a legacy system or external vendor whose domain model does not match yours — the ACL translates between models at the boundary; without it, the legacy model leaks into your domain and makes your codebase conform to an external constraint you cannot control.

---

## C4 Model Diagrams

102. ALWAYS produce C4 Level 1 (System Context) and Level 2 (Container) diagrams for any system with more than two services or external integrations — these two diagrams answer 90% of onboarding questions without reading code; store them as PlantUML or Structurizr DSL in `docs/architecture/` so they are version-controlled and diffable.

103. NEVER use a generic box-and-arrow diagram where C4 is appropriate — C4 enforces that every element has a Name, Technology, and Description, and every relationship has a label; unlabeled arrows make diagrams ambiguous and unactionable.

---

## Latency Numbers Every Engineer Should Know

104. ALWAYS use these canonical latency orders-of-magnitude in back-of-envelope estimates: L1 cache 0.5ns, L2 cache 7ns, main memory 100ns, SSD random read 100µs, HDD seek 10ms, same-datacenter network 0.5ms, cross-region network 150ms — any design requiring sub-millisecond latency for a cross-datacenter synchronous call violates physics.

105. ALWAYS derive these practical rules from the latency table: a Redis lookup costs ~0.5ms; a PostgreSQL query hitting disk ~10ms; a cross-region synchronous call ~150ms; a chain of 5 cross-region synchronous calls has a minimum floor of 750ms regardless of code quality — use these numbers to identify call chains that must become asynchronous.

---

## Multi-Region Architecture

106. ALWAYS make an explicit documented choice between Active-Active and Active-Passive multi-region topology before designing replication — Active-Active allows writes in both regions (requires conflict resolution), Active-Passive routes all writes to one region (simpler consistency, higher write latency from the passive region); the choice propagates into every aspect of the data layer.

107. NEVER assume strong consistency is achievable across regions at acceptable latency — the speed-of-light round trip between US-East and EU-West is ~85ms each way; synchronous replication adds this to every write; design for eventual consistency with a defined staleness SLO and make that SLO visible to the product team before they commit to user-facing requirements.

108. ALWAYS implement a conflict resolution strategy before going Active-Active multi-region — the three standard strategies are: Last-Write-Wins (by logical timestamp), operational transformation (for concurrent document edits), and application-level merge (custom domain logic); LWW loses updates silently; document which strategy applies to each entity type.

---

## Hot Partition / Hot Shard

109. ALWAYS anticipate hot partitions when the partition key is a predictable high-cardinality attribute accessed non-uniformly — a Kafka topic partitioned by `user_id` where one user generates 10,000x average will saturate one partition; a DynamoDB table partitioned by `tenant_id` where one tenant is 1,000x larger will breach per-partition throughput limits regardless of total provisioned capacity.

110. ALWAYS apply one of these four mitigations for a confirmed hot partition: (1) add random suffix salt to the key and scatter-gather on reads; (2) use write sharding with a virtual shard layer; (3) use a time-based partition prefix; (4) move the hot entity to a dedicated partition — document which mitigation applies and its scatter-gather read overhead in an ADR.

111. NEVER treat hot partition problems as solvable purely by increasing capacity — adding capacity raises the ceiling but does not change the distribution; a single viral user will always hit the per-partition limit before the aggregate limit; the mitigation must change the key distribution.


---

## Technical Debt — Tooling and Extraction

112. ALWAYS run `radon cc -s -a` (cyclomatic complexity) and `radon mi -s` (maintainability index) as part of CI reporting and track trends over time — a single snapshot is informative; a rising trend over 4 sprints justifies a scheduled paydown sprint before the slope steepens further.

113. ALWAYS apply the Rule of Three before extracting an abstraction: the first time you write something write it inline; the second time note the duplication; the third time extract — premature abstraction is as damaging as duplication because it locks in the wrong interface before the shape of the variation is known.

114. ALWAYS use `cognitive_complexity` (measured by `flake8-cognitive-complexity` or SonarQube) as the primary refactoring priority signal over cyclomatic complexity — cognitive complexity measures how hard a function is to understand; a function with cognitive complexity > 15 takes measurably longer to review and modify correctly.

---

## LLM Cost Modeling — Pricing and Break-Even

115. ALWAYS use versioned reference prices (Q1 2025, per 1M tokens) when building cost models: Claude Haiku 3.5: $0.80 input / $4.00 output; Claude Sonnet 3.7: $3.00 input / $15.00 output; Claude Opus 3: $15.00 input / $75.00 output — store in a config dict keyed by model string with a `price_valid_from` date; recheck quarterly as prices change.

116. ALWAYS perform a break-even analysis before implementing an LLM response cache using: `break_even_hits = cache_monthly_cost / (avg_tokens × price_per_token)` — a Redis semantic cache at ~$100/month on Sonnet pricing requires ~33K cache hits/month to break even; below that volume the cache costs more than it saves.

117. NEVER implement an LLM caching layer without first measuring empirical cache hit rate on production traffic for at least 48 hours — theoretical hit rates based on assumed query distribution are routinely 2-5x higher than actual; deploy the key-computation logic without the cache first to measure real hit rate before provisioning infrastructure.

118. ALWAYS implement LLM cost chargeback to tenants using a usage ledger table with columns `(tenant_id, model, input_tokens, output_tokens, cache_read_tokens, cost_decimal, request_id, created_at)` — derive invoices from this table, never from aggregated Redis counters which are approximate and have no audit trail.

119. ALWAYS expose per-tenant cost dashboards showing current-period spend, projected end-of-month spend, and breakdown by model tier — tenants cannot optimize their usage without visibility; a surprise invoice at month-end generates more churn than the cost of the transparency tooling.

---

## Code Review — Anti-Patterns and Process

120. NEVER rubber-stamp a PR by approving without reading it — a review that takes under 2 minutes for a 200-line PR is social permission-granting, not a review; rubber-stamping degrades team code quality faster than having no review process because it creates false confidence that issues were caught.

121. NEVER allow nitpicking culture to dominate code review — if more than 40% of comments are `[nit]` style preferences rather than correctness or design issues, redirect nit-level feedback to automated linters; human review cycles should focus on what automation cannot catch.

122. ALWAYS weigh pair programming against async review for high-risk changes — pair programming catches design issues in real-time and eliminates review latency, but costs 2x developer time; use pair programming for: new architectural patterns, security-sensitive flows, complex algorithms, and onboarding; use async review for routine feature work.

123. ALWAYS check these security items explicitly during code review (not caught by static analysis): (1) every new endpoint has an authorization check before first data access; (2) every new query filters by `owner_id` or `tenant_id` in the WHERE clause; (3) every new external HTTP call has connect and read timeouts set; (4) every new file path derived from user input is validated with `os.path.realpath()`.

124. ALWAYS use the two-pass review technique for PRs larger than 300 lines: first pass reads only file names, function signatures, and test names to understand design intent; second pass reads implementation details — reviewers who skip the first pass approve implementation details that implement the wrong design.


---

## Event-Driven Architecture — Schema Evolution and Delivery

125. ALWAYS store processed `event_id` in a deduplication table or Redis SET with TTL before executing side effects — the TTL must equal at least your maximum message retention window plus your maximum retry window; an expired deduplication key means a redelivered old message will be reprocessed.

126. ALWAYS design event schemas for backward compatibility: add fields as optional with defaults, never remove or rename existing fields without a major version bump — consumers on an older schema must deserialize a newer message; break this contract and you require coordinated multi-service deploys.

127. ALWAYS use a schema registry (Confluent Schema Registry for Avro/Protobuf, or JSON Schema with central validation) for any event bus shared across more than two services — ad-hoc JSON with no registry produces schema drift within months.

128. ALWAYS use this decision tree for failed messages: retry (transient failure) → DLQ after N retries (persistent failure, needs inspection) → discard (known-invalid, deprecated producer) — never silently discard a message that could represent a business event without logging it to a structured audit log first.

129. ALWAYS prepare for Kafka consumer group rebalancing by committing offsets only after processing is complete (`enable.auto.commit=false`) and implementing `on_partitions_revoked` to flush in-flight work before partition ownership transfers — a rebalance during auto-commit will reprocess every message since the last 5-second auto-commit interval.

130. ALWAYS monitor Kafka consumer lag per partition (not just per topic) and alert when any single partition's lag exceeds your latency SLO — a lag of 10,000 messages at 1,000 messages/second means a 10-second processing delay; alert at 50% and 100% of your SLO threshold.

131. ALWAYS implement backpressure in async event consumers using a bounded in-memory queue between the network receiver and processing workers — an unbounded queue absorbs traffic spikes into memory until the process OOMs; a bounded queue correctly pauses broker consumption when the pipeline is saturated.

132. NEVER block the event loop inside an async Kafka or Redis Streams consumer coroutine — deserialization, database writes, and external HTTP calls must all be awaited or offloaded with `run_in_executor`; a single blocking call stalls all partition processing for that consumer instance.

---

## Prompt Engineering — Templating, Budgets, and Advanced Patterns

133. ALWAYS use a structured templating library (Jinja2 or equivalent) rather than f-strings for prompts longer than three lines — Jinja2 enables template inheritance, conditional blocks, loop-rendered few-shot examples, and explicit variable escaping; f-strings inline into Python logic making the prompt untestable in isolation and prone to injection via unescaped braces.

134. ALWAYS budget prompt token allocation before writing content using this baseline ratio: system prompt ≤ 25% of context window, few-shot examples ≤ 20%, user/retrieved content ≤ 40%, response buffer ≥ 15% — violating the response buffer causes `stop_reason: max_tokens` truncations.

135. ALWAYS structure chain-of-thought prompts with a reasoning step separator — instruct the model to output `<thinking>...</thinking>` so the reasoning trace is parseable and loggable separately from the final answer; this enables scoring reasoning quality independently of answer quality.

136. ALWAYS implement self-consistency by sampling 3-5 responses at temperature 0.5-0.8 and taking majority vote — reserve this for tasks where accuracy is critical and latency is not; log individual answers and the winning vote for debugging and eval purposes.

137. ALWAYS implement the ReAct (Reason + Act) loop as: `Thought` → `Action` → `Observation` → repeat until `Final Answer` — ReAct makes the model's decision path auditable at each step and enables step-level evaluation and human approval gates.

138. ALWAYS sanitize tool output before injecting it back into the prompt — strip prompt-control characters, apply XML delimiter wrapping (`<tool_result tool="name">...</tool_result>`), and truncate to a maximum token budget per tool result; tool outputs can carry indirect prompt injection payloads.

139. NEVER inject RAG retrieved document content into the system prompt position — place it in the human turn delimited by `<retrieved_document source="...">` tags; system prompt position grants elevated authority in the model's attention; a poisoned document there can override explicit instructions.

140. ALWAYS set `temperature=0.0` for tasks requiring deterministic reproducible output (classification, extraction, structured data generation) and document this in the prompt version metadata — a prompt tested at temperature 0 but deployed without specifying temperature runs at the API default (often 1.0), producing non-deterministic outputs that break parsers.


---

## Model Deprecation and Version Management

141. ALWAYS maintain a model deprecation registry (`docs/model-registry.yaml`) with every pinned model string, its EOL date from the provider's deprecation calendar, the intended replacement, and the owner responsible for migration — provider deprecation notices arrive weeks before the cutoff, not months; an untracked EOL is a silent production breakage waiting to happen.

142. ALWAYS implement a model sunset detection job that runs weekly in CI and pages the owning team when an EOL date is fewer than 60 days away — a 60-day runway allows shadow-testing the replacement on production traffic samples, running golden dataset evals, and deploying before forced cutover; 7 days is a fire drill.

143. NEVER migrate from a sunset model by a direct string replacement — always run the replacement in shadow mode (call both models, log both outputs, serve only the old model's response) for at least 1 week of production traffic before flipping; shadow mode reveals output distribution changes that golden datasets miss.

144. ALWAYS design model version configuration as a runtime environment variable, not a deploy-time constant — `MODEL_NOTE_SUMMARIZER="claude-haiku-4-5"` in env config; when a model is deprecated you need to swap without a redeploy; hardcoded model strings require a code change, review, and deploy cycle under emergency conditions.

145. ALWAYS run the replacement model against your full golden dataset AND a 1,000-sample random draw from recent production traces before promoting — golden datasets cover happy paths; production samples cover the long tail; a model scoring 0.94 on golden and 0.71 on production has a distribution mismatch that will surface as user complaints.

---

## LLM Provider Abstraction

146. ALWAYS wrap all LLM provider API calls behind a single `LLMClient` interface with a fixed method signature (`complete(messages, model, max_tokens, temperature, stream) → LLMResponse`) — raw SDK calls scattered across service files means migrating providers requires N file changes; an interface means one file changes.

147. NEVER import a provider-specific SDK (`anthropic`, `openai`, `google.generativeai`) outside the LLM adapter module — the adapter module is the only file that knows which provider is active; application code imports only from your internal `llm_client` module.

148. ALWAYS use LiteLLM or an equivalent abstraction when your system must support two or more LLM providers simultaneously — LiteLLM provides a unified OpenAI-compatible interface across Anthropic, OpenAI, Cohere, Mistral, and local models; it eliminates the maintenance burden of hand-rolling adapters for each provider's streaming, error codes, and token counting differences.

149. ALWAYS normalize provider error codes into your own error taxonomy before they reach application code — Anthropic returns HTTP 529 for overload, OpenAI returns HTTP 429 with subtypes; without normalization, error handling branches on provider-specific strings making provider swap a correctness risk.

---

## Multi-Provider Rate Limit Management

150. ALWAYS maintain separate `asyncio.Semaphore` instances per provider per model tier — sharing a single semaphore across providers unnecessarily throttles the healthy provider when the other is at capacity; sharing within a provider but across tiers wastes Haiku headroom when Sonnet is saturated.

151. ALWAYS implement per-provider token-per-minute (TPM) budgets as a sliding window in Redis — check token count via `ZRANGEBYSCORE now-60s now` before each call; TPM limits are the primary rate limit vector for large-context requests whereas RPM limits dominate for short requests; both must be tracked independently.

152. ALWAYS implement provider failover: when Provider A returns 429 or 529, route the next N requests to Provider B for a configurable cooldown period, then probe Provider A with a single request before restoring full routing — binary failover without probing causes thundering-herd re-entry when the cooldown expires.

---

## Streaming vs Batch Inference

153. ALWAYS use streaming (`stream=True`) for any LLM response where the user is waiting synchronously for output exceeding 500 tokens — without streaming, TTFB equals full generation time (10-40 seconds); with streaming, TTFB is ~200-800ms; this is the primary driver of perceived latency and user abandonment.

154. NEVER use streaming for LLM calls in background jobs, batch pipelines, or classification tasks — streaming adds connection hold complexity, partial-read error handling, and reassembly overhead; for non-interactive paths, a single blocking call is simpler, cheaper, and easier to retry atomically.

155. ALWAYS use batch inference APIs (Anthropic Batch API, OpenAI batch) for workloads where latency is not user-facing — 50% cost reduction at 24-hour turnaround; suitable: dataset annotation, bulk summarization, overnight embedding generation; unsuitable: any user-interactive path.

156. ALWAYS set a per-stream timeout in addition to per-request timeout — a stream that starts but stalls holds an open connection indefinitely; use `asyncio.timeout(stream_max_seconds)` wrapping the async generator; set `stream_max_seconds = (max_output_tokens / avg_tokens_per_second) × 2.5` as a heuristic.

---

## Context Window Across Providers

157. ALWAYS store context window limits in your model registry alongside pricing and EOL dates — never hardcode limits inline; providers silently extend context windows and stale hardcoded limits cause valid inputs to be silently rejected.

158. NEVER assume a prompt that fits in one provider's context window fits in another's — a 150K-token prompt runs on Claude Sonnet (200K) but fails on GPT-4o (128K); when building provider-agnostic fallback chains, calculate token count first and only route to providers with `context_tokens` exceeding it by at least 15%.

---

## SLOs, SLAs, and Error Budgets

159. ALWAYS define SLOs as internal engineering targets and SLAs as contractual commitments to customers — SLAs must be set at least 10 percentage points below your achieved SLO so normal SLO misses do not trigger SLA penalties; never set an SLA higher than your demonstrated SLO.

160. ALWAYS calculate error budget as `(1 - SLO_percentage) × window_duration_minutes` — a 99.5% SLO over 28 days gives 201.6 minutes of allowed bad events; when the budget is consumed, all non-reliability work stops until the next window.

161. ALWAYS implement burn rate alerting with two windows: fast burn (1-hour window consuming > 5% of 28-day budget → page immediately) and slow burn (6-hour window consuming > 10% of budget → ticket with 4-hour response) — a single static threshold misses gradual degradation that will exhaust the budget before month-end.

162. NEVER retroactively adjust an SLO window to remove an incident — SLO data integrity is the foundation of error budget accountability; if caused by a dependency you don't control, annotate it separately but do not remove it from your SLO calculation.

---

## On-Call for AI-Native Backends

163. ALWAYS maintain a separate on-call runbook for LLM-specific failure modes — LLM failures include: provider outage, model deprecation, cost anomaly, quality regression, and prompt injection attempt; these require different triage steps than a database outage or memory leak.

164. ALWAYS define explicit LLM degraded-mode behaviors in the runbook for each AI-powered feature: (1) fallback to a cheaper model, (2) return a cached response, (3) disable the AI feature and return a static response, (4) queue for async fulfillment — the on-call engineer must know which tier to activate without making a judgment call under pressure.

165. ALWAYS include a cost anomaly runbook section specifying: alert threshold (cost_per_hour exceeds 3x the 7-day p95), first action (identify which endpoint/user/tenant is driving the spike via the usage ledger), immediate remediation (per-user budget cap or kill-switch), and post-incident action — LLM cost spikes can reach $1,000/hour within minutes.

---

## Chaos Engineering

166. ALWAYS start chaos engineering with a defined steady state (`p99 latency < 2s, error rate < 0.1%`) before injecting any failure — chaos experiments are hypothesis-driven; if you do not define steady state first, the experiment has no success criterion.

167. ALWAYS start with minimum blast radius (1% of traffic, single instance) and expand only after confirming recovery — incremental blast radius lets you confirm containment before widening scope.

168. ALWAYS inject these LLM-specific failure modes in game days: provider HTTP 429, provider HTTP 529, provider timeout after streaming starts (partial response), malformed JSON in structured output response, and a 10x cost anomaly via synthetic high-token response — these are failure modes that generic chaos tools (Chaos Monkey, Gremlin) do not cover.

169. NEVER perform chaos experiments without an abort condition and a designated experiment owner with authority to halt — the abort condition must be defined before the experiment starts; an experiment without an abort condition can escalate from controlled injection to a real incident.

---

## Runbook Structure

170. ALWAYS structure every production runbook with these seven sections: (1) Alert context, (2) Immediate triage (first 5 minutes), (3) Diagnostic commands (exact copy-paste with expected output), (4) Remediation steps (ordered, numbered, with rollback after each), (5) Escalation path (who to call if step N fails), (6) Post-incident actions, (7) Related runbooks — a runbook missing any section will be incomplete under incident pressure.

171. ALWAYS include exact copy-paste diagnostic commands in runbooks with expected output examples — a runbook that says "check the database connections" without the command forces the on-call engineer to recall syntax at 3am.

172. ALWAYS test every runbook quarterly by having someone who did not write it execute it against staging while being timed — runbooks written by the system author skip steps that feel obvious to them but are not obvious to the on-call engineer; quarterly testing surfaces gaps before they matter.


---

## Feature Flag Lifecycle

173. ALWAYS assign every feature flag a lifecycle stage at creation — `(experiment | rollout | operational | kill_switch)` — and record it in a central registry (`docs/feature-flags.yaml`) with owner, creation date, and planned removal date; a flag with no lifecycle stage is never cleaned up and becomes permanent configuration debt.

174. ALWAYS follow a four-phase rollout sequence: (1) internal employees only; (2) canary at 1-5% with error-rate monitoring gate; (3) graduated rollout in increments (10% → 25% → 50% → 100%) with minimum soak time between each step; (4) full rollout followed by flag removal from code within one sprint — skipping soak time means an error spike at 50% has already affected half your users before you can roll back.

175. NEVER leave a fully-rolled-out flag in production for more than one sprint after reaching 100% — every live flag is a conditional branch evaluated on every request and adds cognitive load to every code reader; create a cleanup issue at the moment you hit 100%.

176. ALWAYS implement kill-switch flags separately from rollout flags — a kill-switch is a permanent operational control (disable LLM calls under cost spike) and must never be removed; a rollout flag is temporary and must always be removed; mixing them produces flags that look permanent when they should be deleted.

---

## Dark Launches vs Canary Releases

177. ALWAYS distinguish dark launches from canary releases: a dark launch executes the new code path for real traffic but discards the new response (serving only the old response to users), logging outputs for comparison; a canary release serves the new response to a subset of real users — dark launches validate correctness and performance without any user impact; canaries validate the full user experience.

178. ALWAYS run a dark launch before a canary for any change to a core business-logic path (pricing, access control, payment processing) — if dark launch output diverges from the current path on more than 0.5% of requests, stop and investigate before exposing the new path to any users.

---

## Tech Radar

179. ALWAYS maintain a team tech radar (`docs/tech-radar.md`) with four rings — Adopt, Trial, Assess, Hold — and review quarterly; every new dependency proposed in a PR must be placed on the radar before approval.

180. NEVER adopt a technology on Hold without a written exception signed by the tech lead with a time-bounded re-evaluation date — Hold means the technology has a known problem (security, instability, licensing); bypassing it without documentation erodes the radar's authority and makes it decorative.

---

## Dependency Injection vs Service Locator

181. ALWAYS prefer constructor injection (passing dependencies as parameters) over the Service Locator pattern (calling a global registry to retrieve dependencies) — constructor injection makes dependencies explicit in the function signature enabling straightforward testing; the Service Locator hides dependencies requiring knowledge of the registry's internal state for isolation.

182. NEVER implement a module-level singleton that is both constructed and used inside the same import — this couples every caller to the concrete implementation and makes unit testing require monkey-patching; inject the dependency from the composition root downward.

---

## API Versioning

183. ALWAYS default to URL path versioning (`/api/v1/`, `/api/v2/`) for public-facing APIs — it is the most debuggable strategy (version visible in logs, browser history, curl commands), supported by every API gateway and CDN without configuration; header-based versioning fails silently in infrastructure that ignores unknown headers.

184. ALWAYS support at least N-1 versions simultaneously in production and enforce a minimum 6-month deprecation notice for public API versions — clients cannot upgrade on your release schedule; document current, deprecated-with-date, and sunset versions in your OpenAPI spec.

185. ALWAYS treat adding a required field to a request schema, removing a field from a response schema, and changing a field's type as breaking changes for any API consumed by more than one service — "it's just internal" does not reduce coordination cost when the consumer is owned by a different team or runs on a different deploy cadence.

---

## Data Migration Strategies

186. ALWAYS choose between big-bang, incremental, and dual-write migration strategies before writing any migration code: big-bang suits tables under ~100K rows with an acceptable maintenance window; incremental suits large tables without a maintenance window; dual-write suits schema changes where old and new coexist during transition.

187. ALWAYS run a dual-write phase before cutting over reads for any migration that changes a primary key or foreign key structure — dual-write proves the new structure handles production write volume before read traffic touches it; cutting reads to a new schema that has never absorbed production writes is the most common cause of migration-induced incidents.

188. NEVER migrate a table larger than 10M rows without a backfill job that processes in batches of 1,000-10,000 rows with a configurable sleep between batches — unbounded backfills saturate I/O, hold long-lived locks, and starve application queries; a throttled 6-hour backfill is safer than a 30-minute backfill that causes a 2-hour latency spike.

---

## Observability-Driven Development

189. ALWAYS instrument a new endpoint or background job before shipping it — add structured logs, a Prometheus histogram for latency, and error type counters at code-write time, not as a follow-up ticket; observability added after the fact is incomplete because it misses the edge cases the author did not anticipate during retrofit.

190. NEVER define an alert for a metric that does not have a corresponding runbook section — an alert with no runbook produces an on-call engineer who acknowledges and silences it; alert creation and runbook section creation must be a single atomic PR.

191. ALWAYS define the SLO and alert thresholds for a new endpoint before writing the first line of implementation — the SLO defines what "working correctly" means; code written without a defined success criterion has no stopping point for optimization.

---

## Post-Mortems

192. ALWAYS complete a written post-mortem within 72 hours of any production incident causing user-visible impact exceeding 5 minutes — memories degrade and context disappears; a post-mortem written a week later is a reconstruction, not an investigation.

193. ALWAYS structure post-mortems as blameless — the goal is to identify systemic conditions that allowed the incident, not establish individual fault; a culture where engineers fear post-mortems produces under-reporting and no learning.

194. ALWAYS apply the 5 Whys starting from the user-visible symptom, not the first technical cause — asking "why did the deploy fail" stops at the proximate cause; asking "why were users unable to complete checkout for 22 minutes" leads to the systemic gap that is the actual prevention target.

195. ALWAYS ensure every post-mortem action item has a single named owner, a GitHub Issue link, and a due date within 30 days — action items with multiple owners or no due date have near-zero completion rate; an unactioned post-mortem is worse than none because it creates false belief the system has been improved.

---

## Capacity Planning

196. ALWAYS produce a capacity plan for any service expected to grow more than 2x in the next 6 months using: `headroom_months = (current_capacity - current_load) / monthly_growth_rate` — headroom under 3 months requires immediate action; discovering saturation via a production incident is always more expensive than a 30-minute capacity review.

197. ALWAYS plan capacity at the resource that saturates first, not the most obvious one — for Python async services, database connection pool exhaustion, Redis memory, and LLM API rate limits saturate before CPU; model each resource separately with its own utilization curve and headroom calculation.

198. ALWAYS include two traffic growth scenarios — conservative (current growth rate continues) and aggressive (growth doubles post-launch or viral event) — size infrastructure for conservative and document the aggressive trigger point; a plan with only one scenario does not capture the uncertainty that makes capacity planning valuable.

---

## Performance Budgets

199. ALWAYS assign a latency budget to every public endpoint before implementation, expressed as p99 target in milliseconds — without a budget the endpoint has no definition of "fast enough" and performance work is unbounded; define the budget in the design doc alongside the API contract.

200. ALWAYS decompose a latency budget into per-layer allocations: auth middleware, database query, external service call, serialization — `p99 < 200ms` with allocations `(auth: 10ms, DB: 120ms, serialize: 20ms, overhead: 50ms)` turns a single number into a verifiable constraint at each layer; a missing layer allocation is a missing constraint.

201. ALWAYS gate performance budgets in CI using a load test that runs on every PR touching an endpoint's critical path — a 30-second k6 or locust warmup test that fails the PR if p99 exceeds the budget catches regressions at the commit that introduced them, not before release.


---

## Developer Experience (DX) Tooling

202. ALWAYS configure pre-commit hooks using `.pre-commit-config.yaml` with at minimum: `ruff` (linting), `black` (formatting), `gitleaks` (secret scanning), and a fast `pytest -x -q` over the changed module — pre-commit hooks catch issues before they enter the repo; a developer who only discovers lint failures in CI has already pushed, waited, and context-switched.

203. ALWAYS define a `Makefile` at the project root with canonical targets: `make dev`, `make test`, `make lint`, `make fmt`, `make seed`, `make clean` — a project without a Makefile forces every contributor to reconstruct commands from memory; a Makefile is the authoritative local development contract.

204. ALWAYS provide a dev container definition (`devcontainer.json` or `docker-compose.dev.yaml`) for any project with non-trivial system dependencies — a dev container makes onboarding a single `docker compose up` command instead of a half-day environment setup; without it, "works on my machine" is a recurring incident cause.

---

## Documentation Quality

205. ALWAYS write an Architecture Haiku for every service: three lines capturing what it does, what it stores, and what it calls — "Ingests user notes / Stores in PostgreSQL, searches BM25 / Calls Anthropic for summaries"; it gives a new engineer a complete mental model in 15 seconds before reading any code.

206. ALWAYS maintain a `README.md` with these six sections: (1) one-sentence description, (2) prerequisites and setup, (3) how to run tests, (4) environment variables reference, (5) architecture overview, (6) deployment notes — a README missing any of these forces the reader to grep through code to recover information the author already had.

207. ALWAYS maintain a `docs/decision-log.md` recording every significant decision that does not warrant a full ADR — format: `YYYY-MM-DD: Chose orjson over stdlib json for 10x faster serialization. Owner: @dev.` — a decision log captures the 90% of decisions that would otherwise be lost in Slack.

---

## Incident Command Structure

208. ALWAYS designate an Incident Commander (IC) and a Scribe at the start of any P0/P1 incident involving more than one engineer — the IC owns the timeline, delegates diagnostic tasks, and is the sole person authorized to declare mitigation; the Scribe records a timestamped log of all actions and findings; without role separation, everyone talks at once and no one drives to resolution.

209. NEVER let the IC also be the engineer executing diagnostic commands during an active P0 — the IC's attention must stay on coordination and escalation decisions; the moment the IC starts running kubectl or psql commands, incident coordination collapses; assign a hands-on-keyboard engineer separately.

---

## Security Review in Engineering Workflow

210. ALWAYS complete a pre-ship security review checklist before any PR introducing a new authentication flow, data ingestion path, file upload, external service integration, or admin capability — verify: threat model exists, OWASP Top 10 assessed, secrets not hardcoded, input validation at boundary, and authorization check on every new endpoint; security review is a gate embedded in the PR template, not a separate phase.

211. ALWAYS designate a Security Champion per team whose role is to attend security reviews and keep threat models current — teams without a Security Champion accumulate security debt silently because no one tracks it sprint to sprint.

---

## Dependency Graph and Circular Import Detection

212. ALWAYS run `pydeps` or `importlab` as a CI step and fail the build if circular imports are detected — circular imports cause unpredictable initialization order bugs that manifest only under certain import sequences and indicate a module boundary violation that will compound as the codebase grows.

213. ALWAYS visualize the dependency graph before splitting a monolith — run `pydeps src/ --max-bacon=3 --cluster` and inspect which modules have the highest in-degree (split points for stable interfaces) and out-degree (refactoring priority for reducing coupling).

214. ALWAYS configure `import-linter` with a `ContractsFile` that explicitly forbids cross-layer imports and treat violations as build failures — a boundary enforced only by convention is violated within the first deadline crunch.

---

## Testing Strategy for LLM Outputs

215. ALWAYS maintain a golden file test suite for every LLM-backed endpoint: store expected outputs in `tests/golden/`, run the prompt at `temperature=0` against a pinned model version, and assert the output matches within a configurable similarity threshold — golden file tests catch prompt regressions that unit tests cannot because they test the actual model response, not a mock.

216. ALWAYS apply property-based testing (Hypothesis) to LLM output parsers and structured output validators — generate random valid and adversarial inputs, assert the parser never raises an unhandled exception and always returns a typed result; LLM parsers fail on edge cases (empty string, unicode, nested delimiters) impossible to enumerate manually.

217. ALWAYS build a regression test that runs the current prompt against 50 past production inputs and alerts when the fraction of changed outputs exceeds 5% — even a typo fix can shift the output distribution for a non-obvious subset of inputs; the 5% threshold catches material regressions while allowing natural variance.

---

## Semantic Versioning for Internal Libraries

218. ALWAYS apply semver (`MAJOR.MINOR.PATCH`) to any Python package shared across two or more services: MAJOR on breaking API changes, MINOR on backward-compatible additions, PATCH on bug fixes — without semver, consumers pin to exact versions permanently and never receive security patches.

219. NEVER release a MAJOR version of an internal library without a migration guide and a compatibility shim — a hard-cut MAJOR forces all consumers to upgrade simultaneously, creating a coordinated deployment requirement that blocks independent service deploys.

---

## Monorepo vs Polyrepo

220. ALWAYS make the monorepo vs polyrepo decision based on team topology and deploy coupling — choose monorepo when teams frequently make cross-service changes in one PR and services share internal libraries that change together; choose polyrepo when services have independent deploy cadences or teams whose boundaries must be enforced by access control, not convention.

221. NEVER adopt a monorepo without investing in incremental CI from day one — a monorepo where every PR runs every service's full test suite degrades CI time to 30+ minutes within 6 months; incremental CI (run only tests affected by changed files) is not optional, it is the precondition that makes monorepo viable at scale.

---

## Engineering Metrics (DORA)

222. ALWAYS track all four DORA metrics as a team health baseline: Deployment Frequency, Lead Time for Changes (commit to production), Change Failure Rate (% of deploys causing rollback), and Mean Time to Restore — elite benchmarks: DF multiple/day, LT < 1 day, CFR < 5%, MTTR < 1 hour; without measuring these, improvement efforts are directionally unjustified.

223. NEVER treat DORA metrics as performance reviews for individuals — they are system metrics reflecting process and tooling health; using them to evaluate individuals incentivizes gaming (trivial deploys, hiding incidents) and destroys the psychological safety that makes blameless post-mortems possible.

224. ALWAYS set a DORA-based improvement target at the start of each quarter tied to a specific process change — "reduce Lead Time from 4 days to 1 day by automating the staging approval gate" is actionable; "improve DORA scores" is not; the metric without a hypothesis about the cause produces no improvement.

