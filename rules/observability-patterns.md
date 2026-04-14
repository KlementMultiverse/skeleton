---
description: LLM observability rules — Langfuse, LangSmith, tracing, evals, cost control. Loads for all projects.
paths: ["**"]
---

# LLM Observability Patterns

> This file owns: Langfuse data model, LangSmith tracing, structured tracing, evaluation,
>   alerting thresholds, cost control, OpenTelemetry integration.
> Auth rules → security.md / auth-patterns.md
> API design rules → api-design.md

---

## Tracing — Mandatory Fields

1. ALWAYS create a trace with `id=request_id` propagated from HTTP middleware — every trace
   must be findable by its originating HTTP request ID.

2. ALWAYS bind `user_id` to the trace as early as possible after authentication resolves —
   never leave `user_id` unset; it is the primary key for cost attribution.

3. ALWAYS bind `session_id` to group multiple traces from the same conversation — without it,
   multi-turn conversations appear as unrelated isolated events.

4. ALWAYS record `metadata.latency_ms` on every generation — compute it as
   `int((time.monotonic() - t0) * 1000)` wrapping only the LLM API call, not the whole handler.

5. ALWAYS record `metadata.stop_reason` on every generation — `end_turn`, `max_tokens`,
   `tool_use`, `stop_sequence`. A `max_tokens` stop_reason means output was truncated; this
   is a quality defect and must be monitored separately.

6. ALWAYS record `usage.input` and `usage.output` token counts on every generation — these are
   the raw inputs for cost computation. A generation without token counts cannot be costed.

7. ALWAYS record the exact `model` string on every generation — `"claude-3-5-sonnet-20241022"`,
   not `"claude"`. Model aliases obscure cost breakdown by model tier.

8. NEVER create a trace without a name — use a consistent naming convention such as
   `"rag-query"`, `"agent-run"`, `"chat-completion"`. Unnamed traces are impossible to
   filter and aggregate in the Langfuse dashboard.

9. ALWAYS include `tenant_id` in trace metadata for multi-tenant applications — without it,
   you cannot compute per-tenant costs or debug tenant-specific quality issues.

10. ALWAYS include `prompt_version` in trace metadata when prompt management is active — the
    version number must come from the prompt object, not a hardcoded constant.

---

## Langfuse — Data Model Usage

11. USE Span for non-LLM work (retrieval, reranking, tool execution, DB queries) — do not use
    Generation for these; only Generation captures token usage and cost.

12. USE Generation only for actual LLM API calls — calling it on other work inflates token
    counts and distorts cost dashboards.

13. ALWAYS call `generation.end(...)` after the LLM call completes — a generation without an
    end timestamp has no latency, no output, and no token count.

14. ALWAYS call `span.end(...)` after the span's work completes — open spans with no end time
    make the trace timeline unusable.

15. ALWAYS pass `input` and `output` to spans and generations — empty input/output makes
    debugging impossible; you cannot replay or inspect what happened.

16. NEVER put sensitive data (passwords, tokens, PII, payment data) in Langfuse `input` or
    `output` fields — traces are stored and searchable; treat them as logs.

17. ALWAYS use `langfuse_context.update_current_trace(user_id=...)` inside `@observe()`-
    decorated functions rather than constructing a trace object manually — the decorator
    handles parent/child span linking automatically.

18. ALWAYS use `langfuse_context.update_current_observation(...)` to add metadata to the
    current span within an `@observe()` context — do not reach outside the decorator context
    to mutate the span.

19. NEVER call `langfuse.flush()` inside a tight loop — flush once at the end of a batch or
    at process shutdown, not after every generation. Flush is blocking and expensive.

20. ALWAYS call `langfuse.flush()` before process exit, at the end of background tasks, and
    at the end of serverless function handlers — a missing flush means all buffered
    observations are silently dropped.

21. ALWAYS call `langfuse.flush()` in the FastAPI lifespan shutdown block — not in a
    signal handler, not in `__del__`, only in the lifespan's post-yield teardown.

---

## Prompt Versioning

22. ALWAYS fetch prompts via `langfuse.get_prompt(name, version=N)` or
    `langfuse.get_prompt(name, label="production")` — never hardcode prompt text in
    application code; it bypasses versioning and makes rollback impossible.

23. ALWAYS pass the prompt object to the generation via `langfuse_prompt=prompt` (OpenAI
    integration) or `prompt=prompt` (generation constructor) — this links the trace to the
    prompt version in the Langfuse UI.

