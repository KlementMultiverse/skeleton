# AI Memory Architecture — L18: Mem0, Qdrant, Layered Memory

> Stack: Python 3.12, Mem0 0.1+, Qdrant 1.9+, FastAPI, LangGraph.
> Last verified: April 2026.
> AI connection: RAG (L11) gives Claude access to documents. Memory gives Claude access to *who the user is*. RAG is about content. Memory is about identity and context persistence across sessions.

---

## The one thing to remember

Without memory, every conversation starts from zero. With memory, the agent knows you prefer concise answers, hate CLI tools, and asked about Postgres indexing three months ago. Memory is the difference between a stateless API and a persistent collaborator.

---

## Part 1 — Memory Layer Taxonomy

### Why Layers Exist

No single storage technology is optimal for all memory types. The tradeoffs are:

| Property | In-Process Dict | Redis | PostgreSQL | Vector DB |
|---|---|---|---|---|
| Read latency | <1ms | 1-5ms | 5-50ms | 10-100ms |
| Durability | None (lost on restart) | Optional (RDB/AOF) | Full ACID | Collection-level |
| Searchability | Exact key only | Exact key only | SQL + full-text | Semantic similarity |
| Max practical size | RAM (< 1GB) | RAM (< 100GB) | Disk (TB+) | Disk (TB+) |
| Best for | Working state | Session cache | Structured facts | Semantic search |

Different memory types have different needs. You pick the right storage for each layer.

---

### Layer 1 — Short-Term (Working Memory)

**What it is:** The current conversation's messages. In LangGraph terms, the `messages` key in your `State` TypedDict. Lives in memory during the request. Persisted to a checkpointer (SQLite dev / PostgreSQL production) so the user can continue tomorrow.

**Storage:** LangGraph checkpointer. The full message list is serialized to the checkpoint store after each graph step.

**Lifespan:** One conversation thread. Scoped by `thread_id`. Cleared when the thread is deleted.

**Why not vector DB:** You don't search working memory — you read it sequentially in order. A vector similarity search on 20 messages is wasteful. You need the whole list, not the most relevant parts.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import BaseMessage

class State(TypedDict):
    # Annotated with add_messages reducer: new messages append, never overwrite
    messages: Annotated[list[BaseMessage], add_messages]
    summary: str  # compressed history when messages list gets long

def chat_node(state: State) -> dict:
    # state["messages"] is the FULL conversation history for this thread
    # last message is the current user turn
    last_user_msg = state["messages"][-1].content
    response = call_llm(state["messages"])
    return {"messages": [response]}

builder = StateGraph(State)
builder.add_node("chat", chat_node)
builder.add_edge(START, "chat")
builder.add_edge("chat", END)
```

**Practical limit:** Keep working memory under ~50 messages. Beyond that, inject a summarization node that compresses old messages into a `summary` field and trims `messages` to the last 10.

```python
def maybe_summarize(state: State) -> dict:
    if len(state["messages"]) < 20:
        return {}  # nothing to do

    to_summarize = state["messages"][:-5]
    existing_summary = state.get("summary", "")

    summary_prompt = f"""Previous summary: {existing_summary}

New messages to incorporate:
{format_messages(to_summarize)}

Produce a concise 3-sentence summary of what has been discussed."""

    new_summary = call_llm_simple(summary_prompt)

    return {
        "messages": state["messages"][-5:],  # keep only recent
        "summary": new_summary,
    }
```

---

### Layer 2 — Long-Term Semantic Memory

**What it is:** Extracted facts about the user stored as embeddings in a vector database. "Alice prefers Python over JavaScript." "Bob is allergic to nut examples in code." "Carol has 5 years of FastAPI experience."

**Storage:** Vector database (Qdrant in production, pgvector for simpler deployments). Each fact is a text string embedded into a float vector, stored with `user_id` in the payload.

**Lifespan:** Indefinite. Persists across all sessions. Deleted only via explicit GDPR wipe.

**Why vector DB:** You search these memories by semantic relevance to the current query. "Recommend a Python framework" surfaces "user prefers asyncio-based stacks." You can't SQL-query this — you need similarity search.

**Access pattern:** Write on extraction (async, after response). Read on session start (retrieve top-K relevant to current topic).

---

### Layer 3 — Episodic Memory

**What it is:** Compressed summaries of past conversations. Not the raw messages, but a higher-level narrative. "In the March session, we debugged a Redis deadlock. The user was frustrated. Resolution: upgraded to Redis 7.2."

**Storage:** PostgreSQL (structured rows) plus optional vector index for semantic retrieval over episodes.

**Lifespan:** Persistent but degradable. Episodes more than a year old may be compressed further or archived.

**Why different from semantic memory:** Semantic memory stores atomic facts ("user likes Python"). Episodic memory stores narratives ("in session X, the user and agent solved problem Y"). Episodes give context to facts: you know not just *what* the user knows, but *what they've struggled with*.

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Text, DateTime, Integer, func
from sqlalchemy.dialects.postgresql import ARRAY
from pgvector.sqlalchemy import Vector
from datetime import datetime

class EpisodicMemory(Base):
    __tablename__ = "episodic_memories"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[str] = mapped_column(String(255), index=True, nullable=False)
    session_id: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=func.now())
    summary: Mapped[str] = mapped_column(Text, nullable=False)
    topics: Mapped[list[str]] = mapped_column(ARRAY(Text), nullable=False)
    sentiment: Mapped[str] = mapped_column(String(50), nullable=True)
    outcome: Mapped[str] = mapped_column(Text, nullable=True)
    # Optional: embed summary for semantic search over past episodes
    summary_embedding: Mapped[list[float]] = mapped_column(Vector(1024), nullable=True)
```

