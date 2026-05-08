# rule_context_window

> Token-efficient context management rules for all agent providers (Copilot, Claude, Cursor,
> Windsurf, Gemini, Anthropic API, Kimi, OpenRouter, Mistral, OpenClaw, OpenAI).
> Enforced at authoring time - not a runtime concern.

---

## 1. Progressive Disclosure Architecture

The skill loads content in three tiers to minimise context window consumption:

| Tier | What loads | Size target | When |
|---|---|---|---|
| **L1 - Metadata** | YAML frontmatter `name` + `description` only | <= 100 tokens | At agent startup - acts as a menu |
| **L2 - Core Instructions** | Full `SKILL.md` body | <= 500 lines / <= 5 000 tokens | Only when the agent activates the skill |
| **L3 - Resources** | `references/`, `standards/`, `templates/` files | <= 200 lines / <= 1 500 tokens each | Only when an explicit step instructs the agent to read the file |

> **L3 files are never pre-loaded.** Each phase reads only the files it needs via Filesystem MCP
> or GitHub MCP. Max 3 L3 files per phase invocation (R-PD-03).

---

## 2. Hard File-Size Limits

| File type | Hard limit | Enforcement |
|---|---|---|
| `SKILL.md` | <= 500 lines | CI lint rule `R-PD-01` |
| `references/architecture/*.md` | <= 250 lines / <= 1 500 tokens | CI lint rule `R-PD-04` |
| All other `.md` files | <= 200 lines / <= 1 500 tokens | CI lint rule `R-PD-02` |
| Python source files | - (no limit) | Standard line-length linting |

**Rule R-PD-01**: If `SKILL.md` exceeds 500 lines, the CI pipeline fails with:
`[R-PD-01] SKILL.md exceeds 500 lines - extract content to L3 resource files.`

**Rule R-PD-02**: If any `.md` file under `standards/` or `references/` exceeds 200 lines,
the CI pipeline fails with:
`[R-PD-02] {filename} exceeds 200 lines - split into focused sub-files.`

**Rule R-PD-04**: If any `.md` file under `references/architecture/` exceeds 250 lines,
the CI pipeline fails with:
`[R-PD-04] {filename} exceeds 250 lines - split into {component}_pt1.md, {component}_pt2.md, etc.`

> **ADRs are exempt from R-PD-02.** ADRs (`adr/`) are L2 architectural references loaded in full
> only when the agent activates the skill. They may exceed 200 lines.
> SKILL.md is governed separately by R-PD-01 (<= 500 lines).

**Split strategy**: When a file exceeds the limit, split by major `##` section. Each split file
MUST have its own YAML frontmatter and a back-reference to the parent file.

---

## 3. Behavioral Context Loading

L3 files are loaded on demand by the agent via Filesystem MCP or GitHub MCP reads.
No Python code or framework is required - the agent follows these loading rules:

- **Phase starts**: identify required L3 files from the phase description in SKILL.md
- **Load via tool**: call `filesystem.read_file(path)` for local files, or `github.get_file_contents` for remote
- **Count check**: never load more than 3 L3 files per phase (R-PD-03)
- **De-duplication**: track which files are already in context; do not reload them
- **Deferred files**: if more than 3 files are needed, defer lower-priority files to the next phase

Typical loading pattern per phase:

- pre_task: rule_project_modes.md + rule_standards_versions.md (2 files)
- fetch_rules: rule_implementation_standards.md + rule_project_modes.md (2 files)
- research: domain-specific file from CATALOG.md + rule_project_modes.md (2-3 files)
- architecture: rule_diagram_standards_c4.md + rule_standards_conventions.md (2 files)
- implement: rule_implementation_standards.md + rule_standards_conventions.md + rule_standards_versions.md (3 files)
- validate: rule_implementation_standards.md + references/security/rule_security_toolchain.md (2 files)
- post_task: rule_diagram_standards_sequence.md + rule_documentation_standards.md (2 files)

---

## 4. Node Resource Table

| Phase | L3 Resources loaded | Justification |
|---|---|---|
| `pre_task` | `rule_project_modes.md` + `rule_standards_versions.md` | Mode selection + version pins |
| `requirements` | - (uses Context7 / web, not local files) | External docs only |
| `fetch_rules` | `rule_implementation_standards.md` + `rule_project_modes.md` | Full rule set for retrieval planning |
| `research` | `rule_project_modes.md` + `rule_implementation_standards.md` | Mode procedure + quality rules |
| `architecture` | `rule_diagram_standards_c4.md` + `rule_standards_conventions.md` | Diagram + naming rules |
| `confirm` | - (human gate, no LLM call) | Interrupt only |
| `implement` | `rule_implementation_standards.md` + `rule_standards_conventions.md` + `rule_standards_versions.md` | Re-reads standards at every unit |
| `validate` | `rule_implementation_standards.md` + `rule_security_toolchain.md` | Test + security gates |
| `git_gate` | - (human gate, no LLM call) | Interrupt only |
| `post_task` | `rule_diagram_standards_sequence.md` + `rule_documentation_standards.md` | Diagram re-validation + project docs |