24. NEVER promote a new prompt version to the `production` label without first running it
    through the CI golden dataset eval gate — an unvalidated production prompt is a live
    regression.

25. ALWAYS use the `staging` label for canary deployments and A/B tests — never test
    unvalidated prompts using the `production` label.

26. ALWAYS compile prompts via `prompt.compile(**variables)` — never manually substitute
    variables with f-strings; compile() handles escaping and records the variable values.

27. NEVER cache the compiled prompt string indefinitely — cache the prompt object by version
    number with a TTL of 60 seconds to pick up configuration changes without a redeploy.

---

## LangSmith

28. ALWAYS set `LANGCHAIN_TRACING_V2=true` (not `LANGCHAIN_TRACING=true`) — the v2 variable
    enables the current tracing backend. The old variable silently does nothing in recent SDK
    versions.

29. ALWAYS set `LANGCHAIN_PROJECT` to a meaningful project name — do not use the default;
    all traces from all environments land in one bucket and become impossible to filter.

30. ALWAYS use `@traceable(run_type="retriever")` for retrieval functions,
    `@traceable(run_type="chain")` for orchestration, and `@traceable(run_type="llm")` for
    direct LLM calls — correct run_type enables run-type-specific filtering in LangSmith.

31. ALWAYS use `wrap_openai(client)` or equivalent wrapper for third-party LLM clients when
    using LangSmith — unwrapped clients do not capture token usage automatically.

32. NEVER run LangSmith tracing and Langfuse tracing simultaneously on the same LLM call —
    double-tracing wastes bandwidth, doubles storage costs, and produces duplicate records.

33. ALWAYS set `LANGSMITH_TRACING` (not `LANGCHAIN_TRACING_V2`) for non-LangChain code
    using the `@traceable` decorator — the environment variable name changed; verify against
    the current SDK documentation.

---

## Scores and Evaluation

34. ALWAYS attach at least one automated score to every production trace within 60 seconds
    of the trace completing — unscored traces cannot contribute to quality dashboards or
    regression detection.

35. ALWAYS use `observation_id` when scoring a specific generation rather than the whole
    trace — attaching faithfulness to the trace root when faithfulness measures a specific
    LLM call makes score analysis ambiguous.

36. ALWAYS use score names that are consistent across all experiments — `"faithfulness"` not
    `"faith"`, `"answer-faithfulness"`, or `"Faithfulness"`. Inconsistent names fragment
    dashboards.

37. NEVER use a model to judge its own output — always use a different model (or different
    provider) as the judge. Self-evaluation exhibits significant self-serving bias.

38. ALWAYS validate LLM-judge output with Pydantic before recording as a score — the judge
    can return malformed JSON; an unparsed score silently yields 0.0 or an exception.

39. ALWAYS include `comment` in scores produced by LLM-as-judge — the judge's reasoning
    text is essential for debugging score anomalies. A score without reasoning is a number
    with no context.

40. NEVER set RAGAS `faithfulness` threshold below 0.85 for production RAG — below this
    level the model is fabricating claims not present in the retrieved context.

41. NEVER set RAGAS `answer_relevancy` threshold below 0.80 — below this, answers are
    addressing something other than the user's question.

42. ALWAYS include `ground_truth` in test cases when computing `context_recall` — this
    metric is undefined without it; attempting to compute it without ground truth raises
    an error or produces nonsense.

43. NEVER generate ground truth using the same model whose output you are evaluating —
    ground truth must come from human expert review or a separate authoritative source.

---

## Golden Datasets and CI

44. ALWAYS maintain a golden dataset in Langfuse with at least 20 items covering core user
    scenarios — fewer than 20 items produces statistically unreliable average scores.

45. ALWAYS run the golden dataset eval in CI for every PR that touches prompt files,
    LLM configuration, or retrieval logic — changes to non-LLM code do not require it.

46. ALWAYS use the git SHA as part of the Langfuse run name in CI —
    `f"ci-{os.environ['GIT_SHA'][:8]}"` — so each CI run is uniquely identifiable
    in the Langfuse experiments view.

47. ALWAYS make the CI eval job fail the PR when any quality metric drops below its baseline
    threshold — do not merge first and investigate later; an eval gate that does not block
    merges is not a gate.

48. ALWAYS add `langfuse.flush()` at the end of every CI eval script — without it, the
    process exits before scores reach Langfuse and the CI check script reads stale data.

