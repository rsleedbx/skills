---
name: databricks-pipeline-operations
description: >-
  Databricks LakeFlow / DLT pipeline lifecycle operations — create, view, update, run, stop, pause, delete,
  monitor, and troubleshoot pipelines via the Databricks CLI and REST API. Performance analysis using
  pipeline events (num_upserted_rows, flow_progress), Delta table byte metrics (DESCRIBE DETAIL,
  DESCRIBE HISTORY numTargetBytesAdded), MySQL/source table sizes for end-to-end throughput, and
  system.compute.node_timeline for CPU/RAM/network/disk correlation per run (5–7 day lag, covers
  both classic and serverless). Comparison testing: create separate MySQL users, UC connections,
  and pipelines per dimension (routing path, DB engine, network mode) so metrics never interleave
  and databricks-run-with-byte-capture.sh auto-detects secret + routing user from the connection
  name suffix (-public / -png) without extra arguments. Use when performing any pipeline lifecycle
  operation, extracting rows/bytes metrics per run, comparing public vs PNG routing performance,
  checking schema type mappings, building performance baselines, correlating infrastructure resource
  usage with pipeline run timelines, or building pipeline analysis reports in Databricks notebooks
  (prefer notebooks over local canvases — live data, WorkspaceClient SDK, display(), widgets).
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
| open in browser | `databricks pipelines open --pipeline-id <id>` |

**Pause for triggered pipelines**: there is no pause — simply don't trigger a new run. For continuous mode, flip `continuous: false` or stop the in-flight update.

**Delete does not**: drop UC tables, schemas, or (for CDC pipelines) replication slots. Clean those up separately.

## Extracting performance metrics from events

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
  | jq '.updates[] | {update_id, state, creation_time}'
```

### Event API limit: max_results ≤ 200 (not 500)

The events endpoint rejects `max_results > 250`. Use 200 as the safe ceiling.

### Events API pagination — older updates fall off the first page

The events endpoint returns the **most recent events first**. When a pipeline has many updates, events from older runs fall off the first page. Paginate using `next_page_token`:

```bash
PIPELINE_ID="<id>"
PROFILE="<profile>"

# Page 1 (most recent)
RESP=$(databricks -p $PROFILE api get \
  "/api/2.0/pipelines/${PIPELINE_ID}/events?max_results=200")
echo "$RESP" | jq '[.events[] | select(.event_type == "update_progress") | {update_id: .origin.update_id, state: .details.update_progress.state, ts: .timestamp}]'

# Get next page token and fetch older events
TOKEN_NEXT=$(echo "$RESP" | jq -r '.next_page_token // empty')
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/${PIPELINE_ID}/events?max_results=200&page_token=${TOKEN_NEXT}" \
  | jq '[.events[] | select(.event_type == "update_progress") | {update_id: .origin.update_id, state: .details.update_progress.state, ts: .timestamp}]'
```

**When to paginate**: if an update_id appears in `list-updates` output but not in the events, it has fallen off page 1 — fetch page 2.

### Getting exact update durations from update_progress events

`update_progress` events record every state transition for an update. Group by `update_id` to compute exact phase durations without needing system tables:

```bash
# All state transitions for all updates in one call — group by update_id
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/${PIPELINE_ID}/events?max_results=200" \
  | jq '[.events[] | select(.event_type == "update_progress") | {
      update_id: .origin.update_id,
      state: .details.update_progress.state,
      ts: .timestamp
    }] | group_by(.update_id) |
    map({
      update_id: .[0].update_id,
      events: map({state:.state, ts:.ts}) | sort_by(.ts)
    })'
```

Key states and their timing meaning:

| State transition | Duration = | Notes |
|-----------------|-----------|-------|
| `WAITING_FOR_RESOURCES` → `INITIALIZING` | Serverless pool wait | Skipped when pool has capacity |
| `INITIALIZING` → `RUNNING` | Cluster bootstrap + table setup | 60–400s for serverless |
| `RUNNING` → `COMPLETED` | Actual scan/ingest work | This is the **benchmark time** |

**The benchmark time** (the `time` column in a performance log) is always **RUNNING → COMPLETED**, not total wall clock from update creation. This excludes cluster startup overhead and isolates the actual MySQL scan duration.

## Byte metrics — Delta side

### Current table size (DESCRIBE DETAIL)

Returns `sizeInBytes`, `numFiles`, `createdAt`, `lastModified`.

```sql
DESCRIBE DETAIL `catalog`.`schema`.`table`
```

Via SQL statement API (warehouse must exist):
```python
payload = json.dumps({
    "statement": "DESCRIBE DETAIL `cat`.`sch`.`tbl`",
    "warehouse_id": "<wh_id>",
    "wait_timeout": "50s",   # must be 5–50s or 0
    "on_wait_timeout": "CANCEL"
})
subprocess.run(["databricks", "-p", PROFILE, "api", "post",
                "/api/2.0/sql/statements", "--json", payload], ...)
```

### Bytes written per commit (DESCRIBE HISTORY)

Key field: `operationMetrics:numTargetBytesAdded` — bytes written for each MERGE/INSERT commit.

```sql
SELECT version, timestamp,
  operationMetrics:numTargetBytesAdded  AS bytes_added,
  operationMetrics:numTargetRowsInserted AS rows_inserted,
  operationMetrics:executionTimeMs       AS exec_ms
FROM (DESCRIBE HISTORY `catalog`.`schema`.`table`)
WHERE operation = 'MERGE'
ORDER BY version DESC
LIMIT 10
```

One MERGE commit per micro-batch (streaming update interval). Summing `numTargetBytesAdded` across all MERGEs for an update gives total Delta bytes written for that run.

## Byte metrics — MySQL source side

### Static proxy: DATA_LENGTH

```sql
SELECT TABLE_NAME,
  TABLE_ROWS,
  AVG_ROW_LENGTH,
  round(DATA_LENGTH/1024/1024, 1) AS data_mb,
  round(DATA_LENGTH/TABLE_ROWS, 1) AS bytes_per_row
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = '<db>'
  AND TABLE_NAME IN ('intpk', 'intpk_10m', 'intpk_100m');
