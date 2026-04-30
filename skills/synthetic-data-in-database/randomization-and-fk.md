# Randomization and Foreign Key Constraints

## Randomizing data values within a batch

Sequential inserts (seed 0, 1, 2 …) produce obviously artificial data. Randomize the seed used for **data columns** while keeping `fkr__unq` sequential.

### Rule

- **`fkr__unq`** — always `a.pk + OFFSET` (sequential, unique, for range counting). Never randomize.
- **All data columns** (`dt`, `ops`, varchar, numeric, etc.) — randomize within `[OFFSET, OFFSET + BATCH_SIZE)`, not `[0, total_rows)`.

**Why batch-bounded randomization matters:**
- `FLOOR(RAND() * total_rows)` → all batches sample `[0, 1M)` — lower values over-represented
- `FLOOR(OFFSET + RAND() * BATCH_SIZE)` → batch N samples `[N*B, (N+1)*B)` — flat coverage

### MySQL pattern

```sql
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},                                              -- sequential control key
  date_add('1970-01-01 00:00:01',
    interval FLOOR({OFFSET} + RAND() * {BATCH_SIZE}) second),  -- random within batch window
  LPAD(FLOOR({OFFSET} + RAND() * {BATCH_SIZE}), 10, '0')
FROM fkr__seed a;
```

✓ MySQL 8.0.44: `RAND()` is evaluated per row in INSERT SELECT.

### PostgreSQL pattern — use `random()`, not modulo

```sql
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.value,
  timestamp '1970-01-01 00:00:00'
    + (floor({OFFSET} + random() * {BATCH_SIZE}))::int * interval '1 second',
  LPAD((floor({OFFSET} + random() * {BATCH_SIZE}))::text, 10, '0')
FROM generate_series({OFFSET}, {OFFSET} + {BATCH_SIZE} - 1) a(value);
```

✓ PostgreSQL 16.13: `random()` IS evaluated per row.

**Do NOT use modulo `(x % batch_size) + offset`** for data columns in PostgreSQL. When `a.value` comes from `generate_series(OFFSET, OFFSET+BATCH_SIZE-1)` and OFFSET is a multiple of BATCH_SIZE, the modulo is a no-op — data is sequential, not randomized.
✓ PostgreSQL 16.13: confirmed — 10,000 rows from `generate_series(50000, 59999)` with `(v % 10000) + 50000` returned `v` for every row.

### SQL Server pattern — `RAND(CHECKSUM(NEWID()))` required

```sql
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},
  DATEADD(second, {OFFSET} + FLOOR(RAND(CHECKSUM(NEWID())) * {BATCH_SIZE}), '1970-01-01 00:00:01'),
  RIGHT(REPLICATE('0', 10) + CAST({OFFSET} + FLOOR(RAND(CHECKSUM(NEWID())) * {BATCH_SIZE}) AS VARCHAR), 10)
FROM fkr__seed a;
```

**Critical:** `RAND()` in SQL Server is evaluated **once per query** — every row gets the **same value**. `RAND(CHECKSUM(NEWID()))` generates a unique UUID per row.
✓ SQL Azure 12.0.2000.8: `RAND()` = `0.65816…` identical for all 5 rows; `RAND(CHECKSUM(NEWID()))` = 5 distinct values.

### Per-engine random function behavior

| Engine | Safe to use plain `RAND()`? | Multi-row INSERT SELECT formula | Verified |
|--------|----------------------------|--------------------------------|---------|
| SQL Server | **No — same value all rows** | `RAND(CHECKSUM(NEWID()))` | ✓ SQL Azure 12.0.2000.8 |
| PostgreSQL | Yes (`random()` per row) | `floor(offset + random() * batch_size)` | ✓ PostgreSQL 16.13 |
| MySQL | Yes (`RAND()` per row) | `FLOOR(offset + RAND() * batch_size)` | ✓ MySQL 8.0.44 |
| Oracle | Yes (`dbms_random.value()` per row) | `FLOOR(dbms_random.value() * batch_size) + offset` | ⚠ not tested |

### DataGenRange modes (from `my_config.py`)

| Mode | Formula | Notes |
|------|---------|-------|
| `MONOTONIC_INCREASE` | `x` (seed directly) | Sequential; matches PK exactly |
| `RANDOM_ROW_RANGE` *(default)* | `random() * batch_size + batch_offset` | Batch-bounded; recommended |
| `RANDOM_ROW_COUNT` | `random() * total_rows` | Full range; lower values over-represented in early batches |

---

## Foreign key constraints: populate parent first, bound child FK to parent range

### Steps

1. Populate the **parent** table first.
2. Query `MIN(fkr__unq)` and `MAX(fkr__unq)` from the parent.
3. For the **child** table's FK column: `FLOOR(min_parent + RAND() * (max_parent - min_parent + 1))`.

### MySQL example

```sql
-- Read parent range
SELECT MIN(fkr__unq) INTO @fk_min FROM categories WHERE fkr__unq IS NOT NULL;
SELECT MAX(fkr__unq) INTO @fk_max FROM categories WHERE fkr__unq IS NOT NULL;

-- Populate child, FK bounded to parent range
INSERT INTO orders (fkr__unq, category_id, dt, ops)
SELECT
  a.pk + {OFFSET},
  FLOOR(@fk_min + RAND() * (@fk_max - @fk_min + 1)),           -- FK → valid parent row
  date_add('1970-01-01 00:00:01',
    interval FLOOR({OFFSET} + RAND() * {BATCH_SIZE}) second),
  LPAD(FLOOR({OFFSET} + RAND() * {BATCH_SIZE}), 10, '0')
FROM fkr__seed a;
```

### Why this works

Parent `fkr__unq` is a contiguous range `[min_parent, max_parent]`, so any integer in that range is guaranteed to be a valid FK target. If the parent already has holes (partial deletes), query the actual min/max before each batch.

If the parent PK is auto_increment (with gaps), use `fkr__unq` as the FK target column and create a UNIQUE index on it — the child FK column references `fkr__unq`, not the auto-assigned `pk`.
