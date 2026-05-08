# rule_requirements_analysis_pt2

> Phase 1 — Non-Functional, Cross-Cutting, Implied Requirements & Confidence Levels  
> Continued from: `rule_requirements_analysis_pt1.md`

---

## Table of Contents

1. [Cross-Cutting Domains](#1-cross-cutting-domains)
2. [Non-Functional Requirements](#2-non-functional-requirements)
3. [Implied Requirements (Green-Field)](#3-implied-requirements-green-field)
4. [Confidence Levels per Domain](#4-confidence-levels-per-domain)

---

## 1. Cross-Cutting Domains

### 1.1 Security (All Modules)

- **FR-SEC-001** OWASP Top 10 and OWASP LLM Top 10 compliance mandatory for all modules.
- **FR-SEC-002** Prompt injection mitigations: sanitise all external inputs before LLM context injection.
- **FR-SEC-003** Least-privilege tool access: each MCP server exposes only the minimum required tools.
- **FR-SEC-004** All destructive or privileged MCP operations require explicit user approval before execution.
- **FR-SEC-005** No secrets, API keys, or tokens in any tracked file; validated at CI via `gitleaks`.
- **FR-SEC-006** Static analysis on every PR: `bandit` (Python), `opengrep` (multi-lang), `eslint-plugin-security` (JS/TS).
- **FR-SEC-007** Dependency scanning: generate CycloneDX SBOM on every release; upload to `sbom` bucket.
- **FR-SEC-008** All inter-service communication uses TLS 1.2 minimum; TLS 1.3 preferred.
- **FR-SEC-009** LLM outputs validated and sanitised before being written to any database or file system.
- **FR-SEC-010** Adversarial ML defences: input validation against known evasion patterns; monitor for data drift.

### 1.2 DevOps & CI/CD

- **FR-DEV-001** GitHub Actions pipelines for lint → test → build → SBOM → push (Docker images).
- **FR-DEV-002** Conventional commits enforced via `commitlint`; semantic versioning via `release-please`.
- **FR-DEV-003** Docker images built with multi-stage builds; final stage is distroless or minimal (no shell).
- **FR-DEV-004** All images signed with `cosign`; signatures stored in Garage.
- **FR-DEV-005** `ruff` + `black` for Python formatting/linting; `eslint` + `prettier` for TS/Svelte.
- **FR-DEV-006** Test coverage gate: ≥ 80% line coverage on unit tests; CI blocks merge if below threshold.
- **FR-DEV-007** Integration tests spin up ephemeral Docker Compose stack; tear down after every run.
- **FR-DEV-008** E2E tests run via Playwright MCP integration against staging environment.

### 1.3 UI/UX (Svelte / SvelteKit)

- **FR-UX-001** Frontend built with SvelteKit; component-level state with Svelte Runes (`$state`, `$derived`, `$effect`).
- **FR-UX-002** Routing follows SvelteKit file-system conventions; no manual router configuration.
- **FR-UX-003** Accessibility: WCAG 2.2 Level AA compliance; ARIA roles on all interactive elements.
- **FR-UX-004** Design tokens defined in `assets/ui_ux/design_tokens.json`; consumed by all components.
- **FR-UX-005** Responsive design: mobile-first; breakpoints match Material Design 3 spec.
- **FR-UX-006** All async operations show loading state, error state, and success state.
- **FR-UX-007** Code splitting via Vite; no route chunk > 150 KB (gzipped).

### 1.4 Notifications (ntfy)

- **FR-NTF-001** ntfy server self-hosted; topic access restricted with auth tokens.
- **FR-NTF-002** Notification payload ≤ 4 KB; details linked to a Grafana dashboard URL or Loki query.
- **FR-NTF-003** ntfy server URL and auth token loaded from environment variables only.
- **FR-NTF-004** Event categories: `critical`, `warning`, `info`; priority mapped to ntfy priority field.
- **FR-NTF-005** No PII in notification payloads; reference IDs only.

---

## 2. Non-Functional Requirements

### 2.1 Performance

- **NFR-PERF-001** API p99 latency ≤ 200 ms for non-AI endpoints under 100 concurrent users.
- **NFR-PERF-002** AI inference endpoints: p99 ≤ 5 s (streaming preferred; first-token ≤ 800 ms).
- **NFR-PERF-003** InfluxDB write batch size: 1 000–5 000 points per request; write interval ≤ 5 s.
- **NFR-PERF-004** All async I/O; event loop must not be blocked by CPU-bound tasks (offload to `ProcessPoolExecutor`).
- **NFR-PERF-005** Qdrant queries: top-K retrieval ≤ 150 ms at 1 M vectors (use HNSW index, ef=128).

### 2.2 Scalability

- **NFR-SCALE-001** All bounded modules stateless; horizontal scaling via Docker Compose `replicas` or K8s HPA.
- **NFR-SCALE-002** Shared state (sessions, rate limit counters) stored in Redis, not in-process memory.
- **NFR-SCALE-003** IoT ingest pipeline decoupled via MQTT broker; subscriber can scale independently.
- **NFR-SCALE-004** Garage storage scales independently; no storage logic in application tier.

### 2.3 Security (NFR)

- **NFR-SEC-001** Zero hardcoded secrets in any file tracked by git (enforced by gitleaks pre-commit hook).
- **NFR-SEC-002** SBOM generated and signed on every release.
- **NFR-SEC-003** All third-party dependencies pinned to exact versions in `requirements.txt` and `package.json`.
- **NFR-SEC-004** MCP server tool access audited and logged; every privileged call requires user approval.

### 2.4 Reliability

- **NFR-REL-001** Service health checks: liveness (`/healthz`) and readiness (`/readyz`) on every module.
- **NFR-REL-002** Retry with exponential backoff on all external service calls (max 3 retries).
- **NFR-REL-003** Circuit breaker pattern on LLM API calls to prevent cascade failures.
- **NFR-REL-004** MQTT reconnect with exponential backoff; alert via ntfy after 5 failed attempts.

### 2.5 Deployability

- **NFR-DEPLOY-001** Local development: `docker compose up` brings up all services in < 5 min on first run.
- **NFR-DEPLOY-002** All service configuration via environment variables; no config files with secrets.
- **NFR-DEPLOY-003** K8s manifests (Helm charts) deferred to Phase 4; Docker Compose is the initial target.
- **NFR-DEPLOY-004** Cloud deployment (hybrid) uses the same Docker images; only env vars change.

### 2.6 Portability (Skill Agnosticism)

- **NFR-PORT-001** `SKILL.md` YAML frontmatter follows Agent Skills standard v1.0 (`name` + `description` minimum).
- **NFR-PORT-002** Provider-specific files (`CLAUDE.md`, `.cursorrules`, `copilot-instructions.md`) live in `provider-specific/` subfolder.
- **NFR-PORT-003** No provider-specific directives inside `SKILL.md` body.
- **NFR-PORT-004** `SKILL.md` body ≤ 250 lines; all supporting content in `references/`, `scripts/`, `assets/`.

---

## 3. Implied Requirements (Green-Field)

*These are not stated in any upstream spec but are essential for a zero-to-production green-field project.*

- **IMP-001** Docker Compose manifests for ALL services (Mosquitto, InfluxDB, Garage, n8n, LangFuse, Qdrant, PostgreSQL, Redis, Loki, Grafana, Telegraf, ntfy) must ship as `assets/` templates.
- **IMP-002** Librarian PostgreSQL schema must be bootstrapped (migrations run) before any domain-specific work; n8n bootstrap workflow handles this on first run.
- **IMP-003** `provider-specific/` subfolder must contain valid companion files before skill is used.
- **IMP-004** Secret management: all environment variables must be documented in `assets/env_templates/` as `.env.example` files (no actual secrets).
- **IMP-005** Qdrant collections must be created as part of service bootstrap; schema not auto-created.
- **IMP-006** gitleaks pre-commit hook must be installed as part of developer onboarding; documented in `references/devops/`.
- **IMP-007** All Grafana dashboards provisioned from JSON files; no manual UI edits accepted.
- **IMP-008** LangFuse SDK initialised before any LLM call; uninitialised state must throw an error (fail-fast).
- **IMP-009** n8n workflows exported as JSON (`assets/workflows/`) and version-controlled.
- **IMP-010** All services must emit OpenTelemetry spans; services without OTel instrumentation are blocked from production deployment.

---

## 4. Confidence Levels per Domain

*Scale: 0–100%. Entries below 70% trigger an additional research loop in Phase 2.*

| Domain | Confidence | Justification | Flag |
|---|---|---|---|
| FastAPI / Python back-end | 92% | Mature ecosystem, extensive docs, well-known patterns | — |
| LangGraph (AI orchestration) | 85% | Stable v0.2+; breaking changes in edge routing API | — |
| Pydantic v2 | 93% | Stable, widely adopted; v2 migration guide well-documented | — |
| Qdrant (vector DB) | 88% | Stable v1.x; hybrid search BM25 + dense is prod-ready | — |
| mem0 (long-term memory) | 76% | Relatively new; API surface changes between versions | — |
| LangFuse (observability) | 87% | Stable v2.x; MCP server is actively maintained | — |
| Svelte Runes / SvelteKit | 82% | Runes are stable in Svelte 5; SvelteKit 2.x is stable | — |
| paho-mqtt v2 / aiomqtt | 79% | paho-mqtt v2 async API changed significantly; aiomqtt wrapper recommended | — |
| InfluxDB v3 (Line Protocol) | 84% | Line Protocol is stable; v3 query API differs from v2 | — |
| Garage (S3-compatible) | 71% | Stable core; some aioboto3 edge cases with non-AWS endpoints | ⚠️ |
| n8n (workflow automation) | 83% | Self-hosted stable; MCP server is newer, verify tool coverage | — |
| OWASP LLM Top 10 | 95% | Published standard, v1.1 released; well-documented | — |
| MLSecOps / Adversarial ML | 68% | Active research area; ART + TextAttack stable but evolving | ⚠️ |
| OTel / structlog / Loki | 88% | OTel Python SDK stable; structlog + Loki exporter mature | — |
| ntfy (notifications) | 90% | Stable self-hosted; simple REST API; v2.x maintained | — |
| GitHub Actions CI/CD | 94% | Industry standard; highly stable | — |
| MCP Servers (all 9) | 72% | Varying maturity; verify tool coverage per server | ⚠️ |

**Flagged for extra research in Phase 2 (< 80%):**
- `Garage` (71%) → Stream B research
- `MLSecOps / Adversarial ML` (68%) → Stream C research
- `MCP Servers` (72%) → Stream E research
- `paho-mqtt v2 / aiomqtt` (79%) → Stream B research
- `mem0` (76%) → Stream A research

---

*Phase 1 complete. Proceed to Phase 2 research streams.*