```

`DATA_LENGTH` = InnoDB on-disk size — a proxy, not wire bytes. Wire bytes are uncompressed row data (~= DATA_LENGTH ÷ TABLE_ROWS × rows_read), which will exceed `DATA_LENGTH` because InnoDB compresses on disk but sends raw rows over JDBC.

### Assumptions underlying the byte measurement

Before using `performance_schema.status_by_account` as a wire-byte proxy, verify each assumption below. Assumptions marked **[VERIFIED]** were confirmed by a live test on the date shown; re-verify on new Databricks or MySQL versions.

| # | Assumption | Status | How to verify |
|---|------------|--------|---------------|
| A1 | LFC connects to MySQL without protocol compression — `Bytes_sent` = actual wire bytes | **[VERIFIED Apr 2026]** — `useCompression` is not in the UC connection supported options list | Probe the UC connection API: send a dummy option and read the "Supported options:" error |
| A2 | `performance_schema` is enabled on the MySQL instance | **[ASSUMED ON]** — default in MySQL 8; verify before relying on it | `SHOW VARIABLES LIKE 'performance_schema';` — must return `ON` |
| A3 | `status_by_account` counters are cumulative since last server start and have not been flushed | **[ASSUMED]** — detect by comparing `Bytes_sent` delta sign (negative = reset occurred) | `SHOW GLOBAL STATUS LIKE 'Uptime';` before/after; `FLUSH STATUS` between runs resets counters |
| A4 | `Bytes_sent` is measured before TLS encryption — counter is independent of TLS on/off | **[VERIFIED Apr 2026]** — MySQL counts bytes at the protocol layer, above the TLS stack | Enable/disable `trustServerCertificate` on the UC connection; `Bytes_sent` delta should be identical |
| A5 | Each serverless cluster run uses a distinct source IP — `status_by_account` delta for `lfc_user` equals bytes for exactly this run | **[ASSUMED]** — serverless nodes are ephemeral; IP reuse across runs is unlikely but possible | Compare `COUNT(DISTINCT HOST)` in `status_by_account` for `lfc_user` before vs after; new entries are this run's nodes |

**If any assumption is violated**, the `Bytes_sent` delta will be incorrect. The script `databricks-run-with-byte-capture.sh` guards against assumption A3 (warns on negative delta) but cannot guard against A5 automatically.

### Exact wire bytes via performance_schema — pre/post snapshot

MySQL exposes `Bytes_sent` (server → JDBC client) and `Bytes_received` (client → server) per account. Snapshot before and after each triggered run; the delta is the exact wire bytes for that run.

**Step 1 — create one MySQL user per routing path**

Using separate users (instead of a shared `lfc_user`) lets `status_by_account` give clean per-routing byte counts with no thread correlation needed, even if both pipelines run concurrently:

```sql
-- one-time setup (in addition to standard LFC grants)
CREATE USER 'lfc_user_public'@'%' IDENTIFIED BY '<password>';
CREATE USER 'lfc_user_png'@'%'    IDENTIFIED BY '<password>';
-- grant same privileges as lfc_user to both
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'lfc_user_public'@'%';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'lfc_user_png'@'%';
-- update UC connections to use the dedicated user per routing path
```

If separate users are not feasible, fall back to `status_by_thread` (see below) — but concurrent runs become ambiguous.

**Step 2 — pre-run snapshot**

```sql
-- capture before triggering the pipeline
SELECT USER, HOST, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS bytes
FROM performance_schema.status_by_account
WHERE USER IN ('lfc_user_public', 'lfc_user_png')
  AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
ORDER BY USER, VARIABLE_NAME;
```

**Step 3 — trigger the run**

```bash
databricks -p $PROFILE pipelines start-update --pipeline-id <id>
# wait for COMPLETED state:
watch -n 5 'databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<id>/updates?max_results=1" \
  | jq ".updates[0] | {state, update_id}"'
```

**Step 4 — post-run snapshot and delta**

```sql
-- run after pipeline reaches COMPLETED / CANCELED
WITH pre AS (
  SELECT USER, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
  FROM performance_schema.status_by_account
  WHERE USER IN ('lfc_user_public', 'lfc_user_png')
    AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
),
post AS (
  -- re-run the same SELECT after the pipeline finishes and paste results here,
  -- or compare two snapshots taken via application code
  SELECT USER, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
  FROM performance_schema.status_by_account
  WHERE USER IN ('lfc_user_public', 'lfc_user_png')
    AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
)
SELECT post.USER, post.VARIABLE_NAME,
  post.v - pre.v                          AS bytes_delta,
  ROUND((post.v - pre.v) / 1048576.0, 2) AS mb_delta
FROM post JOIN pre USING (USER, VARIABLE_NAME)
ORDER BY post.USER, post.VARIABLE_NAME;
```

`Bytes_sent` delta = exact JDBC read volume for the run (server sent this to the LFC client).

**Caveats:**
- `status_by_account` counters are cumulative since server start (or last `FLUSH STATUS`). If the server restarts between pre and post, the delta is invalid — detect with `SHOW GLOBAL STATUS LIKE 'Uptime'`.
- Any other operation by the same user between snapshots (e.g. admin queries) inflates the delta. With dedicated `lfc_user_public` / `lfc_user_png` users, this is negligible.
- `Bytes_received` tracks query text sent from the JDBC client — always small (a few KB). Focus on `Bytes_sent` for the data volume metric.

### Fallback: status_by_thread (shared user)

If both pipelines use the same `lfc_user`, use thread-level tracking. Capture the thread ID during the run:

```sql
-- while pipeline is running: identify the LFC connection thread
SELECT t.THREAD_ID, t.PROCESSLIST_ID, t.PROCESSLIST_HOST,
       t.PROCESSLIST_DB, t.PROCESSLIST_COMMAND,
       s_sent.VARIABLE_VALUE  AS bytes_sent_so_far,
       s_recv.VARIABLE_VALUE  AS bytes_recv_so_far
