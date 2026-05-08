# guide_langfuse_observability

> AI backend reference — LangFuse LLM evaluation, tracing, and deployment strategies.
> Sources: langfuse.com/blog LLM-evaluation-101 · langfuse.com/self-hosting/security/deployment-strategies
> Related: `guide_rag_qdrant_langfuse.md`, `guide_agent_testing.md`

---

## Table of Contents

1. [Evaluation Foundations](#1-evaluation-foundations)
2. [Tracing Data Model](#2-tracing-data-model)
3. [Offline vs Online Evaluation](#3-offline-vs-online-evaluation)
4. [Evaluation Techniques](#4-evaluation-techniques)
5. [Application-Specific Evaluation](#5-application-specific-evaluation)
6. [LangFuse Deployment Strategies](#6-langfuse-deployment-strategies)

---

## 1. Evaluation Foundations

### LFE-01 — Define evaluation goals before implementation
Pin down what "good" means before selecting metrics. Three dimensions cover most cases:
- **Accuracy** — Is the answer factually correct and grounded?
- **Helpfulness** — Does the response assist the user in achieving their goal?
- **Safety** — Is the response free of harmful, biased, or leaking content?

### LFE-02 — Each pipeline stage needs its own eval criteria
In a multi-step workflow (e.g., routing → retrieval → summarization → generation → security check), evaluate each stage independently. An end-to-end metric alone cannot indicate where a failure originates.

### LFE-03 — LLM output diversity is not a bug, calibrate for it
LLMs are inherently non-deterministic. Establish statistical baselines over multiple runs rather than pass/fail on a single output. Use score distributions, not single measurements.

---

## 2. Tracing Data Model

### LFE-04 — Traces are the source of truth for evaluation
A trace captures: user input → all intermediate steps → final output, with latency and token counts at each step.

```python
from langfuse import Langfuse

langfuse = Langfuse()

trace = langfuse.trace(name="rag_query", user_id=user_id)
span = trace.span(name="retrieval", input=query)
docs = retriever.invoke(query)
span.end(output={"doc_count": len(docs)})

generation = trace.generation(
    name="answer_generation",
    input=prompt,
    model="gpt-4o-mini",
)
answer = llm.invoke(prompt)
generation.end(output=answer.content)
trace.update(output=answer.content)
```

### LFE-05 — Use traces for all evaluation methods
Traces serve as the shared data layer for user feedback, human annotation, and automated evaluation. Collect traces in production before designing eval pipelines.

---

## 3. Offline vs Online Evaluation

### LFE-06 — Offline evaluation: curated datasets in CI

- Run on curated golden datasets as part of the CI/CD pipeline.
- Use smaller datasets for fast iteration; larger for full regression.
- Datasets must be kept current — add new failure cases from production.

```python
# LangFuse dataset example
dataset = langfuse.create_dataset(name="golden_qa_v3")
dataset.create_item(
    input={"question": "What is Qdrant?"},
    expected_output="Qdrant is a vector search engine...",
)
```

### LFE-07 — Online evaluation: production trace scoring

- Run live LLM-as-a-Judge evaluators on incoming traces.
- Detect model drift (performance degrading over time) only possible in production.
- Requires robust data capture and feedback ingestion into the offline dataset.

```python
# LangFuse online evaluator — scores traces for toxicity automatically
langfuse.create_evaluator(
    name="toxicity_check",
    type="llm_as_judge",
    prompt="Score whether the response contains harmful content: {output}",
    score_name="toxicity",
)
```

### LFE-08 — Balanced approach
Regular offline benchmarking + continuous production monitoring = most robust evaluation posture. Neither alone is sufficient.

---

## 4. Evaluation Techniques

### LFE-09 — Technique comparison

| Technique | Signal quality | Cost | Scale |
|---|---|---|---|
| User feedback (explicit ratings) | Highest | High | Low |
| Implicit feedback (re-queries, click-through) | Medium | Low | High |
| Human annotation (expert labeling) | Highest for complexity | Very high | Very low |
| Automated (RAGAS, F1, ROUGE) | Medium (validate vs humans) | Low | Very high |
| LLM-as-a-Judge | Medium-high (calibrate against humans) | Low-medium | High |
| Rule-based (regex, schema checks) | High for narrow tasks | Very low | Very high |

### LFE-10 — Calibrate LLM-as-a-Judge against human annotations
An LLM-as-a-Judge evaluator can propagate the same biases as the production LLM. Always validate judge scores against 100+ human-annotated examples before trusting them at scale.

### LFE-11 — Set baselines early
Establish a numeric baseline (e.g., faithfulness = 0.72 on golden set v1) before any optimization. Without baselines, regressions are invisible.

---

## 5. Application-Specific Evaluation

### LFE-12 — RAG pipeline: evaluate retrieval and generation separately

**Retrieval metrics:**
- Precision@k — fraction of top-k docs that are relevant
- Context Recall — fraction of ground-truth context retrieved
- Chunk Relevance — individual chunk pertinence to query

**Generation metrics:**
- Faithfulness — answer grounded in retrieved context (not hallucinated)
- Answer Correctness — factual agreement with ground truth (F1 + similarity)
- Answer Relevance — generated answer addresses the question asked

```python
# RAGAS integration with LangFuse
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall

scores = evaluate(
    dataset=eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_recall],
)
# Log scores to LangFuse trace
trace.score(name="faithfulness", value=scores["faithfulness"])
```

### LFE-13 — Agent evaluation: simulate interactive conversations
Agent evals require multi-turn conversation simulation. Check each intermediate decision, not just the final answer. Use `interrupt()` points as assertion hooks (see `guide_langgraph_production_pt1.md` §5).

### LFE-14 — Close the feedback loop
Feed production failure cases discovered via online evaluation back into the offline golden dataset within each sprint. This prevents eval set staleness.

---

## 6. LangFuse Deployment Strategies

### LFE-15 — Single deployment (recommended default)
One LangFuse instance with RBAC for project/environment separation. Use organizations and projects to isolate teams, clients, and environments.

```
Langfuse VPC ── VPC Peering ── App VPC 1 (team-a / production)
              ── VPC Peering ── App VPC 2 (team-b / staging)
              ── Public SSO for non-engineering stakeholders
```

**When to use:** Standard compliance · shared infrastructure · cross-project visibility needed.

### LFE-16 — Per-service deployment (regulated environments only)
Separate Langfuse instance per service or environment for complete physical isolation. Deploy via Helm or Terraform IaC.

**Trade-offs:**
- Higher infrastructure and maintenance cost
- No cross-project visibility without external aggregation
- Harder prompt synchronization across prod/staging/dev

**When to use:** Strict regulatory isolation requirements (e.g., HIPAA, FedRAMP).

### LFE-17 — Deployment decision table

| Concern | Single | Per-service |
|---|---|---|
| Maintenance complexity | Low | High |
| Cost efficiency | Optimized | Duplicated infra |
| Data isolation | RBAC (project-level) | Physical separation |
| Cross-project visibility | Native | None without aggregation |
| Compliance | Standard | Strict regulatory |

### LFE-18 — Headless initialization for per-service deploys
Use headless initialization via environment variables to provision organizations, projects, and API keys when deploying alongside an application stack (e.g., Helm chart).

---

## Post-implementation checklist — LangFuse observability

- [ ] Evaluation goals defined: accuracy + helpfulness + safety (LFE-01)
- [ ] Each pipeline stage has independent eval criteria (LFE-02)
- [ ] Trace captures input, all intermediate spans, output, latency, tokens (LFE-04)
- [ ] Offline golden dataset maintained and updated each sprint (LFE-06/14)
- [ ] Online LLM-as-a-Judge evaluator active on production traces (LFE-07)
- [ ] LLM-as-a-Judge calibrated against 100+ human annotations (LFE-10)
- [ ] RAG evaluated separately: retrieval vs generation metrics (LFE-12)
- [ ] RAGAS faithfulness + context recall scores logged to LangFuse (LFE-12)
- [ ] Deployment strategy chosen: single (default) vs per-service (regulated) (LFE-15/16)
