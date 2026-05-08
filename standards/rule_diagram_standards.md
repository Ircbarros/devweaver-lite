# rule_diagram_standards

> Index file — split into focused sub-files per 
ule_context_window.md §2 (≤ 200 lines each).
> **Rule**: Every ADR MUST contain a C4 Level 2 Container diagram + at least one UML Sequence diagram.
> Use \mermaid.live\ to validate all diagrams before committing.

---

## Sub-files

| File | Contents |
|---|---|
| 
ule_diagram_standards_c4.md | Diagram type selection · C4 element types · split threshold · relationship syntax · naming · layout · anti-patterns · known constraints |
| 
ule_diagram_standards_sequence.md | UML Sequence participants · arrow types · activation bars · control flow · notes · \stateDiagram-v2\ rules · anti-patterns · tooling |
| 
ule_context_window.md | Progressive disclosure (L1/L2/L3) · file size limits · node resource table · token budget tracking |

---

## Quick-Reference Rules

- **C4 all levels**: lowchart LR + subgraph (never C4Context, C4Container, C4Component)
- **Flow diagrams**: sequenceDiagram with autonumber
- **State machines**: stateDiagram-v2 with direction LR
- **Split threshold**: > 15 edges or > 3 boundary subgraphs → split L2a / L2b / L2c
- **File size**: SKILL.md ≤ 500 lines · all other .md ≤ 200 lines (R-PD-01 / R-PD-02)
- **CI**: All diagrams must render in mermaid.live (v11.x) before merge