FROM performance_schema.threads t
JOIN performance_schema.status_by_thread s_sent
  ON t.THREAD_ID = s_sent.THREAD_ID AND s_sent.VARIABLE_NAME = 'Bytes_sent'
JOIN performance_schema.status_by_thread s_recv
  ON t.THREAD_ID = s_recv.THREAD_ID AND s_recv.VARIABLE_NAME = 'Bytes_received'
WHERE t.PROCESSLIST_USER = 'lfc_user'
  AND t.PROCESSLIST_COMMAND != 'Sleep'
ORDER BY CAST(s_sent.VARIABLE_VALUE AS UNSIGNED) DESC;
```

The row with the highest `bytes_sent_so_far` during the active run is the LFC JDBC thread. Thread counters reset when the connection closes, so you must sample while the run is live.

### Separate users vs shared user — comparison

| Concern | Separate users | Shared user |
|---------|---------------|-------------|
| Concurrent P1 + P2 runs | Clean — account-level delta per user | Ambiguous — threads interleave |
| Timing requirement | None — snapshot before/after | Must be sampled during the run |
| Counter reset risk | Only on server restart | Same, plus thread close resets to 0 |
| Setup cost | Two extra GRANT statements | None |
| Recommendation | **Preferred** | Use only if user consolidation is required |

## Performance normalization

**Derived metrics per run:**
- `rows/s` = `total_rows / duration_s` (from `num_upserted_rows` sum + update_progress timestamps)
- `MySQL MB/s` = `MySQL DATA_LENGTH / duration_s` — source read throughput ceiling
- `Delta MB/s` = `numTargetBytesAdded` sum / `duration_s` — storage write throughput
- `compression ratio` = `MySQL DATA_LENGTH / Delta sizeInBytes`

**Observed baselines (MySQL 8 → UC Delta, classic compute, zstd):**

| Table size | rows/s (steady state) | MySQL MB/s | Compression ratio |
|---|---|---|---|
| 1M  | ~35–40K | ~2–4 MB/s (startup dominated) | ~4–5× |
| 10M | ~130–150K | ~6–7 MB/s | ~8–9× |
| 100M | ~130–155K | ~6–7 MB/s | ~14–15× |

Compression improves with scale — sequential integer PKs and repeated column values compress extremely well in Parquet/ZSTD columnar layout.

**Small-run throughput is lower**: ~20–25s of every run is fixed overhead (cluster init, JDBC setup, schema resolution) regardless of row count. Establish separate baselines per table size range.

## Troubleshooting decision tree

```
Run COMPLETED with 0 rows?
  → Normal if no source writes between runs.
  → Abnormal if source DML is happening: check cursor column watermark.
    • Query MIN/MAX(dt) in source vs watermark in last flow_progress IDLE event.
    • Verify cursor_columns config matches actual column name.

Run FAILED?
  → Pull all events for that update_id (filter=update_id%3D'<uid>').
  → Look for flow_progress FAILED events first.
  → Check: source DB connectivity, credentials, CDC replication slot health.

Run CANCELED repeatedly?
  → Single cancel = normal (manual stop or timeout). Recurrent = investigate.
  → Check executionTimeMs in DESCRIBE HISTORY operationMetrics.
  → If 3× the baseline duration before cancel, suspect JDBC timeout or OOM.

Rows/s drops vs baseline?
  → Compare MySQL MB/s — if also low, bottleneck is source DB (CPU/IO/network).
  → If MB/s is normal but rows/s dropped, row size increased (wider schema or larger values).

0 rows across many consecutive runs despite known source activity?
  → SCD Type 1 with stale cursor: dt watermark may be far ahead of actual data.
  → Check: SHOW CREATE TABLE in MySQL to verify dt has ON UPDATE CURRENT_TIMESTAMP.
```

## Schema: MySQL → Delta type mapping

| MySQL type | Delta type | Notes |
|---|---|---|
| `bigint unsigned` | `DECIMAL` | Unsigned exceeds Spark BIGINT max; LFC maps to DECIMAL |
| `bigint` | `BIGINT` | Signed fits |
| `timestamp` | `TIMESTAMP` | 1:1; also serves as QBC cursor column |
| `varchar(n)` | `STRING` | n is ignored in Delta |
| `int` | `INT` | 1:1 |
| `decimal(p,s)` | `DECIMAL(p,s)` | 1:1 |
| `datetime` | `TIMESTAMP` | Precision differences possible |

Delta table features enabled by LFC on UC streaming tables: `changeDataFeed`, `deletionVectors`, `rowTracking`, `columnMapping` (name mode), `typeWidening`.

## Compute infrastructure per update

### Getting cluster info from an update

```bash
databricks -p $PROFILE api get "/api/2.0/pipelines/<pid>/updates/<uid>" \
  | jq '{cluster_id: .update.cluster_id,
         serverless:  .update.config.serverless,
         photon:      .update.config.photon,
         edition:     .update.config.edition,
         start:       .update.start_time,
         end:         .update.end_time}'
