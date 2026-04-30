---
name: synthetic-data-in-database
description: >-
  Generating large-volume synthetic data inside a database via INSERT … SELECT with an integer-hub
  pattern — translating source types to a canonical integer then to target types. Per-engine
  SQL expressions for MySQL, PostgreSQL, SQL Server, and Oracle across all supported data types.
  MySQL bootstrap via fkr__seed table (no generate_series). Contiguous range invariant: rows are
  always maintained as a gap-free range [min, max]; deletes only from min or max end; COUNT(*) is
  expensive — always use MAX(pk) - MIN(pk) + 1 on indexed columns instead. How to add new type
  mappings to cast_x_to_int.py / cast_int_to_x.py / convert_int_to_x.py. Gap analysis for
  JavaSqlTypes not yet covered. Use when writing or extending synthetic data generation,
  configuring fkr__seed, verifying row counts, deleting data, or debugging type mismatches.
---

# Synthetic Data Generation in Database

## Validation legend

| Mark | Meaning |
|------|---------|
| ✓ `Engine vX.Y.Z` | Live-tested on that specific version. |
| ⚠ not tested | Derived from source code / docs — treat as assumption until tested. |
| (unmarked) | General SQL standard behaviour. |

**Test environment (Azure, 2026-04-29):** MySQL 8.0.44-azure, PostgreSQL 16.13, SQL Azure 12.0.2000.8. Oracle: no instance — all Oracle claims are ⚠ not tested.

---

## Core concept

Every column value is derived from a single integer seed `x` (`a.value` in SQL):
1. **Type → Integer** (`cast_x_to_int.py`): reads an existing PK and casts to bigint for seeding.
2. **Integer → Type** (`cast_int_to_x.py`): generates target column value from `x` inside `INSERT … SELECT`.
3. **Python-side** (`convert_int_to_x.py`): same mapping for JDBC prepared statements.

**Always start from 0.** Seed stream begins at `insertstart = 0` for deterministic, reproducible data.

```sql
INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value), castIntTo(a.value), ...
FROM <row_source> a   -- a.value starts at 0
```

## File locations

| File | Purpose |
|------|---------|
| `pylib/cast_x_to_int.py` | Type → bigint SQL expressions |
| `pylib/cast_int_to_x.py` | bigint → Type SQL expressions; `castFuncs` dispatch table |
| `pylib/convert_int_to_x.py` | bigint → Python/JDBC values for prepared statements |
| `pylib/jdbc_client.py` | `_insertSelGen`, `_insertSelectSql`, `_insertGenerateSql` — orchestration |
| `pylib/my_config.py` | `BulkMethod` enum, `fkr__seed` default config |
| `pylib/java_sql_types.py` | `JavaSqlTypes` enum — master list of all known types |

---

## Bootstrap: how rows are generated

**Always use `insertstart = 0`.**

### PostgreSQL
```sql
INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value, ...)
FROM generate_series(0, :size - 1) a(value);
```

### MySQL (no generate_series)

Uses pre-populated integer table `fkr__seed` (PK `pk INT`, values 0..batch_size-1). **Recommended batch_size = 1,000,000** (14.5–15s/batch on Azure MySQL 8.0.44 ✓).

```sql
CREATE TABLE fkr__seed (pk INT PRIMARY KEY);  -- populated with 0..999999

INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value, ...)
FROM (SELECT pk - 0 + 0 AS value FROM fkr__seed WHERE :low <= pk AND pk < :high) a;
```

`BulkMethod.generate_series` is NOT the MySQL default — set `bulk_method = BulkMethod.insert_select`.

### SQL Server

Same `fkr__seed` pattern; SQL Server 2022+ (compat ≥160) can use `generate_series()` via `BulkMethod.generate_series`.

### Oracle

Same `fkr__seed` pattern. Alternative (not implemented): `SELECT LEVEL - 1 FROM DUAL CONNECT BY LEVEL <= N`.

For detail on building fkr__seed (bootstrap + doubling pattern), batch loop patterns, and timing benchmarks: see [stats-timing-batching.md](stats-timing-batching.md).

---

## Contiguous range invariant

- **Inserts** extend the high end; **deletes** shrink from the low or high end only
- Never delete from the middle — breaks `MAX - MIN + 1` count invariant
- For AUTO_INCREMENT PKs: use `fkr__unq` seed column for counting (not `pk`) — MySQL InnoDB allocates AUTO_INCREMENT in bulk, producing gaps

```sql
-- Correct range count for seed column
SELECT MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count
FROM target WHERE fkr__unq IS NOT NULL;
```

For full seed column DDL, NULL behavior in UNIQUE indexes, and delete direction rules: see [seed-column-and-range.md](seed-column-and-range.md).

---

## Key type mappings (quick reference)

See [type-reference.md](type-reference.md) for complete per-engine tables.

### UUID / UNIQUEIDENTIFIER
| Engine | Int → Type |
|--------|-----------|
| SQL Server | `convert(uniqueidentifier, convert(binary, FORMAT(cast(x as bigint),'X32'), 2))` |
| PostgreSQL | `CAST(LPAD(TO_HEX(cast(x as bigint)),32,'0') AS UUID)` |
| MySQL | `CAST(HEX(x) AS BINARY(32))` |
| Oracle | `SYS_GUID()` — NOT deterministic from x ⚠ |

