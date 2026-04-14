# skeleton

A growing library of strict architectural rules and study notes for building production-grade, AI-native FastAPI backends.

Every rule file is auto-loaded by Claude Code into any project that matches its path patterns. Clone this into `.claude/rules/` on a new project and the agent follows all rules by default — no repeated instructions.

---

## Rules Files (`/rules`)

| File | Level | Covers |
|---|---|---|
| `http-contracts.md` | L0 | Status codes, URL design, HTTP methods, RFC contracts |
| `async-patterns.md` | L1 | asyncio, event loop, concurrency, TaskGroup, Semaphore |
| `pydantic-patterns.md` | L2 | Validation, LLM output parsing, schema separation |
| `fastapi-patterns.md` | L3 | Routes, DI, lifespan, middleware, error handlers |
| `database-patterns.md` | L4 | SQLAlchemy 2.0 async, migrations, indexes, locking, soft delete |
| `auth-patterns.md` | L5 | JWT, bcrypt, RBAC, refresh tokens, session security |
| `api-design.md` | L6 | Pagination, sorting, idempotency, rate limiting, RFC 7807 |
| `error-handling.md` | L7 | Exception hierarchy, structlog, tenacity retry, circuit breaker |
| `testing-patterns.md` | L8 | pytest-asyncio, DB fixtures, LLM mocking, vcrpy, coverage |
| `security.md` | cross | Secrets, CORS, SQL injection, SSRF, prompt injection, headers |
| `anthropic-patterns.md` | L9 | Messages API, streaming, tool use, prompt caching, error hierarchy |

---

## Study Notes (`/notes`)

Plain-English explanations of each concept — analogy first, technical detail second.

| File | Covers |
|---|---|
| `api-design-study-notes.md` | Pagination, sorting, rate limiting, idempotency |
| `error-handling-study-notes.md` | Exception hierarchy, structured logging, retry, circuit breaker |
| `testing-study-notes.md` | Test pyramid, DB fixtures, mocking LLM, vcrpy, coverage |
| `anthropic-sdk-study-notes.md` | Messages API, tool use loop, streaming, caching, error hierarchy |

---

## How to Use

**On a new project**, copy the rules into `.claude/rules/`:

```bash
cp path/to/skeleton/rules/* .claude/rules/
```

Then tell the agent:

> "Build [project]. Follow all rules in `.claude/rules/`."

The agent reads the rules and applies them without being told twice.

---

## Versioning

A new version is committed after every learning session or rule update.

| Version | What changed |
|---|---|
| v0.1.0 | Initial — L0–L8 rules + study notes |
| v0.2.0 | L9 — Anthropic SDK rules + study notes |
| v0.2.1 | L9 extended — Batches API, Files API, Extended Thinking, Vision |
| v0.2.2 | L9 corrections — 1M context, 1-hour cache TTL, output_config, effort, server-side tools |
| v0.3.0 | L9 complete — MCP integration, multi-agent SDK, Managed Agents, Advisor Tool, Tool Search, Compaction, A2A |
| v0.3.1 | L9 final — tool_choice, PDF inputs, prefilling, Agent Skills, Computer Use (25 concepts, 137 rules) |
| v0.3.2 | L9 fix — sequential renumber 1–141 (duplicate rule numbers 7, 10, 33, 46–48 corrected) |

---

## Roadmap

Rules files will be added as each level is completed:

- [x] L9 — Anthropic SDK (LLM calls, streaming, tool use, prompt caching)
- [ ] L10 — LangGraph (stateful agents, checkpointing, HITL)
- [ ] L11 — RAG + pgvector (embeddings, hybrid search, reranking)
- [ ] L12 — Security deep dive (OWASP API Top 10)
- [ ] L13 — Performance (query optimization, caching)
- [ ] L14 — LLM Observability (Langfuse, tracing, evals)
- [ ] L15 — Production deployment (Docker, health checks, migrations)
- [ ] L16 — Senior patterns (ADRs, system design, cost modeling)
