---
description: RAG and pgvector rules. Loads for projects with embeddings, vector search, or retrieval.
paths: ["*.py", "app/**/*.py"]
---

# RAG + pgvector Rules

Rules are numbered for traceability. Reference them in code comments and PR reviews.

---

## pgvector Setup

1. ALWAYS run `CREATE EXTENSION IF NOT EXISTS vector` in a migration before creating any vector column — the extension does not exist by default.
2. ALWAYS specify the exact dimension count in the Vector column: `Vector(1024)`, `Vector(1536)`, `Vector(384)` — dimension mismatch raises a hard error at insert time.
3. ALWAYS create an HNSW index, not IVFFlat, for production: `USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64)`.
4. NEVER use IVFFlat in production if you are inserting documents continuously — IVFFlat requires periodic `REINDEX` to include new rows in the index.
5. ALWAYS match the index operator class to the query operator: `vector_cosine_ops` → `<=>`, `vector_l2_ops` → `<->`. Mismatch causes a silent full table scan.
6. ALWAYS create the HNSW index in an Alembic migration using `op.execute(raw SQL)` — SQLAlchemy DDL does not support pgvector index syntax natively.
7. ALWAYS add a `NOT NULL` constraint or `CHECK` on `tenant_id` at the database level — application-layer enforcement alone is insufficient.
8. ALWAYS add a GIN index on the `content_tsv` column for BM25: `CREATE INDEX ON document_chunks USING GIN (content_tsv)`.
9. ALWAYS maintain a `content_tsv tsvector` column and update it on insert/update via trigger or explicit `UPDATE ... SET content_tsv = to_tsvector('english', content)`.
10. NEVER exceed 2 million vectors in a single pgvector table without benchmarking query latency — migrate to Qdrant or Pinecone above that threshold.

---

## Embedding Models

11. NEVER hardcode API keys — always read from `os.environ["VOYAGE_API_KEY"]`, `os.environ["OPENAI_API_KEY"]`, `os.environ["COHERE_API_KEY"]`.
12. ALWAYS set `input_type="document"` when embedding chunks at ingest time and `input_type="query"` when embedding queries at search time — Voyage-3 optimises vectors differently for each role.
13. NEVER mix embedding models in the same vector index — cosine similarity between vectors from different models is mathematically meaningless.
14. ALWAYS store the embedding model name alongside every chunk: `embedding_model VARCHAR(100) NOT NULL`. This lets you detect and block model mismatches at query time.
15. ALWAYS verify index model consistency at search time: if stored `embedding_model` differs from the current query model, raise an error rather than returning garbage scores.
16. NEVER switch embedding models without re-embedding ALL documents from source — there is no in-place upgrade path.
17. ALWAYS embed in batches (default 128) to respect API rate limits — never embed one chunk per HTTP request.
18. ALWAYS implement exponential backoff (2^attempt seconds, 3 attempts) around embedding API calls — rate limit errors are transient.
19. NEVER embed raw HTML — strip tags, normalise whitespace, and handle encoding before passing text to an embedding model.
20. ALWAYS normalise whitespace before embedding: strip leading/trailing whitespace, collapse multiple spaces, remove zero-width characters.

---

## Chunking

21. ALWAYS make chunk size configurable via environment variable: `CHUNK_SIZE = int(os.environ.get("RAG_CHUNK_SIZE", "512"))`.
22. ALWAYS make overlap configurable via environment variable: `CHUNK_OVERLAP = int(os.environ.get("RAG_CHUNK_OVERLAP", "50"))`.
23. NEVER hardcode chunk sizes — different document types (code, prose, legal) need different values.
24. ALWAYS measure chunk sizes in tokens using tiktoken, not characters — character counts vary wildly across languages and punctuation density.
25. ALWAYS use `cl100k_base` tokeniser for token counting (compatible with OpenAI and Voyage models).
26. ALWAYS test at least two chunking strategies on real documents before choosing one — empirical testing beats theoretical reasoning.
27. ALWAYS apply contextual retrieval: prepend document title and section header to each chunk before embedding (`"Document: {title}\nSection: {header}\n\n{chunk}"`). This improves retrieval by 15-25%.
28. NEVER store the enriched (prefixed) text as the chunk content — store the original text and reconstruct the enriched version at embed time only. The LLM receives the original text.
29. ALWAYS deduplicate chunks before inserting — duplicate chunks inflate storage and pollute retrieval rankings.
30. ALWAYS preserve source metadata in every chunk: `document_id`, `chunk_index`, `section_header`, `page_number` if available.
31. ALWAYS use hierarchical chunking for long documents: embed small child chunks (512 tokens) for retrieval precision, store parent chunks (1024-2048 tokens) for LLM context richness.
32. NEVER chunk without checking for empty or near-empty chunks — filter out chunks under 20 tokens before embedding.
33. ALWAYS chunk code files by logical unit (function, class) rather than fixed token count — a 512-token window splitting mid-function produces useless chunks.

