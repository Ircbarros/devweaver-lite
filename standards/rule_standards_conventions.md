# rule_standards_conventions

> Naming, file-prefix, folder, and cross-cutting conventions for the fullstack DevWeaver skill.  
> **Rule**: These conventions are enforced by `ruff`, `mypy`, and CI lint jobs. Violations fail the pipeline.

---

## 1. File-Prefix Convention

Every file in the skill's `references/` and `standards/` directories must carry a prefix that signals its category to the AI agent:

| Prefix | Category | Example |
|---|---|---|
| `guide_` | How-to / implementation guide | `guide_fastapi_langgraph.md` |
| `rule_` | Mandatory enforcement rule | `rule_owasp_llm_top10.md` |
| `adr_` | Architecture Decision Record | `adr_002_agnosticism_strategy.md` |
| `plan_` | Planning & task tracking | `plan_001_phases_1_3.md` |

---

## 2. Python Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Package name | `snake_case` | `ai_backend` |
| Module (file) name | `snake_case` | `rag_pipeline.py` |
| Class name | `PascalCase` | `SkillState`, `RagPipeline` |
| Function / method | `snake_case` | `fetch_documents()` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES = 3` |
| Private attribute | `_leading_underscore` | `_client` |
| Dunder | `__double_underscore__` | `__init__` |
| TypedDict key | `snake_case` | `user_id: str` |
| Environment variable | `UPPER_SNAKE_CASE` | `QDRANT_URL` |

---

## 3. Directory Layout Convention

```
project_root/
├── web/                  # FastAPI + Pydantic v2 app
│   ├── api/              # Route handlers grouped by domain
│   ├── models/           # Pydantic models (request/response)
│   ├── services/         # Business logic (no FastAPI dependency)
│   ├── middleware/        # Auth, CORS, security headers
│   └── main.py           # App factory + lifespan
├── ai/                   # LangGraph + RAG + mem0 + LangFuse
│   ├── graphs/           # StateGraph definitions
│   ├── nodes/            # Individual graph node functions
│   ├── rag/              # Retrieval pipeline
│   ├── tools/            # MCP-wrapped tools for LangGraph
│   └── memory/           # mem0 integration layer
├── iot/                  # MQTT ingestion + InfluxDB write
│   ├── mqtt/             # aiomqtt client + handlers
│   ├── influx/           # InfluxDB write pipeline
│   └── schemas/          # Payload JSON schemas
├── storage/              # Garage S3 + file management
│   ├── client.py         # aioboto3 configured client
│   ├── operations.py     # Upload/download/presign functions
│   └── encryption.py     # Fernet client-side encryption
├── observability/        # OTel + structlog + Loki
│   ├── tracing.py        # TracerProvider setup
│   ├── logging.py        # structlog processor chain
│   └── metrics.py        # OTel Meter setup
├── db/                   # Database models + migrations
│   ├── models/           # SQLAlchemy ORM models
│   ├── migrations/       # Alembic migration scripts
│   └── session.py        # Async session factory
├── tests/                # Test suite
│   ├── unit/             # Fast, no I/O
│   ├── integration/      # Uses testcontainers
│   └── e2e/              # Playwright tests
├── frontend/             # SvelteKit 2 app
│   ├── src/routes/       # SvelteKit pages and layouts
│   ├── src/lib/          # Shared components + stores
│   └── src/styles/       # CSS design tokens
├── .github/workflows/    # CI/CD GitHub Actions
├── docker-compose.yml    # Local dev stack
└── pyproject.toml        # Python project config
```

---

## 4. Git Branch and Commit Conventions

**Branch naming**:
- `feat/<short-description>` — new feature
- `fix/<short-description>` — bug fix
- `docs/<short-description>` — documentation only
- `chore/<short-description>` — tooling, deps, CI

**Commit message format** (Conventional Commits):
```
<type>(<scope>): <short description>

[optional body]
[optional footer: BREAKING CHANGE or closes #<issue>]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`  
Scopes: `web`, `ai`, `iot`, `storage`, `observability`, `db`, `ci`, `frontend`

---

## 5. Environment Variable Naming

| Pattern | Example | Purpose |
|---|---|---|
| Service URL | `QDRANT_URL` | Base URL for service |
| API key | `OPENAI_API_KEY` | Third-party credential |
| DB connection | `DATABASE_URL` | Full connection string |
| Feature flag | `FEATURE_RAG_ENABLED` | `"true"` / `"false"` |
| Environment label | `APP_ENV` | `"development"` / `"production"` |
| Internal network | `*_INTERNAL_URL` | Service-to-service URLs |

**Rules**:
- Never hardcode an env var value in Python/JS source code.
- Use `pydantic-settings` `BaseSettings` to validate env var types at startup.
- Separate `.env.development` and `.env.production` files; never commit either.

---

## 6. API Endpoint Naming

- Use lowercase kebab-case paths: `/api/v1/rag-query`
- Group by resource: `/api/v1/documents/<id>`
- Versioning in path: `/api/v1/...`
- Response envelope: `{"data": ..., "meta": {"request_id": "...", "version": "1.0"}}`
- Error envelope: `{"error": {"code": "...", "message": "...", "detail": {...}}}`

---

## 7. LangGraph Convention

- State TypedDict name: `SkillState`
- Node function naming: `node_<state_name>` (e.g., `node_requirements`, `node_architecture`)
- Graph variable: `skill_graph`
- Compiled graph variable: `skill_app`
- Human-approval nodes: always `interrupt_before=["node_confirm"]`
- Confidence threshold to trigger RESEARCH loop: `confidence < 0.9`
