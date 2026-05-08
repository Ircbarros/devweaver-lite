# Sequence: CARE-Lite Retrieval Protocol

> Part of ADR-Lite-001. Shows the 4-tier knowledge retrieval chain used in Phase 3 (FETCH_RULES)
> and Phase 4 (RESEARCH) of every workflow execution.
> See also: `dw_lite_seq_workflow.md` (full workflow context).

---

## Diagram

```mermaid
sequenceDiagram
  autonumber
  participant Skill as Skill Core
  participant Lib   as Markdown Librarian
  participant FS    as Filesystem MCP
  participant C7    as Context7 MCP
  participant Web   as Web Search
  participant Model as Model Knowledge

  Note over Skill: CARE-Lite 4-tier retrieval begins
  Skill->>+Lib: CATALOG scan - keyword match on domain and topics
  Lib-->>-Skill: Candidate file list with ISBN, path, confidence estimate

  alt Confidence >= 0.9 - tier 1 sufficient
    Skill->>+FS: Read matched rule or reference files
    FS-->>-Skill: File content loaded
    Note over Skill: Proceed with tier 1 result - stop retrieval
  else Confidence < 0.9 - escalate to tier 2
    Skill->>+C7: resolve-library-id for relevant library
    C7-->>Skill: Library ID resolved
    Skill->>C7: get-library-docs with resolved ID
    C7-->>-Skill: Official library documentation
    alt Confidence >= 0.8 - tier 2 sufficient
      Note over Skill: Proceed with tier 2 result - stop retrieval
    else Confidence < 0.8 - escalate to tier 3
      Skill->>+Web: Search official docs, blogs, community forums
      Web-->>-Skill: Search results and excerpts
      alt Confidence >= 0.7 - tier 3 sufficient
        Note over Skill: Proceed with tier 3 result - stop retrieval
      else Confidence < 0.7 - fallback to tier 4
        Skill->>+Model: Use model training knowledge as last resort
        Model-->>-Skill: Best-effort answer
        Note over Skill: Annotate response with [tier-4-fallback] tag
      end
    end
  end

  Note over Skill: Retrieval complete - inject context into active phase
```

---

## Confidence Thresholds

| Tier | Source | Stop threshold | Fallback action |
|---|---|---|---|
| 1 | Markdown Librarian + Filesystem MCP | >= 0.9 | Escalate to tier 2 |
| 2 | Context7 MCP (official docs) | >= 0.8 | Escalate to tier 3 |
| 3 | Web search (docs, blogs, forums) | >= 0.7 | Escalate to tier 4 |
| 4 | Model training knowledge | Always | Tag `[tier-4-fallback]` |
