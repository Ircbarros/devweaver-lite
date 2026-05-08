---
plan_id: "plan-NNN"                    # sequential zero-padded: plan-001, plan-002, ...
title: ""                              # human-readable plan title
scope: ""                              # 1-line scope summary
status: "NOT_STARTED"                  # NOT_STARTED | IN_PROGRESS | BLOCKED | COMPLETE
created_at: ""                         # ISO-8601
updated_at: ""                         # updated by post_task_node on every task close
depends_on: []                         # ["plan-001", ...] — blocking plans
task_ids: []                           # ["task-NNN", ...] — all tasks under this plan
---

# plan-NNN — [Title]

> **Scope**: {scope}
> **Depends on**: {depends_on or "—"}
> **Status**: {status}

---

## 1. Context & Decisions

<!-- Why this plan exists · key design choices made before any task starts -->

| Decision | Value |
|---|---|
| <!-- decision A --> | <!-- value --> |
| <!-- decision B --> | <!-- value --> |

---

## 2. Phases & Checkpoints

<!-- One section per phase. Each phase contains one or more checkpoints (CP).
     A new task is created for each CP — see templates/template_task.md.
     The next CP is blocked until the previous CP user approval is received. -->

### Phase N — [Phase Name]

**Goal**: <!-- 1 sentence -->
**Status**: ⬜ Not started · 🟡 In Progress · ✅ Complete · 🔴 Blocked

#### Checkpoint NA — [Deliverable Name]

**Deliverable**: <!-- primary output file(s) -->

| Step | Deliverable | Status |
|---|---|---|
| N.1 <!-- step name --> | <!-- output file or artifact --> | ⬜ |
| N.2 <!-- step name --> | <!-- output file or artifact --> | ⬜ |
| ⚠️ USER APPROVAL | — | ⬜ Waiting |

<!-- Add rules or constraints specific to this checkpoint here. -->

---

## 3. Output Artifacts

<!-- Artifact tree for all deliverables across the entire plan.
     Mark each artifact with its checkpoint (CPNa) and current status emoji. -->

```
{root_folder}/
├── <!-- artifact path -->    ⬜ CPNa
└── <!-- artifact path -->    ⬜ CPNb
```

---

## 4. Verification Checklist

<!-- One checklist item per meaningful acceptance criterion.
     Group by theme: Structure · Content · Quality · CI/Docs. -->

**Structure**
- [ ] <!-- acceptance criterion 1 -->

**Content**
- [ ] <!-- acceptance criterion 2 -->

**Quality / CI**
- [ ] <!-- acceptance criterion 3 -->
