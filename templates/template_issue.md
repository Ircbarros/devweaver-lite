---
issue_id: "ISSUE-NNNN"
slug: "brief-description-kebab-case"      # 3-6 words · kebab-case · lowercase
application: ""                            # ai | web | iot | storage | observability
severity: ""                               # CRITICAL | HIGH | MEDIUM | LOW
status: "INVESTIGATING"                    # INVESTIGATING | IN_PROGRESS | WORKAROUND_APPLIED | RESOLVED | CLOSED
created_at: "YYYY-MM-DDTHH:MM:SSZ"
resolved_at: ""                            # fill when status = RESOLVED
task_id: ""                                # SkillState task_id that created this record
related_issues: []                         # list of ISSUE-NNNN references
---

# ISSUE-NNNN — [Short descriptive title]

> **Application**: `{application}`
> **Severity**: {severity}
> **Status**: {status}
> **Created**: {created_at}
> **Resolved**: {resolved_at}

---

## Quick Summary

<!-- Non-technical. 2–4 sentences. What broke, what was the user/service impact, how it was fixed. -->
<!-- Example: "LangGraph checkpoint schema migration failed during upgrade, preventing all tasks from
loading previous state. Affected all in-progress tasks (0 data loss). Fixed by running idempotent
DDL patch followed by graph build restart." -->

| Field | Value |
|---|---|
| **Impact** | <!-- Services/users affected --> |
| **Duration** | <!-- HH:MM from first symptom to resolution --> |
| **Detection** | <!-- How was it found? Alert / user report / monitoring / test failure --> |
| **Root Cause (brief)** | <!-- One sentence --> |

---

## Symptoms & Detection

<!-- Copy/paste exact error messages — they must be searchable by others who hit the same issue. -->

**Error messages observed:**
```
<!-- paste verbatim — do not paraphrase -->
```

**Affected components:**
-

**How detected:**
- [ ] Monitoring alert
- [ ] User report
- [ ] Developer observation during task
- [ ] Automated test failure (unit / integration / E2E / security)
- [ ] n8n pipeline alert
- [ ] Other: ___

**First observed**: <!-- ISO timestamp or approximate time -->
**Scope**: <!-- number of services/users affected -->

---

## Root Cause Analysis

### Technical Explanation
<!-- Explain to a technical audience. What is THE root cause — not just symptoms. -->

### Plain Language Explanation
<!-- Explain to a non-technical audience in simple terms. -->

### Contributing Factors
<!-- Were there multiple causes? List them — a single root cause is rare. -->

1. <!-- Primary cause -->
2. <!-- Contributing factor (if any) -->

### What Was NOT the Cause
<!-- Hypotheses that were ruled out during investigation. Saves future investigators time. -->

- ~~Hypothesis A~~ — ruled out because...

---

## Investigation Process

<!-- Chronological timeline. Include dead ends — they prevent others from repeating the same wrong turns. -->

| Time | Action | Result |
|---|---|---|
| HH:MM | <!-- command or check --> | <!-- output or finding --> |
| HH:MM | | |

**Key commands used (with full output):**
```bash
# Command 1
$
# Output:

# Command 2
$
# Output:
```

---

## Solution

<!-- Step-by-step. Must be reproducible by someone else starting from scratch. -->

### Steps Applied

1. <!-- Step 1 with explanation -->
   ```bash
   # Command
   $
   # Full output:
   ```

2. <!-- Step 2 -->

### Why This Fix Works
<!-- Explain the mechanism. Why does this action address the root cause — not just the symptoms? -->

---

## Verification Steps

<!-- Specific proof. Not "it works" but "I tested X and observed Y". -->

- [ ] **Test 1**: <!-- what was tested → expected result → actual result -->
- [ ] **Test 2**:
- [ ] **Survived restart**: <!-- yes / no + evidence -->
- [ ] **No regressions**: <!-- what was checked to ensure nothing else broke -->
- [ ] **Monitoring confirms**: <!-- metric or log that shows normal operation -->

---

## Prevention Measures

<!-- Actionable and specific. Not "be more careful" but "added pre-commit hook that validates X". -->

### Immediate Changes Applied
<!-- What was changed right now to prevent recurrence -->

- [ ]

### Long-Term Recommendations
<!-- What should be done — CI checks, monitoring alerts, architecture changes -->

- [ ] Add monitoring alert: <!-- specific metric / threshold -->
- [ ] Add CI validation: <!-- specific check -->
- [ ] Update documentation: <!-- which file needs updating -->
- [ ] Consider architectural change: <!-- reference relevant ADR if applicable -->

---

## Knowledge Transfer

<!-- What does everyone need to know? What transfers to other contexts? -->

**Key insight:**
> <!-- One-sentence learning that applies beyond this specific issue -->

**Reusable commands added to runbook:**
- [ ] `<!-- command -->` — added to `<!-- runbook path -->`

**Documentation gaps found:**
-

**Monitoring gaps found:**
-

---

## Cross-References

<!-- Fill AFTER creating the document — mandatory per rule_issue_documentation.md §6 -->

- [ ] SKILL.md Known Issues updated
- [ ] Relevant ADR updated: `<!-- adr/adr_NNN_name.md -->`
- [ ] Runbook updated: `<!-- references/runbooks/... -->`
- [ ] Librarian sync triggered (`librarian_sync` n8n pipeline)

**Links:**
- ADR:
- Runbook:
- Related task: `tasks/{task_id}`

---

## Related Issues

<!-- Other ISSUE docs that share the same root cause or contribute to a pattern -->

| Issue | Application | Relationship |
|---|---|---|
| [ISSUE-NNNN](../path/ISSUE-NNNN-slug.md) | <!-- app --> | <!-- shared root cause / pattern --> |

---

## Appendix

<!-- Full log excerpts, diffs, or screenshots that were too long for the main body -->

<details>
<summary>Full log output</summary>

```
<!-- paste full logs here -->
```

</details>
