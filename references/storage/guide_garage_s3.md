# guide_garage_s3

> Stream B — Garage S3-compatible object storage + OTel/Loki/Grafana observability patterns.

---

## Table of Contents

1. [Garage S3 — Architecture & Gotchas](#1-garage-s3--architecture--gotchas)
2. [aioboto3 Configuration](#2-aioboto3-configuration)
3. [Pre-signed URLs](#3-pre-signed-urls)
4. [Bucket Naming & Security](#4-bucket-naming--security)
5. [OpenTelemetry Python SDK](#5-opentelemetry-python-sdk)
6. [structlog + OTel Correlation IDs](#6-structlog--otel-correlation-ids)
7. [Loki — Label Design & Config](#7-loki--label-design--config)
8. [Grafana — Dashboard-as-Code](#8-grafana--dashboard-as-code)
9. [Stable Versions](#9-stable-versions)

---

## 1. Garage S3 — Architecture & Gotchas

**No AWS IAM**: Garage uses access key pairs (format: `GK…` public, secret key).  
**Lifcycle policies**: Garage 0.9.x does **NOT** support native bucket lifecycle; version pruning must be done manually or via n8n scheduled workflow.  
**SSE-S3**: Not natively supported in Garage 0.9.x — use client-side encryption:

```python
from cryptography.fernet import Fernet
KEY = Fernet(os.getenv("STORAGE_ENCRYPTION_KEY").encode())

async def upload_encrypted(data: bytes, bucket: str, key: str) -> None:
    encrypted = KEY.encrypt(data)
    async with _s3_client() as s3:
        await s3.put_object(Bucket=bucket, Key=key, Body=encrypted)
```

**Known aioboto3 gotchas** with non-AWS endpoints:
- `head_object()` may return 403 even on existing objects → use `get_object(Range="bytes=0-0")` as an existence check.
- Replication quorum requires > 50% nodes online; single-node = no HA.
- Pre-signed URL TTL checked only client-side (not enforced by Garage server-side).

---

## 2. aioboto3 Configuration

```python
import aioboto3, os
from contextlib import asynccontextmanager

_SESSION = aioboto3.Session(
    aws_access_key_id=os.getenv("GARAGE_ACCESS_KEY"),
    aws_secret_access_key=os.getenv("GARAGE_SECRET_KEY"),
)

@asynccontextmanager
async def _s3_client():
    async with _SESSION.client(
        "s3",
        endpoint_url=os.getenv("GARAGE_ENDPOINT_URL"),  # e.g. http://garage-api:3900
        region_name="garage",                            # Arbitrary; Garage ignores it
        config=botocore.config.Config(signature_version="s3v4"),
    ) as s3:
        yield s3
```

**Security**: Garage endpoint must NOT be exposed to the public internet (reverse-proxy with auth only).

---

## 3. Pre-signed URLs

```python
async def generate_upload_url(bucket: str, key: str) -> str:
    async with _s3_client() as s3:
        return await s3.generate_presigned_url(
            "put_object",
            Params={"Bucket": bucket, "Key": key},
            ExpiresIn=900,  # 15 minutes
        )

async def generate_download_url(bucket: str, key: str) -> str:
    async with _s3_client() as s3:
        return await s3.generate_presigned_url(
            "get_object",
            Params={"Bucket": bucket, "Key": key},
            ExpiresIn=3600,  # 1 hour
        )
```

---

## 4. Bucket Naming & Security

**Convention**: `{project}-{env}-{purpose}` — e.g., `aibuilder-prod-models`, `aibuilder-staging-sbom`.

```python
async def ensure_bucket(bucket: str) -> None:
    async with _s3_client() as s3:
        await s3.create_bucket(Bucket=bucket)
        await s3.put_bucket_versioning(
            Bucket=bucket,
            VersioningConfiguration={"Status": "Enabled"},
        )
```

**No public ACLs**: All objects private by default; access only via pre-signed URLs or internal proxy.  
**Separate buckets per sensitivity**: models, sbom, raw-iot, exports — principle of least-privilege per access key.

---

## 5. OpenTelemetry Python SDK

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT")))
)
trace.set_tracer_provider(provider)

FastAPIInstrumentor.instrument()      # Auto-instrument FastAPI
HTTPXClientInstrumentor().instrument() # Auto-instrument httpx calls
```

**`traceparent` propagation**: The above auto-instrumentation handles W3C TraceContext header injection on outgoing requests.

---

## 6. structlog + OTel Correlation IDs

```python
import structlog
from opentelemetry import trace as otel_trace

def add_trace_context(logger, method, event_dict):
    span = otel_trace.get_current_span()
    if span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict

structlog.configure(
    processors=[
        add_trace_context,
        structlog.processors.JSONRenderer(),
    ]
)
log = structlog.get_logger()
log.info("rag.query", domain="ai", tokens=512)
```

**Every log entry must include**: `trace_id`, `span_id`, `service`, `env`, `module`.

---

## 7. Loki — Label Design & Config

**Labels** (low-cardinality only): `app`, `env`, `module`  
**Never use as labels**: user_id, request_id, timestamp, error message (high-cardinality → performance cliff).

```yaml
# Docker Compose Loki config (loki-config.yaml excerpt)
limits_config:
  ingestion_rate_mb: 10
  max_label_names_per_series: 5  # Enforce low cardinality
```

**Log shipping** (2026 recommendation): Use **Grafana Alloy** (replaces Promtail);

```river
// alloy-config.river (excerpt)
loki.source.docker "containers" {
  targets = discovery.docker.all.targets
  labels = {app="aibuilder", env=env("APP_ENV")}
}
```

---

## 8. Grafana — Dashboard-as-Code

```
assets/observability/
├── grafana/
│   ├── dashboards/
│   │   ├── api_performance.json
│   │   ├── iot_ingestion.json
│   │   └── llm_tracing.json
│   └── provisioning/
│       ├── datasources.yaml
│       └── dashboards.yaml
```

```yaml
# provisioning/datasources.yaml
apiVersion: 1
datasources:
  - name: InfluxDB
    type: influxdb
    url: http://influxdb:8086
    jsonData: { version: Flux, organization: main }
    secureJsonData: { token: ${INFLUX_TOKEN} }
  - name: Loki
    type: loki
    url: http://loki:3100
```

**Rule**: Never configure dashboards via Grafana UI; all dashboards version-controlled as JSON in `assets/`.

---

## 9. Stable Versions

| Package / Tool | Version | Stability |
|---|---|---|
| Garage | `0.9.x` | ✅ Stable (no lifecycle policies) |
| `aioboto3` | `12.1.x` | ✅ Stable |
| `opentelemetry-sdk` | `1.22.x` | ✅ Stable |
| `opentelemetry-exporter-otlp` | `1.22.x` | ✅ Stable |
| `structlog` | `24.x` | ✅ Stable |
| Loki | `3.x` | ✅ Stable |
| Grafana | `10.x` | ✅ Stable |
| Grafana Alloy | `1.x` | ✅ Stable (replaces Promtail) |
