# Implementation Standards & Agent Behavioral Rules

> **Applies to**: All implementation tasks executed by the DevWeaver Skill  
> **Enforced at**: IMPLEMENT phase — `adr/adr_002_agnosticism_strategy.md` §2.4  
> **Related standards**: `rule_standards_conventions.md` · `rule_diagram_standards.md` · `rule_standards_versions.md`

---

## 1. Scope & Discovery Rules

These rules apply **before and throughout** every implementation task, not only at start-up.

| # | Rule | Level |
|---|---|---|
| S-1 | Read **all** referenced documents fully — never skip lines unless 100% confirmed irrelevant | MANDATORY |
| S-2 | If a best practice is not found in the Librarian or SkillState, follow the **web search fallback chain** (§1.1) | MANDATORY |
| S-3 | **Never assume** knowledge you do not have — if uncertain, trigger S-2 before writing any code | MANDATORY |
| S-4 | Always load and consume files matching the patterns in §7 before starting | MANDATORY |
| S-5 | Ask clarifying questions **before** proceeding if any requirement is ambiguous or appears out of scope | MANDATORY |
| S-6 | After each task unit: *"Did I miss anything?"* — loop until no open points remain | MANDATORY |

### 1.1 Web Search Fallback Chain

When a best practice is not available in the Librarian or injected SkillState, search in this order:

