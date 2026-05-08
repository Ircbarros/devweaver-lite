# guide_mqtt_production

> IoT Stream B — MQTT production: topic design, security hardening, scalable architecture.
> Related: `guide_mqtt_influxdb.md` · `guide_influxdb_schema.md` · `guide_grafana_dashboards.md` · `guide_loki_logging.md`

---

## Table of Contents

1. [Topic Naming Rules](#1-topic-naming-rules)
2. [Telemetry Topic Schema](#2-telemetry-topic-schema)
3. [Command Topic Schema](#3-command-topic-schema)
4. [Wildcard Policy](#4-wildcard-policy)
5. [Device Shadow Best Practices](#5-device-shadow-best-practices)
6. [Client-Side Security (Python/Paho)](#6-client-side-security-pythonpaho)
7. [Server-Side Security](#7-server-side-security)
8. [Data-at-Rest Security](#8-data-at-rest-security)
9. [Scalable Architecture Patterns](#9-scalable-architecture-patterns)

---

## 1. Topic Naming Rules

**General rules (AWS IoT Core-validated):**
- Max **7 forward slashes** per topic name
- Max **256 bytes** UTF-8 total
- Use **lowercase letters + numbers + dashes** only — no camelCase, no spaces
- No leading `/` (e.g., `dt/app/ctx/device` not `/dt/app/ctx/device`)
- No wildcards (`+` or `#`) in published topic names
- General-to-specific hierarchy **left-to-right**
- Topics starting with `$` are reserved by the broker
- **Prefix `dt/` for all telemetry** (data), **`cmd/` for all commands** — never overlap

---

## 2. Telemetry Topic Schema

```
dt/<application>/<context>/<thing-name>/<dt-type>
```

| Segment | Description |
|---|---|
| `dt` | Fixed data prefix; always include |
| `application` | IoT app identifier or hardware version |
| `context` | Location, group-id, or provisioning-time info |
| `thing-name` | Unique device identity (Thing Name) |
| `dt-type` | Optional subcomponent e.g. `geo`, `accel`, `temp` |

**Example:**
```
dt/building-monitor/floor-3/sensor-001/temp
dt/building-monitor/floor-3/sensor-001/humid
```

---

## 3. Command Topic Schema

```
# Request
cmd/<application>/<context>/<destination-id>/<req-type>
# Response
cmd/<application>/<context>/<destination-id>/<res-type>
```

**Payload requirements** — every command payload MUST include:
- `session-id`: correlates request to response
- `response-topic`: where device should publish the response

---

## 4. Wildcard Policy

| Actor | Allowed | Forbidden |
|---|---|---|
| Device (pub/sub) | `+` single-level wildcard | `#` multi-level wildcard |
| Rules engine / backend | `#` | — |
| Devices MUST NOT subscribe to the root `#` | | |

---

## 5. Device Shadow Best Practices

- One shadow per device — no sharing between devices
- Use **Named Shadows** for logical property groups (e.g., `firmware`, `config`, `sensors`)
- Shadow for **low-to-medium TPS** state sync only
- For **high-frequency** data, use dedicated MQTT topics (not shadow)
- Store **firmware version** in shadow fields
- Use `clientToken` field to track the request sender

---

## 6. Client-Side Security (Python/Paho)

```python
import ssl
import paho.mqtt.client as mqtt

client = mqtt.Client(client_id="device-001", clean_session=True)

# 1. Username/password authentication
client.username_pw_set("device-001", "secret-password")

# 2. TLS/SSL — always use port 8883, never 1883
client.tls_set(
    ca_certs="/certs/ca.crt",
    certfile="/certs/client.crt",   # mTLS: present own cert
    keyfile="/certs/client.key",
    tls_version=ssl.PROTOCOL_TLSv1_2,
)

# 3. Always validate the broker certificate
client.tls_insecure_set(False)

client.connect("broker.example.com", 8883)
```

**Security rules:**
- Always use **port 8883** (TLS), never plain 1883
- Enforce `PROTOCOL_TLSv1_2` or higher
- `tls_insecure_set(False)` — never skip cert validation
- **mTLS preferred**: device presents its own certificate

---

## 7. Server-Side Security

| Layer | Controls |
|---|---|
| Transport | TLS/SSL on port 8883 |
| Authentication | Username + password per device |
| Authorization | ACLs per username AND per client-ID |
| Topic ACLs | Restrict publish/subscribe to device-specific topic prefix |

**ACL pattern (Mosquitto-style):**
```
# Device can only publish/subscribe to its own dt/ and cmd/ prefixes
user device-001
topic readwrite dt/+/+/device-001/#
topic readwrite cmd/+/+/device-001/#
```

---

## 8. Data-at-Rest Security

- **Auth keys**: hash + salt (bcrypt/argon2); never store plaintext
- **Key storage**: Use HSM or cloud KMS for production secrets
- **RBAC** on all data access (broker admin, InfluxDB tokens)
- **Data retention policies**: define TTL per bucket; GDPR-compliant deletion
- **Network segmentation**: VLANs / NSGs to isolate broker from public internet
- **Cloud**: isolated containers, private VPC, cloud-native encryption (AWS KMS / GCP Cloud KMS)

---

## 9. Scalable Architecture Patterns

**Stateless clients** — each device/subscriber can reconnect to any broker node.

| Setting | Use Case |
|---|---|
| `clean_session=True` | Ephemeral telemetry; memory-efficient |
| `clean_session=False` | Command-and-control; guaranteed delivery |

**Topic hierarchy for large deployments:**
```
<region>/<plant>/<line>/<device>/<parameter>
eu-west/plant-1/line-a/pump-003/pressure
```

**Monitoring broker health:**
- Export broker metrics (connections, message rate, queue depth) via Prometheus
- Visualize with Grafana (see `guide_grafana_dashboards.md`)

**Staging rollouts (IoT Jobs):**
- Organize thing groups by firmware version, hardware rev, and environment
- Deploy firmware updates to a subset first; monitor for 24h before full rollout