---

### Layer 4 — Procedural Memory

**What it is:** How-to knowledge. Rules. Preferences for HOW things are done. "Always respond in the user's native language." "Never use bullet points in code explanations." "Format all SQL with uppercase keywords."

**Storage:** System prompts and rules files. Not a database — text files read at startup and injected into every prompt. Advanced: a `user_rules` table fetched in full at session start.

**Lifespan:** Until explicitly changed. Versioned in git.

**Why not a database (for most cases):** Procedural memory is not searched — it applies universally. You always inject ALL procedural rules for a user, not a semantic subset. A flat file or a full table scan is correct.

```python
PROCEDURAL_MEMORY_TEMPLATE = """
## Your Instructions for this User

Communication style:
{communication_rules}

Technical preferences:
{technical_rules}

Explicit requests:
{user_explicit_rules}
"""

async def build_procedural_system_prompt(user_id: str, db: AsyncSession) -> str:
    result = await db.execute(
        select(UserRule).where(UserRule.user_id == user_id)
    )
    rules = result.scalars().all()

    comm_rules = [r.content for r in rules if r.category == "communication"]
    tech_rules = [r.content for r in rules if r.category == "technical"]
    explicit_rules = [r.content for r in rules if r.category == "explicit"]

    return PROCEDURAL_MEMORY_TEMPLATE.format(
        communication_rules="\n".join(f"- {r}" for r in comm_rules) or "None set.",
        technical_rules="\n".join(f"- {r}" for r in tech_rules) or "None set.",
        user_explicit_rules="\n".join(f"- {r}" for r in explicit_rules) or "None set.",
    )
```

---

### Layer Summary

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer           Storage            Scope       TTL              │
├──────────────────────────────────────────────────────────────────┤
│  Working memory  LangGraph state    Thread      Session          │
│  Semantic facts  Vector DB (Qdrant) User        Indefinite       │
│  Episodic        PostgreSQL + vec   User        Long (1yr+)      │
│  Procedural      System prompt/DB   User/Global Indefinite       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Part 2 — Mem0: Python Memory Library

### What Mem0 Does

Mem0 is a memory abstraction layer. You hand it a conversation turn (a list of messages), and it:

1. Runs an LLM (default: gpt-4o-mini) with a prompt asking "what facts about this user are worth remembering?"
2. Extracts candidate memories as text strings
3. Embeds each candidate and checks cosine similarity against existing memories for that user (deduplication at threshold ~0.95)
4. Writes new or updated memories to the configured vector store

On retrieval, you give it a query string and `user_id`. It embeds the query and runs ANN search over that user's stored memories.

**What Mem0 is NOT:** It is not a conversation store. It stores extracted facts derived from messages. Raw messages stay in your LangGraph checkpointer.

---

### Installation

```bash
pip install mem0ai
pip install qdrant-client        # Qdrant backend
pip install "mem0ai[async]"      # Async support
```

---

### Core API: In-Memory (Development)

```python
from mem0 import Memory

# Default: in-memory Qdrant, gpt-4o-mini for extraction
m = Memory()

messages = [
    {"role": "user", "content": "I've been learning FastAPI for 3 months now."},
    {"role": "assistant", "content": "That's great! What have you built so far?"},
    {"role": "user", "content": "A small notes API with JWT auth. I prefer async patterns."},
]
result = m.add(messages, user_id="klement")
# result: {"results": [{"id": "...", "memory": "Learning FastAPI for 3 months", "event": "ADD"}, ...]}

# Semantic search
memories = m.search("What does the user know about FastAPI?", user_id="klement", limit=5)
# memories: {"results": [{"id": "...", "memory": "...", "score": 0.92}, ...]}

# Get ALL memories for a user
all_memories = m.get_all(user_id="klement")

# GDPR: delete all memories for a user
m.delete_all(user_id="klement")

# Delete single memory by ID
m.delete(memory_id="some-uuid")
```

---

### Core API: Qdrant Backend (Production)

```python
import os
from mem0 import Memory

config = {
    "vector_store": {
        "provider": "qdrant",
        "config": {
            "collection_name": "user_memories",
            "host": "localhost",
            "port": 6333,
            "embedding_model_dims": 1536,
            "on_disk": True,
        }
    },
    "llm": {
        "provider": "openai",
        "config": {
            "model": "gpt-4o-mini",
            "temperature": 0,
            "max_tokens": 2000,
        }
    },
    "embedder": {
        "provider": "openai",
        "config": {
            "model": "text-embedding-3-small",
        }
    }
}

m = Memory.from_config(config)
```

**With Qdrant Cloud:**

```python
config = {
    "vector_store": {
        "provider": "qdrant",
        "config": {
            "collection_name": "user_memories",
            "url": os.environ["QDRANT_URL"],
            "api_key": os.environ["QDRANT_API_KEY"],
            "embedding_model_dims": 1536,
        }
    },
}
```

