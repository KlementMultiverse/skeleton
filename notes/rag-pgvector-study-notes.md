# RAG + pgvector Complete — L11

> Stack: Python 3.12, SQLAlchemy 2.0 async, FastAPI, pgvector 0.3+, asyncpg.
> Last verified: April 2026.
> AI connection: RAG is what makes Claude answer questions about YOUR data instead of its training data. Every retrieved chunk becomes a message in the conversation you already know how to build with the Anthropic SDK (L10).

---

## What RAG Is

**Analogy:** A lawyer preparing for court. They don't memorize every case that ever existed — they send a paralegal to retrieve the most relevant ones, then argue from those documents. RAG is that paralegal: retrieve first, then generate.

Without RAG, Claude only knows what Anthropic trained it on (cutoff August 2025). With RAG:
- Your internal docs become answerable
- Knowledge stays fresh (re-embed when docs change)
- Answers are grounded (you can cite the source chunk)
- Context window stays manageable (you inject 3-10 chunks, not 10,000 pages)

**The three-step loop:** embed your docs once → on each query, embed the question → find closest docs → inject into Claude prompt.

---

## Concept 1 — Embeddings (Numerical Coordinates in Meaning-Space)

**Analogy:** Words on a map. "Dog" and "puppy" are neighbouring cities. "Dog" and "tax return" are on different continents. Embeddings are GPS coordinates for meaning.

An embedding model converts any text into a fixed-length float vector. Texts with similar meaning produce vectors that are close together in that high-dimensional space. Distance = dissimilarity. This is why you can find "relevant" documents without exact keyword matches.

```python
import os
import voyageai

# voyage-3: 1024 dims, context window 32K tokens, best for code + technical docs
client = voyageai.Client(api_key=os.environ["VOYAGE_API_KEY"])

text = "How do I set up pgvector?"
result = client.embed([text], model="voyage-3", input_type="query")
vector: list[float] = result.embeddings[0]  # 1024 floats, always normalized to unit length
print(len(vector))  # 1024
```

**Rules:**
- Documents use `input_type="document"`. Queries use `input_type="query"`. Different types produce slightly different vectors optimised for asymmetric retrieval. Always set this.
- Voyage embeddings are pre-normalized to unit length. Cosine similarity and dot product are equivalent for normalized vectors — use cosine for safety.
- NEVER mix two different embedding models in the same vector index. The coordinate systems are incompatible — similarity scores become meaningless garbage.

---

## Concept 2 — pgvector Setup (CREATE EXTENSION, Vector Column, HNSW Index)

**Analogy:** Postgres got a GPS module installed. Normal indexes find rows by exact value. The pgvector extension adds an ANN (approximate nearest neighbour) index that finds rows by *direction in vector space*.

```sql
-- Run once per database (requires superuser or pg_extension privilege)
CREATE EXTENSION IF NOT EXISTS vector;
```

```python
# SQLAlchemy 2.0 async model
from pgvector.sqlalchemy import Vector
from sqlalchemy import String, Text, Integer, Index
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

EMBEDDING_DIMS = 1024  # voyage-3. Change to 1536 for text-embedding-3-small, 384 for MiniLM

class Base(DeclarativeBase):
    pass

class DocumentChunk(Base):
    __tablename__ = "document_chunks"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    document_id: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    tenant_id: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    chunk_index: Mapped[int] = mapped_column(Integer, nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    # Full-text search column (BM25 — see Concept 7)
    content_tsv = mapped_column("content_tsv", nullable=True)
    # Parent chunk text for hierarchical retrieval
    parent_content: Mapped[str | None] = mapped_column(Text, nullable=True)
    embedding: Mapped[list[float]] = mapped_column(Vector(EMBEDDING_DIMS), nullable=True)

    # HNSW index for approximate nearest neighbour search
    # Created separately after table creation because SQLAlchemy DDL for pgvector indexes
    # uses raw SQL — see migration below.

# Alembic migration (or run directly):
# CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
# WITH (m = 16, ef_construction = 64);
#
# CREATE INDEX ON document_chunks USING GIN (content_tsv);  -- for BM25
#
# UPDATE document_chunks SET content_tsv = to_tsvector('english', content);
```

