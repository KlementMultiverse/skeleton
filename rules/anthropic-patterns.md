---
description: Anthropic SDK rules. Loads for all FastAPI projects using Claude.
paths: ["*.py", "app/**/*.py", "main.py"]
---

# Anthropic SDK Rules (L9)

> Streaming output → see `fastapi-patterns.md` rule 21
> Retry logic (tenacity) → see `error-handling.md`
> Credentials from env → see `security.md`

---

## Client Initialization

1. Initialize `AsyncAnthropic` ONCE in lifespan — NEVER inside a route handler
   ```python
   @asynccontextmanager
   async def lifespan(app: FastAPI):
       app.state.llm = AsyncAnthropic(
           api_key=settings.anthropic_api_key,
           max_retries=3,
           timeout=httpx.Timeout(60.0, connect=5.0)
       )
       yield
   ```
2. Inject client via `request.app.state.llm` or `Depends()` — never re-create per request
3. Always set explicit `timeout` — default has no timeout (requests hang forever)
4. Set `max_retries=3` on client — SDK retries transient errors automatically

## Messages API

5. Always set `max_tokens` explicitly — required field, never omit
6. Always check `stop_reason` before reading content:
   - `"end_turn"` → read `message.content[0].text`
   - `"tool_use"` → handle tool calls, send results back, continue loop
   - `"max_tokens"` → response truncated — increase `max_tokens` or reduce input
   - `"model_context_window_exceeded"` → input too large — truncate and retry
7. `stop_details` is now a structured object on the response alongside `stop_reason` — same info, richer format
7. Log `usage.input_tokens` and `usage.output_tokens` on EVERY call — needed for cost tracking
9. NEVER put sensitive data in messages — conversation history is logged
10. Use `claude-sonnet-4-6` as default model — balance of speed and quality
    Use `claude-haiku-4-5-20251001` for high-volume utility calls (commit messages, summaries)
    Use `claude-opus-4-6` for complex reasoning only
    NEVER call `claude-3-haiku-20240307` — retired April 19, 2026

## Tool Use Loop

10. Tool use requires a loop — NEVER assume one round trip is enough
    ```python
    messages = [{"role": "user", "content": task}]
    while True:
        response = await client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )
        if response.stop_reason == "end_turn":
            break
        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            results = await execute_tools(response.content)
            messages.append({"role": "user", "content": results})
    ```
11. Append assistant message BEFORE tool results — message history must be chronological
12. Always include `tool_use_id` in tool results — Claude matches results to calls by ID
13. Cap tool loop at `max_iterations=20` — prevent infinite loops
14. NEVER pass user-supplied content directly as tool input without validation — prompt injection risk

## Streaming

15. Use `client.messages.stream()` async context manager for streaming
    ```python
    async with client.messages.stream(...) as stream:
        async for text in stream.text_stream:
            yield text
        final = await stream.get_final_message()  # has usage stats
    ```
16. Always call `stream.get_final_message()` after streaming — only way to get usage stats
17. Use `StreamingResponse` with async generator for streaming to HTTP clients — see `fastapi-patterns.md`
18. NEVER buffer full streaming response before sending — defeats the purpose

## Token Counting

19. Count tokens BEFORE large calls — use `client.messages.count_tokens()`
    ```python
    count = await client.messages.count_tokens(
        model="claude-sonnet-4-6",
        system=system_prompt,
        messages=messages,
        tools=tools
    )
    ```
20. If input tokens > 90% of context window → truncate middle messages first
21. NEVER truncate: system prompt, first user message, most recent user message
22. Log token count before batch operations — catch runaway costs early

## Prompt Caching

23. Cache system prompts longer than 1024 tokens — use `cache_control: {"type": "ephemeral"}`
    ```python
    system=[{
        "type": "text",
        "text": long_system_prompt,
        "cache_control": {"type": "ephemeral"}
    }]
    ```
