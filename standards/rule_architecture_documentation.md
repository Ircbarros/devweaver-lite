# rule_architecture_documentation

> **Scope**: Architecture document creation process, naming, required sections,
> diagram conventions, version management, and SkillState integration
> **File limit**: ≤ 200 lines (R-PD-02)
> **Template**: `templates/template_architecture.md`

---

## §1 When To Create

Create an architecture document when:

1. A **new bounded module** is scaffolded (mode = SCRATCH)
2. A **major feature** changes a module's external interfaces, data model, or MCP scope
3. A **refactor** significantly changes a module's internal structure (mode = REFACTOR)
4. An **approved ADR** introduces a new component or changes an existing one
5. A **new external system or MCP server** is integrated

**Do NOT** create for: bug fixes, version-only dependency bumps (unless API surface
changes), documentation-only updates, or configuration changes.

---

## §2 Naming Convention

**Path**: `references/architecture/{component}.md`

| Part | Rule |
|---|---|
| `{component}` | Bounded module or sub-system: `ai` · `web` · `iot` · `storage` · `observability` · `rag_pipeline` · `skill_activation` |

**Version field** in frontmatter:
- **Minor** (`1.0 → 1.1`): additive change — new fields, section expansions
- **Major** (`1.x → 2.0`): breaking interface change — requires ADR amendment

**Examples:**
- `references/architecture/ai.md`
- `references/architecture/rag_pipeline.md`
- `references/architecture/web_api.md`

---

## §3 Status Lifecycle

```
DRAFT → REVIEW → APPROVED → DEPRECATED
```

- **DRAFT**: Agent-generated at `architecture_node` — not yet reviewed
- **REVIEW**: Presented at `confirm_node` gate
- **APPROVED**: Developer approved at `confirm_node`; `updated_at` recorded
- **DEPRECATED**: Superseded by new version or component removed — keep file, add deprecation note

---

## §4 Required Sections & Priority

Use `templates/template_architecture.md` as the base. Fill in priority order:

**Critical** (always required):
1. Frontmatter — `doc_id` · `version` · `status` · `component` · `adr_refs` · `task_id`
2. Executive Summary — 1-2 sentences · scope · key decisions linked to ADRs
3. Context & Problem Statement — why the component exists + constraints
4. Architecture Overview — C4 L1 context diagram + C4 L2 module diagram (§5 rules)
5. Security Architecture — auth model · secrets policy · OWASP mapping for AI components

**Required** (fill when applicable):
6. Component Details — tech stack with versions pinned to `rule_standards_versions.md`
7. Data Architecture — data flow sequence diagram + storage table
8. Monitoring & Observability — structlog events emitted · LangFuse observations

**Optional** (include if SLAs or HA exist):
9. Scalability & Performance — latency targets · context budget impact
10. Reliability & Availability — PostgresSaver resume strategy · RTO/RPO
11. Risks & Mitigations — link to `IssueRecord` doc_path where applicable

**File size**: Architecture docs follow **R-PD-04** (`rule_context_window.md §2`) — `references/architecture/` files **≤ 250 lines / ≤ 1 500 tokens**.
If a document exceeds 250 lines, split sequentially using the `_pt{N}` suffix:

| Part | Filename | Contains |
|---|---|---|
| Part 1 | `{component}_pt1.md` | Frontmatter · Executive Summary · Context · Architecture Overview (C4 diagrams) |
| Part 2 | `{component}_pt2.md` | Component Details · Data Architecture · Data Flow diagram |
| Part 3 | `{component}_pt3.md` | Security · Scalability · Reliability · Observability |
| Part 4 | `{component}_pt4.md` | Risks · Future Enhancements · Related Docs · Appendices · Changelog |

**Split rules:**
- Every part file carries its **own frontmatter** (`doc_id`, `version`, `status`, `component`) — use the same `doc_id` with a part suffix (e.g., `arch-ai-pt1`)
- Every part file has a `# See also` back-reference listing all sibling part files
- Stop splitting as soon as each part fits ≤ 250 lines — do not create empty parts

---

## §5 Diagram Rules

All diagrams **must** render in `mermaid.live` (v11.x) per `standards/rule_diagram_standards.md`:

| View | Mermaid type | Required conventions |
|---|---|---|
| C4 L1 System Context | `flowchart LR` | `classDef external stroke-dasharray:5 5,fill:#f8f8f8` on all external nodes |
| C4 L2 Modules & Data Stores | `flowchart LR` with `subgraph` | `subgraph` names end in `_b` · databases use `[("...")]` cylinder |
| Request / Data Flow | `sequenceDiagram` | `autonumber` · participant aliases ≤ 12 chars |
| State machine | `stateDiagram-v2` | `direction LR` · `state X : description` for annotated states |

**Mandatory from `rule_diagram_standards_c4.md`:**
- C4 node labels: line 1 = name, line 2 = `[Technology]`, line 3 = key responsibility
- `direction LR` inside every `subgraph` with > 3 nodes
- No nested `subgraph` beyond 2 levels
- Never use ASCII art — all diagrams must be Mermaid

---

## §6 Cross-Reference Strategy

After creating or updating an architecture document:

- [ ] Link from every `adr_refs` ADR: add `references/architecture/{component}.md` to its `## Consequences` section
- [ ] Update SKILL.md Reference Index — add `references/architecture/{component}.md` to L3 tier if absent
- [ ] Trigger `adr_index_pipeline` n8n pipeline to index in the Librarian
- [ ] Update `plans/` deliverable status if this doc was a plan artifact

**ADR cross-reference addition format:**
```markdown
### Architecture Docs
- [`references/architecture/{component}.md`](../references/architecture/{component}.md)
  v{version} — {status}
```

---

## §7 SkillState Integration (C-20)

**`architecture_node`** — creates doc, status = DRAFT:
1. Scan `references/architecture/` — increment version if file exists (`1.0 → 1.1`), else `"1.0"`
2. Create/update `references/architecture/{component}.md` from `templates/template_architecture.md`
3. Append `ArchitectureDocRecord(status=DRAFT, doc_path=...)` to `SkillState.architecture_docs`

**`confirm_node`** — after developer approval:
- Set `ArchitectureDocRecord.status = APPROVED` · set `updated_at`

**`post_task_node`** — architecture sync:
- Diff committed files vs `ArchitectureDocRecord.doc_path`
- If content changed: bump version (`minor` by default) · set `updated_at` · trigger `adr_index_pipeline`
- Validate all Mermaid diagram blocks still render (mmdc CI check)

---

## §8 Quality Checklist

Before setting `status = APPROVED`:

- [ ] Frontmatter complete: `doc_id` · `version` · `status` · `component` · `adr_refs` · `task_id`
- [ ] Executive Summary references specific ADRs (not generic statements)
- [ ] C4 L1 diagram renders in `mermaid.live` v11.x · external nodes have dashed borders
- [ ] C4 L2 diagram renders · `subgraph` names end in `_b` · databases use cylinder shape
- [ ] All `[Technology]` labels match versions in `rule_standards_versions.md`
- [ ] Security section covers: auth model · secret format · OWASP mapping (AI components)
- [ ] All `adr_refs` in frontmatter match referenced ADR links in body
- [ ] File ≤ 500 lines (or split per §4 rule)
- [ ] Cross-references added per §6 checklist
- [ ] No ASCII art — all diagrams use Mermaid syntax
