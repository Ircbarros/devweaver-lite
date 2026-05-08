# rule_diagram_standards_sequence

> UML Sequence and State Machine diagramming conventions.
> **See also**: `rule_diagram_standards_c4.md` · `rule_diagram_standards.md` (index)

---

## 1. Participant Declaration

Always declare all participants explicitly at the top — never rely on implicit creation:

```
sequenceDiagram
  autonumber
  actor       Dev  as Developer
  participant API  as "API Gateway"
  participant DB   as "PostgreSQL"
```

Declare participants **left-to-right** in the dominant message-flow direction.

---

## 2. Message Arrow Types

| Syntax | Meaning | When to use |
|---|---|---|
| `A->>B: msg` | Synchronous call | Blocking RPC, function call |
| `A-->>B: msg` | Response (dotted) | Return from synchronous call |
| `A-)B: msg` | Async fire-and-forget | Queue write, event publish, mem0 |
| `A-xB: msg` | Lost / dropped message | Timeout, error, connection drop |

**Rules:**
- Never use `A->B` (no arrowhead) for inter-service calls.
- Always label return arrows with payload or status: `-->>B: 200 OK + {answer}`.
- Use `-)` for all fire-and-forget async calls (mem0, ntfy, event bus).

---

## 3. Activation Bars

Use `+`/`-` shorthand consistently — avoid mixing with explicit `activate`/`deactivate`:

```
Client->>+ServiceA: Request
ServiceA-->>-Client: Response
```

---

## 4. Control Flow Structures

```
alt user is authenticated
  A->>B: fetch profile
else unauthenticated
  A->>B: return 401
end

opt admin role
  A->>B: fetch audit logs
end

loop confidence < 0.9
  LG->>LIB: re-query with refined terms
end

par dense search
  LG->>QD: dense_search(vector)
and sparse search
  LG->>QD: sparse_search(bm25)
end

rect rgb(255, 245, 180)
  note over LG,Dev: ⚠ interrupt_before — human approval required
  LG-->>Dev: present deliverables
  alt approved
    Dev->>LG: approve
  else changes requested
    Dev->>LG: return feedback
  end
end
```

---

## 5. Notes

```
Note right of Alice: Right-side annotation
Note left  of Bob:   Left-side annotation
Note over  Alice,Bob: Spans both participants
```

Use notes for:
- Security gates (`⚠ approval required`)
- State transitions (`transition RESEARCH → ARCHITECTURE`)
- Out-of-band events not shown in the flow

---

## 6. stateDiagram-v2 (State Machine)

Use `stateDiagram-v2` for LangGraph state machines and workflow state transitions.

```
stateDiagram-v2
  direction LR
  [*] --> IDLE
  IDLE --> RUNNING : task_id received
  state RUNNING : active execution
  RUNNING --> DONE : all gates passed
  RUNNING --> ERROR : unrecoverable failure
  DONE --> [*]
  ERROR --> [*]
```

**Rules:**
- `direction LR` always for LangGraph state diagrams.
- `interrupt_before` gates: expressed as state description — `state CONFIRM : ⚠ interrupt_before — human approval gate`
- Do NOT use `note right of` inside `stateDiagram-v2` — the parser is unreliable; use state descriptions instead.

---

## 7. Sequence Diagram Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| > 8 participants | Split by sub-flow or use `box` to group |
| Implicit participant creation | Declare all `participant`/`actor` at top |
| `A->>B` for queue write | Use `A-)B` (async) |
| Unlabelled `-->>` return | Always label: `-->>: 200 OK + {data}` |
| Mixed service/code-level actors | Decide one abstraction level per diagram |
| Deeply nested `alt/loop/par` | Extract inner blocks as a sub-flow diagram |
| Mixed `+/-` and `activate`/`deactivate` | Use `+`/`-` shorthand exclusively |

---

## 8. Known Mermaid Sequence Constraints

| Constraint | Detail |
|---|---|
| `end` in message text | Escape as `"end"`, `(end)`, or `[end]` |
| `<<->>` bidirectional | Requires Mermaid v11.0.0+ |
| `create`/`destroy` | Requires Mermaid v10.3.0+; destruction bug fixed in v10.7.0+ |
| `autonumber` | Applies to the full diagram — no per-section reset |
| Nested `par` | Max 2 levels — split beyond that |
| `box` grouping | Uses color name or `rgb()`/`rgba()`; `transparent` suppresses fill |

---

## 9. Tooling

| Tool | Purpose |
|---|---|
| `mermaid.live` | Validate and preview diagrams in browser |
| `mmdc` (Mermaid CLI) | Generate SVG/PNG in CI: `npm i -g @mermaid-js/mermaid-cli` |
| VS Code extension | `bierner.markdown-mermaid` for live preview |

**CI rule**: All diagrams must render without errors in `mermaid.live` (Mermaid v11.x) before PR merge.