---

## Ingestion Pipeline

34. ALWAYS validate document text before chunking: strip null bytes, enforce UTF-8, reject empty strings.
35. ALWAYS run ingestion inside a transaction — either all chunks for a document succeed or none do.
36. ALWAYS store `document_id` with every chunk so you can delete all chunks for a document when it is updated or removed.
37. ALWAYS delete existing chunks for a document before re-ingesting it — never accumulate stale duplicate versions.
38. ALWAYS log ingestion duration and chunk count per document — this detects slow embedding API calls early.
39. NEVER ingest synchronously in a FastAPI request handler for large documents — use a background task or queue.
40. ALWAYS track ingestion status per document (pending / ingesting / complete / failed) in a separate table — gives users visibility and enables retry.

---

## Similarity Search

41. ALWAYS use the `<=>` cosine distance operator for text embeddings — never `<->` (L2) unless your embedding model explicitly produces non-normalised vectors.
42. ALWAYS filter by `tenant_id` in the WHERE clause BEFORE the ORDER BY vector distance — pre-filtering is cheaper than post-filtering.
43. NEVER apply `similarity_threshold` filtering as a post-retrieval step — add `1 - (embedding <=> query_vec) >= threshold` in the WHERE clause so the index can skip low-similarity rows.
44. ALWAYS parameterise the embedding literal as a bound parameter, not an f-string: use `text("... <=> :embedding::vector")` with `{"embedding": str(vector)}`.
45. NEVER return more than top-20 candidates from raw vector search — pass them to a reranker, not directly to the LLM.
46. ALWAYS request top-20 from retrieval and rerank to top-5 before injecting into the LLM context.
47. NEVER use vector-only search in production — always combine with BM25 via hybrid search.

---

## Hybrid Search (BM25 + RRF)

48. ALWAYS implement hybrid search using PostgreSQL native `tsvector`/`tsquery` for BM25 — do not add an external search engine (Elasticsearch, Typesense) unless pgvector BM25 is insufficient.
49. ALWAYS use Reciprocal Rank Fusion (RRF) to merge vector and BM25 results: `1 / (rrf_k + rank)` per result list, then sum. Default `rrf_k = 60`.
50. ALWAYS use `plainto_tsquery('english', :query)` for BM25 queries — not `to_tsquery` (requires user-formatted boolean query syntax).
51. ALWAYS use `FULL OUTER JOIN` between vector and BM25 result sets in RRF — a document appearing in only one list still gets a score.
52. NEVER tune `vector_weight` vs BM25 weight manually without measuring RAGAS context_precision before and after — the default 70/30 split is a starting point, not a law.
53. ALWAYS expose `vector_weight` as a configurable parameter, not a hardcoded float.

---

## Reranking

54. ALWAYS rerank after hybrid search — raw similarity scores (cosine or BM25) do not account for fine-grained semantic nuance.
55. ALWAYS retrieve top-20 candidates and rerank to top-5 — rerankers are slower than ANN search; limit their input.
56. NEVER pass more than 20 documents to Cohere Rerank in a single call — performance degrades and latency spikes above that.
57. ALWAYS use `rerank-v3.5` (Cohere) as the default reranking model — it outperforms v2 on technical content.
58. ALWAYS copy chunk metadata (document_id, chunk_index) to the reranked output — the reranker returns only index positions, not full chunk data.
59. ALWAYS add `rerank_score` to each returned chunk so callers can apply a threshold if needed.
60. NEVER block the FastAPI response on synchronous reranking calls — wrap in `asyncio.to_thread` if using the synchronous Cohere client.

