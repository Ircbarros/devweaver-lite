# Container Security — Production Reference

> **Sources**: OWASP Docker Security Cheat Sheet · portainer.io/blog/container-security-best-practices · sysdig.com Dockerfile best practices (security sections)
> **See also**: `references/containers/guide_dockerfile_best_practices.md` · `references/containers/guide_portainer_operations.md`

---

## 1. OWASP Docker Security Rules (R-00 → R-13)

| Rule | Requirement | Implementation |
|---|---|---|
| R-00 | Keep host + Docker up to date | Patch kernel; `docker engine upgrade` regularly |
| R-01 | Never expose Docker socket | No `-v /var/run/docker.sock` to containers; no TCP daemon |
| R-02 | Run as non-root user | `USER` in Dockerfile or `--user 4000` at runtime |
| R-03 | Drop all capabilities, add only required | `--cap-drop all --cap-add CHOWN` |
| R-04 | Prevent privilege escalation | `--security-opt=no-new-privileges` |
| R-05 | Network segmentation by default | Custom networks; deny-by-default; avoid flat topology |
| R-05a | Firewall port binding | Bind published ports to localhost: `-p 127.0.0.1:8000:8000` |
| R-06 | Linux Security Modules | Enable seccomp + AppArmor profiles; don't disable defaults |
| R-07 | Limit resources | `--memory`, `--cpus`, `--ulimit nofile`, `--restart=on-failure:5` |
| R-08 | Read-only filesystem | `--read-only --tmpfs /tmp` for ephemeral writes |
| R-09 | CI/CD scanning | Trivy · Grype · Docker Scout in every pipeline |
| R-10 | Daemon log level | Keep at `info`; never left at `debug` in production |
| R-11 | Rootless Docker mode | Run daemon as non-root where isolation is critical |
| R-12 | Docker Secrets | `docker secret create` / Compose `secrets:` for passwords |
| R-13 | Supply chain security | SBOM generation; image signing (Notary/Sigstore); trusted registry |

---

## 2. Runtime Hardening Examples

```bash
# Minimal privilege container
docker run \
  --cap-drop all \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --security-opt seccomp=/etc/docker/seccomp.json \
  --read-only \
  --tmpfs /tmp \
  --user 1000:1000 \
  --memory 512m \
  --cpus 0.5 \
  --restart on-failure:3 \
  myimage:tag
```

Docker Compose equivalent:
```yaml
services:
  api:
    image: myimage:1.2.3
    read_only: true
    tmpfs: [/tmp]
    security_opt:
      - no-new-privileges:true
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    user: "1000:1000"
    deploy:
      resources:
        limits: { cpus: "0.5", memory: 512M }
        reservations: { cpus: "0.1", memory: 128M }
    restart: unless-stopped
```

---

## 3. Secrets Management

| Method | Use Case | Notes |
|---|---|---|
| Docker Secrets (Swarm) | Service passwords, tokens | Mounted as `/run/secrets/<name>` |
| Compose `secrets:` block | Development / single-host | File-based; do not commit to git |
| Environment injection | CI/CD pipelines | `--env` at runtime; never in Dockerfile |
| External vault | Production | HashiCorp Vault · AWS Secrets Manager · Doppler |

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt
services:
  app:
    secrets: [db_password]
```

**Never**: `ENV DB_PASSWORD=secret` in Dockerfile — exposed in all image layers.

---

## 4. Network Security

```yaml
# Deny-by-default custom network isolation
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true        # no external access

services:
  nginx:
    networks: [frontend, backend]
  api:
    networks: [backend]
  db:
    networks: [backend]   # only api can reach db
```

UFW firewall bypass — always bind published ports to localhost:
```yaml
ports:
  - "127.0.0.1:8080:8080"  # safe
  # NOT "8080:8080"        # bypasses UFW; exposed to all interfaces
```

---

## 5. Image Supply Chain

```bash
# Enable Docker Content Trust (image signing)
export DOCKER_CONTENT_TRUST=1

# Generate SBOM
docker sbom myimage:tag

# Sign with cosign (Sigstore)
cosign sign --key cosign.key myregistry/myimage:tag

# Verify before pull
cosign verify --key cosign.pub myregistry/myimage:tag
```

**Registry controls**:
- Enforce image scanning policy at admission (Kubernetes: OPA Gatekeeper / Kyverno)
- Block images from unapproved registries
- Require images to pass vulnerability scan before production deploy

---

## 6. Runtime Monitoring

| Tool | Purpose | Command |
|---|---|---|
| Falco | Syscall-level anomaly detection | Alerts on unexpected exec, network connections |
| Tetragon | eBPF-based runtime security | Fine-grained process + file monitoring |
| `docker events` | Docker daemon event stream | `docker events --filter type=container` |
| cAdvisor | Container resource metrics | Exposes `/metrics` for Prometheus scrape |

---

## 7. Container Security Maturity Model

| Level | Practices |
|---|---|
| **Low** | Non-root user · no privileged containers · image scanning in CI |
| **Medium** | + Read-only filesystem · resource limits · network segmentation · Docker Secrets |
| **High** | + Rootless daemon · seccomp/AppArmor profiles · SBOM + signing · runtime monitoring (Falco) · supply chain attestation |

---

## 8. Key Anti-Patterns

| Anti-Pattern | Risk |
|---|---|
| `--privileged` flag | Full host access — attacker owns the host |
| `-v /:/host` bind mount | Root filesystem exposure |
| `-v /var/run/docker.sock` | Docker-in-Docker privilege escalation |
| Hardcoded secrets in env vars | Leaks in `docker inspect`, logs, manifests |
| `latest` tags in production | Undetermined image; supply chain risk |
| Containers running as UID 0 | Any exploit → root privilege on host |
| Flat overlay network | Lateral movement across all services |
