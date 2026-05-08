# Sequence: 10-Phase Workflow Execution

> Part of ADR-Lite-001. Shows the end-to-end request lifecycle from task intake to completion.
> See also: `dw_lite_seq_care_lite.md` (CARE-Lite retrieval detail).

---

## Diagram

```mermaid
sequenceDiagram
  autonumber
  actor       Dev   as Developer
  participant Skill as DevWeaver-Lite Skill
  participant Lib   as Markdown Librarian
  participant MCP   as MCP Layer
  participant Proj  as Target Project

  Dev->>+Skill: Describe task and provide project_root
  Note over Skill: Phase 1 - PRE-TASK
  Skill->>Dev: Confirm scope, constraints, acceptance criteria
  Dev-->>Skill: Scope confirmed

  Note over Skill: Phase 2 - REQUIREMENTS
  Skill->>Skill: Extract functional and non-functional requirements
  Skill->>Dev: Present structured requirements draft
  Dev-->>Skill: Requirements approved

  Note over Skill: Phase 3 - FETCH_RULES
  Skill->>+Lib: CATALOG scan - keyword match on domain and task type
  Lib-->>-Skill: Matched rule file paths (tier 1 candidates)
  Skill->>+MCP: Context7 - fetch library and framework docs (tier 2)
  MCP-->>-Skill: Official library documentation

  Note over Skill: Phase 4 - RESEARCH
  Skill->>+MCP: GitHub MCP - read existing codebase patterns
  MCP-->>-Skill: Existing code context and conventions
  Skill->>Skill: Synthesize rules, docs, and codebase into working context

  Note over Skill: Phase 5 - ARCHITECTURE
  Skill->>Skill: Produce C4 Level 2 container + sequence diagrams
  Skill->>Skill: Produce implementation plan (phased, test-first)
  Skill->>Dev: Present architecture proposal and plan

  rect rgb(255, 245, 180)
    Note over Skill,Dev: CONFIRM gate - human approval required
    Skill-->>Dev: Deliverables: requirements + architecture + plan
    alt Developer approves
      Dev->>Skill: Approve
    else Developer requests changes
      Dev->>Skill: Return feedback
      Skill->>Skill: Revise proposal
      Skill->>Dev: Re-present revised deliverables
    end
  end

  Note over Skill: Phase 6 - IMPLEMENT
  Skill->>+Proj: Write tests first (TDD), then implementation
  Proj-->>-Skill: Files written and staged

  Note over Skill: Phase 7 - VALIDATE
  Skill->>+Proj: Run tests, linter, type-checker, security scan
  Proj-->>-Skill: Validation results
  alt All checks pass
    Skill->>Dev: Present validation summary
  else Checks failing
    Skill->>Skill: Fix failures and re-validate
  end

  rect rgb(255, 245, 180)
    Note over Skill,Dev: GIT GATE - human approval required before commit
    Skill-->>Dev: Staged diff and test results
    alt Developer approves
      Dev->>Skill: Approve commit
      Skill->>+MCP: GitHub MCP - commit and push to branch
      MCP-->>-Skill: Commit SHA confirmed
    else Developer requests amendments
      Dev->>Skill: Return amendment request
      Skill->>Skill: Apply amendments
    end
  end

  Note over Skill: Phase 8 - POST-TASK
  Skill->>Skill: 6-step sweep: docs update, README, ADR, CATALOG, templates, CHANGELOG
  Skill-->>-Dev: Task complete - summary and next steps
```