---

## Context Assembly (Injecting into LLM)

61. ALWAYS wrap retrieved context in explicit XML-style delimiters before injecting into the prompt: `<context>...</context>` and `<question>...</question>`. This prevents prompt injection from chunk content.
62. ALWAYS label each source chunk with a number `[Source N: document_id]` so the LLM can cite it.
63. ALWAYS use `parent_content` instead of `child_content` when injecting into the LLM — child chunks are optimised for retrieval precision, parent chunks provide the narrative context the LLM needs.
64. NEVER inject more than 5 chunks into a single Claude message unless the combined token count is verified to stay under the model's context window.
65. ALWAYS instruct Claude (in system prompt) to only use information from the provided context — this reduces hallucination.
66. ALWAYS return the source document IDs alongside the generated answer — the caller needs them for attribution and audit.
67. NEVER pass raw user input directly into the context injection without sanitisation — a user could embed prompt injection in their query.

---

## Multi-Tenant Isolation

68. ALWAYS include `tenant_id` as a non-optional parameter in every service function that reads from or writes to the vector index — never make it optional with a default of `None`.
69. ALWAYS add `AND tenant_id = :tenant_id` to every SQL query touching `document_chunks` — no exceptions, including COUNT queries and DELETE operations.
70. ALWAYS validate that `tenant_id` is non-empty in Pydantic request schemas before the request reaches the service layer.
71. NEVER allow a user to supply their own `tenant_id` from a client-facing API without verifying it matches their authenticated identity — derive `tenant_id` from the auth token, not from the request body.
72. ALWAYS scope document deletes by both `document_id` AND `tenant_id` — deleting by `document_id` alone risks cross-tenant data loss if IDs collide.
73. ALWAYS add a database-level CHECK constraint: `CHECK (tenant_id IS NOT NULL AND tenant_id <> '')`.

---

## Evaluation (RAGAS)

74. ALWAYS pin the RAGAS version in requirements: `ragas==0.1.21` — newer versions have breaking Dataset schema changes and intermittent NaN scores.
75. ALWAYS wrap `evaluate()` in a `try/except Exception` — RAGAS raises when the LLM judge returns invalid JSON, which happens non-deterministically.
76. ALWAYS build a golden dataset of at least 50 question/ground_truth pairs from real user queries before shipping — synthetic datasets miss distribution of actual usage.
77. NEVER evaluate RAG quality by vibe or manual inspection alone — always compute all four metrics: faithfulness, answer_relevancy, context_precision, context_recall.
78. ALWAYS run RAGAS evaluation before AND after any chunking or retrieval change — treat score regression as a blocking issue.
79. ALWAYS pass `contexts` as `list[list[str]]` to RAGAS — a flat `list[str]` causes silent evaluation errors.
80. ALWAYS pass `ground_truth` for `context_recall` — without it, context_recall returns NaN.

---

## Performance

81. ALWAYS set `hnsw.ef_search` at query time for precision/speed tradeoff: `SET hnsw.ef_search = 100` for production, `= 40` for development. Higher = better recall, more latency.
82. ALWAYS use `LIMIT` before the reranker, not after — the database does the heavy lifting, the reranker does fine-grained scoring on a small set.
83. NEVER run embedding generation synchronously in a write path that is under SLA — use a queue (Celery, ARQ) for background ingestion.
84. ALWAYS batch embedding calls: 128 documents per API call for Voyage, 2048 for OpenAI text-embedding-3-small.
85. ALWAYS add a composite index on `(tenant_id, document_id)` for efficient per-document deletes and existence checks.
86. NEVER use `SELECT *` from `document_chunks` in retrieval queries — always select only the columns needed (id, content, parent_content, document_id, chunk_index, score).
87. ALWAYS monitor embedding API latency per batch and log it — spikes indicate rate limiting or upstream degradation before it affects users.
