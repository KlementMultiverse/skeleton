---
description: Testing rules for async FastAPI backends — pytest-asyncio, DB fixtures, httpx, LLM mocking, vcrpy, coverage.
paths: ["*.py", "tests/**/*.py", "test_*.py", "conftest.py"]
---

# Testing Rules (L8)

> TDD mandate (test first, then code) → see `universal.md` rule 4
> `dependency_overrides` pattern → see `fastapi-patterns.md`
> Async fixture scoping → see `async-patterns.md`

---

## Test Pyramid

### DO
- 70% unit (no I/O, no LLM), 20% integration (real DB, mocked LLM), 10% E2E (recorded LLM)
- Unit tests: test one function in isolation — no DB, no network, no LLM
- Integration tests: real DB with rollback-per-test, LLM always mocked
- E2E tests: use vcrpy to record real LLM responses once, replay forever
- Test behavior (what the code does), not implementation (how it does it)
- Put shared fixtures in `conftest.py` — pytest auto-discovers it, no import needed

### NEVER
- NEVER spin up a DB or call an API in a unit test — it's now an integration test
- NEVER mock the DB in integration tests — mock the LLM instead; DB mocks hide ORM bugs
- NEVER depend on test execution order — each test must be runnable in isolation
- NEVER test private methods or internal implementation details — test the public interface

---

## pytest-asyncio Setup

### DO
- Set `asyncio_mode = "auto"` in `pyproject.toml` — avoids `@pytest.mark.asyncio` on every test
- Use `@pytest_asyncio.fixture` for ALL async fixtures — NOT `@pytest.fixture` (wrong decorator causes errors in newer versions)
- `asyncio_mode = "auto"` uses a session-scoped event loop — required for session-scoped async fixtures to work

### NEVER
- NEVER use `asyncio.get_event_loop()` in fixtures — returns a different loop than pytest-asyncio manages; use `asyncio.get_running_loop()`
- NEVER use sync fixtures that block the event loop (e.g. `time.sleep()`) — use `await asyncio.sleep()` in async fixtures

### pyproject.toml config
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
addopts = "--cov=app --cov-branch --cov-report=term-missing --cov-fail-under=80"

[tool.coverage.run]
omit = ["*/migrations/*", "*/alembic/*", "*/__init__.py"]
```

### Required packages
```
pytest-asyncio    # async test support
pytest-mock       # mocker fixture (AsyncMock, patch)
pytest-recording  # vcrpy wrapper (@pytest.mark.vcr)
httpx             # AsyncClient for route testing
pytest-cov        # coverage reporting
```

---

## Async DB Fixture

### DO
- Create tables at `session` scope — once per test run, not per test (slow)
- Wrap each test in a connection-level transaction — inner `session.commit()` calls flush but do NOT commit to DB
- Roll back the connection-level transaction after each test — zero persistent state
- Set `expire_on_commit=False` on ALL test sessions — prevents `MissingGreenlet` on attribute access after flush
- Override `get_db` with the test session so route handlers use the same transaction

### NEVER
- NEVER use `scope="function"` for `create_all` — recreating tables per test is too slow
- NEVER use simple `session.rollback()` as the only cleanup — if route handler calls `session.commit()`, the outer transaction is committed and rollback is too late
- NEVER leave `dependency_overrides` set after a test — it leaks into subsequent tests

### DB fixture template
```python
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

TEST_DATABASE_URL = "postgresql+asyncpg://test:test@localhost/testdb"

# Session-scoped: create engine + tables once per test run
@pytest_asyncio.fixture(scope="session")
async def engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

# Function-scoped: connection-level transaction, always rolled back
# This works even when route handlers call session.commit() —
# commits flush to the connection but don't escape to the DB
@pytest_asyncio.fixture
async def db(engine):
    conn = await engine.connect()
    trans = await conn.begin()
    session = AsyncSession(bind=conn, expire_on_commit=False)
    yield session
    await session.close()
    await trans.rollback()  # undoes everything, including any session.commit() calls
    await conn.close()

