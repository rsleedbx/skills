# Stats Collection, Timing, and Batch Sizing

## Stats collection: MIN/MAX for all tables in one UNION query

Collect `(table_name, min_pk, max_pk)` for all tables in **one SQL call** — up to 300 tables per execution:

```python
per_table_subqueries = [minMaxSql(table) for table in tables]
batched = " UNION ".join(per_table_subqueries[:300])
cursor.execute(batched)
```

### Strategy 1 — `MIN()` / `MAX()` aggregate (most types)

```sql
SELECT l.table_name, r.min, r.max, r.row_count
FROM
    (SELECT 'orders' AS table_name, NULL AS min, NULL AS max) l
LEFT JOIN
    (SELECT 'orders'           AS table_name,
            castToInt(MIN(pk)) AS min,
            castToInt(MAX(pk)) AS max,
            0                  AS row_count
     FROM orders
     WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)) r
  ON l.table_name = r.table_name
```

### Strategy 2 — `ORDER BY … LIMIT 1` (PostgreSQL UUID only)

`uuid` does not support `MIN()`/`MAX()` in PostgreSQL. Use two ORDER BY subqueries:

```sql
SELECT l.table_name,
       castToInt(rmin.min) AS min,
       castToInt(rmax.max) AS max, l.row_count
FROM (SELECT 'events' AS table_name, NULL AS min, NULL AS max, 0 AS row_count) l
LEFT JOIN (SELECT 'events' AS table_name, pk AS min
           FROM events ORDER BY pk ASC  LIMIT 1) rmin ON l.table_name = rmin.table_name
LEFT JOIN (SELECT 'events' AS table_name, pk AS max
           FROM events ORDER BY pk DESC LIMIT 1) rmax ON l.table_name = rmin.table_name
```

### Why LEFT JOIN

Empty table → data subquery returns 0 rows → result is `(table_name, NULL, NULL, 0)`. Without LEFT JOIN, empty tables return zero rows and cannot be distinguished from unscanned tables.

### Range WHERE clause

```sql
WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)
```
Prevents reading rows outside the synthetic data range. `wrap_on_overflow=False` caps at max representable value.

### Batching limit: 300 subqueries per UNION

```python
max_union = 300   # from MyConfig.limits.max_union
# 1000 tables → ceil(1000/300) = 4 round trips
```

---

## Timing: always use DB-native profiling, not client-side timing

Client-side `time` includes network round-trip, connection setup, secret fetch, and firewall checks — typically 4–20× the actual DB execution time.

### MySQL — `SET profiling = 1` + `SHOW PROFILES`

```sql
SET profiling = 1;
-- run queries
SHOW PROFILES;
-- Query_ID | Duration   | Query
--        1 | 14.56s     | INSERT INTO target ...   ← pure DB time
```

✓ MySQL 8.0.44: `SHOW PROFILES` still works despite deprecation notice.