```

`cluster_id` is always present. `serverless: null` in the top-level pipeline spec does NOT mean classic — check `.update.config.serverless` for the effective value per run.

### Serverless cluster characteristics

For serverless pipeline clusters (suffix `-v2n`):
- `node_type_id: null` — instance type abstracted
- `num_workers: null` — auto-scaled
- `cluster_cores: null` — not exposed
- Architecture: `aarch64` (ARM nodes)
- One **fresh cluster per update** — no reuse between runs
- Cluster name pattern: `dlt-execution-<pipeline_id>`
- Cluster ID pattern: `MMDD-HHMMSS-<random>-v2n`

### Infrastructure timeline per run

```
total cluster lifetime = bootstrap + work + wind-down
```

| Phase | How to compute | Typical range |
|---|---|---|
| bootstrap | `cluster.start_time → update.start_time` | 65–400s |
| work | `update.start_time → update.end_time` | 6s–11min |
| wind-down | `update.end_time → cluster.terminated_time` | 17–34s |

- **Small/empty runs**: 3–16% work efficiency (bootstrap dominates)
- **100M row runs**: 70–82% work efficiency
- **Wind-down is consistent**: ~20s regardless of run size
- **Bootstrap variance**: driven by serverless pool availability, not data volume; concurrent runs inflate it

### When `serverless: null` in spec but actually serverless

The top-level pipeline spec can show `serverless: null` while every run uses serverless compute. The ground truth is `.update.config.serverless` in the update detail response. The `-v2n` cluster ID suffix is also a reliable serverless indicator.

## Compute resource metrics — system tables

CPU, RAM, network, and disk usage for every pipeline cluster are stored in `system.compute.node_timeline`. The table covers **both classic and serverless clusters** (serverless cluster IDs with the `-v2n` suffix appear normally).

### Ingestion lag

System tables have an ingestion lag of **5–7 days**. Use this for post-hoc analysis, not real-time monitoring. There is no supported API for real-time CPU/RAM/disk access on serverless clusters (no Ganglia, no public driver DNS).

### Table: `system.compute.node_timeline`

Granularity: **1-minute intervals**, one row per node per minute.

| Column | Type | Meaning |
|--------|------|---------|
| `cluster_id` | string | Matches the update's `cluster_id` |
| `instance_id` | string | Unique VM instance |
| `start_time` / `end_time` | timestamp | 1-minute window |
| `driver` | boolean | `true` = driver node |
| `node_type` | string | Cloud VM type (e.g. `Standard_D8as_v5`) |
| `cpu_user_percent` | double | CPU in userland |
| `cpu_system_percent` | double | CPU in kernel |
| `cpu_wait_percent` | double | CPU blocked on I/O |
| `mem_used_percent` | double | Node memory used |
| `mem_swap_percent` | double | Swap in use |
| `network_sent_bytes` | bigint | Outbound bytes in window |
| `network_received_bytes` | bigint | Inbound bytes in window |
| `disk_free_bytes_per_mount_point` | map<string,bigint> | Free bytes per mount |
| `spark_jvm_heap_usage_bytes` | bigint | JVM heap used by Spark |
| `spark_jvm_heap_limit_bytes` | bigint | Max JVM heap available |
| `spark_container_memory_used_bytes` | bigint | Non-reclaimable container memory |
| `spark_container_memory_file_cache_bytes` | bigint | Reclaimable file cache in container |

### Table: `system.lakeflow.pipeline_update_timeline`

The bridge between a pipeline update and its cluster:

| Column | Meaning |
|--------|---------|
| `pipeline_id` | Pipeline UUID |
| `update_id` | Update UUID |
| `compute.cluster_id` | Cluster used for this update |
| `compute.type` | `CLASSIC_COMPUTE` or `SERVERLESS` |
| `period_start_time` | Update start (UTC) |
| `period_end_time` | Update end (UTC) |
| `result_state` | `SUCCEEDED`, `FAILED`, `CANCELED` |
| `trigger_type` | `JOB_TASK`, `USER_ACTION`, etc. |

### Correlation query: pipeline run → resource metrics

```sql
WITH pipeline_runs AS (
  SELECT
    pipeline_id,
    update_id,
    result_state,
    compute.cluster_id   AS cluster_id,
    compute.type         AS compute_type,
    period_start_time    AS run_start,
    period_end_time      AS run_end
  FROM system.lakeflow.pipeline_update_timeline
  WHERE pipeline_id IN ('<pipeline_id_1>', '<pipeline_id_2>')
),
node_metrics AS (
  SELECT
    pr.pipeline_id,
    pr.update_id,
    pr.result_state,
    nt.instance_id,
    nt.driver,
    nt.node_type,
    nt.start_time                                      AS minute,
    ROUND(nt.cpu_user_percent + nt.cpu_system_percent, 1)  AS cpu_active_pct,
    ROUND(nt.cpu_wait_percent, 1)                      AS cpu_io_wait_pct,
    ROUND(nt.mem_used_percent, 1)                      AS mem_pct,
    ROUND(nt.network_sent_bytes     / 1048576.0, 2)    AS net_sent_mb,
    ROUND(nt.network_received_bytes / 1048576.0, 2)    AS net_recv_mb,
    ROUND(nt.spark_jvm_heap_usage_bytes / 1073741824.0, 2) AS jvm_heap_gb,
    ROUND(nt.spark_jvm_heap_limit_bytes / 1073741824.0, 2) AS jvm_heap_limit_gb,
    ROUND(nt.spark_container_memory_used_bytes / 1073741824.0, 2) AS container_mem_gb
  FROM system.compute.node_timeline nt
  JOIN pipeline_runs pr
    ON nt.cluster_id = pr.cluster_id
   AND nt.start_time BETWEEN pr.run_start AND pr.run_end
)
SELECT * FROM node_metrics
ORDER BY pipeline_id, update_id, minute, driver DESC, instance_id
```

### Per-run aggregate (bottleneck fingerprinting)

```sql
WITH pipeline_runs AS (
  SELECT pipeline_id, update_id, compute.cluster_id AS cluster_id,
         period_start_time AS run_start, period_end_time AS run_end
  FROM system.lakeflow.pipeline_update_timeline
  WHERE pipeline_id = '<pipeline_id>'
),
agg AS (
  SELECT
    pr.update_id,
    nt.driver,
    nt.node_type,
    COUNT(DISTINCT nt.instance_id)                         AS node_count,
    ROUND(AVG(nt.cpu_user_percent + nt.cpu_system_percent), 1) AS avg_cpu_pct,
    ROUND(MAX(nt.cpu_user_percent + nt.cpu_system_percent), 1) AS peak_cpu_pct,
    ROUND(AVG(nt.mem_used_percent), 1)                     AS avg_mem_pct,
    ROUND(MAX(nt.mem_used_percent), 1)                     AS peak_mem_pct,
    ROUND(SUM(nt.network_received_bytes) / 1048576.0, 1)   AS total_net_recv_mb,
    ROUND(SUM(nt.network_sent_bytes)     / 1048576.0, 1)   AS total_net_sent_mb,
    ROUND(AVG(nt.cpu_wait_percent), 1)                     AS avg_io_wait_pct
  FROM system.compute.node_timeline nt
  JOIN pipeline_runs pr
    ON nt.cluster_id = pr.cluster_id
   AND nt.start_time BETWEEN pr.run_start AND pr.run_end
  GROUP BY pr.update_id, nt.driver, nt.node_type
)
SELECT * FROM agg ORDER BY update_id, driver DESC
```

### Interpreting resource signals

| Symptom | Likely cause |
|---------|--------------|
| `cpu_active_pct` > 80% on driver | JDBC read saturation or merge computation |
| `cpu_io_wait_pct` > 20% | Disk spill or slow remote storage writes |
| `net_recv_mb` >> `net_sent_mb` on driver | Expected: JDBC data ingestion (reading source) |
| `net_sent_mb` >> `net_recv_mb` on driver | Data shuffle to workers (unexpected for single-table LFC) |
| `mem_used_percent` > 90% | Risk of spill or OOM; reduce batch size or upgrade node type |
| `jvm_heap_gb` near `jvm_heap_limit_gb` | JVM pressure; may cause GC pauses → throughput jitter |
| `container_mem_gb` high, `jvm_heap_gb` low | Off-heap usage (Arrow, Photon native) |

### Getting cluster_id without system tables (if within 7-day lag window)

Retrieve the cluster ID from the updates API directly:

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pid>/updates/<uid>" \
  | jq '.update.cluster_id'
```