# Override get_db so routes use the test session (same transaction)
@pytest.fixture(autouse=True)
def override_db(db):
    app.dependency_overrides[get_db] = lambda: db
    yield
    app.dependency_overrides.clear()  # autouse=True clears after every test automatically
```

---

## httpx.AsyncClient — Integration Testing Routes

### DO
- Use `httpx.AsyncClient(transport=ASGITransport(app=app), base_url="http://test")` — no real server needed
- Always use `async with` — ensures client is properly closed
- Override `get_current_user` to inject a fake user for authenticated route tests
- Use `autouse=True` fixture to clear `dependency_overrides` after every test automatically
- Test both success AND error paths — assert status codes, response shapes, and RFC 7807 fields

### NEVER
- NEVER share a single client instance across tests — state leaks between tests
- NEVER forget `base_url` — httpx requires it even with in-process transport
- NEVER leave `dependency_overrides` populated after a test — use `autouse=True` teardown

### httpx template
```python
import pytest
import pytest_asyncio
import httpx
from httpx import ASGITransport

@pytest_asyncio.fixture
async def client(db):
    # Inject fake auth user — skips real JWT verification
    app.dependency_overrides[get_current_user] = lambda: User(id=1, role="user")
    async with httpx.AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
    app.dependency_overrides.clear()

# Usage
async def test_create_note(client):
    response = await client.post("/notes", json={"title": "hello"})
    assert response.status_code == 201
    assert response.json()["title"] == "hello"

# Testing exceptions
async def test_note_not_found(client):
    response = await client.get("/notes/99999")
    assert response.status_code == 404
    assert response.json()["title"] == "Resource Not Found"   # RFC 7807 title field

# pytest.raises for unit-level exception testing (not routes)
async def test_raises_not_found():
    with pytest.raises(NotFoundError, match="Note 99 not found"):
        await get_note(id=99, db=...)
```

---

## Mocking LLM Calls

### DO
- Mock at the SDK method level — `mocker.patch("app.services.llm_client.messages.create")`
- Use `AsyncMock` for all async methods — regular `Mock` cannot be awaited
- Return a realistic response object — if code accesses `response.content[0].text`, mock must have that structure
- Test error paths too — mock raising `RateLimitError`, `APIConnectionError` to verify retry/breaker logic fires
- Install `pytest-mock` for the `mocker` fixture — it's not built into pytest

### NEVER
- NEVER mock at the HTTP layer (`httpx.post`) — tests Anthropic's SDK, not your code
- NEVER make real LLM API calls in unit or integration tests — non-deterministic, slow, costs money
- NEVER use regular `Mock()` for async methods — use `AsyncMock`

### Mocking template
```python
from unittest.mock import AsyncMock, MagicMock
from anthropic import RateLimitError

# Mock successful response
def make_llm_response(text: str):
    content_block = MagicMock()
    content_block.text = text
    response = MagicMock()
    response.content = [content_block]
    response.stop_reason = "end_turn"
    return response

async def test_summarize_note(mocker):
    mock_create = mocker.patch(
        "app.services.llm_client.messages.create",
        new_callable=AsyncMock,
        return_value=make_llm_response("A short summary."),
    )
    result = await summarize_note("Long note text...")
    assert result == "A short summary."
    mock_create.assert_called_once()

# Mock error path — verify retry/circuit breaker fires
async def test_summarize_retries_on_rate_limit(mocker):
    mocker.patch(
        "app.services.llm_client.messages.create",
        new_callable=AsyncMock,
        side_effect=RateLimitError(...),
    )
    with pytest.raises(ExternalServiceError):
        await summarize_note("Long note text...")

# Mocking streaming (async context manager)
async def test_stream_summary(mocker):
    async def mock_text_stream():
        yield "Hello"
        yield " world"

    mock_stream = MagicMock()
    mock_stream.__aenter__ = AsyncMock(return_value=mock_stream)
    mock_stream.__aexit__ = AsyncMock(return_value=False)
    mock_stream.text_stream = mock_text_stream()

    mocker.patch("app.services.llm_client.messages.stream", return_value=mock_stream)
    result = await stream_summary("Long note...")
    assert result == "Hello world"