**Fully local with Ollama (no API keys):**

```python
config = {
    "vector_store": {
        "provider": "qdrant",
        "config": {
            "collection_name": "memories_local",
            "host": "localhost",
            "port": 6333,
            "embedding_model_dims": 768,   # nomic-embed-text dims
        }
    },
    "llm": {
        "provider": "ollama",
        "config": {
            "model": "llama3.1:latest",
            "ollama_base_url": "http://localhost:11434",
        }
    },
    "embedder": {
        "provider": "ollama",
        "config": {
            "model": "nomic-embed-text:latest",
            "ollama_base_url": "http://localhost:11434",
        }
    }
}
```

---

### Async Usage for FastAPI

```python
import os
from functools import lru_cache
from fastapi import FastAPI, BackgroundTasks, Depends
from mem0 import AsyncMemory

app = FastAPI()

@lru_cache(maxsize=1)
def get_memory_client() -> AsyncMemory:
    config = {
        "vector_store": {
            "provider": "qdrant",
            "config": {
                "collection_name": "memories",
                "host": os.environ["QDRANT_HOST"],
                "port": int(os.environ.get("QDRANT_PORT", "6333")),
                "embedding_model_dims": 1536,
            }
        }
    }
    return AsyncMemory.from_config(config)

async def fire_memory_update(mem: AsyncMemory, messages: list[dict], user_id: str) -> None:
    try:
        await mem.add(messages, user_id=user_id)
    except Exception as e:
        import logging
        logging.getLogger(__name__).error(f"Memory update failed: {e}", exc_info=True)

@app.post("/chat/{user_id}")
async def chat(
    user_id: str,
    body: ChatRequest,
    background_tasks: BackgroundTasks,
    mem: AsyncMemory = Depends(get_memory_client),
):
    # 1. Retrieve memories BEFORE LLM call
    memories = await mem.search(body.message, user_id=user_id, limit=5)

    # 2. Build system prompt with memories
    system_prompt = format_system_prompt_with_memories(memories["results"])

    # 3. Call LLM
    response = await call_llm(system_prompt, body.message, body.history)

    # 4. Extract new memories AFTER response — user doesn't wait for this
    all_messages = body.history + [
        {"role": "user", "content": body.message},
        {"role": "assistant", "content": response},
    ]
    background_tasks.add_task(fire_memory_update, mem, all_messages, user_id)

    return {"response": response}
```

---

### Memory Extraction Pipeline (Internals)

When you call `m.add(messages, user_id=...)`, Mem0 internally:

1. **LLM extraction call:** Sends messages to extraction LLM with a prompt asking for memorable facts. Returns a JSON array of strings.
2. **Candidate generation:** e.g. `["User has been learning FastAPI for 3 months", "User prefers async patterns", "User built a notes API with JWT auth"]`
3. **Deduplication check:** Embeds each candidate and searches existing memories. If cosine similarity > ~0.95, merges/updates instead of adding.
4. **Write:** Each surviving memory written to the vector store with `payload={"user_id": "...", "memory": "...", "created_at": "..."}`.

The extraction LLM is cheap and separate — `gpt-4o-mini` or `claude-haiku-4-5` is fine. It runs in the background.

---

### GDPR: Complete Memory Deletion

```python
# Delete ALL memories for a user
m.delete_all(user_id="alice")

# Delete single memory
m.delete(memory_id="specific-memory-uuid")

# Verify deletion
remaining = m.get_all(user_id="alice")
assert len(remaining["results"]) == 0

# Async variant
await mem.delete_all(user_id="alice")
```

**Important:** `delete_all` only clears Mem0's vector store. A complete GDPR wipe must also clear episodic memory in PostgreSQL and procedural rules. See Part 6 for the full wipe implementation.

---

## Part 3 — Qdrant Vector Database

### What Qdrant Is

Qdrant is a dedicated vector search engine. It runs as a separate process/service, not a PostgreSQL extension.

**Core mental model:**
- **Collections:** like SQL tables, but for vectors. Fixed vector dimension and distance metric.
- **Points:** the rows. Each has an `id` (integer or UUID), a `vector` (float list), and a `payload` (arbitrary JSON).
- **HNSW index:** built automatically over all vectors. Enables ANN search in sub-millisecond time over millions of vectors.

---

### Installation and Startup

```bash
# Self-hosted via Docker
docker run -d \
    --name qdrant \
    -p 6333:6333 \
    -p 6334:6334 \
    -v $(pwd)/qdrant_storage:/qdrant/storage \
    qdrant/qdrant:latest

pip install qdrant-client
```

```python
from qdrant_client import QdrantClient, AsyncQdrantClient

# Synchronous (scripts, tests, migrations)
client = QdrantClient(host="localhost", port=6333)

# Async (FastAPI routes, LangGraph nodes)
async_client = AsyncQdrantClient(host="localhost", port=6333)

# In-memory (unit tests only)
test_client = QdrantClient(":memory:")

# Qdrant Cloud
cloud_client = QdrantClient(
    url="https://your-cluster-url.qdrant.io",
    api_key=os.environ["QDRANT_API_KEY"],
)
```

