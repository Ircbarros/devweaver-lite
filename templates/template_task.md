---
task_id: "task-NNN"                    # sequential zero-padded under plan: task-001, task-002, ...
plan_id: "plan-NNN"                    # parent plan this task belongs to
checkpoint: ""                         # plan checkpoint this task delivers: "CPNa"
title: ""                              # human-readable task title (≤ 10 words)
status: "OPEN"                         # OPEN | IN_PROGRESS | BLOCKED | COMPLETE
trigger_source: "MANUAL"               # MANUAL | N8N_VERSION_MONITOR | N8N_GUIDE_REFRESH
mode: ""                               # SCRATCH | IMPROVE | CONTINUE | REFACTOR
source_of_truth: ""                    # CODE | STANDARDS
created_at: ""                         # ISO-8601
completed_at: ""                       # ISO-8601 — set by post_task_node
session_id: ""                         # mem0 scope key
---

# task-NNN — [Title]

> **Plan**: [{plan_id}](../plans/{plan_id}.md) · **Checkpoint**: {checkpoint}
> **Mode**: {mode} · **Source of truth**: {source_of_truth}
> **Status**: {status}

---

## 1. Scope & Objectives

<!-- What this task must produce. Be specific — ambiguity causes scope creep. -->

**Deliverables:**
-
-

**Out of scope for this task:**
-

**Blocked by / depends on:**
- <!-- task-NNN — reason -->

---

## 2. Open Points

<!-- Unresolved questions or decisions that must be answered before IMPLEMENT.
     Pre-populate from PRE-TASK discovery. Clear before CONFIRM gate. -->

| # | Question / Open Point | Owner | Resolved? |
|---|---|---|---|
| 1 | <!-- question --> | Dev / Agent | ⬜ |

---

## 3. Implementation Log

<!-- Chronological record of significant decisions and actions taken during IMPLEMENT.
     The agent appends to this section — one line per unit, not a full narrative. -->

| Time | Node | Action | Outcome |
|---|---|---|---|
| <!-- HH:MM --> | `implement_node` | <!-- action --> | <!-- result --> |

---

## 4. Issues

<!-- IssueRecord entries raised during this task.
     CRITICAL/HIGH: doc created immediately. MEDIUM/LOW: doc created at post_task_node.
     Full docs live in issues/{application}/ISSUE-NNNN-slug.md (C-19). -->

| Issue ID | Severity | Status | Doc |
|---|---|---|---|
| <!-- ISSUE-NNNN --> | <!-- CRITICAL/HIGH/MEDIUM/LOW --> | <!-- IN_PROGRESS/RESOLVED --> | <!-- [link](../../issues/...) --> |

---

## 5. Test Results

<!-- Summary of test runs at VALIDATE phase. Gate must pass before GIT_GATE. -->

| Suite | Passed? | Failures | Coverage |
|---|---|---|---|
| Unit | ⬜ | <!-- --> | <!-- -->% |
| Integration | ⬜ | <!-- --> | — |
| E2E | ⬜ | <!-- --> | — |
| Security | ⬜ | <!-- bandit / safety / semgrep findings --> | — |

---

## 6. Deviation Report

<!-- Deviations from standards detected during FETCH_RULES or IMPLEMENT.
     Non-zero deviations require source_of_truth = STANDARDS and a CP-A confirmation. -->

| Module | Severity | Violations |
|---|---|---|
| <!-- module --> | <!-- CRITICAL/HIGH/MEDIUM/LOW --> | <!-- list --> |

---

## 7. Git Record

| Field | Value |
|---|---|
| Git state | <!-- UNCOMMITTED / COMMITTED / PR_OPEN / MERGED / SKIPPED --> |
| Commit ref | <!-- SHA or PR URL --> |
| PR ref | <!-- URL or — --> |
| Files changed | <!-- list of paths --> |

---

## 8. Task Summary

<!-- Written by post_task_node. Brief · well-explained · indexable by the Librarian.
     Format: what was done · what changed · key decisions made · watch-outs for next task. -->

<!-- Fill at task close -->
