# guide_agent_testing

> AI backend reference — Agent testing pyramid: unit tests, evals, simulations, and the Scenario framework.
> Sources: langwatch.ai/scenario · langwatch.ai the-agent-testing-pyramid
> Related: `guide_langfuse_observability.md`, `guide_langgraph_production_pt1.md`

---

## Table of Contents

1. [The Agent Testing Pyramid](#1-the-agent-testing-pyramid)
2. [Layer 1 — Unit Tests](#2-layer-1--unit-tests)
3. [Layer 2 — Evals & Optimization](#3-layer-2--evals--optimization)
4. [Layer 3 — Simulations](#4-layer-3--simulations)
5. [Scenario Framework (LangWatch)](#5-scenario-framework-langwatch)

---

## 1. The Agent Testing Pyramid

Three layers form the foundation of a robust agent testing strategy. Each layer catches different failure classes and serves a different stage of development maturity.

```
          /\
         /  \
        / L3 \        Simulations — multi-turn, binary outcomes
       /------\
      /   L2   \      Evals & Optimization — RAG accuracy, LLM quality, DSPy
     /----------\
    /     L1     \    Unit Tests — APIs, pipelines, memory, auth, rate limits
   /______________\
```

> **AT-01** Apply pyramid proportions to team maturity: early-stage may skip some L1 coverage; mature systems invest heavily in L2. Never skip L3 simulation for any production-critical path.

| Layer | Purpose | When to invest |
|---|---|---|
| L1 Unit Tests | Verify individual components work correctly | From day one |
| L2 Evals & Optimization | Measure LLM / retrieval quality; optimize prompts | Once basic pipeline works |
| L3 Simulations | Test end-to-end agent behavior; binary business outcomes | Before production release |

---

## 2. Layer 1 — Unit Tests

### AT-02 — What to test at unit level

| Component | Test focus |
|---|---|
| API connections | Correct auth headers, timeout handling, error propagation |
| Data pipelines | Chunking, embedding, upsert — verify expected output shape |
| Memory storage/retrieval | Write + read roundtrip; scoping by `user_id` / `session_id` |
| Authentication | Valid JWT accepted; expired/tampered JWT rejected |
| Rate limiting | Requests above threshold are rejected with `429` |

```python
# Example: memory unit test
def test_memory_isolation(mem_client):
    mem_client.add("user_a_secret", user_id="user_a")
    results = mem_client.search("secret", user_id="user_b")
    assert len(results) == 0   # user b cannot see user a's memories
```

### AT-03 — Mock external dependencies
All calls to LLMs, vector DBs, and external APIs must be mocked at L1. Tests should be deterministic and fast (<100 ms each).

```python
@pytest.fixture
def mock_qdrant(monkeypatch):
    monkeypatch.setattr("myapp.retriever.qdrant_client.query_points",
                        lambda *a, **kw: MOCK_SEARCH_RESULTS)
```

---

## 3. Layer 2 — Evals & Optimization

### AT-04 — RAG retrieval metrics

| Metric | Measures | Tool |
|---|---|---|
| Precision@k | Fraction of top-k results that are relevant | RAGAS |
| Context Relevance | Retrieved docs answer the query | RAGAS / Quotient |
| Chunk Relevance | Individual chunks contain pertinent information | Quotient |
| Faithfulness | LLM answer grounded in retrieved context | RAGAS |
| Answer Semantic Similarity | Cosine distance between answer and ground truth | RAGAS |

### AT-05 — LLM response quality metrics

| Metric | Description |
|---|---|
| ROUGE-L | Overlap between generated and reference summaries |
| BERTScore | Contextual similarity via BERT embeddings |
| Hallucination rate | Responses containing ungrounded claims |
| Human annotation | Expert scores for clarity, coherence, relevance |

### AT-06 — Prompt optimization with DSPy
Use DSPy to compile and optimize prompts against L2 eval metrics before deploying prompt changes to production.

```python
import dspy
from dspy.evaluate import Evaluate

program = dspy.ChainOfThought("question, context -> answer")
evaluator = Evaluate(devset=eval_set, metric=faithfulness_metric)
optimized = dspy.MIPROv2().compile(program, trainset=train_set)
```

### AT-07 — Fine-tuning loop
Apply RLHF or GRPO fine-tuning only after exhausting prompt optimization. Track fine-tuning experiments with LangFuse or MLflow.

---

## 4. Layer 3 — Simulations

### AT-08 — Binary outcome framing
Frame every simulation as a binary question that maps to business value:
- "Can the agent cancel an order without an order number?" → yes / no
- "Does the agent escalate correctly when safety threshold is triggered?" → yes / no
- "Can the agent complete checkout across 3 tool-call turns?" → yes / no

Binary outcomes are directly reportable to non-technical stakeholders as pass/fail test suites.

### AT-09 — Multi-turn conversation simulation
Simulations drive a complete conversation with the agent across multiple turns, including edge cases and adversarial inputs.

```python
# Scenario framework (LangWatch) — pseudocode pattern
import scenario

@scenario.test
async def test_cancel_without_order_number():
    result = await scenario.run(
        agent=my_agent,
        conversation=[
            scenario.user("I want to cancel my order"),
            scenario.agent_should("ask for order number or email"),
            scenario.user("I don't have the order number"),
            scenario.agent_should("offer alternative verification"),
        ],
    )
    assert result.passed
```

### AT-10 — Edge case coverage
Each simulation suite MUST cover:
- Happy path (expected, clean input)
- Missing required fields (partial input)
- Adversarial input (prompt injection attempt, out-of-scope request)
- Boundary conditions (maximum allowed turns, maximum token budget)
- Multi-language or locale variants if the agent is internationalized

---

## 5. Scenario Framework (LangWatch)

### AT-11 — Scenario library setup
`langwatch/scenario` is available in Python, TypeScript, and Go. It integrates with LangGraph, CrewAI, and PydanticAI agents via a `call()` interface.

```python
# pip install langwatch-scenario

import scenario
from scenario import AgentInput, AgentResponse

class MyLangGraphAgent(scenario.Agent):
    async def call(self, input: AgentInput) -> AgentResponse:
        result = await langgraph_app.ainvoke(
            {"messages": input.messages},
            config={"configurable": {"thread_id": input.thread_id}},
        )
        return AgentResponse(messages=result["messages"])
```

### AT-12 — Integrate Scenario into CI
Run simulation tests in CI as part of the pull-request gate. Use `--marker simulation` to separate simulation runs from fast unit tests.

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["unit", "eval", "simulation"]
```

```bash
# Fast gate (PR) — unit only
pytest -m unit

# Full gate (main merge) — all layers
pytest -m "unit or eval or simulation"
```

---

## Post-implementation checklist — agent testing

- [ ] Unit tests cover: API auth, data pipeline, memory isolation, rate limiting (AT-02)
- [ ] All external dependencies mocked at L1; tests are deterministic (AT-03)
- [ ] Retrieval evaluated with Precision@k, Context Relevance, Faithfulness (AT-04/05)
- [ ] Prompt changes validated with DSPy before production (AT-06)
- [ ] Simulations framed as binary outcomes with business-value mapping (AT-08)
- [ ] Each simulation suite covers happy path + missing fields + adversarial input (AT-10)
- [ ] Scenario agent wraps LangGraph `ainvoke` via `call()` interface (AT-11)
- [ ] CI gate separates unit (fast) from simulation (full merge) (AT-12)
