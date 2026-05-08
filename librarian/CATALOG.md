# librarian/CATALOG.md

Master document index for DevWeaver-Lite. All 58 knowledge files indexed by ISBN, domain, tier, and topic.
L2 files load at skill activation. L3 files load on demand (max 3 per phase, R-PD-03).

---

## Standards (14 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| std-001 | standards | rule | L3 | project modes, SCRATCH, IMPROVE, CONTINUE, REFACTOR | standards/rule_project_modes.md | 4 project mode procedures with source-of-truth decision |
| std-002 | standards | rule | L3 | quality gates, git gate, OWASP, G-1 to G-6, Q-1 to Q-10 | standards/rule_implementation_standards.md | Implementation quality gates and git commit rules |
| std-003 | standards | rule | L3 | version pins, packages, Python, Node, Docker | standards/rule_standards_versions.md | Pinned versions for all packages and infrastructure |
| std-004 | standards | rule | L3 | naming, directory layout, bounded modules, conventions | standards/rule_standards_conventions.md | Naming rules, directory layout, bounded module structure |
| std-005 | standards | rule | L3 | docs format, changelog, R-DS-10, no emojis | standards/rule_documentation_standards.md | Documentation format, changelog, R-DS-10 rules |
| std-006 | standards | rule | L3 | token budget, L1/L2/L3 tiers, R-PD-01 to R-PD-04 | standards/rule_context_window.md | Token budget tracking, file size limits, provider budgets |
| std-007 | standards | rule | L3 | C4 diagrams, L1 context, L2a/b/c, Mermaid | standards/rule_diagram_standards_c4.md | C4 diagram conventions and Mermaid syntax |
| std-008 | standards | rule | L3 | sequence diagrams, Mermaid, conventions | standards/rule_diagram_standards_sequence.md | Sequence diagram conventions |
| std-009 | standards | rule | L3 | architecture docs, references/architecture/, overview | standards/rule_architecture_documentation.md | Architecture documentation standards |
| std-010 | standards | rule | L3 | clean code, principles, readability | standards/rule_clean_code.md | Clean code principles |
| std-011 | standards | rule | L3 | issues, TECH-DEBT, severity, CRITICAL/HIGH/MEDIUM/LOW | standards/rule_issue_documentation.md | Issue and TECH-DEBT record format |
| std-012 | standards | rule | L3 | plans, tasks, phases, CP notation | standards/rule_plan_task_documentation.md | Plan and task file conventions |
| std-013 | standards | rule | L3 | taxonomy, 31 subtopics, 6 tiers, coverage map | standards/rule_skill_taxonomy.md | 31-subtopic coverage map for DevWeaver-Lite |
| std-014 | standards | rule | L3 | diagram index, standards overview | standards/rule_diagram_standards.md | Diagram standards index and overview |

---

## AI Backend References (9 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| ai-001 | ai_backend | guide | L3 | FastAPI, LangGraph, integration, routers | references/ai_backend/guide_fastapi_langgraph.md | FastAPI + LangGraph integration patterns |
| ai-002 | ai_backend | guide | L3 | LangGraph, production, deployment, part 1 | references/ai_backend/guide_langgraph_production_pt1.md | LangGraph production (part 1) |
| ai-003 | ai_backend | guide | L3 | LangGraph, production, deployment, part 2 | references/ai_backend/guide_langgraph_production_pt2.md | LangGraph production (part 2) |
| ai-004 | ai_backend | guide | L3 | LangFuse, tracing, observability, confidence | references/ai_backend/guide_langfuse_observability.md | LangFuse tracing and confidence metadata |
| ai-005 | ai_backend | guide | L3 | mem0, memory, session, cross-session | references/ai_backend/guide_mem0_memory.md | mem0 session memory facade patterns |
| ai-006 | ai_backend | guide | L3 | Qdrant, RAG, vector search, part 1 | references/ai_backend/guide_qdrant_rag_pt1.md | Qdrant RAG implementation (part 1) |
| ai-007 | ai_backend | guide | L3 | Qdrant, RAG, vector search, part 2 | references/ai_backend/guide_qdrant_rag_pt2.md | Qdrant RAG implementation (part 2) |
| ai-008 | ai_backend | guide | L3 | RAG, LangFuse, combined, tracing | references/ai_backend/guide_rag_qdrant_langfuse.md | RAG + LangFuse combined tracing |
| ai-009 | ai_backend | guide | L3 | agent testing, LangGraph, pytest, patterns | references/ai_backend/guide_agent_testing.md | LangGraph agent testing patterns |

---

## Coding References (3 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| cod-001 | coding | guide | L3 | Python, style, typing, Pydantic v2 | references/coding/guide_python_standards.md | Python style, typing, Pydantic v2 standards |
| cod-002 | coding | guide | L3 | Python, advanced, patterns, part 1 | references/coding/guide_python_advanced_pt1.md | Advanced Python patterns (part 1) |
| cod-003 | coding | guide | L3 | Python, advanced, patterns, part 2 | references/coding/guide_python_advanced_pt2.md | Advanced Python patterns (part 2) |

---