24. Cache static tool definitions — they don't change between calls
25. Cache RAG context when the same documents are used across multiple turns
26. Cache TTL is **1 hour** (GA since Aug 2025) — cache hit within 1 hour = ~90% cost reduction
27. Minimum 1024 tokens required for caching — shorter content is not cached
28. Check `usage.cache_read_input_tokens > 0` to verify cache hit — log cache miss rate
29. **Automatic caching mode** — set `cache_control` once on the request body; system auto-advances the cache point as conversation grows. Use when you don't want to manually manage breakpoints:
    ```python
    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        cache_control={"type": "ephemeral"},  # on request body, not content block
        system="...",
        messages=messages
    )
    ```

## Structured Output

30. **Breaking change (Jan 2026):** `output_format=` parameter renamed to `output_config=`:
    ```python
    # OLD (pre-Jan 2026, broken on SDK ≥ 0.75):
    result = client.messages.parse(output_format=MyModel, ...)

    # NEW (current):
    result = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        output_config={"format": {"type": "json_object", "schema": MyModel.model_json_schema()}},
        messages=[...]
    )
    ```
31. Always use `strict=True` on Pydantic models validating LLM output — see `pydantic-patterns.md`
32. NEVER trust unvalidated LLM output — always parse through Pydantic before saving to DB
33. On `ValidationError`: retry ONCE with error appended to prompt — do NOT retry blindly

## Error Hierarchy

33. RETRY with exponential backoff:
    - `RateLimitError` (429) — respect `Retry-After` header if present
    - `APIConnectionError` — network issue
    - `APITimeoutError` — request timed out
    - `APIStatusError` with status 529 — Claude overloaded
34. NEVER retry:
    - `AuthenticationError` (401) — wrong API key, fix the config
    - `PermissionDeniedError` (403) — no access
    - `BadRequestError` (400) — bad prompt or params, fix the code
    - `ValidationError` — schema mismatch, fix the code
35. Use `tenacity` for retry logic — see `error-handling.md` for full pattern
36. On `RateLimitError`: check `e.response.headers.get("retry-after")` before fixed backoff

## Context Window Management

38. Context window limits by model (GA, no beta header, standard pricing):
    - `claude-sonnet-4-6`: **1M tokens** input, 64k output
    - `claude-opus-4-6`: **1M tokens** input, 128k output
    - `claude-haiku-4-5-20251001`: 200k tokens input, 64k output
39. Handle `stop_reason == "model_context_window_exceeded"` — input too large, truncate and retry
40. Truncate MIDDLE of conversation first — preserve system + first + last messages
41. For coding agents: cap codebase context at 50k tokens even with 1M window — RAG selects relevant files, cost scales with context size
42. NEVER send full file contents for files >500 lines — chunk and select relevant sections

## Extended Thinking

43. Enable extended thinking for complex reasoning tasks (math, multi-step logic, hard code review):
    ```python
    # Sonnet 4.6 — manual budget_tokens (still supported)
    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=16000,
        thinking={"type": "enabled", "budget_tokens": 10000},
        messages=[{"role": "user", "content": "Solve..."}]
    )

    # Opus 4.6 — use effort parameter (budget_tokens deprecated on Opus 4.6)
    response = await client.messages.create(
        model="claude-opus-4-6",
        max_tokens=16000,
        thinking={"type": "adaptive"},  # Opus 4.6 default — adaptive reasoning depth
        effort="high",                   # "low" | "medium" | "high"
        messages=[{"role": "user", "content": "Solve..."}]
    )
    ```
44. `budget_tokens` must be < `max_tokens` — extended thinking uses tokens from the max_tokens budget
45. Suppress thinking blocks from response (Mar 2026): `thinking={"type": "enabled", "budget_tokens": N, "display": "omitted"}` — faster streaming, same billing, thinking preserved for signatures
46. Thinking blocks MUST be included when you pass prior assistant turns back (for multi-turn) — strip to save tokens with `display: "omitted"`
47. NEVER display thinking blocks to end users — they are internal reasoning artifacts
48. Use extended thinking ONLY when needed — it costs more tokens and is slower

## Vision (Image Input)

46. Pass images as content blocks alongside text:
    ```python
    # From URL
    messages=[{"role": "user", "content": [
        {"type": "image", "source": {"type": "url", "url": "https://example.com/image.png"}},
        {"type": "text", "text": "What does this diagram show?"}
    ]}]

    # From base64 (for local files)
    import base64
    with open("image.png", "rb") as f:
        data = base64.standard_b64encode(f.read()).decode("utf-8")
    messages=[{"role": "user", "content": [
        {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": data}},
        {"type": "text", "text": "What's in this screenshot?"}
    ]}]
    ```