**Creating the HNSW index in SQLAlchemy migrations:**

```python
from alembic import op

def upgrade() -> None:
    op.execute(
        """
        CREATE INDEX IF NOT EXISTS idx_chunks_embedding_hnsw
        ON document_chunks
        USING hnsw (embedding vector_cosine_ops)
        WITH (m = 16, ef_construction = 64)
        """
    )
    op.execute(
        """
        CREATE INDEX IF NOT EXISTS idx_chunks_content_tsv
        ON document_chunks USING GIN (content_tsv)
        """
    )
```

---

## Concept 3 — Embedding Models: Which One to Use

**Analogy:** Three translators. One is superb with code and technical jargon (Voyage). One is cheap and good enough for general prose (OpenAI). One works offline with no API key (MiniLM).

| Model | Dims | Context | Best For | Cost |
|---|---|---|---|---|
| `voyage-3` | 1024 | 32K tokens | Code, technical docs, mixed content | ~$0.06/1M tokens |
| `text-embedding-3-small` | 1536 | 8K tokens | General English prose, chat history | ~$0.02/1M tokens |
| `all-MiniLM-L6-v2` | 384 | 256 tokens | Local dev, CI, no API key | Free |

```python
# voyage-3 (recommended for this stack)
import voyageai, os
voyage = voyageai.Client(api_key=os.environ["VOYAGE_API_KEY"])

def embed_documents(texts: list[str]) -> list[list[float]]:
    result = voyage.embed(texts, model="voyage-3", input_type="document")
    return result.embeddings  # list[list[float]], each 1024 dims

def embed_query(text: str) -> list[float]:
    result = voyage.embed([text], model="voyage-3", input_type="query")
    return result.embeddings[0]
```

```python
# text-embedding-3-small (OpenAI — if already paying for GPT)
from openai import AsyncOpenAI
import os

openai_client = AsyncOpenAI(api_key=os.environ["OPENAI_API_KEY"])

async def embed_query_openai(text: str) -> list[float]:
    resp = await openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return resp.data[0].embedding  # 1536 floats
```

```python
# all-MiniLM-L6-v2 (local, sentence-transformers)
from sentence_transformers import SentenceTransformer

_local_model = SentenceTransformer("all-MiniLM-L6-v2")  # loads once, ~80MB

def embed_local(texts: list[str]) -> list[list[float]]:
    return _local_model.encode(texts, normalize_embeddings=True).tolist()
```

---

## Concept 4 — Chunking Strategies

**Analogy:** Splitting a book to mail it. Fixed-size chunking = cut every 50 pages whether mid-chapter or not. Semantic chunking = cut at chapter breaks. Hierarchical chunking = keep chapter summaries alongside page chunks.

### Strategy A: Fixed-size with overlap (simplest, good default)

```python
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")  # same tokeniser as OpenAI + Voyage

def chunk_fixed(
    text: str,
    chunk_size: int = 512,   # tokens — configurable via env var
    overlap: int = 50,        # ~10% overlap keeps context across boundaries
) -> list[str]:
    tokens = enc.encode(text)
    chunks = []
    start = 0
    while start < len(tokens):
        end = start + chunk_size
        chunk_tokens = tokens[start:end]
        chunks.append(enc.decode(chunk_tokens))
        if end >= len(tokens):
            break
        start += chunk_size - overlap  # slide forward by (chunk_size - overlap)
    return chunks
```

### Strategy B: Semantic (paragraph/sentence boundaries)

```python
import re

def chunk_semantic(text: str, max_tokens: int = 512) -> list[str]:
    """Split on paragraph breaks, then merge small paragraphs up to max_tokens."""
    paragraphs = [p.strip() for p in re.split(r"\n\s*\n", text) if p.strip()]
    chunks: list[str] = []
    current_parts: list[str] = []
    current_tokens = 0

    for para in paragraphs:
        para_tokens = len(enc.encode(para))
        if current_tokens + para_tokens > max_tokens and current_parts:
            chunks.append("\n\n".join(current_parts))
            current_parts = []
            current_tokens = 0
        current_parts.append(para)
        current_tokens += para_tokens

    if current_parts:
        chunks.append("\n\n".join(current_parts))

    return chunks
```

