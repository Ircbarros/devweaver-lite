# iot domain index

IoT references: MQTT v5, InfluxDB pipeline, schema design, Grafana, Loki logging.
Use this index when the task involves IoT data ingestion, time-series storage, or monitoring.

| ISBN | File | Topics | Summary |
|---|---|---|---|
| iot-001 | references/iot/guide_mqtt_production.md | MQTT, production, v5, QoS, patterns | MQTT v5 production patterns |
| iot-002 | references/iot/guide_mqtt_influxdb.md | MQTT, InfluxDB, pipeline, ingestion, bridge | MQTT to InfluxDB ingestion pipeline |
| iot-003 | references/iot/guide_influxdb_schema.md | InfluxDB, schema, v3, measurements, tags | InfluxDB v3 schema design |
| iot-004 | references/iot/guide_grafana_dashboards.md | Grafana, dashboards, panels, Flux queries | Grafana dashboard patterns |
| iot-005 | references/iot/guide_loki_logging.md | Loki, logging, aggregation, labels | Loki log aggregation |

> Usage note: load iot-001 + iot-002 for MQTT ingestion tasks.
> Load iot-003 for InfluxDB schema design during ARCHITECTURE phase.
> Load iot-004 + iot-005 for monitoring/observability tasks (2 of 3 R-PD-03 slots).
