# Dockerfile Best Practices — Production Reference

> **Sources**: docs.docker.com/build/building/best-practices · docker.com/blog/speed-up-your-development-flow · sysdig.com/learn-cloud-native/dockerfile-best-practices
> **See also**: `references/containers/guide_container_security.md` · `references/containers/guide_docker_performance.md`

---

## 1. Image Selection

| Decision | Rule |
|---|---|
| Base image source | Use Docker Official Images or Verified Publisher images only |
| Size | Prefer `alpine` (<6 MB) or `distroless` over full distros |
| Two-image pattern | Use a build image (compiler + tools) + slim production image |
| Build stage base | `golang:1.x`, `node:X.Y.Z`, `maven:X-jdk-Y` |
| Production base | `gcr.io/distroless/static`, `alpine:X.Y.Z`, `*-slim` variants |

Pin versions with digest for full supply-chain integrity:
```dockerfile
FROM alpine:3.21@sha256:a8560b3...
```

---

## 2. Multi-Stage Builds

```dockerfile
# Stage 1 — builder
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Stage 2 — production (distroless = no shell, no package manager)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**Key rules**:
- Dependencies (e.g. `go.mod`, `package.json`) before source code — cache-stable layer first
- Only copy final artifacts into the production stage
- Build tools, source code, intermediate files never reach prod image

---

## 3. Layer Cache Optimisation

```dockerfile
# BAD — invalidates cache on every source change
COPY . /code
RUN npm ci

# GOOD — npm ci only re-runs when package.json changes
COPY package.json package-lock.json /code/
RUN npm ci
COPY src /code/src
```

Order instructions from least-changing → most-changing:
1. Base image (`FROM`)
2. System packages (`RUN apt-get` / `apk add`)
3. Dependency manifests + install (`COPY package*.json` → `RUN npm ci`)
4. Application source (`COPY src`)

---

## 4. Instruction Reference

```dockerfile
FROM alpine:3.21              # always pin; avoid 'latest'
LABEL org.opencontainers.image.version="1.2.3"

# Combine RUN → single layer; clean package cache in same step
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app                  # absolute paths only; never 'cd' in RUN
COPY --chown=app:app . .      # use COPY not ADD unless extracting remote tar
ENV APP_PORT=8080             # use RUN to unset secrets within same layer
EXPOSE 8080                   # documentation only; enforce at runtime
USER app                      # always run as non-root
ENTRYPOINT ["/app/server"]    # exec form: PID 1 receives signals
CMD ["--config=/etc/app.yaml"]# default flags; overridable at runtime
```

---

## 5. Non-Root User Pattern

```dockerfile
RUN groupadd -r appuser && useradd --no-log-init -r -g appuser appuser
# Set permissions before switching
RUN chown -R appuser:appuser /app
USER appuser
```

Distroless images include `nonroot` user (UID 65532):
```dockerfile
USER nonroot:nonroot
```

---

## 6. .dockerignore

Exclude everything not required for the build:
```
.git
.gitignore
*.md
*.log
node_modules/
__pycache__/
.env*
tests/
dist/
```

**Why**: entire build context is sent to daemon before build starts; smaller context = faster builds and no accidental secret copying.

---

## 7. Build Flags for CI

```bash
# Always pull latest base image tag; disable build cache for scheduled CI builds
docker build --pull --no-cache -t myimage:${GIT_SHA} .

# Build specific stage (dev/testing)
docker build --target development -t myimage:dev .
```

---

## 8. HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Kubernetes: use `livenessProbe` + `readinessProbe` in Pod spec instead.

---

## 9. Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `FROM ubuntu:latest` | Tag mutable; unexpected CVEs | Pin to `ubuntu:24.04@sha256:...` |
| `COPY . /app` | Copies secrets, .git, node_modules | Use `.dockerignore` + explicit paths |
| `RUN apt-get update` alone | Cache bust breaks subsequent install | Always chain with `apt-get install` |
| `ENV SECRET=password` | Leaks into layers | Inject at runtime via `--env` or Docker Secrets |
| `ADD url tar.gz` | Less predictable than `RUN curl` | Use `RUN curl` + checksum verify |
| Root user in production | Privilege escalation risk | `USER nonroot` or `USER 65532` |
| Tool `sudo` in image | Unpredictable TTY/signal behaviour | Use `gosu` for privilege drop |

---

## 10. Linting and Scanning

```bash
# Lint Dockerfile locally
hadolint Dockerfile

# Scan image for CVEs (free)
trivy image myimage:tag
grype myimage:tag

# Docker Scout (integrated with docker CLI)
docker scout cves myimage:tag
```

Integrate `hadolint` + `trivy` in CI before push to registry.