### Timestamp / DateTime
| Engine | Int → Type |
|--------|-----------|
| SQL Server | `DATEADD(ss, x, '01 JAN 1970')` |
| PostgreSQL | `timestamp '1970-01-01 00:00:00' + (x) * interval '1 second'` |
| MySQL | `date_add('1970-01-01T00:00:01', interval x second)` (start at 1, not 0 — see gotchas) |
| Oracle DATE | `DATE '1970-01-01' + NUMTODSINTERVAL(x,'SECOND')` |

### String / VARCHAR
| Engine | Int → Type |
|--------|-----------|
| SQL Server | `cast(RIGHT(CONCAT(replicate(cast('0' as varchar(max)),N), x), N) as VARCHAR(N))` |
| PostgreSQL | `RIGHT(CONCAT(encode(gen_random_bytes(N/2),'hex'), x), N)` |
| MySQL | `RIGHT(CONCAT(repeat('0',N), x), N)` |
| Oracle | `substr(lpad(x, N, '0'), -N)` |

---

## Schema introspection

For batching principle, case sensitivity, SQL templates (all 4 engines), JDBC fields vs INFORMATION_SCHEMA, and decision rules: see [schema-introspection.md](schema-introspection.md).

---

## Randomization and foreign key constraints

For batch-bounded randomization, per-engine `RAND()` behavior, DataGenRange modes, and FK population pattern: see [randomization-and-fk.md](randomization-and-fk.md).

**Key rule:** SQL Server `RAND()` returns the **same value for all rows** in an INSERT SELECT. Use `RAND(CHECKSUM(NEWID()))` instead. ✓ SQL Azure 12.0.2000.8: confirmed.

---

## Adding a new type mapping

1. **`cast_int_to_x.py`**: write `castIntToXxx(...)` returning a SQL expression string per engine.
2. **`castFuncs` dict** (same file): add `JavaSqlTypes.XXX.name.upper(): castIntToXxx`.
3. **`cast_x_to_int.py`**: write `castXxxToBigInt(...)` and add to `castXToIntFuncs`.
4. **`convert_int_to_x.py`**: write `convertIntToXxx(...)` returning a Python value; add to `convertFuncs`.
5. **`java_sql_types.py`**: confirm type exists in `JavaSqlTypes`; add if not.

For parameterized Oracle types like `TIMESTAMP(6) WITH TIME ZONE`, add a regex branch to `getCastFunction()` instead of polluting `castFuncs`.

---

## Known gaps (JavaSqlTypes with no castFuncs entry)

| Type | Suggested approach |
|------|--------------------|
| `SERIAL4` / `SERIAL8` | Delegate to `castIntToInteger` / `castIntToBigInt` |
| `ROWVERSION` | Treat as `BINARY(8)` — SQL Server auto-managed, skip in INSERT |
| `LONG_RAW` | Delegate to `castIntToBinary` with Oracle RAW semantics |
| `BFILE` | `BFILENAME('MY_DIR', lpad(to_char(x),20,'0') || '.bin')` |
| `SQLXML` | `castIntToXml` (synonym for XML at SQL level) |
| `NULL` | Return literal `NULL` |
| `ARRAY` | `ARRAY(SELECT castIntTo(v, elem) FROM generate_series(1,N) v)` for PostgreSQL |
| `REF_CURSOR` | Not insertable — skip |
| `OTHER` / `DISTINCT` / `REF` | Fall back to `castIntToInteger`; log warning |

---

## Engine-specific gotchas

- **MySQL AUTO_INCREMENT gaps (`innodb_autoinc_lock_mode=2`):** ✓ MySQL 8.0.44: bulk INSERT SELECT allocates AUTO_INCREMENT in pre-allocated ranges — two 10,000-row batches produced pk 1–10000 then 16384–26383. Use seed-derived column for verification.
- **MySQL TIMESTAMP epoch zero:** ✓ MySQL 8.0.44: `date_add('1970-01-01 00:00:00', interval 0 second)` is rejected — `1970-01-01 00:00:00` is the reserved NULL sentinel. Use `'1970-01-01 00:00:01'` as base epoch.
- **MySQL `ON UPDATE CURRENT_TIMESTAMP`:** Any UPDATE to the row resets `dt`. ✓ MySQL 8.0.44: all dt values reset to current wall time after `fkr__unq` backfill UPDATE.
- **PostgreSQL UUID:** `MIN()`/`MAX()` are not defined for `uuid` — use `ORDER BY pk ASC LIMIT 1` / `ORDER BY pk DESC LIMIT 1`.
- **PostgreSQL SERIAL vs GENERATED AS IDENTITY:** SERIAL sets `column_default = nextval(...)` with `is_identity = 'NO'`. Only `GENERATED AS IDENTITY` sets `is_identity = 'YES'`. Both require different detection.
- **SQL Server `RAND()` in INSERT SELECT:** Evaluated **once per query** — all rows get the same value. Use `RAND(CHECKSUM(NEWID()))`. ✓ SQL Azure 12.0.2000.8: confirmed.
- **SQL Server filtered index for `fkr__unq`:** Requires `SET QUOTED_IDENTIFIER ON` before `CREATE INDEX`. ✓ SQL Azure 12.0.2000.8: `Msg 1934` without it.
- **PostgreSQL modulo no-op:** `(x % batch_size) + offset` where `x = generate_series(offset, offset+batch-1)` and offset is multiple of batch_size = sequential data, not random. Use `random()` instead. ✓ PostgreSQL 16.13: confirmed.
