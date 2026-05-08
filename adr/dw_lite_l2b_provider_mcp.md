# Level 2b - Container View: Provider Packages and MCP Integration

> Part of ADR-Lite-001. See also: `dw_lite_l2a_core_containers.md` (core knowledge containers).
> **Rule applied**: L2 split per `rule_diagram_standards_c4.md` §3 (>3 boundary subgraphs).

---

## Scope

This diagram shows how each provider package connects to MCP servers. It distinguishes
HTTP cloud endpoints (zero install) from stdio npx transports (auto-download on first use)
and highlights the Playwright MCP as approval-gated.

---

## Diagram

```mermaid
---
title: "Level 2b - Container View: Provider Packages and MCP Integration"
---
flowchart LR

  subgraph ide_providers ["IDE Provider Packages [4 packages]"]
    copilot["Copilot Package\n[.github/ + .vscode/mcp.json]\nCopilot-instructions + VS Code\nMCP config template"]
    cursor["Cursor Package\n[.cursorrules + .cursor/mcp.json]\nCursorRules + Cursor\nMCP config template"]
    windsurf["Windsurf Package\n[.windsurfrules + .windsurf/mcp.json]\nWindsurfRules + Windsurf\nMCP config template"]
    claude_d["Claude Desktop Package\n[CLAUDE.md + config.json.example]\nCLAUDE.md + Desktop\nMCP config example"]
  end

  subgraph web_providers ["Web Provider Packages [7 packages]"]
    web_pkgs["Web + Generic Packages\n[*_context.md + generic/README.md]\nGemini / OpenAI / OpenRouter / Kimi\nMistral / OpenClaw / Generic"]
  end

  subgraph mcp_http ["MCP: HTTP Cloud [zero install]"]
    gh_http["GitHub MCP\n[HTTP]\nhttps://api.githubcopilot.com/mcp\nCopilot session auth - no token"]
    context7["Context7 MCP\n[HTTP]\nhttps://mcp.context7.com/mcp\nCONTEXT7_API_KEY"]
  end

  subgraph mcp_stdio ["MCP: stdio [npx -y auto-download]"]
    gh_stdio["GitHub MCP\n[stdio binary]\ngithub-mcp-server\nRequires GITHUB_TOKEN"]
    playwright["Playwright MCP\n[npx stdio]\n@playwright/mcp\nApproval-gated"]
    filesystem["Filesystem MCP\n[npx stdio]\nserver-filesystem\nAllowedDirectories scoped"]
    memory["Memory MCP\n[npx stdio]\nserver-memory\nOptional - cross-session state"]
  end

  copilot -->|"HTTP - no token required"| gh_http
  copilot -->|"HTTP - CONTEXT7_API_KEY"| context7
  copilot -.->|"Optional npx -y"| playwright
  copilot -.->|"Optional npx -y"| filesystem
  cursor -->|"stdio - GITHUB_TOKEN required"| gh_stdio
  cursor -->|"HTTP - CONTEXT7_API_KEY"| context7
  cursor -.->|"Optional npx -y"| playwright
  windsurf -->|"stdio - GITHUB_TOKEN required"| gh_stdio
  windsurf -->|"HTTP - CONTEXT7_API_KEY"| context7
  windsurf -.->|"Optional npx -y"| playwright
  claude_d -->|"stdio - GITHUB_TOKEN required"| gh_stdio
  claude_d -->|"mcp-remote wrapper"| context7
  claude_d -.->|"Optional npx -y"| playwright
  claude_d -.->|"Optional npx -y"| filesystem
  web_pkgs -.->|"No MCP - model knowledge only"| context7

  classDef approval fill:#fef9c3,stroke:#eab308
  playwright:::approval

  classDef external stroke-dasharray:5 5,fill:#f8f8f8
  gh_http:::external
  gh_stdio:::external
  context7:::external
  playwright:::external
  filesystem:::external
  memory:::external
```

---

## Notes

- Dotted edges (`-.->`) from web_pkgs to context7 indicate no MCP connection is possible
  for web/chat providers; context7 is shown only to illustrate the capability gap.
- The `playwright` node is styled with the approval class to signal that every call
  requires explicit human confirmation (interrupt_before: true in mcp_config.json).
- `memory` is not shown with any provider edge because it requires explicit user opt-in;
  it is not configured by default in any MCP template.