49. NEVER use the same dataset for both development iteration and regression testing — keep
    a locked golden dataset for CI and a separate development dataset for exploration.

50. ALWAYS promote a random sample of production traces to the golden dataset when they have
    high human quality scores — production-derived golden datasets catch real distribution
    drift; synthetic ones do not.

---

## Request ID Propagation

51. ALWAYS generate a `request_id` in HTTP middleware (UUID4) if the incoming request does
    not carry an `X-Request-ID` header — never generate it inside route handlers.

52. ALWAYS pass `request_id` back in the `X-Request-ID` response header — clients and load
    balancers can then log it for end-to-end correlation.

53. ALWAYS use the `request_id` as the Langfuse `trace.id` — this makes every LLM trace
    discoverable from access logs using only the HTTP request ID.

54. ALWAYS propagate `request_id` as a parameter (or via context var) into every function
    that creates LLM traces — do not read it from a global; globals break in async contexts
    with concurrent requests.

55. ALWAYS store `request_id` in a `contextvars.ContextVar` for async propagation — do not
    use `threading.local()` in async code; it is per-thread, not per-coroutine.

---

## Cost Attribution and Control

56. ALWAYS compute cost in `Decimal`, not `float` — floating-point arithmetic accumulates
    errors when summing millions of micro-cost transactions. Use
    `Decimal(input_tokens) * Decimal("0.000003")`.

57. ALWAYS store model pricing in a configuration dict, not hardcoded in business logic —
    model prices change; a configuration dict is updated without a code change.

58. ALWAYS pass `usage.input_cost` and `usage.output_cost` to Langfuse generations when you
    compute cost yourself — Langfuse can also compute cost from token counts if you configure
    model prices in the UI, but explicit cost fields take precedence and are more accurate for
    custom or fine-tuned models.

59. ALWAYS set `max_tokens` on every LLM call — never leave it unset. An unset `max_tokens`
    allows the model to generate up to its context window limit, which can produce $10 charges
    from a single malicious or accidental request.

60. ALWAYS count tokens before making expensive LLM calls on user-supplied input — use
    `client.messages.count_tokens(...)` for Anthropic. If the token count exceeds the hard
    limit, raise a domain exception before spending money.

61. ALWAYS enforce a per-user monthly token budget using an atomic Redis counter —
    `INCRBYFLOAT` + `EXPIRE` in a pipeline. Check the budget before each LLM call and raise
    a `BudgetExceededError` domain exception if exceeded.

62. NEVER check the budget after the call — the money is already spent. Budget checks must
    happen before the API call.

63. ALWAYS set a TTL on every Redis budget key — the TTL must be long enough to cover the
    full billing period (1 month) but no longer. Without a TTL, budget keys accumulate
    indefinitely.

64. ALWAYS route simple classification and extraction tasks to the cheapest available model
    tier — `claude-3-5-haiku`, `gpt-4o-mini`. Reserve expensive models for tasks that
    measurably benefit from stronger reasoning.

65. NEVER perform model routing based on user-supplied complexity labels — users will always
    claim their task is complex. Route based on measured signals: context length, query
    word count, task type classifier.

66. ALWAYS summarise chat history when it exceeds the token budget for context — use a cheap
    model for summarisation. Never truncate silently; truncation without summary loses key
    facts from the conversation.

67. ALWAYS strip redundant whitespace, repeated newlines, and boilerplate from prompts
    before counting tokens — prompt bloat can reduce input tokens by 10-20% with zero quality
    impact.

---

## Alerting Thresholds

68. ALWAYS alert when error_rate > 1% over any 5-minute window — at 1% on a 1000 RPM
    service, 10 users per minute receive errors. This is P1 territory.

69. ALWAYS alert when P99 latency > 30s — 30 seconds is the UX abandonment threshold for
    most interactive LLM applications. Beyond this, users have already given up.

70. ALWAYS alert when cost_per_request > $0.10 — a single $0.10 request on a 100K RPD
    service costs $10K/day. Catch runaway costs before the billing cycle.

71. ALWAYS alert when context_utilization > 90% — at 90%+ of the context window, inputs are
    being silently truncated. Quality degrades unpredictably and the model may lose critical
    earlier context.

72. ALWAYS alert on token_count anomalies: when a user's per-request token count exceeds
    5x their 24-hour P95 baseline — this is the primary early signal of prompt injection
    attempts. A sudden spike means someone submitted a massive payload.