Then query `node_timeline` with that cluster ID and the time window from the update response (`start_time` / `end_time` fields).

## CDC cleanup after pipeline delete

LFC creates PostgreSQL replication slots named `dbx_%_<GATEWAY_PIPELINE_ID>`.

```sql
-- drop after pipeline deletion
SELECT pg_drop_replication_slot(slot_name)
  FROM pg_replication_slots
  WHERE slot_name LIKE 'dbx_%_<gateway_pipeline_id>';
```

MySQL binlog-based CDC: no explicit slot to drop, but confirm `endless_dml_loop` stored procedure is stopped if it was running.

## Notebook pattern for pipeline analysis

**Prefer a Databricks workspace notebook over a local canvas or script** for pipeline analysis. The notebook runs on serverless compute with direct access to the Workspace API, Unity Catalog system tables, and `display()` for rich interactive output. It is shareable, schedulable, and reproducible.

### Notebook structure (one cell per section)

| Cell | Purpose | Magic / language |
|------|---------|-----------------|
| 1 | Widgets: pipeline IDs, date range | `%python` + `dbutils.widgets` |
| 2 | SDK setup + fetch updates | `%python` — `WorkspaceClient` |
| 3 | Fetch events → rows per run | `%python` — `list_pipeline_events()` |
| 4 | Run history table | `display(df)` |
| 5 | Throughput / bytes charts | `display(df)` (click chart icon) |
| 6 | System tables: update timeline | `%sql` |
| 7 | System tables: node_timeline join | `%sql` |
| 8 | Aggregate resource metrics | `display(df)` |

### Cell 1 — widgets

```python
dbutils.widgets.text("pipeline_ids", "id1,id2", "Pipeline IDs (comma-separated)")
dbutils.widgets.text("lookback_days", "7", "Lookback days")
pipeline_ids = [p.strip() for p in dbutils.widgets.get("pipeline_ids").split(",")]
lookback_days = int(dbutils.widgets.get("lookback_days"))
```

### Cell 2 — SDK setup + update list

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service import pipelines
import pandas as pd
from datetime import datetime, timezone, timedelta

w = WorkspaceClient()

rows = []
for pid in pipeline_ids:
    for upd in w.pipelines.list_updates(pipeline_id=pid).updates or []:
        rows.append({
            "pipeline_id": pid,
            "update_id":   upd.update_id,
            "state":       upd.state.value if upd.state else None,
            "cluster_id":  upd.cluster_id,
            "start_time":  upd.start_time,
            "end_time":    upd.end_time,
        })

updates_df = pd.DataFrame(rows)
display(updates_df)
```

### Cell 3 — rows ingested per update (from events)

```python
event_rows = []
for pid in pipeline_ids:
    for upd_id in updates_df["update_id"].tolist():
        for evt in (w.pipelines.list_pipeline_events(
                pipeline_id=pid,
                filter=f"update_id='{upd_id}' AND event_type='flow_progress'"
            ).events or []):
            m = (evt.details.flow_progress.metrics if evt.details
                 and evt.details.flow_progress else None)
            if m and (m.num_upserted_rows or 0) > 0:
                event_rows.append({
                    "pipeline_id": pid,
                    "update_id":   upd_id,
                    "table":       evt.message.split("'")[1].split(".")[-1] if evt.message else "",
                    "rows":        m.num_upserted_rows,
                    "ts":          evt.timestamp,
                })

events_df = pd.DataFrame(event_rows)
display(events_df)
```

### Cell 4 — run history with throughput (SQL on system tables)

```sql
%sql
SELECT
  pipeline_id,
  update_id,
  result_state,
  compute.type            AS compute_type,
  compute.cluster_id      AS cluster_id,
  period_start_time,
  period_end_time,
  ROUND((unix_timestamp(period_end_time) - unix_timestamp(period_start_time)) / 60.0, 1) AS duration_min
FROM system.lakeflow.pipeline_update_timeline
WHERE pipeline_id IN ('${pipeline_ids}')   -- widget value, comma-separated
  AND period_start_time >= current_timestamp() - INTERVAL ${lookback_days} DAYS