1. **Context7 MCP** — `resolve-library-id` then `get-library-docs`
2. **Official tool documentation** (vendor docs site)
3. **Blog posts** (engineering blogs, dev.to, Medium)
4. **Community forums** (GitHub Discussions, Stack Overflow, Reddit)
5. **Books** (O'Reilly, Manning, Packt)
6. **YouTube channels** (official vendor channels, conference talks)

Stop at the first step that yields a confident, verifiable answer. Never synthesise an answer from incomplete evidence.

---

## 2. Dependency Validation Rules

| # | Rule |
|---|---|
| D-1 | Review all required dependencies **before** starting a new task |
| D-2 | Validate by checking `tasks/` and `plans/` folders first |
| D-3 | Research additional dependencies that may be required but not listed in the plan |
| D-4 | Pin all dependency versions to exact values — never allow floating specifiers (`>=`, `^`, `~`) unless explicitly approved in `rule_standards_versions.md` |

---

## 3. Code Quality Rules (C-22 — Clean Code + Zen of Python)

All code produced by this skill MUST satisfy the rules defined in **`standards/rule_clean_code.md`** (C-22).
That file is the single authoritative source; load it at every `implement_node` invocation.

Key principle groups defined there:

| Group | IDs | Summary |
|---|---|---|
| Zen of Python | Z-01 – Z-19 | All 19 aphorisms applied language-agnostically |
| Naming | N-01 – N-10 | Intent-revealing, pronounceable, domain-language names |
| Functions | F-01 – F-08 | Do one thing, DRY, no side effects, max 3 args |
| Comments & Docs | CC-01 – CC-06 | No noise, no dead code, public API documented |
| Classes / SOLID | O-01 – O-08 | SRP · OCP · LSP · ISP · DIP · Law of Demeter |
| Error Handling | E-01 – E-05 | Exceptions with context, no silent failures |
| Format & Structure | R-01 – R-06 | Newspaper rule, formatter enforced |
| Emergence | §8 | Kent Beck's four rules of simple design |

### 3.1 Post-Unit Validation Checklist

Run this checklist after **every unit of code** created or modified:

- [ ] Naming conventions match `rule_standards_conventions.md` and N-01 to N-10
- [ ] All functions pass F-01 to F-08 (single responsibility, no side effects, DRY)
- [ ] Comments: only CC-02 types present; no noise, no dead code
- [ ] Classes/design: O-01 to O-08 (SOLID + Demeter) applied
- [ ] Error handling: E-01 to E-05 — no silent failures
- [ ] Format: R-01 to R-06 applied; configured formatter has been run
- [ ] Emergence §8: design passes all four simple-design rules
- [ ] Idempotency verified — safe to run multiple times without unintended side effects
- [ ] No ghost variables — no unused imports, dead assignments, or unreachable code
- [ ] Zero-trust and OWASP rules applied (`rule_owasp_llm_top10.md`, `rule_security_toolchain.md`)
- [ ] All commands in READMEs and docs verified correct for the current implementation

---

## 4. Issue Management Rules

When an issue is encountered during implementation:

| Step | Action |
|---|---|
| 1 | Create an **ISSUE document** under the `issues/` folder following the issues template |
| 2 | Update the ISSUE document for **every** new symptom, root-cause finding, or resolution step |
| 3 | Use sub-sections within the doc as needed — following issues guidelines |
| 4 | Update the **Librarian database** — any table or file that references or hyperlinks to this issue |
| 5 | Update the **ISSUE README** (`issues/README.md`) with the new or updated entry |

### 4.1 Required Sections in Every ISSUE Document

- **Timeline** — chronological sequence of events
- **Root cause analysis** — confirmed cause, not speculation
- **Resolution steps** — exact steps taken to resolve
- **Prevention / anti-failure pattern** — what was changed to prevent recurrence
- **References** — affected PRs, commits, ADRs, and hyperlinks

---

## 5. Confidence & Recovery Rules

When the agent is uncertain or lost, follow this escalation order:

| Priority | Action |
|---|---|
| 1 | Use **Confidence Signals** from `requirements/` or the active plan file |
| 2 | Check the **task memory file** (`tasks/<task-id>.md`) for prior session context |
| 3 | Search `references/`, `README`s, and `git log` for historical context |
| 4 | Apply the **web search fallback chain** (§1.1) |
| 5 | Ask the **user** targeted questions to reconstruct missing context |

Never proceed past an uncertainty without resolving it at the appropriate priority level.

---

## 6. Anti-Hallucination Rules

If a wrong answer or hallucination is detected at any point:

| Step | Action |
|---|---|
| 1 | Apply an **anti-failure pattern** in the affected code — guard clause, assertion, or explicit validation |
| 2 | **Warn the user immediately** — explain what was wrong and exactly what was changed |
| 3 | Create a detailed **ISSUE document** covering: wrong answer, root cause, fix applied, prevention pattern |
| 4 | Update the Librarian and all affected hyperlinks referencing the corrected artefact |

---

## 7. Reference Folder Glob Patterns (Mandatory Consumption)

The agent MUST scan and consume all files matching these glob patterns before and during implementation:

```
**/references/**
**/reference/**
**/standards/**
**/standard/**
**/best-practices/**
**/best-practice/**
**/guidelines/**
**/guideline/**
**/templates/**
**/template/**
```

Files found under these patterns are treated as **authoritative** and override any LLM-internal assumption.

---

## 8. Git Operation Rules

Git operations (commit, PR creation, merge) are **never performed automatically** by the agent.

| # | Rule |
|---|---|
| G-1 | The agent MUST write files to the working tree only — no `git commit`, `git push`, or PR creation without explicit user confirmation |
| G-2 | Git operations are **only available** after ALL of the following gates pass: security scan, unit tests, integration tests, E2E tests, and zero open implementation issues |
| G-3 | The agent MUST present the user with a clear choice: (1) commit, (2) create PR, (3) merge, (4) skip — and wait for explicit approval for each |
| G-4 | Each git action (commit / PR / merge) requires a **separate, individual approval** — bulk approval is not permitted |
| G-5 | If any test suite fails or a security issue is found, all git operations are **blocked** until the issue is resolved and all gates re-pass |
| G-6 | Merge is only permitted after a PR exists and has passed any configured CI/CD checks |

### 8.1 Git Gate Checklist (must be fully satisfied before presenting git options to the user)

- [ ] Security scan passed (OWASP · zero-trust · dependency audit)
- [ ] Unit test suite passed (0 failures)
- [ ] Integration test suite passed (0 failures)
- [ ] E2E test suite passed (Playwright · 0 failures)
- [ ] No open implementation issues in the task tracking file
- [ ] Self-check loop completed — no missed items

---

## 9. Task & Plan Update Rules

At the end of every implementation unit:

| # | Rule |
|---|---|
| T-1 | Update the **task memory file** (`tasks/<task-id>.md`) with status, open points, and issues found |
| T-2 | Update the **PLAN document** to reflect completed deliverables |
| T-3 | Verify **folder structure documentation** and correct any broken hyperlinks or stale references |
| T-4 | Save all updates in a brief but well-explained summary within the task file |