---

### Creating Collections

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, HnswConfigDiff,
    ScalarQuantization, ScalarQuantizationConfig,
)

# Production collection with HNSW tuning and scalar quantization
client.create_collection(
    collection_name="memories_production",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
    ),
    hnsw_config=HnswConfigDiff(
        m=16,               # graph connectivity
        ef_construct=200,   # build quality
        full_scan_threshold=10_000,
    ),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type="int8",        # 4x memory reduction
            quantile=0.99,
            always_ram=True,
        )
    ),
    on_disk_payload=True,   # payload JSON on disk (saves RAM)
)

# Named vectors — multiple embedding types in one collection
client.create_collection(
    collection_name="multimodal_memories",
    vectors_config={
        "text": VectorParams(size=1536, distance=Distance.COSINE),
        "image": VectorParams(size=512, distance=Distance.EUCLIDEAN),
    }
)

# Idempotent startup check
if not client.collection_exists("user_memories"):
    client.create_collection(...)
```

**HNSW parameter guide:**

| Parameter | Value | Effect |
|---|---|---|
| `m=16` | Standard | Good balance recall/RAM |
| `m=32` | Higher quality | 2x RAM on graph, better recall |
| `ef_construct=100` | Fast build | Slightly worse recall |
| `ef_construct=200` | Good build | Better recall, 2x build time |

---

### Upserting Points

```python
import uuid, time
from qdrant_client.models import PointStruct

# Batch upsert — efficient for bulk writes
points = [
    PointStruct(
        id=str(uuid.uuid4()),
        vector=embedding_vector,
        payload={
            "user_id": user_id,
            "memory": memory_text,
            "created_at_ts": time.time(),
            "confidence": 0.9,
        }
    )
    for memory_text, embedding_vector in zip(memory_texts, embedding_vectors)
]

BATCH_SIZE = 100
for i in range(0, len(points), BATCH_SIZE):
    client.upsert(
        collection_name="user_memories",
        points=points[i:i + BATCH_SIZE],
        wait=True,
    )
```

---

### Searching with Filters

**Payload indexes are mandatory before filtered search:**

```python
# Create indexes ONCE at collection setup
client.create_payload_index("user_memories", "user_id", "keyword")
client.create_payload_index("user_memories", "category", "keyword")
client.create_payload_index("user_memories", "created_at_ts", "float")
```

Without payload indexes, Qdrant does a full scan for every filtered query — catastrophic at 10M points.

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

# User-isolated semantic search
results = client.search(
    collection_name="user_memories",
    query_vector=embed("python libraries"),
    query_filter=Filter(
        must=[
            FieldCondition(key="user_id", match=MatchValue(value="alice")),
        ]
    ),
    limit=5,
    score_threshold=0.7,
)

# Complex filter: must + must_not + date range
cutoff = time.time() - (90 * 86400)
results = client.search(
    collection_name="user_memories",
    query_vector=query_vec,
    query_filter=Filter(
        must=[
            FieldCondition(key="user_id", match=MatchValue(value="alice")),
            FieldCondition(key="created_at_ts", range=Range(gte=cutoff)),
        ],
        must_not=[
            FieldCondition(key="deprecated", match=MatchValue(value=True)),
        ],
    ),
    limit=5,
)
```

---

### Quantization: Memory Reduction

```python
from qdrant_client.models import (
    BinaryQuantization, BinaryQuantizationConfig,
    SearchParams, QuantizationSearchParams,
)

# Binary quantization — 32x memory reduction
client.create_collection(
    collection_name="large_scale_memories",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    quantization_config=BinaryQuantization(
        binary=BinaryQuantizationConfig(always_ram=True)
    ),
)

# Search with oversampling to recover recall
results = client.search(
    collection_name="large_scale_memories",
    query_vector=embed(query),
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,       # re-rank with full precision
            oversampling=3.0,   # fetch 3x candidates before reranking
        )
    ),
    limit=10,
)
```

| Quantization | Memory reduction | Recall loss | When to use |
|---|---|---|---|
| None | 1x | 0% | < 1M vectors |
| INT8 scalar | 4x | < 1% | 1M–50M vectors (most cases) |
| Binary | 32x | ~5–10% | > 50M vectors |

---

### Snapshots: Backup and Restore

```python
# Create snapshot
snapshot_info = client.create_snapshot(collection_name="user_memories")
print(snapshot_info.name)

# List snapshots
for s in client.list_snapshots(collection_name="user_memories"):
    print(f"{s.name}  {s.size / 1024 / 1024:.1f} MB")

# Restore from local file
client.recover_snapshot(
    collection_name="user_memories_restored",
    location="file:///tmp/snapshots/user_memories-123.snapshot",
)

# Restore from pre-signed S3 URL
import boto3
s3 = boto3.client("s3")
presigned_url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-backups", "Key": "snapshots/user_memories-123.snapshot"},
    ExpiresIn=3600,
)
client.recover_snapshot(
    collection_name="user_memories_restored",
    location=presigned_url,
)
```

**Backup schedule:** Nightly snapshots → upload to S3 (keep 7 daily, 4 weekly) → weekly restore test to staging.

---

### Qdrant Cloud vs Self-Hosted