ORDER BY period_start_time DESC
```

`display()` is implicit for `%sql` cells — click the chart icon to switch between table and bar chart views.

### Cell 5 — resource metrics per run (node_timeline join)

```sql
%sql
WITH runs AS (
  SELECT pipeline_id, update_id, compute.cluster_id AS cluster_id,
         period_start_time AS run_start, period_end_time AS run_end
  FROM system.lakeflow.pipeline_update_timeline
  WHERE pipeline_id IN ('${pipeline_ids}')
    AND period_start_time >= current_timestamp() - INTERVAL ${lookback_days} DAYS
)
SELECT
  r.pipeline_id,
  r.update_id,
  nt.driver,
  nt.node_type,
  COUNT(DISTINCT nt.instance_id)                                AS node_count,
  ROUND(AVG(nt.cpu_user_percent + nt.cpu_system_percent), 1)    AS avg_cpu_pct,
  ROUND(MAX(nt.cpu_user_percent + nt.cpu_system_percent), 1)    AS peak_cpu_pct,
  ROUND(AVG(nt.mem_used_percent), 1)                            AS avg_mem_pct,
  ROUND(SUM(nt.network_received_bytes) / 1048576.0, 1)          AS net_recv_mb,
  ROUND(SUM(nt.network_sent_bytes)     / 1048576.0, 1)          AS net_sent_mb,
  ROUND(AVG(nt.cpu_wait_percent), 1)                            AS avg_io_wait_pct
FROM system.compute.node_timeline nt
JOIN runs r
  ON nt.cluster_id = r.cluster_id
 AND nt.start_time BETWEEN r.run_start AND r.run_end
GROUP BY r.pipeline_id, r.update_id, nt.driver, nt.node_type
ORDER BY r.pipeline_id, r.update_id, nt.driver DESC
```

### Cell 6 — Delta bytes per commit

```python
delta_sql = """
  SELECT version, timestamp,
    CAST(operationMetrics:numTargetBytesAdded  AS BIGINT) AS bytes_added,
    CAST(operationMetrics:numTargetRowsInserted AS BIGINT) AS rows_inserted,
    CAST(operationMetrics:executionTimeMs       AS BIGINT) AS exec_ms
  FROM (DESCRIBE HISTORY {catalog}.{schema}.{table})
  WHERE operation = 'MERGE'
  ORDER BY version DESC
  LIMIT 20
"""
df = spark.sql(delta_sql.format(catalog="cat", schema="sch", table="tbl"))
display(df)
```

### `display()` chart tips

- `display(df)` with a DataFrame shows a table; click the **+** or chart icon to render as bar, line, or scatter.
- For explicit chart config, use `display(df)` and then Databricks auto-suggests visualizations based on column types.
- For `%sql` cells, `display()` is automatic — set the visualization type in the cell output controls.
- Computed columns (e.g. `avg_cpu_pct`, `rows_per_sec`) work directly as chart axes without extra transformation.

### Why notebook over canvas

| Concern | Canvas | Databricks notebook |
|---------|--------|---------------------|
| Data freshness | Embed inline at write time | Live — re-run to refresh |
| Workspace API access | CLI subprocess calls | Native SDK (`WorkspaceClient`) |
| System table queries | SQL Statement API via subprocess | `%sql` or `spark.sql()` directly |
| Parameterization | Hardcoded constants | `dbutils.widgets` |
| Sharing | File in `.cursor/projects/` | Published notebook URL or job |

---

## Comparison testing — create separate identifiers per dimension

When testing different configurations side by side (routing paths, DB engines, network modes, compute tiers, etc.), **assign a distinct identifier at every layer** so metrics stay clean and auto-detection works without extra arguments.

### Why separate identifiers

- `performance_schema.status_by_account` (MySQL) buckets bytes by `USER` — a shared user conflates both paths.
- The pipeline spec stores exactly one `connection_name` — the connection name must encode the configuration.
- `run-logs/byte-capture.jsonl` records `conn_name` and `routing` — grouping by those fields gives per-configuration trends with no post-hoc labelling.
- Concurrent runs of two pipelines against different endpoints produce clean, non-interleaving byte counters only when the users are distinct.

### Identifier stack per comparison axis

For each variable dimension (routing, DB engine, etc.) create one dedicated identifier at each layer:

| Layer | Public path | PNG path | Why |
|-------|-------------|----------|-----|
| MySQL user | `lfc_user_public` | `lfc_user_png` | `performance_schema` buckets bytes by USER |
| UC connection | `<prefix>-<key>-public` | `<prefix>-<key>-png` | Pipeline spec stores one connection name; suffix encodes routing |
| Connection comment | `{"secrets":{"scope":"…","key":"…"}}` | same format | `databricks-run-with-byte-capture.sh` reads this to auto-detect the secret |
| Databricks secret | `<key>` (shared) | same | Single secret stores all routing user credentials under `routing.public.*` / `routing.png.*` |
| LakeFlow pipeline | `<prefix>-<key>-qbc-public` | `<prefix>-<key>-qbc-png` | One pipeline per connection; system tables correlate by pipeline ID |
| UC schema (destination) | `db_ingest_public` | `db_ingest_png` | Separate Delta tables per path; prevents MERGE conflicts on concurrent runs |

### Naming convention rule

> **Always encode the variable dimension as a suffix in the name**, using a consistent vocabulary:
> - routing path: `-public` / `-png`
> - mode: `-qbc` / `-cdc`
> - other axes: add a new suffix token (e.g. `-tls` / `-notls`, `-classic` / `-serverless`)

The `databricks-run-with-byte-capture.sh` script derives routing from the `-public` / `-png` suffix of the connection name automatically. Any new suffix token requires a corresponding case in `_routing_from_conn_name`.

### Auto-detection chain (how identifiers flow at run time)

```
./databricks-run-with-byte-capture.sh <PIPELINE_ID>
  │
  ├─ GET /api/2.0/pipelines/<id>
  │    → spec.ingestion_definition.connection_name  (e.g. "robert-lee-vm-mysql-png")
  │
  ├─ suffix "-png" → routing = "png"
  │
  ├─ GET /api/2.1/unity-catalog/connections/robert-lee-vm-mysql-png
  │    → comment = {"secrets":{"scope":"png-testing","key":"vm-mysql"}}
  │
  ├─ load_db_secret "vm-mysql"
  │    → routing__png__user     = "lfc_user_png"
  │    → routing__png__password = "…"
  │    → host_fqdn, dba__user, dba__password, …
  │
  └─ snapshot performance_schema WHERE USER = 'lfc_user_png'
       → bytes are clean for this path only
