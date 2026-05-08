# guide_grafana_dashboards

> IoT Stream B — Grafana: dashboard design patterns, alerting best practices, dashboards-as-code.
> Related: `guide_mqtt_production.md` · `guide_influxdb_schema.md` · `guide_loki_logging.md`

---

## Table of Contents

1. [Observability Strategy (USE/RED/Four Golden Signals)](#1-observability-strategy-useredfour-golden-signals)
2. [Dashboard Design Principles](#2-dashboard-design-principles)
3. [Visualization Choices](#3-visualization-choices)
4. [Dashboard Management Maturity](#4-dashboard-management-maturity)
5. [Dashboards-as-Code (Grafonnet/Jsonnet)](#5-dashboards-as-code-grafonnetjsonnet)
6. [GitOps Provisioning](#6-gitops-provisioning)
7. [Grafana-Specific Tips](#7-grafana-specific-tips)
8. [Alerting Best Practices](#8-alerting-best-practices)
9. [Anti-Patterns](#9-anti-patterns)

---

## 1. Observability Strategy (USE/RED/Four Golden Signals)

| Method | Focus | Metrics |
|---|---|---|
| **USE** | Infrastructure (machines) | Utilization, Saturation, Errors |
| **RED** | Services / user experience | Rate, Errors, Duration |
| **Four Golden Signals** | User-facing systems | Latency, Traffic, Errors, Saturation |

**Rule**: Alert on symptoms (RED), diagnose with causes (USE). Map dashboards 1:1 to your chosen method.

---

## 2. Dashboard Design Principles

**Before creating any dashboard:**
- Define the goal: *"Which sensors are in trouble?"* — if no goal, skip the dashboard
- Design for **reduced cognitive load** — another engineer at 2 AM must understand it in seconds
- Follow a consistent observability strategy across all dashboards

**Mandatory practices:**
- Meaningful, self-describing names (no filler words like "monitoring" or "dashboard")
- Add a Text panel describing purpose, key links, and usage instructions
- Tag all dashboards by domain (`iot`, `infrastructure`, `business`)
- Cross-reference related dashboards via panel links or dashboard links
- Avoid unnecessary auto-refresh (hourly data → 5-min refresh is wasteful)

---

## 3. Visualization Choices

| Visualization | When to use |
|---|---|
| **Stat** (with traffic light colors) | High-level health-at-a-glance; green/blue/amber/red thresholds |
| **Time series** (graph) | Trends over time; set Y-min = 0 for non-negative metrics |
| **Heatmap** | Latency distribution (histogram percentiles: p50/p95/p99) |
| **Bar gauge** | Ranked comparison (top N offenders) |
| **Table** | Multi-column data; avoid for primary monitoring use |

**Stat panel pattern for IoT sensor health:**
```promql
sum by (sensor_id) (last_over_time(sensor_ok{site="plant-1"}[2m]))
```
Color mode: **Background** — whole screen green in healthy state.

**Y-axis rule:** Always set `Y-Min = 0` for metrics that can't go negative (temperature, packet count, error rate).

---

## 4. Dashboard Management Maturity

| Level | Characteristics |
|---|---|
| **Low** | Everyone edits dashboards; copies proliferate; no version control |
| **Medium** | Template variables prevent dashboard per-host; drill-downs; JSON in version control |
| **High** | Scripted dashboard generation (grafonnet/grafanalib); no browser edits; sprawl reviewed regularly |

**Target: High.** All dashboards generated from code, reviewed via PR, deployed via GitOps.

---

## 5. Dashboards-as-Code (Grafonnet/Jsonnet)

**Setup:**
```bash
# Install tools
brew install go-jsonnet jsonnet-bundler   # macOS
go install github.com/google/go-jsonnet/cmd/jsonnet@latest

# Initialize in repo
cd my-monitoring-repo
jb init
jb install https://github.com/grafana/grafonnet-lib/grafonnet
echo "/vendor" >> .gitignore
```

**Minimal dashboard (jsonnet):**
```jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';

grafana.dashboard.new(
  timezone='utc',
  title='IoT Plant Monitor (high-level)',
  uid='iot-plant-monitor',
)
.addPanel(
  grafana.stat.new(
    title='Sensor health — plant-1',
    datasource='influxdb',
    // ...
  ),
  gridPos={ x: 0, y: 0, w: 24, h: 8 },
)
```

**Environment distinction (dev/prod):**
```jsonnet
assert std.extVar('env') == 'dev' || std.extVar('env') == 'prod';
local cfg = { dev: { title_prefix: '[DEV] ' }, prod: { title_prefix: '' } }[std.extVar('env')];
```

Render: `jsonnet -J vendor --ext-str env=prod dashboards/iot.jsonnet`

---

## 6. GitOps Provisioning

**Workflow:**
1. Developer edits `.jsonnet` file
2. CI renders → `dashboard.json`
3. JSON committed to repo (or applied via `kubectl apply`)
4. Grafana reads provisioning directory (`updateIntervalSeconds: 10`)

**Grafana provisioning config (`dashboards.yaml`):**
```yaml
apiVersion: 1
providers:
  - name: iot
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/iot
```

**Rule:** Users may edit dashboards in the browser for incident investigation, but all
saved changes are overwritten on the next CI deploy. Treat production Grafana database as ephemeral.

---

## 7. Grafana-Specific Tips

| Tip | Implementation |
|---|---|
| UTC timezone everywhere | `grafana.dashboard.new(timezone='utc')` |
| Explicit data source | Never use `default`; always name it: `datasource='influxdb'` |
| Tooltip sort order | Set to `Decreasing` for error-rate panels |
| Link to logs/runbooks | `addLinks([{title: 'Logs', url: lokiExploreURL}])` |
| Annotate deployments | Add Loki or Prometheus annotation for deploy events |
| Grid: 12 or 24 columns | Full-width panels readable on small screens |

---

## 8. Alerting Best Practices

- **Alert on symptoms**, not infrastructure events: latency and error rate, not CPU spike
- **Pending period**: require condition to persist before firing (reduces flapping)
  ```promql
  avg_over_time(sensor_error_rate[5m]) > 0.05   # smooth over 5 min
  ```
- **Group related alerts** via notification policies — one page per root cause, not one per symptom
- **Every alert must have an owner** (team label) and scope (service/sensor group)
- **Link runbook + dashboard** in alert annotations
- **Remove alerts that never lead to action** — review quarterly

**Flapping mitigation:**
```promql
# Raw — reacts to transient spikes (avoid)
sensor_temp > 85

# Smoothed — requires sustained breach
avg_over_time(sensor_temp[5m]) > 85
```

**Graduate to SLOs:** If an alert fires more than weekly, define an error budget SLO instead.

---

## 9. Anti-Patterns

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Dashboard per sensor/host | Unmanageable sprawl | Use template variables (`$sensor_id`) |
| Manually created dashboards | Goes stale; can't review | Dashboards-as-code + GitOps |
| Copying dashboards to "customize" | Misses bug fixes; drift | URL parameters for view customization |
| Metric name per service variant | Can't plot multiple on one graph | Consistent naming + labels |
| `#` wildcard in MQTT + no labeled Stat panels | No health-at-a-glance | Stat + traffic light per sensor group |
| Alert on every metric | Alert fatigue → ignored | Alert only on user-visible symptoms |
