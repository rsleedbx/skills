---
name: databricks-zerobus
description: ZeroBus ingest into Unity Catalog Delta tables via HTTP REST or gRPC SDK, with optional OpenTelemetry. Use when writing ZeroBus ingest code, configuring auth (OAuth, service principal, SDK), setting the server endpoint, benchmarking HTTP vs gRPC, or adding OTel tracing to ZeroBus.
---

# ZeroBus Ingest

ZeroBus is a Databricks serverless ingestion API that writes rows directly into Unity Catalog Delta tables. Two transports: **HTTP REST** (no SDK install, OAuth token) and **gRPC SDK** (`databricks-zerobus-ingest-sdk`, streaming).

## Transport reference

- **HTTP REST** — `requests.Session` / `aiohttp`: see [http.md](http.md)
- **gRPC SDK** — `ZerobusSdk` / `ZerobusSdkAsync` / `zbhelper` wrapper: see [grpc.md](grpc.md)
- **OpenTelemetry** — span injection, Grafana, secrets bootstrap: see [otel.md](otel.md)

---

## Server Endpoint

```
https://{workspace_id}.zerobus.{workspace_hostname}
```

Example:
```
workspace:  https://e2-dogfood.staging.cloud.databricks.com
zerobus:    https://6051921418418893.zerobus.e2-dogfood.staging.cloud.databricks.com
```

HTTP insert URL:
```
{ZEROBUS_INGEST_URL}/zerobus/v1/tables/{TABLE_NAME}/insert
```

gRPC endpoint auto-constructed by `IngestConfig.from_workspace_client()` — no manual config needed.

**Endpoint resolution order (gRPC SDK):**
1. `server_endpoint=` argument
2. `ZEROBUS_SERVER_ENDPOINT` env var
3. Auto-constructed from workspace SDK config

---

## Table Name

Must be fully qualified: `catalog.schema.table` (e.g. `main.robert_lee.airquality`).

---

## Auth

### SDK auth — local dev (gRPC)

Uses `~/.databrickscfg`. No service principal needed.

```python
from src.zbhelper.zerobus_ingest import IngestConfig
config = IngestConfig.from_workspace_client("main.default.my_table")
```

### Service principal — CI / production (gRPC)

```bash
# Required env vars:
DATABRICKS_CLIENT_ID, DATABRICKS_CLIENT_SECRET,
ZEROBUS_SERVER_ENDPOINT, DATABRICKS_WORKSPACE_URL, ZEROBUS_TABLE_NAME
```

```python
config = IngestConfig.from_env()
```

### OAuth token — HTTP REST

See [http.md](http.md) — requires client credentials + `zerobusDirectWriteApi` resource scope.

---

## Unity Catalog Prerequisites

1. Target Delta table must exist with matching column names.
2. Principal needs `USE CATALOG`, `USE SCHEMA`, `SELECT + MODIFY` on the table.
3. For HTTP: include these as `authorization_details` in the token request (see [http.md](http.md)).
