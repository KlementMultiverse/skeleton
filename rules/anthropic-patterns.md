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
7. Log `usage.input_tokens` and `usage.output_tokens` on EVERY call — needed for cost tracking
8. NEVER put sensitive data in messages — conversation history is logged
9. Use `claude-sonnet-4-6` as default model — balance of speed and quality
   Use `claude-haiku-4-5` for high-volume utility calls (commit messages, summaries)
   Use `claude-opus-4-6` for complex reasoning only

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
26. Cache TTL is 5 minutes — if same content called within 5 min, cache hit (~90% cost reduction)
27. Minimum 1024 tokens required for caching — shorter content is not cached
28. Check `usage.cache_read_input_tokens > 0` to verify cache hit — log cache miss rate

## Structured Output

29. Use `client.messages.parse(output_format=MyModel)` when response shape is known
30. Always use `strict=True` on Pydantic models validating LLM output — see `pydantic-patterns.md`
31. NEVER trust unvalidated LLM output — always parse through Pydantic before saving to DB
32. On `ValidationError`: retry ONCE with error appended to prompt — do NOT retry blindly

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

37. Context window limits by model:
    - `claude-sonnet-4-6`: 200k tokens input
    - `claude-opus-4-6`: 200k tokens input
    - `claude-haiku-4-5`: 200k tokens input
38. Truncate MIDDLE of conversation first — preserve system + first + last messages
39. For coding agents: cap codebase context at 50k tokens — use RAG to select relevant files
40. NEVER send full file contents for files >500 lines — chunk and select relevant sections

## Extended Thinking

41. Enable extended thinking for complex reasoning tasks (math, multi-step logic, hard code review):
    ```python
    response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=16000,
        thinking={"type": "enabled", "budget_tokens": 10000},
        messages=[{"role": "user", "content": "Solve..."}]
    )
    # thinking blocks come BEFORE text blocks in response.content
    for block in response.content:
        if block.type == "thinking":
            # internal reasoning — do NOT send to users
            pass
        elif block.type == "text":
            print(block.text)
    ```
42. `budget_tokens` must be < `max_tokens` — extended thinking uses tokens from the max_tokens budget
43. Thinking blocks MUST be included when you pass prior assistant turns back — strip them to save tokens
44. NEVER display thinking blocks to end users — they are internal reasoning artifacts
45. Use extended thinking ONLY when needed — it costs more tokens and is slower

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

## Coding Agent Specific

57. Agent runs tool loop until `stop_reason == "end_turn"` — no fixed step count
58. Always cap with `max_iterations` guard — NEVER let agent loop without limit
59. Log every tool call: tool name, input, output, duration — full audit trail
60. Sandbox isolation is mandatory for code execution — NEVER run agent-generated code on host
61. Commit message generation: use `claude-haiku-4-5` (cheap, fast, good enough)
62. PR body generation: use `claude-sonnet-4-6` (needs quality reasoning about changes)
