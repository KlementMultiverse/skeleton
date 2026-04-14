---
description: FastAPI rules. Loads for all FastAPI projects.
paths: ["*.py", "main.py", "app/**/*.py", "routes/**/*.py"]
---

# FastAPI Rules

> Status codes → see `http-contracts.md`
> async def rules → see `async-patterns.md`
> Auth dependency chain → see `auth-patterns.md`
> Pagination and sorting → see `api-design.md`
> CORS and credentials → see `security.md`

---

## Route Handlers

1. Use `status_code=201` on POST routes that create a resource (FastAPI param — rules in `http-contracts.md`)
2. Use `status_code=204` on DELETE routes (FastAPI param — rules in `http-contracts.md`)
3. Use `response_model_exclude_none=True` on routes where null optional fields should not appear in output
4. Use `response_model_exclude_unset=True` when response model has fields that were not explicitly set (different from `exclude_none` — unset means never touched, None means explicitly null)
5. Always declare `response_model=` explicitly — enables automatic serialization, validation, and OpenAPI docs
   — `WWW-Authenticate: Bearer` on 401 → see `http-contracts.md` rule 18

## Dependency Injection

6. Use `Depends()` for auth, DB session, and any shared logic
7. Use `Annotated[type, Depends(fn)]` syntax — cleaner than inline Depends
8. Use `dependency_overrides` in tests — NEVER patch at module level
   ```python
   app.dependency_overrides[get_db] = override_get_db
   ```
9. NEVER open DB connections inside route handlers — use pool from lifespan via Depends
10. Use `app.state` to store app-wide objects (DB engine, LLM client) initialized in lifespan
    ```python
    @asynccontextmanager
    async def lifespan(app: FastAPI):
        app.state.db_engine = create_async_engine(settings.db_url)
        app.state.llm_client = AsyncAnthropic(api_key=settings.anthropic_key)
        yield
        await app.state.db_engine.dispose()
    ```

## Lifespan

11. Use lifespan context manager for DB pool and LLM client init
12. NEVER use `on_event("startup")` or `on_event("shutdown")` — deprecated
    ```python
    from contextlib import asynccontextmanager

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        # startup: init DB engine, LLM client
        yield
        # shutdown: close connections

    app = FastAPI(lifespan=lifespan)
    ```
13. NEVER initialize Anthropic/LLM client inside a route — initialize once in lifespan

## Middleware

14. X-Request-ID middleware is required on every app — rule in `http-contracts.md`
15. CORSMiddleware with explicit origins is required — rule in `security.md`
    ```python
    from fastapi.middleware.cors import CORSMiddleware
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["https://myapp.com"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    ```
16. Middleware added last executes first

## Error Handling

17. Register ALL 3 exception handlers — full templates in `api-design.md`:
    - `RequestValidationError` → 422 RFC 7807 shape
    - `HTTPException` → RFC 7807 shape with fixed `title` per status code
    - bare `Exception` → 500, NEVER leak stack trace
18. All handlers MUST set `media_type="application/problem+json"` — not `application/json`

## Endpoints

19. Always include `GET /health` endpoint — returns `{"status": "ok"}`
20. Use `APIRouter` with `prefix` and `tags` to organize routes by domain
21. Use `StreamingResponse` with async generator for LLM streaming output

## WebSockets

22. Use `@app.websocket("/ws/{id}")` for real-time bidirectional communication
    ```python
    @app.websocket("/ws/{session_id}")
    async def websocket_endpoint(websocket: WebSocket, session_id: str):
        await websocket.accept()
        try:
            while True:
                data = await websocket.receive_text()
                await websocket.send_text(f"Response: {data}")
        except WebSocketDisconnect:
            pass  # client disconnected
    ```
23. Security checks (auth, rate limiting) must be done INSIDE the WebSocket endpoint — HTTP middleware does not run on WebSocket connections
24. LLM streaming to clients: use WebSocket for bidirectional, `StreamingResponse` for unidirectional
    — LLM streaming output via StreamingResponse uses `Content-Type: text/event-stream` (SSE)

## Background Tasks

25. Use `BackgroundTasks` only for fire-and-forget work that can be lost on crash
26. NEVER use `BackgroundTasks` for work that must survive a server crash — use ARQ or Celery
27. `BackgroundTasks` swallow exceptions silently — always wrap the task body in `try/except` and log errors:
    ```python
    def send_email_task(to: str, subject: str) -> None:
        try:
            send_email(to, subject)
        except Exception:
            logger.exception("background_email_failed", to=to)
    ```

## Router-Level Dependencies

28. Apply shared auth/rate-limiting to an entire router via `dependencies=` — no need to repeat on every route:
    ```python
    admin_router = APIRouter(
        prefix="/admin",
        tags=["admin"],
        dependencies=[Depends(require_admin)],  # all routes require admin
    )
    ```
29. NEVER repeat the same `Depends(auth)` on every route in a router — use router-level dependencies; repeated deps are easy to miss when adding new routes

## Request.state — Per-Request Context

30. Use `request.state` to attach per-request data in middleware (request_id, authenticated user) so dependencies deeper in the chain can access it without re-fetching:
    ```python
    # Middleware attaches
    async def auth_middleware(request: Request, call_next):
        user = await get_user_from_token(request)
        request.state.user = user
        return await call_next(request)

    # Dependency reads — no DB call needed
    def current_user(request: Request) -> User:
        return request.state.user
    ```
31. NEVER use `request.state` as a substitute for proper dependency injection — use it only for data that middleware has already resolved; if data needs complex logic, use a `Depends()` dependency

## Testing

32. Use `httpx.AsyncClient` with `ASGITransport(app=app)` for integration tests — no real server needed
33. Always clear `dependency_overrides` after each test — full pattern in `testing-patterns.md`
