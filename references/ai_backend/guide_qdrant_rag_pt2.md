# guide_qdrant_rag_pt2

> AI backend reference — Qdrant vector search production patterns: text extraction, chunking, embeddings, search boosting, query optimization.
> Sources: qdrant.tech/blog/hitchhikers-guide · qdrant.tech/blog/rag-evaluation-guide
> Related: `guide_qdrant_rag_pt1.md`, `guide_rag_qdrant_langfuse.md`

---

## Table of Contents

1. [Text Extraction](#1-text-extraction)
2. [Chunking Strategies](#2-chunking-strategies)
3. [Embedding Models](#3-embedding-models)
4. [Hybrid Search](#4-hybrid-search)
5. [Search Boosting](#5-search-boosting)
6. [Query Optimization](#6-query-optimization)
7. [Qdrant Collection Setup Patterns](#7-qdrant-collection-setup-patterns)

---

## 1. Text Extraction

### QRP-01 — Match parser to document type

| Document type | Recommended approach |
|---|---|
| Text-only PDFs / plain documents | PyPDF / PyMuPDF — fast, cheap |
| Documents with tables, images, scanned pages | LlamaParse (agentic/OCR-based) |
| Image-dense or scanned PDFs | ColPali / ColQwen (visual language retrieval) |
| HTML / markdown | Custom parsers; strip boilerplate before chunking |

> **QRP-02** Clean, well-structured raw text is the highest-ROI investment in RAG quality. Garbage in → garbage vectors.

### QRP-03 — Validate extraction output before chunking
Spot-check extracted text on 10% of documents before building the index. Poor extraction quality causes consistently irrelevant retrieval that is hard to attribute later.

---

## 2. Chunking Strategies

### QRP-04 — Chunking strategy selection

| Chunking type | When to use |
|---|---|
| Sentence / token-based | Text that reads sentence-by-sentence; dense Q&A content |
| Semantic / embedding-based (late chunking) | Paragraphs; topic-coherent units |
| Agentic / neural chunking | Higher-order semantic units (e.g., "all info about X in a paper") |
| Fixed-size + overlap | Baseline; fast to implement; always start here |
| AST-based (for code) | Code repositories; split by function/class boundary |

### QRP-05 — Chunking parameter defaults and effects

| Parameter | Effect if too small | Effect if too large |
|---|---|---|
| `chunk_size` | High chunk relevance; low context relevance; fragmented answers | Low chunk relevance; higher faithfulness variance |
| `chunk_overlap` | Context breaks across boundary; coherence loss | Duplicate information; wasted tokens |
| Retrieval window `k` | Misses relevant docs | Dilutes context; "Lost in the Middle" risk |

> **QRP-06** Start with `chunk_size=512`, `chunk_overlap=64`, `k=3`. Increase `k` before increasing `chunk_size`. Evaluate with RAGAS after each change — see `guide_qdrant_rag_pt1.md` §7.

---

## 3. Embedding Models

### QRP-07 — Dense vs sparse embeddings

| Type | Captures | Best for |
|---|---|---|
| Dense (e.g., `bge-small-en`, `text-embedding-3-small`) | Semantic meaning | General-purpose retrieval |
| Sparse (BM-25, SPLADE, BM-42) | Keywords / exact matches | Legal, medical, code, product search |
| Multivector (ColBERT) | Token-level alignment | High-precision passage retrieval |

### QRP-08 — Model selection
Use the [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) as the benchmark reference. Prioritize:
1. **Retrieval performance** on BEIR subsets closest to your domain.
2. **Model size** trade-off: `bge-small-en` (384 dims) vs `bge-large-en` (1024 dims) vs OpenAI `text-embedding-3-large` (3072 dims).
3. **FastEmbed** for lightweight local loading compatible with Qdrant's native embedding API.

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(url=settings.QDRANT_URL, api_key=settings.QDRANT_API_KEY)

# Use Qdrant's native FastEmbed-backed embedding
client.upsert(
    collection_name="knowledge_base",
    points=[
        models.PointStruct(
            id=point_id,
            vector=models.Document(text=chunk, model="BAAI/bge-small-en"),
            payload={"source": source, "tenant_id": tenant_id},
        )
    ],
)
```

---

## 4. Hybrid Search

### QRP-09 — Combine dense + sparse for keyword-important domains
Hybrid search uses both dense (semantic) and sparse (keyword) retrieval, then fuses or re-scores the results.

```python
from qdrant_client.models import SparseVector, NamedSparseVector, NamedVector

results = client.query_points(
    collection_name="knowledge_base",
    prefetch=[
        models.Prefetch(query=query_dense_vector, using="dense", limit=20),
        models.Prefetch(query=models.SparseVector(indices=sparse_idx, values=sparse_vals),
                        using="sparse", limit=20),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),  # Reciprocal Rank Fusion
    limit=5,
)
```

### QRP-10 — Hybrid search decision guide

| Domain | Hybrid search needed? |
|---|---|
| General knowledge Q&A | Optional (dense usually sufficient) |
| Legal / regulatory documents | Yes — exact term matching critical |
| Medical / clinical records | Yes — ICD codes, drug names |
| Code search | Yes — function names, error codes |
| E-commerce product catalog | Yes — product codes, SKUs |

---

## 5. Search Boosting

### QRP-11 — Semantic caching
Store vectorized representations of past queries with their answers in a small secondary Qdrant collection. Before running the full RAG pipeline, check the cache for semantically similar previous queries.

```python
CACHE_COLLECTION = "semantic_cache"
CACHE_SIMILARITY_THRESHOLD = 0.95

async def check_semantic_cache(query: str) -> str | None:
    query_vector = embed(query)
    results = client.query_points(
        collection_name=CACHE_COLLECTION,
        query=query_vector,
        score_threshold=CACHE_SIMILARITY_THRESHOLD,
        limit=1,
    ).points
    if results:
        return results[0].payload["answer"]
    return None

async def store_in_cache(query: str, answer: str):
    client.upsert(CACHE_COLLECTION, points=[
        models.PointStruct(
            id=uuid4().hex,
            vector=embed(query),
            payload={"answer": answer, "cached_at": datetime.utcnow().isoformat()},
        )
    ])
```

### QRP-12 — Binary quantization for large-scale collections
For millions of vectors (especially high-dimension embeddings like 1536-dim OpenAI), binary quantization reduces memory footprint by ~32x. Combine with re-scoring to recover precision.

```python
client.create_collection(
    collection_name="large_scale_kb",
    vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
    quantization_config=models.BinaryQuantization(
        binary=models.BinaryQuantizationConfig(always_ram=True)
    ),
)

# Search with oversampling + rescoring to recover precision
results = client.query_points(
    collection_name="large_scale_kb",
    query=query_vector,
    search_params=models.SearchParams(
        quantization=models.QuantizationSearchParams(rescore=True, oversampling=2.0)
    ),
    limit=5,
)
```

---

## 6. Query Optimization

### QRP-13 — Query transformation by type

| Query type | Strategy |
|---|---|
| Generic / too broad | **HyDE** (Hypothetical Document Embedding): expand query into a synthetic document, embed and retrieve |
| Specific / targeted | No transformation — retrieve directly |
| Complex / multi-faceted | **Decomposition**: split into sub-queries; multi-step retrieval; combine results |

```python
# HyDE: generate hypothetical document, embed it, retrieve
async def hyde_retrieve(query: str, llm, retriever) -> list[str]:
    hypothesis = await llm.ainvoke(
        f"Write a short paragraph that answers: {query}"
    )
    return retriever.invoke(hypothesis.content)
```

### QRP-14 — Eval-first query optimization
Before adding query transformation, measure baseline retrieval quality. Add HyDE or decomposition only if Precision@k or Context Relevance is measurably below threshold. Complexity without measured gain is waste.

---

## 7. Qdrant Collection Setup Patterns

### QRP-15 — Multi-tenant collection isolation strategies

| Strategy | When to use | Trade-off |
|---|---|---|
| **Payload filter per query** | ≤ 10,000 tenants | Simple; single collection; filter adds latency |
| **Named vectors per tenant** | High-performance multi-tenant | Complex schema; good isolation |
| **Separate collection per tenant** | Strict regulatory isolation | Operational complexity at scale |

```python
# Payload filter — recommended for most multi-tenant cases
results = client.query_points(
    collection_name="shared_kb",
    query=query_vector,
    query_filter=models.Filter(must=[
        models.FieldCondition(key="tenant_id", match=models.MatchValue(value=tenant_id))
    ]),
    limit=5,
)
```

### QRP-16 — Index configuration for production
```python
client.create_collection(
    collection_name="production_kb",
    vectors_config=models.VectorParams(
        size=384,
        distance=models.Distance.COSINE,
        on_disk=True,     # store vectors on disk for very large collections
    ),
    hnsw_config=models.HnswConfigDiff(
        m=16,             # higher m = better recall, more RAM
        ef_construct=100, # higher = better index quality, slower build
    ),
    optimizers_config=models.OptimizersConfigDiff(
        indexing_threshold=20_000,  # index segments once they exceed this size
    ),
)
```

---

## Post-implementation checklist — Qdrant RAG production

- [ ] Parser matched to document type; extraction spot-checked on 10% of docs (QRP-01/03)
- [ ] Chunking strategy tailored to document type; started with `size=512, overlap=64` (QRP-04/06)
- [ ] Embedding model benchmarked on domain data via MTEB (QRP-08)
- [ ] Hybrid search added for keyword-critical domains (QRP-10)
- [ ] Semantic cache in place to reduce redundant LLM calls (QRP-11)
- [ ] Binary quantization + re-scoring configured for collections > 1M vectors (QRP-12)
- [ ] Query transformation (HyDE / decomposition) added only after measuring baseline (QRP-14)
- [ ] Multi-tenant queries scoped via payload filter with `tenant_id` (QRP-15)
- [ ] HNSW params tuned; `on_disk=True` for very large collections (QRP-16)
