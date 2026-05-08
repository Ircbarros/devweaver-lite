# Docker Performance Optimization — Production Reference

> **Sources**: sematext.com/top-docker-metrics-to-watch · medium.com Docker performance optimization (Bilakanti · @rnab)
> **See also**: `references/containers/guide_dockerfile_best_practices.md` · `references/containers/guide_portainer_operations.md`

---

## 1. Performance Domains

| Domain | Key Levers |
|---|---|
| **Image size** | Multi-stage builds · distroless · `.dockerignore` · layer merging |
| **Resource limits** | `--memory` · `--cpus` · `--cpuset-cpus` · cgroup constraints |
| **Storage** | `overlay2` driver · named volumes > bind mounts · `tmpfs` for ephemeral I/O |
| **Networking** | Custom bridge networks · host mode (high perf) · DNS tuning |
| **Logging** | `json-file` driver with rotation limits · avoid unbounded log growth |
| **Daemon** | `overlay2` storage driver · `live-restore` enabled |

---

## 2. Image Optimisation

```dockerfile
# Combine RUN commands → fewer layers, smaller image
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# .dockerignore — exclude build artifacts from context
.git
*.md
node_modules/
__pycache__/
dist/
.env*
```

Multi-stage build reduces final image to runtime-only artefacts:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY src ./src
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --production
CMD ["node", "dist/app.js"]
```

---

## 3. Container Resource Management

### CPU and Memory Limits

```yaml
# docker-compose.yml (Compose v3 deploy syntax)
services:
  api:
    image: myapi:1.2.3
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.1"
          memory: 128M
```

CLI equivalent:
```bash
docker run --cpus="0.5" --memory="512m" myimage
```

### CPU Pinning (high-density hosts)
```bash
# Pin container to CPUs 0 and 1 only
docker run --cpuset-cpus="0,1" myimage
```

### Memory Tuning Workflow
1. Monitor baseline: `docker stats`
2. Set limit based on P99 usage + buffer
3. Watch `container_memory_failcnt` metric (indicates limit hits)
4. Watch for OOM events: `docker events --filter event=oom`

---

## 4. Key Metrics to Watch

### Host-Level

| Metric | Alert Threshold | Action |
|---|---|---|
| Host CPU utilisation | >85% sustained | Scale out; investigate throttled containers |
| Host memory usage | >90% | Review container reservations; add capacity |
| Host disk space | >80% | Run `docker system prune`; add volume capacity |
| Container count | Anomaly detected | Verify orchestration scheduling policy |

### Container-Level

| Metric | Meaning | Alert |
|---|---|---|
| `cpu_throttled_time` | Container hitting CPU limit | Increase `cpus` limit or optimise app |
| `memory_failcnt` | Container hitting memory limit | Increase `memory` limit; check for leaks |
| `memory_usage` | Raw memory consumption | Baseline and alert at > 80% of limit |
| `blkio_read/write_bps` | Disk I/O throughput | Throttle batch jobs; use `tmpfs` where possible |
| `net_rx/tx_bytes` | Network traffic | Detect unexpected spikes (DoS, client bugs) |

```bash
# Real-time container stats
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

---

## 5. Storage Optimisation

```yaml
# Named volumes — managed by Docker; portable across hosts
volumes:
  postgres_data: {}

services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data   # GOOD
      # NOT: - /opt/data:/var/lib/postgresql/data  # host-dependent

  # tmpfs for ephemeral (in-memory) I/O — no disk writes
  cache:
    image: redis:7-alpine
    tmpfs: [/data]
```

Disable swap for latency-sensitive workloads:
```bash
docker run --memory-swappiness=0 myimage
```

---

## 6. Networking Optimisation

```bash
# Host network mode — removes network namespace overhead (high-throughput use cases)
docker run --network host myimage

# DNS tuning — use faster nameserver
docker run --dns 8.8.8.8 myimage

# Or set daemon-wide in /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

**Warning**: host network mode removes container network isolation. Only use for services where network performance is critical and isolation is provided by other means (e.g., internal service, firewall rules).

---

## 7. Logging

```yaml
# Limit log size and file count per container
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "5"
    tag: "{{.Name}}/{{.ID}}"
```

Global daemon default (`/etc/docker/daemon.json`):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## 8. Docker Daemon Tuning

`/etc/docker/daemon.json`:
```json
{
  "storage-driver": "overlay2",
  "live-restore": true,
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Hard": 64000, "Soft": 64000 }
  }
}
```

| Setting | Benefit |
|---|---|
| `overlay2` | Best performance storage driver; copy-on-write |
| `live-restore: true` | Containers keep running if daemon restarts |
| `default-ulimits` | Prevent file descriptor exhaustion under load |

---

## 9. Monitoring Stack Reference

```yaml
# Minimal Prometheus + Grafana + cAdvisor stack
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports: ["8080:8080"]

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
```

Prometheus scrape config targets: `cadvisor:8080` for container metrics, `node_exporter:9100` for host metrics.

---

## 10. Disk Cleanup Commands

```bash
# Remove all stopped containers, unused networks, dangling images, build cache
docker system prune -f

# Also remove unused volumes (⚠ destructive — confirm no data loss)
docker system prune -f --volumes

# Remove only dangling images
docker image prune -f

# Check disk usage breakdown
docker system df -v
```

Schedule `docker system prune -f` as a weekly cron job on build/CI hosts.
