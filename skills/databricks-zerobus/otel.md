# ZeroBus — OpenTelemetry

ZeroBus supports propagating OpenTelemetry trace context into ingested rows, making it possible to correlate ingestion events with downstream Delta table queries in Grafana or Jaeger.

## Reference notebooks (zerobusdemo)

| Notebook | Purpose |
|---|---|
| `notebooks/otel/grafana/secrets-bootstrap.ipynb` | Provision Databricks secrets for Grafana/OTel credentials |
| `notebooks/otel/grafana/zerobus-otel.ipynb` | End-to-end ZeroBus ingest with OTel span injection, Grafana dashboard |
| `notebooks/otel/uc/zerobus-uc-otel.ipynb` | Unity Catalog variant |
| `notebooks/otel/jaeger/jaeger.sh` | Local Jaeger setup for trace visualization |

## Secrets bootstrap

Run `secrets-bootstrap.ipynb` once per workspace to store OTel credentials in Databricks secret scopes. The notebook uses `dbutils.secrets.put()` — see `notebooks/otel/grafana/secrets-bootstrap.ipynb` for the full scope/key list.

## Span injection pattern

```python
from opentelemetry import trace
from opentelemetry.propagate import inject

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("zerobus.ingest"):
    carrier = {}
    inject(carrier)   # adds traceparent / tracestate headers
    # pass carrier as additional columns or headers in the ingest call
```

## Grafana integration

- OTel collector endpoint and credentials stored in Databricks secrets.
- The `zerobus-otel.ipynb` notebook reads secrets via `dbutils.secrets.get()` and configures the OTLP exporter at startup.
- Spans are exported to Grafana Tempo; logs to Grafana Loki.

## Setup notes

- Requires `opentelemetry-sdk`, `opentelemetry-exporter-otlp` in the cluster/venv.
- The OTel collector must be reachable from the Databricks serverless compute network.
- For local dev, use Jaeger (`notebooks/otel/jaeger/jaeger.sh`) to avoid needing a cloud collector.