47. Supported media types: `image/jpeg`, `image/png`, `image/gif`, `image/webp`
48. Max image size: 5MB per image, up to 20 images per request
49. URL images are fetched by Anthropic's servers — NEVER use URLs containing credentials

## Batches API

50. Use Batches API for bulk processing (100s–1000s of requests) — 50% cheaper than real-time:
    ```python
    batch = await client.messages.batches.create(
        requests=[
            {"custom_id": f"req-{i}", "params": {
                "model": "claude-haiku-4-5",
                "max_tokens": 256,
                "messages": [{"role": "user", "content": text}]
            }}
            for i, text in enumerate(texts)
        ]
    )
    # Poll until complete
    while batch.processing_status != "ended":
        await asyncio.sleep(60)
        batch = await client.messages.batches.retrieve(batch.id)
    # Stream results
    async for result in await client.messages.batches.results(batch.id):
        if result.result.type == "succeeded":
            print(result.custom_id, result.result.message.content[0].text)
    ```
51. Batches process within 24 hours — NEVER use for latency-sensitive paths
52. Use Batches API for: bulk classification, embedding generation prep, nightly summaries, dataset labeling

## Files API (beta)

53. Upload reusable files once, reference by ID in multiple requests — avoids resending large content:
    ```python
    # Upload once
    with open("codebase.txt", "rb") as f:
        file = await client.beta.files.upload(
            file=("codebase.txt", f, "text/plain")
        )
    file_id = file.id

    # Reference in messages (no re-upload)
    messages=[{"role": "user", "content": [
        {"type": "text", "text": "Review this codebase for security issues"},
        {"type": "document", "source": {"type": "file", "file_id": file_id}}
    ]}]
    ```
54. Files are stored server-side — `client.beta.files.list()`, `delete(file_id)`, `retrieve(file_id)`
55. Use Files API for: large context docs shared across many requests, uploaded PDFs, static reference material
56. NEVER store sensitive user data via Files API — files persist server-side

## Server-Side Tools (GA — no beta headers required)

57. Anthropic provides built-in tools you can enable without implementing them yourself:
    ```python
    tools = [
        {"type": "web_search_20250305", "name": "web_search"},   # live web search
        {"type": "web_fetch_20250124", "name": "web_fetch"},     # fetch URL content
        {"type": "code_execution_20250522", "name": "code_execution"},  # Python execution sandbox
        {"type": "memory_tool_20250618", "name": "memory_tool"},  # cross-conversation memory
    ]
    ```
58. Server-side tools are GA as of Feb 17, 2026 — no `anthropic-beta` headers needed
59. Code execution runs in Anthropic's sandbox — safe for executing LLM-generated code
60. When code execution and web_fetch are used together, code execution incurs no extra charge
61. Use server-side tools when you don't want to build and maintain tool infrastructure yourself

## Coding Agent Specific

62. Agent runs tool loop until `stop_reason == "end_turn"` — no fixed step count
63. Always cap with `max_iterations` guard — NEVER let agent loop without limit
64. Log every tool call: tool name, input, output, duration — full audit trail
65. Sandbox isolation is mandatory for code execution — use Anthropic's code_execution tool OR containerize
66. Commit message generation: use `claude-haiku-4-5-20251001` (cheap, fast, good enough)
67. PR body generation: use `claude-sonnet-4-6` (needs quality reasoning about changes)

## MCP (Model Context Protocol) Integration

> Full MCP build rules → see `MCP-SOP.md`. This section covers SDK-level integration only.

68. MCP is the standardized protocol layer OVER the tool use loop. The 6-step flow:
    ```
    1. INIT     → client spawns MCP server subprocess (STDIO) or connects to URL (HTTP)
    2. DISCOVER → client sends tools/list, server returns all tool schemas
    3. INJECT   → tool schemas are put into Claude's context via tools= parameter
    4. DECIDE   → Claude sees user request + tool list, outputs tool_use block
    5. ROUTE    → MCP client sends tools/call to server, server executes
    6. RETURN   → server result becomes tool_result block, appended to messages
    ```

