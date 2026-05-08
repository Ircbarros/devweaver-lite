# guide_cicd_patterns

> Stream D — GitHub Actions CI/CD, conventional commits, Docker best practices, ntfy notifications.

---

## Table of Contents

1. [GitHub Actions Workflow Structure](#1-github-actions-workflow-structure)
2. [Conventional Commits & Semantic Versioning](#2-conventional-commits--semantic-versioning)
3. [Docker Multi-Stage Builds](#3-docker-multi-stage-builds)
4. [Dependency Caching](#4-dependency-caching)
5. [Pinned Action Versions](#5-pinned-action-versions)
6. [ntfy Notification Patterns](#6-ntfy-notification-patterns)
7. [Stable Versions](#7-stable-versions)

---

## 1. GitHub Actions Workflow Structure

**Pipeline order**: `lint → test → build → scan → sign → push → notify`

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install ruff mypy
      - run: ruff check src/ && mypy src/

  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[dev]"
      - run: pytest --cov=src --cov-fail-under=80 tests/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          push: false
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: pip install bandit pip-audit cyclonedx-bom
      - run: bandit -r src/ -c pyproject.toml
      - run: pip-audit -r requirements.txt
      - run: trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE

  push:
    needs: scan
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - uses: docker/build-push-action@v6
        with: { push: true, tags: "ghcr.io/${{ github.repository }}:${{ github.sha }}" }
      - run: cosign sign --yes ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Coverage gate**: `--cov-fail-under=80` blocks merge below 80% line coverage.

---

## 2. Conventional Commits & Semantic Versioning

**Format**: `<type>(<scope>): <description>`  
**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`

```bash
git commit -m "feat(ai): add hybrid Qdrant search to RAG pipeline"
git commit -m "fix(iot): handle MQTT reconnect backoff correctly"
git commit -m "chore(deps): upgrade langgraph to 0.2.55"
```

**commitlint** config (`.commitlintrc.yaml`):
```yaml
extends: ["@commitlint/config-conventional"]
rules:
  scope-empty: [2, "never"]  # Require scope
  subject-max-length: [2, "always", 72]
```

**release-please** (automated versioning):
```yaml
# .github/workflows/release-please.yml
- uses: googleapis/release-please-action@v4
  with: { release-type: python }
```

---

## 3. Docker Multi-Stage Builds

```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install uv
COPY requirements.txt .
RUN uv pip install --system -r requirements.txt

# Production stage (distroless — no shell, no package manager)
FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY src/ ./src/
ENTRYPOINT ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0"]
```

**Rules**:
- Final stage: distroless or `python:3.12-slim` minimum (no `python:3.12`).
- No `COPY . .` in production stage; only copy required files.
- Use `uv` for faster pip installs in CI.

---

## 4. Dependency Caching

```yaml
# Python (uv)
- uses: actions/setup-python@v5
  with: { python-version: "3.12" }
- uses: astral-sh/setup-uv@v5
- run: uv pip install --system -r requirements.txt

# Node.js (pnpm)
- uses: pnpm/action-setup@v4
  with: { version: "9" }
- uses: actions/setup-node@v4
  with: { node-version: "20", cache: "pnpm" }
- run: pnpm install --frozen-lockfile
```

---

## 5. Pinned Action Versions

| Action | Pinned Version | Use |
|---|---|---|
| `actions/checkout` | `v4` | All jobs |
| `actions/setup-python` | `v5` | Python jobs |
| `actions/setup-node` | `v4` | Node jobs |
| `docker/build-push-action` | `v6` | Docker builds |
| `docker/login-action` | `v3` | Registry login |
| `docker/setup-buildx-action` | `v3` | BuildKit |
| `sigstore/cosign-installer` | `v3` | Image signing |
| `googleapis/release-please-action` | `v4` | Semantic versioning |
| `pnpm/action-setup` | `v4` | Node package manager |
| `astral-sh/setup-uv` | `v5` | Fast Python installs |

---

## 6. ntfy Notification Patterns

**Self-hosted Docker Compose**:
```yaml
ntfy:
  image: binwiederhier/ntfy:v2.11.0
  command: serve
  environment:
    NTFY_AUTH_FILE: /var/lib/ntfy/user.db
    NTFY_CACHE_FILE: /var/cache/ntfy/cache.db
  ports: ["127.0.0.1:2586:80"]  # Do NOT expose to internet
  volumes: [ntfy-data:/var/lib/ntfy, ntfy-cache:/var/cache/ntfy]
```

**Python send pattern**:
```python
import httpx, os

async def notify(title: str, message: str, priority: str = "default") -> None:
    """Send ntfy push notification. Never expose URL or token in logs."""
    await httpx.AsyncClient().post(
        f"{os.getenv('NTFY_URL')}/{os.getenv('NTFY_TOPIC')}",
        headers={
            "Authorization": f"Bearer {os.getenv('NTFY_TOKEN')}",
            "Title": title,
            "Priority": priority,  # min|low|default|high|urgent
        },
        content=message[:4000],  # 4 KB payload limit
    )
```

**Event categories**: `urgent` = service crash; `high` = MQTT reconnect failure; `default` = CI/CD failure; `low` = info.

---

## 7. Stable Versions

| Tool | Version | Notes |
|---|---|---|
| ntfy | `2.11.x` | ✅ Stable |
| `commitlint` | `19.x` | ✅ Stable |
| `release-please` | `16.x` | ✅ Stable |
| `uv` (Python PM) | `0.4.x` | ✅ Stable |
| `pnpm` | `9.x` | ✅ Stable |