---

## 5. CI Enforcement

```yaml
# .github/workflows/lint-docs.yml  (excerpt)
- name: Check doc line limits
  run: |
    python scripts/check_doc_limits.py \
      --skill-max 500 \
      --doc-max 200 \
      --arch-max 250 \
      --paths standards/ references/
```

```python
# scripts/check_doc_limits.py  (excerpt)
from pathlib import Path
import sys

SKILL_MAX, DOC_MAX, ARCH_MAX = 500, 200, 250
errors = []
for p in Path("standards").rglob("*.md"):
    lines = p.read_text().count("\n")
    limit = SKILL_MAX if p.name == "SKILL.md" else DOC_MAX
    if lines > limit:
        errors.append(f"[R-PD-02] {p} has {lines} lines (limit {limit})")
for p in Path("references").rglob("*.md"):
    lines = p.read_text().count("\n")
    is_arch = "architecture" in str(p.parent)
    limit, rule = (ARCH_MAX, "R-PD-04") if is_arch else (DOC_MAX, "R-PD-02")
    if lines > limit:
        errors.append(f"[{rule}] {p} has {lines} lines (limit {limit})")
if errors:
    for e in errors: print(e)
    sys.exit(1)
```

---

## 6. Token Budget Awareness

Track token use manually or via provider-native context tracking. When
`context_budget_remaining < 10 000`, defer all non-critical L3 file loads to the next turn.

Budget defaults (configurable per provider in `provider-specific/{provider}/mcp_config.json`):

| Provider | Model context | Reserved output | Available budget |
|---|---|---|---|
| GitHub Copilot | 128 000 | 8 000 | 120 000 |
| Claude | 200 000 | 8 000 | 192 000 |
| Cursor | 128 000 | 8 000 | 120 000 |
| Windsurf | 128 000 | 8 000 | 120 000 |
| Gemini | 1 000 000 | 8 000 | 992 000 |
| Anthropic API direct | 200 000 | 8 000 | 192 000 |
| Kimi (Moonshot AI) | 128 000 | 8 000 | 120 000 |
| OpenRouter (agnostic) | 128 000 | 8 000 | 120 000 |
| Mistral AI (La Plateforme) | 128 000 | 8 000 | 120 000 |
| OpenClaw (multi-backend) | 128 000-200 000 | 8 000 | 120 000 default |
| OpenAI (GPT-4o) | 128 000 | 8 000 | 120 000 |
| OpenAI (o3 / o4-mini) | 200 000 | 8 000 | 192 000 |

---

## 7. Living Documents Protocol

Every file is classified by update frequency. Stale files produce wrong recommendations.

| File | Frequency | Update trigger |
|---|---|---|
| `rule_standards_versions.md` | On release | New package version released |
| `references/research/guide_*.md` | Monthly or on major release | Manual update or scripts/sync_catalog.py |
| `adr/*.md` | On architecture change | POST-TASK arch sync when ADR scope is affected |
| `tasks/*.md` | Per session | Updated by every post_task phase |
| `rule_implementation_standards.md` | Rarely - quarterly | Major coding convention change |
| `rule_project_modes.md` | Rarely - quarterly | New mode added or procedure changed |
| `rule_context_window.md` | Rarely - quarterly | Model context limits change or new provider added |
| `rule_skill_taxonomy.md` | Rarely - on taxonomy change | New subtopic added or gap resolved |
| `rule_diagram_standards_*.md` | Rarely - on Mermaid major version | Mermaid v12+ breaking change |
| `librarian/CATALOG.md` | When reference files change | Manual update or scripts/sync_catalog.py |

**Rules:**
- A new `references/` file is created when a pattern appears in 3+ separate tasks, a HIGH+
  IssueRecord resolves with a generalisable fix, or a new library/framework version is adopted.
- An existing file is **updated** (never replaced wholesale) for new version pins, resolved
  deviations, or triggered guide refreshes.
- A file not loaded in 3 months is a candidate for archival - flag in post_task phase.