### Strategy C: Hierarchical (small chunks retrieve, parent chunks for context)

**Analogy:** Index cards with footnotes pointing to the full page. You search the index cards (small, precise), but when you find a match you hand the reader the full page (rich context).

```python
from dataclasses import dataclass

@dataclass
class HierarchicalChunk:
    child_text: str       # embedded + retrieved (512 tokens)
    parent_text: str      # injected into LLM context (1024-2048 tokens)
    chunk_index: int
    parent_index: int

def chunk_hierarchical(
    text: str,
    child_size: int = 512,
    parent_size: int = 1536,
    child_overlap: int = 50,
) -> list[HierarchicalChunk]:
    parent_chunks = chunk_fixed(text, chunk_size=parent_size, overlap=100)
    result: list[HierarchicalChunk] = []
    chunk_index = 0

    for parent_idx, parent_text in enumerate(parent_chunks):
        children = chunk_fixed(parent_text, chunk_size=child_size, overlap=child_overlap)
        for child_text in children:
            result.append(
                HierarchicalChunk(
                    child_text=child_text,
                    parent_text=parent_text,
                    chunk_index=chunk_index,
                    parent_index=parent_idx,
                )
            )
            chunk_index += 1

    return result
```

**Contextual Retrieval (prepend document context to every chunk before embedding):**

```python
def enrich_chunk(chunk: str, doc_title: str, section_header: str) -> str:
    """
    Prepend document context before embedding.
    Improves retrieval accuracy by 15-25% — the embedding model sees WHERE
    this chunk comes from, not just what it says.
    """
    return f"Document: {doc_title}\nSection: {section_header}\n\n{chunk}"
```

---

## Concept 5 — Ingestion Pipeline (Load → Chunk → Embed → Store)

**Analogy:** A factory assembly line. Raw material (document) enters one end. Labeled, boxed, catalogued product (embedded chunks in Postgres) comes out the other end.

```python
import os
import asyncio
from pathlib import Path
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
import voyageai

DATABASE_URL = os.environ["DATABASE_URL"]  # postgresql+asyncpg://...
engine = create_async_engine(DATABASE_URL, echo=False)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

voyage = voyageai.Client(api_key=os.environ["VOYAGE_API_KEY"])

CHUNK_SIZE = int(os.environ.get("RAG_CHUNK_SIZE", "512"))
CHUNK_OVERLAP = int(os.environ.get("RAG_CHUNK_OVERLAP", "50"))
EMBED_BATCH_SIZE = int(os.environ.get("RAG_EMBED_BATCH_SIZE", "128"))

async def ingest_document(
    document_id: str,
    tenant_id: str,
    text: str,
    title: str,
    section_header: str = "",
    session: AsyncSession = None,
) -> int:
    """Ingest one document. Returns number of chunks stored."""
    # 1. Chunk
    raw_chunks = chunk_fixed(text, chunk_size=CHUNK_SIZE, overlap=CHUNK_OVERLAP)

    # 2. Enrich with context before embedding
    enriched = [enrich_chunk(c, title, section_header) for c in raw_chunks]

    # 3. Embed in batches with basic retry
    embeddings: list[list[float]] = []
    for batch_start in range(0, len(enriched), EMBED_BATCH_SIZE):
        batch = enriched[batch_start : batch_start + EMBED_BATCH_SIZE]
        for attempt in range(3):
            try:
                result = voyage.embed(batch, model="voyage-3", input_type="document")
                embeddings.extend(result.embeddings)
                break
            except Exception as e:
                if attempt == 2:
                    raise
                await asyncio.sleep(2 ** attempt)  # exponential backoff

    # 4. Store chunks + embeddings
    rows = [
        DocumentChunk(
            document_id=document_id,
            tenant_id=tenant_id,
            chunk_index=i,
            content=raw_chunks[i],
            embedding=embeddings[i],
        )
        for i in range(len(raw_chunks))
    ]

    async with (session or SessionLocal()) as db:
        db.add_all(rows)
        # Also update tsvector column for BM25
        for row in rows:
            await db.execute(
                """
                UPDATE document_chunks
                SET content_tsv = to_tsvector('english', :content)
                WHERE id = :id
                """,
                # Note: runs after flush to get IDs
            )
        await db.commit()

    return len(rows)
```

