---
project: "{project_name}"
version: "{semver}"
status: "DRAFT"
audience: "all"
last_updated: "{ISO-8601}"
---

# {Project Name}

> {One sentence: what this does and for whom.}
> Example: "FastAPI + LangGraph agent that scaffolds production-grade full-stack apps from a SKILL.md spec."

[![CI](https://img.shields.io/{ci_badge_path})]({ci_url})
[![Coverage](https://img.shields.io/{coverage_badge_path})]({coverage_url})
[![License](https://img.shields.io/badge/license-{license}-blue)](LICENSE)

---

## Why This Exists

<!-- NEED beat (Truby): what problem does this solve? For whom? What changes for the user after they use it?    -->
<!-- 2-3 sentences · active voice · second person · no backstory - start in the middle of the reader's world. -->

---

## Quick Start

<!-- SOLUTION beat: working example that gets the reader to a running system in < 5 minutes. -->

```bash
# 1 - clone and enter
git clone {repo_url}
cd {project_dir}

# 2 - copy env file and fill values
cp .env.example .env

# 3 - start services
docker compose up -d

# 4 - verify
curl http://localhost:{port}/health
```

Expected output:

```json
{"status": "ok", "version": "{semver}"}
```

---

## Architecture

<!-- Audience: Developer. 1–2 sentences + diagram link. Do not explain internals (Ousterhout: interface only). -->

The system is composed of {N} bounded modules. See the [Architecture Overview](docs/architecture/overview.md)
for C4 diagrams and decision rationale.

| Module | Role |
|---|---|
| `{module_1}/` | {one-line role} |
| `{module_2}/` | {one-line role} |

---

## Documentation

| What | Location |
|---|---|
| Getting started | [docs/getting-started/](docs/getting-started/) |
| API reference | [docs/api/reference.md](docs/api/reference.md) |
| Architecture | [docs/architecture/overview.md](docs/architecture/overview.md) |
| Deployment guide | [docs/guides/deployment.md](docs/guides/deployment.md) |
| Runbooks | [docs/runbooks/](docs/runbooks/) |
| Troubleshooting | [docs/troubleshooting.md](docs/troubleshooting.md) |
| Changelog | [CHANGELOG.md](CHANGELOG.md) |

---

## Configuration

All environment variables and config file options are documented in
[docs/getting-started/configuration.md](docs/getting-started/configuration.md).

Key variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `{VAR_1}` | Yes | (none) | {one-line description} |
| `{VAR_2}` | No | `{default}` | {one-line description} |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for branch naming, PR process, and code style requirements.

---

## License

{License name}. See [LICENSE](LICENSE) for full text.