73. NEVER set error_rate alerts with a window shorter than 3 minutes — LLM APIs have natural
    transient spikes from rate limiting. A 1-minute window produces excessive false positives.

74. NEVER set a single global alerting threshold across all user segments — power users have
    legitimately higher token counts. Compute baselines per user cohort.

75. ALWAYS include `tenant_id`, `user_id`, and `model` in every alert payload — an alert
    that says "high error rate" without identifying which tenant or model is actionless.

---

## OpenTelemetry

76. ALWAYS use the standardised `gen_ai.*` span attribute names from the OTel GenAI semantic
    conventions — do not invent custom attribute names for model, token counts, or provider.
    Custom names break compatibility with every OTel-aware backend.

77. ALWAYS set `gen_ai.system` to the provider name (`"anthropic"`, `"openai"`) and
    `gen_ai.request.model` to the exact model string — these are the two attributes all
    backends use for model-level aggregation.

78. ALWAYS set `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` as span
    attributes on every LLM span — these are what cost dashboards and token-usage reports
    read. Missing them means the span is invisible to cost tooling.

79. ALWAYS use `BatchSpanProcessor` in production, not `SimpleSpanProcessor` —
    SimpleSpanProcessor is synchronous and blocks the request thread on every span export.

80. ALWAYS configure a sampling strategy — never export 100% of traces in production at
    high traffic. `TraceIdRatioBased(0.10)` at 10% is a reasonable default; supplement with
    always-sample rules for errors and low quality scores.

81. NEVER sample out error traces — always export spans where `error=true` regardless of
    the base sampling rate. Error traces are precisely the ones you need most.

82. NEVER export raw prompt content to OTel backends unless you control the backend and
    have verified its data retention and access controls — prompt content can contain PII.
    Export token counts and model names, not prompt text, to OTel; use Langfuse for prompt
    content with its proper access controls.

83. ALWAYS use a separate tracer name per service — `trace.get_tracer("rag-service")` not
    `trace.get_tracer("app")`. Tracer name is the primary grouping dimension for filtering
    in Jaeger and Grafana Tempo.

---

## Anti-Patterns (NEVER do these)

84. NEVER skip tracing for internal or background LLM calls — cost and quality issues often
    originate in background jobs, not user-facing endpoints. Everything must be traced.

85. NEVER use `print()` or ad-hoc logging as a substitute for traces — logs are not
    queryable by prompt version, model, user, or eval score. Use a tracing platform.

86. NEVER aggregate cost by model name using string prefix matching — `"claude-3"` matches
    both Haiku and Sonnet. Always match on the exact model string.

87. NEVER store Langfuse or LangSmith API keys in code — environment variables only.
    Use `pydantic-settings` to validate presence at startup.

88. NEVER block the request thread on score computation — compute scores in a background
    task. LLM-as-judge takes 2-5 seconds; adding that to request latency is unacceptable.

89. NEVER create a new `Langfuse()` client per request — create it once at application
    startup and reuse. Multiple clients with separate buffers miss the flush on shutdown.

90. NEVER disable tracing in production to reduce latency — Langfuse's async client adds
    < 1ms overhead. If tracing is too slow, switch to `BatchSpanProcessor` or reduce the
    flush interval; do not disable.

91. NEVER hardcode prompt text inside route handlers or service functions — prompt text
    belongs in Langfuse prompt management (or a dedicated prompts module), not business logic.

92. NEVER evaluate prompt quality only in development — production traffic has a different
    distribution than your dev test cases. Always run continuous evals on a sample of
    production traces.

93. NEVER use a fixed window rate-limiting algorithm for token budget enforcement — fixed
    windows allow 2x burst at window boundaries. Use a sliding window (Redis sorted sets
    or `INCRBYFLOAT` with sub-key time buckets).

94. NEVER expose raw Langfuse trace URLs or trace IDs in client-facing error responses —
    trace IDs can be used to probe trace contents if Langfuse is publicly accessible.

95. NEVER let the LLM-judge model score responses that contain its own training data
    verbatim — this produces inflated faithfulness scores. Use diverse, novel test cases.

96. NEVER report eval scores as pass/fail without showing the raw metric value — "passed"
    hides whether the score was 0.851 (barely passing) or 0.97 (excellent). Always log
    the numeric value alongside the pass/fail decision.

