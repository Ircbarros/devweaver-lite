# Portainer Operations — Production Reference

> **Sources**: oneuptime.com Portainer best practices (environments · stacks · large-scale optimization) · portainer.io container + security best practices
> **See also**: `references/containers/guide_container_security.md` · `references/containers/guide_docker_performance.md`

---

## 1. Environment Organization

### Naming Conventions
```
# Pattern: {env}-{region}-{type}
prod-us-east-docker
staging-eu-west-docker
dev-local-docker

# Stacks: {team}-{app}-{env}
platform-webapp-prod
platform-api-staging
```

### RBAC Role Hierarchy
```yaml
viewer:
  - View containers and stacks
  - Access logs; no deployments

developer:
  - Deploy to dev/staging
  - Manage stacks in assigned environments

operator:
  - All developer permissions
  - Deploy to production (with approval workflow)
  - Manage volumes and networks

admin:
  - Full access; user management; environment configuration
```

**Enterprise tip**: Connect LDAP / Active Directory / OIDC for centralized authentication.

---

## 2. Stack Management (GitOps)

### Deploy from Git Repository
1. Stacks → Add Stack → **Repository**
2. Provide Git URL, branch, and path to compose file
3. Enable **Auto Update** for GitOps workflows

Benefits: version-controlled · peer-reviewed (PRs) · auditable (git history) · reproducible.

### Compose File Standards
```yaml
# BAD: hardcoded values
services:
  app:
    image: myapp:latest              # mutable tag
    environment:
      - DB_PASSWORD=mysecretpassword # secret in compose file

# GOOD: GitOps-ready compose file
services:
  app:
    image: myapp:${APP_VERSION:-1.0.0}   # pinned; overridable
    environment:
      - DB_PASSWORD=${DB_PASSWORD}        # injected by Portainer env vars
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
        reservations: { cpus: "0.25", memory: 128M }
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "100m", max-file: "5" }
```

### Image Pinning
```yaml
# Avoid latest
image: nginx:latest          # BAD

# Pin minor version
image: nginx:1.25.4          # GOOD

# Pin by digest (fully reproducible)
image: nginx@sha256:3b4b...  # BEST for production
```

### Stack Naming Convention
```
<app>-<tier>
wordpress-production
api-staging
monitoring-shared
```

---

## 3. Volume and Network Hygiene

```yaml
# Prefer named volumes over bind mounts
volumes:
  db-data: {}

services:
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data  # named volume — portable
      # NOT: - /opt/myapp/db:/var/lib/postgresql/data  # host-dependent
```

---

## 4. Portainer Instance Security

```yaml
services:
  portainer:
    image: portainer/portainer-ee:latest
    command:
      - --ssl
      - --sslcert=/certs/portainer.crt
      - --sslkey=/certs/portainer.key
      - --admin-password-file=/run/secrets/portainer-password
    secrets: [portainer-password]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
      - portainer-certs:/certs
```

Key Portainer security features:
- **RBAC**: granular per-environment permissions
- **Audit log**: `Admin → Logs → Activity` — who did what, when, where
- **Image policy**: restrict deployments to internal registry only
- **OPA Gatekeeper**: block privileged containers at admission

---

## 5. Large-Scale Optimisation

### Performance Bottleneck Reference

| Bottleneck | Symptom | Fix |
|---|---|---|
| Snapshot interval too short | High CPU, slow UI | `--snapshot-interval 10m` (default 5m) |
| BoltDB database growth | Slow page loads, high memory | Restart with `--compact-db` periodically |
| High API polling load | High API server load | Rate-limit via nginx; use least-privilege tokens |
| Slow `/data` volume | Slow DB operations | Use SSD-backed storage for `portainer_data` |
| Insufficient resources | Portainer OOM killed | Increase CPU/memory limits |

### Snapshot Tuning
```bash
# 100+ containers / 20+ stacks → start at 15m
docker run -d portainer/portainer-ce:latest --snapshot-interval 15m
```

### BoltDB Compaction
```bash
docker stop portainer && docker rm portainer
docker run -d --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -p 9000:9000 \
  portainer/portainer-ce:latest \
  --compact-db --snapshot-interval 10m
```

### Resource Limits for Portainer (Swarm)
```yaml
deploy:
  resources:
    limits: { cpus: "2.0", memory: 1G }
    reservations: { cpus: "0.5", memory: 256M }
```

### API Rate Limiting (nginx)
```nginx
http {
  limit_req_zone $binary_remote_addr zone=portainer_api:10m rate=10r/s;
  server {
    location /api/ {
      limit_req zone=portainer_api burst=20 nodelay;
      proxy_pass http://portainer:9000;
    }
  }
}
```

### SSD-Backed Storage
```bash
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/ssd/portainer \
  --opt o=bind \
  portainer_ssd_data
```

---

## 6. Periodic Audit Script

```bash
#!/bin/bash
# Run monthly
echo "--- Dangling volumes ---"
docker volume ls -qf dangling=true

echo "--- Stopped containers ---"
docker ps -a --filter status=exited

echo "--- Unused images ---"
docker images -qf dangling=true

echo "--- Disk usage ---"
docker system df -v
```

---

## 7. Multi-Environment Topology

**Single Portainer Server + Agents**:
- Portainer Server: dedicated management node
- Portainer Agent: installed on each Docker host
- Portainer Edge Agent: for air-gapped or remote environments
- Portainer does **not** support multiple server instances against the same cluster — scale vertically

**Monitoring Portainer itself**:
```bash
docker stats portainer --no-stream   # quick check
# Full monitoring: deploy cAdvisor + Prometheus + Grafana
```
