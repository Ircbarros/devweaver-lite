# guide_loki_logging

> IoT Stream B — Grafana Loki: label design, configuration best practices, logging at scale.
> Related: `guide_grafana_dashboards.md` · `guide_mqtt_production.md` · `guide_influxdb_schema.md`

---

## Table of Contents

1. [Loki Data Model](#1-loki-data-model)
2. [Label Best Practices](#2-label-best-practices)
3. [Dynamic Labels — Use Sparingly](#3-dynamic-labels--use-sparingly)
4. [Label Cardinality Rules](#4-label-cardinality-rules)
5. [Time Ordering of Logs](#5-time-ordering-of-logs)
6. [Compression Configuration](#6-compression-configuration)
7. [Chunk Tuning](#7-chunk-tuning)
8. [Production Limits Configuration](#8-production-limits-configuration)
9. [Ingester Configuration](#9-ingester-configuration)
10. [Query Patterns](#10-query-patterns)

---

## 1. Loki Data Model

Loki stores logs as **streams**: a set of key=value label pairs + a sequence of log lines ordered by timestamp.

```
Stream: {app="sensor-api", env="prod", region="eu-west"}
  2024-01-01T10:00:00Z  MQTT message received: topic=dt/plant-1/sensor-001/temp
  2024-01-01T10:00:01Z  Wrote 42 points to InfluxDB bucket=raw_telemetry
```

Labels define streams. More unique label combinations = more streams = larger index.

---

## 2. Label Best Practices

**Static labels are good** — use for bounded, infrastructure-level attributes:

| Good static labels | Examples |
|---|---|
| Region / cluster | `region="eu-west"`, `cluster="prod-k8s"` |
| Application / service | `app="mqtt-ingestor"`, `service="telemetry-api"` |
| Namespace / environment | `namespace="iot"`, `env="prod"` |
| Host/instance | `host="worker-01"` (bounded set) |

These labels enable fast stream selection: `{app="mqtt-ingestor", env="prod"}`.

**Clients:** Grafana Alloy, Fluentd, Fluent Bit, Docker plugin — review what labels each adds dynamically. Audit with `logcli series --analyze-labels`.

---

## 3. Dynamic Labels — Use Sparingly

Adding a label dynamically splits one stream into N streams. Prefer **filter expressions** over extra labels.

```logql
# Instead of label level="error" on every line, use filter expression:
{app="mqtt-ingestor"} |= "level=error"

# This is equally fast for moderate-volume apps and avoids 5x stream multiplication
```

**When dynamic labels ARE justified:**
- Label cardinality is provably low (tens of values)
- Values are long-lived: `/load`, `/save`, `/update` (not trace IDs)
- The label is used in the vast majority of queries
- App volume is high enough that chunks fill before `max_chunk_age`

**Never extract as labels:**
- Trace IDs, request IDs, order IDs, session IDs → unbounded cardinality
- Raw timestamp values → unique per line
- User IDs or email addresses → PII + unbounded

---

## 4. Label Cardinality Rules

| Metric | Target |
|---|---|
| Active streams per tenant | < 100,000 |
| Streams in a 24-hour window | < 1,000,000 |
| Values per dynamic label | Single digits to tens |

**Audit high-cardinality labels:**
```bash
logcli series --analyze-labels '{app="mqtt-ingestor"}'
# Look for labels with Unique Values close to Found In Streams — those are label anti-patterns
```

**Fix:** Remove the high-cardinality label; filter in LogQL instead:
```logql
{app="mqtt-ingestor"} | logfmt | requestId="32422355"
```

---

## 5. Time Ordering of Logs

Loki requires **strictly increasing timestamps per stream**.

```bash
# This will be REJECTED (out of order within same stream):
{job="syslog"} 00:00:02 message
{job="syslog"} 00:00:01 message  ← REJECTED

# Fix: add instance label to create separate streams per source
{job="syslog", instance="host1"} 00:00:02 message
{job="syslog", instance="host2"} 00:00:01 message  ← ACCEPTED (separate stream)
```

**Options when app emits out-of-order logs:**
- Add `instance` label to isolate sources
- Let the shipping agent assign ingestion timestamp (not extracted from log line)
- Enable `unordered_writes: true` in Loki config (available since Loki 2.4)

---

## 6. Compression Configuration

| Algorithm | Speed | Compression ratio | Recommendation |
|---|---|---|---|
| `snappy` | Very fast | Lower | Default; best for query speed |
| `lz4` | Fast | Medium | Good balance for storage-constrained |
| `gzip` | Slow | High | Avoid — causes slow queries |

```yaml
ingester:
  chunk_encoding: snappy   # recommended default
```

---

## 7. Chunk Tuning

Target compressed chunk size of **1.5 MB** (`chunk_target_size: 1572864`).

Rules:
- Small chunks = too many chunk open/close ops + overhead on reads
- `max_chunk_age: 2h` — flush chunk after 2 hours even if not full
- `chunk_idle_period: 30m` — flush if no new data for 30 minutes
- A dynamic label is justified only if the app can fill a 1.5 MB chunk faster than `max_chunk_age`

**Avoid stream fragmentation:** Splitting one log source into too many streams results
in many half-full chunks constantly flushed by `chunk_idle_period`.

---

## 8. Production Limits Configuration

```yaml
limits_config:
  # Rate limits
  ingestion_rate_strategy: global
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  per_stream_rate_limit: 3MB
  per_stream_rate_limit_burst: 15MB

  # Stream limits
  max_global_streams_per_user: 10000
  max_streams_per_user: 0             # 0 = enforce global limit only

  # Validation
  max_line_size: 256KB
  max_line_size_truncate: false
  max_label_name_length: 1024
  max_label_value_length: 2048
  max_label_names_per_series: 15

  # Time constraints
  reject_old_samples: true
  reject_old_samples_max_age: 168h    # 7 days
  creation_grace_period: 10m
  unordered_writes: true
```

---

## 9. Ingester Configuration

```yaml
ingester:
  chunk_idle_period: 30m
  chunk_target_size: 1572864           # 1.5 MB
  chunk_encoding: snappy
  max_chunk_age: 2h

  wal:
    enabled: true
    dir: /loki/wal
    checkpoint_duration: 5m
    flush_on_shutdown: true            # prevent data loss on restart
```

**Authentication:** Loki has no built-in auth — always run an authenticating
reverse proxy (nginx) in front.

---

## 10. Query Patterns

**Prefer filter expressions over label extraction:**
```logql
# Efficient: stream selection + text filter
{app="mqtt-ingestor", env="prod"} |= "MQTT message received"

# Metric from logs: count ingestion errors per minute
sum(count_over_time({app="mqtt-ingestor"} |= "ERROR" [1m]))
```

**Structured metadata for high-cardinality values:**
Instead of extracting `trace_id` as a label (bad), attach it as structured metadata
at the client side (Alloy) — queryable but not indexed.

**Debugging stream cardinality:**
```bash
logcli series '{app="mqtt-ingestor"}' | wc -l
logcli series '{app="mqtt-ingestor"}' --analyze-labels
```