97. NEVER use `eval()` or `exec()` to dispatch LLM-requested tool calls — always match
    the tool name against an explicit allowlist dict. LLM outputs are untrusted inputs.

98. NEVER trust `content_length` alone to estimate token count — byte length and token
    count have a non-linear relationship, especially for non-English text and code. Always
    use the tokenizer or the count_tokens API endpoint.

99. NEVER silently swallow exceptions from the tracing SDK — Langfuse and LangSmith clients
    can raise on network errors; catch and log them, but let the primary LLM call continue.
    Observability failure must never cause application failure.

100. NEVER defer the golden dataset build until "after launch" — without a golden dataset,
     you cannot detect regressions from the very first prompt change. Build it from day one
     with at least 20 manually-verified examples.

101. NEVER mix Langfuse scores from different judge models in the same trend chart —
     Claude-3-5-Haiku judging and Claude-3-5-Sonnet judging produce different score
     distributions. Tag scores with judge model name and filter separately.

102. ALWAYS include the judge model name in the Langfuse score `comment` field —
     `comment=f"judge={judge_model} reasoning={reasoning}"`. When scores change, you need
     to know whether the change is from quality improvement or judge model change.

103. ALWAYS rate-limit the LLM-as-judge pipeline separately from the main application —
     eval jobs running in CI or background can consume your entire LLM API quota, starving
     production traffic. Use a separate API key or a hard concurrency limit.

104. NEVER use `.format()` or f-strings to build the LLM judge prompt with raw user content —
     use explicit delimiters: `<answer>{answer}</answer>`. This prevents prompt injection
     via the content being evaluated.

105. ALWAYS version the judge prompt separately from the application prompt — a judge prompt
     change is a breaking change to your eval scores. Track both versions in every score's
     metadata.

---

## Langfuse SDK v3 — Migration and Flush

106. ALWAYS use the context manager pattern in Langfuse SDK v3 — `with langfuse.start_as_current_span("name") as span:` replaces the `@observe` decorator model when you need explicit span control; both APIs coexist but do not mix them in the same trace hierarchy.

107. ALWAYS call `langfuse.flush()` before process exit in short-lived scripts, Lambda functions, and Cloud Run jobs — the SDK batches traces asynchronously; without an explicit flush, the last batch may never be sent.

108. NEVER rely on `atexit` hooks to flush Langfuse in serverless environments — the runtime may freeze the process before the atexit handler runs; always flush explicitly inside the request handler's `finally` block.

109. ALWAYS set `flush_interval=0.5` (seconds) and `flush_at=5` (events) for serverless functions — the defaults assume long-lived processes; reducing both thresholds ensures traces are sent before the function returns.

---

## W3C Traceparent Propagation

110. ALWAYS propagate the W3C `traceparent` header across service boundaries — extract it from the incoming request with `TraceContextTextMapPropagator().extract(carrier=request.headers)` and inject it into outgoing calls; without propagation, distributed traces are fragmented into disconnected spans.

111. ALWAYS attach the `trace_id` from the active OTel span to the Langfuse `trace_id` field when both systems are used — `langfuse_trace_id = format(span.get_span_context().trace_id, "032x")`. This lets you click from a Langfuse generation to the OTel trace in Jaeger or Honeycomb.

112. NEVER generate a Langfuse `trace_id` independently when an OTel trace is already active — inventing a separate ID breaks the correlation. Always derive it from the active span context.

---

## Sampling Strategy

113. ALWAYS set `LANGFUSE_SAMPLE_RATE` (or the SDK equivalent) before enabling self-hosting — at 100 requests/second with 10 spans/request, that is 86 million spans/day; without sampling, your ClickHouse write throughput and storage costs will blow the budget in days.

114. NEVER confuse `sample_rate` (SDK-level, applied before sending) with OTel `ParentBasedTraceIdRatio` (applied at trace creation) — SDK-level sampling loses traces that are already partially written; OTel-level sampling makes the decision once at trace creation and propagates it consistently.

115. ALWAYS sample at the root span level and propagate the sampling decision via `tracestate` — never sample independently at each service; services must respect the upstream sampling decision or you get partial traces.

---

## Offline vs Online Evaluation

116. ALWAYS distinguish offline eval (run against a static golden dataset before deployment) from online eval (run against live production traces after deployment) — they answer different questions; offline checks for regressions, online detects distribution shift and real-user quality.

117. ALWAYS run offline evals in CI and block the PR merge on score drops — this is the regression gate. Online evals are a monitoring signal, not a deployment gate.

