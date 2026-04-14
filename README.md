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
| `langgraph-patterns.md` | L10 | StateGraph, reducers, checkpointing, HITL, streaming, Send, Command |
| `rag-patterns.md` | L11 | Embeddings, pgvector, chunking, hybrid search, reranking, RAGAS |
| `security-complete.md` | L12 | OWASP API Top 10, JWT attacks, SSRF, CORS, encryption, RLS, DoS |
| `performance-patterns.md` | L13 | Profiling, index design, caching, LLM cost, ORJSONResponse, uvloop |
| `observability-patterns.md` | L14 | Langfuse, LangSmith, cost attribution, RAGAS, LLM-as-judge, OTel |
| `deployment-patterns.md` | L15 | Docker multi-stage, health checks, Gunicorn, migrations, CI/CD |
| `senior-patterns.md` | L16 | ADRs, system design, technical debt, LLM cost modeling, DORA metrics |
| `voice-patterns.md` | L17 | Latency budget, audio formats, Ultravox, VAD, telephony, PCI-DSS, GDPR, SLOs |
| `memory-patterns.md` | L18 | Mem0, Qdrant HNSW, memory layers, injection, extraction, GDPR erasure, multi-agent |
| `distributed-patterns.md` | L19 | Matrix/Synapse, federation, event DAG, Application Services, event-driven, Kafka, Saga |
| `tenancy-patterns.md` | L20 | RLS, family/group data, RBAC, invite system, shared AI memory, GDPR, multi-tenancy |
| `aws-patterns.md` | L21 | IAM Identity Center, CDK v2, ECS Fargate Graviton4, Aurora Serverless v2, Valkey, Bedrock Converse API, awslabs MCPs |
| `gcp-patterns.md` | L22 | IAM/WIF, Cloud Run Gen2, AlloyDB AI, Vertex AI Gemini 2.0, Pub/Sub, Terraform CFT, GCP MCP servers |

---

## Study Notes (`/notes`)

Plain-English explanations of each concept — analogy first, technical detail second.

