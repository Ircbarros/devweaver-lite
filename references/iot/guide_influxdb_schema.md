# guide_influxdb_schema

> IoT Stream B — InfluxDB v2/v3 schema design: data model, cardinality, query performance.
> Related: `guide_mqtt_influxdb.md` · `guide_mqtt_production.md` · `guide_grafana_dashboards.md`

---

## Table of Contents

1. [Data Model Overview](#1-data-model-overview)
2. [Tags vs Fields](#2-tags-vs-fields)
3. [Schema Restrictions](#3-schema-restrictions)
4. [Avoid Wide Schemas](#4-avoid-wide-schemas)
5. [Avoid Sparse Schemas](#5-avoid-sparse-schemas)
6. [Design for Query Simplicity](#6-design-for-query-simplicity)
7. [Series Cardinality Control](#7-series-cardinality-control)
8. [Bucket & Retention Strategy](#8-bucket--retention-strategy)
9. [Downsampling Pattern](#9-downsampling-pattern)
10. [Common Schema Mistakes](#10-common-schema-mistakes)

---

## 1. Data Model Overview

```
Organization
└── Bucket (≈ database; has retention policy)
    └── Measurement (≈ table)
        ├── Tags     → indexed, string-only, metadata
        ├── Fields   → measured values (int/float/string/bool)
        └── Timestamp → nanosecond Unix UTC; never null
```

**Primary key** = `timestamp` + all `tag key=value` pairs on the row.

---

## 2. Tags vs Fields

| Use Tags for | Use Fields for |
|---|---|
| Metadata / identifying info (`host`, `region`, `sensor_id`) | Measured values (`temperature`, `pressure`, `voltage`) |
| Things you **filter by** in WHERE clauses | Things you **aggregate** (SUM, AVG, MAX) |
| Low-cardinality, bounded values | High-cardinality or unbounded values |
| String values only | int / float / string / bool / uint |

> **InfluxDB 3 (Cloud Serverless)**: infinite tag value cardinality is supported.
> **InfluxDB 2 (TSM)**: high tag cardinality causes performance degradation — keep below practical limits.

---

## 3. Schema Restrictions

- **No duplicate names** for tags and fields within the same measurement (column conflict)
- **Max columns per measurement** is configurable; if exceeded, writes fail
- Every row must have a `time` column; all other columns count toward the limit
- Avoid reserved keywords and special characters in measurement/tag/field names:
  - Requires double-quoting in SQL/InfluxQL → reduces query readability

---

## 4. Avoid Wide Schemas

A **wide schema** has too many columns (tags + fields) in one measurement.

**Problems:**
- Increased resource usage during ingestion and compaction
- Complex primary keys with many tags → slower sort performance
- Slower queries when selecting many columns

**Solution:** Split unrelated fields into separate measurements with their own retention.

---

## 5. Avoid Sparse Schemas

A **sparse schema** has many null values per row.

**Causes:**
- Non-homogenous measurements (rows have different tag/field sets)
- Fields written with different timestamps → creates two rows each with one null

**Fix:** Write all fields for a row at the same timestamp. Homogenous schemas — every row has the same tag and field keys.

---

## 6. Design for Query Simplicity

**Rule: one tag per data attribute** — split concatenated values into separate tags.

```
# Bad — embeds multiple attributes into one tag value
home,sensor=loc-kitchen.model-A612.id-1726ZA temp=72.1

# Good — each attribute is its own tag
home,location=kitchen,sensor_model=A612,sensor_id=1726ZA temp=72.1
```

The bad form requires regex queries (`LIKE '%id-1726ZA%'`); the good form allows
simple equality (`WHERE sensor_id = '1726ZA'`) — faster and less error-prone.

---

## 7. Series Cardinality Control

**Series cardinality** = unique combinations of `bucket + measurement + tag set + field keys`.

**Runaway cardinality causes:**
- User IDs / email addresses / timestamps / UUIDs stored as tags
- Log messages stored as tags
- Event/order/request IDs as tags

**Rules:**
- Store unbounded identifiers (user_id, request_id) as **fields**, not tags
- Keep each dynamic tag to **single-digit or tens of values** (dozens max for TSM engine)
- Target < 100,000 active streams per tenant in large deployments

---

## 8. Bucket & Retention Strategy

Separate data into distinct buckets when:
- Different retention policies are needed (e.g., raw sensor data → 2 weeks; downsampled → 1 year)
- Different data frequencies (high-frequency telemetry vs. static device metadata)
- Token-scoped access is required per data category

**IoT example:**
```
Bucket: raw_telemetry    (retention: 14 days)  → temperature, pressure at 10s interval
Bucket: hourly_agg       (retention: 1 year)   → hourly min/max/avg aggregates
Bucket: device_registry  (retention: infinite) → device metadata, firmware versions
```

---

## 9. Downsampling Pattern

Downsampling aggregates high-precision data into lower-precision summaries.
Run as a Task (InfluxDB scheduled query):

```flux
from(bucket: "raw_telemetry")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "temperature")
  |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
  |> to(bucket: "hourly_agg")
```

Use downsampling + retention together:
1. Downsample → write to `hourly_agg`
2. Set 14-day retention on `raw_telemetry` to auto-drop old precision data

---

## 10. Common Schema Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Log messages as tags | Unbounded cardinality (UUIDs, timestamps in logs) | Store log attributes (level, service) as tags; full log line as field |
| IDs (userId, orderId) as tags | Unbounded cardinality explosion | Move to fields |
| Too many measurements (one per metric) | InfluxDB is not a key-value store | Group related metrics in one measurement with tags for context |
| Same data in multiple buckets | Wasted storage, inconsistent queries | Design one canonical bucket per data category |
| Mixing unrelated fields in one measurement | Forces joins; sparse rows when cadences differ | Split by write cadence and semantic relationship |
