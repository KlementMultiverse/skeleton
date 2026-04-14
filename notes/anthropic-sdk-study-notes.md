# L9: Anthropic SDK — Complete Study Notes

## The one thing to remember

Every LLM call is: **send messages → get response → decide what to do next.**
Everything else is details on top of this loop.

---

## Concept 1 — Client Initialization

**Analogy:** A database connection pool. You create it once at startup, reuse it everywhere. You never open a new DB connection per request — same for the LLM client.

```python
# CORRECT — once at startup
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.llm = AsyncAnthropic(
        api_key=settings.anthropic_api_key,
        max_retries=3,
        timeout=httpx.Timeout(60.0, connect=5.0)
    )
    yield

# WRONG — never do this
async def my_route():
    client = AsyncAnthropic(...)  # new client every request = connection leak
```

**Why `max_retries=3`?** The SDK retries transient errors automatically. Without it, a single network blip fails your request.

**Why explicit timeout?** Without it, a slow Claude response hangs your server forever. 60s for the full response, 5s to establish the connection.

---

## Concept 2 — Messages API

**Analogy:** A chat conversation stored as a list. You keep appending to it. Claude always sees the full history.

```python
message = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,           # REQUIRED — how long can the reply be?
    system="You are...",       # Claude's instructions (not in messages list)
    messages=[
        {"role": "user",      "content": "Hello"},
        {"role": "assistant", "content": "Hi there!"},
        {"role": "user",      "content": "What is 2+2?"}
    ]
)
```

**`stop_reason` — the most important field:**

| stop_reason | What happened | What to do |
|---|---|---|
| `"end_turn"` | Claude finished | Read `message.content[0].text` |
| `"tool_use"` | Claude wants a tool | Execute tool, send result back |
| `"max_tokens"` | Response cut off | Increase `max_tokens` or shorten input |
| `"model_context_window_exceeded"` | Input too large | Truncate messages and retry |

Always check `stop_reason` before reading content. Assuming it's always `end_turn` is the #1 bug.

The response also includes `stop_details` — a structured version of the same info, useful for logging.

---

## Concept 3 — Tool Use Loop

**Analogy:** You hire a contractor (Claude) to renovate your house. They need a hammer (tool). You give them the hammer. They use it. They might need a drill next. You give them that. They keep working until the job is done.

The loop:
```
You: "Add a docstring to main.py"
Claude: "I need to read main.py first" (tool_use)
You: [reads main.py, sends contents]
Claude: "Now I'll write the updated file" (tool_use)
You: [writes the file]
Claude: "Done. I added a docstring." (end_turn)
```

```python
messages = [{"role": "user", "content": "Add a docstring to main.py"}]

while True:
    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )

    if response.stop_reason == "end_turn":
        break  # done!

    if response.stop_reason == "tool_use":
        # 1. append Claude's response to history
        messages.append({"role": "assistant", "content": response.content})

        # 2. execute each tool Claude requested
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = await execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,   # must match
                    "content": result
                })

        # 3. send results back
        messages.append({"role": "user", "content": tool_results})
        # loop again
```

**This loop IS the coding agent.** The Vercel repos, Stripe Minions — they're all variations of this loop.

---

## Concept 4 — Streaming

**Analogy:** Watching a movie vs. waiting for the full download. Streaming shows you frames as they arrive. Same idea — show Claude's words as they're generated.

```python
async with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a poem about the sea"}]
) as stream:
    async for text in stream.text_stream:
        print(text, end="", flush=True)  # prints word by word

    # AFTER streaming completes — get usage stats
    final = await stream.get_final_message()
    print(f"\nTokens used: {final.usage.input_tokens} in, {final.usage.output_tokens} out")
```