```

No arguments beyond the pipeline ID are needed because the pipeline encodes its own configuration.

### Adding a new comparison dimension

When you add a new variable (e.g. TLS on vs off):

1. **MySQL**: create `lfc_user_tls` / `lfc_user_notls` (same grants as `lfc_user`).
2. **Secret**: save their passwords under `routing.tls.*` / `routing.notls.*` in the existing secret.
3. **UC connections**: create `<prefix>-<key>-tls` and `<prefix>-<key>-notls`.
4. **Pipelines**: create `<prefix>-<key>-qbc-tls` and `<prefix>-<key>-qbc-notls`.
5. **`_routing_from_conn_name`**: add `*-tls) echo "tls"` / `*-notls) echo "notls"` cases.
6. **`databricks-run-with-byte-capture.sh`**: add the new routing token to the `lfc_user` selection block.

### When a single user is acceptable

Use a shared `lfc_user` (no routing split) only when:
- Runs are strictly sequential (never concurrent), AND
- You don't need per-path byte isolation, AND
- The dimension being compared is not the routing path (e.g. comparing two DB engines, not two network paths)

Even then, encode the dimension in the connection name suffix so the auto-detection chain remains intact.

---

## SQL Statement API vs pipeline events API — choosing the right tool

### SQL Statement API requires a running warehouse

The SQL Statement API (`POST /api/2.0/sql/statements`) executes against a SQL warehouse. If the warehouse is `STOPPED`, the API wakes it up — which takes **2–3 minutes**. The practical `wait_timeout` ceiling is 30s, so the call times out before the warehouse is ready:

```
HTTP 200: {"status": {"state": "PENDING"}}   ← warehouse still starting
HTTP 200: {"status": {"state": "RUNNING"}}   ← query running but not done yet
...timeout after 30s...
```

**Workaround options**:
1. Poll the statement ID in a loop until state = `SUCCEEDED` (total wait can be 3–5 min).
2. Start the warehouse first: `databricks warehouses start --id <wh_id> --wait`.
3. **Use the pipeline events API instead** — it never requires a warehouse.

### When to use each

| Need | Use | Why |
|------|-----|-----|
| Update durations, state transitions, flow timelines | Pipeline events API | Always available, no warehouse |
| Which tables (flows) ran in each update | Pipeline events API (`flow_progress`) | Always available, no warehouse |
| Cross-update trend analysis, node_timeline join | SQL Statement API (`system.*` tables) | Events API can't join system tables |
| Real-time monitoring during a run | Pipeline events API | Low latency |
| Historical resource metrics (CPU/RAM/disk) | SQL Statement API + `system.compute.node_timeline` | Only source; 5–7 day lag |

**Default**: when a warehouse is stopped and the answer can come from pipeline events, always use the events API. Only fall back to the SQL Statement API when system table data is strictly required.

---

## Identifying update_id from a benchmark record

When a pipeline has been run many times and you need to find which `update_id` corresponds to a specific benchmark row (e.g. "the 10M row run that took 67 seconds"), use this three-step procedure:

### Step 1 — list all completed updates

```bash
databricks -p $PROFILE pipelines list-updates <pipeline_id> \
  --max-results 20 --output json \
  | jq '[.updates[] | select(.state == "COMPLETED") | {update_id, state, creation_time, cause}]'
```

If you have more updates than fit in max_results, reduce the search space first using `creation_time` (milliseconds since epoch) to narrow to the date range of the benchmark.

### Step 2 — determine which tables ran in each update

Pull `flow_progress` COMPLETED events. The flow name encodes the table name. Each update lists only the flows that actually ran (so updates before a table was added will not show that flow):

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200" \
  | jq '[.events[] | select(.details.flow_progress != null and .details.flow_progress.status == "COMPLETED") | {
      update_id: .origin.update_id,
      flow: .origin.flow_name,      # table name is the last segment after "."
      ts: .timestamp
    }] | sort_by(.ts)'
```

The **largest table present** (by its flow name suffix) characterizes the run:
- only `_intpk_upsert` → 1M row run
- `_intpk_upsert` + `_intpk_10m_upsert` → 10M row run
- all three (+ `_intpk_100m_upsert`) → 100M row run

If older updates fall off page 1, paginate using `next_page_token` (see Events API pagination above).

### Step 3 — match by duration (RUNNING → last flow COMPLETED)

Get the exact duration from `update_progress` events:

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200" \
  | jq '[.events[] | select(.event_type == "update_progress") | {
      update_id: .origin.update_id, state: .details.update_progress.state, ts: .timestamp
    }] | group_by(.update_id) | map({
      update_id: .[0].update_id,
      events: map({state:.state,ts:.ts}) | sort_by(.ts)
    })'
```

Compute `RUNNING → COMPLETED` duration for each update and compare to the `time` column in your benchmark record. Differences of ±5s are normal (manual measurement vs API timestamp rounding).

### Combining step 2 + step 3 matching rule

> The correct `update_id` is the one where:
> 1. The flow names match the expected table size(s), AND
> 2. The RUNNING → COMPLETED duration matches the recorded `time` value (within ±5s)

Both conditions must hold. If only duration matches but wrong flows, it is a coincidental match.

---



When an incremental ("nothing changed") run is unexpectedly slow, there are four possible bottlenecks. Always measure before assuming.

### Step 1: measure actual phase durations

```bash
# Get cluster start, update start→RUNNING→COMPLETED events, cluster end
PIPELINE_ID="<id>"
UPDATE_ID="<uid>"
PROFILE="<profile>"

# cluster lifetime
databricks -p $PROFILE api get \
  "/api/2.0/clusters/get?cluster_id=$(databricks -p $PROFILE api get \
    "/api/2.0/pipelines/$PIPELINE_ID/updates/$UPDATE_ID" \
    | jq -r '.update.cluster_id')" \
  | jq '{start_time,terminated_time,state}'

