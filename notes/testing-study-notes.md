# Testing — Study Notes (Klement's Plain English Guide)

> Written from first principles. Every concept explained with analogies before code.

---

## Concept 1: The Test Pyramid

**The building inspection analogy:**

A skyscraper has three layers of inspection:
- **Foundation checks** (thousands) — every bolt, every weld. Fast, cheap, done by the crew.
- **Floor inspections** (hundreds) — check rooms connect, plumbing flows. Needs a specialist.
- **Final walkthrough** (a few) — walk through as a real tenant. Slow, expensive, catches the rest.

```
        /\
       /E2E\          ← 10%  real services, slow, expensive
      /------\
     /  Integ  \      ← 20%  real DB, mocked LLM, medium speed
    /------------\
   /    Unit       \  ← 70%  no I/O, no LLM, fast, lots of them
  /________________\
```

**Unit** — one function, pure logic, no DB/network/LLM. Run on every file save.
**Integration** — multiple pieces together. Real DB, LLM mocked. Run before commit.
**E2E** — whole system. Real LLM calls recorded once and replayed. Run before deploy.

### Edge cases

**1. Unit tests have zero I/O** — if you spin up a DB, it's now an integration test.

**2. Don't mock the DB in integration tests** — mock the LLM (costs money, non-deterministic). Use a real test DB. DB mocks hide ORM bugs — you'd be testing your mock, not your queries.

**3. Test behavior, not implementation** — don't assert that `save_note` called `session.add()`. Assert that after `save_note`, the note exists in the DB. If you test implementation, refactoring breaks tests for no reason.

**4. Tests must be independent** — each test runs alone in any order. No test depends on another having run first.

**5. Coverage is a floor, not a goal** — 80% minimum means you tested 80% of code paths. 100% coverage with no assertions = worthless. But <80% = blind spots you don't know about.

**One sentence:** 70% unit (fast, no I/O), 20% integration (real DB, mocked LLM), 10% E2E (recorded responses) — more tests at the bottom = fast feedback loop.

---

## Concept 2: pytest-asyncio — Testing Async Code

**The problem:** Your FastAPI app is async. pytest is sync. You can't `await` inside a regular test.

`pytest-asyncio` runs the event loop for you:

```python
async def test_create_note():   # ← works because of asyncio_mode = "auto"
    result = await create_note(title="hello")
    assert result.id is not None
```

**Auto mode** — add to `pyproject.toml`:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```
Every `async def test_*` runs automatically. No `@pytest.mark.asyncio` needed on each test.

### The fixture scope problem

Fixtures have scope: `function` (new per test) or `session` (one per test run). The event loop must outlive all fixtures using it. With `asyncio_mode = "auto"`, pytest-asyncio uses a session-scoped loop by default — so session-scoped async fixtures work.

### Edge cases

**1. `asyncio_mode = "auto"` is required** — without it you get "coroutine was never awaited" errors.

**2. `@pytest_asyncio.fixture` not `@pytest.fixture`** — for async fixtures you must use the pytest-asyncio decorator. Wrong decorator = warnings or errors in newer versions.

**3. Don't use `asyncio.get_event_loop()`** — may return a different loop. Use `asyncio.get_running_loop()`.

**4. Sync fixtures work in async tests** — but async fixtures cannot be used in sync tests.

**One sentence:** `asyncio_mode = "auto"` makes every `async def test_*` just work — no per-test decorator needed.

---

## Concept 3: Async DB Fixture — Real Database in Tests

**The strategy:** wrap each test in a transaction, roll it back when done. Real DB, real queries, zero persistent state.

### The critical problem: route handlers call `session.commit()`

Simple rollback pattern:
```python
async with session.begin():
    yield session
    await session.rollback()   # ← BROKEN if route called session.commit() first
```
If your route handler calls `session.commit()`, the data is committed to the DB. The subsequent `rollback()` is too late — there's nothing to roll back.

**Fix: connection-level transaction**

```python
conn = await engine.connect()
trans = await conn.begin()                        # open connection-level transaction
session = AsyncSession(bind=conn)

# yield session to test...
# route handlers call session.commit() — this FLUSHES but doesn't escape the connection-level transaction

await trans.rollback()   # ← undoes EVERYTHING, even after session.commit()
```

The key: `session.commit()` inside a connection that has an open transaction flushes data to the connection buffer but doesn't commit to the DB. The connection-level `rollback()` undoes it all.

### Edge cases

**1. Create tables once per test run** — `scope="session"` for the engine. Per-test table creation is slow.

**2. `expire_on_commit=False`** — SQLAlchemy expires objects after commit. In async context, accessing expired objects triggers lazy load → `MissingGreenlet`. Always set this on test sessions.

**3. Override `get_db` in FastAPI** — routes use `Depends(get_db)`. Override it to inject the test session so the route and the test are in the same transaction.

**4. `autouse=True` for cleanup** — use `autouse=True` on the override fixture so dependency_overrides are cleared after every test automatically.

**One sentence:** Use a connection-level transaction that route handler `session.commit()` calls can't escape — rollback at the end undoes everything.

---

## Concept 4: httpx.AsyncClient — Testing Routes

**The postal service analogy:** To test a post office, hand the letter directly to the worker. Same processing, zero travel time.

`httpx.AsyncClient` with `ASGITransport` sends HTTP requests directly to your ASGI app in memory — no server, no port, full route + middleware + dependency chain tested.

```python
async with httpx.AsyncClient(
    transport=ASGITransport(app=app),
    base_url="http://test"
) as client:
    response = await client.post("/notes", json={"title": "hello"})
