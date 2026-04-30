# Latency Triage and Update ID Matching

## When a run takes longer than usual

### Step 1: measure actual phase durations

```bash
PIPELINE_ID="<id>"
UPDATE_ID="<uid>"
PROFILE="<profile>"

# Update_progress event timeline for one update
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/$PIPELINE_ID/events?max_results=200" \
  | jq '[.events[] | select(.origin.update_id=="'"$UPDATE_ID"'" and .event_type=="update_progress")
         | {ts:.timestamp[11:19], state:.details.update_progress.state}] | sort_by(.ts)'
```

Phase breakdown:

| Phase | Duration = | Expected (empty run) | Signals a problem |
|-------|-----------|----------------------|-------------------|
| Bootstrap | INITIALIZING → RUNNING | 60–400s | >400s = serverless pool contention |
| Work (per table) | RUNNING → last flow COMPLETED | 1–30s per table | >30s per table = source scan issue |
| Wind-down | last flow COMPLETED → cluster_end | 15–30s | never the bottleneck |

### Step 2: triage by bottleneck

#### Source bottleneck (MySQL / database)

**Signal**: Bootstrap fast, work phase slow, `Bytes_sent` delta is tiny (< 100 KB) but wall time is high → MySQL doing a full table scan, returning 0 rows.

| Root cause | How to confirm | Fix |
|-----------|---------------|-----|
| Missing index on QBC cursor column (`dt`) | `SHOW INDEX FROM <table>` — no `idx_dt` | `ALTER TABLE <tbl> ADD INDEX idx_dt (dt)` |
| Full table scan on empty check | `EXPLAIN SELECT * FROM intpk WHERE dt > '...' ORDER BY dt LIMIT 1000` — `type=ALL` | Add the index |
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
./databricks-run-with-byte-capture.sh $PIPELINE_PUBLIC_ID
./databricks-run-with-byte-capture.sh $PIPELINE_PNG_ID
# Compare MB/s between the two run-logs/byte-capture.jsonl entries
```

Degrading MB/s on PNG but not public → PNG routing or private endpoint issue.

#### Target bottleneck (Databricks / Delta write)

**Signal**: `Bytes_sent` delta is large (actual data flowing), but pipeline takes longer than prior runs with same volume.

```sql
-- Check Delta write metrics
DESCRIBE HISTORY <catalog>.<schema>.<table>
-- look at operationMetrics.executionTimeMs and numTargetBytesAdded
```

Cross-check with `system.compute.node_timeline` (5–7 day lag):
```sql
SELECT start_time, ROUND(cpu_io_wait_percent,1) AS io_wait_pct,
       ROUND(mem_used_percent,1) AS mem_pct
FROM system.compute.node_timeline
WHERE cluster_id = '<cluster_id>'
ORDER BY start_time
-- high io_wait → disk spill or slow Delta write
```

#### Serverless bootstrap contention

**Signal**: INITIALIZING → RUNNING duration > 300s. Work phase is fast once RUNNING.

Bootstrap latency varies by time of day and workspace load. For recurring jobs, consider scheduling at off-peak hours.

### Decision tree

```
Run slower than baseline?
├── Is INITIALIZING → RUNNING > 5 min? → serverless pool contention (not source/network/target)
└── Is RUNNING → COMPLETED slow?
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

Analyze trends in a notebook:
```python
import json, pandas as pd
rows = [json.loads(l) for l in open("run-logs/byte-capture.jsonl")]
df = pd.DataFrame(rows)
df["ts"] = pd.to_datetime(df["ts"])
display(df[["ts","routing","conn_name","wall_s","bytes_sent","mb_per_s","state"]]
          .sort_values(["routing","ts"]))
```

---

## Identifying update_id from a benchmark record

When you have a benchmark record (row count, time, pipeline_id) and need to find the matching `update_id`, use this three-step procedure.

### Step 1 — list completed updates

```bash
databricks -p $PROFILE pipelines list-updates <pipeline_id> \
  --max-results 20 --output json \
  | jq '[.updates[] | select(.state == "COMPLETED") | {update_id, state, creation_time, cause}]'
```

### Step 2 — determine which tables ran in each update

Pull `flow_progress` COMPLETED events. The flow name suffix encodes the table:

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200" \
  | jq '[.events[] | select(.details.flow_progress != null
         and .details.flow_progress.status == "COMPLETED") | {
      update_id: .origin.update_id,
      flow: .origin.flow_name,
      ts: .timestamp
    }] | sort_by(.ts)'
```

The **largest table present** characterizes the run:
- only `_intpk_upsert` → 1M row run
- + `_intpk_10m_upsert` → 10M row run
- + `_intpk_100m_upsert` → 100M row run

If older updates fell off page 1, paginate using `next_page_token` (see SKILL.md Events API pagination).

### Step 3 — match by duration (RUNNING → last flow COMPLETED)

```bash
databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<pipeline_id>/events?max_results=200" \
  | jq '[.events[] | select(.event_type == "update_progress") | {
      update_id: .origin.update_id,
      state: .details.update_progress.state, ts: .timestamp
    }] | group_by(.update_id) |
    map({update_id: .[0].update_id, events: map({state:.state,ts:.ts}) | sort_by(.ts)})'
```

Compute RUNNING → COMPLETED for each candidate. Compare to the `time` value in your benchmark record (differences of ±5s are normal).

### Matching rule

> The correct `update_id` is the one where:
> 1. Flow names match the expected table size(s), AND
> 2. RUNNING → COMPLETED duration matches the recorded `time` value (within ±5s)
>
> Both conditions must hold. If only duration matches but wrong flows, it is a coincidental match.