**FastAPI ingest endpoint:**

```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/rag", tags=["rag"])

class IngestRequest(BaseModel):
    document_id: str
    tenant_id: str
    text: str
    title: str
    section_header: str = ""

@router.post("/ingest")
async def ingest(req: IngestRequest) -> dict:
    try:
        count = await ingest_document(
            document_id=req.document_id,
            tenant_id=req.tenant_id,
            text=req.text,
            title=req.title,
            section_header=req.section_header,
        )
        return {"chunks_stored": count}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## Concept 6 — Similarity Search with Cosine Distance (`<=>` operator)

**Analogy:** Shining a flashlight from your question into a room full of document chunks. The ones it illuminates most brightly are the most relevant. The `<=>` operator measures the angle between your flashlight beam and each chunk — smaller angle = brighter = more relevant.

```python
from sqlalchemy import text, select
from sqlalchemy.ext.asyncio import AsyncSession

async def vector_search(
    query: str,
    tenant_id: str,
    top_k: int = 20,
    session: AsyncSession = None,
) -> list[dict]:
    """Pure vector similarity search. Returns top_k chunks by cosine distance."""
    query_embedding = embed_query(query)
    embedding_literal = str(query_embedding)  # "[0.1, 0.2, ...]"

    stmt = text(
        """
        SELECT
            id,
            document_id,
            chunk_index,
            content,
            parent_content,
            1 - (embedding <=> :embedding::vector) AS score
        FROM document_chunks
        WHERE tenant_id = :tenant_id
          AND embedding IS NOT NULL
        ORDER BY embedding <=> :embedding::vector
        LIMIT :top_k
        """
    )

    async with (session or SessionLocal()) as db:
        rows = await db.execute(
            stmt,
            {
                "embedding": embedding_literal,
                "tenant_id": tenant_id,
                "top_k": top_k,
            },
        )
        return [dict(r._mapping) for r in rows]
```

**The three pgvector distance operators:**

| Operator | Distance Type | When to Use |
|---|---|---|
| `<=>` | Cosine distance | ALWAYS use this for text embeddings (normalized vectors) |
| `<->` | Euclidean (L2) | Image embeddings, not normalized vectors |
| `<#>` | Negative inner product | Only when vectors are guaranteed unit-length AND you want max speed |

**Index operator class must match query operator:**
- `vector_cosine_ops` → use `<=>` in queries
- `vector_l2_ops` → use `<->` in queries
- Mismatch means the index is ignored (full table scan, silent perf disaster)

---

## Concept 7 — Hybrid Search: Vector + BM25 via Reciprocal Rank Fusion (RRF)

**Analogy:** Two detectives working the same case. The keyword detective (BM25) finds documents that contain the exact words "JWT token expiry". The semantic detective (vector) finds documents about "authentication timeout" even without those exact words. RRF combines their ranked suspect lists into one. This is why hybrid always beats either alone.

BM25 is PostgreSQL's built-in full-text search — no external service needed. It uses `tsvector`/`tsquery` types.

