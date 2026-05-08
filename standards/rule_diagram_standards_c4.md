# rule_diagram_standards_c4

> C4 diagramming conventions — `flowchart LR` + `subgraph` for all structural levels (L1/L2/L3).
> **See also**: `rule_diagram_standards_sequence.md` · `rule_diagram_standards.md` (index)

---

## 1. Diagram Type Selection

| What to convey | Mermaid keyword | Who reads it |
|---|---|---|
| System actors + external boundaries (L1) | `flowchart LR` + `subgraph` | Everyone |
| Internal containers + data stores (L2) | `flowchart LR` + `subgraph` | Architects + ops |
| Internals of one complex container (L3) | `flowchart LR` + `subgraph` | Developers |
| Time-ordered cross-service interactions | `sequenceDiagram` | Developers |
| State machine transitions | `stateDiagram-v2` | Developers |
| Git branching model | `gitGraph` | All |

**Rule**: Use `flowchart LR` for ALL C4 levels — `C4Context`, `C4Container`, `C4Component` lack auto-layout.  
**Rule**: Never use `graph TD` — `flowchart LR` minimises vertical overlap via Dagre.

---

## 2. C4 Element Types

```
%% Person / actor
id(["Person Name\n[Person]"])

%% System or Container (service / app)
id["Label\n[Technology]\nShort description"]

%% Database / data store (cylinder)
id[("Label\n[Technology]\nShort description")]

%% External system (dashed border via classDef)
id["Label\n[External System]\nShort description"]:::external
classDef external stroke-dasharray:5 5,fill:#f8f8f8

%% Boundary wrapper
subgraph boundary_id ["Boundary Label"]
  ...
end
```

---

## 3. Split Threshold Rules

**Threshold**: Split an L2 diagram when **either** limit is reached:
- > 15 edges in one diagram, OR
- > 3 boundary subgraphs

| View suffix | Scope |
|---|---|
| `L2a` | Core modules + data stores |
| `L2b` | Integration / MCP layer |
| `L2c` | Cross-cutting concern (Librarian / observability / auth) |

Naming: title = `Level 2a · <Scope Label>` · heading = `### 2.2 Level 2a — <Scope Label>`  
Each view must be self-contained — redeclare every referenced node.

---

## 4. Relationship Syntax

```
%% Standard labelled directed edge
A -->|"Reads session data<br/>asyncpg"| B

%% Dotted edge — cross-cutting concerns only
A -.->|"cross-cuts"| B

%% Unlabelled — only for self-evident SKILL.md routing lines
A --> B
```

**Rules:**
- Every edge touching a data-store MUST include driver/protocol in the label.
- Use `-.->` for observability / auth cross-cuts.
- Inter-subgraph edges MUST be labelled. Intra-subgraph MAY be unlabelled.

---

## 5. Naming Conventions

| Field | Convention | Example |
|---|---|---|
| Node alias | `snake_case`, unique | `mcp_qdrant`, `pg_db` |
| Label | Title-case noun phrase, ≤ 3 words | `"Qdrant MCP"` |
| Technology | `Name version` or `Protocol` | `"FastAPI 0.115"`, `"asyncpg"` |
| Description | One sentence, active voice, ≤ 12 words | `"Stores embedded domain knowledge"` |

---

## 6. Layout and Style

```
---
title: Level 2a · My Scope
---
flowchart LR

classDef external stroke-dasharray:5 5,fill:#f8f8f8
classDef approval fill:#fef9c3,stroke:#eab308
external_system:::external
```

- Declare nodes top-to-bottom within each `subgraph` in the order you want them stacked.
- Put all `subgraph` declarations **before** edge definitions.
- `classDef` goes at the bottom of the diagram body.

---

## 7. C4 Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| `graph TD` for system architecture | Use `flowchart LR` |
| L2 showing class names / methods | L2 shows containers only; L3 shows components |
| Unlabelled or vague `Rel` ("Uses", "Calls") | Use verb phrases: `"Reads session data from"` |
| Missing technology tag | Always fill technology string |
| > 15 edges in one diagram | Split into L2a / L2b / L2c |
| `BiRel` for all relationships | Reserved for true symmetric protocols only |
| No `title` line | Every diagram must have a `title` |

---

## 8. Known Mermaid C4 Constraints

| Constraint | Detail |
|---|---|
| `C4Context` / `C4Container` | Marked experimental — use `flowchart LR` instead |
| `Lay_U/D/L/R` | Not supported in Mermaid |
| `AddElementTag`, `Legend`, `sprite` | Not implemented |
| CSS theming | Fixed internal CSS; `%%{init:…}%%` does not apply |
| `UpdateRelStyle` | Must follow its `Rel()` immediately |