| Aspect | Self-Hosted | Qdrant Cloud |
|---|---|---|
| Setup time | ~30 min | ~5 min |
| Ops burden | You | Managed |
| Cost at 10M vectors | ~$50/mo EC2 | ~$200+/mo |
| Data residency | Your infra | Their infra |
| GDPR compliance | Full control | Requires DPA review |
| HA / replication | Manual cluster | Built-in |

For user memory data (PII), self-hosted is easier to comply with GDPR. Qdrant Cloud adds a vendor to your data processing chain.

---

## Part 4 — Memory Injection Strategy

### Where to Inject: System Prompt

Always inject retrieved memories into the system prompt, never the user message.

```python
def build_messages_with_memory(
    user_message: str,
    memories: list[dict],
    base_system_prompt: str,
) -> list[dict]:
    if memories:
        memory_block = "\n".join(f"- {m['memory']}" for m in memories)
        memory_section = f"\n\n## What you know about this user\n\n{memory_block}\n\nUse this context to personalize responses."
    else:
        memory_section = ""

    return [
        {"role": "system", "content": base_system_prompt + memory_section},
        {"role": "user", "content": user_message},
    ]
```

---

### Top-K Selection with Freshness Weighting

```python
import math, time

def rerank_memories(
    candidates: list[dict],
    top_k: int = 5,
    freshness_weight: float = 0.2,
    half_life_days: float = 90.0,
) -> list[dict]:
    now = time.time()
    for m in candidates:
        similarity = m.get("score", 0.0)
        age_days = (now - m.get("created_at_ts", now)) / 86400
        freshness = math.exp(-age_days * math.log(2) / half_life_days)
        m["combined_score"] = (
            (1 - freshness_weight) * similarity + freshness_weight * freshness
        )
    candidates.sort(key=lambda x: x["combined_score"], reverse=True)
    return candidates[:top_k]
```

---

### Token Budget

```python
def truncate_memories_to_budget(memories: list[dict], max_tokens: int = 1500) -> list[dict]:
    # Rough approximation: 4 chars per token
    total_chars = 0
    MAX_CHARS = max_tokens * 4
    selected = []
    for m in memories:
        chars = len(m["memory"]) + 3
        if total_chars + chars > MAX_CHARS:
            break
        selected.append(m)
        total_chars += chars
    return selected
```

---

### Handling Memory Conflicts

```python
# At injection: instruct the model to prefer the most recent fact
MEMORY_SECTION_TEMPLATE = """
## What you know about this user

{memories}

If any facts above contradict each other, use the most recently created one.
When genuinely uncertain, ask the user to clarify.
"""

# At write: detect near-duplicates and update instead of duplicating
CONTRADICTION_THRESHOLD = 0.90

async def write_with_conflict_resolution(qdrant, user_id, new_memory, vector):
    existing = await qdrant.search(
        collection_name="user_memories",
        query_vector=vector,
        query_filter=Filter(must=[FieldCondition(key="user_id", match=MatchValue(value=user_id))]),
        limit=1,
        score_threshold=CONTRADICTION_THRESHOLD,
    )
    if existing:
        # Delete old contradicting memory, write new
        await qdrant.delete(
            collection_name="user_memories",
            points_selector=PointIdsList(points=[existing[0].id]),
        )
    await upsert_memory(qdrant, user_id, new_memory, vector)
```

---

### Staleness: TTL via Scheduled Cleanup

Qdrant has no native TTL. Use a scheduled job:

```python
async def expire_old_memories_job(
    qdrant: AsyncQdrantClient,
    collection: str = "user_memories",
    max_age_days: int = 365,
) -> None:
    cutoff_ts = time.time() - (max_age_days * 86400)
    await qdrant.delete(
        collection_name=collection,
        points_selector=FilterSelector(
            filter=Filter(
                must=[FieldCondition(key="created_at_ts", range=Range(lt=cutoff_ts))],
                must_not=[FieldCondition(key="pinned", match=MatchValue(value=True))],
            )
        ),
    )
```

---

## Part 5 — Memory Extraction

### When to Run

```
User sends message
  → [sync]  retrieve memories → inject → call LLM → send response
  → [async] extract memories from this turn → deduplicate → write to Qdrant
```

---

### Extraction Prompt

```python
EXTRACTION_PROMPT = """
You are analyzing a conversation to extract memorable facts about the user.

## Conversation
{conversation_text}

## Task
Extract facts worth remembering for future sessions.

INCLUDE:
- Explicit preferences ("I prefer...", "I hate...", "I always...")
- Skills and experience ("I've been learning X for Y months")
- Goals and projects ("I'm building...", "I want to...")
- Hard constraints ("I can't use X because...")
- Communication preferences ("respond in English", "be concise")

DO NOT INCLUDE:
- Temporary context ("I'm tired today", "this morning...")
- Questions the user asked
- Content from the assistant's responses
- PII, credentials, medical data

Return ONLY a JSON array of strings. Return [] if nothing is worth remembering.
Example: ["User prefers Python over JavaScript", "User is building a FastAPI notes API"]
"""

async def extract_memories_from_turn(messages: list[dict], extraction_llm) -> list[str]:
    conversation_text = "\n".join(
        f"{m['role'].upper()}: {m['content']}"
        for m in messages[-10:]
    )
    response = await extraction_llm.complete(
        EXTRACTION_PROMPT.format(conversation_text=conversation_text)
    )
    import json
    try:
        memories = json.loads(response.text.strip())
        return [m for m in memories if isinstance(m, str) and len(m) > 10]
    except json.JSONDecodeError:
        return []
```