**In FastAPI:**
```python
@app.get("/notes/{id}/summarize")
async def stream_summary(id: int, request: Request):
    client = request.app.state.llm
    note = await get_note(id)

    async def generate():
        async with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=512,
            messages=[{"role": "user", "content": f"Summarize: {note.content}"}]
        ) as stream:
            async for text in stream.text_stream:
                yield text

    return StreamingResponse(generate(), media_type="text/plain")
```

---

## Concept 5 — Token Counting

**Analogy:** Checking if your suitcase fits the airline's weight limit BEFORE going to the airport. Cheaper than paying the overweight fee.

```python
# Check BEFORE the actual call
count = await client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system="You are a coding agent...",
    messages=messages,
    tools=tools
)

print(f"This will cost {count.input_tokens} input tokens")

if count.input_tokens > 180_000:  # 90% of 200k context window
    messages = truncate_middle(messages)
```

**Why this matters for coding agents:**
A full codebase can be millions of tokens. If you naively send everything, you'll:
1. Hit the context limit and get an error
2. Spend $10 on a single request
3. Slow down the response (more tokens = slower)

Count first. Truncate intelligently. Then send.

---

## Concept 6 — Prompt Caching

**Analogy:** Caching in a database. If the same data is requested repeatedly, serve from cache instead of recomputing. Same idea — if the same system prompt is sent repeatedly, Claude serves from cache.

```python
response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": "You are a coding agent. Full codebase:\n\n" + codebase_content,
        "cache_control": {"type": "ephemeral"}   # ← this gets cached
    }],
    messages=[{"role": "user", "content": user_question}]
)

# Check if cache was hit
if response.usage.cache_read_input_tokens > 0:
    print("Cache hit! Saved money.")
```

**The rules:**
- Minimum 1024 tokens to cache (shorter = not worth caching)
- **1 hour TTL** (GA since Aug 2025) — same content within 1 hour = cache hit
- ~90% cost reduction on cache hit
- Cache the system prompt (same every request) ✓
- Don't cache user messages (different every request) ✗

**Automatic caching mode (new, Feb 2026):** Instead of marking individual blocks, put `cache_control` on the whole request and the system manages cache point advancement as the conversation grows. Simpler for multi-turn agents.

**For coding agents:** Cache the codebase. It's the same for all requests in a session.

---

## Concept 7 — Structured Output

**Analogy:** A form vs. a letter. A form forces specific fields. A letter is free text. When you need specific data, use a form (structured output), not a letter (free text).

```python
from pydantic import BaseModel

class CodeReview(BaseModel):
    has_bugs: bool
    severity: str           # "none", "low", "medium", "high", "critical"
    issues: list[str]
    suggested_fix: str | None

response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    output_config={"format": {"type": "json_object", "schema": CodeReview.model_json_schema()}},
    messages=[{"role": "user", "content": f"Review this code:\n{code}"}]
)

import json
review = CodeReview.model_validate_json(response.content[0].text)
if review.has_bugs and review.severity == "critical":
    alert_team()
```

**Breaking change note:** The old `output_format=` parameter was renamed to `output_config=` in Jan 2026. If you see old code using `output_format`, update it.

Claude is forced to return exactly this shape. No parsing, no guessing.

---

## Concept 8 — Error Hierarchy