69. **MCP Connector (remote HTTPS servers)** — beta header `mcp-client-2025-11-20`:
    ```python
    response = client.beta.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Check the weather in Tokyo"}],
        mcp_servers=[{
            "type": "url",
            "url": "https://my-weather-server.example.com/sse",
            "name": "weather",
            "authorization_token": "OAUTH_TOKEN",  # optional
        }],
        tools=[{"type": "mcp_toolset", "mcp_server_name": "weather"}],
        betas=["mcp-client-2025-11-20"],
    )
    ```

70. MCP Connector supports `https://` URLs ONLY — STDIO servers cannot connect via MCP Connector
71. Use allowlist to expose only specific tools from a server:
    ```python
    tools=[{
        "type": "mcp_toolset",
        "mcp_server_name": "weather",
        "default_config": {"enabled": False},           # block everything by default
        "configs": {"get_weather": {"enabled": True}},  # allow only this one
    }]
    ```
72. MCP response includes two new block types — handle them in tool loop:
    - `mcp_tool_use` — Claude calling an MCP tool (similar to `tool_use`)
    - `mcp_tool_result` — server's response to that call
73. **STDIO MCP servers** (local Python) — use `mcp` package directly, NOT via MCP Connector:
    ```python
    from mcp import ClientSession, StdioServerParameters
    from mcp.client.stdio import stdio_client

    server_params = StdioServerParameters(command="python", args=["server.py"])
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            # convert to Anthropic format and pass as tools= in messages.create
    ```
74. NEVER use `print()` in an MCP server — stdout is the JSON-RPC wire, print corrupts the protocol
75. Always log MCP servers to `stderr` — `logging.basicConfig(stream=sys.stderr, level=logging.INFO)`
76. Use `@mcp.tool` from `fastmcp` for server-side tool definition — docstring IS the tool schema
77. Docstring must answer 4 things: what it does, parameter format + example, return format + example, error strings

## Claude Agent SDK — Multi-Agent Patterns

78. Install: `pip install claude-agent-sdk`. The SDK wraps the Claude agent runtime with an async generator loop:
    ```python
    from claude_agent_sdk import query, ClaudeAgentOptions

    async for message in query(
        prompt="Fix the authentication bug in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"],
            permission_mode="acceptEdits",
        ),
    ):
        if hasattr(message, "content"):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)
    ```

79. **Orchestrator/worker topology** — define workers as `AgentDefinition`, orchestrator invokes via `"Agent"` tool:
    ```python
    from claude_agent_sdk import AgentDefinition

    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Agent"],   # "Agent" required to spawn workers
        agents={
            "security-scanner": AgentDefinition(
                description="Scans code for security vulnerabilities. Use for any security review.",
                prompt="You are a security expert. Identify vulnerabilities.",
                tools=["Read", "Grep"],            # workers: read-only is safest
                model="opus",
            ),
            "test-runner": AgentDefinition(
                description="Runs pytest and analyzes failures. Use to execute tests.",
                prompt="You are a test engineer. Run tests and report results.",
                tools=["Bash", "Read"],
                model="sonnet",
            ),
        },
    )
    ```
80. Subagents CANNOT spawn subagents — NEVER put `"Agent"` in a subagent's tools list
81. **Parallel execution** — `asyncio.gather` runs multiple workers concurrently:
    ```python
    security_result, coverage_result = await asyncio.gather(
        run_worker("security-scanner", "Scan src/ for vulnerabilities"),
        run_worker("coverage-analyzer", "Analyze test coverage for src/"),
    )
    ```
82. **Sequential handoffs** — pass output of one agent as prompt to next (string injection):
    ```python
    research = await collect_result("Research FastAPI auth best practices", tools=["WebSearch"])
    implementation = await collect_result(f"Based on:\n{research}\n\nImplement JWT in auth.py", tools=["Read","Write"])
    ```
83. Resume a session across turns: `ClaudeAgentOptions(resume=session_id)` — subagent retains full history
84. `AgentDefinition.model` accepts `"sonnet" | "opus" | "haiku" | "inherit"` — use `"inherit"` to match parent
85. Always pass context explicitly in the prompt string — that is the only channel from orchestrator to subagent