### PostgreSQL — `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
INSERT INTO target (fkr__unq, dt, ops)
SELECT a.value, ... FROM generate_series(...) a(value);
-- "Execution Time: 13.488 ms"
```
✓ PostgreSQL 16.13: `EXPLAIN ANALYZE INSERT ... SELECT generate_series(0,9999)` returned `Execution Time: 13.488 ms`.

### SQL Server — `SET STATISTICS TIME ON`

```sql
SET STATISTICS TIME ON;
INSERT INTO target (fkr__unq, dt, ops) SELECT ... FROM fkr__seed a;
SET STATISTICS TIME OFF;
-- SQL Server Execution Times: CPU time = 0 ms, elapsed time = 2 ms.
```
✓ SQL Azure 12.0.2000.8: confirmed.

### Oracle — `SET TIMING ON` (sqlplus)

```sql
SET TIMING ON
INSERT INTO target (fkr__unq, dt, ops) SELECT ... FROM fkr__seed a;
-- "Elapsed: 00:00:00.31"
```

### Observed benchmarks

✓ MySQL 8.0.44-azure (2 vCores), 2026-04-29:

| Operation | DB time | Client elapsed | Network overhead |
|-----------|---------|----------------|-----------------|
| INSERT SELECT 10,000 rows | 0.243s | ~5.7s | ~23× |
| SELECT MIN/MAX/COUNT 200k rows | 0.089s | ~5.4s | ~60× |
| SELECT MIN/MAX/COUNT (UNION ALL) 2 × 1M rows | 1.349s | ~8.4s | ~6× |

✓ PostgreSQL 16.13: INSERT SELECT 10,000 rows via generate_series = 13.488 ms.

---

## Batch sizing: avoid OOM on large inserts

**Never insert all rows in one batch.** `fkr__seed` holds `max_batch_size` rows — run multiple batches with increasing offsets.

### Choosing batch size

```
target_working_set_mb  = 256
avg_row_bytes          = sum of max byte size of every column in INSERT
batch_size             = min(1_000_000, floor(target_working_set_mb * 1_048_576 / avg_row_bytes))
```

### Rough caps by column type

| Dominant column type | Max bytes/row | Safe batch size |
|----------------------|---------------|-----------------|
| INT / BIGINT / TIMESTAMP only | < 100 B | **1,000,000** ✓ |
| VARCHAR(255) × 5 | ~1.3 KB | 200,000 |
| TEXT / MEDIUMTEXT | 64 KB – 16 MB | 1,000 |
| BLOB / VARBINARY(65535) | 64 KB | 1,000 |
| MEDIUMBLOB | 16 MB | 100 |
| LONGBLOB × multiple columns | > 100 MB | 10 or fewer |

**Recommended default: 1,000,000** (14.5–15s/batch on Azure MySQL 8.0.44 ✓)

### Scale benchmark: 10 million rows ✓ (MySQL 8.0.44-azure, 2026-04-29)

Phase 1 — build `fkr__seed` 10k → 1M (7 doublings): **8.5s total**
Phase 2 — 10 × 1M-row batches: **~14.5–15s each, ~148s DB / 156s wall-clock**
Throughput: **~67,600 rows/sec** into a table with UNIQUE index on `fkr__unq` and AUTO_INCREMENT on `pk`.

**Do not add external parallelism** — the DB parallelizes INSERT SELECT internally. Multiple connections cause lock contention and are slower overall.

### Building fkr__seed at 1M rows — bootstrap then double

**Phase 1 — Bootstrap 1K rows (VALUES cross join, all engines):**
```sql
WITH d(x) AS (
  SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5
  UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9 UNION ALL SELECT 10
)
INSERT INTO fkr__seed (pk)
SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1   -- SQL Server / PostgreSQL
-- MySQL: (a.x - 1)*100 + (b.x - 1)*10 + (c.x - 1) AS pk
FROM d a CROSS JOIN d b CROSS JOIN d c;
-- → 1,000 rows (pk 0..999)
```

**Why not a 4-level geometric CTE?** Produces at most 65,536 rows (2¹⁶). ✓ SQL Azure 12.0.2000.8: confirmed `TOP 1000000` on e0→e4 CTE inserted exactly 65,536 rows.

**Phase 2 — Double to 1M (10 steps, idempotent):**
```sql
DECLARE @c INT; SELECT @c = COUNT(*) FROM fkr__seed;          -- 1000
INSERT INTO fkr__seed SELECT pk + @c FROM fkr__seed; SELECT @c=COUNT(*) FROM fkr__seed;  -- 2000
-- ... repeat doubling ...
INSERT INTO fkr__seed SELECT pk + @c FROM fkr__seed; SELECT @c=COUNT(*) FROM fkr__seed;  -- 512000
INSERT INTO fkr__seed SELECT pk + @c FROM fkr__seed WHERE pk < (1000000 - @c);           -- 1M
```

✓ MySQL 8.0.44-azure: 7 doublings from 10K = **8.5s total**
✓ SQL Azure 12.0.2000.8: 10 doublings from 1K = **~8.9s total**

### Batch size IS the fkr__seed size

`fkr__seed` is created with exactly `batch_size` rows. When changing batch size, resize `fkr__seed` first using the doubling pattern. Do **not** make it as large as `total_rows`.

### MySQL batch loop pattern

```sql
INSERT INTO target (dt, ops)
SELECT
  date_add('1970-01-01 00:00:01', interval a.value second),
  RIGHT(CONCAT(repeat('0', 255), a.value), 255)
FROM (
  SELECT pk + :offset AS value
  FROM fkr__seed
  WHERE 0 <= pk AND pk < :batch_size
) a;
-- Validate after each batch
SELECT MIN(pk) AS min_pk, MAX(pk) AS max_pk, MAX(pk) - MIN(pk) + 1 AS estimated_count FROM target;
```

### PostgreSQL batch loop pattern

```sql
INSERT INTO target (col1, col2)
SELECT castIntTo(a.value, ...)
FROM generate_series(:offset, :offset + :batch_size - 1) a(value);
```

### Determining batch size empirically

1. Start with `batch_size = 100` for any table with LOB columns.
2. Run one batch, observe DB memory/CPU.
3. Double batch size until memory pressure or query time grows super-linearly.
4. Use 50% of the largest stable batch size as production default.
