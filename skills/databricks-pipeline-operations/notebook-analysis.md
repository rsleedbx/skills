# Notebook Pattern for Pipeline Analysis

**Prefer a Databricks workspace notebook over a local canvas or script** — live data, native WorkspaceClient SDK, `display()` for rich output, shareable, schedulable, reproducible.

## Notebook structure

| Cell | Purpose | Magic / language |
|------|---------|-----------------|
| 1 | Widgets: pipeline IDs, date range | `%python` + `dbutils.widgets` |
| 2 | SDK setup + fetch updates | `%python` — `WorkspaceClient` |
| 3 | Fetch events → rows per run | `%python` — `list_pipeline_events()` |
| 4 | Run history table | `%sql` — `system.lakeflow.pipeline_update_timeline` |
| 5 | Resource metrics per run | `%sql` — `node_timeline` join |
| 6 | Delta bytes per commit | `%python` — `DESCRIBE HISTORY` |

## Cell 1 — widgets

```python
dbutils.widgets.text("pipeline_ids", "id1,id2", "Pipeline IDs (comma-separated)")
dbutils.widgets.text("lookback_days", "7", "Lookback days")
pipeline_ids = [p.strip() for p in dbutils.widgets.get("pipeline_ids").split(",")]
lookback_days = int(dbutils.widgets.get("lookback_days"))
```

## Cell 2 — SDK setup + update list

```python
from databricks.sdk import WorkspaceClient
import pandas as pd

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

## Cell 3 — rows ingested per update (from events)

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

## Cell 4 — run history with throughput (SQL on system tables)

```sql
%sql
SELECT
  pipeline_id, update_id, result_state,
  compute.type            AS compute_type,
  compute.cluster_id      AS cluster_id,
  period_start_time, period_end_time,
  ROUND((unix_timestamp(period_end_time) - unix_timestamp(period_start_time)) / 60.0, 1) AS duration_min
FROM system.lakeflow.pipeline_update_timeline
WHERE pipeline_id IN ('${pipeline_ids}')
  AND period_start_time >= current_timestamp() - INTERVAL ${lookback_days} DAYS
ORDER BY period_start_time DESC
```

## Cell 5 — resource metrics per run (node_timeline join)

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
  r.pipeline_id, r.update_id, nt.driver, nt.node_type,
  COUNT(DISTINCT nt.instance_id)                                AS node_count,
  ROUND(AVG(nt.cpu_user_percent + nt.cpu_system_percent), 1)   AS avg_cpu_pct,
  ROUND(MAX(nt.cpu_user_percent + nt.cpu_system_percent), 1)   AS peak_cpu_pct,
  ROUND(AVG(nt.mem_used_percent), 1)                           AS avg_mem_pct,
  ROUND(SUM(nt.network_received_bytes) / 1048576.0, 1)         AS net_recv_mb,
  ROUND(SUM(nt.network_sent_bytes)     / 1048576.0, 1)         AS net_sent_mb,
  ROUND(AVG(nt.cpu_wait_percent), 1)                           AS avg_io_wait_pct
FROM system.compute.node_timeline nt
JOIN runs r
  ON nt.cluster_id = r.cluster_id
 AND nt.start_time BETWEEN r.run_start AND r.run_end
GROUP BY r.pipeline_id, r.update_id, nt.driver, nt.node_type
ORDER BY r.pipeline_id, r.update_id, nt.driver DESC
```

## Cell 6 — Delta bytes per commit

```python
delta_sql = """
  SELECT version, timestamp,
    CAST(operationMetrics:numTargetBytesAdded  AS BIGINT) AS bytes_added,
    CAST(operationMetrics:numTargetRowsInserted AS BIGINT) AS rows_inserted,
    CAST(operationMetrics:executionTimeMs       AS BIGINT) AS exec_ms
  FROM (DESCRIBE HISTORY {catalog}.{schema}.{table})
  WHERE operation = 'MERGE'
  ORDER BY version DESC LIMIT 20
"""
df = spark.sql(delta_sql.format(catalog="cat", schema="sch", table="tbl"))
display(df)
```

## display() chart tips

- `display(df)` shows a table; click **+** or the chart icon to render as bar, line, or scatter.
- For `%sql` cells, `display()` is automatic — set the visualization type in the cell output controls.
- Computed columns (e.g. `avg_cpu_pct`, `rows_per_sec`) work directly as chart axes.

## Why notebook over canvas

| Concern | Canvas | Databricks notebook |
|---------|--------|---------------------|
| Data freshness | Embed inline at write time | Live — re-run to refresh |
| Workspace API access | CLI subprocess calls | Native SDK (`WorkspaceClient`) |
| System table queries | SQL Statement API via subprocess | `%sql` or `spark.sql()` directly |
| Parameterization | Hardcoded constants | `dbutils.widgets` |
| Sharing | File in `.cursor/projects/` | Published notebook URL or job |
