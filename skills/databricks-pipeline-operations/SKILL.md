---
name: databricks-pipeline-operations
description: >-
  Databricks LakeFlow / DLT pipeline lifecycle operations — create, view, update, run, stop, pause, delete,
  monitor, and troubleshoot pipelines via the Databricks CLI and REST API. Performance analysis using
  pipeline events (flow_progress, num_upserted_rows), Delta table byte metrics (DESCRIBE DETAIL,
  DESCRIBE HISTORY), source database wire-byte measurement, and system.compute.node_timeline for
  CPU/RAM/network/disk correlation per run (5–7 day lag, classic and serverless). Comparison testing:
  create separate source users, UC connections, and pipelines per test dimension (network path,
  DB engine, compute tier, etc.) so metrics never interleave and auto-detection works from
  connection name suffixes without extra arguments. Use when performing any pipeline lifecycle
  operation, extracting rows/bytes metrics per run, comparing configurations, checking schema type
  mappings, building performance baselines, correlating infrastructure resource usage with pipeline
  run timelines, identifying which update_id matches a benchmark record, or building pipeline
  analysis reports in Databricks notebooks.
---

# Databricks Pipeline Operations

## Lifecycle operations — CLI quick reference

All commands use `databricks -p <PROFILE>`.

| Operation | Command |
|-----------|---------|
| view | `databricks pipelines get --pipeline-id <id>` |
| list all | `databricks pipelines list-pipelines` |
| create | `databricks pipelines create --json @spec.json` |
| update/modify | `databricks pipelines update --pipeline-id <id> --json @patch.json` |
| run (triggered) | `databricks pipelines start-update --pipeline-id <id>` |
| full refresh | `databricks pipelines start-update --pipeline-id <id> --full-refresh-selection <table>` |
| stop | `databricks pipelines stop --pipeline-id <id>` |
| pause (continuous only) | PATCH `{"continuous": false}` via `databricks api patch /api/2.0/pipelines/<id>` |
| tail live events | `databricks pipelines logs --pipeline-id <id>` |
| list all updates | `databricks api get "/api/2.0/pipelines/<id>/updates?max_results=20"` |
| delete | `databricks pipelines delete --pipeline-id <id>` |

**Pause for triggered pipelines**: there is no pause — simply don't trigger a new run. For continuous mode, flip `continuous: false` or stop the in-flight update.

**Delete does not**: drop UC tables, schemas, or (for CDC pipelines) replication slots. Clean those up separately.

## Performance metrics from pipeline events

### Primary signal: is work being done?

```bash
# rows ingested per micro-batch for a specific update
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?filter=update_id%3D'<uid>'&max_results=200" \
  | jq '[.events[]
    | select(.event_type=="flow_progress"
         and (.details.flow_progress.metrics.num_upserted_rows // 0) > 0)
    | { table: (.message | split("'"'"'")[1] | split(".")[-1]),
        rows:  .details.flow_progress.metrics.num_upserted_rows,
        ts:    .timestamp }]'
```

**A COMPLETED run with 0 rows is not an error** — QBC cursor found no rows with `dt > watermark`. Expected when no source writes occurred between runs.

### Update state summary

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/updates?max_results=20" \
  | jq '.updates[] | {update_id, state, creation_time, cause}'
