# rule_requirements_analysis_pt1

> Phase 1 — Functional Requirements per Bounded Module  
> Continued in: `rule_requirements_analysis_pt2.md`

---

## Table of Contents

1. [web/ — FastAPI Back-End API](#1-web--fastapi-back-end-api)
2. [ai/ — AI & RAG Pipeline](#2-ai--ai--rag-pipeline)
3. [iot/ — IoT Data Acquisition](#3-iot--iot-data-acquisition)
4. [storage/ — Object Storage](#4-storage--object-storage)
5. [observability/ — Monitoring & Tracing](#5-observability--monitoring--tracing)

---

## 1. web/ — FastAPI Back-End API

### 1.1 Functional Requirements

- **FR-WEB-001** Expose a versioned REST API (`/api/v1/`) built with FastAPI.
- **FR-WEB-002** Validate all request and response bodies with Pydantic v2 models (`model_config = ConfigDict(strict=True)`).
- **FR-WEB-003** Implement JWT RS256 authentication with access tokens (15 min TTL) and refresh tokens (7 day TTL).
- **FR-WEB-004** Provide OAuth2 password bearer scheme via FastAPI dependency injection.
- **FR-WEB-005** Apply per-endpoint rate limiting (sliding window, configurable via env vars).
- **FR-WEB-006** Auto-generate OpenAPI 3.1 documentation at `/docs` and `/redoc`; disable in production unless explicitly enabled.
- **FR-WEB-007** Implement CORS with an explicit allowlist (no wildcard origins in production).
- **FR-WEB-008** Support structured health-check endpoints: `GET /healthz` (liveness) and `GET /readyz` (readiness).
- **FR-WEB-009** Use FastAPI dependency injection for all cross-cutting concerns (auth, DB sessions, rate limiting, tracing).
- **FR-WEB-010** All I/O-bound operations must be `async`; no blocking calls in the event loop.

### 1.2 Router & Module Layout

- Routers grouped by domain: `routers/auth.py`, `routers/ai.py`, `routers/iot.py`, `routers/storage.py`.
- Schemas co-located per router: `schemas/auth_schema.py`, `schemas/ai_schema.py`, etc.
- Services layer decoupled from routers: `services/auth_service.py`, etc.
- No business logic in router functions; routers only parse, validate, delegate, and return.

### 1.3 Security Requirements (web-specific)

- **FR-WEB-SEC-001** Passwords hashed with bcrypt (≥12 rounds).
- **FR-WEB-SEC-002** JWT private key loaded from environment variable; never stored in source code.
- **FR-WEB-SEC-003** All responses include security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Content-Security-Policy`.
- **FR-WEB-SEC-004** SQL queries use parameterised statements only (via SQLAlchemy ORM or asyncpg).
- **FR-WEB-SEC-005** Secrets injected via environment variables; validated at startup with Pydantic `BaseSettings`.

---

## 2. ai/ — AI & RAG Pipeline

### 2.1 Functional Requirements

- **FR-AI-001** Implement a LangGraph `StateGraph` as the central orchestrator for all AI workflows.
- **FR-AI-002** RAG pipeline: chunk → embed → store in Qdrant → retrieve top-K → re-rank → generate.
- **FR-AI-003** Use mem0 for long-term, cross-session memory storage and retrieval (user + entity memories).
- **FR-AI-004** Integrate LangFuse for full LLM call tracing (input tokens, output tokens, latency, errors).
- **FR-AI-005** Before any LLM action, query the Librarian System to fetch applicable rules and standards.
- **FR-AI-006** Support multiple LLM providers (OpenAI, Anthropic, Google) via a provider-agnostic abstraction layer.
- **FR-AI-007** Implement human-in-the-loop pauses at all destructive or irreversible state transitions.
- **FR-AI-008** All agent tool calls must follow the principle of least privilege; no tool may exceed its declared scope.
- **FR-AI-009** Prompt injection mitigations: input sanitisation before LLM calls, output validation before execution.
- **FR-AI-010** Confidence scoring: if confidence < 90%, trigger a RESEARCH loop before proceeding to IMPLEMENTATION.

### 2.2 RAG Pipeline Specifics

- Embedding model: configurable (default: `text-embedding-3-small`, 1536 dims).
- Chunk strategy: recursive character splitting, 512 tokens, 64-token overlap.
- Re-ranking: cross-encoder or LLM-as-judge before final context assembly.
- Qdrant collection per domain (web, ai, iot, storage, observability, security, devops, ui_ux).
- Payload mandatory fields: `document_id` (FK to Librarian SQL), `doc_type`, `domain`, `last_updated`.

### 2.3 LangGraph State Schema (TypedDict)

```python
class SkillState(TypedDict):
    domain: str                   # active domain being processed
    current_task: str             # task name / ID
    phase: str                    # IDLE | REQUIREMENTS | FETCH_RULES | RESEARCH | ...
    confidence: float             # 0.0–1.0
    librarian_ref: str | None     # ISBN PK of last rule fetched
    memory_ref: str | None        # mem0 memory ID
    messages: list[dict]          # full conversation / tool call history
    errors: list[str]             # accumulated errors this session
    human_approval_required: bool # gate flag
```

---

## 3. iot/ — IoT Data Acquisition

### 3.1 Functional Requirements

- **FR-IOT-001** Subscribe to MQTT topics using paho-mqtt v2 (async API with `aiomqtt` wrapper).
- **FR-IOT-002** Topic hierarchy: `{site}/{device}/{sensor}` — no wildcards in publish paths.
- **FR-IOT-003** Write time-series data to InfluxDB using Line Protocol; batch size ≤ 5 000 points.
- **FR-IOT-004** Enable TLS for all Mosquitto connections (CA cert + client cert/key).
- **FR-IOT-005** Authenticate MQTT clients with username/password AND client certificates.
- **FR-IOT-006** Graceful reconnect with exponential backoff (max 5 retries, then alert via ntfy).
- **FR-IOT-007** All MQTT payloads validated against a JSON schema before InfluxDB write.
- **FR-IOT-008** Dead-letter queue for malformed payloads; stored in PostgreSQL `iot_dlq` table.
- **FR-IOT-009** InfluxDB Organisation, Bucket, and Token loaded from environment variables only.
- **FR-IOT-010** Telegraf agent configured to scrape and forward metrics from all services to InfluxDB.

### 3.2 InfluxDB Line Protocol Rules

- Measurement name: snake_case, matches sensor type (e.g., `temperature_celsius`).
- Tag keys: `site`, `device`, `sensor_id` — low cardinality; no free-text tags.
- Field keys: numeric values only per measurement; string fields only in separate measurements.
- Timestamps: nanosecond precision; UTC only.

---

## 4. storage/ — Object Storage

### 4.1 Functional Requirements

- **FR-STG-001** Use Garage as the S3-compatible object store; all interactions via `aioboto3`.
- **FR-STG-002** Pre-signed URLs for all uploads and downloads (TTL: upload 15 min, download 1 hour).
- **FR-STG-003** All buckets must have versioning enabled from creation.
- **FR-STG-004** Lifecycle policies: expire non-current versions after 30 days; delete incomplete multipart uploads after 7 days.
- **FR-STG-005** Bucket naming convention: `{project}-{env}-{purpose}` (e.g., `aibuilder-prod-models`).
- **FR-STG-006** Garage endpoint, access key, and secret key loaded exclusively from environment variables.
- **FR-STG-007** No public bucket ACLs; all access via pre-signed URLs or server-side proxy.
- **FR-STG-008** Server-side encryption enabled on all buckets (SSE-S3 minimum).
- **FR-STG-009** SBOM artefacts (CycloneDX JSON) uploaded to a dedicated `sbom` bucket after every CI run.
- **FR-STG-010** Garage endpoint must NOT be exposed to the public internet; access via internal network only.

---

## 5. observability/ — Monitoring & Tracing

### 5.1 Functional Requirements

- **FR-OBS-001** Instrument all services with OpenTelemetry SDK (Python); export traces to a configurable OTLP endpoint.
- **FR-OBS-002** Propagate `traceparent` headers across all HTTP calls between modules.
- **FR-OBS-003** Structured logging with `structlog`; every log entry includes `trace_id`, `span_id`, `service`, `env`.
- **FR-OBS-004** Export logs to Loki; configure Loki labels as `{app, env, module}` only (avoid high cardinality).
- **FR-OBS-005** Telegraf scrapes Prometheus-format metrics from all services and forwards to InfluxDB.
- **FR-OBS-006** Grafana dashboards provisioned via JSON files in `assets/observability/`; no manual UI configuration.
- **FR-OBS-007** LangFuse traces every LLM call; dashboard available at `http://langfuse:3000`.
- **FR-OBS-008** ntfy push notifications sent on: service crash, MQTT reconnect failure, n8n pipeline failure, CI/CD failure.
- **FR-OBS-009** Alert thresholds defined in `assets/observability/rule_alert_thresholds.yaml`.
- **FR-OBS-010** No observability data sent to any external/cloud service unless explicitly configured.

---

*Continues in `rule_requirements_analysis_pt2.md` (non-functional, cross-cutting, implied, confidence levels).*
