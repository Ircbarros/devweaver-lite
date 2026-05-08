# rule_librarian_query

> CARE-Lite retrieval protocol for DevWeaver-Lite.
> Replaces the Qdrant hybrid search + PostgreSQL retrieval from full DevWeaver.
> See also: librarian/CATALOG.md (master index) · adr/adr_lite_001_architecture.md Decision 3

---

## 1. What is CARE-Lite

CARE-Lite is the 4-tier knowledge retrieval chain used during the RESEARCH phase.
The name stands for: Contextualise - Augment - Retrieve - Evaluate.

- **Contextualise**: identify the domain and topics needed from the task description
- **Augment**: look up relevant files in CATALOG.md by domain and topic keywords
- **Retrieve**: load the identified files via Filesystem MCP or GitHub MCP
- **Evaluate**: assess confidence; advance to the next tier if below threshold

CARE-Lite preserves the behavioral semantics of full DevWeaver's CARE chain
(Librarian -> Context7 -> Web -> Model) with Markdown-based retrieval at tier 1.

---

## 2. CATALOG Lookup Procedure

### Step 1 - Identify domain

Map the task to one or more domains from this list:
standards, ai_backend, coding, containers, security, ui_ux, iot, devops, storage, research, teaching, templates, adr

### Step 2 - Scan CATALOG.md

Use the Topics column to find files relevant to the task.
Example: task involves "pytest + LangGraph agents" -> domain=ai_backend, isbn=ai-009.

### Step 3 - Load via Filesystem MCP

Load up to 3 files per phase (R-PD-03):
```
filesystem.read_file(path="references/ai_backend/guide_agent_testing.md")
```

### Step 4 - Evaluate confidence

| Score | Meaning | Action |
|---|---|---|
| >= 0.9 | High confidence - CATALOG covers the topic fully | Stop here, proceed to ARCHITECTURE |
| >= 0.8 | Good confidence - gap exists, supplement needed | Advance to Context7 MCP (tier 2) |
| >= 0.7 | Partial confidence - library docs needed | Advance to web search (tier 3) |
| < 0.7 | Low confidence | Use model knowledge, flag tier 4 explicitly |

---

## 3. Domain Index Usage

When the domain is already known, load the domain index instead of full CATALOG.md:

```
filesystem.read_file(path="librarian/index/ai_backend.md")
```

This saves context tokens vs. loading the 160-line CATALOG. Use domain indexes when:
- You already know the domain (e.g., task is clearly about security)
- You need to see all files in a domain at once
- CATALOG.md is already in context from session start

Use full CATALOG.md when:
- Domain is unknown or spans multiple domains
- Session is just starting (load at PRE-TASK)
- Cross-domain lookup needed (e.g., security + containers)

---

## 4. Confidence Thresholds - 4-Tier Fallback Chain

| Tier | Source | Stop threshold | How to use |
|---|---|---|---|
| 1 | CATALOG.md + Filesystem MCP reads | confidence >= 0.9 | Read file, assess coverage |
| 2 | Context7 MCP (library docs) | confidence >= 0.8 | resolve_library_id then get_library_docs |
| 3 | Web search (official docs, blogs, forums) | confidence >= 0.7 | Targeted search for specific API or pattern |
| 4 | Model knowledge | always flags | State: "Using model knowledge (tier 4) - verify before implementing" |

When using tier 4, explicitly state in the response: "Confidence tier 4 (model knowledge) -
no local or external source found. Verify this information before implementing."

---

## 5. R-PD-03 Enforcement

Rule R-PD-03: A phase MUST NOT load more than 3 L3 resource files per invocation.

Enforcement procedure:
1. Before loading a file, check the count of files already loaded this phase.
2. If count >= 3, do not load additional files. Note which files were deferred.
3. If more files are needed, split the task into two phases or two turns.
4. L2 files (CATALOG.md, ADR, rule_librarian_query.md) do not count toward the 3-file limit.

Priority order when near the 3-file limit:
1. rule_implementation_standards.md (always load for IMPLEMENT/VALIDATE phases)
2. The most topic-specific reference file for the current task
3. rule_standards_versions.md (for IMPLEMENT phase)

---

## 6. Example Lookup Scenario

Task: "Add MQTT ingestion to the IoT module and store readings in InfluxDB"

Step 1 - Identify domains: iot (primary), storage (secondary)

Step 2 - Scan CATALOG.md topics:
- iot-001 (guide_mqtt_production): topics: MQTT, production, v5, patterns - MATCH
- iot-002 (guide_mqtt_influxdb): topics: MQTT, InfluxDB, pipeline, ingestion - MATCH (exact)
- iot-003 (guide_influxdb_schema): topics: InfluxDB, schema, v3, design - MATCH

Step 3 - Load files (3 files, R-PD-03 limit reached):
```
filesystem.read_file("references/iot/guide_mqtt_production.md")
filesystem.read_file("references/iot/guide_mqtt_influxdb.md")
filesystem.read_file("references/iot/guide_influxdb_schema.md")
```

Step 4 - Evaluate: topics fully covered by local files. Confidence >= 0.9. Stop at tier 1.

Result: proceed to ARCHITECTURE phase with 3 loaded reference files.