---

### Deduplication

```python
DEDUPLICATION_THRESHOLD = 0.95

async def write_memories_deduplicated(qdrant, embedder, user_id, new_memories):
    written = 0
    for memory_text in new_memories:
        vector = await embedder.embed(memory_text)
        existing = await qdrant.search(
            collection_name="user_memories",
            query_vector=vector,
            query_filter=Filter(must=[FieldCondition(key="user_id", match=MatchValue(value=user_id))]),
            limit=1,
            score_threshold=DEDUPLICATION_THRESHOLD,
        )
        if existing:
            continue  # skip duplicate
        await qdrant.upsert(
            collection_name="user_memories",
            points=[PointStruct(
                id=str(uuid.uuid4()),
                vector=vector,
                payload={"user_id": user_id, "memory": memory_text, "created_at_ts": time.time()}
            )],
            wait=False,
        )
        written += 1
    return written
```

---

### When NOT to Extract

```python
import re

SKIP_PATTERNS = [
    r"\b(today|this morning|right now|currently feeling|at the moment)\b",
    r"\bpassword\b|\bsecret\b|\bapi.?key\b|\btoken\b",
    r"\bcredit.?card\b|\bssn\b|\bsocial security\b",
    r"\bdiagnos(is|ed)\b|\bprescri(bed|ption)\b",
]

def should_skip_extraction(message: str) -> bool:
    lower = message.lower()
    return any(re.search(p, lower) for p in SKIP_PATTERNS)

def is_extractable_turn(messages: list[dict]) -> bool:
    user_messages = [m for m in messages if m["role"] == "user"]
    total_chars = sum(len(m["content"]) for m in user_messages)
    return total_chars > 100  # minimum substance threshold
```

---

## Part 6 — Privacy and Multi-Tenancy

### GDPR: Complete Multi-Layer Wipe

```python
from sqlalchemy import delete
import logging

logger = logging.getLogger(__name__)

async def gdpr_erase_user_complete(
    user_id: str,
    qdrant: AsyncQdrantClient,
    pg_session: AsyncSession,
    mem0_client: AsyncMemory,
) -> dict[str, str]:
    results = {}

    # 1. Mem0 vector memories
    try:
        await mem0_client.delete_all(user_id=user_id)
        results["mem0_memories"] = "erased"
    except Exception as e:
        results["mem0_memories"] = f"error: {e}"

    # 2. Direct Qdrant deletion (belt-and-suspenders)
    try:
        await qdrant.delete(
            collection_name="user_memories",
            points_selector=FilterSelector(
                filter=Filter(must=[FieldCondition(key="user_id", match=MatchValue(value=user_id))])
            ),
        )
        results["qdrant_direct"] = "erased"
    except Exception as e:
        results["qdrant_direct"] = f"error: {e}"

    # 3. Episodic memory in PostgreSQL
    deleted = await pg_session.execute(
        delete(EpisodicMemory).where(EpisodicMemory.user_id == user_id)
    )
    results["episodic"] = f"{deleted.rowcount} rows deleted"

    # 4. Procedural rules
    deleted = await pg_session.execute(
        delete(UserRule).where(UserRule.user_id == user_id)
    )
    results["procedural_rules"] = f"{deleted.rowcount} rows deleted"

    await pg_session.commit()

    # 5. Audit log — RETAIN even after data wipe (regulatory requirement)
    await pg_session.execute(
        insert(GDPRAuditLog).values(
            user_id=user_id,
            event="gdpr_erasure_complete",
            timestamp=datetime.utcnow(),
            details=str(results),
        )
    )
    await pg_session.commit()

    return results
```

---

### PII Detection Before Storage

```python
import re

PII_PATTERNS = [
    (r"\b\d{3}-\d{2}-\d{4}\b", "SSN"),
    (r"\b4[0-9]{12}(?:[0-9]{3})?\b", "visa_card"),
    (r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", "email"),
    (r"(?i)password\s*[:=]\s*\S+", "password"),
    (r"(?i)api[_-]?key\s*[:=]\s*\S+", "api_key"),
    (r"(?i)bearer\s+[A-Za-z0-9\-._~+/]+=*", "bearer_token"),
]

def sanitize_memory_for_storage(memory: str) -> str | None:
    for pattern, pii_type in PII_PATTERNS:
        if re.search(pattern, memory):
            logger.warning(f"PII ({pii_type}) in memory candidate — dropped")
            return None
    return memory

# In extraction pipeline:
safe_memories = [sanitize_memory_for_storage(m) for m in raw_extracted_memories]
safe_memories = [m for m in safe_memories if m is not None]
```

---

### Family / Group Scope

