# Compute Resource Metrics — System Tables

## system.compute.node_timeline

Granularity: **1-minute intervals**, one row per node per minute. 5–7 day ingestion lag.

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
| `spark_container_memory_file_cache_bytes` | bigint | Reclaimable file cache |

## system.lakeflow.pipeline_update_timeline

Bridge between a pipeline update and its cluster:

| Column | Meaning |
|--------|---------|
| `pipeline_id` | Pipeline UUID |
| `update_id` | Update UUID |
| `compute.cluster_id` | Cluster used for this update |
| `compute.type` | `CLASSIC_COMPUTE` or `SERVERLESS` |
| `period_start_time` | Update start (UTC) |
| `period_end_time` | Update end (UTC) |
| `result_state` | `SUCCEEDED`, `FAILED`, `CANCELED` |
| `trigger_type` | `JOB_TASK`, `USER_ACTION`, `API_CALL`, etc. |

## Getting cluster_id without system tables

Within the 7-day window where system tables haven't been populated yet, retrieve cluster_id from the updates API directly:

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pid>/updates/<uid>" \
  | jq '.update.cluster_id'
```

## Correlation query: pipeline run → resource metrics

```sql
WITH pipeline_runs AS (
  SELECT
    pipeline_id, update_id, result_state,
    compute.cluster_id   AS cluster_id,
    compute.type         AS compute_type,
    period_start_time    AS run_start,
    period_end_time      AS run_end
  FROM system.lakeflow.pipeline_update_timeline
  WHERE pipeline_id IN ('<pipeline_id_1>', '<pipeline_id_2>')
),
node_metrics AS (
  SELECT
    pr.pipeline_id, pr.update_id, pr.result_state,
    nt.instance_id, nt.driver, nt.node_type,
    nt.start_time                                          AS minute,
    ROUND(nt.cpu_user_percent + nt.cpu_system_percent, 1) AS cpu_active_pct,
    ROUND(nt.cpu_wait_percent, 1)                         AS cpu_io_wait_pct,
    ROUND(nt.mem_used_percent, 1)                         AS mem_pct,
    ROUND(nt.network_sent_bytes     / 1048576.0, 2)       AS net_sent_mb,
    ROUND(nt.network_received_bytes / 1048576.0, 2)       AS net_recv_mb,
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

## Per-run aggregate (bottleneck fingerprinting)

```sql
WITH pipeline_runs AS (
  SELECT pipeline_id, update_id, compute.cluster_id AS cluster_id,
         period_start_time AS run_start, period_end_time AS run_end
  FROM system.lakeflow.pipeline_update_timeline
  WHERE pipeline_id = '<pipeline_id>'
)
SELECT
  pr.update_id, nt.driver, nt.node_type,
  COUNT(DISTINCT nt.instance_id)                          AS node_count,
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
ORDER BY pr.update_id, nt.driver DESC
```

## Interpreting resource signals

| Symptom | Likely cause |
|---------|--------------|
| `cpu_active_pct` > 80% on driver | JDBC read saturation or merge computation |
| `cpu_io_wait_pct` > 20% | Disk spill or slow remote storage writes |
| `net_recv_mb` >> `net_sent_mb` on driver | Expected: JDBC data ingestion (reading source) |
| `net_sent_mb` >> `net_recv_mb` on driver | Data shuffle to workers (unexpected for single-table LFC) |
| `mem_used_percent` > 90% | Risk of spill or OOM |
| `jvm_heap_gb` near `jvm_heap_limit_gb` | JVM pressure; may cause GC pauses → throughput jitter |
| `container_mem_gb` high, `jvm_heap_gb` low | Off-heap usage (Arrow, Photon native) |
