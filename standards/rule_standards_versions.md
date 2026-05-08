# rule_standards_versions

> Master stable versions table for all packages used across the 5 bounded modules.  
> **Rule**: Pin these versions in every `pyproject.toml` and `package.json`. Do not upgrade without running the full test suite and updating this table.

---

## Python Packages

| Package | Pinned Version | Module | Notes |
|---|---|---|---|
| `fastapi` | `0.115.0` | web | Pydantic v2 native |
| `uvicorn[standard]` | `0.30.x` | web | Use `uvicorn[standard]` for HTTP/2 |
| `pydantic` | `2.7.x` | web, ai | `ConfigDict(strict=True)` required |
| `httpx` | `0.27.x` | web, ai | Async HTTP client |
| `python-jose[cryptography]` | `3.3.x` | web | JWT RS256 |
| `passlib[bcrypt]` | `1.7.4` | web | Password hashing |
| `redis[asyncio]` | `5.0.x` | web | Rate limiting, caching |
| `langgraph` | `0.2.55` | ai | Pin exact; pre-1.0 API can change |
| `langchain-core` | `0.3.x` | ai | Transitive via langgraph |
| `langchain-openai` | `0.2.x` | ai | OpenAI embeddings + LLM |
| `qdrant-client` | `1.9.x` | ai | Async client for Qdrant v1.9 |
| `langfuse` | `2.x` | ai | Tracing; use fail-fast init |
| `mem0ai` | `0.1.12` | ai | ⚠️ Beta — pin exact; test upgrades |
| `opentelemetry-sdk` | `1.25.x` | observability | Stable releases only |
| `opentelemetry-instrumentation-fastapi` | `0.46b0` | observability | Match SDK version |
| `opentelemetry-exporter-otlp` | `1.25.x` | observability | gRPC exporter |
| `structlog` | `24.2.x` | observability | JSON rendering |
| `aiomqtt` | `2.x` | iot | Async MQTT v5 |
| `paho-mqtt` | `2.x` | iot | Fallback / CLI usage only |
| `influxdb3-python` | `0.7.x` | iot | InfluxDB v3 Flight SQL |
| `aioboto3` | `13.x` | storage | Async S3 client |
| `cryptography` | `42.x` | storage | Fernet for client-side encryption |
| `psycopg[async]` | `3.2.x` | all | Async PostgreSQL driver |
| `sqlalchemy[asyncio]` | `2.0.x` | all | ORM with async support |
| `alembic` | `1.13.x` | all | DB migrations |
| `ruff` | `0.5.x` | dev | Linter + formatter |
| `mypy` | `1.10.x` | dev | Type checker |
| `pytest` | `8.x` | dev | Test runner |
| `pytest-asyncio` | `0.23.x` | dev | Async test support |
| `bandit` | `1.7.x` | dev/security | SAST scanner |
| `pip-audit` | `2.7.x` | dev/security | Dependency vuln scan |
| `cyclonedx-bom` | `4.x` | dev/security | SBOM generation |
| `gitleaks` | `8.x` | dev/security | Secret scanning |

---

## Node / Front-End Packages

| Package | Pinned Version | Notes |
|---|---|---|
| `svelte` | `5.x` | Runes enabled (`$state`, `$derived`, `$effect`) |
| `@sveltejs/kit` | `2.x` | SvelteKit 2 SSR/CSR routing |
| `vite` | `5.x` | Build tool + dev server |
| `typescript` | `5.5.x` | Strict mode required |
| `dompurify` | `3.x` | Sanitize all LLM output before DOM injection |
| `eslint` | `9.x` | Flat config format |
| `@typescript-eslint/parser` | `7.x` | TS-aware linting |
| `pnpm` | `9.x` | Package manager (use over npm/yarn) |
| `vitest` | `2.x` | Unit testing |
| `@playwright/test` | `1.45.x` | E2E testing |

---

## Infrastructure Components

| Component | Version | Notes |
|---|---|---|
| PostgreSQL | `16.x` | With `pgvector` extension if backup embedding needed |
| Portainer CE | `2.x` | Container management UI; `portainer/portainer-ce:2` pins v2 series — see `adr_004` |
| Qdrant | `1.9.x` | Match `qdrant-client` version |
| n8n | `1.x` | Self-hosted; pin Docker image digest |
| InfluxDB | `3.x Enterprise` | IOx storage engine (columnar) |
| Garage | `0.9.x` | S3-compatible; SSE not supported |
| Grafana | `11.x` | Dashboard + Loki datasource |
| Loki | `3.x` | Log aggregation |
| Grafana Alloy | `1.x` | Log/metric collector (replaces Promtail) |
| Redis | `7.x` | Rate limiting + session cache |

---

## Version Upgrade Policy

1. Never upgrade a pinned dependency without a failing test that requires it.
2. Run `pip-audit` and `trivy` after any upgrade.
3. For `mem0ai`, `langgraph`, `langfuse` — check changelogs for breaking API changes before upgrading.
4. Update this table immediately after confirming a new version is stable.
5. All upgrades require a passing CI pipeline before merging to `main`.
