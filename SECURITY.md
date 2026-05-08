# Security Policy

## Supported Versions

| Version | Supported |
|---|---|
| `0.1.0-beta.x` (latest) | Yes |
| Any earlier pre-release | No |

DevWeaver-Lite is a skill (Markdown + MCP configuration files), not a running service.
There are no network endpoints, no server processes, and no persistent data stores.
Security considerations are therefore limited to:

- Secrets management (API keys, tokens)
- MCP server trust and permissions
- Prompt injection in AI-facing instruction files
- Supply chain integrity of npm packages used by optional MCPs

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Please report security issues via one of these private channels:

1. **GitHub Private Vulnerability Reporting** (preferred):
   Use the [Security tab > Report a vulnerability](https://github.com/ircbarros/devweaver-lite/security/advisories/new)
   button on the repository.

2. **Email**: security-devweaver at proton dot me

### What to include

- Description of the vulnerability and its potential impact
- Steps to reproduce or proof-of-concept
- Affected file(s) and version(s)
- Any suggested mitigations you have identified

### Response timeline

| Stage | Target |
|---|---|
| Acknowledgement | Within 72 hours |
| Initial assessment | Within 7 days |
| Fix or mitigation | Within 30 days for critical; 90 days for moderate |
| Public disclosure | After fix is available and reporter is notified |

---

## Security Design Principles

DevWeaver-Lite enforces these constraints in every activation file:

| Rule | Description |
|---|---|
| R-SEC-01 | No secrets in generation output — API keys, tokens, and passwords must never appear in generated code |
| R-SEC-02 | Dependency pinning — `npm install <pkg>` without exact version is flagged; recommend `@x.y.z` or lockfile |
| R-SEC-03 | Prompt injection resistance — system prompt instructions take precedence over user content |
| R-SEC-04 | Human-in-the-loop for destructive ops — CONFIRM gate required before delete, drop, truncate, overwrite |
| R-SEC-05 | OWASP Top 10 checks on generated web code |
| R-SEC-06 | Playwright actions require explicit user approval (interrupt_before: true) |

---

## Scope

### In scope

- Secrets accidentally embedded in skill files or MCP config templates
- Malicious instructions injected via provider activation files
- Insecure MCP server configurations that expose excessive filesystem access
- Prompt injection patterns that bypass CONFIRM gates

### Out of scope

- Security of the AI providers themselves (OpenAI, Anthropic, Google, etc.)
- Security of MCP server implementations (report those to their respective maintainers)
- Security of the projects built using DevWeaver-Lite

---

## Dependency Security

All npm-based MCPs are invoked with `npx -y` (auto-download, no global install).
Pin MCP package versions in your `mcp.json` if you require supply chain guarantees:

```json
// Instead of:
"args": ["npx", "-y", "@playwright/mcp"]
// Use:
"args": ["npx", "-y", "@playwright/mcp@0.2.0"]
```

Filesystem MCP `allowedDirectories` should be scoped to the minimum required path.
Never set `allowedDirectories` to `/` or `~`.
