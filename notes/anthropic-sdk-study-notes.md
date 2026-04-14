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

Always check `stop_reason` before reading content. Assuming it's always `end_turn` is the #1 bug.

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
- 5 minute TTL — same content within 5 min = cache hit
- ~90% cost reduction on cache hit
- Cache the system prompt (same every request) ✓
- Don't cache user messages (different every request) ✗

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

result = client.messages.parse(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    output_format=CodeReview,
    messages=[{"role": "user", "content": f"Review this code:\n{code}"}]
)

review = result.parsed_output
if review.has_bugs and review.severity == "critical":
    alert_team()
```

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

---

## How all 9 concepts connect in the coding agent

```
1. Client init     → AsyncAnthropic created once at startup
2. Messages API    → send task + conversation history
3. Tool use loop   → read files, write files, run commands
4. Streaming       → show progress to user in real-time
5. Token counting  → check codebase fits in context before sending
6. Prompt caching  → cache codebase content (same across turns)
7. Structured out  → get structured result (files changed, PR title)
8. Error handling  → retry on rate limits, fail fast on bad requests
9. Context mgmt    → truncate old tool results when context fills up
```

This is the complete Anthropic SDK layer of the coding agent.
