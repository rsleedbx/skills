# Seed Column (fkr__unq) and Range Maintenance

## Seed column: controlled range index alongside AUTO_INCREMENT PK

When the PK is AUTO_INCREMENT / SERIAL / IDENTITY, add a separate **seed column** `fkr__unq` — a nullable `BIGINT UNSIGNED` with a `UNIQUE` index — that the generator fills with the exact seed integer. The `fkr__` prefix signals ownership by the synthetic data generator.

### DDL pattern

```sql
-- MySQL / MariaDB
CREATE TABLE target (
  pk       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  fkr__unq BIGINT UNSIGNED NULL DEFAULT NULL,
  dt       TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  ops      VARCHAR(255) DEFAULT 'insert',
  PRIMARY KEY (pk),
  UNIQUE KEY uk_fkr__unq (fkr__unq)           -- NULLs not compared, multiple NULLs OK
);

-- PostgreSQL
CREATE TABLE target (
  pk       BIGSERIAL PRIMARY KEY,
  fkr__unq BIGINT NULL DEFAULT NULL,
  CONSTRAINT uq_fkr__unq UNIQUE (fkr__unq)    -- NULLs not compared, multiple NULLs OK
);

-- SQL Server: plain UNIQUE allows only ONE NULL — use a filtered index
CREATE TABLE target (
  pk       BIGINT IDENTITY(1,1) NOT NULL PRIMARY KEY,
  fkr__unq BIGINT NULL DEFAULT NULL,
);
SET QUOTED_IDENTIFIER ON;
CREATE UNIQUE INDEX uq_fkr__unq ON target (fkr__unq) WHERE fkr__unq IS NOT NULL;
-- ✓ SQL Azure 12.0.2000.8: Msg 1934 when QUOTED_IDENTIFIER not set

-- Oracle
CREATE TABLE target (
  pk       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  fkr__unq NUMBER(19) NULL,
  CONSTRAINT uq_fkr__unq UNIQUE (fkr__unq)    -- NULLs ignored (NULL != NULL)
);
```

### NULL behavior in UNIQUE indexes

| Engine | Multiple NULLs? | Solution | Verified |
|--------|-----------------|---------|---------|
| MySQL / MariaDB | Allowed | Plain `UNIQUE KEY` | ✓ MySQL 8.0.44 |
| PostgreSQL | Allowed | Plain `UNIQUE` constraint | ✓ PostgreSQL 16.13 |
| SQL Server | Only one NULL | Filtered index: `WHERE fkr__unq IS NOT NULL` | ✓ SQL Azure 12.0.2000.8: `Msg 2601` on second NULL |
| Oracle | Allowed | Plain `UNIQUE` constraint | ⚠ not tested |

### INSERT pattern — offset shift on fkr__seed

```sql
-- MySQL — batch N, OFFSET = N * batch_size
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},
  date_add('1970-01-01 00:00:01', interval (a.pk + {OFFSET}) second),
  LPAD(a.pk + {OFFSET}, 10, '0')
FROM fkr__seed a;

-- PostgreSQL
INSERT INTO target (fkr__unq, dt, ops)
SELECT a.value,
  timestamp '1970-01-01 00:00:00' + a.value * interval '1 second',
  LPAD(a.value::text, 10, '0')
FROM generate_series({OFFSET}, {OFFSET} + {batch_size} - 1) a(value);
```

### Backfill fkr__unq on existing rows

```sql
-- MySQL: recover seed integer from ops column
UPDATE target SET fkr__unq = CAST(ops AS UNSIGNED);
```

### Validation — filter NULLs

```sql
-- Fast: two index lookups
SELECT
  MIN(fkr__unq)                      AS min_seed,
  MAX(fkr__unq)                      AS max_seed,
  MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count
FROM target WHERE fkr__unq IS NOT NULL;

-- Full check with contiguity
SELECT
  MIN(fkr__unq), MAX(fkr__unq),
  MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count,
  COUNT(*)                            AS actual_count,
  MAX(fkr__unq) - MIN(fkr__unq) + 1 = COUNT(*) AS contiguous
FROM target WHERE fkr__unq IS NOT NULL;
```

### When fkr__unq doesn't exist (fallback)

```sql
SELECT
  MIN(cast(ops AS signed))                                  AS min_seed,
  MAX(cast(ops AS signed))                                  AS max_seed,
  MAX(cast(ops AS signed)) - MIN(cast(ops AS signed)) + 1   AS estimated_count,
  COUNT(*)                                                   AS actual_count
FROM target;
```
Slower than an indexed integer column — prefer `fkr__unq` for any table designed for synthetic data.

---

## Range maintenance and verification

**The data is always a contiguous integer range when you control the PK.** Inserts extend the high end; deletes shrink from low or high end. There are never holes.

**Exception — AUTO_INCREMENT / SERIAL / IDENTITY:** MySQL 8 InnoDB with `innodb_autoinc_lock_mode=2` allocates AUTO_INCREMENT in bulk, producing gaps between batches. ✓ MySQL 8.0.44: two 10,000-row bulk inserts produced pk 1–10000 then 16384–26383. Use seed-derived column for counting.

### Never use COUNT(*) on large tables

```sql
-- Fast: two index lookups (valid ONLY for non-auto-increment PKs you control)
SELECT MIN(pk), MAX(pk), MAX(pk) - MIN(pk) + 1 AS estimated_count FROM target;

-- For AUTO_INCREMENT tables: count via seed-derived column
SELECT
  MIN(cast(ops AS signed)),
  MAX(cast(ops AS signed)),
  MAX(cast(ops AS signed)) - MIN(cast(ops AS signed)) + 1 AS estimated_count,
  COUNT(*) AS actual_count
FROM target;
```

### Delete patterns — always from one end

```sql
-- Delete from low end (oldest rows)
DELETE FROM target WHERE pk < :new_low_watermark;

-- Delete from high end (most recent rows)
DELETE FROM target WHERE pk > :new_high_watermark;
```

**Never delete from the middle** — creates gaps and breaks `MAX - MIN + 1` count invariant.

**AUTO_INCREMENT direction rule:** Always delete from the **low end (head)** — deleting from the high end leaves the next INSERT starting at a much higher value:
```python
# From jdbc_client.py::getDeleteStart
if pk_df.iloc[0].IS_AUTOINCREMENT == 'YES':
    delete_from_head = True   # delete from min(pk) end
```

**Cannot UPDATE an AUTO_INCREMENT / IDENTITY PK:**
```python
# jdbc_client.py::updateRangePk
if pk_df.iloc[0].IS_AUTOINCREMENT == 'YES':
    return None   # identity PK values cannot be reassigned
```

**Cannot UPDATE columns in UNIQUE keys:** `table_null_notnull_nouqifpk_columns` excludes unique-key columns from UPDATEs to avoid constraint violations.