118. NEVER use online eval scores as the sole deployment signal — production traffic has different distribution than the golden dataset; an online score drop might be query distribution change, not quality regression. Correlate with offline scores.

---

## Human Feedback

119. ALWAYS create a Langfuse score with `data_type="BOOLEAN"` for thumbs-up/thumbs-down feedback — `langfuse.score(trace_id=tid, name="user_feedback", value=1.0, data_type="BOOLEAN", comment="thumbs_up")`. Boolean type enables aggregation as a rate in dashboards.

120. ALWAYS attach the `user_id` and `session_id` to human feedback scores — a score without user context cannot distinguish whether one user gave all the negative feedback (a single unhappy user) or feedback is distributed across users (a systemic issue).

121. NEVER ignore thumbs-down feedback — set up a Langfuse webhook or polling job to forward negative scores to a Slack channel or PagerDuty within 5 minutes; negative user feedback is the most reliable signal of a real quality regression.

---

## Rolling Score Alerting

122. ALWAYS alert on rolling 7-day average score drops, not single-trace scores — individual scores have high variance; the rolling average filters noise and surfaces genuine quality drift.

123. ALWAYS set separate alert thresholds for faithfulness, answer_relevancy, and context_precision — a drop in context_precision signals retrieval degradation; a drop in faithfulness signals hallucination increase; they require different remediation.

124. NEVER combine judge scores from different time windows in the same alert rule — a 1-hour p99 alert and a 7-day average alert have completely different baseline distributions; mixing them produces false positives on legitimate usage spikes.

---

## NLI Hallucination Detection

125. ALWAYS use a Natural Language Inference (NLI) cross-encoder (e.g., `cross-encoder/nli-deberta-v3-base`) as a fallback hallucination detector when no external judge API is available — check if the claim in the LLM output is entailed by the retrieved context. Score below 0.5 entailment probability = potential hallucination.

126. ALWAYS log the NLI entailment score as a Langfuse score named `nli_entailment` alongside the claim text — this creates a searchable, trending signal for grounding failures without adding LLM API cost.

---

## Cache Token Accounting

127. ALWAYS account for `cache_read_input_tokens` and `cache_creation_input_tokens` separately from `input_tokens` in cost calculations — Anthropic charges `cache_creation_input_tokens` at 1.25× the base rate and `cache_read_input_tokens` at 0.1× the base rate; conflating them into a single `input_tokens` total produces incorrect cost accounting.

128. ALWAYS log the cache hit ratio per prompt (cache_read / total_input) as a Langfuse metric — a cache hit ratio below 0.5 on a static-system-prompt workload indicates the prompt is changing too frequently or the TTL is misconfigured.

---

## Output Ratio Monitoring

129. ALWAYS track the output/input token ratio per endpoint and alert when it exceeds 3.0 — a ratio above 3.0 often indicates the model is verbose-looping or not following length instructions; it also inflates cost and latency.

130. ALWAYS tag Langfuse generations with `metadata={"endpoint": "/api/v1/summarize"}` — without endpoint tagging, cost and quality dashboards are uninterpretable because different endpoints have completely different token patterns.

---

## Circuit Breaker and DLQ

131. ALWAYS implement a circuit breaker (CLOSED → OPEN → HALF-OPEN) on the LLM API call path — when the circuit is OPEN, return a cached response or a graceful degradation message; never queue an unbounded backlog of retries that will overwhelm the API when it recovers.

132. ALWAYS route failed LLM calls to a Dead Letter Queue (DLQ) after 3 retry attempts — failed calls in the DLQ are replayable once the issue is resolved; silently dropping them loses user work and makes failures invisible.

133. NEVER retry the same LLM call more than 3 times without exponential backoff and jitter — retry storms occur when a 429 causes 1,000 clients to simultaneously retry at the same interval, creating a thundering herd that prolongs the outage.

---

## OTel Advanced Patterns

134. ALWAYS attach `user.id` and `tenant.id` via OTel Baggage for cross-service propagation — `baggage.set("user.id", user_id)` propagates through all downstream spans automatically; without Baggage, downstream services must duplicate the user resolution logic.

135. ALWAYS use span events (not span attributes) for streaming LLM chunk content — `span.add_event("chunk", {"chunk.text": text, "chunk.index": i})` keeps the span timeline intact and avoids attribute size limits; span attributes have a 256-byte default value limit that streaming tokens will frequently exceed.

