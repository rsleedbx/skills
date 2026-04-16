# ZeroBus — Benchmarking Pattern

Standard structure for measuring ZeroBus ingest latency. Implemented in `zerobusdemo`; applies to any ZeroBus integration.

## Four-phase structure

| Phase | What it measures |
|---|---|
| **4a — singles** | Per-row RTT: separate send and wait/ack timings for N individual inserts |
| **4b — batch** | Batch wall time: run `_batch_runs` times to get a distribution (min/mean/median/max) |
| **4c — visibility** | Delta table commit lag: poll `COUNT(*)` until expected row count appears |
| **4d — key metrics** | Print all results in a consistent format |

## Config variables

```python
_n = 1000               # total rows per run
_singles = min(10, _n)  # rows sent individually in 4a
_batch_runs = 10        # consecutive batch sends in 4b (configurable)
```

## Timing variables

```python
# 4a
t_4a0               # perf_counter() at first single row send
_singles_wall_s     # total wall time for all singles
_row_send_seconds   # list[float]: per-row send duration
_row_wait_seconds   # list[float]: per-row wait/ack (0.0 for HTTP — no separate ack)
_row_ack_seconds    # list[float]: per-row total (send + wait)

# 4b
_batch_run_seconds  # list[float]: wall time per batch run (send + wait)
_batch_send_seconds # list[float]: send portion per run
_batch_wait_seconds # list[float]: wait/ack portion per run (0.0 for HTTP)
_batch_n = max(0, _n - _singles)
_total_rows_inserted = _singles + _batch_n * len(_batch_run_seconds)
_ingest_4a4b_s = _singles_wall_s + sum(_batch_run_seconds)

# 4c
_t_after_close      # perf_counter() after stream.close() or last HTTP insert
_visibility_s       # time from _t_after_close → COUNT(*) reached target
_visibility_from_first_send_s  # time from t_4a0 → COUNT(*) reached target
```

## 4a — singles loop

```python
t_4a0 = time.perf_counter()
for i in range(_singles):
    t_row = time.perf_counter()
    # ... send one row ...
    _row_send_seconds.append(...)   # time to send
    _row_wait_seconds.append(...)   # time to wait for ack (0.0 for HTTP)
_singles_wall_s = time.perf_counter() - t_4a0
```

## 4b — batch loop (distribution)

```python
for _ in range(_batch_runs):
    t_bs0 = time.perf_counter()
    # ... send batch ...
    _batch_send_seconds.append(time.perf_counter() - t_bs0)
    t_bw0 = time.perf_counter()
    # ... wait for ack ...
    _batch_wait_seconds.append(time.perf_counter() - t_bw0)
    _batch_run_seconds.append(_batch_send_seconds[-1] + _batch_wait_seconds[-1])
```

## 4c — visibility poll

```python
_target_count = _row_before + _total_rows_inserted
_poll_deadline = _t_after_close + 120.0
while time.perf_counter() < _poll_deadline:
    _row_visible = spark.sql(f"SELECT COUNT(*) AS c FROM {TABLE_NAME}").collect()[0]["c"]
    if _row_visible >= _target_count:
        break
    time.sleep(0.15)
_t_visible = time.perf_counter()
_visibility_s = _t_visible - _t_after_close
_visibility_from_first_send_s = _t_visible - t_4a0
```

## 4d — key metrics print

```python
from statistics import mean, median

# Singles distribution
print(f"  4a ({_singles} singles): wall {_singles_wall_s*1000:.1f} ms  (~{_singles_wall_s/_singles*1000:.1f} ms/row)")
print(f"      per-row send: min={min(_row_send_seconds)*1000:.1f} ms  mean={mean(_row_send_seconds)*1000:.1f} ms  median={median(_row_send_seconds)*1000:.1f} ms  max={max(_row_send_seconds)*1000:.1f} ms")
print(f"      per-row wait: min={min(_row_wait_seconds)*1000:.1f} ms  mean={mean(_row_wait_seconds)*1000:.1f} ms  median={median(_row_wait_seconds)*1000:.1f} ms  max={max(_row_wait_seconds)*1000:.1f} ms")

# Batch distribution
if _batch_run_seconds and _batch_n:
    print(
        f"  4b ({_batch_n} batched, {len(_batch_run_seconds)} runs):"
        f"  wall  min={min(_batch_run_seconds)*1000:.1f} ms"
        f"  mean={mean(_batch_run_seconds)*1000:.1f} ms"
        f"  median={median(_batch_run_seconds)*1000:.1f} ms"
        f"  max={max(_batch_run_seconds)*1000:.1f} ms"
    )
    print(f"      send: min={min(_batch_send_seconds)*1000:.1f} ms  mean={mean(_batch_send_seconds)*1000:.1f} ms  ...")
    print(f"      wait: min={min(_batch_wait_seconds)*1000:.1f} ms  mean={mean(_batch_wait_seconds)*1000:.1f} ms  ...")

# Visibility
print(f"  visibility: {_visibility_from_first_send_s*1000:.1f} ms (from first row send)")
print(f"  visibility: {_visibility_s*1000:.1f} ms (from end of ingest)")
```

## Protocol-specific notes

| Protocol | `_row_wait_seconds` | `_batch_wait_seconds` | `_stream_close_s` |
|---|---|---|---|
| gRPC sync/async | actual `wait_for_offset` time | actual `wait_for_offset` time | `stream.close()` duration |
| HTTP sync/async | `0.0` (POST is atomic) | `0.0` | `None` |

## Typical results (warm connection, 990-row batch)

| Pattern | Singles amortized | Batch mean | Visibility lag |
|---|---|---|---|
| HTTP sync (Session) | ~400 ms/row | 200–600 ms | 1–8 s |
| HTTP async (gather) | ~250 ms wall | 200–600 ms | 1–8 s |
| gRPC sync | ~200 ms/row | 400–600 ms | 1–8 s |
| gRPC async | ~200 ms wall | 400–600 ms | 1–8 s |
