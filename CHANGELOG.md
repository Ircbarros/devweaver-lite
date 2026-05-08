# Changelog

All notable changes to DevWeaver-Lite are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Provider-specific troubleshooting guides
- Additional framework injection examples (Haystack, AutoGen v0.4)
- Shell bootstrap script for project setup verification

---

## [0.1.0-beta.1] - 2026-05-08

### Added
- **Zero-install architecture** — full skill definition embedded inline in every provider activation
  file; no `Load SKILL.md` path references that would fail in web providers (Decision 8 in ADR)
- **10-provider support** — GitHub Copilot, Cursor, Windsurf, Claude, Gemini, OpenAI, OpenRouter,
  Kimi, Mistral, OpenClaw, plus a generic provider guide
- **MCP config templates** — provider-specific templates for VS Code (`.vscode/mcp.json`),
  Cursor (`.cursor/mcp.json`), Windsurf (`.windsurf/mcp.json`), and Claude Desktop
- **GitHub MCP** via native Copilot HTTP endpoint (no token required for VS Code Copilot),
  or `github-mcp-server` stdio binary for other IDE providers
- **Context7 MCP** via HTTP endpoint (zero install); `mcp-remote` wrapper for Claude Desktop
- **Playwright MCP** optional via `npx -y @playwright/mcp` (approval-gated, human-in-the-loop)
- **Filesystem MCP** optional via `npx -y @modelcontextprotocol/server-filesystem` for L3 references
- **FastAPI-docs MCP** and **InfluxDB MCP** stubs in all `mcp_config.json` behavioral references
- **Markdown Librarian** (`librarian/CATALOG.md`) — keyword-indexed knowledge catalog replacing
  Qdrant + PostgreSQL for zero-infra deployments
- **SYSTEM_PROMPT.md** universal flat injection for LangChain, AutoGen, CrewAI, Agno, Dify,
  and any OpenAI-compatible API
- **14 standards files** — R-PD-*, R-DS-*, R-SEC-* rule sets
- **35 references** across architecture, security (OWASP), data science, and teaching domains
- **6 reusable templates** for ADRs, PRDs, and CARE plan files
- **ADR-Lite-001** architectural decision record documenting all 8 design decisions
- **Project path handling** — IDE providers detect workspace path automatically; web providers
  prompt the user at session start
- Context budget enforcement per provider (Gemini 992K, Claude 192K, others 120K)

### Security
- R-SEC-01 through R-SEC-06 OWASP-aligned security rules embedded in every activation file
- CONFIRM gate required before all destructive operations
- Prompt injection resistance rules (R-SEC-03)
- No secrets or credentials in any committed file

### Architecture Notes
- All activation files are self-contained (~230 lines each); no external file reads required
- `mcp_config.json` files serve as behavioral references (allowed_tools, interrupt_before) —
  they are NOT VS Code/IDE MCP config files; the new `.vscode/mcp.json` templates serve that role
- VS Code MCP uses `"servers"` key (not `"mcpServers"`) with HTTP type for Copilot-native endpoints
- Cursor and Windsurf use `"mcpServers"` (camelCase) key

---

[Unreleased]: https://github.com/ircbarros/devweaver-lite/compare/v0.1.0-beta.1...HEAD
[0.1.0-beta.1]: https://github.com/ircbarros/devweaver-lite/releases/tag/v0.1.0-beta.1
