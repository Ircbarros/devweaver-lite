# guide_langgraph_production_pt2

> AI backend reference — LangGraph production patterns: memory, streaming, testing, deployment, security.
> Sources: swarnendu.de/langgraph-best-practices
> Related: `guide_langgraph_production_pt1.md`, `guide_fastapi_langgraph.md`

---

## Table of Contents

1. [Memory & Persistence](#1-memory--persistence)
2. [Streaming](#2-streaming)
3. [Testing](#3-testing)
4. [Deployment](#4-deployment)
5. [Security](#5-security)

---

## 1. Memory & Persistence

### LP-17 — PostgresSaver for production
Use `PostgresSaver` (or `AsyncPostgresSaver`) for all production deployments. `MemorySaver` (in-process dict) is for local development only.

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from psycopg_pool import AsyncConnectionPool

pool = AsyncConnectionPool(conninfo=settings.DB_URL, max_size=20)
checkpointer = AsyncPostgresSaver(pool)

app = graph.compile(checkpointer=checkpointer)
```

### LP-18 — thread_id as first-class key
Always scope conversations with a structured `thread_id`. Never use bare UUIDs.

```python
# Good: structured and traceable
thread_id = f"tenant-{tenant_id}:user-{user_id}:session-{session_id}"
config = {"configurable": {"thread_id": thread_id}}

result = await app.ainvoke(input_state, config=config)
```

### LP-19 — checkpoint_ns per tenant
Use `checkpoint_ns` to isolate checkpoints per tenant, preventing cross-tenant data leakage.

```python
config = {
    "configurable": {
        "thread_id": thread_id,
        "checkpoint_ns": f"tenant-{tenant_id}",
    }
}
```

### LP-20 — InMemoryStore namespace convention
For long-term cross-session preferences, scope `InMemoryStore` (and production equivalents) by `(tenant, user, category)`.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
namespace = (tenant_id, user_id, "preferences")
store.put(namespace, key="language", value={"lang": "en"})
prefs = store.search(namespace)
```

---

## 2. Streaming

### LP-21 — Stream mode selection

| Mode | Use case | Bandwidth |
|---|---|---|
| `messages` | Chat UX — token-by-token text | High |
| `updates` | Dashboards — state delta per node | Low (recommended default) |
| `values` | Full state snapshots (debug, audit) | High |
| `custom` | Tailored payloads via `StreamWriter` | Variable |

```python
# Updates stream — bandwidth-friendly for dashboards
async for chunk in app.astream(state, config=config, stream_mode="updates"):
    node_name, delta = list(chunk.items())[0]
    await websocket.send_json({"node": node_name, "delta": delta})

# Messages stream — token UX for chat
async for chunk in app.astream(state, config=config, stream_mode="messages"):
    token, metadata = chunk
    await websocket.send_text(token.content)
```

### LP-22 — Send API for parallel fan-out streaming
```python
from langgraph.types import Send

def supervisor_node(state: AgentState) -> list[Send]:
    return [
        Send("specialist_node", {"task": t})
        for t in state["tasks"]
    ]
```

---

## 3. Testing

### LP-23 — Test graphs, not just functions
Test via `invoke`/`ainvoke` on the compiled graph. Assert the resulting state and the last node executed. Unit-testing individual node functions alone is insufficient.

```python
import pytest
from langgraph.graph import StateGraph

@pytest.mark.asyncio
async def test_rag_graph_happy_path(rag_graph, mock_retriever):
    result = await rag_graph.ainvoke(
        {"messages": [HumanMessage(content="What is Qdrant?")],
         "context": [], "error": None, "step_count": 0},
        config={"configurable": {"thread_id": "test-001"}},
    )
    assert result["answer"]
    assert result["error"] is None
    assert result["step_count"] < MAX_STEPS
```

### LP-24 — Mock LLM and tools deterministically
```python
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_llm():
    with patch("myapp.nodes.llm") as mock:
        mock.ainvoke = AsyncMock(return_value=AIMessage(content="Mocked answer"))
        yield mock
```

### LP-25 — Property-style invariant checks
Write tests that assert graph invariants across many inputs, not just one happy path.

```python
@pytest.mark.parametrize("query", ADVERSARIAL_QUERIES)
async def test_graph_never_exceeds_max_steps(query, rag_graph):
    result = await rag_graph.ainvoke(make_state(query))
    assert result["step_count"] <= MAX_STEPS
    assert result.get("error") is None or result["error"]["retries"] <= MAX_RETRIES
```

---

## 4. Deployment

### LP-26 — Environment config via `configurable`
Never hardcode model names, API keys, or feature flags inside nodes. Inject via `config["configurable"]`.

```python
def llm_node(state: AgentState, config: RunnableConfig) -> dict:
    model_name = config["configurable"].get("model", "gpt-4o-mini")
    llm = ChatOpenAI(model=model_name)
    return {"answer": llm.invoke(state["messages"]).content}
```

### LP-27 — Trace everything
Instrument every production graph with LangSmith or OpenTelemetry. Capture `thread_id`, `tenant_id`, latency, and token counts per node.

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = f"prod-{settings.ENV}"
```

### LP-28 — Connection pooling + checkpoint pruning
- Use `AsyncConnectionPool` with `max_size` scaled to concurrency.
- Schedule regular pruning of old checkpoints to prevent unbounded storage growth.

```python
# Prune checkpoints older than 30 days
await checkpointer.adelete_thread(
    config={"configurable": {"thread_id": old_thread_id}}
)
```

### LP-29 — Use `updates` stream mode for production dashboards
`updates` mode sends only state deltas per node — lowest bandwidth cost for high-scale deployments.

---

## 5. Security

### LP-30 — State contains sensitive data — sanitize before persistence
Before any state field is checkpointed, sanitize PII from content fields (emails, phone numbers, financial data).

```python
import re

PII_PATTERNS = [
    (r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", "[EMAIL]"),
    (r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", "[CARD]"),
]

def sanitize_pii(text: str) -> str:
    for pattern, replacement in PII_PATTERNS:
        text = re.sub(pattern, replacement, text)
    return text
```

### LP-31 — Validate all external inputs
Apply Pydantic validation at every graph entry point (API handler, webhook, n8n trigger). Never trust raw payload values inside nodes.

### LP-32 — Row-level security per tenant
Scope all DB queries (Postgres checkpointer, Qdrant collections, mem0 store) to `tenant_id + thread_id`. Never issue unscoped queries in multi-tenant deployments.

```python
# Qdrant: always filter by tenant_id
search_results = qdrant_client.query_points(
    collection_name="knowledge_base",
    query=query_vector,
    query_filter=models.Filter(
        must=[models.FieldCondition(
            key="tenant_id",
            match=models.MatchValue(value=tenant_id),
        )]
    ),
    limit=5,
)
```

### LP-33 — Encrypt checkpoints at rest
For regulated environments, enable transparent disk encryption on Postgres and configure SSL for all DB connections.

```python
pool = AsyncConnectionPool(
    conninfo="postgresql://...?sslmode=require",
    max_size=20,
)
```

---

## Post-implementation checklist — production patterns pt2

- [ ] `AsyncPostgresSaver` used in production; `MemorySaver` only in tests (LP-17)
- [ ] `thread_id` uses structured format `tenant:user:session` (LP-18)
- [ ] `checkpoint_ns` scoped per tenant (LP-19)
- [ ] Streaming mode matches use case (`updates` for dashboards, `messages` for chat) (LP-21)
- [ ] Graph tested via `ainvoke` with state assertion, not just node unit tests (LP-23)
- [ ] LLM + tools mocked deterministically in tests (LP-24)
- [ ] Model config injected via `config["configurable"]` (LP-26)
- [ ] LangSmith/OTel tracing enabled in production (LP-27)
- [ ] PII sanitized before state is checkpointed (LP-30)
- [ ] All DB queries scoped to `tenant_id + thread_id` (LP-32)
