# rule_skill_taxonomy

> Coverage map for the 31-subtopic, 6-tier Claude Skill Taxonomy Reference (2026-03-02).
> Adapted for the DevWeaver-Lite skill - network/infrastructure subtopics replaced by
> app-builder equivalents, LangGraph-specific references adapted for Lite behavioral model.
> See also: rule_context_window.md section 7 (Living Documents) · adr_lite_001_architecture.md

---

## 1. Coverage Map

### Tier 1 - Core Identity

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 1 | Identity | Phase 4 | `SKILL.md` preamble - 2-4 sentence identity block |
| 2 | Constraints | Phase 4 + partial | `SKILL.md` Hard Constraints block · `rule_implementation_standards.md` section 8 git gate |
| 3 | Anti-patterns | partial | `rule_diagram_standards_c4.md` section 7 · `rule_diagram_standards_sequence.md` section 7 · `rule_implementation_standards.md` section 6 anti-hallucination |
| 4 | Instruction Hierarchy | Phase 4 | `SKILL.md` - defines override rules (what user CAN vs CANNOT override per tier) |
| 5 | Confidence Signals | done | CARE-Lite protocol in `librarian/rule_librarian_query.md` - confidence thresholds 0.9 / 0.8 / 0.7 |

### Tier 2 - Context and Ground Truth

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 6 | Infrastructure State | done | No SkillState TypedDict; session state tracked via Memory MCP or task files. PRE-TASK plan-check phase |
| 7 | Version Context | done | `rule_standards_versions.md` - pinned versions for all packages |
| 8 | Source of Truth Map | done | `rule_project_modes.md` section 2 - CODE vs STANDARDS decision per mode |
| 9 | Reference Index | done | `adr_lite_001_architecture.md` section 6 · `librarian/CATALOG.md` (full index) |

### Tier 3 - Domain Knowledge

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 10 | Repo Structure | done | `rule_standards_conventions.md` section 2 directory layout |
| 11 | Naming Conventions | done | `rule_standards_conventions.md` - naming rules for Python, files, directories |
| 12 | Tech Stack Patterns | done | `references/research/*` (4 guide files) + domain-specific references |
| 13 | Variable Strategy | done | `rule_implementation_standards.md` section 1 scope + discovery (S-1 to S-6) |
| 14 | Credentials / Auth | done | `references/security/rule_security_toolchain.md` |
| 15 | Idempotency Rules | done | `rule_implementation_standards.md` section 3 Q-1 to Q-10 |
| 16 | Network Topology | N/A | Not applicable - replaced by **bounded module topology** (`rule_standards_conventions.md` section 2) |

### Tier 4 - Workflows and Operations

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 17 | Workflow Modes | done | SKILL.md section 5 (10-phase flow) · `rule_project_modes.md` (4 modes) |
| 18 | Risk Classification | done | `rule_implementation_standards.md` section 8 G-1 to G-6 |
| 19 | Rollback Strategy | done | `rule_implementation_standards.md` section 8 git gate (skip option) |
| 20 | Testing Strategy | done | `rule_implementation_standards.md` section 4 (unit / integration / E2E / security) |
| 21 | Teaching Examples | Phase 4 | `references/teaching/guide_teaching_examples.md` |

### Tier 5 - Quality and Learning

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 22 | AI Failure Modes | done | POST-TASK step 4 (failure log) · `rule_implementation_standards.md` section 6 |
| 23 | Documentation Standards | done | `rule_context_window.md` file size limits R-PD-01/02 · `rule_standards_conventions.md` |
| 24 | Memory Building Protocol | done | CATALOG.md manual updates + scripts/sync_catalog.py + Memory MCP for cross-session preferences |
| 25 | External References Protocol | done | CARE-Lite chain: CATALOG -> Filesystem MCP -> Context7 -> Web -> Model (librarian/rule_librarian_query.md) |

### Tier 6 - Skill Meta / Architecture

| # | Subtopic | Status | Covered by |
|---|---|---|---|
| 26 | Skill Architecture | done | `adr/adr_lite_001_architecture.md` - full Lite architecture decisions |
| 27 | Progressive Disclosure | done | `rule_context_window.md` section 1 L1/L2/L3 tiers |
| 28 | Modularity Rules | done | `rule_context_window.md` section 2 R-PD-02 · <=200 lines per file |
| 29 | Living Documents Protocol | done | `rule_context_window.md` section 7 update-frequency table |
| 30 | Session Management | done | SKILL.md section 7 PRE-TASK (start) + POST-TASK (end) checklists |
| 31 | Context Budget Awareness | done | `rule_context_window.md` section 6 provider budget table |

---

## 2. Notes for Lite

> All 31 subtopics satisfactory for DevWeaver-Lite. Skillgrade: **31 / 31 = 100%**.

| # | Subtopic | Lite adaptation |
|---|---|---|
| 1 | Identity | 2-4 sentence block at top of SKILL.md - names role, ADR grounding, teaching posture, Lite mode |
| 2 | Constraints | Hard Constraints section: 10 rules in NEVER [action]: [reason] format |
| 4 | Instruction Hierarchy | Explicit 4-tier override policy in SKILL.md section 3 |
| 21 | Teaching Examples | 4-6 complete worked examples in references/teaching/guide_teaching_examples.md |

---

## 3. Adaptation Notes

This taxonomy was designed for infrastructure/network skills (OPNsense, Ansible).
Adaptations made for DevWeaver-Lite:

| Original subtopic | DevWeaver-Lite equivalent |
|---|---|
| S5 Confidence Signals | CARE-Lite confidence thresholds in librarian/rule_librarian_query.md |
| S6 Infrastructure State | No SkillState TypedDict; Memory MCP + task files for session tracking |
| S8 Source of Truth Map | rule_project_modes.md section 2 - per-task CODE vs STANDARDS decision |
| S16 Network Topology | Bounded module topology in rule_standards_conventions.md section 2 |
| S14 Credentials / Auth | Zero-trust toolchain in rule_security_toolchain.md + MCP minimum-privilege config |
| S24 Memory Building Protocol | CATALOG.md manual updates + scripts/sync_catalog.py + Memory MCP |
| S25 External References Protocol | CARE-Lite 4-tier chain (CATALOG -> Filesystem MCP -> Context7 -> Web -> Model) |
| S26 Skill Architecture | adr_lite_001_architecture.md (Decision 1-7) |
| S30 Session Management | SKILL.md section 7 start/end checklists (no n8n trigger) |