```python
async def hybrid_search(
    query: str,
    tenant_id: str,
    top_k: int = 20,
    rrf_k: int = 60,          # RRF smoothing constant — 60 is standard
    vector_weight: float = 0.7,  # relative weight of vector vs BM25
    session: AsyncSession = None,
) -> list[dict]:
    """
    Hybrid BM25 + vector search with Reciprocal Rank Fusion.
    Returns top_k results. Call rerank() on these before returning to LLM.
    """
    query_embedding = embed_query(query)
    embedding_literal = str(query_embedding)

    stmt = text(
        """
        WITH vector_results AS (
            SELECT
                id,
                ROW_NUMBER() OVER (ORDER BY embedding <=> :embedding::vector) AS rank
            FROM document_chunks
            WHERE tenant_id = :tenant_id
              AND embedding IS NOT NULL
            ORDER BY embedding <=> :embedding::vector
            LIMIT :top_k
        ),
        bm25_results AS (
            SELECT
                id,
                ROW_NUMBER() OVER (
                    ORDER BY ts_rank_cd(content_tsv, query) DESC
                ) AS rank
            FROM document_chunks,
                 plainto_tsquery('english', :query_text) AS query
            WHERE tenant_id = :tenant_id
              AND content_tsv @@ query
            LIMIT :top_k
        ),
        rrf AS (
            SELECT
                COALESCE(v.id, b.id) AS id,
                (
                    COALESCE(:vector_weight / (:rrf_k + v.rank), 0) +
                    COALESCE((1 - :vector_weight) / (:rrf_k + b.rank), 0)
                ) AS rrf_score
            FROM vector_results v
            FULL OUTER JOIN bm25_results b ON v.id = b.id
        )
        SELECT
            dc.id,
            dc.document_id,
            dc.chunk_index,
            dc.content,
            dc.parent_content,
            rrf.rrf_score
        FROM rrf
        JOIN document_chunks dc ON dc.id = rrf.id
        ORDER BY rrf.rrf_score DESC
        LIMIT :top_k
        """
    )

    async with (session or SessionLocal()) as db:
        rows = await db.execute(
            stmt,
            {
                "embedding": embedding_literal,
                "tenant_id": tenant_id,
                "query_text": query,
                "top_k": top_k,
                "rrf_k": rrf_k,
                "vector_weight": vector_weight,
            },
        )
        return [dict(r._mapping) for r in rows]
```

**Why RRF works:** A document ranked #1 by vector but #50 by BM25 scores: `1/(60+1) + 1/(60+50) = 0.0164 + 0.0091 = 0.0255`. A document ranked #5 by both scores: `1/(60+5) + 1/(60+5) = 0.0154 + 0.0154 = 0.0308` — the consistent winner beats the polarised winner. RRF is robust to score scale differences because it only uses ranks.

---

## Concept 8 — Reranking: Cross-Encoder After Retrieval

**Analogy:** The hybrid search shortlists 20 candidates for a job. The reranker is the hiring manager who actually reads each CV and re-scores them. Bi-encoder embeddings compress meaning; cross-encoders read query + document together and produce a precise relevance score.

Cohere Rerank v3 reads the query and each chunk simultaneously (cross-encoder architecture) and returns a relevance score. Typical workflow: retrieve top-20, rerank to top-5.

```python
import cohere
import os

co = cohere.Client(api_key=os.environ["COHERE_API_KEY"])

def rerank(
    query: str,
    chunks: list[dict],
    top_n: int = 5,
    model: str = "rerank-v3.5",
) -> list[dict]:
    """
    Rerank chunks using Cohere Rerank v3.5.
    chunks must have a 'content' key.
    Returns top_n chunks in relevance order.
    """
    if not chunks:
        return []

    documents = [c["content"] for c in chunks]
    response = co.rerank(
        query=query,
        documents=documents,
        model=model,
        top_n=top_n,
    )

    reranked = []
    for hit in response.results:
        chunk = chunks[hit.index].copy()
        chunk["rerank_score"] = hit.relevance_score
        reranked.append(chunk)

    return reranked
```

**Full retrieval pipeline (hybrid + rerank together):**

```python
async def retrieve(
    query: str,
    tenant_id: str,
    final_top_k: int = 5,
) -> list[dict]:
    """Production retrieval: hybrid search → rerank → return top_k."""
    # 1. Hybrid search returns top-20 candidates
    candidates = await hybrid_search(query, tenant_id, top_k=20)
    # 2. Reranker reads all 20, returns best 5
    return rerank(query, candidates, top_n=final_top_k)
```

---

## Concept 9 — Metadata Filtering: WHERE Clause Before ANN Search

**Analogy:** A library catalogue filtered by section before you search. You don't search every shelf for "Python" — you filter to "Computer Science > Programming Languages" first.

Metadata filtering uses normal SQL WHERE clauses BEFORE the vector similarity sort. This is much more efficient than filtering after retrieval.

