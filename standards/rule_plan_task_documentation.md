# rule_plan_task_documentation

> **Scope**: Plan and task document creation process, naming, required sections,
> status lifecycle, and SkillState integration (C-21)
> **File limit**: ≤ 200 lines (R-PD-02)
> **Templates**: `templates/template_plan.md` · `templates/template_task.md`

---

## §1 When To Create

**Create a plan document when:**
1. A new multi-phase body of work is scoped with ≥ 2 user checkpoints
2. An n8n automated pipeline spawns a structured work sequence
3. An existing plan phase expands significantly (add a new phase section + bump `updated_at`)

**Create a task document when:**
1. PRE-TASK node initialises work for any checkpoint in a plan
2. A standalone single-checkpoint job is started with no parent plan
3. An n8n autonomous trigger fires (`trigger_source = N8N_*`)

**Do NOT** create separate task docs for: sub-steps within a checkpoint (tracked in the task's
Implementation Log), documentation-only fixes, or git-only operations.

---

## §2 Naming Convention

**Plans**: `plans/plan-NNN.md` or `plans/plan-NNN-{slug}.md`
**Tasks**: `tasks/task-NNN.md` or `tasks/auto/task-NNN.md` (n8n-triggered)

| Part | Rule |
|---|---|
| `NNN` | Zero-padded 3-digit sequential across all plans or all tasks |
| `{slug}` | Optional 3-5 word kebab-case suffix for discoverability |
| `tasks/auto/` | All n8n-triggered tasks land here (`trigger_source = N8N_*`) |

**Examples:**
- `plans/plan-001-phases-1-3.md`
- `tasks/task-001-adr-002-architecture.md`
- `tasks/auto/task-007-fastapi-version-monitor.md`

---

## §3 Status Lifecycles

**Plan statuses:**
```
NOT_STARTED → IN_PROGRESS → BLOCKED → COMPLETE
```
- **BLOCKED**: Depends-on plan not complete or user approval outstanding
- **COMPLETE**: All checkpoints accepted by developer; `updated_at` set by `post_task_node`

**Task statuses:**
```
OPEN → IN_PROGRESS → BLOCKED → COMPLETE
```
- **IN_PROGRESS**: Set when `pre_task_node` creates or loads the task file
- **BLOCKED**: Open points remain unresolved at CONFIRM gate — do not proceed
- **COMPLETE**: `post_task_node` sets this after git record finalised + task summary written

---

## §4 File Size Limits

Both plan and task files fall under **R-PD-02** (≤ 200 lines / ≤ 1 500 tokens).

**Split strategy for plans that exceed 200 lines:**
- Split by phases: `plan-NNN-phases-1-3.md` + `plan-NNN-phase-4.md`
- Each part file carries its own frontmatter and a `> Depends on / see also` line

**Task files that grow large:**
- Move verbose log content to an appendix section at the bottom
- If the task itself exceeds 200 lines, the task scope was too large — split into two tasks

---

## §5 Section Priority

### Plan template fill order (use `templates/template_plan.md`):

**Critical** (always required):
1. Frontmatter — `plan_id` · `title` · `scope` · `status` · `depends_on`
2. Context & Decisions — key choices made before any work starts
3. Phases & Checkpoints — one CP block per checkpoint, step table with status emoji

**Required** (fill as work progresses):
4. Output Artifacts — full artifact tree with CP and status markers
5. Verification Checklist — structured acceptance criteria grouped by theme

### Task template fill order (use `templates/template_task.md`):

**Critical** (fill at `pre_task_node` before RESEARCH):
1. Frontmatter — `task_id` · `plan_id` · `checkpoint` · `mode` · `source_of_truth`
2. Scope & Objectives — deliverables + out-of-scope + blockers
3. Open Points — pre-populate from PRE-TASK discovery; all resolved before CONFIRM

**Required** (fill during IMPLEMENT → VALIDATE):
4. Implementation Log — one line per significant action
5. Issues — IssueRecord entries per C-19
6. Test Results — gate summary per test suite
7. Git Record — set at GIT_GATE

**Required** (fill at `post_task_node`):
8. Task Summary — brief · well-explained · indexable by Librarian

**Optional** (fill when non-zero):
- Deviation Report — only if deviations found in FETCH_RULES or IMPLEMENT

---

## §6 SkillState Integration (C-21)

**`pre_task_node`**:
1. Scan `tasks/` (or `tasks/auto/`) → assign next sequential `task-NNN`
2. Create `tasks/task-NNN.md` from `templates/template_task.md`
3. Populate frontmatter: `task_id` · `plan_id` · `checkpoint` · `trigger_source` · `mode` · `session_id`
4. Pre-fill §2 Open Points from PRE-TASK discovery questions
5. Confirm `plans/plan-NNN.md` exists; create from `templates/template_plan.md` if absent

**`confirm_node`** (after approval):
- Clear all Open Points (mark Resolved = ✅)
- Set plan checkpoint step to 🟡

**`post_task_node`**:
1. Set `status = COMPLETE` · set `completed_at`
2. Write §8 Task Summary (brief · well-explained · watch-outs for next task)
3. Update parent plan: mark checkpoint step ✅ · update `task_ids` list · set `updated_at`
4. Trigger `librarian_sync` n8n pipeline to index the completed task doc
5. If all plan checkpoints complete → set plan `status = COMPLETE`

---

## §7 Quality Checklist

Before setting task `status = COMPLETE`:

- [ ] Frontmatter complete: `task_id` · `plan_id` · `checkpoint` · `mode` · `source_of_truth`
- [ ] All Open Points resolved (no ⬜ remaining in §2)
- [ ] Implementation Log has at least one entry per IMPLEMENT unit
- [ ] All IssueRecord entries have a doc link (or confirmed `doc_path = None` with reason)
- [ ] Test Results table filled — no ⬜ in Passed? column
- [ ] Git Record complete — `git_state` is not `UNCOMMITTED`
- [ ] Task Summary written — readable by someone unfamiliar with the task
- [ ] Parent plan updated — checkpoint marked ✅ · `updated_at` refreshed