## Containers References (4 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| cnt-001 | containers | guide | L3 | Dockerfile, best practices, layers, multi-stage | references/containers/guide_dockerfile_best_practices.md | Dockerfile best practices and multi-stage builds |
| cnt-002 | containers | guide | L3 | container security, scanning, CVE | references/containers/guide_container_security.md | Container security scanning patterns |
| cnt-003 | containers | guide | L3 | Docker, performance, tuning | references/containers/guide_docker_performance.md | Docker performance tuning |
| cnt-004 | containers | guide | L3 | Portainer, operations, management | references/containers/guide_portainer_operations.md | Portainer operations guide |

---

## Security References (2 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| sec-001 | security | rule | L3 | OWASP, LLM Top 10, prompt injection, compliance | references/security/rule_owasp_llm_top10.md | OWASP LLM Top 10 compliance rules |
| sec-002 | security | rule | L3 | bandit, pip-audit, gitleaks, semgrep, toolchain | references/security/rule_security_toolchain.md | Security toolchain setup and usage |

---

## UI/UX References (4 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| ui-001 | ui_ux | guide | L3 | SvelteKit, patterns, components | references/ui_ux/guide_svelte_patterns.md | SvelteKit 2 patterns and components |
| ui-002 | ui_ux | guide | L3 | SvelteKit, advanced, stores, routing | references/ui_ux/guide_svelte_advanced.md | SvelteKit advanced patterns |
| ui-003 | ui_ux | guide | L3 | UI design, principles, part 1 | references/ui_ux/guide_ui_design_pt1.md | UI design principles (part 1) |
| ui-004 | ui_ux | guide | L3 | UI design, principles, part 2 | references/ui_ux/guide_ui_design_pt2.md | UI design principles (part 2) |

---

## IoT References (5 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| iot-001 | iot | guide | L3 | MQTT, production, v5, patterns | references/iot/guide_mqtt_production.md | MQTT v5 production patterns |
| iot-002 | iot | guide | L3 | MQTT, InfluxDB, pipeline, ingestion | references/iot/guide_mqtt_influxdb.md | MQTT to InfluxDB pipeline |
| iot-003 | iot | guide | L3 | InfluxDB, schema, v3, design | references/iot/guide_influxdb_schema.md | InfluxDB v3 schema design |
| iot-004 | iot | guide | L3 | Grafana, dashboards, panels, queries | references/iot/guide_grafana_dashboards.md | Grafana dashboard patterns |
| iot-005 | iot | guide | L3 | Loki, logging, aggregation | references/iot/guide_loki_logging.md | Loki log aggregation |

---

## DevOps References (1 file)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| dev-001 | devops | guide | L3 | CI/CD, pipelines, GitHub Actions, patterns | references/devops/guide_cicd_patterns.md | CI/CD pipeline patterns |

---

## Storage References (1 file)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| sto-001 | storage | guide | L3 | Garage, S3, object storage, compatible | references/storage/guide_garage_s3.md | Garage S3-compatible object storage |

---

## Research References (4 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| res-001 | research | guide | L3 | MCP servers, patterns, part 1 | references/research/guide_mcp_servers_pt1.md | MCP server patterns (part 1) |
| res-002 | research | guide | L3 | MCP servers, patterns, part 2 | references/research/guide_mcp_servers_pt2.md | MCP server patterns (part 2) |
| res-003 | research | rule | L3 | requirements analysis, part 1 | references/research/rule_requirements_analysis_pt1.md | Requirements analysis (part 1) |
| res-004 | research | rule | L3 | requirements analysis, part 2 | references/research/rule_requirements_analysis_pt2.md | Requirements analysis (part 2) |

---

## Teaching References (2 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| tch-001 | teaching | guide | L3 | examples, worked, real tasks, patterns | references/teaching/guide_teaching_examples.md | Worked examples from real tasks |
| tch-002 | teaching | guide | L3 | QA, edge cases, simulation, behavior | references/teaching/guide_qa_simulation.md | QA edge-case simulation results |

---

## Templates (6 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| tpl-001 | templates | template | L3 | architecture, document, template | templates/template_architecture.md | Architecture document template |
| tpl-002 | templates | template | L3 | README, documentation, template | templates/template_doc_readme.md | README template |
| tpl-003 | templates | template | L3 | runbook, operations, template | templates/template_doc_runbook.md | Runbook template |
| tpl-004 | templates | template | L3 | issue, TECH-DEBT, record, template | templates/template_issue.md | Issue and TECH-DEBT record template |
| tpl-005 | templates | template | L3 | plan, delivery, phases, template | templates/template_plan.md | Delivery plan template |
| tpl-006 | templates | template | L3 | task, session, template | templates/template_task.md | Task file template |

---

## ADR (1 file)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| adr-001 | adr | adr | L2 | architecture, zero-infra, MCPs, Markdown Librarian | adr/adr_lite_001_architecture.md | Lite architecture decisions: zero-infra, 5 MCPs, CARE-Lite |

---

## Librarian (2 files)

| ISBN | Domain | Type | Tier | Topics | Path | Summary |
|---|---|---|---|---|---|---|
| lib-001 | librarian | rule | L2 | CATALOG, index, retrieval, CARE-Lite | librarian/CATALOG.md | This file - master document index |
| lib-002 | librarian | rule | L2 | CARE-Lite, retrieval protocol, confidence thresholds | librarian/rule_librarian_query.md | CARE-Lite retrieval protocol and query procedure |