```python
async def filtered_search(
    query: str,
    tenant_id: str,
    document_id: str | None = None,  # optional: narrow to one document
    top_k: int = 20,
) -> list[dict]:
    query_embedding = embed_query(query)
    embedding_literal = str(query_embedding)

    # Build dynamic WHERE clause
    filters = ["tenant_id = :tenant_id", "embedding IS NOT NULL"]
    params: dict = {"tenant_id": tenant_id, "embedding": embedding_literal, "top_k": top_k}

    if document_id:
        filters.append("document_id = :document_id")
        params["document_id"] = document_id

    where_clause = " AND ".join(filters)

    stmt = text(
        f"""
        SELECT id, document_id, chunk_index, content,
               1 - (embedding <=> :embedding::vector) AS score
        FROM document_chunks
        WHERE {where_clause}
        ORDER BY embedding <=> :embedding::vector
        LIMIT :top_k
        """  # noqa: S608 — not user-supplied table name, safe
    )

    async with SessionLocal() as db:
        rows = await db.execute(stmt, params)
        return [dict(r._mapping) for r in rows]
```

---

## Concept 10 — RAGAS Evaluation

**Analogy:** A restaurant critic who orders every dish and scores it on taste, presentation, and value separately. RAGAS is that critic for RAG — four orthogonal scores that diagnose different failure modes.

| Metric | What It Measures | Failure Mode It Catches |
|---|---|---|
| `faithfulness` | Does the answer only claim things supported by retrieved chunks? | Hallucination |
| `answer_relevancy` | Does the answer address the question? | Off-topic answers |
| `context_precision` | Are the retrieved chunks actually relevant to the question? | Retrieval noise |
| `context_recall` | Did retrieval find all the chunks needed to answer? | Missing information |

```python
# Pin RAGAS version — NaN scores occur when LLM judge returns invalid JSON in newer versions
# pip install ragas==0.1.21

import os
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)

def run_ragas_eval(
    questions: list[str],
    answers: list[str],
    contexts: list[list[str]],  # list of retrieved chunk lists, one per question
    ground_truths: list[str],   # reference answers for context_recall
) -> dict:
    """
    Run RAGAS evaluation. Returns dict of metric → score (0-1).
    Wrap in try/except — NaN scores crash evaluate() when LLM judge returns bad JSON.
    """
    ds = Dataset.from_dict(
        {
            "question": questions,
            "answer": answers,
            "contexts": contexts,
            "ground_truth": ground_truths,
        }
    )

    try:
        result = evaluate(
            ds,
            metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
        )
        return {
            "faithfulness": result["faithfulness"],
            "answer_relevancy": result["answer_relevancy"],
            "context_precision": result["context_precision"],
            "context_recall": result["context_recall"],
        }
    except Exception as e:
        # RAGAS can raise on NaN / invalid LLM judge JSON — log and return partial
        print(f"RAGAS eval error: {e}")
        return {"error": str(e)}
```

**Golden dataset — minimum viable eval:**

```python
# 50 question-answer pairs is sufficient for stable RAGAS metrics.
# Curate from real user queries, not synthetic generation.
GOLDEN_DATASET = [
    {
        "question": "What is the default chunk size?",
        "ground_truth": "512 tokens with 50-token overlap.",
    },
    # ... 49 more ...
]
```

---

## Concept 11 — Chunking Token Budget: Why 512 Tokens with 50-Token Overlap

**Analogy:** A sticky note. Big enough to hold one complete thought. Small enough to be precise when searched. The overlap is like a page header — the last line of the previous sticky note appears at the top of this one so you never lose context mid-sentence.

**Why 512 tokens:**
- Most embedding models have a hard context window. Voyage-3: 32K. But embedding quality degrades for very long sequences — the model "averages out" too much meaning.
- At 512 tokens (~380 words), one chunk = one cohesive idea. A paragraph, a function, a section. The embedding captures that one idea precisely.
- At 1024 tokens, retrieval precision drops because the embedding blends multiple ideas.
- At 256 tokens, too many chunks for any single concept — retrieval needs more of them.

