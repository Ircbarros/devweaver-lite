# Project Mode Standards & Procedures

> **Applies to**: RESEARCH phase (`adr/adr_002_agnosticism_strategy.md` §2.4) — triggered at FETCH_RULES → RESEARCH transition  
> **Related standards**: `rule_implementation_standards.md` · `rule_standards_conventions.md` · `rule_diagram_standards.md`

---

## 1. Mode Selection

Before any RESEARCH or ARCHITECTURE work begins, the agent MUST present the following selection to the user and wait for **explicit confirmation**. No default is assumed.

| # | Mode | When to use |
|---|---|---|
| 1 | **FROM SCRATCH** | New project — no existing codebase, architecture, or constraints |
| 2 | **IMPROVE** | Existing project — add features, fix bugs, or enhance specific areas |
| 3 | **CONTINUE** | In-progress project with a delivery plan — resume from a known state |
| 4 | **REFACTOR** | Existing project that must be fully realigned to current standards |

The user may also specify a **combination** (e.g., CONTINUE + partial REFACTOR on touched modules). Combinations must be confirmed explicitly and scoped before proceeding.

---

## 2. Source of Truth Decision (Modes 2 & 3)

For IMPROVE and CONTINUE modes the agent MUST ask:

> **"Should we treat the existing code as the source of truth (adapt) or treat our standards as the source of truth (refactor)?"**

| Choice | Label | Behaviour |
|---|---|---|
| Existing code | `source = CODE` | Preserve existing patterns and architecture. Apply missing best practices **only to new or touched code**. Log all deviations in untouched code as `TECH-DEBT` ISSUE records but do not fix them in this task. |
| Our standards | `source = STANDARDS` | All deviations are **in scope** for this task. Refactor alongside any new work. Use checkpoint per module before proceeding. |

This decision is irreversible within a task — it cannot be changed mid-session without creating a new task.

---

## 3. Mode 1 — FROM SCRATCH

### 3.1 Definition
Greenfield project with no codebase, ADRs, or prior architecture. Standards are the only source of truth.

### 3.2 Prerequisite Inputs (collect from user before RESEARCH)
- Project name and one-sentence description
- Target deployment environment (cloud / on-prem / hybrid)
- Language and framework preferences — or defer to `rule_standards_versions.md`
- Any hard constraints (compliance requirements, legacy integrations, performance SLAs)

### 3.3 Procedure

| Step | Action |
|---|---|
| 1 | Load full standards stack — all files matching §7 glob patterns in `rule_implementation_standards.md` |
| 2 | Load all domain-relevant references from Librarian (all 8 Qdrant collections) |
| 3 | Gather all prerequisite inputs from user (`§3.2`) |
| 4 | Generate architecture plan using standards as the **only** truth |
| **CP-A** | Present architecture to user — confirm scope, bounded modules, data stores, and MCP configuration |
| 5 | Proceed to IMPLEMENT only after CP-A is approved |
| 6 | All public surface areas (APIs, schemas, interfaces) designed to be idiomatic and explicit from the start |

### 3.4 Checkpoints
| Checkpoint | Trigger |
|---|---|
| **CP-A** | Architecture review — mandatory before IMPLEMENT |
| **CP-B** | First module implementation review — after first module is written, before continuing |
| **CP-C** | Integration point review — before wiring modules together |

---

## 4. Mode 2 — IMPROVE

### 4.1 Definition
An existing, working project where the goal is to add features, fix known issues, or enhance specific areas.

### 4.2 Prerequisite Inputs
- Existing codebase context (file tree, key modules, current architecture doc or ADRs if any)
- Clear description of what needs to be improved or added
- Source of truth decision (`§2` above)
- Explicit list of areas that are **out of scope** for this task

### 4.3 Procedure — `source = CODE` (Adapt)

| Step | Action |
|---|---|
| 1 | Map existing structure to bounded module model (ai/, web/, iot/, storage/, observability/) |
| 2 | Identify the **minimal delta** needed to deliver the improvement |
| 3 | Apply standards only to new and touched code — never modify untouched code |
| 4 | For every deviation found in untouched code: create a `TECH-DEBT` ISSUE document and log it |
| **CP-A** | Present delta plan to user — confirm scope is bounded and no untouched areas are being changed |
| 5 | Proceed to IMPLEMENT only after CP-A is approved |

### 4.4 Procedure — `source = STANDARDS` (Refactor + Improve)

| Step | Action |
|---|---|
| 1 | Run gap analysis: existing code vs `rule_standards_conventions.md` and `rule_implementation_standards.md` |
| 2 | Generate deviation report: module-by-module, severity-ranked (CRITICAL → LOW) |
| **CP-A** | Present deviation report — user confirms which deviations are **in scope** for this task |
| 3 | Refactor only confirmed-in-scope deviations, alongside the improvement work |
| **CP-B** | Per-module refactor + improvement review — before moving to next module |
| 4 | Run unit + integration tests per module before moving to the next |
| **CP-C** | Integration review after all touched modules are complete |