**Analogy:** Car problems. Flat tire (retry — fixable) vs. engine seized (don't retry — fundamental problem).

```
RETRY (temporary problems):
  RateLimitError (429)    → too many requests, slow down
  APIConnectionError      → network blip, try again
  APITimeoutError         → slow response, try again
  APIStatusError 529      → Claude servers overloaded, wait longer

NEVER RETRY (your fault):
  AuthenticationError     → wrong API key — fix the code
  PermissionDeniedError   → no access — fix the config
  BadRequestError         → bad prompt/params — fix the code
  ValidationError         → Pydantic mismatch — fix the schema
```

```python
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
from anthropic import RateLimitError, APIConnectionError, APITimeoutError

@retry(
    wait=wait_exponential(min=1, max=60),
    stop=stop_after_attempt(3),
    retry=retry_if_exception_type((RateLimitError, APIConnectionError, APITimeoutError))
)
async def call_claude(client, messages):
    return await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=messages
    )
```

---

## Concept 9 — Context Window Management

**Analogy:** A whiteboard. It has finite space. When it fills up, you erase the oldest notes — but you always keep the current task and the most recent work visible.

**Context windows (updated 2026):**
- `claude-sonnet-4-6` → **1M tokens** input (GA, standard pricing)
- `claude-opus-4-6` → **1M tokens** input (GA, standard pricing)
- `claude-haiku-4-5-20251001` → 200k tokens input

```python
def truncate_middle(messages: list, keep_first: int = 1) -> list:
    """
    Remove messages from the middle when context is too large.
    Always keep: first N messages + last message.
    """
    if len(messages) <= keep_first + 1:
        return messages
    # keep first message(s) and last message
    return messages[:keep_first] + messages[-1:]
```

**For coding agents:**
- System prompt = instructions + codebase context (never remove)
- First user message = the original task (never remove)
- Middle messages = tool calls and results from earlier steps (remove these first)
- Last user message = current step (never remove)
- Even with 1M context, cap codebase at 50k — cost scales linearly with tokens sent

---

## Concept 10 — Extended Thinking

**Analogy:** A chess player who works through moves in their head before speaking. The "thinking" is internal — you only hear their conclusion. Extended thinking gives Claude a scratchpad to reason step-by-step before answering.

```python
# Sonnet 4.6 — manual budget (still supported)
response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=[{"role": "user", "content": "Why does this async bug only appear under load?"}]
)

# Opus 4.6 — effort parameter (budget_tokens deprecated on Opus 4.6)
response = await client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    effort="high",   # "low" | "medium" | "high"
    messages=[{"role": "user", "content": "Design the architecture for this agent system"}]
)

for block in response.content:
    if block.type == "thinking":
        pass          # internal reasoning — keep for multi-turn but never show users
    elif block.type == "text":
        print(block.text)   # the actual answer
```

**New: suppress thinking blocks** (Mar 2026) — set `"display": "omitted"` to get faster streaming without thinking content. The signature is still preserved so multi-turn works. Use this in production to reduce latency.

**When to use:**
- Complex debugging (why does this fail under race conditions?)
- Architecture decisions (which design pattern fits best?)
- Security audits (what attack vectors does this code expose?)
- NOT for simple tasks — costs more tokens and is slower

**Key rule:** `budget_tokens` must be less than `max_tokens`. If thinking uses 10k tokens, max_tokens must be >10k.

---

## Concept 11 — Vision (Image Input)

**Analogy:** Showing a mechanic a photo of the broken part instead of trying to describe it. Images transmit information that text can't.

```python
import base64

# From a local file
with open("screenshot.png", "rb") as f:
    data = base64.standard_b64encode(f.read()).decode("utf-8")

response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": [
        {
            "type": "image",
            "source": {"type": "base64", "media_type": "image/png", "data": data}
        },
        {"type": "text", "text": "What errors are shown in this screenshot?"}
    ]}]
)
```

**For coding agents:** Pass screenshots of UI bugs, error dialogs, or browser dev tools output. Claude reads them accurately.

Supported: `image/jpeg`, `image/png`, `image/gif`, `image/webp`. Max 5MB per image, up to 20 per request.

---

## Concept 12 — Batches API

**Analogy:** Sending all your letters in one postal bag vs. driving to the post office for each one. Batches = cheaper, slower, bulk processing.

```python
# Submit 1000 requests at once
batch = await client.messages.batches.create(
    requests=[
        {"custom_id": f"doc-{i}", "params": {
            "model": "claude-haiku-4-5",
            "max_tokens": 256,
            "messages": [{"role": "user", "content": f"Summarize: {doc}"}]
        }}
        for i, doc in enumerate(documents)
    ]
)

# Come back later (processes within 24 hours)
while batch.processing_status != "ended":
    await asyncio.sleep(60)
    batch = await client.messages.batches.retrieve(batch.id)

async for result in await client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        summaries[result.custom_id] = result.result.message.content[0].text
```

**50% cheaper** than real-time. Use for: nightly summaries, bulk classification, dataset labeling, offline analysis. NEVER for anything user-facing.

---

## Concept 13 — Files API

**Analogy:** Uploading a document to Google Drive once vs. emailing it as an attachment every time. Upload once, reference by ID.

```python
# Upload a large file once
with open("large_codebase.txt", "rb") as f:
    file = await client.beta.files.upload(
        file=("large_codebase.txt", f, "text/plain")
    )

# Reference it in many requests without re-sending 100k tokens
response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": [
        {"type": "document", "source": {"type": "file", "file_id": file.id}},
        {"type": "text", "text": "Find all security vulnerabilities"}
    ]}]
)

# Manage files
files = await client.beta.files.list()
await client.beta.files.delete(file.id)
```

**For coding agents:** Upload the entire codebase once per session. Reuse across all turns. Saves bandwidth and is faster than including text in every message.

---

## Concept 14 — Server-Side Tools (GA)

**Analogy:** Pre-built tools at a hardware store vs. making your own tools. You could build a drill yourself, but why? Anthropic gives you pre-built tools you just enable.

```python
# No need to implement these — Anthropic runs them in their infrastructure
tools = [
    {"type": "web_search_20250305", "name": "web_search"},
    {"type": "web_fetch_20250124", "name": "web_fetch"},
    {"type": "code_execution_20250522", "name": "code_execution"},
    {"type": "memory_tool_20250618", "name": "memory_tool"},
]

response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    messages=[{"role": "user", "content": "Search for the latest Python async best practices"}]
)
# Claude calls web_search automatically — you get the result, no server needed
```

**GA since Feb 17, 2026** — no beta headers required anymore.

**When to use over your own tools:**
- You want web search/fetch without running your own servers
- You need sandboxed code execution (safer than your own sandbox)
- You want cross-conversation memory without building a memory DB

**For coding agents:** The `code_execution` tool is the easiest way to safely run LLM-generated code.

---

## Concept 15 — MCP (Model Context Protocol) Integration

**Analogy:** MCP is the USB-C standard for AI tools. Before MCP, every app had its own tool format. With MCP, any AI app can plug into any MCP server without custom code — same protocol, same message format.

**The 6-step flow (from Week 2-3 of the course):**
```
1. INIT     → client spawns MCP server (STDIO) or connects to URL (HTTP/SSE)
2. DISCOVER → tools/list → server returns all tool schemas + descriptions
3. INJECT   → tool schemas go into Claude's context as tools= parameter
4. DECIDE   → Claude sees task + tools, outputs tool_use block
5. ROUTE    → MCP client sends tools/call to server, server executes
6. RETURN   → result becomes tool_result, appended to messages → loop continues
```

**Two integration paths:**

*Path A — Remote server via MCP Connector (beta):*
```python
response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Check weather in Tokyo"}],
    mcp_servers=[{
        "type": "url",
        "url": "https://my-mcp-server.example.com/sse",
        "name": "weather",
    }],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "weather"}],
    betas=["mcp-client-2025-11-20"],
)
```
Anthropic's servers handle the MCP client connection. You don't manage it.

*Path B — Local STDIO server (manual):*
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(command="python", args=["weather_server.py"])
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        # convert to Anthropic format → pass as tools= in messages.create
```

**3 Hard Rules for building MCP servers (from MCP-SOP.md):**
1. **NEVER `print()`** — stdout is the JSON-RPC wire. `print()` corrupts the protocol. Use `logging` to `stderr`.
2. **Always `async def`** for tools that make network calls — use `httpx`, never `requests`
3. **Docstring IS the tool schema** — FastMCP sends your docstring to Claude. It must answer: what does it do, parameter format + example, return format + example, error strings

**Transport choice:**
- STDIO → personal, runs on your machine, one connection at a time
- HTTP/SSE → shared, cloud-hosted, multiple concurrent users

---

## Concept 16 — Claude Agent SDK (Multi-Agent Patterns)

**Analogy:** A law firm. The senior partner (orchestrator) takes the case, assigns research to one associate, drafting to another, both work in parallel, partner reviews the combined output.

Install: `pip install claude-agent-sdk`

**Pattern 1 — Basic agent loop:**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

async for message in query(
    prompt="Review auth.py for security issues",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Glob"],
        permission_mode="acceptEdits",
    ),
):
    if hasattr(message, "content"):
        for block in message.content:
            if hasattr(block, "text"):
                print(block.text)
```

**Pattern 2 — Orchestrator/worker topology:**
```python
from claude_agent_sdk import AgentDefinition

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Agent"],   # "Agent" = can spawn workers
    agents={
        "security-scanner": AgentDefinition(
            description="Scans code for security issues. Use for any security review.",
            prompt="You are a security expert. Find vulnerabilities.",
            tools=["Read", "Grep"],   # workers: read-only is safest
            model="opus",
        ),
        "test-runner": AgentDefinition(
            description="Runs tests and analyzes failures.",
            prompt="You are a test engineer.",
            tools=["Bash", "Read"],
            model="sonnet",
        ),
    },
)
```
Claude decides which worker to call based on each `description`. Workers CANNOT spawn more workers.

**Pattern 3 — Parallel execution:**
```python
security_result, coverage_result = await asyncio.gather(
    run_worker("security-scanner", "Scan src/ for vulnerabilities"),
    run_worker("coverage-analyzer", "Analyze test coverage"),
)
```

**Pattern 4 — Sequential handoff (output of A → input to B):**
```python
research = await collect_result("Research FastAPI auth best practices", tools=["WebSearch"])
code = await collect_result(f"Based on:\n{research}\nImplement JWT auth", tools=["Write","Edit"])
review = await collect_result(f"Review this implementation:\n{code}", tools=["Read","Grep"])
```

**Key rules:**
- Subagents CANNOT spawn subagents — never put `"Agent"` in a worker's tools
- Context passes via prompt string — only channel from orchestrator to worker
- Resume sessions: `ClaudeAgentOptions(resume=session_id)` — worker retains full history

---

## Concept 17 — Managed Agents (beta)

**Analogy:** AWS Lambda for agents. You define what the agent does; Anthropic runs the container, the loop, the tools, the sandboxing. You just stream the results.

```python
# 1. Define the agent once (reusable)
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-sonnet-4-6",
    system="You are a helpful coding assistant.",
    tools=[{"type": "agent_toolset_20260401"}],  # built-in: bash, files, web search
)

# 2. Environment = cloud container config
environment = client.beta.environments.create(
    name="my-env",
    config={"type": "cloud", "networking": {"type": "unrestricted"}},
)

# 3. Session = one running instance
session = client.beta.sessions.create(
    agent=agent.id, environment_id=environment.id, title="Fix the auth bug"
)

# 4. Stream events
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(session.id, events=[{
        "type": "user.message",
        "content": [{"type": "text", "text": "Fix auth.py"}],
    }])
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                print(block.text, end="")
        if event.type == "session.status_idle":
            break
```

`agent_toolset_20260401` includes: Bash, file read/write/edit/glob/grep, web search, web fetch, MCP servers.

**When to use:** You want zero infrastructure — no container management, no tool implementation, no loop management. Anthropic handles all of it.

---

## Concept 18 — Advisor Tool (beta)

**Analogy:** A junior developer (executor) coding fast, occasionally asking a senior architect (advisor) for strategic guidance. The junior handles the routine work; the senior only gets involved for hard decisions.

```python
tools = [{
    "type": "advisor_20260301",
    "name": "advisor",          # must be exactly "advisor"
    "model": "claude-opus-4-6", # advisor = smarter, slower, more expensive
    "max_uses": 5,              # cap per request to control cost
}]

response = client.beta.messages.create(
    model="claude-sonnet-4-6",  # executor = fast, cheap
    max_tokens=4096,
    betas=["advisor-tool-2026-03-01"],
    tools=tools,
    messages=[{"role": "user", "content": "Design a concurrent worker pool in Go"}],
)
```

The executor calls the advisor autonomously when it needs strategic guidance. You don't control when — Claude decides.

**Cost model:** Advisor tokens are billed at Opus rates and appear in `usage.iterations[]`, separate from executor tokens.

**Key rules:**
- Advisor output doesn't stream — SSE pauses while advisor runs
- In multi-turn conversations: always pass `advisor_tool_result` blocks back — removing them causes 400
- Valid pairs: haiku+opus, sonnet+opus, opus+opus

**When to use:** Architecture decisions, complex debugging, security audits — not for simple tasks.

---

## Concept 19 — Tool Search Tool + Compaction API

### Tool Search Tool

**Analogy:** A phone directory. Instead of reading every name aloud to the model (expensive), you give it a search function and it looks up what it needs.

```python
tools = [
    # The search tool — always loaded, never defer
    {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
    # Large catalog — deferred (not in context until searched)
    {"name": "get_weather", "description": "...", "input_schema": {...}, "defer_loading": True},
    {"name": "query_database", "description": "...", "input_schema": {...}, "defer_loading": True},
    # Always loaded (small set, always needed)
    {"name": "list_files", "description": "...", "input_schema": {...}},
]
```

Supports up to 10,000 tools. A typical multi-server MCP setup (GitHub+Slack+Sentry+Grafana+Splunk = ~55k tokens) reduces to <8k tokens — **85% savings**.

Use `tool_search_tool_bm25` for natural language search; `tool_search_tool_regex` for pattern matching.

### Compaction API

**Analogy:** Meeting minutes. Instead of re-reading the full 3-hour meeting transcript every time, you refer to the 1-page summary.

```python
runner = client.beta.messages.tool_runner(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    messages=messages,
    compaction_control={
        "enabled": True,
        "context_token_threshold": 50000,  # compact when context hits this
    },
)
final = runner.get_final_message()
```

Real savings: 208k tokens → 86k tokens (58% reduction) in a multi-step agent workflow. For Opus 4.6, server-side automatic compaction is preferred (no config needed).

---

## Concept 20 — A2A Protocol (Agent-to-Agent)

**Analogy:** REST APIs let services talk to services. A2A lets agents talk to agents — same idea, but the "service" is another AI agent.

A2A (Google, Apache 2.0, April 2025) is the open standard for agents built on different frameworks to communicate. A Claude agent can call a Gemini agent and vice versa.

**Agent Card** — every A2A agent publishes at `GET /v1/agentCard`:
```json
{
  "name": "weather-agent",
  "skills": [{"id": "current-weather", "name": "Get Current Weather"}],
  "capabilities": {"streaming": true},
  "url": "https://weather-agent.example.com"
}
```

**Wrapping Anthropic as an A2A server:**
```python
from a2a.server import A2AServer, Task
import anthropic

claude = anthropic.Anthropic()

class ClaudeA2AAgent(A2AServer):
    async def handle_message(self, task: Task) -> Task:
        response = claude.messages.create(
            model="claude-sonnet-4-6", max_tokens=2048,
            messages=[{"role": "user", "content": task.messages[-1].parts[0].text}]
        )
        task.artifacts = [{"parts": [{"type": "text", "text": response.content[0].text}]}]
        task.state = "completed"
        return task
```

**Calling another A2A agent from your code:**
```python
import httpx
result = httpx.post("https://other-agent.example.com", json={
    "jsonrpc": "2.0", "method": "SendMessage", "id": "1",
    "params": {"message": {"role": "user", "parts": [{"type": "text", "text": "your task"}]}}
})
```

**Wire format:** JSON-RPC 2.0 over HTTPS. Streaming via SSE. Task states: `working` → `completed` | `failed` | `input_required`.

**When to use:** Cross-framework agent pipelines, published agent services, multi-company integrations. There is no Anthropic SDK native A2A support — you wrap the Messages API yourself.

---

## Concept 21 — tool_choice and disable_parallel_tool_use

**Analogy:** A manager telling an employee "you MUST use the company template for this report" vs "use whatever works." `tool_choice` is how you enforce or relax that constraint.

```python
# auto (default) — Claude decides
tool_choice={"type": "auto"}

# any — Claude MUST use at least one tool
tool_choice={"type": "any"}

# tool — Claude MUST use this specific tool
tool_choice={"type": "tool", "name": "extract_data"}

# Prevent parallel tool calls — one tool per turn
tool_choice={"type": "auto", "disable_parallel_tool_use": True}
```

**When to use each:**
- `auto` → normal agent use. Claude decides.
- `any` → structured extraction. Force Claude to always return data via a tool schema instead of prose.
- `"tool"` → when you need a specific tool invoked unconditionally (e.g., always log via `audit_tool` before proceeding)
- `disable_parallel_tool_use` → when tools mutate shared state or order matters (e.g., create user → then create profile, not simultaneously)

---

## Concept 22 — PDF Document Inputs

**Analogy:** Handing Claude a printed report vs. reading it aloud. PDFs are documents — a different content type from images and plain text.

```python
import base64

# Local PDF
with open("annual_report.pdf", "rb") as f:
    data = base64.standard_b64encode(f.read()).decode("utf-8")

response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{"role": "user", "content": [
        {"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": data}},
        {"type": "text", "text": "Extract all budget figures and action items"},
    ]}]
)

# Or from a URL
messages=[{"role": "user", "content": [
    {"type": "document", "source": {"type": "url", "url": "https://example.com/report.pdf"}},
    {"type": "text", "text": "Summarize the key findings"},
]}]
```

Three source types: `base64`, `url`, `file` (Files API). Use Files API when you'll reference the same PDF in many requests.

A 100-page PDF ≈ 50k–100k tokens. Always count tokens before sending large PDFs.

---

## Concept 23 — Prefilling Assistant Turns

**Analogy:** Filling in the first words of a sentence and asking someone to finish it. The context shapes the continuation.

```python
messages = [
    {"role": "user", "content": "Return the city as JSON. Input: 'John visited Paris last week'"},
    {"role": "assistant", "content": '{"city": "'},  # Claude continues from here
]
# Claude will complete: Paris"}
```

**What it's useful for:**
- Force JSON output without schema enforcement
- Force Claude to start inside a code block: `{"content": "```python\n"}`
- Skip preambles ("Sure, I'd be happy to...") by starting with the actual answer

**Critical limitation:** Prefilling is **NOT supported on `claude-opus-4-6`**. Sending an assistant turn prefix to Opus 4.6 returns an error. Only use with Haiku and Sonnet models.

---

## Concept 24 — Agent Skills (beta)

**Analogy:** Installing a plugin. A skill is a bundled package (instructions + scripts) Claude can activate to gain a specialized capability.

```python
# With Managed Agents
agent = client.beta.agents.create(
    name="Office Document Agent",
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
    messages=[{"role": "user", "content": "Build a sales dashboard in Excel with this data: ..."}],
)
```

**Built-in Anthropic skills:** `excel-20251002`, `powerpoint-20251002`, `word-20251002`, `pdf-20251002`

Skills require the `code_execution` tool — they work by generating and running code internally. You can also upload custom skills via `/v1/skills`.

---

## Concept 25 — Computer Use

**Analogy:** Screen sharing with an intern who can actually control your mouse. You show them the screen, they click and type, you see the result.

Computer use gives Claude control of a real desktop GUI via a screenshot → action → screenshot loop.

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

# The loop: Claude requests action → you execute → send screenshot back
while True:
    response = await client.messages.create(
        model="claude-sonnet-4-6", max_tokens=4096,
        tools=tools, messages=messages,
    )
    if response.stop_reason == "end_turn":
        break
    for block in response.content:
        if block.type == "tool_use" and block.name == "computer":
            # execute the action (click, type, scroll, etc.)
            screenshot = execute_action(block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": [{"type": "image", "source": {
                    "type": "base64", "media_type": "image/png", "data": screenshot
                }}],
            })
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": tool_results})
```

**Available actions:** `screenshot`, `left_click`, `right_click`, `double_click`, `type`, `key`, `scroll`, `mouse_move`, `left_click_drag`

**Three tools, three purposes:**
| Tool | Purpose |
|---|---|
| `computer_20250124` | Mouse + keyboard control via screenshots |
| `bash_20250124` | Shell commands, persistent session (env vars survive between calls) |
| `text_editor_20250429` | Precise str_replace file edits — safer than bash for code |

**Security:** ALWAYS run in an isolated VM or container. NEVER on your host machine. Claude can execute arbitrary actions on whatever desktop it sees.

**When to use:** Only when the target has no API. If there's an API, use tool use (faster, cheaper, more reliable). Computer use is the last resort for legacy GUIs.

---

## How all 25 concepts connect in the coding agent

```
CORE SDK
1.  Client init        → AsyncAnthropic once at startup; max_retries=3; explicit timeout
2.  Messages API       → always check stop_reason (end_turn/tool_use/max_tokens/exceeded)
3.  Tool use loop      → while stop_reason=="tool_use": execute → append → send back
4.  Streaming          → token by token; get_final_message() for usage stats
5.  Token counting     → count before large calls; 90% threshold → truncate middle
6.  Prompt caching     → 1-hour TTL, min 1024 tokens; auto-caching mode available
7.  Structured output  → output_config={"format": {...}} — returns Pydantic-shaped JSON
8.  Error handling     → RETRY: 429/connection/timeout/529. NEVER: 400/401/403/validation
9.  Context mgmt       → 1M tokens (Sonnet/Opus 4.6); truncate middle; cap codebase at 50k