```

---

## vcrpy / pytest-recording

### DO
- Use `pytest-recording` (wraps vcrpy) — simpler decorator-based API
- Commit cassette files to version control — they're YAML, deterministic, shared across team
- Scrub auth headers before committing — cassettes record `Authorization: Bearer sk-ant-...`
- Set `record_mode="none"` in CI — fails loudly if cassette is missing (prevents accidental real API calls)
- Delete and re-record cassettes when prompts change — stale cassettes test the wrong thing
- Use for E2E tests only — unit/integration tests use mocks

### NEVER
- NEVER commit cassettes with real API keys in headers — scrub them in `vcr_config`
- NEVER use vcrpy in unit or integration tests — use mocks there, vcrpy for E2E only
- NEVER forget to re-record when prompts change — test passes but tests wrong behavior

### vcrpy template
```python
# conftest.py — scrub auth headers globally
@pytest.fixture(scope="module")
def vcr_config():
    return {
        "filter_headers": ["authorization", "x-api-key", "anthropic-auth-token"],
        "record_mode": "none",  # fail if cassette missing — prevents surprise real calls in CI
    }

# Usage — first run records, every run after replays
@pytest.mark.vcr
async def test_summarize_real_llm():
    result = await summarize_note("This is a long note about Python async patterns...")
    assert len(result) > 0
    assert len(result) < 500  # summary should be shorter than original
```

---

## Coverage

### DO
- Run with `--cov=app --cov-branch --cov-report=term-missing` — branch coverage catches untested if/else paths
- Set `--cov-fail-under=80` in CI — fails the build if coverage drops below threshold
- Require 100% coverage on auth paths and data mutation paths — these are security-critical
- Exclude boilerplate in `.coveragerc` — migrations, alembic, `__init__.py`
- Use `# pragma: no cover` on intentionally untested lines (e.g. `if __name__ == "__main__":`)

### NEVER
- NEVER treat coverage as a quality metric — a test with no assertions gives 100% line coverage and zero confidence
- NEVER count line coverage without branch coverage — `if condition:` needs both true and false paths tested
- NEVER skip `--cov-fail-under` in CI — without it, coverage silently degrades over time

### CI command
```bash
pytest --cov=app --cov-branch --cov-report=term-missing --cov-fail-under=80
```

---

## @pytest.mark.parametrize

### DO
- Use `@pytest.mark.parametrize` when the same test logic applies to multiple inputs — avoids copy-pasting tests
- Name parametrize IDs explicitly with `ids=` for readable test output

```python
@pytest.mark.parametrize("title,expected_status", [
    ("valid title", 201),
    ("",            422),   # empty title fails validation
    ("a" * 256,     422),   # too long fails validation
], ids=["valid", "empty", "too_long"])
async def test_create_note_validation(client, title, expected_status):
    response = await client.post("/notes", json={"title": title})
    assert response.status_code == expected_status
```

---

## pytest.raises

### DO
- Use `pytest.raises(ExcType, match="pattern")` — verify BOTH exception type AND message
- `match=` is a regex — test the meaningful part of the message, not the whole string

```python
async def test_get_note_not_found():
    with pytest.raises(NotFoundError, match="Note 99"):
        await get_note(id=99, db=session)

async def test_unauthorized_access():
    with pytest.raises(ForbiddenError):
        await get_note(id=1, db=session, current_user=other_user)
```

### NEVER
- NEVER use bare `pytest.raises(Exception)` — too broad, catches everything including bugs
- NEVER test only the exception type without checking the message when two different errors share the same type

---

## Strict Rules (apply to all test code)

- NEVER commit tests with `print()` statements — use `capfd` fixture or logging if output is needed
- NEVER use `time.sleep()` in tests — use `asyncio.sleep()` or mock time
- NEVER write a test that always passes regardless of code changes — every test must be able to fail
- Every test file must be importable standalone — no hidden global state dependencies
- Test names must describe what they test: `test_create_note_returns_201`, not `test_create`
- 100% of auth paths and data mutation paths must have tests — no exceptions
- Every test that hits a route must assert BOTH the status code AND the response body shape
