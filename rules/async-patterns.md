---
description: Python async rules. Loads for all Python projects.
paths: ["*.py", "app/**/*.py", "routes/**/*.py"]
---

# Async Pattern Rules

> Retry logic → see `error-handling.md`
> DB async session → see `database-patterns.md`

---

## Core Rules

1. All FastAPI route handlers must be `async def` — this is the source of truth for this rule
2. All DB calls must be awaited: `await session.execute(...)`
3. All LLM API calls must be awaited: `await client.messages.create(...)`
4. All HTTP calls must use `httpx.AsyncClient` — NEVER `requests` in async context
5. NEVER call `time.sleep()` inside a coroutine — use `await asyncio.sleep()`
6. NEVER run CPU-heavy operations directly in async routes — use `run_in_executor`
7. NEVER use synchronous DB drivers (psycopg2) in async routes — use asyncpg

## Concurrency

8. Use `asyncio.TaskGroup` (Python 3.11+) for structured concurrent operations — preferred over gather
9. NEVER use `asyncio.gather()` when you need structured error propagation
   — `return_exceptions=True` hides errors silently, tasks continue even when others fail
   — `TaskGroup` cancels all tasks on first failure — correct behavior
10. NEVER use `asyncio.gather()` for large batches without a Semaphore — floods external APIs
11. Use `asyncio.Semaphore` to cap concurrent calls to any external service in a loop:
    ```python
    sem = asyncio.Semaphore(10)  # max 10 concurrent LLM calls

    async def call_with_limit(prompt: str) -> str:
        async with sem:
            return await client.messages.create(...)

    results = await asyncio.gather(*[call_with_limit(p) for p in prompts])
    ```

## Event Loop

13. Install `uvloop` in production — 2–4x faster than default event loop
    ```python
    # requirements.txt: uvloop
    import uvloop
    uvloop.install()  # call before app starts
    ```
14. NEVER block the event loop — one blocking call stalls ALL requests simultaneously
15. Understand: async is concurrent (one thread, cooperative), NOT parallel (no CPU speedup)
16. Use `asyncio.get_running_loop().run_in_executor(None, blocking_fn)` to call sync/blocking code from async context
    — FastAPI `def` routes do this automatically; explicit sync functions need it manually
    — NEVER use `asyncio.get_event_loop()` — deprecated since Python 3.10, returns wrong loop in nested contexts
    ```python
    import asyncio
    loop = asyncio.get_running_loop()   # ← correct inside an async function
    result = await loop.run_in_executor(None, some_blocking_function, arg1, arg2)
    ```
17. NEVER swallow `asyncio.CancelledError` — it signals task cancellation; always re-raise it
    ```python
    try:
        await some_coroutine()
    except asyncio.CancelledError:
        # cleanup if needed...
        raise   # MUST re-raise — swallowing it orphans the task
    ```
18. Use `asyncio.timeout(seconds)` (Python 3.11+) to set deadline on any coroutine — raises `TimeoutError` on expiry
    ```python
    async with asyncio.timeout(30.0):   # deadline: 30 seconds
        result = await slow_external_call()
    ```

## Streaming

19. Use `async for` when consuming async generators — NEVER collect streaming output into a list first
20. Use `AsyncGenerator` type hint for streaming return types
    ```python
    from collections.abc import AsyncGenerator

    async def stream_llm() -> AsyncGenerator[str, None]:
        async with client.messages.stream(...) as stream:
            async for text in stream.text_stream:
                yield text
    ```

## ExceptionGroup (Python 3.11+)

21. When multiple tasks in a `TaskGroup` fail, Python raises `ExceptionGroup` — catch it with `except*` syntax (Python 3.11+):
    ```python
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(task_a())
            tg.create_task(task_b())
    except* ValueError as eg:
        for exc in eg.exceptions:
            logger.error("task_failed", error=str(exc))
    except* httpx.HTTPError as eg:
        raise ExternalServiceError("upstream failed") from eg.exceptions[0]
    ```
22. NEVER catch bare `Exception` after `TaskGroup` — it catches `ExceptionGroup` as a whole and hides which tasks failed; use `except*` to handle per-exception-type

## ContextVar — Request-Scoped Data

23. Use `contextvars.ContextVar` to propagate request-scoped values (request_id, user_id) through async call chains without passing them as arguments:
    ```python
    from contextvars import ContextVar

    request_id: ContextVar[str] = ContextVar("request_id", default="")

    # Set in middleware
    async def add_request_id(request: Request, call_next):
        token = request_id.set(request.headers.get("X-Request-ID", str(uuid4())))
        response = await call_next(request)
        request_id.reset(token)
        return response

    # Read anywhere in the async call chain — no parameter passing needed
    async def some_service_fn():
        logger.info("doing work", request_id=request_id.get())
    ```
24. `ContextVar` is coroutine-safe — each async task gets its own copy; it does NOT leak between concurrent requests like a global variable would
