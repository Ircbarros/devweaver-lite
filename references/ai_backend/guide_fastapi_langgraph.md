# guide_fastapi_langgraph

> Stream A — FastAPI & LangGraph patterns for the full-stack AI skill.  
> Related: `guide_rag_qdrant_langfuse.md` · `guide_langgraph_production_pt1.md` · `guide_langgraph_production_pt2.md` · `guide_agent_testing.md`

---

## Table of Contents

1. [FastAPI Project Structure](#1-fastapi-project-structure)
2. [Pydantic v2 Patterns](#2-pydantic-v2-patterns)
3. [JWT RS256 Auth via Dependency Injection](#3-jwt-rs256-auth-via-dependency-injection)
4. [Async I/O Rules](#4-async-io-rules)
5. [Rate Limiting](#5-rate-limiting)
6. [Security Headers Middleware](#6-security-headers-middleware)
7. [LangGraph StateGraph Patterns](#7-langgraph-stategraph-patterns)
8. [Human-in-the-Loop](#8-human-in-the-loop)
9. [LangGraph + FastAPI Integration](#9-langgraph--fastapi-integration)
10. [Stable Versions](#10-stable-versions)

---

## 1. FastAPI Project Structure

```
web/
├── routers/      # Domain-grouped routes (auth, ai, iot, storage)
├── schemas/      # Pydantic v2 models per router
├── services/     # Business logic; routers delegate only
├── dependencies/ # Reusable FastAPI Depends (auth, db, rate limit, otel)
└── main.py       # App factory; register routers + middleware only
```

- No business logic in router functions.
- Absolute imports only; prefix private helpers with `_`.
- `response_model` on every endpoint (drives OpenAPI docs and output validation).

---

## 2. Pydantic v2 Patterns

```python
from pydantic import BaseModel, ConfigDict, Field

class UserCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    email: str = Field(..., min_length=1, max_length=254)
    password: str = Field(..., min_length=12)
```

- `ConfigDict(strict=True)` — rejects coercions (int → str silently allowed in lax mode).
- Use `model_validate()` (v2) not `parse_obj()` (v1 removed).
- Use `model_config = ConfigDict(from_attributes=True)` on ORM response models.

---

## 3. JWT RS256 Auth via Dependency Injection

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
import os

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")
_PUBLIC_KEY = open(os.getenv("JWT_PUBLIC_KEY_PATH")).read()

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, _PUBLIC_KEY, algorithms=["RS256"])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return user_id
```

- Private key loaded from env var path; never stored in source code.
- Access token TTL: 15 min. Refresh token TTL: 7 days.
- Refresh tokens stored hashed in PostgreSQL; revoked on logout.

---

## 4. Async I/O Rules

| Operation | Pattern |
|---|---|
| Database queries | `async` always (asyncpg, SQLAlchemy async) |
| HTTP calls to external services | `async` with httpx.AsyncClient |
| File reads in hot path | `async` with aiofiles |
| CPU-bound work (embeddings, crypto) | `asyncio.to_thread()` or `ProcessPoolExecutor` |
| Pydantic validation / JSON parsing | Sync is fine (fast, not I/O) |

**Anti-pattern**: Calling `requests.get()` inside an `async def` handler — blocks the event loop.

---

## 5. Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address, storage_uri=os.getenv("REDIS_URL"))
app.state.limiter = limiter

@router.post("/query")
@limiter.limit("10/minute")
async def llm_query(request: Request, body: QueryRequest):
    ...
```

- Store limits in Redis (`storage_uri`) for distributed/replicated deployments.
- Key by authenticated user ID for logged-in routes, by IP for anonymous.

---

## 6. Security Headers Middleware

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        return response
```

---

## 7. LangGraph StateGraph Patterns

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class SkillState(TypedDict):
    domain: str
    current_task: str
    phase: str           # IDLE|REQUIREMENTS|FETCH_RULES|RESEARCH|ARCHITECTURE|CONFIRM|IMPLEMENT|VALIDATE
    confidence: float    # 0.0–1.0; < 0.9 loops back to RESEARCH
    librarian_ref: str | None
    memory_ref: str | None
    messages: Annotated[list, add_messages]
    errors: list[str]
    human_approval_required: bool

def route_after_research(state: SkillState) -> str:
    return "ARCHITECTURE" if state["confidence"] >= 0.9 else "RESEARCH"

builder = StateGraph(SkillState)
builder.add_node("REQUIREMENTS", requirements_node)
builder.add_node("FETCH_RULES", fetch_rules_node)
builder.add_node("RESEARCH", research_node)
builder.add_conditional_edges("RESEARCH", route_after_research,
    {"ARCHITECTURE": "ARCHITECTURE", "RESEARCH": "RESEARCH"})
```

- Use `Annotated[list, add_messages]` for message accumulation (auto-merges).
- Keep state small; large blobs go to PostgreSQL checkpoint, reference by ID in state.

---

## 8. Human-in-the-Loop

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver(connection_string=os.getenv("POSTGRES_URL"))
graph = builder.compile(
    interrupt_before=["CONFIRM", "IMPLEMENT"],  # Pause before destructive steps
    checkpointer=checkpointer
)

# Resume after user approval
config = {"configurable": {"thread_id": task_id}}
graph.invoke({"human_approval_required": False}, config)
```

- Use `PostgresSaver` in production; `MemorySaver` only in unit tests.
- `interrupt_before` > `interrupt_after` for cleaner recovery on crash.

---

## 9. LangGraph + FastAPI Integration

```python
@router.post("/task/start")
async def start_task(body: TaskRequest) -> TaskResponse:
    config = {"configurable": {"thread_id": str(uuid4())}}
    result = await graph.ainvoke(body.dict(), config)
    return TaskResponse(thread_id=config["configurable"]["thread_id"], result=result)

@router.post("/task/{thread_id}/resume")
async def resume_task(thread_id: str, approval: ApprovalRequest):
    config = {"configurable": {"thread_id": thread_id}}
    state = graph.get_state(config)
    await graph.ainvoke({"human_approval_required": approval.approved}, config)
```

---

## 10. Stable Versions

| Package | Version | Stability |
|---|---|---|
| `fastapi` | `0.115.0` | ✅ Stable |
| `pydantic` | `2.7.x` | ✅ Stable |
| `python-jose[cryptography]` | `3.3.0` | ✅ Stable |
| `langgraph` | `0.2.55` | ✅ Stable |
| `slowapi` | `0.1.9` | ✅ Stable |
| `httpx` | `0.27.x` | ✅ Stable |

**Breaking changes to watch**: LangGraph checkpoint format changed in 0.2.x (auto-migrates from 0.1.x).