assert response.status_code == 201
```

### Edge cases

**1. Always `async with`** — client must be closed after use.

**2. `base_url` is required** — even though there's no real network. `"http://test"` is the convention.

**3. Inject fake auth user** — override `get_current_user` with a lambda returning a fake user. Don't make tests depend on JWT logic.

**4. `autouse=True` to clear overrides** — a fixture with `autouse=True` that calls `app.dependency_overrides.clear()` in teardown runs after every test, so you never accidentally leak an override.

**5. Test RFC 7807 shapes** — on error responses, assert `response.json()["title"]` and `response.json()["status"]`, not just the status code.

**One sentence:** `ASGITransport` routes HTTP requests to your app in memory — full integration test without a running server.

---

## Concept 5: Mocking LLM Calls

**The problem:** Real LLM calls are slow, expensive, and non-deterministic. Tests would be flaky and cost money.

**Mock at the SDK method level, not the HTTP level:**
```python
# WRONG — testing Anthropic's SDK
mocker.patch("httpx.AsyncClient.post", ...)

# RIGHT — testing your code
mocker.patch("app.services.llm_client.messages.create", new_callable=AsyncMock, ...)
```

`mocker` comes from `pytest-mock` — install it separately.

`AsyncMock` is required for async methods — regular `Mock` can't be awaited.

### Edge cases

**1. Return realistic shape** — if your code does `response.content[0].text`, the mock must have that structure. Use `MagicMock()` to build it.

**2. Test error paths** — mock `side_effect=RateLimitError(...)` and verify your retry logic actually fires. This is the most important thing to test about LLM integration.

**3. Streaming is a context manager** — `client.messages.stream()` is an `async with` block. Mocking it requires setting `__aenter__` and `__aexit__` on the mock object.

**4. Prevent real calls in non-E2E tests** — add a conftest that patches the client to raise if called without a cassette or mock.

**One sentence:** Mock the SDK method your code actually calls with `AsyncMock` — deterministic, free, and tests your code not the SDK.

---

## Concept 6: vcrpy / pytest-recording — Record Once, Replay Forever

**The tape recorder analogy:** Record a real conversation once. Play it back forever. Same conversation, zero effort.

First run → real API call made, response saved to `cassettes/test_name.yaml`.
Every run after → cassette replayed. No network, no cost, deterministic.

```python
@pytest.mark.vcr
async def test_llm_summarize():
    result = await summarize_note("Long note...")
    assert "summary" in result
```

### Edge cases

**1. Commit cassettes to git** — they're YAML, small, and everyone gets the same replay.

**2. Scrub auth headers** — cassettes record `Authorization: Bearer sk-ant-...`. Configure `vcr_config` to filter them before committing.

**3. `record_mode="none"` in CI** — fails loudly if cassette is missing. Prevents surprise real API calls in CI when someone forgets to commit the cassette.

**4. Re-record when prompts change** — delete the cassette, run once to record the new response. Stale cassette = testing the wrong behavior.

**5. E2E only** — unit and integration tests use mocks. vcrpy is for E2E tests where you want to capture what the real API actually returns.

**One sentence:** vcrpy records real API calls to YAML cassette files on first run, replays them forever — deterministic, free, and accurate to what the real API returned.

---

## Concept 7: Coverage

**The map analogy:** Coverage is a map of roads you've driven. Uncolored roads = code paths you've never executed. Something could be broken there.

```bash
pytest --cov=app --cov-branch --cov-report=term-missing --cov-fail-under=80
```

**Minimums:**
- 80% overall — the floor
- 100% on auth paths — security-critical, every branch must be tested
- 100% on data mutation paths — every create/update/delete path

### Edge cases

**1. Coverage ≠ quality** — a test that calls a function but asserts nothing gives 100% line coverage and zero confidence.

**2. Branch coverage > line coverage** — `if condition:` has two branches. `--cov-branch` catches both. Line coverage only confirms the line ran, not which branch.

**3. `--cov-fail-under=80` in CI** — without it, coverage silently degrades. Each PR drops it 0.5% until you have 40% and no one noticed.

**4. Exclude boilerplate** — migrations, alembic, `__init__.py` don't need coverage. Add to `.coveragerc`.

**One sentence:** `--cov-branch --cov-fail-under=80` in CI catches untested branches and fails the build before they accumulate — coverage is a floor, not a goal.

---

## Concept 8: @pytest.mark.parametrize

When the same test logic applies to many inputs, don't copy-paste the test. Parametrize it:

```python
@pytest.mark.parametrize("title,expected_status", [
    ("valid title", 201),
    ("",            422),   # empty = validation fails
    ("a" * 256,     422),   # too long = validation fails
], ids=["valid", "empty", "too_long"])
async def test_create_note_validation(client, title, expected_status):
    response = await client.post("/notes", json={"title": title})
    assert response.status_code == expected_status
```

`ids=` makes test output readable: `test_create_note_validation[valid]` instead of `test_create_note_validation[valid title-201]`.

---

## Concept 9: pytest.raises

For testing that exceptions are raised correctly:

```python
async def test_get_note_not_found():
    with pytest.raises(NotFoundError, match="Note 99"):
        await get_note(id=99, db=session)
```

`match=` is a regex. Always check it — two different functions might raise `NotFoundError` with different messages, and `match=` is how you verify you got the right one.

**One sentence:** `pytest.raises(Type, match="pattern")` tests both the exception type AND the message — bare `pytest.raises(Exception)` is too broad.
