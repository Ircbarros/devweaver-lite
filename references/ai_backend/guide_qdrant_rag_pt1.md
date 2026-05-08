# guide_qdrant_rag_pt1

> AI backend reference — RAG evaluation framework: metrics, tools, failure modes, and eval-driven optimization.
> Sources: qdrant.tech/blog/rag-evaluation-guide · qdrant.tech articles/rapid-rag-optimization-with-qdrant-and-quotient
> Related: `guide_qdrant_rag_pt2.md`, `guide_rag_qdrant_langfuse.md`, `guide_langfuse_observability.md`

---

## Table of Contents

1. [Why Evaluate Your RAG Pipeline](#1-why-evaluate-your-rag-pipeline)
2. [Retrieval Metrics](#2-retrieval-metrics)
3. [Generation Metrics](#3-generation-metrics)
4. [Evaluation Frameworks](#4-evaluation-frameworks)
5. [Common Failure Modes and Solutions](#5-common-failure-modes-and-solutions)
6. [Eval Dataset Construction](#6-eval-dataset-construction)
7. [Eval-Driven Optimization Loop](#7-eval-driven-optimization-loop)

---

## 1. Why Evaluate Your RAG Pipeline

### QRE-01 — Three failure stages
RAG systems can fail at retrieval (wrong docs), augmentation (outdated or gap-ridden context), or generation (hallucination, bias). Evaluate each stage independently to locate root causes.

| Stage | Failure | Symptom |
|---|---|---|
| Retrieval | Low precision / poor recall | Wrong documents returned |
| Augmentation | Context gaps / stale data | Incomplete or outdated context |
| Generation | Hallucination / bias | Answer contradicts retrieved context |

### QRE-02 — Evaluation is continuous, not one-time
RAG systems drift as data, user queries, and LLMs change. Integrate evaluation into CI/CD and continue calibrating embedding models, retrieval algorithms, and LLMs over time.

---

## 2. Retrieval Metrics

### QRE-03 — Core retrieval metrics

| Metric | What it measures | Best for |
|---|---|---|
| **Precision@k** | Fraction of top-k results that are relevant | ANN algorithm accuracy |
| **Mean Reciprocal Rank (MRR)** | Position of first relevant document in results | Ranked retrieval quality |
| **DCG / NDCG** | Relevance-weighted rank; NDCG normalizes across queries | Full ranking quality |
| **Context Relevance** | Retrieved docs collectively answer the query | RAG pipeline assessment |
| **Chunk Relevance** | Individual chunk pertinence to query | Chunking quality |

### QRE-04 — Precision@k as the primary ANN metric
For evaluating the approximate nearest-neighbor (ANN) index configuration specifically, `Precision@k` is the most appropriate metric — it directly measures how well the algorithm approximates exact search results.

---

## 3. Generation Metrics

### QRE-05 — Core generation metrics

| Metric | What it measures |
|---|---|
| **Faithfulness** | Answer grounded in retrieved context (no hallucination) — primary metric |
| **Answer Correctness** | Factual agreement with ground truth (F1 + semantic similarity) |
| **Answer Semantic Similarity** | Cosine distance between generated answer and ground truth |
| **ROUGE-L** | Lexical overlap with reference summary |
| **BERTScore** | Contextual similarity via BERT embeddings |

> **QRE-06** Faithfulness is the most critical metric for RAG. Low faithfulness = the LLM is generating answers not supported by retrieved context. Address retrieval quality first, then generation.

---

## 4. Evaluation Frameworks

### QRE-07 — RAGAS
Uses a dataset of `{question, context, answer, ground_truth}` tuples. Computes faithfulness, answer relevancy, context recall, context precision, and answer similarity.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall, context_precision

from datasets import Dataset

data = {
    "question":    [...],
    "contexts":    [...],    # list of retrieved chunks per question
    "answer":      [...],    # LLM generated answer
    "ground_truth":[...],
}
result = evaluate(Dataset.from_dict(data),
                  metrics=[faithfulness, answer_relevancy, context_recall])
print(result)
```

### QRE-08 — Arize Phoenix
Open-source tool that traces response generation step-by-step. Define "evaluators" using LLMs to score outputs, detect hallucinations, and check answer accuracy. Tracks latency, token usage, and errors.

### QRE-09 — Framework selection guide

| Use case | Recommended |
|---|---|
| Standard QA datasets with ground truth | RAGAS |
| Visual trace debugging, step-by-step audit | Arize Phoenix |
| Custom pipeline with CI integration | RAGAS + LangFuse tracing |
| Comprehensive production observability | LangFuse (see `guide_langfuse_observability.md`) |

---

## 5. Common Failure Modes and Solutions

### QRE-10 — Improper chunking
**Symptom**: High variance in context relevance; poor faithfulness.  
**Solutions**:
- Align chunk size with the token limit of the embedding model.
- Apply chunk overlap (10–20%) to preserve context across boundaries.
- Tailor chunking strategy to document type: HTML → by heading; legal → by section; code → by AST node.

### QRE-11 — Wrong embedding model
**Symptom**: Semantically similar queries return irrelevant results.  
**Solutions**:
- Use the [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) to benchmark embedding models for your domain.
- For specialized domains (legal, medical, code), evaluate domain-specific embedding models.
- FastEmbed provides lightweight model loading for quick experimentation.

### QRE-12 — Unoptimized retrieval
**Symptom**: Adequate embedding quality but still irrelevant results.  
**Solutions**:
- Match similarity metric to embedding type: `Cosine` for normalized dense vectors; `Dot Product` for unnormalized.
- Add hybrid search (dense + sparse BM-25/SPLADE) for keyword-important domains.
- Apply re-ranking with cross-encoder models after initial ANN retrieval.
- Tune hyperparameters: chunk size, chunk overlap, retrieval window size.

### QRE-13 — Poor LLM generation
**Symptom**: Good retrieval metrics but low faithfulness / answer quality.  
**Solutions**:
- Use [Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard) for model selection benchmarks.
- Experiment with prompt templates; small prompt changes cause significant quality shifts.
- Evaluate multiple LLMs on the same retrieval output before committing to one.

---

## 6. Eval Dataset Construction

### QRE-14 — Required columns

| Column | Description |
|---|---|
| `question` | Realistic queries representative of production usage |
| `ground_truth` | Precise expected answer |
| `context` | Retrieved documents for the question |
| `answer` | LLM-generated response from RAG pipeline |

### QRE-15 — Dataset creation approaches

| Method | Best for |
|---|---|
| Hand-crafted QA pairs | High control; small scale; domain-critical cases |
| LLM-generated synthetic data (T5, GPT-4) | Rapid dataset bootstrapping |
| RAGAS `TestsetGenerator` | Automated question generation with diverse types |
| Production traces + human annotation | Most representative of real usage |

---

## 7. Eval-Driven Optimization Loop

### QRE-16 — Systematic experimentation methodology
Change one variable per experiment. Track every experiment with distinct collection names or run IDs.

```
Experiment design:
  Variables: embedding model · chunk size · chunk overlap · retrieval window · LLM
  Fixed:     evaluation dataset · metrics (faithfulness primary)

Loop:
  1. Establish baseline (Exp 1)
  2. Vary ONE parameter (e.g., chunk size)  → Exp 2
  3. Vary ONE parameter (e.g., retrieval window) → Exp 3
  4. Vary ONE parameter (e.g., embedding model) → Exp 4
  5. Vary ONE parameter (e.g., LLM)             → Exp 5
  6. Pick best combination from 1–5 and fine-tune
```

### QRE-17 — Key findings from empirical experiments (Qdrant + Quotient study)
1. **Irrelevant retrieval → hallucination**: when chunk relevance is low, faithfulness drops sharply.
2. **Smaller chunks + more docs** outperforms larger chunks + fewer docs for faithfulness.
3. **Dynamic retrieval window** (query-dependent doc count) improves context relevance for hard queries.
4. **Embedding model choice** is domain-specific: benchmark on your data before committing.
5. **LLM swap** (Mistral → GPT-3.5-turbo) improved all metrics including faithfulness, suggesting generation bottlenecks are real.

---

## Post-implementation checklist — Qdrant RAG evaluation

- [ ] Three failure stages identified and evaluated independently (QRE-01)
- [ ] Faithfulness tracked as primary metric; baseline established (QRE-02/06)
- [ ] Precision@k used for ANN configuration tuning (QRE-04)
- [ ] RAGAS or equivalent framework integrated into evaluation pipeline (QRE-07)
- [ ] Eval dataset includes: question, ground_truth, context, answer columns (QRE-14)
- [ ] One variable changed per experiment; each run has a unique identifier (QRE-16)
- [ ] Chunking strategy tailored to document type (QRE-10)
- [ ] Embedding model benchmarked on domain data via MTEB (QRE-11)
- [ ] Re-ranking added if retrieval accuracy plateaus (QRE-12)