**Why 50-token overlap (~10%):**
- Boundaries split sentences mid-thought. The overlap ensures both adjacent chunks contain the sentence, so neither loses the context.
- 10% overlap is the sweet spot: enough context repair, not so much that you're embedding duplicate content.
- At 20%+ overlap, storage and embedding cost grows significantly with little retrieval gain.

**Always make chunk sizes configurable:**
```python
CHUNK_SIZE = int(os.environ.get("RAG_CHUNK_SIZE", "512"))
CHUNK_OVERLAP = int(os.environ.get("RAG_CHUNK_OVERLAP", "50"))
```

---

## Concept 12 — HNSW vs IVFFlat: When to Use Each

**Analogy:** HNSW is a highway network — many routes, fast routing, expensive to build. IVFFlat is a city divided into postal codes — cheap to set up, slower to cross zones.

| Property | HNSW | IVFFlat |
|---|---|---|
| Build time | Slow (indexes every insert) | Fast (batch build required) |
| Query speed | Very fast | Fast |
| Memory | Higher | Lower |
| Incremental inserts | Yes — each insert updates index | No — must rebuild periodically |
| Best for | Production (< 2M vectors) | Batch analytics pipelines |

**Use HNSW for all production RAG.** You'll be inserting documents continuously; IVFFlat requires periodic `REINDEX` to incorporate new rows.

```sql
-- HNSW tuning parameters:
-- m = 16: connections per layer. Higher = better recall, more memory. Default 16 is good.
-- ef_construction = 64: build-time search width. Higher = better index quality, slower build.
-- ef_search (query-time): SET hnsw.ef_search = 100;  -- higher = better recall, slower
CREATE INDEX ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

**pgvector practical limit:** ~2 million vectors before query latency degrades noticeably on standard hardware. Above 2M: migrate to Qdrant or Pinecone.

---

## Concept 13 — Never Mixing Embedding Models in the Same Index

**Analogy:** Two people describing locations using different coordinate systems — one using latitude/longitude, one using a city grid. Combining them produces nonsense directions.

Each embedding model defines its own coordinate space. `voyage-3` dimensions are not comparable to `text-embedding-3-small` dimensions. Cosine similarity between vectors from different models is mathematically undefined and will silently return nonsensical scores.

```python
# Store the model name with every chunk — enforce at ingest time
class DocumentChunk(Base):
    ...
    embedding_model: Mapped[str] = mapped_column(String(100), nullable=False)
    # e.g., "voyage-3", "text-embedding-3-small", "all-MiniLM-L6-v2"

# At search time, verify model consistency
async def verify_index_model(session: AsyncSession, model: str) -> None:
    result = await session.execute(
        text("SELECT DISTINCT embedding_model FROM document_chunks LIMIT 5")
    )
    models = {r[0] for r in result}
    if models and model not in models:
        raise ValueError(
            f"Embedding model mismatch: index contains {models}, "
            f"query uses '{model}'. Re-embed all documents before switching models."
        )
```

**If you must switch models:** delete all chunk rows and re-embed from source documents. There is no upgrade path.

---

## Concept 14 — Context Injection into Claude (Retrieved Chunks → Messages)

**Analogy:** You've hired Claude as a consultant. You hand it a folder of the relevant documents you retrieved, then ask your question. The system prompt is the brief; the retrieved chunks are the folder; the user message is your question.

```python
import anthropic
import os

claude = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def build_rag_prompt(
    query: str,
    chunks: list[dict],
    use_parent_context: bool = True,
) -> list[dict]:
    """
    Convert retrieved chunks into Claude messages.
    use_parent_context=True injects parent_content instead of child content
    (hierarchical retrieval pattern — more context for the LLM).
    """
    context_parts = []
    for i, chunk in enumerate(chunks, 1):
        text = chunk.get("parent_content") or chunk["content"] if use_parent_context else chunk["content"]
        source = chunk.get("document_id", "unknown")
        context_parts.append(f"[Source {i}: {source}]\n{text}")

    context_block = "\n\n---\n\n".join(context_parts)

    return [
        {
            "role": "user",
            "content": (
                f"Use the following context to answer the question. "
                f"Only use information from the provided context. "
                f"If the context does not contain enough information, say so.\n\n"
                f"<context>\n{context_block}\n</context>\n\n"
                f"<question>{query}</question>"
            ),
        }
    ]

