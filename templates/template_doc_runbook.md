---
component: "{component_name}"
version: "{semver}"
status: "DRAFT"
audience: "Operator"
last_updated: "{ISO-8601}"
related_issues: []
---

# {Component} Operations Runbook

> **Scope**: Day-to-day operational procedures and incident response for `{component}`.
> For architecture context see [docs/architecture/overview.md](../architecture/overview.md).

---

## §1 Service Overview

<!-- NEED beat: what does this component do and why does an operator care about keeping it healthy? -->

| Property | Value |
|---|---|
| Service name | `{service_name}` |
| Port / endpoint | `{port}` |
| Health check | `GET http://localhost:{port}/health` |
| Logs | `structlog` → Loki · label: `service="{service_name}"` |
| Metrics dashboard | `{grafana_dashboard_name}` |
| On-call contact | `{team_or_channel}` |

---

## §2 Start / Stop / Restart

```bash
# Start
docker compose up -d {service_name}

# Graceful stop
docker compose stop {service_name}

# Restart
docker compose restart {service_name}

# Status
docker compose ps {service_name}

# Live logs
docker compose logs -f {service_name}
```

---

## §3 Common Incidents

<!-- Per incident: Need → Obstacle → Solution arc (rule_documentation_standards.md §4.2).         -->
<!-- Start in action (The Moth): open with the symptom the operator already sees, not with theory. -->

### {Incident Title}

**Symptom**: {what the operator observes - exact log line, metric threshold, alert name}
**Cause**: {why this happens - concise, no implementation internals}
**Resolution**:

```bash
{resolution_command}
```

**Expected outcome**: {what is true after the fix. Example: "Health check returns 200 within 30 s"}
**If unresolved**: {escalation path - linked IssueRecord or on-call contact}

---

## §4 Scaling

<!-- OBSTACLE: capacity limit · SOLUTION: how to expand (be specific - Goldberg) -->

| Dimension | Current limit | Scale action |
|---|---|---|
| `{dimension_1}` | `{current_value}` | `{exact_command_or_config_change}` |
| `{dimension_2}` | `{current_value}` | `{exact_command_or_config_change}` |

---

## §5 Backup & Recovery

| Item | Storage location | Recovery command |
|---|---|---|
| `{data_item_1}` | `{path_or_service}` | `{command}` |
| `{data_item_2}` | `{path_or_service}` | `{command}` |

---

## §6 Monitoring Checklist

Daily / on-deploy verification (operators tick these - do not describe, just check):

- [ ] Health endpoint returns `{"status": "ok"}`
- [ ] No ERROR-level log entries in the last 24 h
- [ ] Grafana dashboard `{dashboard_name}` - no sustained spikes
- [ ] Disk usage < 80 % on `{volume_or_path}`
- [ ] Pending DLQ entries = 0

---

## §7 See Also

- [CHANGELOG.md](../../CHANGELOG.md)
- [docs/guides/deployment.md](../guides/deployment.md)
- [docs/troubleshooting.md](../troubleshooting.md)
- [docs/architecture/overview.md](../architecture/overview.md)
