# rule_issue_documentation

> **Scope**: Issue document creation process, naming convention, severity classification,
> cross-reference strategy, and lifecycle management for the DevWeaver skill
> **File limit**: ≤ 200 lines (R-PD-02)
> **Template**: `templates/template_issue.md`

---

## §1 When To Create

Create an issue document when:

1. A bug or problem took **> 15 minutes** to diagnose
2. A **service outage or degradation** affected dependent services or pipelines
3. A **misconfiguration** caused unexpected behavior in any bounded module
4. A **workaround** was implemented for a known limitation
5. Troubleshooting revealed a **gap in documentation or monitoring**
6. The **same problem recurs** — second occurrence triggers mandatory documentation

**Do NOT** create for: simple typos, expected behavior, planned maintenance, or third-party
issues with no actionable workaround.

---

## §2 Naming Convention

**Path**: `issues/{application}/ISSUE-{NNNN}-{slug}.md`

| Part | Rule |
|---|---|
| `{application}` | Short bounded module name: `ai` · `web` · `iot` · `storage` · `observability` |
| `{NNNN}` | Zero-padded 4-digit sequential per `{application}` folder |
| `{slug}` | 3–6 words · kebab-case · lowercase |

```bash
# Determine next number before creating
ls issues/{application}/ | sort | tail -1
# Empty folder → 0001 · Last is ISSUE-0003 → use 0004
```

**Examples:**
- `issues/ai/ISSUE-0001-langgraph-checkpoint-schema-migration.md`
- `issues/web/ISSUE-0002-jwt-token-expiry-race-condition.md`
- `issues/storage/ISSUE-0001-garage-fernet-decrypt-failure.md`

---

## §3 Severity & Response

| Severity | Condition | Response | Document? |
|---|---|---|---|
| CRITICAL | System down · data loss risk · security breach | Immediate | Full doc + incident report |
| HIGH | Major service degraded · user-facing impact | Within 1 hour | Full document |
| MEDIUM | Minor degraded · workaround available · no user impact | Same day | Full doc if resolution > 1 hour |
| LOW | Cosmetic · future risk · optimization opportunity | As available | Brief doc or backlog only |

---

## §4 Status Lifecycle

```
INVESTIGATING → IN_PROGRESS → WORKAROUND_APPLIED → RESOLVED → CLOSED
                                        ↓
                              (if permanent fix found)
```

- **CLOSED**: No recurrence for 30+ days — move to `issues/{application}/archive/`
- **Keep** SKILL.md Known Issues entry even after archival — historical context is valuable

---

## §5 Document Structure

Use `templates/template_issue.md` as the base. Fill sections in priority order:

**Critical** (always required):
1. Quick Summary — non-technical · impact + resolution time
2. Symptoms & Detection — exact error messages (copy/paste, searchable)
3. Root Cause Analysis — the WHY explained to both technical and non-technical readers
4. Solution — numbered steps with exact commands **and full output**
5. Prevention Measures — actionable changes, not "be more careful"

**Required** (fill when applicable):
6. Investigation Process — chronological timeline including dead ends
7. Verification Steps — specific proof (what was tested, expected vs actual)
8. Knowledge Transfer — key learnings that transfer to other contexts

**Optional**:
9. Related Issues — link to docs sharing root cause
10. Appendix — full log excerpts, screenshots, config diffs

**Writing rule**: Write for 4 audiences simultaneously — (1) future self in 6 months,
(2) someone with identical symptoms, (3) a learner, (4) an auditor.

---

## §6 Cross-Reference Checklist

After creating an issue document, update **ALL** of:

- [ ] `SKILL.md` — add entry to **Known Issues** section (use format below)
- [ ] Relevant ADR — add note to **Consequences → Known Issues** subsection if architectural
- [ ] Relevant runbook — add Troubleshooting section with commands + link to ISSUE doc
- [ ] Application state file — if a permanent workaround is in place

**SKILL.md Known Issues format:**
```markdown
## Known Issues

N. **[Short description]** — {application}
   - Issue: [ISSUE-NNNN](issues/{application}/ISSUE-NNNN-{slug}.md) · Severity: {severity}
   - Workaround: {one-line workaround or "None — permanently fixed"}
```

---

## §7 Integration with SkillState & C-19

**CRITICAL / HIGH — document immediately at `implement_node`:**
- Assign `ISSUE-NNNN` (scan `issues/{application}/` for next sequential number)
- Create doc from `templates/template_issue.md`; set `IssueRecord.doc_path` + `status=IN_PROGRESS`
- Warn developer: severity · issue_id · anti-failure pattern applied

**MEDIUM / LOW — defer to `post_task_node`** (if resolution > 15 minutes):
- `IssueRecord.doc_path = None` recorded during IMPLEMENT
- `post_task_node` creates all deferred doc(s), sets `status=RESOLVED`

**After any issue doc is created:**
- Update SKILL.md Known Issues
- Trigger `librarian_sync` n8n pipeline to index the new document
- For CRITICAL/HIGH with resolution → also append to `ai_failure_modes_log` (C-18)

---

## §8 Quality Checklist

Before finalizing any issue document:

- [ ] Title is descriptive — folder scan reveals the problem at a glance
- [ ] Quick Summary is non-technical (executive-level clarity)
- [ ] Root Cause explains WHY (not just what happened)
- [ ] Solution is reproducible by someone else following the steps
- [ ] Commands include **full output**, not just the command itself
- [ ] Verification steps are specific: "tested X → expected Y → got Z"
- [ ] Prevention measures are actionable (CI hook / monitoring alert / architecture change)
- [ ] Cross-references added per §6 (SKILL.md · ADR · runbook)
- [ ] Filename follows `ISSUE-NNNN-slug.md` convention (§2)
- [ ] Status reflects reality — RESOLVED only if truly fixed, not just mitigated