## Managed Agents (beta)

86. Managed Agents = Anthropic runs the agent loop, container, and tool execution for you (beta `managed-agents-2026-04-01`):
    ```python
    # 1. Create reusable agent definition
    agent = client.beta.agents.create(
        name="Coding Assistant",
        model="claude-sonnet-4-6",
        system="You are a helpful coding assistant.",
        tools=[{"type": "agent_toolset_20260401"}],  # built-in: bash, files, web search
    )
    # 2. Create environment (cloud container)
    environment = client.beta.environments.create(
        name="my-env",
        config={"type": "cloud", "networking": {"type": "unrestricted"}},
    )
    # 3. Start session
    session = client.beta.sessions.create(
        agent=agent.id, environment_id=environment.id, title="Fix auth bug"
    )
    # 4. Stream events
    with client.beta.sessions.events.stream(session.id) as stream:
        client.beta.sessions.events.send(
            session.id,
            events=[{"type": "user.message", "content": [{"type": "text", "text": "Fix auth.py"}]}],
        )
        for event in stream:
            if event.type == "session.status_idle":
                break
    ```
87. `agent_toolset_20260401` includes: Bash, file ops (read/write/edit/glob/grep), web search, web fetch, MCP servers
88. Rate limits: 60 req/min create endpoints, 600 req/min read/stream endpoints
89. Use Managed Agents when you want zero infrastructure — Anthropic handles sandboxing and tool execution

## Advisor Tool (beta)

90. Advisor Tool pairs a fast executor model with a smarter advisor model mid-generation (beta `advisor-tool-2026-03-01`):
    ```python
    tools = [{
        "type": "advisor_20260301",
        "name": "advisor",                     # must be exactly "advisor"
        "model": "claude-opus-4-6",            # advisor model
        "max_uses": 5,                         # optional per-request cap
    }]
    response = client.beta.messages.create(
        model="claude-sonnet-4-6",             # executor model
        max_tokens=4096,
        betas=["advisor-tool-2026-03-01"],
        tools=tools,
        messages=[{"role": "user", "content": "Design a concurrent worker pool"}],
    )
    ```
91. Valid executor/advisor pairs: haiku+opus, sonnet+opus, opus+opus — invalid pairs return 400
92. Advisor tokens are billed at advisor model rates — appear in `usage.iterations[]`
93. Advisor output does NOT stream — SSE pauses while advisor runs, then result arrives whole
94. In multi-turn: pass `advisor_tool_result` blocks back — removing them mid-conversation causes 400
95. Use Advisor Tool for: complex architecture decisions, hard debugging, security audits — NOT simple tasks

## Tool Search Tool

96. Tool Search Tool enables dynamic tool discovery for large catalogs (10+ tools) — no beta header needed:
    ```python
    tools = [
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        # mark large catalog tools as deferred — not loaded into context until searched
        {"name": "get_weather", "description": "...", "input_schema": {...}, "defer_loading": True},
        {"name": "search_files", "description": "...", "input_schema": {...}, "defer_loading": True},
        # always-loaded tools: no defer_loading
        {"name": "list_files", "description": "...", "input_schema": {...}},
    ]
    ```
97. At least one tool must NOT have `defer_loading` — at least one must always be in context
98. Supports up to 10,000 tools in catalog — use for large MCP server sets (GitHub+Slack+Sentry+Grafana = ~55k tokens → saves 85%)
99. Use `tool_search_tool_bm25` for natural language searches; `tool_search_tool_regex` for pattern matching
100. NEVER add `defer_loading` to the search tool itself — it must always be loaded

## Compaction API (context summarization)

101. Compaction auto-summarizes context when token threshold is hit — prevents context overflow in long agent runs:
     ```python
     runner = client.beta.messages.tool_runner(
         model="claude-sonnet-4-6",
         max_tokens=4096,
         tools=tools,
         messages=messages,
         compaction_control={
             "enabled": True,
             "context_token_threshold": 50000,   # compact when context reaches this size
         },
     )
     final = runner.get_final_message()
     ```
