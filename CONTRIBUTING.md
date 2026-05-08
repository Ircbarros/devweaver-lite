# Contributing to DevWeaver-Lite

Thank you for your interest in contributing! DevWeaver-Lite is a provider-agnostic AI skill delivered as Markdown files and MCP configuration templates.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Ways to Contribute](#ways-to-contribute)
- [Good First Issues](#good-first-issues)
- [Development Setup](#development-setup)
- [Pull Request Process](#pull-request-process)
- [Commit Style](#commit-style)
- [File Standards](#file-standards)
- [Issue Labels](#issue-labels)

---

## Code of Conduct

This project follows the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

By participating you agree to uphold a respectful, harassment-free environment.

Violations can be reported to the maintainers via the Security contact in [SECURITY.md](SECURITY.md).

---

## Ways to Contribute

| Type | How |
|---|---|
| Bug report | Open an issue using the Bug Report template |
| Feature request | Open an issue using the Feature Request template |
| New provider | Open an issue or PR adding `provider-specific/<name>/` package |
| Standard / reference update | Edit files in `standards/` or `references/` and open a PR |
| Documentation | Edit any `.md` file and open a PR |
| Security issue | See [SECURITY.md](SECURITY.md) — do NOT open a public issue |

---

## Good First Issues

Issues labelled [`good first issue`](https://github.com/ircbarros/devweaver-lite/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22)
are scoped to be completed without deep prior knowledge of the codebase.
Typical examples:

- Add missing provider context file (e.g., Grok, Perplexity)
- Improve troubleshooting section in `README.md`
- Add a new reference document to `references/`
- Fix a broken link or typo

Comment on the issue to claim it before starting work.

---

## Development Setup

DevWeaver-Lite has no build system — it is pure Markdown and JSON.

**Recommended tools** (for local linting before push):

```bash
# Markdown linting
npm install -g markdownlint-cli2

# YAML linting
pip install yamllint

# JSON validation (Node.js built-in)
node -e "require('fs').readdirSync('.').filter(f=>f.endsWith('.json')).forEach(f=>{ JSON.parse(require('fs').readFileSync(f,'utf8')); console.log('OK', f); })"
```

Run the linters:

```bash
markdownlint-cli2 "**/*.md" "#node_modules"
yamllint .github/workflows/
```

These same checks run in CI — fix them locally first to keep the feedback loop short.

---

## Pull Request Process

1. **Fork** the repository and create a feature branch from `main`:
   ```bash
   git checkout -b feat/add-perplexity-provider
   ```

2. **Make your changes** following the [File Standards](#file-standards) below.

3. **Run linters** locally (see Development Setup) and fix all warnings.

4. **Open a PR** against `main` with a clear title and description.
   Link the related issue if one exists (`Closes #123`).

5. **CI must pass** — the PR will not be merged with failing lint or secret-scan checks.

6. **One approval** from a maintainer is required before merge.

7. **Squash merge** is preferred to keep the commit history clean.

---

## Commit Style

Use the [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<scope>): <short summary>

[optional body]

[optional footer(s)]
```

| Type | When to use |
|---|---|
| `feat` | New provider, new standard, new reference |
| `fix` | Bug in activation file, broken MCP config |
| `docs` | README, CONTRIBUTING, ADR updates |
| `chore` | CI, .gitignore, tooling |
| `refactor` | Restructuring without behavior change |
| `security` | Security-related changes |

Examples:
```
feat(provider): add Perplexity context file
fix(copilot): correct npx -y flag in mcp.json
docs(readme): add Grok to provider table
security(mcp): scope filesystem allowedDirectories
```

---

## File Standards

These constraints apply to all files and are enforced by the CI linter:

| Rule | Requirement |
|---|---|
| R-PD-01 | Skill files <= 500 lines |
| R-PD-02 | No emoji in instruction files (breaks some tokenizers) |
| R-PD-03 | No secrets, tokens, or passwords in any committed file |
| R-DS-10 | No em-dash (`—`) — use hyphen-minus (`-`) |
| JSON files | Valid JSON; use JSONC comments only in files documented as JSONC |
| YAML files | 2-space indent, no tabs |
| Markdown | markdownlint-cli2 compliant |

New provider packages must follow the structure of an existing provider (e.g., `provider-specific/cursor/`):

```
provider-specific/<name>/
  <activation-file>          # e.g., .cursorrules, context.md
  mcp_config.json            # behavioral reference (allowed_tools, etc.)
  <mcp_template>.json        # optional: IDE MCP config template
```

---

## Issue Labels

| Label | Meaning |
|---|---|
| `good first issue` | Small, well-scoped, beginner-friendly |
| `bug` | Something is broken |
| `enhancement` | New feature or improvement |
| `provider` | Related to a specific AI provider |
| `mcp` | Related to MCP server configuration |
| `security` | Security-relevant change |
| `documentation` | Docs-only change |
| `needs-triage` | Awaiting maintainer review |
| `help wanted` | Good for community contribution |

---

## Support

For questions that are not bugs or feature requests:

- Open a [Discussion](https://github.com/ircbarros/devweaver-lite/discussions) on GitHub
- Check existing issues and the `references/teaching/guide_qa_simulation.md` for common patterns