```python
# Payload schema for group-scoped memories
{
    "user_id": "alice",
    "scope": "family",               # "private" | "family" | "org"
    "family_id": "smith-family",
    "memory": "Smith family only eats halal meat",
}

# Query: memories visible to alice in her family context
async def get_scoped_memories(qdrant, user_id, family_id, query_vec) -> list:
    should_filters = [
        Filter(must=[
            FieldCondition(key="user_id", match=MatchValue(value=user_id)),
            FieldCondition(key="scope", match=MatchValue(value="private")),
        ])
    ]
    if family_id:
        should_filters.append(
            Filter(must=[
                FieldCondition(key="family_id", match=MatchValue(value=family_id)),
                FieldCondition(key="scope", match=MatchValue(value="family")),
            ])
        )
    return await qdrant.search(
        collection_name="user_memories",
        query_vector=query_vec,
        query_filter=Filter(should=should_filters),
        limit=10,
    )
```

---

## Part 7 — pgvector vs Qdrant Decision Matrix

| Criterion | pgvector | Qdrant |
|---|---|---|
| Vector count sweet spot | < 5M per table | 5M → 1B+ |
| Query throughput | ~500–2K QPS | ~5K–50K QPS |
| ACID transactions | Full (same DB) | No |
| Collection isolation | Manual (schemas) | Native |
| Quantization | Not built-in | INT8 + binary built-in |
| Backup | pg_dump | Snapshot API |
| Infrastructure | PG extension only | Separate service |
| Team learning curve | Low (SQL) | Medium (new API) |
| Cost at 10M (self-hosted) | PG instance only | Qdrant (lighter) |

### When NOT to Use pgvector
- Vector count exceeds 5M and query latency matters
- Need collection-level isolation with different configs per tenant
- Stack is not PostgreSQL
- Need built-in quantization or fast point-in-time restore

### When NOT to Use Qdrant
- Entire stack already PostgreSQL with < 1M vectors
- Need ACID (vector + metadata write in same DB transaction)
- No capacity to operate another service
- Data is deeply relational with join-heavy queries

### pgvector Advantage: ACID Transactions

```python
# pgvector: atomic write — both row and vector in one transaction
async with session.begin():
    chunk = MemoryChunk(
        user_id=user_id,
        content=memory_text,
        embedding=vector,     # pgvector Vector column
        created_at=datetime.utcnow(),
    )
    session.add(chunk)
    # Crash here = nothing written — no orphaned vectors
```

With Qdrant + PostgreSQL, you write to two separate systems. A crash between writes leaves them out of sync. Mitigate with a transactional outbox pattern.

---

### Migration Path: pgvector → Qdrant

```python
async def migrate_pgvector_to_qdrant(
    pg_session: AsyncSession,
    qdrant: AsyncQdrantClient,
    pg_model: type,
    collection_name: str,
    batch_size: int = 500,
) -> int:
    offset = 0
    total = 0
    while True:
        result = await pg_session.execute(
            select(pg_model).order_by(pg_model.id).offset(offset).limit(batch_size)
        )
        rows = result.scalars().all()
        if not rows:
            break
        await qdrant.upsert(
            collection_name=collection_name,
            points=[
                PointStruct(
                    id=str(row.id),
                    vector=list(row.embedding),
                    payload={"user_id": row.user_id, "memory": row.content, "created_at_ts": row.created_at.timestamp()}
                )
                for row in rows
            ],
            wait=True,
        )
        total += len(rows)
        offset += batch_size
    return total
# Strategy: dual-write period → verify counts match → switch reads → stop PG writes
```

---

## Part 8 — Production Patterns

### Memory Warming on Session Start

```python
import json
from redis.asyncio import Redis

WARM_CACHE_KEY = "memory_warm:{user_id}"
WARM_CACHE_TTL = 300

async def warm_memory_cache(user_id: str, qdrant: AsyncQdrantClient, redis: Redis) -> None:
    cache_key = WARM_CACHE_KEY.format(user_id=user_id)
    if await redis.exists(cache_key):
        return

    result, _ = await qdrant.scroll(
        collection_name="user_memories",
        scroll_filter=Filter(must=[FieldCondition(key="user_id", match=MatchValue(value=user_id))]),
        limit=50,
        with_payload=True,
        with_vectors=False,
    )
    memories = [
        {"memory": p.payload["memory"], "id": str(p.id), "created_at_ts": p.payload.get("created_at_ts", 0), "score": 1.0}
        for p in result
    ]
    memories.sort(key=lambda m: m["created_at_ts"], reverse=True)
    await redis.setex(cache_key, WARM_CACHE_TTL, json.dumps(memories))
```

---

### Circuit Breaker

```python
from enum import Enum
from dataclasses import dataclass, field

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

@dataclass
class MemoryCircuitBreaker:
    failure_threshold: int = 5
    recovery_timeout_seconds: float = 60.0
    _state: CircuitState = field(default=CircuitState.CLOSED, init=False)
    _failure_count: int = field(default=0, init=False)
    _last_failure_at: float = field(default=0.0, init=False)

    def should_skip(self) -> bool:
        if self._state == CircuitState.CLOSED:
            return False
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._last_failure_at > self.recovery_timeout_seconds:
                self._state = CircuitState.HALF_OPEN
                return False
            return True
        return False

    def record_success(self) -> None:
        self._state = CircuitState.CLOSED
        self._failure_count = 0

    def record_failure(self) -> None:
        self._failure_count += 1
        self._last_failure_at = time.monotonic()
        if self._failure_count >= self.failure_threshold:
            self._state = CircuitState.OPEN

_memory_circuit = MemoryCircuitBreaker()

async def resilient_memory_search(mem0, query, user_id, limit=5) -> list[dict]:
    if _memory_circuit.should_skip():
        return []  # degrade gracefully — LLM still responds without personalization
    try:
        result = await mem0.search(query, user_id=user_id, limit=limit)
        _memory_circuit.record_success()
        return result.get("results", [])
    except Exception as e:
        _memory_circuit.record_failure()
        logger.error(f"Memory search failed: {e}")
        return []
```