# event timeline (INITIALIZING → RUNNING → COMPLETED per-flow)
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/$PIPELINE_ID/events?max_results=50" \
  | jq '[.events[] | select(.origin.update_id=="'"$UPDATE_ID"'")
         | {ts:.timestamp[11:19], type:.event_type, msg:(.message//""[0:80])}]'
```

Phase breakdown from the events:

| Phase | Duration = | Expected (empty run) | Signals a problem |
|-------|-----------|----------------------|-------------------|
| Bootstrap | cluster_start → RUNNING event | 60–400s | >400s = serverless pool contention |
| Work (per table) | RUNNING → last flow COMPLETED | 1–5s per table | >30s per table = source scan issue |
| Wind-down | last flow COMPLETED → cluster_end | 15–30s | never the bottleneck |

### Step 2: triage by bottleneck

#### Source bottleneck (MySQL / database)

**Signal**: Bootstrap fast, work phase slow, `Bytes_sent` delta is small (< 1 MB) but wall time is high.

The byte capture script (`databricks-run-with-byte-capture.sh`) gives the direct measurement:
- Large `Bytes_sent` delta → MySQL is sending real data (data volume issue or catching up).
- Tiny `Bytes_sent` delta (< 100 KB) but slow work phase → MySQL is doing a **full table scan** and returning 0 rows.

Root causes and fixes:

| Root cause | How to confirm | Fix |
|-----------|---------------|-----|
| Missing index on QBC cursor column (`dt`) | `SHOW INDEX FROM <table>` — no `idx_dt` | `ALTER TABLE <tbl> ADD INDEX idx_dt (dt)` |
| Full table scan on empty check | `EXPLAIN SELECT * FROM intpk WHERE dt > '...' ORDER BY dt LIMIT 1000` — shows `type=ALL` | Add the index |
| Slow query on large table | MySQL slow query log, `SHOW PROCESSLIST` during the run | Add index, or shard the table |
| Azure MySQL IOPS throttled | Azure portal → MySQL metrics → IO percent | Scale up compute tier |

**Expected scan times without `idx_dt`**:

| Row count | Empty-check wall time |
|-----------|----------------------|
| 100K | ~1s |
| 1M | ~50s |
| 10M | ~450s |
| 100M | ~75 min |

After adding the index, all of the above drop to < 1s.

**Verify the index is used**:
```sql
-- Should show type=ref or range, NOT type=ALL
EXPLAIN SELECT * FROM intpk WHERE dt > '2026-01-01 00:00:00' ORDER BY dt LIMIT 1000;
```

#### Network bottleneck (public routing vs PNG)

**Signal**: Bootstrap fast, work phase has high `Bytes_sent` delta but low throughput (MB/s).

```bash
# Compare two runs: each pipeline uses its own connection (-public / -png).
# Secret key and routing user are auto-detected from the connection name.
./databricks-run-with-byte-capture.sh $PIPELINE_PUBLIC_ID
./databricks-run-with-byte-capture.sh $PIPELINE_PNG_ID
# Compare MB/s between the two run-logs/byte-capture.jsonl entries
```

Check `run-logs/byte-capture.jsonl` for throughput trend across runs. Degrading MB/s on PNG but not public → PNG routing or private endpoint issue.

#### Target bottleneck (Databricks / Delta write)

**Signal**: `Bytes_sent` delta is large (actual data flowing), but pipeline takes longer than prior runs with same volume.

Check Delta HISTORY for the target table:
```sql
-- num target bytes added and write duration
DESCRIBE HISTORY <catalog>.<schema>.<table>
-- look at operationMetrics.executionTimeMs and numTargetBytesAdded
```

Cross-check with `system.compute.node_timeline` (5–7 day lag):
```sql
-- high cpu_io_wait_percent → disk spill or slow Delta write
SELECT start_time, ROUND(cpu_io_wait_percent,1) AS io_wait_pct,
       ROUND(mem_used_percent,1) AS mem_pct
FROM system.compute.node_timeline
WHERE cluster_id = '<cluster_id>'
ORDER BY start_time
```

#### Serverless bootstrap contention

**Signal**: Bootstrap duration > 300s. Work phase is fast once RUNNING.

```bash
# Check cluster start vs update start vs RUNNING event
# If cluster_start → RUNNING > 5 min → pool contention, not a code issue
```

Bootstrap latency varies by time of day and workspace load. For recurring jobs, consider a continuous pipeline or scheduling at off-peak hours.

### Decision tree summary

```
Run slower than baseline?
├── Is bootstrap > 5 min? → serverless pool contention (not source/network/target)
└── Is work phase slow?
    ├── Bytes_sent delta < 100 KB?
    │   └── MySQL doing full scan, returning 0 rows → MISSING INDEX on dt column
    ├── Bytes_sent delta large, throughput low?
    │   └── Network bottleneck → compare public vs PNG routing
    └── Bytes_sent delta large, throughput OK, Delta write slow?
        └── Target bottleneck → check DESCRIBE HISTORY + node_timeline io_wait
```

### Saving run metrics for trend analysis

Each run appended by `databricks-run-with-byte-capture.sh` to `run-logs/byte-capture.jsonl`:
```jsonc
{"ts":"2026-04-30T15:57:13Z","pipeline_id":"79d81b30-...","update_id":"f7935a-...",
 "conn_name":"robert-lee-vm-mysql-png","routing":"png","secret_key":"vm-mysql",
 "db_host":"...internal.cloudapp.net","mysql_user":"lfc_user_png",
 "wall_s":626,"bytes_sent":33801,"bytes_recv":34473,"mb_per_s":0.0001,"state":"COMPLETED"}
```

The pipeline `description` field is also updated with the last-run summary via `PUT /api/2.0/pipelines/<id>` — visible in the Databricks UI pipeline detail page.

To analyze trends across routing paths in a notebook:
```python
import json, pandas as pd
rows = [json.loads(l) for l in open("run-logs/byte-capture.jsonl")]
df = pd.DataFrame(rows)
df["ts"] = pd.to_datetime(df["ts"])
# side-by-side comparison by routing path
display(df[["ts","routing","conn_name","wall_s","bytes_sent","mb_per_s","state"]]
          .sort_values(["routing","ts"]))
```
| Scheduling | Not schedulable | Wrap in a Job for recurring reports |