### 4.5 Checkpoints
| Checkpoint | Trigger |
|---|---|
| **CP-A** | Delta plan (`source = CODE`) or deviation report (`source = STANDARDS`) — mandatory |
| **CP-B** | Per-module review when `source = STANDARDS` |
| **CP-C** | Integration review after all module changes |

---

## 5. Mode 3 — CONTINUE

### 5.1 Definition
An in-progress project with an existing delivery plan. The agent resumes from a known state.

### 5.2 Architecture Status Check (always performed before user is asked anything)
1. Query Librarian for existing ADRs
2. Validate ADRs against current standards version
3. Report status to user:
   - **Defined and consistent** — proceed normally
   - **Defined but stale** — flag stale decisions as `TECH-DEBT`; confirm with user whether to update ADRs in this session
   - **Missing** — treat as Mode 2 IMPROVE with `source = STANDARDS` to generate architecture first

### 5.3 Prerequisite Inputs
- Existing codebase + current task tracking files (`tasks/`)
- Delivery plan phase and list of open tasks (`plans/`)
- Source of truth decision (`§2` above)
- Session scope: which open task or plan phase to work on

### 5.4 Procedure

| Step | Action |
|---|---|
| 1 | Load task memory file (`tasks/<task-id>.md`) and delivery plan from `plans/` |
| 2 | Confirm architecture status (`§5.2`) |
| 3 | Apply source-of-truth decision to all code touched in this session |
| **CP-A** | Confirm session scope with user — which task/phase, what's in and out of scope |
| 4 | Proceed to IMPLEMENT only after CP-A is approved |
| 5 | New code always follows standards; deviations in untouched code → `TECH-DEBT` ISSUE if `source = CODE` |
| **CP-B** | Architecture consistency review if stale ADRs were found in `§5.2` |

### 5.5 Checkpoints
| Checkpoint | Trigger |
|---|---|
| **CP-A** | Session scope confirmation — mandatory |
| **CP-B** | Architecture consistency review (if stale ADRs detected) |
| **CP-C** | Integration review after session work is complete |

---

## 6. Mode 4 — REFACTOR

### 6.1 Definition
Standards are the **single source of truth**. All existing code is treated as a deviation baseline. Goal is full alignment to current standards within the confirmed scope.

### 6.2 Prerequisite Inputs
- Existing codebase context (full file tree — no shortcuts)
- Scope boundaries: which modules are explicitly in scope
- Any excluded areas (legacy integrations, third-party code, contractually fixed APIs)

### 6.3 Procedure

| Step | Action |
|---|---|
| 1 | Run gap analysis across all in-scope modules: naming (`rule_standards_conventions.md`), code quality (Zen of Python), security (`rule_owasp_llm_top10.md`), idempotency, test coverage, documentation completeness |
| 2 | Generate deviation report: module, file, violation type, severity |
| **CP-A** | Present deviation report — user confirms scope, priority order, and exclusions. **No code is changed before CP-A is approved.** |
| 3 | Execute refactor module by module in confirmed priority order |
| **CP-B** | Per-module refactor review — user confirms module is complete before moving to next |
| 4 | Run full test suite (unit + integration) after each module — do not proceed if failures exist |
| 5 | Update task tracking file, affected READMEs, and hyperlinks per module |
| **CP-C** | Final integration review after all in-scope modules complete |

### 6.4 Deviation Severity Reference

| Severity | Examples |
|---|---|
| **CRITICAL** | Security vulnerability, data loss risk, hardcoded secrets |
| **HIGH** | OWASP violation, missing authentication, broken zero-trust boundary |
| **MEDIUM** | Naming convention violations, missing error handling, no idempotency guarantee |
| **LOW** | Style issues, missing non-obvious comments, redundant imports |

### 6.5 Checkpoints
| Checkpoint | Trigger |
|---|---|
| **CP-A** | Deviation report + scope confirmation — **mandatory, no code change before this** |
| **CP-B** | Per-module refactor completion review |
| **CP-C** | Final integration review |

---

## 7. Universal Rules Across All Modes

| # | Rule |
|---|---|
| U-1 | Mode MUST be confirmed by the user before RESEARCH begins — no assumption, no default |
| U-2 | All checkpoints are **mandatory** — the agent never proceeds past one without explicit user confirmation |
| U-3 | All deviations found in any mode MUST be documented as `TECH-DEBT` or `ISSUE` records |
| U-4 | The source-of-truth decision (existing code vs standards) MUST be explicit — never inferred |
| U-5 | New code **always** follows current standards regardless of mode or source-of-truth decision |
| U-6 | Test suites must pass before any git operation — see §8 of `rule_implementation_standards.md` |
| U-7 | Scope changes mid-task require a new user confirmation — the agent never silently expands scope |
| U-8 | `TECH-DEBT` issue records created in `source = CODE` modes MUST be linked from the delivery plan |