102. Compaction threshold guidelines: 5k–20k for sequential per-item tasks, 50k–100k for general workflows
103. Detecting compaction: if `len(messages)` decreases between turns, compaction occurred
104. Typical savings: 58% token reduction in multi-step agent workflows (208k → 86k tokens)
105. For Opus 4.6: use server-side automatic compaction (no config needed) — preferred over SDK-level

## A2A Protocol (Agent-to-Agent)

106. A2A is Google's open standard (Apache 2.0) for inter-agent communication across frameworks (Claude ↔ Gemini ↔ LangChain etc.)
107. Every A2A agent exposes an **Agent Card** at `GET /v1/agentCard` — declares capabilities, skills, auth scheme
108. Wire format: JSON-RPC 2.0 over HTTPS + SSE for streaming. Install: `pip install a2a-sdk`
109. **Wrap an Anthropic agent as an A2A server:**
     ```python
     from a2a.server import A2AServer, Task
     import anthropic

     claude = anthropic.Anthropic()

     class ClaudeA2AAgent(A2AServer):
         async def handle_message(self, task: Task) -> Task:
             user_text = task.messages[-1].parts[0].text
             response = claude.messages.create(
                 model="claude-sonnet-4-6",
                 max_tokens=2048,
                 messages=[{"role": "user", "content": user_text}]
             )
             task.artifacts = [{"parts": [{"type": "text", "text": response.content[0].text}]}]
             task.state = "completed"
             return task
     ```
110. **Call another A2A agent from your Anthropic code:**
     ```python
     import httpx
     # Discover capabilities
     card = httpx.get("https://other-agent.example.com/v1/agentCard").json()
     # Send task
     result = httpx.post("https://other-agent.example.com", json={
         "jsonrpc": "2.0", "method": "SendMessage", "id": "1",
         "params": {"message": {"role": "user", "parts": [{"type": "text", "text": "..."}]}}
     })
     ```
111. A2A task states: `working` → `completed` | `failed` | `input_required` | `canceled`
112. Use A2A when you need: cross-framework agent calls, published agent services, multi-company agent pipelines
113. There is NO Anthropic SDK native A2A integration — wrap the Messages API yourself in an A2A handler

## tool_choice — Controlling Tool Selection

114. Use `tool_choice` to control whether and how Claude uses tools:
     ```python
     # Default — Claude decides whether to use a tool
     tool_choice={"type": "auto"}

     # Force Claude to call at least one tool (any tool from the list)
     tool_choice={"type": "any"}

     # Force a specific tool by name — Claude MUST call it
     tool_choice={"type": "tool", "name": "get_weather"}
     ```
115. Force one tool at a time — prevent parallel calls with `disable_parallel_tool_use`:
     ```python
     tool_choice={"type": "auto", "disable_parallel_tool_use": True}
     ```
116. `disable_parallel_tool_use=True` makes Claude call one tool per turn — use when tool order matters or each call mutates shared state
117. `tool_choice={"type": "any"}` is useful for structured extraction: force Claude to always return data via a tool schema instead of free text
118. NEVER use `tool_choice={"type": "tool", ...}` in a loop without checking `stop_reason` — if Claude has nothing to call the tool with, it will error

## PDF Document Inputs

119. Pass PDFs as `document` content blocks — separate from images, separate from Files API:
     ```python
     import base64

     # From local file (base64)
     with open("report.pdf", "rb") as f:
         data = base64.standard_b64encode(f.read()).decode("utf-8")
     messages=[{"role": "user", "content": [
         {"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": data}},
         {"type": "text", "text": "Summarize the key findings in this report"}
     ]}]

     # From URL
     messages=[{"role": "user", "content": [
         {"type": "document", "source": {"type": "url", "url": "https://example.com/report.pdf"}},
         {"type": "text", "text": "Extract all action items"}
     ]}]

     # From Files API (upload once, reference many times)
     messages=[{"role": "user", "content": [
         {"type": "document", "source": {"type": "file", "file_id": file_id}},
         {"type": "text", "text": "Find all budget figures"}
     ]}]
     ```