ADVANCED SDK
10. Ext. thinking      → effort= for Opus 4.6; budget_tokens= for Sonnet; display="omitted"
11. Vision             → base64 or URL images as content blocks; up to 20 per request
12. Batches API        → 50% cheaper; 24hr processing; bulk classification / nightly jobs
13. Files API          → upload once, reference by file_id across many requests (beta)
14. Server-side tools  → GA: web_search, web_fetch, code_execution, memory_tool

AGENTIC LAYER
15. MCP integration    → 6-step flow; Connector for remote; mcp package for STDIO; never print()
16. Agent SDK          → orchestrator/worker; parallel asyncio.gather; sequential handoffs
17. Managed Agents     → zero-infra hosting; agent+environment+session+events (beta)
18. Advisor Tool       → executor + advisor model mid-generation; haiku/sonnet + opus (beta)
19. Tool Search+Compact → defer_loading for large catalogs; compaction for long runs (beta)
20. A2A Protocol       → wrap Messages API in A2A handler; JSON-RPC 2.0 inter-agent calls

INPUT / CONTROL
21. tool_choice        → auto/any/tool/disable_parallel — control when + how tools are called
22. PDF inputs         → document content block; base64/url/file_id; ~50-100k tokens per PDF
23. Prefilling         → force output format by starting assistant turn; NOT on Opus 4.6
24. Agent Skills       → bundled capabilities (excel/pdf/pptx); requires code_execution (beta)
25. Computer use       → screenshot→action loop; bash+editor tools; ALWAYS in isolated VM
```

This is the complete Anthropic SDK + agentic layer. All 25 concepts. 137 rules.
