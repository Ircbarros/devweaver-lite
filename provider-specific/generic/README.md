# DevWeaver-Lite - Generic Provider Setup

> Use this package when your provider is not one of the named providers.
> Works with any OpenAI-compatible API, any system message field, or any
> file-upload context injection mechanism.

---

## Zero-Install Quick Start

1. Copy SYSTEM_PROMPT.md (at the devweaver-lite root) or the relevant provider context file.
2. Paste it as the system prompt / system instructions / custom instructions in your provider.
3. At the start of each session, tell the agent:
   - Your project root path: "I am working on the project at /path/to/my-project"
   - The project mode: SCRATCH / IMPROVE / CONTINUE / REFACTOR

No npm, no environment variables, no installation required.

---

## Provider Compatibility

| Interface | How to inject |
|---|---|
| System prompt field | Paste SYSTEM_PROMPT.md or provider context file content |
| Custom instructions | Paste in the instructions field |
| Knowledge / Memory upload | Upload SKILL.md + librarian/CATALOG.md as knowledge files |
| OpenAI-compatible API | messages=[{"role": "system", "content": open("SYSTEM_PROMPT.md").read()}] |
| LangChain | SystemMessage(content=open("SYSTEM_PROMPT.md").read()) |
| AutoGen | AssistantAgent(system_message=open("SYSTEM_PROMPT.md").read()) |
| CrewAI | Agent(backstory=open("SYSTEM_PROMPT.md").read()) |
| Dify / Flowise | Paste into the System Prompt field |
| Agno | Agent(instructions=open("SYSTEM_PROMPT.md").read()) |

---

## MCP Configuration (optional)

If your provider supports MCP servers, configure at minimum:

| MCP | Type | Config |
|---|---|---|
| GitHub | HTTP or stdio binary | HTTP: https://api.githubcopilot.com/mcp (Copilot) or binary with GITHUB_TOKEN |
| Context7 | HTTP | url: https://mcp.context7.com/mcp |
| Playwright | stdio / npx | command: npx, args: ["-y", "@playwright/mcp"] |
| Filesystem | stdio / npx | command: npx, args: ["-y", "@modelcontextprotocol/server-filesystem", "/your/project/path"] |

See the named provider packages (copilot/, cursor/, windsurf/) for ready-to-copy MCP config files.

---

## Providing the Project Path

For providers without workspace context (most web and API providers), tell the agent at the
start of each session:

  "Work on the project at /absolute/path/to/my-project"

The agent will store this as project_root and use it for all file operations.

---

## Context Budget

Set context_budget_remaining = (your model's total context window) - 8000.

Common values:
- GPT-4o: 120 000 (128 000 - 8 000)
- Claude 3.5 Sonnet: 192 000 (200 000 - 8 000)
- Gemini 1.5 Pro: 992 000 (1 000 000 - 8 000)
- Mistral Large: 120 000 (128 000 - 8 000)
- Kimi k1.5: 120 000 (128 000 - 8 000)

---

## Troubleshooting

- Skill does not activate: verify the full content was pasted, not just the header.
- CONFIRM gate not firing: check that the provider did not truncate the system prompt.
- L3 references unavailable: this is expected for web/API providers without Filesystem MCP.
  The agent will use model knowledge as fallback (CARE-Lite tier 3).