120. PDFs count toward the same 1M context window — a 100-page PDF ≈ 50k–100k tokens
121. Max image media types (jpeg/png/gif/webp) are different from document media type (`application/pdf`) — use correct type for each
122. For repeated PDF analysis across many requests, always use Files API (upload once, save re-upload cost every call)

## Prefilling Assistant Turns

123. Prefill the assistant turn to force output format or continuation:
     ```python
     messages = [
         {"role": "user", "content": "Extract the city from: 'John visited Paris last week'"},
         {"role": "assistant", "content": '{"city": "'},  # Claude continues from here
     ]
     response = await client.messages.create(
         model="claude-sonnet-4-6",
         max_tokens=50,
         messages=messages,
     )
     # response.content[0].text will be: Paris"}
     ```
124. Prefilling is useful for: forcing JSON output, forcing specific response structure, continuing mid-sentence
125. **NOT supported on `claude-opus-4-6`** — prefilling assistant messages causes an error on Opus 4.6
126. NEVER use prefilling in production to bypass safety behaviors — it is for format control only

## Agent Skills (beta)

127. Agent Skills are bundled capabilities (instructions + scripts + resources) Claude loads dynamically (beta `skills-2025-10-02`):
     ```python
     # With Managed Agents — attach at agent creation
     agent = client.beta.agents.create(
         name="Document Processor",
         model="claude-sonnet-4-6",
         system="Process office documents.",
         tools=[{"type": "agent_toolset_20260401"}],
         skills=["excel-20251002", "pdf-20251002", "powerpoint-20251002"],
     )

     # With standalone Messages API
     response = client.beta.messages.create(
         model="claude-sonnet-4-6",
         max_tokens=4096,
         betas=["skills-2025-10-02"],
         tools=[{"type": "code_execution_20250522", "name": "code_execution"}],
         skills=["excel-20251002"],
         messages=[{"role": "user", "content": "Create a monthly sales report in Excel"}],
     )
     ```
128. Built-in Anthropic skills: `excel-20251002`, `powerpoint-20251002`, `word-20251002`, `pdf-20251002`
129. Skills require `code_execution` tool to be present — they generate and execute code internally
130. Custom skills uploadable via `/v1/skills` endpoints — bundle your own instructions + scripts as a reusable skill

## Computer Use

131. Computer use lets Claude control a desktop/browser via screenshots and actions (tools: `computer_20250124`, `bash_20250124`, `text_editor_20250429`):
     ```python
     tools = [
         {
             "type": "computer_20250124",
             "name": "computer",
             "display_width_px": 1280,
             "display_height_px": 800,
             "display_number": 1,
         },
         {"type": "bash_20250124", "name": "bash"},
         {"type": "text_editor_20250429", "name": "str_replace_based_edit_tool"},
     ]
     response = await client.messages.create(
         model="claude-sonnet-4-6",
         max_tokens=4096,
         tools=tools,
         messages=[{"role": "user", "content": "Open Firefox and search for FastAPI docs"}],
     )
     ```
132. Computer use follows the tool use loop — Claude returns `action` blocks, you execute them and return screenshots:
     ```python
     # Claude returns action blocks like:
     # {"type": "tool_use", "name": "computer", "input": {"action": "screenshot"}}
     # {"type": "tool_use", "name": "computer", "input": {"action": "left_click", "coordinate": [760, 400]}}
     # {"type": "tool_use", "name": "computer", "input": {"action": "type", "text": "FastAPI"}}

     # You capture screenshot → base64 encode → return as tool_result with image block
     tool_results = [{
         "type": "tool_result",
         "tool_use_id": block.id,
         "content": [{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": screenshot_b64}}],
     }]
     ```
133. Computer use actions: `screenshot`, `left_click`, `right_click`, `double_click`, `middle_click`, `type`, `key`, `scroll`, `mouse_move`, `cursor_position`, `left_click_drag`
134. ALWAYS run computer use in an isolated VM or container — NEVER on your host machine (security risk)
135. `bash_20250124` tool runs shell commands in a persistent session — state is maintained between calls (env vars, working directory)
136. `text_editor_20250429` uses str_replace for precise file edits — safer than bash redirection for code editing
137. Computer use is slower and more expensive than tool use — only use when you need to interact with real GUIs that have no API