---

### Episodic Memory Creation at Session End

```python
EPISODE_PROMPT = """
Summarize this conversation in 3-5 sentences for long-term memory.
Include: main topics, problems and whether solved, key decisions, user's emotional state if notable.

Conversation: {conversation}

Output format:
TOPICS: comma-separated topics
OUTCOME: resolved | unresolved | partial | informational
SENTIMENT: positive | neutral | mixed | frustrated
SUMMARY: <3-5 sentence narrative>
"""

async def create_episode_from_session(messages, user_id, session_id, llm, pg, embedder, qdrant):
    if len(messages) < 4:
        return -1
    conversation = "\n".join(f"{m['role'].upper()}: {m['content'][:300]}" for m in messages)
    response = await llm.complete(EPISODE_PROMPT.format(conversation=conversation))
    # ... parse output, create EpisodicMemory row, embed, upsert to Qdrant
```

---

### Monitoring

```python
from prometheus_client import Counter, Histogram, Gauge

memory_search_seconds = Histogram("memory_search_duration_seconds", "Qdrant search latency",
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0])
memory_extraction_seconds = Histogram("memory_extraction_duration_seconds", "Extraction latency",
    buckets=[0.5, 1.0, 2.0, 5.0, 10.0, 30.0])
memory_circuit_open = Gauge("memory_circuit_breaker_open", "1 if circuit open")
memories_written_total = Counter("memories_written_total", "Total memory writes", ["source"])
```

| Metric | Alert threshold | Action |
|---|---|---|
| `memory_search_duration_seconds` P99 | > 500ms | Qdrant overloaded |
| Empty results rate | > 50% of queries | Extraction pipeline failing |
| `memory_circuit_breaker_open` | == 1 for > 2 min | Page on-call |
| `memory_extraction_duration_seconds` P95 | > 10s | Extraction model slow |

---

## Quick Reference

### Mem0 API Cheatsheet

```python
from mem0 import Memory, AsyncMemory

# Sync
m = Memory.from_config(config)
m.add(messages, user_id="u")
m.search("query", user_id="u", limit=5, threshold=0.7)
m.get_all(user_id="u")
m.delete_all(user_id="u")       # GDPR wipe
m.delete(memory_id="uuid")
m.update(memory_id="uuid", data="new text")
m.add(messages, user_id="u", metadata={"category": "preference"})

# Async
m = AsyncMemory.from_config(config)
await m.add(messages, user_id="u")
await m.search("q", user_id="u", limit=5)
await m.delete_all(user_id="u")
```

### Qdrant Client Cheatsheet

```python
from qdrant_client import QdrantClient, AsyncQdrantClient
from qdrant_client.models import *

client = QdrantClient(host="localhost", port=6333)
client.create_collection("name", vectors_config=VectorParams(size=1536, distance=Distance.COSINE))
client.create_payload_index("name", "user_id", "keyword")  # MANDATORY
client.upsert("name", points=[PointStruct(id=..., vector=..., payload=...)])
client.search("name", query_vector=vec, limit=5, query_filter=Filter(...))
client.scroll("name", scroll_filter=Filter(...), limit=100)
client.delete("name", points_selector=FilterSelector(filter=Filter(...)))
client.create_snapshot("name")
client.recover_snapshot("new_name", location="file:///path.snapshot")
```

### Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No `user_id` filter in search | Cross-user memory leak (security hole) | Always `must=[FieldCondition("user_id", ...)]` |
| `mem0.add()` in request path | +2-5s latency per response | Use `BackgroundTasks` |
| No payload index on `user_id` | O(n) full scan at scale | `create_payload_index` at setup |
| Multiple `Memory()` instances | Separate in-memory state per worker | One `AsyncMemory` per process |
| Dims mismatch in config | Garbage similarity scores | Config dims must match embedder exactly |
| GDPR wipe only via Mem0 | Episodic/procedural memories survive | Wipe Qdrant + PostgreSQL + Redis |
| Sync `Memory()` in async FastAPI | Event loop blocked | Use `AsyncMemory()` |
| No circuit breaker | Qdrant outage brings down whole app | Wrap all calls in circuit breaker |
| Storing PII in long-term memory | GDPR violation | Run `sanitize_memory_for_storage()` |
| `wait=True` on every upsert | Blocks until indexed | Use `wait=False` for fire-and-forget |

---

*Sources: [Mem0 documentation](https://docs.mem0.ai), [Qdrant documentation](https://qdrant.tech/documentation), [Mem0 + Qdrant integration](https://qdrant.tech/documentation/frameworks/mem0/), [LangGraph memory overview](https://docs.langchain.com/oss/python/langgraph/memory)*