| File | Covers |
|---|---|
| `api-design-study-notes.md` | Pagination, sorting, rate limiting, idempotency |
| `error-handling-study-notes.md` | Exception hierarchy, structured logging, retry, circuit breaker |
| `testing-study-notes.md` | Test pyramid, DB fixtures, mocking LLM, vcrpy, coverage |
| `anthropic-sdk-study-notes.md` | Messages API, tool use loop, streaming, caching, error hierarchy |
| `langgraph-study-notes.md` | StateGraph, reducers, checkpointing, HITL, streaming, Send, Command |
| `rag-pgvector-study-notes.md` | Embeddings, chunking, hybrid search, reranking, RAGAS evaluation |
| `security-study-notes.md` | OWASP API Top 10, JWT algorithm confusion, SSRF, encryption, audit logging |
| `performance-study-notes.md` | Profiling, query optimization, caching strategies, LLM cost, measurement |
| `llm-observability-study-notes.md` | Langfuse tracing, cost attribution, RAGAS evals, LLM-as-judge, OTel |
| `production-deployment-study-notes.md` | Docker, health checks, Gunicorn, zero-downtime migrations, CI/CD |
| `senior-patterns-study-notes.md` | ADRs, system design, tech debt, LLM cost, code review, event-driven, DORA |
| `voice-study-notes.md` | Latency budget, audio formats, Ultravox, VAD/barge-in, telephony, production architecture |
| `ai-memory-study-notes.md` | Memory layers, Mem0 API, Qdrant, injection, extraction, privacy, pgvector vs Qdrant |
| `distributed-systems-study-notes.md` | CAP/PACELC, Raft, Matrix protocol, Synapse, AS integration, event sourcing, Kafka |
| `multi-user-tenancy-study-notes.md` | Tenancy models, RLS deep dive, family data model, RBAC, invites, shared AI memory |
| `aws-study-notes.md` | IAM Identity Center, CLI v2/SSO, CDK v2 Pipelines, compute selection, Bedrock Converse API, cross-region inference, awslabs MCPs |
| `gcp-study-notes.md` | gcloud CLI/ADC, Cloud Run Gen2, AlloyDB AI pgvector, Vertex AI Gemini 2.0 Flash, Pub/Sub, Terraform, GCP MCP servers |

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
| v0.4.0 | L10 — LangGraph (StateGraph, reducers, checkpointing, HITL, streaming, Send, Command, RetryPolicy) |
| v0.4.7 | L10 complete — 161 rules: full control design principles, superstep isolation, streaming+HITL, durable agents, reflexion, tool routing, determinism |
| v0.5.0 | L10 final — 194 rules: compile-once, graceful shutdown, runtime model config, streaming tests, duplicate fix |
| v0.6.0 | L11 — RAG + pgvector (embeddings, chunking, hybrid search, reranking, RAGAS, 87 rules) |
| v0.6.1 | L11 fix — ef_construction 64→200, halfvec storage, HyDE, async embedding clients, ef_search pool fix |
| v0.7.0 | L0-L8 gap fill — webhooks, cache-control, ExceptionGroup, ContextVar, SecretStr, AliasGenerator, .returning(), jwt aud, bulk ops, Sentry, factory_boy, respx |
| v0.8.0 | L12 — Security OWASP (BOLA, mass assignment, JWT alg confusion, SSRF DNS, CORS, field encryption, RLS, DoS, 85 rules) |
| v0.8.6 | L12 final — 188 rules after 6 iterations (kid/jku JWT, BFLA, Argon2id, WebSocket, SSTI, container, HTTP smuggling, deserialization, STRIDE) |
| v0.9.0 | L13 — Performance (profiling, index design, caching, LLM cost/caching, ORJSONResponse, uvloop, 110 rules) |
| v0.9.6 | L13 final — 146 rules after 6 iterations (pg_stat_statements, nplusone, SKIP LOCKED, keyset pagination, async worker tuning, embedding batch sizing) |
| v1.0.0 | L14 — LLM Observability (Langfuse, LangSmith, cost attribution, RAGAS, LLM-as-judge, OTel, 135 rules) |
| v1.1.0 | L15 — Production Deployment (Docker multi-stage, health checks, Gunicorn, zero-downtime migrations, CI/CD, 130 rules) |
| v1.2.0 | L16 — Senior Patterns (ADRs, system design, tech debt, LLM cost, DORA, chaos engineering, 224 rules) |
| v1.3.0 | L17 — Real-Time Voice (latency budget, Ultravox, VAD, telephony, PCI-DSS, GDPR, SLOs, 199 rules) |
| v1.4.0 | L18 — AI Memory (Mem0, Qdrant HNSW, memory layers, injection, extraction, GDPR erasure, multi-agent, 159 rules) |
| v1.5.0 | L19 — Distributed Systems (Matrix/Synapse, federation, event DAG, AS integration, Kafka, Saga, CQRS, 166 rules) |
| v1.6.0 | L20 — Multi-User Family Tenancy (RLS, family data model, RBAC, invite system, shared AI memory, GDPR, 158 rules) |
| v1.7.0 | L21 — AWS (IAM Identity Center, CDK v2 Pipelines, ECS Fargate Graviton4, Aurora Serverless v2, Valkey, Bedrock Converse API, cross-region inference, awslabs MCP servers, 227 rules) |
| v1.8.0 | L22 — GCP (Workload Identity Federation, Cloud Run Gen2, AlloyDB AI, Vertex AI Gemini 2.0 Flash, Pub/Sub, Terraform CFT, GCP MCP servers, 205 rules) |

---

## Roadmap

Rules files will be added as each level is completed:

- [x] L9 — Anthropic SDK (LLM calls, streaming, tool use, prompt caching)
- [x] L10 — LangGraph (stateful agents, checkpointing, HITL)
- [x] L11 — RAG + pgvector (embeddings, hybrid search, reranking)
- [x] L12 — Security deep dive (OWASP API Top 10)
- [x] L13 — Performance (query optimization, caching, LLM cost)
- [x] L14 — LLM Observability (Langfuse, tracing, evals)
- [x] L15 — Production deployment (Docker, health checks, migrations)
- [x] L16 — Senior patterns (ADRs, system design, cost modeling)
- [x] L17 — Real-time voice (Ultravox, WebRTC, latency budgets, telephony)
- [x] L18 — AI Memory (Mem0, Qdrant, memory layers, injection, GDPR erasure)
- [x] L19 — Distributed Systems (Matrix/Synapse, federation, event DAG, Kafka, Saga)
- [x] L20 — Multi-User Family Tenancy (RLS, family data model, RBAC, invite system, shared AI memory)
- [x] L21 — AWS (IAM Identity Center, CDK v2 Pipelines, ECS Fargate Graviton4, Aurora Serverless v2, Valkey, Bedrock Converse API, awslabs MCPs)
- [x] L22 — GCP (gcloud CLI/WIF, Vertex AI Gemini 2.0, Cloud Run Gen2, AlloyDB AI, Pub/Sub, Terraform CFT, GCP MCPs)
- [ ] L23 — Azure (az CLI, Azure OpenAI, Container Apps, Blob Storage, Service Bus, Cosmos DB, Terraform, Azure MCPs)
