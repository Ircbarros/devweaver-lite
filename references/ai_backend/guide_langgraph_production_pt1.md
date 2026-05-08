# guide_langgraph_production_pt1

> AI backend reference — LangGraph production patterns: state design, graph flow, error handling, HITL.
> Sources: swarnendu.de/langgraph-best-practices · docs.langchain.com thinking-in-langgraph
> Related: `guide_langgraph_production_pt2.md`, `guide_fastapi_langgraph.md`

---

## Table of Contents

1. [Thinking in LangGraph — 5-Step Method](#1-thinking-in-langgraph--5-step-method)
2. [State Design](#2-state-design)
3. [Graph Flow](#3-graph-flow)
4. [Error Handling](#4-error-handling)
5. [Human-in-the-Loop (HITL)](#5-human-in-the-loop-hitl)

---

## 1. Thinking in LangGraph — 5-Step Method

| Step | Action | Rule |
|---|---|---|
| 1. Map workflow | Identify discrete steps as future nodes | Each logical unit = one node |
| 2. Classify nodes | `LLM` · `data` · `action` · `user-input` | Label before wiring |
| 3. Design state | Raw data only; format on-demand inside nodes | Never store formatted prompts in state |
| 4. Build nodes | One testable function per node | Pure functions; no side-effects on shared state |
| 5. Wire graph | Simple edges first; conditionals only at real branches | See §3 |

> **LP-01** State = raw data only. Format prompts inside nodes, not in state fields.

```python
# Good: raw data in state, format inside node
class AgentState(TypedDict):
    query: str
    context: list[str]      # raw retrieved docs
    answer: str             # LLM output, not prompt

# Bad: formatted prompt stored in state
class AgentState(TypedDict):
    full_prompt: str        # never do this
```

---

## 2. State Design

### LP-02 — Minimal TypedDict
Use `TypedDict` (or Pydantic `BaseModel`) with only fields that persist across steps or are expensive to recompute.

```python
from typing import Annotated, TypedDict
import operator

class AgentState(TypedDict):
    messages:   Annotated[list, operator.add]   # append-only
    context:    list[str]
    error:      dict | None                     # typed error slot
    step_count: int                             # cycle guard
```

### LP-03 — Reducers sparingly
Apply `add_messages` / `operator.add` reducers only on genuinely append-only fields. All other fields use last-write semantics.

### LP-04 — Node purity
Nodes are pure functions: receive state → return state delta. No direct mutation of state dict.

```python
def retrieve_node(state: AgentState) -> dict:
    docs = retriever.invoke(state["messages"][-1].content)
    return {"context": docs}    # return delta only
```

### LP-05 — Boundary validation
Validate external inputs (user queries, tool results, webhook payloads) at the graph boundary using Pydantic, not inside nodes.

```python
from pydantic import BaseModel, field_validator

class InputPayload(BaseModel):
    query: str
    tenant_id: str

    @field_validator("query")
    @classmethod
    def no_empty_query(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("query must not be empty")
        return v.strip()
```

---

## 3. Graph Flow

### LP-06 — Simple edges first
Wire every stable, deterministic transition as a simple edge. Never use `add_conditional_edges` for paths that do not actually branch.

### LP-07 — Conditional edges at real branch points only
```python
def route(state: AgentState) -> str:
    if state.get("error") and state["error"]["fatal"]:
        return "error_handler"
    if state["step_count"] >= MAX_STEPS:
        return "error_handler"
    return "continue"

graph.add_conditional_edges("tool_node", route, {
    "continue": "llm_node",
    "error_handler": "error_handler_node",
})
```

### LP-08 — Cycle guard
Every graph with cycles MUST carry a `step_count` field incremented each iteration and a `should_continue` guard that routes to termination at `MAX_STEPS`.

```python
MAX_STEPS = 10

def increment_step(state: AgentState) -> dict:
    return {"step_count": state["step_count"] + 1}
```

### LP-09 — Supervisor + specialist pattern
Use the `Send` API to fan out work to specialist sub-graphs. Supervisor returns `Command` objects, not plain strings.

```python
from langgraph.types import Command
from typing import Literal

def supervisor_node(state: AgentState) -> Command[Literal["researcher", "writer"]]:
    next_agent = planner.invoke(state["messages"])
    return Command(goto=next_agent, update={"messages": state["messages"]})
```

---

## 4. Error Handling

### LP-10 — Error taxonomy & routing

| Error type | Strategy | LangGraph pattern |
|---|---|---|
| Transient (network, timeout) | Retry with backoff | `RetryPolicy(max_attempts=3, initial_interval=1.0)` on node |
| LLM-recoverable (bad output) | Loop back with error in state | `error` field set → conditional edge re-routes to LLM |
| User-fixable (missing input) | Pause for input | `interrupt()` — see §5 |
| Unexpected | Bubble up | Re-raise; let graph-level handler catch |
| Saga / exhausted retries | Compensation | Route to `error_handler` node; return degraded response |

### LP-11 — RetryPolicy on transient nodes
```python
from langgraph.pregel import RetryPolicy

graph.add_node(
    "tool_node",
    tool_fn,
    retry=RetryPolicy(max_attempts=3, initial_interval=1.0),
)
```

### LP-12 — Typed error dict in state
```python
def tool_node(state: AgentState) -> dict:
    try:
        result = external_api_call(state["query"])
        return {"context": [result], "error": None}
    except ExternalAPIError as exc:
        return {"error": {"msg": str(exc), "fatal": False, "retries": 0}}
```

### LP-13 — `MAX_RETRIES` guard before degraded response
```python
MAX_RETRIES = 2

def error_handler_node(state: AgentState) -> dict:
    err = state["error"]
    if err["retries"] < MAX_RETRIES and not err["fatal"]:
        return {"error": {**err, "retries": err["retries"] + 1}}
    # exhausted — return degraded response
    return {"answer": "Sorry, I was unable to complete this request.", "error": None}
```

---

## 5. Human-in-the-Loop (HITL)

### LP-14 — `interrupt()` for high-risk actions
Place `interrupt()` inside a node that performs a high-stakes action (e.g., write to DB, send email). Returns control to the caller with a payload.

```python
from langgraph.types import interrupt

def approval_node(state: AgentState) -> dict:
    decision = interrupt({
        "action": state["pending_action"],
        "request": "Human approval required",
    })
    # decision payload is returned when graph is resumed
    return {"human_decision": decision, "approved": decision.get("approved", False)}
```

### LP-15 — Deterministic resume path
After `interrupt()`, the graph resumes at the same node. The resume path MUST be deterministic — branch on the decision payload stored in state, not on external side-effects.

```python
def after_approval_node(state: AgentState) -> dict:
    if state["approved"]:
        return execute_action(state)
    return {"answer": "Action cancelled by human reviewer."}
```

### LP-16 — Capture full decision payload
Always store the complete human decision dict in state for auditability, not just a boolean.

---

## Post-implementation checklist — production patterns pt1

- [ ] All state fields typed; `Annotated` reducers on append-only fields only (LP-02/03)
- [ ] Nodes are pure functions returning state deltas (LP-04)
- [ ] External inputs validated with Pydantic at graph boundary (LP-05)
- [ ] Conditional edges only at genuine branch points (LP-07)
- [ ] Every cycle has `step_count` + `MAX_STEPS` guard (LP-08)
- [ ] `RetryPolicy` on all nodes with transient failure risk (LP-11)
- [ ] Typed error dict in state; degraded response after `MAX_RETRIES` (LP-12/13)
- [ ] `interrupt()` placed before all high-risk actions (LP-14)
- [ ] Resume path is deterministic; decision payload stored in state (LP-15/16)
