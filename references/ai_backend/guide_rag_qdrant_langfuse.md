# guide_rag_qdrant_langfuse

> Stream A — RAG pipeline, Qdrant vector DB, LangFuse observability, and mem0 integration.  
> Related: `guide_fastapi_langgraph.md` · `guide_qdrant_rag_pt1.md` · `guide_qdrant_rag_pt2.md` · `guide_langfuse_observability.md` · `guide_mem0_memory.md`

---

## Table of Contents

1. [RAG Pipeline Overview](#1-rag-pipeline-overview)
2. [Qdrant Collection Setup](#2-qdrant-collection-setup)
3. [Hybrid Search (Dense + Sparse)](#3-hybrid-search-dense--sparse)
4. [LangFuse Tracing](#4-langfuse-tracing)
5. [mem0 Long-Term Memory](#5-mem0-long-term-memory)
6. [mem0 vs LangGraph State Boundary](#6-mem0-vs-langgraph-state-boundary)
7. [Stable Versions](#7-stable-versions)

---

## 1. RAG Pipeline Overview

```
[User Query]
    │
    ▼
[Sanitise + validate input]           ← Prompt injection mitigation
    │
    ▼
[Embed query] text-embedding-3-small   ← 1536-dim vector
    │
    ▼
[Qdrant hybrid search]                ← Dense + BM25 sparse, top-K=20
    │
    ▼
[Re-rank] cross-encoder or LLM-judge  ← Reduce to top-K=5
    │
    ▼
[Context assembly]                    ← Prepend rules from Librarian
    │
    ▼
[LLM generation] via LangGraph node   ← Traced by LangFuse
    │
    ▼
[Output validation + sanitise]
```

**Chunk strategy**: `RecursiveCharacterTextSplitter`, 512 tokens, 64-token overlap.  
**Re-ranking** (2026 best practice): Cross-encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`) preferred over LLM-as-judge for latency; use LLM-as-judge only when faithfulness > speed.

---

## 2. Qdrant Collection Setup

```python
from qdrant_client import AsyncQdrantClient
from qdrant_client.models import (
    VectorParams, Distance, SparseVectorParams,
    HnswConfigDiff, PayloadSchemaType
)

client = AsyncQdrantClient(url=QDRANT_URL, api_key=QDRANT_API_KEY)

await client.create_collection(
    collection_name="ai_backend_docs",  # One collection per domain — {domain}_docs per C-03
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    sparse_vectors_config={"bm25": SparseVectorParams()},
    hnsw_config=HnswConfigDiff(m=16, ef_construct=128)
)

# Mandatory payload indexes
for field, schema in {
    "isbn":         PayloadSchemaType.KEYWORD,    # FK to Librarian PostgreSQL documents.isbn
    "doc_type":     PayloadSchemaType.KEYWORD,
    "domain":       PayloadSchemaType.KEYWORD,
    "last_updated": PayloadSchemaType.DATETIME,
}.items():
    await client.create_payload_index("ai_backend_docs", field, schema)
```

**Per-domain collections** (9 total, per C-03 `{domain}_docs` naming): `ai_backend_docs`,
`iot_docs`, `storage_docs`, `security_docs`, `coding_docs`, `devops_docs`, `ui_ux_docs`,
`research_docs`, `containers_docs`.

**Payload mandatory fields** per point:
- `document_id` — FK back to Librarian PostgreSQL ISBN PK
- `doc_type` — `rule` | `guide` | `adr` | `issue` | `standard`
- `domain` — matches collection domain
- `last_updated` — ISO 8601 timestamp for recency scoring

---

## 3. Hybrid Search (Dense + Sparse)

```python
from qdrant_client.models import Prefetch, FusionQuery, Fusion

results = await client.query_points(
    collection_name="ai_rules",
    prefetch=[
        Prefetch(query=dense_vector, using="dense", limit=50),
        Prefetch(query=SparseVector(indices=bm25_indices, values=bm25_values),
                 using="bm25", limit=50),
    ],
    query=FusionQuery(fusion=Fusion.RRF),  # Reciprocal Rank Fusion
    limit=20,
    with_payload=True,
)
```

**Target**: top-K=20 from Qdrant → re-rank → top-K=5 for context.  
**Latency SLO**: < 150 ms at 1M vectors with HNSW (ef_construct=128).

---

## 4. LangFuse Tracing

```python
from langfuse import Langfuse
from langfuse.callback import CallbackHandler

langfuse = Langfuse(
    public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
    secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
    host=os.getenv("LANGFUSE_HOST", "http://langfuse:3000"),
)

# In LangGraph node — wrap LLM call
handler = CallbackHandler(trace_name="rag-generation", user_id=user_id)
result = await llm.ainvoke(messages, config={"callbacks": [handler]})
```

**Fail-fast rule**: If `Langfuse` client cannot connect at startup → raise and do NOT allow uninitialised calls.

```python
def _assert_langfuse_ready():
    if not langfuse.auth_enabled:  
        raise RuntimeError("LangFuse not initialised; check LANGFUSE_* env vars")
```

**Self-hosted Docker**: LangFuse runs on port 3000; env vars `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`.

---

## 5. mem0 Long-Term Memory

```python
from mem0 import AsyncMemory

_memory = AsyncMemory(config={
    "llm": {"provider": "openai", "config": {"model": "gpt-4o-mini"}},
    "embedder": {"provider": "openai", "config": {"model": "text-embedding-3-small"}},
    "vector_store": {
        "provider": "qdrant",
        "config": {"host": QDRANT_HOST, "port": 6333, "collection_name": "user_memories"}
    },
})

# Store after interaction (fire-and-forget to avoid adding latency)
asyncio.create_task(_memory.add(messages=messages, user_id=user_id, infer=True))

# Retrieve before next LLM call
results = await _memory.search(query=user_query, filters={"user_id": user_id}, top_k=5)
memories = [r["memory"] for r in results.get("results", [])]
```

**Security**: Redact PII before `.add()`; enforce `user_id` scoping at the API layer.

**Known issues** (v0.1.x, pre-1.0, ⚠️ beta):
- Entity linking may fail silently; validate in production.
- Search adds ~100–300 ms (Qdrant + optional LLM re-rank).
- Memory format not migration-guaranteed across patch versions.

---

## 6. mem0 vs LangGraph State Boundary

| What | Where stored |
|---|---|
| Long-term user preferences, cross-session facts | **mem0** (Qdrant-backed) |
| Current task flow, active steps, messages | **LangGraph state** (PostgresSaver) |
| Transient retrieved context (single request) | **LangGraph state** (`messages`) |
| Canonical rules/standards (persistent) | **Librarian** PostgreSQL + Qdrant |

**Rule**: mem0 is read once at session start to seed LangGraph state; not polled during graph execution.

---

## 7. Stable Versions

| Package | Version | Stability |
|---|---|---|
| `qdrant-client` (async) | `1.9.x` | ✅ Stable |
| `opentelemetry-sdk` | `1.22.x` | ✅ Stable |
| `langfuse` | `2.x` | ✅ Stable |
| `mem0ai` | `0.1.12` | ⚠️ Beta (pre-1.0) |
| `langchain-text-splitters` | `0.2.x` | ✅ Stable |
| `fastembed` | `0.3.x` | ✅ Stable |

**Flagged**: `mem0ai` is < 1.0; pin exact version and test upgrades in isolation.