async def rag_answer(
    query: str,
    tenant_id: str,
    model: str = "claude-sonnet-4-5",
) -> dict:
    """Full RAG pipeline: retrieve → build prompt → generate."""
    # 1. Retrieve
    chunks = await retrieve(query, tenant_id, final_top_k=5)

    # 2. Build messages
    messages = build_rag_prompt(query, chunks)

    # 3. Generate
    response = claude.messages.create(
        model=model,
        max_tokens=1024,
        system=(
            "You are a helpful assistant. Answer questions using only the provided context. "
            "Cite source numbers [Source N] when making claims."
        ),
        messages=messages,
    )

    return {
        "answer": response.content[0].text,
        "sources": [c.get("document_id") for c in chunks],
        "chunks_used": len(chunks),
    }
```

---

## Concept 15 — Multi-Tenant RAG Isolation

**Analogy:** A shared library building where each company's documents are in locked rooms. The query can only open the door matching the requester's tenant ID. Cross-tenant contamination is the most severe production RAG bug.

```python
# EVERY query, ingest, and delete MUST include tenant_id in WHERE clause.
# Enforced by making tenant_id non-optional in all service functions.

async def delete_document(document_id: str, tenant_id: str) -> int:
    """Delete all chunks for a document. Always scoped to tenant."""
    async with SessionLocal() as db:
        result = await db.execute(
            text(
                "DELETE FROM document_chunks "
                "WHERE document_id = :doc_id AND tenant_id = :tenant_id "
                "RETURNING id"
            ),
            {"doc_id": document_id, "tenant_id": tenant_id},
        )
        await db.commit()
        return result.rowcount

# Enforce via a Pydantic base schema that all RAG requests extend
from pydantic import BaseModel, field_validator

class TenantScopedRequest(BaseModel):
    tenant_id: str

    @field_validator("tenant_id")
    @classmethod
    def tenant_id_not_empty(cls, v: str) -> str:
        if not v or not v.strip():
            raise ValueError("tenant_id is required and cannot be blank")
        return v.strip()

class SearchRequest(TenantScopedRequest):
    query: str
    document_id: str | None = None
    top_k: int = 5
```

**Database-level enforcement:** add a `CHECK` constraint so the application can never insert a chunk without a tenant:

```sql
ALTER TABLE document_chunks
ADD CONSTRAINT chk_tenant_id_not_empty
CHECK (tenant_id IS NOT NULL AND tenant_id <> '');
```

---

## Key Imports Reference Card

```python
# pgvector
from pgvector.sqlalchemy import Vector           # SQLAlchemy column type
# pip install pgvector sqlalchemy asyncpg

# Embeddings
import voyageai                                   # voyage-3
from openai import AsyncOpenAI                   # text-embedding-3-small
from sentence_transformers import SentenceTransformer  # all-MiniLM-L6-v2 (local)

# Chunking / tokenisation
import tiktoken                                  # tokeniser (pip install tiktoken)
enc = tiktoken.get_encoding("cl100k_base")

# Reranking
import cohere                                    # Cohere Rerank v3.5
# pip install cohere

# Evaluation
from datasets import Dataset                     # pip install datasets
from ragas import evaluate                       # pip install ragas==0.1.21
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)

# SQLAlchemy async
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy import text, String, Text, Integer
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# FastAPI
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, field_validator

# Anthropic
import anthropic
```

**Environment variables required:**

```bash
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/mydb
VOYAGE_API_KEY=pa-...
OPENAI_API_KEY=sk-...         # if using text-embedding-3-small
COHERE_API_KEY=...            # if using Cohere Rerank
ANTHROPIC_API_KEY=sk-ant-...

# Tunable via env — never hardcode
RAG_CHUNK_SIZE=512
RAG_CHUNK_OVERLAP=50
RAG_EMBED_BATCH_SIZE=128
