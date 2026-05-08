# Level 2a - Core Containers: Knowledge and Skill Definition

> Part of ADR-Lite-001. See also: `dw_lite_l2b_provider_mcp.md` (provider and MCP integration).
> **Rule applied**: L2 split per `rule_diagram_standards_c4.md` §3 (>3 boundary subgraphs).

---

## Scope

This diagram shows the internal containers of the DevWeaver-Lite skill package that are
responsible for knowledge storage, skill definition, and workflow instructions.

---

## Diagram

```mermaid
---
title: "Level 2a - Core Containers: Knowledge and Skill Definition"
---
flowchart LR

  subgraph skill_core ["Skill Core [Markdown Instructions]"]
    shared_skill["Shared SKILL.md\n[Markdown + YAML]\nCanonical 233-line skill definition\nand 10-phase workflow"]
    system_prompt["SYSTEM_PROMPT.md\n[Markdown]\nFlat injection for API\nand framework use"]
  end

  subgraph activation ["Provider Activation Files [10 Providers]"]
    ide_act["IDE Activation\n[.md / .rules files]\nCopilot / Cursor / Windsurf / Claude\nFull SKILL.md embedded inline"]
    web_act["Web Activation\n[*_context.md files]\nGemini / OpenAI / OpenRouter\nKimi / Mistral / OpenClaw"]
  end

  subgraph librarian_layer ["Markdown Librarian [Knowledge Index]"]
    catalog[("CATALOG.md\n[Markdown]\n56 knowledge files indexed\nby domain, tier, topics, path")]
    domain_idx[("Domain Indexes\n[Markdown]\n9 per-domain filtered views\nof CATALOG.md")]
    query_rule["rule_librarian_query.md\n[Markdown]\nCARE-Lite tier 1 retrieval\nprotocol and confidence rules"]
  end

  subgraph knowledge ["Reference Knowledge [L3 Reads via Filesystem MCP]"]
    stds[("Standards\n[Markdown]\n14 rule files\nR-PD / R-DS / R-SEC series")]
    refs[("References\n[Markdown]\n35 domain guides across\n9 knowledge domains")]
    tmpl[("Templates\n[Markdown]\n6 reusable files\nADR / PRD / CARE plan")]
  end

  shared_skill -->|"Embedded inline in"| ide_act
  shared_skill -->|"Embedded inline in"| web_act
  shared_skill -->|"Derived into flat form"| system_prompt
  query_rule -->|"Scans and ranks"| catalog
  catalog -->|"Filtered into"| domain_idx
  catalog -.->|"Refers to paths in"| refs
  catalog -.->|"Refers to paths in"| stds
  catalog -.->|"Refers to paths in"| tmpl

  classDef store fill:#dbeafe,stroke:#3b82f6
  catalog:::store
  domain_idx:::store
  refs:::store
  stds:::store
  tmpl:::store
```

---

## Notes

- `shared_skill` is the single source of truth; activation files are generated from it.
- Dotted edges (`-.->`) indicate that CATALOG.md stores paths only; actual file content
  is read on demand by the Filesystem MCP during CARE tier 1 execution.
- `system_prompt` is equivalent to `ide_act` / `web_act` but targets API and framework callers.
