# guide_teaching_examples

> Worked examples extracted from real completed tasks.
> Format per S21 (rule_skill_taxonomy.md §1): Context - Wrong approach - Correct approach - Why.
> **See also**: `standards/rule_skill_taxonomy.md` §1 item 21 · `references/teaching/guide_qa_simulation.md`

---

## Example 1 - Mode Selection: CONTINUE vs REFACTOR

**Context**: Developer requests "update the MQTT pipeline to use MQTT v5." An existing pipeline
using MQTT v3 is in production. A `tasks/` file exists with a partial plan.

**Wrong approach**: The agent starts a REFACTOR sweep of the entire `iot/` module because the
pipeline "deviates from current standards."

**Correct approach**:
1. Recognize the existing task file - this is CONTINUE mode.
2. Ask the source-of-truth question: existing CODE or our STANDARDS?
3. If developer answers CODE: update only the MQTT client call sites for v5 compatibility.
   Leave everything else untouched; log any found deviations as TECH-DEBT ISSUE records.
4. If developer answers STANDARDS: scope the refactor explicitly to the `iot/` module only,
   confirm at CP-A before changing a single line.

**Why**: REFACTOR without explicit scope confirmation violates rule_project_modes.md U-2. The
agent never self-assigns a broader refactor than what the user confirms.

---

## Example 2 - CARE Retrieval: Confidence Tier Fallback

**Context**: Research node queries Librarian (Qdrant) for "SvelteKit SSR streaming patterns."
The top result has a confidence score of 0.83.

**Wrong approach**: The agent uses the 0.83 result directly without flagging the confidence tier,
implying the answer is authoritative.

**Correct approach**:
1. Librarian tier requires confidence >= 0.9. Score 0.83 does not clear the threshold.
2. Fall to Context7 MCP: fetch official SvelteKit docs for streaming.
3. If Context7 returns >= 0.8 confidence, stop and use that result.
4. Always log the confidence tier used in `SkillState.research_confidence_tier`.
5. In the CONFIRM gate output include: "Source: Context7 (confidence 0.85)."

**Why**: Silently using below-threshold results introduces hallucination risk. Explicit tier
logging ensures the developer can validate the source and the POST-TASK sweep records it.

---

## Example 3 - Context Budget: Mid-Task Warning

**Context**: A long IMPLEMENT phase has loaded multiple L3 files across nodes.
`SkillState.context_budget_remaining` drops to 8 500 tokens during `implement_node`.

**Wrong approach**: Agent continues loading new L3 files as if the budget were unlimited,
causing truncated outputs and incomplete implementations later in the node.

**Correct approach**:
1. When `context_budget_remaining < 10 000`, emit a WARNING log entry in SkillState.
2. Defer all non-critical L3 file loads to the next session checkpoint.
3. At the CONFIRM gate (or if already past it, at the next natural pause), inform the developer:
   "Context budget is low (8 500 tokens remaining). Recommend completing current module,
   running POST-TASK, and resuming in a new session for the next module."
4. Never silently truncate a file load - always fail visibly and propgate the warning.

**Why**: Silent context overflow produces partial implementations without any error signal.
Proactive warning preserves correctness at the cost of splitting the task, which is recoverable.

---

## Example 4 - Architecture Phase: Component Doc vs Overview

**Context**: A new `storage/` module is being added to an existing project (IMPROVE mode,
source = STANDARDS). The ARCHITECTURE phase produces a new ADR.

**Wrong approach**: The agent only creates `references/architecture/storage.md` and considers
the architecture phase complete.

**Correct approach** (per ADR-003 C-20 and C-23):
1. Create `references/architecture/storage.md` - the component-level architecture doc.
2. Update `docs/architecture/overview.md` - add the new module to the system-level overview
   with a one-line description and a link to `references/architecture/storage.md`.
3. Both writes happen in the ARCHITECTURE phase, before the CONFIRM gate fires.
4. The CONFIRM gate output includes both file paths so the developer can review both changes.

**Why**: `docs/architecture/overview.md` is the top-level navigation artifact for the whole
system. Omitting it means the new module is invisible to any future CONTINUE or REFACTOR run
that starts by reading the overview.

---

## Example 5 - Git Gate: Partial Approval

**Context**: VALIDATE passes all tests. The git gate fires with three proposed actions:
(A) `git add web/main.py`, (B) `git add references/architecture/web.md`,
(C) `git commit -m "feat(web): add FastAPI trigger endpoint"`.

**Wrong approach**: Agent presents all three as a single batch and asks "approve all?"

**Correct approach** (per ADR-003 C-05 and rule_implementation_standards.md §8):
1. Present each action individually with file path and diff summary.
2. Developer can approve A, approve B, but reject C (e.g., wants to adjust the commit message).
3. For each rejected action: record the rejection reason in SkillState, hold that action,
   and do not proceed past it.
4. Only execute approved actions, in order, one at a time.
5. POST-TASK still runs after any approved actions, even if some were skipped.

**Why**: Batch git approval bypasses the per-action consent requirement. A developer who
rejects the commit message still wants the file staged - these are independent decisions.

---

## Example 6 - POST-TASK Sweep: Issue Documentation

**Context**: During the IMPLEMENT phase, the agent identifies a missing input validation on a
FastAPI endpoint (potential injection risk - HIGH severity per rule_implementation_standards.md).
The endpoint is in scope and gets fixed. A second endpoint with the same pattern is out of scope
for this task.

**Wrong approach**: Agent fixes both endpoints (violating IMPROVE source=CODE scope rules)
or fixes neither and silently moves on.

**Correct approach**:
1. Fix the in-scope endpoint as part of the current task.
2. In POST-TASK issue sweep (step 3 of 6): create `issues/ISS-{id}.md` for the out-of-scope
   endpoint using `templates/template_issue.md`.
3. Severity: HIGH (OWASP A3 injection risk).
4. Status: OPEN - to be addressed in a future IMPROVE task.
5. Link the issue record from the CHANGELOG entry for this task.

**Why**: Silently fixing out-of-scope code violates the source-of-truth agreement. Creating an
ISSUE record ensures the deviation is tracked, prioritised, and addressed intentionally.