```

### Event API limit: max_results ≤ 200

The events endpoint rejects `max_results > 250`. Use 200 as the safe ceiling.

### Events are newest-first — paginate for older updates

```bash
RESP=$(databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200")
TOKEN_NEXT=$(echo "$RESP" | jq -r '.next_page_token // empty')
# Fetch older events
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200&page_token=${TOKEN_NEXT}"
```

If an `update_id` appears in `list-updates` but not in events, it fell off page 1 — paginate.

### Exact update durations from update_progress events

Group `update_progress` events by `update_id` to get state transition timestamps without needing system tables:

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200" \
  | jq '[.events[] | select(.event_type == "update_progress") | {
      update_id: .origin.update_id,
      state: .details.update_progress.state, ts: .timestamp
    }] | group_by(.update_id) |
    map({update_id: .[0].update_id, events: map({state:.state,ts:.ts}) | sort_by(.ts)})'
```

**Benchmark time = RUNNING → COMPLETED** (not total wall clock). This excludes cluster startup and isolates the actual scan/ingest duration.

| State transition | Duration = | Notes |
|-----------------|-----------|-------|
| WAITING_FOR_RESOURCES → INITIALIZING | Serverless pool wait | Skipped when pool has capacity |
| INITIALIZING → RUNNING | Cluster bootstrap + table setup | 60–400s for serverless |
| **RUNNING → COMPLETED** | **Actual scan/ingest work** | **This is the benchmark time** |

## SQL Statement API vs pipeline events API

The SQL Statement API (`POST /api/2.0/sql/statements`) requires a **running** warehouse. If the warehouse is `STOPPED`, it must wake up first (2–3 min), causing `wait_timeout` failures. The events API has no such dependency.

| Need | Use | Why |
|------|-----|-----|
| Update durations, state transitions, flow timelines | Pipeline events API | Always available, no warehouse |
| Which tables (flows) ran in each update | Pipeline events (`flow_progress`) | Always available, no warehouse |
| Cross-update trend analysis, node_timeline join | SQL Statement API | Events API can't join system tables |
| Historical resource metrics (CPU/RAM/disk) | SQL Statement API + `system.compute.node_timeline` | Only source; 5–7 day lag |

**Default**: use events API when it can answer the question. Only use SQL Statement API when system table data is strictly required.

## Byte metrics — Delta side

```sql
-- Current table size
DESCRIBE DETAIL `catalog`.`schema`.`table`   -- → sizeInBytes, numFiles

-- Bytes written per commit
SELECT version, timestamp,
  operationMetrics:numTargetBytesAdded  AS bytes_added,
  operationMetrics:numTargetRowsInserted AS rows_inserted,
  operationMetrics:executionTimeMs       AS exec_ms
FROM (DESCRIBE HISTORY `catalog`.`schema`.`table`)
WHERE operation = 'MERGE' ORDER BY version DESC LIMIT 10
```

## Byte metrics — MySQL source side

`Bytes_sent` from `performance_schema.status_by_account` = exact JDBC wire bytes per user. Snapshot before and after each triggered run; the delta is the exact wire bytes for that run.

Use separate MySQL users per routing path (`lfc_user_public`, `lfc_user_png`) so byte counters are clean even during concurrent runs.

**For the full pre/post snapshot SQL, assumptions table, and thread-level fallback**, see [byte-metrics.md](byte-metrics.md).

## Performance normalization

**Derived metrics per run:**
- `rows/s` = `total_rows / duration_s`
- `MySQL MB/s` = `MySQL DATA_LENGTH / duration_s` — source read throughput ceiling
- `Delta MB/s` = `numTargetBytesAdded` sum / `duration_s` — storage write throughput

**Observed baselines (MySQL 8 → UC Delta, serverless, zstd):**

| Table size | rows/s (steady state) | MySQL MB/s | Compression ratio |
|---|---|---|---|
| 1M  | ~35–40K | ~2–4 MB/s (startup dominated) | ~4–5× |
| 10M | ~130–150K | ~6–7 MB/s | ~8–9× |
| 100M | ~130–155K | ~6–7 MB/s | ~14–15× |

**~20–25s of every run is fixed overhead** (cluster init, JDBC setup, schema resolution). Establish separate baselines per table size range.

## Troubleshooting decision tree

```
Run COMPLETED with 0 rows?
  → Normal if no source writes between runs.
  → Abnormal if source DML is happening: check cursor column watermark.

Run FAILED?
  → Pull all events for that update_id.
  → Look for flow_progress FAILED events first.
  → Check: source DB connectivity, credentials, CDC replication slot health.

Run CANCELED repeatedly?
  → Single cancel = normal. Recurrent = investigate.
  → Check executionTimeMs in DESCRIBE HISTORY operationMetrics.

Rows/s drops vs baseline?
  → Compare MySQL MB/s — if also low, bottleneck is source DB.
  → If MB/s normal but rows/s dropped, row size increased.

0 rows across many consecutive runs despite known source activity?
  → SCD Type 1 with stale cursor: check SHOW CREATE TABLE for ON UPDATE CURRENT_TIMESTAMP.
```

**For full latency triage (bootstrap vs source vs network vs target bottleneck), identifying update_ids from benchmark records, and saving run metrics**, see [latency-triage.md](latency-triage.md).

## Schema: MySQL → Delta type mapping

| MySQL type | Delta type | Notes |
|---|---|---|
| `bigint unsigned` | `DECIMAL` | Unsigned exceeds Spark BIGINT max |
| `bigint` | `BIGINT` | Signed fits |
| `timestamp` | `TIMESTAMP` | 1:1; also serves as QBC cursor column |
| `varchar(n)` | `STRING` | n is ignored in Delta |
| `int` | `INT` | 1:1 |
| `decimal(p,s)` | `DECIMAL(p,s)` | 1:1 |
| `datetime` | `TIMESTAMP` | Precision differences possible |

Delta table features enabled by LFC: `changeDataFeed`, `deletionVectors`, `rowTracking`, `columnMapping` (name mode), `typeWidening`.

## Compute infrastructure per update

```bash
# Get cluster info for a specific update
databricks -p $PROFILE api get "/api/2.0/pipelines/<pid>/updates/<uid>" \
  | jq '{cluster_id: .update.cluster_id,
         serverless:  .update.config.serverless,
         start:       .update.start_time,
         end:         .update.end_time}'
```

`serverless: null` in the pipeline spec does NOT mean classic — check `.update.config.serverless` per run. The `-v2n` cluster ID suffix is a reliable serverless indicator.

**Serverless cluster**: one fresh cluster per update, `node_type_id: null`, `num_workers: null`, ARM architecture, cluster name `dlt-execution-<pipeline_id>`.

**For system.compute.node_timeline schema, correlation SQL queries, and resource signal interpretation**, see [system-tables.md](system-tables.md).

## CDC cleanup after pipeline delete

LFC creates PostgreSQL replication slots named `dbx_%_<GATEWAY_PIPELINE_ID>`.

```sql
SELECT pg_drop_replication_slot(slot_name)
  FROM pg_replication_slots
  WHERE slot_name LIKE 'dbx_%_<gateway_pipeline_id>';
```

MySQL binlog-based CDC: no explicit slot to drop.

## Comparison testing

Create one distinct identifier per layer per dimension (routing, DB engine, etc.) so metrics never interleave:

| Layer | Public path | PNG path |
|-------|-------------|----------|
| MySQL user | `lfc_user_public` | `lfc_user_png` |
| UC connection | `<prefix>-<key>-public` | `<prefix>-<key>-png` |
| LakeFlow pipeline | `<prefix>-<key>-qbc-public` | `<prefix>-<key>-qbc-png` |
| UC schema (destination) | `db_ingest_public` | `db_ingest_png` |

The `databricks-run-with-byte-capture.sh` script derives routing from the `-public` / `-png` suffix automatically — no extra arguments beyond the pipeline ID.

**For the full identifier stack, auto-detection chain, and how to add new dimensions**, see [comparison-testing.md](comparison-testing.md).

## Notebook pattern for pipeline analysis

**Prefer a Databricks workspace notebook over a local canvas** — live data, WorkspaceClient SDK, `display()`, widgets, schedulable.

Structure: widgets → SDK + fetch updates → events → run history table → throughput charts → system tables → node_timeline join → Delta bytes.

**For full cell-by-cell notebook code**, see [notebook-analysis.md](notebook-analysis.md).
