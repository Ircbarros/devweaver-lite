# guide_mqtt_influxdb

> Stream B — IoT data acquisition: paho-mqtt v2/aiomqtt + InfluxDB v3 Line Protocol patterns.
> Related: `guide_garage_s3.md` · `guide_mqtt_production.md` · `guide_influxdb_schema.md` · `guide_grafana_dashboards.md` · `guide_loki_logging.md`

---

## Table of Contents

1. [MQTT Topic Design](#1-mqtt-topic-design)
2. [aiomqtt Connection & Subscription](#2-aiomqtt-connection--subscription)
3. [TLS + Auth Configuration](#3-tls--auth-configuration)
4. [Payload Validation](#4-payload-validation)
5. [Reconnection with Backoff](#5-reconnection-with-backoff)
6. [InfluxDB v3 Line Protocol](#6-influxdb-v3-line-protocol)
7. [Batch Write Pattern](#7-batch-write-pattern)
8. [Telegraf → InfluxDB Config](#8-telegraf--influxdb-config)
9. [Dead-Letter Queue (PostgreSQL)](#9-dead-letter-queue-postgresql)
10. [Stable Versions](#10-stable-versions)

---

## 1. MQTT Topic Design

**Format**: `{site}/{device}/{sensor}`  
**Examples**: `warehouse-1/device-42/temperature`, `office-a/hvac-01/humidity`

Rules:
- Topic components: lowercase, hyphens only (no slashes in component names).
- Never publish to wildcard topics; use wildcards in subscriptions only.
- Reserved prefix `$SYS/` is broker-only; never publish there.
- Subscribe per sensor group: `warehouse-1/+/temperature` (single-level wildcard `+`).
- Max topic length: 65 535 bytes (UTF-8); keep under 128 chars in practice.

---

## 2. aiomqtt Connection & Subscription

```python
import aiomqtt, asyncio, json, os

async def subscribe_iot():
    async with aiomqtt.Client(
        hostname=os.getenv("MQTT_HOST"),
        port=int(os.getenv("MQTT_PORT", "8883")),
        username=os.getenv("MQTT_USER"),
        password=os.getenv("MQTT_PASSWORD"),
        tls_params=_build_tls(),
    ) as client:
        async with client.messages() as messages:
            await client.subscribe("warehouse-1/+/+")
            async for message in messages:
                await _handle_message(message)
```

**paho-mqtt v1 → v2 breaking changes**:
- `loop_*()` methods removed; not needed with aiomqtt async context manager.
- `MQTTv311` constant moved to `mqtt.MQTTv5` / `mqtt.MQTTv311` explicitly.
- Callback registration no longer done in `__init__`.

---

## 3. TLS + Auth Configuration

```python
import ssl, aiomqtt

def _build_tls() -> aiomqtt.TLSParameters:
    return aiomqtt.TLSParameters(
        ca_certs="/etc/certs/ca.crt",
        certfile="/etc/certs/client.crt",   # Client cert (mTLS)
        keyfile="/etc/certs/client.key",
        cert_reqs=ssl.CERT_REQUIRED,
        tls_version=ssl.PROTOCOL_TLSv1_2,   # Min TLS 1.2; prefer 1.3
    )
```

**Mosquitto 2.x config** (must set explicitly):
```conf
allow_anonymous false
require_certificate true
cafile /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key
```

---

## 4. Payload Validation

```python
from jsonschema import validate, ValidationError

SENSOR_SCHEMA = {
    "type": "object",
    "properties": {
        "value": {"type": "number"},
        "unit": {"type": "string"},
    },
    "required": ["value"],
}

async def _handle_message(message: aiomqtt.Message):
    try:
        payload = json.loads(message.payload.decode())
        validate(instance=payload, schema=SENSOR_SCHEMA)
        await _write_to_influx(message.topic, payload)
    except (ValidationError, json.JSONDecodeError) as exc:
        await _send_to_dlq(str(message.payload), str(exc))
```

---

## 5. Reconnection with Backoff

```python
import asyncio, ntfy_client

async def subscribe_with_backoff(max_retries: int = 5):
    for attempt in range(max_retries):
        try:
            await subscribe_iot()
            break  # Success; exit loop
        except aiomqtt.MqttError as exc:
            backoff = min(2 ** attempt, 30)  # Cap at 30s
            if attempt == max_retries - 1:
                await ntfy_client.notify("MQTT reconnect failed", priority="critical")
                raise
            await asyncio.sleep(backoff)
```

---

## 6. InfluxDB v3 Line Protocol

**Format**: `measurement[,tag=value]... field=value[,field=value]... [timestamp_ns]`

```
temperature,site=warehouse-1,device=sensor-42 value=22.5 1715000000000000000
```

Rules:
- **Tags** (indexed): `site`, `device`, `sensor_id` — low cardinality (< 1 000 unique values).
- **Fields**: numeric readings only; strings in a separate `_meta` measurement.
- **Timestamps**: nanosecond precision, UTC only.
- Batch point limit: 1 000–5 000 points per write request.

**InfluxDB v2 → v3 changes**:
- Query language: Flux removed → use **SQL** queries now.
- API endpoint: `/api/v3/write`, `/api/v3/query/sql`.
- Continuous queries removed → use scheduled tasks.

---

## 7. Batch Write Pattern

```python
from influxdb_client import InfluxDBClient
from influxdb_client.client.write_api import SYNCHRONOUS
import os

_client = InfluxDBClient(
    url=os.getenv("INFLUX_URL"),
    token=os.getenv("INFLUX_TOKEN"),
    org=os.getenv("INFLUX_ORG"),
)
_write_api = _client.write_api(write_options=SYNCHRONOUS)

def write_batch(points: list[str]) -> None:
    """Write ≤ 5 000 Line Protocol strings to InfluxDB."""
    if len(points) > 5000:
        raise ValueError("Batch size exceeds 5 000 point limit")
    _write_api.write(
        bucket=os.getenv("INFLUX_BUCKET"),
        org=os.getenv("INFLUX_ORG"),
        write_precision="ns",
        record="\n".join(points),
    )
```

---

## 8. Telegraf → InfluxDB Config

```toml
[[inputs.prometheus]]
  urls = ["http://prometheus:9090/metrics"]
  interval = "10s"

[[outputs.influxdb_v3]]
  urls  = ["http://influxdb:8086"]
  bucket = "prometheus_data"
  org   = "main"
  token = "${INFLUX_TOKEN}"
```

- Telegraf buffer lag: up to 30s if output is down; monitor via ntfy alert.

---

## 9. Dead-Letter Queue (PostgreSQL)

```sql
CREATE TABLE IF NOT EXISTS iot_dlq (
    id SERIAL PRIMARY KEY,
    raw_payload TEXT NOT NULL,
    error_message TEXT,
    topic TEXT,
    received_at TIMESTAMPTZ DEFAULT NOW()
);
```

```python
async def _send_to_dlq(raw: str, error: str, topic: str = "") -> None:
    async with db.pool.acquire() as conn:
        await conn.execute(
            "INSERT INTO iot_dlq (raw_payload, error_message, topic) VALUES ($1, $2, $3)",
            raw, error, topic
        )
```

---

## 10. Stable Versions

| Package | Version | Stability |
|---|---|---|
| `aiomqtt` | `0.17.x` | ✅ Stable |
| `paho-mqtt` | `2.1.x` | ✅ Stable |
| `influxdb-client` | `1.18.x` | ✅ Stable |
| `jsonschema` | `4.22.x` | ✅ Stable |
| Telegraf | `1.31.x` | ✅ Stable |
| Mosquitto | `2.0.x` | ✅ Stable |

**Gotchas**:
- aiomqtt `messages()` queue may overflow on burst; throttle with MQTT broker QoS 1.
- Garage 0.9.x has no native SSE; use client-side encryption before object upload.
- Telegraf output lag up to 30s if InfluxDB unavailable.
