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

Claims in this skill are marked with one of three statuses:

| Mark | Meaning |
|------|---------|
| ✓ `Engine vX.Y.Z` | Live-tested on that specific engine version. Behaviour may differ on other versions — re-test when upgrading. |
| ⚠ not tested | Derived from source code reading, official docs, or reasoning — not confirmed by a live query. Treat as assumption until tested. |
| (unmarked) | General SQL standard behaviour unlikely to vary across versions. |

**Test environment (Azure, 2026-04-29):**

| Engine | Version |
|--------|---------|
| MySQL | **8.0.44-azure** (Azure MySQL Flexible Server) |
| PostgreSQL | **16.13** (Azure PostgreSQL Flexible Server, x86_64, gcc 13.2.0) |
| SQL Server | **Azure SQL / SQL Azure 12.0.2000.8** (2026-03-13 build) |
| Oracle | No instance available — all Oracle claims are ⚠ not tested |

## Core concept

Every column value is derived from a single integer seed `x` (called `a.value` in SQL):
1. **Type → Integer** (`cast_x_to_int.py`): reads an existing PK column and casts it to bigint for seeding.
2. **Integer → Type** (`cast_int_to_x.py`): generates the target column value from `x` inside `INSERT … SELECT`.
3. **Python-side** (`convert_int_to_x.py`): same mapping for JDBC prepared statements.

**Always start from 0.** The seed integer stream begins at 0 (`insertstart = 0`). Row 0 produces the first value for every column. This ensures deterministic, reproducible data and consistent round-trip verification.

This lets the database generate millions of rows with no Python loops:
```sql
INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value), castIntTo(a.value), ...
FROM <row_source> a   -- a.value starts at 0; see Bootstrap section
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

## Bootstrap: how rows are generated

**Always use `insertstart = 0`** — the first row is seeded with `a.value = 0`.

### PostgreSQL
```sql
-- insertstart = 0; generates values 0, 1, 2, ..., size-1
INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value, ...)
FROM generate_series(0, :size - 1) a(value);
```

### MySQL (no generate_series)

MySQL uses a pre-populated integer table `fkr__seed` (PK `pk INT`, values 0..batch_size-1). **Recommended batch_size = 1,000,000** (pk 0..999,999) — tested at 14.5–15s per 1M-row batch on Azure MySQL Flexible Server (MySQL 8.0.44-azure). Use 10,000 only for tables with large columns (BLOB, wide TEXT) where a 1M-row batch would OOM.
```sql
-- Created once:
CREATE TABLE fkr__seed (pk INT PRIMARY KEY);
-- populated with 0..999

-- Per-batch INSERT SELECT; deletestart=0, insertstart=0 → value starts at 0:
INSERT INTO target (col1, col2, ...)
SELECT castIntTo(a.value, ...)
FROM (
  SELECT pk - 0 + 0 AS value   -- pk=0 → value=0
  FROM fkr__seed
  WHERE :low <= pk AND pk < :high
) a;
```
- `BulkMethod.generate_series` is NOT the MySQL default — set `bulk_method = BulkMethod.insert_select`.
- Bootstrap fallback: if `fkr__seed` is empty, `jdbc_client.py` falls back to repeated single-row `insert()` until non-empty, then switches to INSERT SELECT.

### SQL Server
Same `fkr__seed` pattern by default; `insertstart = 0`. SQL Server 2022+ (compat ≥160) can use `generate_series()` via `BulkMethod.generate_series`.

### Oracle
Same `fkr__seed` pattern; `insertstart = 0`. Oracle alternative (not implemented): `SELECT LEVEL - 1 FROM DUAL CONNECT BY LEVEL <= N`.

## Schema introspection: PK, UNIQUE, INDEX, FK

### Batching principle — one round trip per operation

Never query one table at a time. Retrieve metadata for **all tables in one SQL call**:

| Operation | Pattern |
|-----------|---------|
| Column discovery | `WHERE TABLE_SCHEMA = :catalog` — **no table filter**; returns all tables at once. Filter by table in application code. |
| Index / PK / UNIQUE | `WHERE TABLE_SCHEMA = :catalog AND TABLE_NAME IN ('t1','t2',…)` — all tables in one query. |
| Stats collection | `UNION ALL` subqueries per table, chunked at **max 300** per execution to stay within query size limits. |

**IN-list construction** (no prepared statements for index queries):
```python
# All table names in one IN list — one DB round trip regardless of table count
table_names_sql = ",".join([f"'{t}'" for t in table_names])
where = f"TABLE_NAME IN ({table_names_sql})"
```

**Prepared statement param limits by engine** (when using `?` placeholders):
| Engine | Param limit |
|--------|------------|
| SQL Server ≥ v8 | 2,100 |
| SQL Server v7 | 1,024 |
| PostgreSQL | 32,767 (use `unnest()` for larger arrays to bypass 65,535 wire limit) |
| MySQL / Oracle | No hard limit documented in code; use string IN list |

### Case sensitivity per engine

| Engine | Default | Quote char | Meaning |
|--------|---------|------------|---------|
| MySQL | `DEFAULT` (case-insensitive) | `` ` `` | Unquoted identifiers are case-insensitive; backtick-quote only when needed |
| PostgreSQL (raw) | `DEFAULT` (case-insensitive unquoted) | `"` | Unquoted = case-insensitive; `"Name"` = case-sensitive |
| PostgreSQL (managed/Databricks) | `QUOTE` | `"` | Always double-quote identifiers; triggers case-sensitive matching |
| SQL Server | `CASE_INSENSITIVE` | `"` | Always case-insensitive regardless of quoting |
| Oracle | `QUOTE` | `"` | Unquoted identifiers stored as UPPERCASE; quoted identifiers are case-sensitive |

**Effect on queries:**
- INFORMATION_SCHEMA `WHERE` clauses use **literal equality** — no `LOWER()`/`UPPER()` applied in SQL
- Case-sensitivity is applied **after fetch** using pandas `.str.match('^name$', case=case_sensitive_search)`
- For Oracle: unquoted table/column names in DDL are stored as UPPERCASE in ALL_TABLES — use `UPPER(:name)` when filtering

### Special character and identifier quoting rules

Wrap an identifier in the engine's quote char when:
1. It matches a SQL reserved keyword for that engine
2. It contains non-word characters (`\W` — anything not `[a-zA-Z0-9_]`)
3. It starts with a digit
4. Engine is configured with `QUOTE` (Oracle, managed PostgreSQL) — always quote

```sql
-- MySQL: backtick-quote special names
SELECT `order`, `group`, `my-col` FROM `my.table`;

-- PostgreSQL / Oracle / SQL Server: double-quote
SELECT "Order", "my col", "123col" FROM "my schema"."my table";
```

**Oracle case rule:** identifiers without quotes are stored/compared as UPPERCASE:
```sql
-- These are the same in Oracle (all resolve to TABLE_NAME = 'MYTABLE'):
SELECT * FROM mytable;
SELECT * FROM MYTABLE;
SELECT * FROM MyTable;
-- Only this is different (stored as 'MyTable'):
SELECT * FROM "MyTable";
```

### SQL templates — one query for all tables

#### MySQL
```sql
-- Columns: one call for entire schema, all tables
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = ':catalog'
ORDER BY TABLE_NAME, ORDINAL_POSITION;

-- PK / UNIQUE / INDEX: all tables in one IN list
SELECT
    TABLE_SCHEMA AS TABLE_CAT, 'Null' AS TABLE_SCHEM,
    TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX AS KEY_SEQ,
    CASE WHEN INDEX_NAME = 'PRIMARY' THEN 'PRIMARY KEY'
         WHEN NON_UNIQUE = 0         THEN 'UNIQUE'
         ELSE '' END AS CONSTRAINT_TYPE,
    INDEX_TYPE AS TYPE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = ':catalog'
  AND TABLE_NAME IN (':t1',':t2', …)   -- all tables, one query
ORDER BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME;

-- Foreign keys
SELECT COLUMN_NAME, CONSTRAINT_NAME,
       REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = ':catalog'
  AND TABLE_NAME IN (':t1',':t2', …)
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

#### PostgreSQL
```sql
-- Columns: JDBC getColumns(catalog, schema, null, null) equivalent
SELECT * FROM information_schema.columns
WHERE table_catalog = ':catalog' AND table_schema = ':schema'
ORDER BY table_name, ordinal_position;

-- PK / UNIQUE / INDEX: all tables in one IN list
SELECT
    ':catalog' AS TABLE_CAT, tnsp.nspname AS TABLE_SCHEM,
    trel.relname AS TABLE_NAME, i.relname AS INDEX_NAME,
    a.attname AS COLUMN_NAME,
    array_position(ix.indkey, a.attnum) + 1 AS KEY_SEQ,
    CASE WHEN indisprimary THEN 'PRIMARY KEY'
         WHEN indisunique  THEN 'UNIQUE'
         ELSE '' END AS CONSTRAINT_TYPE
FROM pg_class trel
JOIN pg_index     ix   ON trel.oid = ix.indrelid
JOIN pg_class     i    ON i.oid    = ix.indexrelid
JOIN pg_attribute a    ON a.attrelid = trel.oid
JOIN pg_namespace tnsp ON trel.relnamespace = tnsp.oid
WHERE a.attnum = ANY(ix.indkey)          -- only indexed columns
  AND trel.relkind = 'r'                 -- only regular tables
  AND tnsp.nspname = ':schema'
  AND trel.relname IN (':t1',':t2', …)  -- all tables, one query
ORDER BY tnsp.nspname, trel.relname, i.relname,
         array_position(ix.indkey, a.attnum);

-- PostgreSQL large IN list: use unnest() to avoid 65535 param limit
-- WHERE trel.relname = ANY(ARRAY[':t1',':t2',…])
```

#### SQL Server
```sql
-- Columns: JDBC getColumns equivalent
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_CATALOG = ':catalog' AND TABLE_SCHEMA = ':schema'
ORDER BY TABLE_NAME, ORDINAL_POSITION;

-- PK / UNIQUE / INDEX: all tables in one IN list
SELECT
    db_name() AS TABLE_CAT, s.name AS TABLE_SCHEM,
    t.name AS TABLE_NAME, ind.name AS INDEX_NAME,
    col.name AS COLUMN_NAME, ic.key_ordinal AS KEY_SEQ,
    CASE WHEN ind.is_primary_key = 1 THEN 'PRIMARY KEY'
         WHEN ind.is_unique = 1      THEN 'UNIQUE'
         ELSE '' END AS CONSTRAINT_TYPE,
    ind.type_desc AS TYPE
FROM sys.indexes       ind
JOIN sys.index_columns ic  ON ind.object_id = ic.object_id AND ind.index_id  = ic.index_id
JOIN sys.columns       col ON ic.object_id  = col.object_id AND ic.column_id = col.column_id
JOIN sys.tables        t   ON col.object_id = t.object_id
JOIN sys.schemas       s   ON t.schema_id   = s.schema_id
WHERE s.name = ':schema'
  AND t.name IN (':t1',':t2', …)         -- all tables, one query
ORDER BY s.name, t.name, ind.name, ic.key_ordinal;
```

#### Oracle
```sql
-- Columns: JDBC getColumns equivalent
-- Note: unquoted names stored as UPPERCASE in Oracle
SELECT * FROM ALL_TAB_COLUMNS
WHERE OWNER = UPPER(':schema')           -- always UPPER for unquoted Oracle names
ORDER BY TABLE_NAME, COLUMN_ID;

-- PK / UNIQUE / INDEX: all tables in one IN list
SELECT
    ':catalog' AS TABLE_CAT, t.OWNER AS TABLE_SCHEM,
    t.TABLE_NAME, i.INDEX_NAME, ic.COLUMN_NAME,
    ic.COLUMN_POSITION AS KEY_SEQ,
    CASE WHEN c.CONSTRAINT_TYPE = 'P' THEN 'PRIMARY KEY'
         WHEN i.UNIQUENESS = 'UNIQUE'  THEN 'UNIQUE'
         ELSE '' END AS CONSTRAINT_TYPE,
    i.INDEX_TYPE AS TYPE
FROM ALL_TABLES      t
JOIN ALL_INDEXES     i   ON t.OWNER = i.OWNER AND t.TABLE_NAME = i.TABLE_NAME
JOIN ALL_IND_COLUMNS ic  ON i.OWNER = ic.INDEX_OWNER AND i.INDEX_NAME = ic.INDEX_NAME
LEFT JOIN ALL_CONSTRAINTS c ON i.OWNER = c.OWNER AND i.INDEX_NAME = c.INDEX_NAME
WHERE t.OWNER = UPPER(':schema')         -- UPPER for unquoted Oracle names
  AND t.TABLE_NAME IN (UPPER(':t1'), UPPER(':t2'), …)  -- all tables, one query
ORDER BY t.TABLE_NAME, i.INDEX_NAME, ic.COLUMN_POSITION;
```

### JDBC `getColumns()` fields vs INFORMATION_SCHEMA fields

The reference code uses JDBC metadata (engine-agnostic). When writing raw SQL against INFORMATION_SCHEMA, map accordingly:

**JDBC `IS_AUTOINCREMENT` and `IS_GENERATEDCOLUMN` do NOT map 1:1 to INFORMATION_SCHEMA across all engines — live-tested findings below.**

| JDBC field | MySQL INFORMATION_SCHEMA | PostgreSQL INFORMATION_SCHEMA | SQL Server INFORMATION_SCHEMA |
|-----------------------------|--------------------------|------------|------------|
| `COLUMN_DEF != 'Null'` (has any default) | `COLUMN_DEFAULT IS NOT NULL` | `column_default IS NOT NULL` | `COLUMN_DEFAULT IS NOT NULL` — **IMPORTANT: identity and rowversion columns have `COLUMN_DEFAULT = NULL`; this filter alone is INSUFFICIENT for SQL Server** |
| `IS_AUTOINCREMENT = 'YES'` | `EXTRA LIKE '%auto_increment%'` | `is_identity = 'YES'` — **SERIAL columns have `is_identity = 'NO'`; only `GENERATED AS IDENTITY` sets this to `'YES'`. SERIAL is caught by `column_default LIKE 'nextval(%'` instead.** | `COLUMNPROPERTY(OBJECT_ID(...),'IsIdentity') = 1` |
| `IS_GENERATEDCOLUMN = 'YES'` | `EXTRA LIKE '%GENERATED%'` | `is_generated = 'ALWAYS'` (computed/stored columns only; identity and serial are `'NEVER'`) | `is_computed = 1` in `sys.columns` |
| SQL Server rowversion special case | n/a | n/a | `DATA_TYPE = 'timestamp'` — `COLUMN_DEFAULT = NULL` and `IS_IDENTITY = 0`; only `DATA_TYPE` catches it |
| `IS_NULLABLE` | `IS_NULLABLE` | `is_nullable` | `IS_NULLABLE` |

**MySQL important:** `EXTRA LIKE '%GENERATED%'` catches `DEFAULT_GENERATED on update CURRENT_TIMESTAMP` columns. `COLUMN_DEFAULT IS NOT NULL` would also catch `dt` independently since `COLUMN_DEFAULT = 'CURRENT_TIMESTAMP'`.

**PostgreSQL critical:** SERIAL (`a SERIAL`) sets `column_default = nextval(...)` with `is_identity = 'NO'`. Only `GENERATED AS IDENTITY` sets `is_identity = 'YES'` with `column_default = NULL`. Both must be excluded from INSERT; they require different detection predicates.

**SQL Server critical:** IDENTITY columns and `timestamp` (rowversion) both have `COLUMN_DEFAULT = NULL` — neither is caught by `COLUMN_DEF != 'Null'`. The `IS_AUTOINCREMENT` check catches identity; the `TYPE_NAME = 'timestamp'` check catches rowversion. Both checks are necessary.

**Live-tested results:**

PostgreSQL 16.13 — `tmp_pg_id_test` with all four column types:
```
column  column_default             is_identity  is_generated
a       nextval('..._a_seq'::..)   NO           NEVER        -- SERIAL: caught by column_default != NULL
b       (null)                     YES          NEVER        -- GENERATED AS IDENTITY: caught by is_identity
c       (null)                     NO           ALWAYS       -- GENERATED AS STORED: caught by is_generated
d       42                         NO           NEVER        -- DEFAULT 42: caught by column_default != NULL
```

MySQL 8.0.44-azure — `intpk` and `dtix`:
```
TABLE  COLUMN    COLUMN_DEFAULT     EXTRA                                     ACTION
intpk  pk        NULL               auto_increment                            SKIP (IS_AUTOINCREMENT=YES)
intpk  dt        CURRENT_TIMESTAMP  DEFAULT_GENERATED on update CURRENT_TS    SKIP (IS_GENERATEDCOLUMN=YES)
intpk  ops       insert             (none)                                    SKIP (COLUMN_DEF!=Null)
intpk  fkr__unq  NULL               (none)                                    INCLUDE
dtix   pk        NULL               (none)                                    INCLUDE (nullable, no default)
dtix   dt        CURRENT_TIMESTAMP  DEFAULT_GENERATED on update CURRENT_TS    SKIP (IS_GENERATEDCOLUMN=YES)
dtix   ops       insert             (none)                                    SKIP (COLUMN_DEF!=Null)
dtix   fkr__unq  NULL               (none)                                    INCLUDE
```

SQL Server (Azure SQL 12.0.2000.8) — `#ts_test` with identity + rowversion + static default:
```
column  DATA_TYPE   COLUMN_DEFAULT  IS_IDENTITY_FLAG  ACTION
id      int         NULL            1                 SKIP (IS_AUTOINCREMENT=YES via COLUMNPROPERTY)
rv      timestamp   NULL            0                 SKIP (TYPE_NAME='timestamp')
val     int         ((42))          0                 SKIP (COLUMN_DEF!=Null)
```

### Decision rules from introspection output

| Finding | Action |
|---------|--------|
| `IS_AUTOINCREMENT = 'YES'` | **Omit from INSERT** — DB assigns |
| `IS_GENERATEDCOLUMN = 'YES'` | **Must omit from INSERT** — DB computes |
| `COLUMN_DEF != 'Null'` (any default) | **Omit from INSERT** — let DB use default |
| SQL Server `TYPE_NAME = 'timestamp'` | **Must omit** — rowversion auto-managed |
| `CONSTRAINT_TYPE = 'PRIMARY KEY'` and `IS_AUTOINCREMENT = 'NO'` | Use as seed: `castIntTo(a.value)` |
| `CONSTRAINT_TYPE = 'UNIQUE'` | Derive from PK seed to guarantee uniqueness |
| Regular index (`CONSTRAINT_TYPE = ''`) | Use `buildCastRand` range |
| FK (`REFERENCED_TABLE_NAME IS NOT NULL`) | Parent table must have rows first |
| `IS_NULLABLE = 'NO'` and `COLUMN_DEF = 'Null'` and not index | **Must generate** — INSERT fails otherwise |
| All columns omitted (non-Oracle) | Use `INSERT INTO table DEFAULT VALUES` |

### DB-managed columns: always omit from INSERT

If the database owns the column value (auto-increment, sequence, generated, or `ON UPDATE` timestamp), **do not specify it in the INSERT column list**. Let the database populate it. Attempting to set these columns is either an error or pointless — the value will be overwritten anyway.

**Detection: MySQL 8.0.44**
```sql
SELECT COLUMN_NAME, COLUMN_DEFAULT, EXTRA
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = :table;
```
✓ MySQL 8.0.44:

| EXTRA value | Meaning | Action |
|-------------|---------|--------|
| `auto_increment` | AUTO_INCREMENT PK | Omit from INSERT |
| `DEFAULT_GENERATED on update CURRENT_TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` | Omit — DB sets on INSERT and overwrites on any UPDATE |
| `DEFAULT_GENERATED` | `DEFAULT CURRENT_TIMESTAMP` (INSERT only, no ON UPDATE) | Omit — DB sets on INSERT; generator value would be overwritten by any later UPDATE anyway |
| `STORED GENERATED` / `VIRTUAL GENERATED` | Computed column | Must omit — INSERT is rejected |
| (empty, COLUMN_DEFAULT IS NOT NULL) | Static default (`DEFAULT 'insert'`, `DEFAULT 0`) | Can omit (DB uses default) or set explicitly |
| (empty, COLUMN_DEFAULT IS NULL, IS_NULLABLE = NO) | No default, NOT NULL | Must set — INSERT fails otherwise |

**Detection: PostgreSQL 16.13**
```sql
SELECT column_name, column_default, is_identity, is_generated
FROM information_schema.columns
WHERE table_schema = current_schema() AND table_name = :table;
```
✓ PostgreSQL 16.13:

| Signal | Meaning | Action |
|--------|---------|--------|
| `column_default LIKE 'nextval(%'` | SERIAL / SEQUENCE | Omit from INSERT |
| `is_identity = 'YES'` | GENERATED AS IDENTITY | Omit from INSERT |
| `is_generated = 'ALWAYS'` | GENERATED ALWAYS AS (expr) | Must omit |
| `column_default = 'CURRENT_TIMESTAMP'` | `DEFAULT CURRENT_TIMESTAMP` | Can omit; no ON UPDATE mechanism in PostgreSQL — value stays until explicit UPDATE |

**Detection: SQL Server (Azure SQL 12.0.2000.8)**
```sql
SELECT COLUMN_NAME, COLUMN_DEFAULT,
       COLUMNPROPERTY(OBJECT_ID(TABLE_NAME), COLUMN_NAME, 'IsIdentity') AS IS_IDENTITY
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_CATALOG = DB_NAME() AND TABLE_NAME = :table;
```
✓ SQL Azure 12.0.2000.8:

| Signal | Meaning | Action |
|--------|---------|--------|
| `IS_IDENTITY = 1` | IDENTITY column | Omit from INSERT |
| `COLUMN_DEFAULT LIKE '(getdate%)' OR LIKE '(GETUTCDATE%)'` | Timestamp default | Can omit; SQL Server has no `ON UPDATE CURRENT_TIMESTAMP` — value is set only at INSERT |
| `COLUMN_DEFAULT LIKE '(NEXT VALUE FOR %)'` | Sequence default | Can omit |

**Key difference — MySQL `ON UPDATE CURRENT_TIMESTAMP` vs other engines:**

MySQL is the only mainstream engine with `ON UPDATE CURRENT_TIMESTAMP`. This means any `UPDATE` to the row — even updating an unrelated column like `fkr__unq` — overwrites `dt`. PostgreSQL and SQL Server timestamp defaults are INSERT-only.

✓ MySQL 8.0.44: observed — running `UPDATE intpk SET fkr__unq = CAST(ops AS UNSIGNED)` reset all `dt` values from epoch-derived timestamps to the current wall time (`2026-04-29 15:06:07`).

**Reference implementation uses JDBC `DatabaseMetaData.getColumns()` fields, not INFORMATION_SCHEMA:**

The production code in `db_info.py` / `jdbc_client.py` uses three JDBC metadata fields — not engine-specific INFORMATION_SCHEMA `EXTRA` strings:

| JDBC field | Meaning | Skip condition |
|-----------|---------|---------------|
| `COLUMN_DEF` | Column default value string; JDBC normalizes SQL `NULL` → `"Null"` (the string) | Skip if `COLUMN_DEF != 'Null'` — i.e. **any** non-null default means skip |
| `IS_AUTOINCREMENT` | `'YES'` / `'NO'` | Skip if `'YES'` |
| `IS_GENERATEDCOLUMN` | `'YES'` / `'NO'` | Skip if `'YES'` (computed/stored/virtual columns) |

**SQL Server extra:** `TYPE_NAME != 'timestamp'` — SQL Server `timestamp` (rowversion) is auto-managed and cannot be set regardless of other flags.

**Key insight — `COLUMN_DEF` covers everything:**

`COLUMN_DEF != 'Null'` already excludes `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` (MySQL JDBC returns `'CURRENT_TIMESTAMP'` as the string for these columns), static string defaults, sequence defaults, and any other value. This one check handles all default-having columns portably across engines — no need to parse MySQL `EXTRA` or PostgreSQL `column_default` patterns.

**`DEFAULT_GENERATED` (MySQL INFORMATION_SCHEMA `EXTRA`) is NOT used by the reference code.** The JDBC `COLUMN_DEF` field is preferred because it is engine-agnostic.

**The actual pandas query from `db_info.py::set_regular_columns` (source of truth):**

```python
# MySQL / PostgreSQL / Oracle
null_cols = df.query(
    "TABLE_NAME.str.match('^{table}$', case=case_sensitive) "
    "and COLUMN_DEF == 'Null' "           # no default → include
    "and IS_AUTOINCREMENT != 'YES' "      # not identity/auto_increment
    "and IS_GENERATEDCOLUMN != 'YES' "    # not computed/virtual
    "and CATSCHTABCOL not in @index_cols" # not a PK/UQ/index column
    "and IS_NULLABLE == 'YES' "           # nullable columns pool
    "and not(USED_IN_PKUQ > 0)"           # not used in any PK/UQ
)

# SQL Server adds one more exclusion:
null_cols = df.query(
    "... and TYPE_NAME != 'timestamp'"    # SQL Server rowversion (auto-managed)
)
```

**Practical Python rule (engine-agnostic, mirrors reference code):**

```python
def should_skip_column(col, db_type):
    if col.IS_AUTOINCREMENT == 'YES':   return True   # identity / auto_increment
    if col.IS_GENERATEDCOLUMN == 'YES': return True   # computed / virtual / stored
    if col.COLUMN_DEF != 'Null':        return True   # has any default (DB will set it)
    # SQL Server rowversion: COLUMN_DEF IS NULL and IS_AUTOINCREMENT='NO', only DATA_TYPE distinguishes it
    if db_type == DbType.SQLSERVER and col.TYPE_NAME == 'timestamp': return True
    return False
```

**Why all four checks are needed** (not just `COLUMN_DEF != 'Null'`):
- MySQL identity: `COLUMN_DEFAULT = NULL` but `EXTRA LIKE '%auto_increment%'` → caught by `IS_AUTOINCREMENT`
- PG `GENERATED AS IDENTITY`: `column_default = NULL` → caught by `IS_AUTOINCREMENT` (only)
- SQL Server identity: `COLUMN_DEFAULT = NULL` → caught by `IS_AUTOINCREMENT` (only)
- SQL Server rowversion: `COLUMN_DEFAULT = NULL`, `IS_AUTOINCREMENT = 0` → caught by `TYPE_NAME` (only)

**Fallback when all columns are skipped — engine differences ✓ (live-tested):**

If every column is DB-managed (e.g. a table with only an identity PK and all-defaulted columns):

| Engine | Syntax | Works? |
|--------|--------|--------|
| PostgreSQL | `INSERT INTO t DEFAULT VALUES` | ✓ tested (16.13) |
| SQL Server | `INSERT INTO t DEFAULT VALUES` | ✓ tested (Azure SQL 12.0) |
| MySQL | `INSERT INTO t DEFAULT VALUES` | **FAILS** — `ERROR 1064: syntax error` |
| MySQL | `INSERT INTO t VALUES ()` (empty parens) | ✓ tested (8.0.44-azure) |
| Oracle | Must specify at least one column | raises if `colnames` is empty |

**All engines:** If a NOT NULL column has no default and is excluded from the INSERT column list, the insert will fail:
- PostgreSQL: `null value in column "x" violates not-null constraint` ✓
- MySQL: `ERROR 1364: Field 'x' doesn't have a default value` ✓
- SQL Server: similar error

The reference code uses `INSERT INTO table default values` for the non-Oracle path, which is correct for PostgreSQL and SQL Server but will fail on MySQL. MySQL requires `INSERT INTO table VALUES ()` instead.

**MySQL introspection query to classify all columns before building INSERT:**
```sql
SELECT
  COLUMN_NAME,
  COLUMN_DEFAULT,
  EXTRA,
  CASE
    WHEN EXTRA LIKE '%auto_increment%'    THEN 'SKIP — auto_increment'
    WHEN EXTRA LIKE '%DEFAULT_GENERATED%' THEN 'SKIP — DB default/ON UPDATE'
    WHEN EXTRA LIKE '%GENERATED%'         THEN 'SKIP — computed column'
    WHEN COLUMN_DEFAULT IS NOT NULL       THEN 'OPTIONAL — has static default'
    WHEN IS_NULLABLE = 'NO'              THEN 'REQUIRED — no default, NOT NULL'
    ELSE                                      'OPTIONAL — nullable, no default'
  END AS generator_action
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = :table
ORDER BY ORDINAL_POSITION;
```
✓ MySQL 8.0.44 on `intpk`/`dtix`:

| Table | Column | Action |
|-------|--------|--------|
| intpk | pk | SKIP — auto_increment |
| intpk | dt | SKIP — DB default/ON UPDATE |
| intpk | ops | OPTIONAL — has static default |
| intpk | fkr__unq | OPTIONAL — nullable, no default |
| dtix | pk | OPTIONAL — nullable, no default |
| dtix | dt | SKIP — DB default/ON UPDATE |
| dtix | ops | OPTIONAL — has static default |
| dtix | fkr__unq | OPTIONAL — nullable, no default |

**Correct INSERT for `intpk` (omit `pk` and `dt`):**
```sql
-- pk: auto_increment — DB assigns
-- dt: DEFAULT_GENERATED ON UPDATE — DB sets on INSERT, overwrites on any UPDATE
-- ops: optional — generator sets for seed-derived string
-- fkr__unq: optional — generator sets for range counting
INSERT INTO intpk (fkr__unq, ops)
SELECT
  a.pk + {OFFSET},
  LPAD(a.pk + {OFFSET}, 10, '0')
FROM fkr__seed a;
```

**Correct INSERT for `dtix` (omit `dt`):**
```sql
INSERT INTO dtix (pk, fkr__unq, ops)
SELECT
  FLOOR({OFFSET} + RAND() * {BATCH_SIZE}),
  a.pk + {OFFSET},
  LPAD(a.pk + {OFFSET}, 10, '0')
FROM fkr__seed a;
```


## Key type mappings (quick reference)

See [type-reference.md](type-reference.md) for complete per-engine tables.

### UUID / UNIQUEIDENTIFIER
| Engine | Type → Int | Int → Type |
|--------|-----------|-----------|
| SQL Server | `convert(bigint, convert(varbinary, x, 2))` | `convert(uniqueidentifier, convert(binary, FORMAT(cast(x as bigint),'X32'), 2))` |
| PostgreSQL | `('x'\|\|right(replace(cast(x as varchar),'-',''),16))::bit(64)::bigint` | `CAST(LPAD(TO_HEX(cast(x as bigint)),32,'0') AS UUID)` |
| MySQL | `cast(conv(x,16,10) as signed)` | `CAST(HEX(x) AS BINARY(32))` |
| Oracle | `TO_NUMBER(RAWTOHEX(x),'XXXX...')` | `LOWER(REGEXP_REPLACE(RAWTOHEX(SYS_GUID()),...))` — **NOT deterministic from x** |

### Timestamp / DateTime
| Engine | Type → Int | Int → Type |
|--------|-----------|-----------|
| SQL Server | `cast(DATEDIFF(s,'1970-01-01 00:00:00.000',x) as bigint)` | `DATEADD(ss, x, '01 JAN 1970')` |
| PostgreSQL | `cast(extract(epoch from x) as bigint)` | `timestamp '1970-01-01 00:00:00' + (x) * interval '1 second'` |
| MySQL | `TIME_TO_SEC(timediff(x,'1970-01-01 00:00:00'))` | `date_add('1970-01-01T00:00:00', interval x second)` |
| Oracle DATE | `(x - date '1970-01-01') * 86400` | `DATE '1970-01-01' + NUMTODSINTERVAL(x,'SECOND')` |
| Oracle TIMESTAMP | `(cast(x at time zone 'UTC' as date) - date '1970-01-01') * 86400` | `TIMESTAMP '1970-01-01 00:00:00' + NUMTODSINTERVAL(x,'SECOND')` |

### String / VARCHAR
| Engine | Int → Type |
|--------|-----------|
| SQL Server | `cast(RIGHT(CONCAT(replicate(cast('0' as varchar(max)),N), x), N) as VARCHAR(N))` |
| PostgreSQL | `RIGHT(CONCAT(encode(gen_random_bytes(N/2),'hex'), x), N)` |
| MySQL | `RIGHT(CONCAT(repeat('0',N), x), N)` |
| Oracle | `substr(lpad(x, N, '0'), -N)` |

## Stats collection: MIN/MAX for all tables in one UNION query

### The pattern

To know the current seed range of every table, the code collects `(table_name, min_pk, max_pk)` for all tables in **one SQL call** — up to 300 tables per execution — by building one subquery per table and joining them with `UNION`:

```python
per_table_subqueries = [minMaxSql(table) for table in tables]   # one per table
batched = " UNION ".join(per_table_subqueries[:300])             # one query, 300 tables
cursor.execute(batched)
```

Each subquery selects a literal `'table_name'` string as the join key so the caller knows which row belongs to which table after UNION.

### Two MIN/MAX strategies

#### Strategy 1 — `MIN()` / `MAX()` aggregate (most types)

Used for integer, varchar, timestamp, binary, and all types that support SQL aggregate functions:

```sql
-- One subquery per table; UNION'd together for bulk fetch
SELECT l.table_name, r.min, r.max, r.row_count
FROM
    (SELECT 'orders' AS table_name, NULL AS min, NULL AS max) l
LEFT JOIN
    (SELECT 'orders'              AS table_name,
            castToInt(MIN(pk))    AS min,
            castToInt(MAX(pk))    AS max,
            0                     AS row_count
     FROM orders
     WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)) r
  ON l.table_name = r.table_name
```

`castToInt(MIN(pk))` normalises the result to a bigint regardless of the column type, so all tables return the same output shape.

#### Strategy 2 — `ORDER BY … LIMIT 1` (PostgreSQL UUID only)

PostgreSQL `uuid` does not support `MIN()` / `MAX()` aggregates. Use two separate `ORDER BY … LIMIT 1` subqueries instead — one for min (ASC), one for max (DESC):

```sql
SELECT l.table_name,
       castToInt(rmin.min) AS min,
       castToInt(rmax.max) AS max,
       l.row_count
FROM
    (SELECT 'events' AS table_name, NULL AS min, NULL AS max, 0 AS row_count) l
LEFT JOIN
    (SELECT 'events' AS table_name, pk AS min
     FROM events
     WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)
     ORDER BY pk ASC  LIMIT 1) rmin
  ON l.table_name = rmin.table_name
LEFT JOIN
    (SELECT 'events' AS table_name, pk AS max
     FROM events
     WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)
     ORDER BY pk DESC LIMIT 1) rmax
  ON l.table_name = rmin.table_name    -- note: intentionally joins to rmin, not rmax
```

**Why `ORDER BY col ASC LIMIT 1` instead of `MIN(col)`:**
`MIN()`/`MAX()` are not defined for the `uuid` type in PostgreSQL. `ORDER BY ... LIMIT 1` is equivalent and works for any orderable type — but requires two subqueries (one per direction) where `MIN()`/`MAX()` returns both in one.

### Why LEFT JOIN (not INNER JOIN)

The anchor row `(SELECT 'table_name', NULL, NULL, 0)` is always present.  
LEFT JOIN to the data subquery means:
- **Empty table** → data subquery returns 0 rows → result is `(table_name, NULL, NULL, 0)` — table is visible but empty
- **Non-empty table** → result is `(table_name, min_int, max_int, 0)`
- Without LEFT JOIN, an empty table would return **zero rows** — you could not distinguish "empty" from "table not scanned"

### The range WHERE clause

Both strategies filter the scan to the known seed range:
```sql
WHERE castIntTo(0) <= pk AND pk <= castIntTo(999999999999)
-- castIntTo(0)            → type-specific lower bound (e.g. UUID '00000000-...')
-- castIntTo(999999999999) → type-specific upper bound, wrap_on_overflow=False
--                           so it caps at max representable value rather than wrapping
```
This prevents reading rows outside the synthetic data range (e.g. rows inserted by other processes).

### Batching limit: 300 subqueries per UNION

```python
max_union = 300   # from MyConfig.limits.max_union
# 1000 tables → ceil(1000/300) = 4 round trips
```

Each execution returns one row per table. Results are concatenated across batches.

### Types that DO NOT support MIN/MAX (require ORDER BY path)

| Type | Engine | Reason |
|------|--------|--------|
| `uuid` | PostgreSQL | No aggregate function defined for uuid |
| Any non-orderable type used as PK | Any | `ORDER BY col LIMIT 1` works for any orderable type |

All other types (int, bigint, varchar, timestamp, binary, etc.) use `MIN()`/`MAX()`.
## Batch sizing: avoid OOM on large inserts

### Rule

**Never insert all rows in one batch.** One INSERT SELECT that produces 100,000 rows with BLOB columns can exhaust DB working memory and crash the server. `fkr__seed` only needs to hold `max_batch_size` rows — run multiple batches with increasing offsets to reach the total.

### How it works

```
total_rows   = 10,000,000
batch_size   = 1,000,000    ← recommended default (14.5–15s/batch on Azure MySQL 8.0.44 ✓)
num_batches  = 10           ← ceil(total_rows / batch_size)

Batch 0: seed values       0..   999,999  → INSERT SELECT pk + 0       FROM fkr__seed
Batch 1: seed values 1,000,000.. 1,999,999 → INSERT SELECT pk + 1000000 FROM fkr__seed
...
Batch 9: seed values 9,000,000.. 9,999,999 → INSERT SELECT pk + 9000000 FROM fkr__seed
```

`fkr__seed` stays at `batch_size` rows (1,000,000). Each batch shifts the seed window via a constant offset, so each call to the DB generates at most `batch_size` rows at a time.

**Do not add external parallelism** (multiple connections running batches concurrently). The database already parallelizes the INSERT…SELECT internally across its CPU and I/O threads. Multiple connections cause lock contention on AUTO_INCREMENT and the UNIQUE index and are slower overall.

### Building fkr__seed at 1M rows via doubling ✓ (MySQL 8.0.44-azure)

Start from 10,000 rows (initial `fkr__seed` size) and double 7 times — total 8.5s:

```sql
-- Start: pk 0..9999 (10k rows)
INSERT INTO fkr__seed (pk) SELECT pk + 10000   FROM fkr__seed;              -- → 20k   (0.09s)
INSERT INTO fkr__seed (pk) SELECT pk + 20000   FROM fkr__seed;              -- → 40k   (0.22s)
INSERT INTO fkr__seed (pk) SELECT pk + 40000   FROM fkr__seed;              -- → 80k   (0.37s)
INSERT INTO fkr__seed (pk) SELECT pk + 80000   FROM fkr__seed;              -- → 160k  (0.68s)
INSERT INTO fkr__seed (pk) SELECT pk + 160000  FROM fkr__seed;              -- → 320k  (1.34s)
INSERT INTO fkr__seed (pk) SELECT pk + 320000  FROM fkr__seed;              -- → 640k  (2.73s)
INSERT INTO fkr__seed (pk) SELECT pk + 640000  FROM fkr__seed WHERE pk < 360000; -- → 1M (3.30s)
-- Result: pk 0..999999 (1,000,000 rows), 8.5s total
```

### Choosing batch size

Estimate average row size first, then cap batch size so the working set stays within a safe fraction of DB RAM:

```
target_working_set_mb  = 256          # conservative — adjust up if DB has ≥ 16 GB RAM
avg_row_bytes          = sum of max byte size of every column in the INSERT
batch_size             = min(1_000_000, floor(target_working_set_mb * 1_048_576 / avg_row_bytes))
num_batches            = ceil(total_rows / batch_size)
```

### Rough batch size caps by column type

| Dominant column type | Max bytes/row (typical) | Safe batch size |
|----------------------|------------------------|-----------------|
| INT / BIGINT / TIMESTAMP only | < 100 B | **1,000,000** ✓ |
| VARCHAR(255) × 5 | ~1.3 KB | 200,000 |
| TEXT / MEDIUMTEXT | 64 KB – 16 MB | 1,000 |
| BLOB / VARBINARY(65535) | 64 KB | 1,000 |
| MEDIUMBLOB / BYTEA large | 16 MB | 100 |
| LONGBLOB / CLOB × multiple columns | > 100 MB | 10 or fewer |

**Example danger case:** 3 × BLOB columns at 10 MB each → 30 MB/row. A batch of 1,000 rows requires 30 GB of working memory — certain OOM. Use batch size 5–10 and test empirically.

### MySQL batch loop pattern

```sql
-- fkr__seed has batch_size rows (e.g. 1,000,000 for INT/BIGINT/TIMESTAMP tables), not total_rows
-- Run this block num_batches times, incrementing :offset by batch_size each time

-- Batch :batch_num  (offset = batch_num * batch_size)
INSERT INTO target (dt, ops)
SELECT
  date_add('1970-01-01 00:00:01', interval a.value second),
  RIGHT(CONCAT(repeat('0', 255), a.value), 255)
FROM (
  SELECT pk + :offset AS value        -- shift seed window per batch
  FROM fkr__seed
  WHERE 0 <= pk AND pk < :batch_size
) a;

-- Validate after each batch (MIN/MAX, not COUNT*)
SELECT MIN(pk) AS min_pk, MAX(pk) AS max_pk, MAX(pk) - MIN(pk) + 1 AS estimated_count
FROM target;
```

### PostgreSQL batch loop pattern

```sql
-- generate_series with per-batch window
INSERT INTO target (col1, col2)
SELECT castIntTo(a.value, ...)
FROM generate_series(:offset, :offset + :batch_size - 1) a(value);
-- offset = batch_num * batch_size, starting at 0
```

### Determining batch size empirically

1. Start with `batch_size = 100` for any table containing LOB columns.
2. Run one batch, observe DB memory/CPU.
3. Double batch size until you see memory pressure or query time grows super-linearly.
4. Use 50% of the largest stable batch size as the production default.
5. Record the final value in the skill or config for the specific table schema.

### Batch size IS the fkr__seed size — resize it when changing batch size

`fkr__seed` is created with exactly `batch_size` rows (pk 0..batch_size-1). When you change the target batch size (e.g. 10k → 1M), resize `fkr__seed` first using the doubling pattern above. Do **not** make `fkr__seed` as large as `total_rows` — each batch just shifts the window via the offset constant.

**Resizing fkr__seed** uses the same doubling trick on itself (see "Building fkr__seed at 1M rows via doubling" above). No external data needed — the table bootstraps from its own existing rows.

## Timing: always use DB-native profiling, not client-side timing

Client-side `time` commands include network round-trip, connection setup, secret fetch, and firewall checks — typically 4–20× the actual DB execution time. Always use the database's own profiling to measure how long the SQL ran inside the engine.

### MySQL — `SET profiling = 1` + `SHOW PROFILES`

```sql
SET profiling = 1;

-- run your queries here
INSERT INTO target (fkr__unq, dt, ops) SELECT ... FROM fkr__seed a;

SELECT MIN(fkr__unq), MAX(fkr__unq), MAX(fkr__unq)-MIN(fkr__unq)+1, COUNT(*) FROM target WHERE fkr__unq IS NOT NULL;

-- read server-side duration for every statement in this session (no network time)
SHOW PROFILES;
-- Query_ID | Duration   | Query
--        1 | 0.00007300 | -- comment
--        2 | 0.24338925 | INSERT INTO target ...     ← pure DB execution time
--        3 | 0.08863150 | SELECT MIN(fkr__unq) ...
```

✓ MySQL 8.0.44: `SHOW PROFILES` still works despite deprecation notice. Deprecated in MySQL 8.0 in favour of `performance_schema.events_statements_history`, but remains functional. Use `performance_schema` for production monitoring; `SHOW PROFILES` is sufficient for ad-hoc testing.

### PostgreSQL — `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
INSERT INTO target (fkr__unq, dt, ops)
SELECT a.value, ... FROM generate_series(...) a(value);
-- "Execution Time: 13.488 ms" — pure server time, no network
```
✓ PostgreSQL 16.13: `EXPLAIN ANALYZE INSERT ... SELECT generate_series(0,9999)` returned `Execution Time: 13.488 ms`.

### SQL Server — `SET STATISTICS TIME ON`

```sql
SET STATISTICS TIME ON;

INSERT INTO target (fkr__unq, dt, ops) SELECT ... FROM fkr__seed a;

SET STATISTICS TIME OFF;
-- Output: SQL Server Execution Times:
--   CPU time = 0 ms,  elapsed time = 2 ms.
-- "elapsed time" is server-wall-clock, not including network transfer
```
✓ SQL Azure 12.0.2000.8: `SET STATISTICS TIME ON` output confirmed — `CPU time = 0 ms, elapsed time = 2 ms`.

### Oracle — `SET TIMING ON` (sqlplus)

```sql
SET TIMING ON
INSERT INTO target (fkr__unq, dt, ops) SELECT ... FROM fkr__seed a;
-- "Elapsed: 00:00:00.31"  — server-reported wall time
```

### Observed benchmark

✓ MySQL 8.0.44-azure, Azure MySQL Flexible Server (2 vCores), 2026-04-29:

| Operation | DB time (`SHOW PROFILES`) | Client elapsed | Network overhead ratio |
|-----------|--------------------------|---------------|----------------------|
| `INSERT … SELECT` 10,000 rows into `intpk` | **0.243 sec** | ~5.7 sec | ~23× |
| `SELECT MIN/MAX/COUNT` on 200k rows | **0.089 sec** | ~5.4 sec | ~60× |
| `SELECT MIN/MAX/COUNT (UNION ALL)` on 2 × 1M-row tables | **1.349 sec** | ~8.4 sec | ~6× |

✓ PostgreSQL 16.13, Azure PostgreSQL Flexible Server, 2026-04-29:

| Operation | DB time (`EXPLAIN ANALYZE`) |
|-----------|----------------------------|
| `INSERT … SELECT generate_series(0,9999)` 10,000 rows | **13.488 ms** |

In-DB approach achieves ~41,000 rows/sec (MySQL) vs. ~1,000–5,000 rows/sec for client-generated + prepared-statement upload. The client overhead is dominated by connection setup, secret fetch, and firewall checks — not the actual query.

### Scale benchmark: 10 million rows ✓ (MySQL 8.0.44-azure, 2026-04-29)

Table: `intpk_10m` — `pk BIGINT UNSIGNED AUTO_INCREMENT`, `dt TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, `ops VARCHAR(255) DEFAULT 'insert'`, `fkr__unq BIGINT UNSIGNED UNIQUE`. Only `fkr__unq` populated by generator; `pk`/`dt`/`ops` are DB-managed.

**Phase 1 — build `fkr__seed` 10k → 1M (7 doublings):**

| Step | DB time |
|------|---------|
| 10k → 20k | 0.089s |
| 20k → 40k | 0.222s |
| 40k → 80k | 0.370s |
| 80k → 160k | 0.683s |
| 160k → 320k | 1.339s |
| 320k → 640k | 2.731s |
| 640k → 1M | 3.301s |
| **Total** | **8.5s** |

**Phase 2 — 10 × 1M-row batches (offsets 0, 1M, 2M … 9M):**

| Batch | DB time |
|-------|---------|
| 0 (offset 0) | 14.56s |
| 1 (offset 1M) | 14.50s |
| 2 (offset 2M) | 14.84s |
| 3 (offset 3M) | 14.64s |
| 4 (offset 4M) | 14.51s |
| 5 (offset 5M) | 14.82s |
| 6 (offset 6M) | 15.04s |
| 7 (offset 7M) | 14.83s |
| 8 (offset 8M) | 14.99s |
| 9 (offset 9M) | 14.77s |
| **Total** | **~148s DB / 156s wall-clock** |

**Throughput: ~67,600 rows/sec** (DB time) into a table with a UNIQUE index on `fkr__unq` and AUTO_INCREMENT on `pk`.

Each batch cost is **flat at ~14.5–15s regardless of how full the table is** — InnoDB inserts are B-tree appends to the UNIQUE index and AUTO_INCREMENT does not cause a table scan.

**No external parallelism used.** The database parallelizes INSERT…SELECT internally. Multiple client connections add lock contention on AUTO_INCREMENT and the UNIQUE index and are slower overall.

## Seed column: controlled range index alongside AUTO_INCREMENT PK

When the PK is AUTO_INCREMENT / SERIAL / IDENTITY, the database owns its values and bulk inserts create gaps. Add a separate **seed column** named **`fkr__unq`** — a nullable `BIGINT UNSIGNED` with a `UNIQUE` index — that the generator fills with the exact seed integer. Apps that do not touch this column leave it `NULL`; the uniqueness constraint only applies to non-NULL values.

The `fkr__` prefix mirrors `fkr__seed` (the bootstrap numbers table) and signals that the column is owned by the synthetic data generator, not the application.

### DDL pattern

`fkr__unq` is **nullable** so existing application code that inserts rows without specifying it causes no constraint violation. NULL values are excluded from the unique index on all engines except SQL Server, which requires a filtered index.

```sql
-- MySQL / MariaDB — add to existing table
ALTER TABLE target ADD COLUMN fkr__unq BIGINT UNSIGNED NULL DEFAULT NULL;
ALTER TABLE target ADD UNIQUE INDEX uk_fkr__unq (fkr__unq);
-- Multiple NULLs are allowed — MySQL UNIQUE index skips NULL comparisons.
-- Or at CREATE time:
CREATE TABLE target (
  pk       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  fkr__unq BIGINT UNSIGNED NULL DEFAULT NULL,  -- NULL = app row, non-NULL = generated row
  dt       TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  ops      VARCHAR(255) DEFAULT 'insert',
  PRIMARY KEY (pk),
  UNIQUE  KEY uk_fkr__unq (fkr__unq)           -- NULLs not compared, so multiple NULLs OK
);

-- PostgreSQL
CREATE TABLE target (
  pk       BIGSERIAL PRIMARY KEY,
  fkr__unq BIGINT NULL DEFAULT NULL,
  ...
  CONSTRAINT uq_fkr__unq UNIQUE (fkr__unq)     -- NULLs not compared, multiple NULLs OK
);

-- SQL Server: plain UNIQUE allows only ONE NULL — use a filtered index instead
CREATE TABLE target (
  pk       BIGINT IDENTITY(1,1) NOT NULL PRIMARY KEY,
  fkr__unq BIGINT NULL DEFAULT NULL,
  ...
);
CREATE UNIQUE INDEX uq_fkr__unq ON target (fkr__unq) WHERE fkr__unq IS NOT NULL;
-- Filtered index enforces uniqueness only on non-NULL values.

-- Oracle
CREATE TABLE target (
  pk       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  fkr__unq NUMBER(19) NULL,
  ...
  CONSTRAINT uq_fkr__unq UNIQUE (fkr__unq)     -- Oracle UNIQUE ignores NULLs
);
```

### NULL behavior in UNIQUE indexes by engine

| Engine | Multiple NULLs in UNIQUE index? | Solution | Verified |
|--------|---------------------------------|---------|---------|
| MySQL / MariaDB | Allowed | Plain `UNIQUE KEY` | ✓ MySQL 8.0.44: Added UNIQUE INDEX on 200k-row all-NULL column — succeeded |
| PostgreSQL | Allowed | Plain `UNIQUE` constraint | ✓ PostgreSQL 16.13: Inserted 3 NULLs into UNIQUE column — all succeeded |
| SQL Server | Only one NULL allowed | Filtered index: `WHERE fkr__unq IS NOT NULL` | ✓ SQL Azure 12.0.2000.8: `Msg 2601: duplicate key value is (<NULL>)` on second NULL |
| Oracle | Allowed (NULL != NULL) | Plain `UNIQUE` constraint | ⚠ not tested — no Oracle DB available |

**SQL Server filtered index requires `SET QUOTED_IDENTIFIER ON`** before creation (sqlcmd may not set this by default):
```sql
SET QUOTED_IDENTIFIER ON;
CREATE UNIQUE INDEX uq_fkr__unq ON target (fkr__unq) WHERE fkr__unq IS NOT NULL;
```
✓ SQL Azure 12.0.2000.8: `Msg 1934` (CREATE INDEX failed) observed when `QUOTED_IDENTIFIER` was not set.

### INSERT pattern — offset shift on fkr__seed

`fkr__seed` holds exactly `batch_size` rows (pk 0..batch_size-1). Each batch shifts the seed window by adding a constant OFFSET to `a.pk`. `fkr__seed` size equals the batch size — resize it using the doubling pattern when changing the target batch size.

```sql
-- MySQL — batch N, OFFSET = N * batch_size
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},
  date_add('1970-01-01 00:00:01', interval (a.pk + {OFFSET}) second),
  LPAD(a.pk + {OFFSET}, 10, '0')
FROM fkr__seed a;
-- fkr__seed has pk 0..9999 (batch_size=10000); shift gives seeds OFFSET..OFFSET+9999

-- PostgreSQL
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.value,
  timestamp '1970-01-01 00:00:00' + a.value * interval '1 second',
  LPAD(a.value::text, 10, '0')
FROM generate_series({OFFSET}, {OFFSET} + {batch_size} - 1) a(value);
```

### Backfill fkr__unq on existing rows (after ALTER)

When adding `fkr__unq` to an already-populated table, backfill from the seed-derived column (`ops` stores the zero-padded seed):

```sql
-- MySQL: recover seed integer from ops column, assign to fkr__unq
UPDATE target SET fkr__unq = CAST(ops AS UNSIGNED);
```

### Validation via fkr__unq — filter NULLs

Always add `WHERE fkr__unq IS NOT NULL` to exclude app-inserted rows:

```sql
-- Fast: two index lookups on fkr__unq, generator rows only
SELECT
  MIN(fkr__unq)                      AS min_seed,
  MAX(fkr__unq)                      AS max_seed,
  MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count
FROM target
WHERE fkr__unq IS NOT NULL;

-- Full check: confirm no gaps, count matches
SELECT
  MIN(fkr__unq)                      AS min_seed,
  MAX(fkr__unq)                      AS max_seed,
  MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count,
  COUNT(*)                            AS actual_count,
  MAX(fkr__unq) - MIN(fkr__unq) + 1
    = COUNT(*)                        AS contiguous
FROM target
WHERE fkr__unq IS NOT NULL;
-- contiguous = 1 means seed range is intact with no holes
```

### When fkr__unq doesn't exist on an existing table

Use the seed-derived data column (e.g. `ops` stores the zero-padded seed integer):

```sql
SELECT
  MIN(cast(ops AS signed))                                  AS min_seed,
  MAX(cast(ops AS signed))                                  AS max_seed,
  MAX(cast(ops AS signed)) - MIN(cast(ops AS signed)) + 1   AS estimated_count,
  COUNT(*)                                                   AS actual_count,
  MAX(cast(ops AS signed)) - MIN(cast(ops AS signed)) + 1
    = COUNT(*)                                               AS contiguous
FROM target;
```
This works but is slower than an indexed integer column. Prefer `fkr__unq BIGINT UNIQUE` for any table designed for synthetic data testing.

### Why UNSIGNED BIGINT (nullable)

- Seed values start at 0 — UNSIGNED avoids wasting half the range
- `NULL DEFAULT NULL` means apps never see a constraint violation when omitting the column
- For 100M+ rows, `BIGINT` (0 to 18.4 × 10¹⁸) is never a limit
- PostgreSQL `BIGINT` is always signed — start seeds at 0, still fine
- Oracle uses `NUMBER(19)` (signed, sufficient for any practical seed range)

### Summary: PK vs fkr__unq roles

| Column | Managed by | NULL? | Contiguous? | Use for counting? |
|--------|-----------|-------|-------------|-------------------|
| `pk` AUTO_INCREMENT | Database | No | No (bulk gaps) | No |
| `fkr__unq BIGINT UNIQUE NULL` | Generator | Yes (app rows) | Yes (non-NULL) | Yes — `MAX - MIN + 1 WHERE IS NOT NULL` |
| `ops VARCHAR` (existing tables) | Generator (string-encoded) | No | Yes (cast needed) | Fallback — slower |
## Randomizing data values within a batch

Sequential inserts (seed 0, 1, 2 …) produce obviously artificial data: timestamps always advance by one second, strings are always zero-padded incrementing numbers. To produce more realistic-looking data, randomize the seed used for **data columns** while keeping `fkr__unq` sequential (it must stay unique and contiguous for counting).

### Rule

- **`fkr__unq`** — always `a.pk + OFFSET` (sequential, unique, used for range counting). Never randomize this.
- **All data columns** (`dt`, `ops`, varchar, numeric, etc.) — randomize within the **current batch's own window** `[OFFSET, OFFSET + BATCH_SIZE)`, not across `[0, total_rows)`.

#### Why batch-bounded randomization matters

Using `FLOOR(RAND() * total_rows)` samples from the full range on every batch. Early batches (low OFFSET) will draw the same pool of values as late batches, causing lower-range values to appear in every batch and producing a non-uniform distribution overall. Bounding to `[OFFSET, OFFSET + BATCH_SIZE)` ensures each batch contributes exactly one window of coverage so the final distribution is flat:

```
FLOOR(RAND() * total_rows)           →  all batches sample [0, 1M)  — lower values over-represented
FLOOR(OFFSET + RAND() * BATCH_SIZE)  →  batch N samples [N*B, (N+1)*B) — flat coverage
```

A small number of duplicate *data values* across rows is acceptable and expected with this approach. The unique constraint is only on `fkr__unq`, not on data columns.

### MySQL pattern

```sql
-- OFFSET = batch_num * batch_size
-- Each batch's data columns draw from [OFFSET, OFFSET + BATCH_SIZE)
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},                                                        -- sequential control key
  date_add('1970-01-01 00:00:01',
    interval FLOOR({OFFSET} + RAND() * {BATCH_SIZE}) second),             -- random within batch window
  LPAD(FLOOR({OFFSET} + RAND() * {BATCH_SIZE}), 10, '0')                 -- random within batch window
FROM fkr__seed a;
```

### PostgreSQL pattern — use `random()`, not modulo

```sql
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.value,                                                                            -- sequential control key
  timestamp '1970-01-01 00:00:00'
    + (floor({OFFSET} + random() * {BATCH_SIZE}))::int * interval '1 second',        -- truly random per row
  LPAD((floor({OFFSET} + random() * {BATCH_SIZE}))::text, 10, '0')
FROM generate_series({OFFSET}, {OFFSET} + {BATCH_SIZE} - 1) a(value);
```

✓ PostgreSQL 16.13: `random()` IS evaluated per row in a multi-row `INSERT ... SELECT` — each row gets a different value.

**Do NOT use modulo `(x % batch_size) + offset`** for data columns in PostgreSQL. When `a.value` comes from `generate_series(OFFSET, OFFSET+BATCH_SIZE-1)` and OFFSET is a multiple of BATCH_SIZE, `(a.value % batch_size) + offset = a.value` — the modulo is a no-op and the data is **sequential, not randomized**. ✓ PostgreSQL 16.13: 10,000 rows from `generate_series(50000, 59999)` with formula `(v % 10000) + 50000` returned `v` for every row — confirmed by query verdict `"SEQUENTIAL — modulo is a no-op"`.

The reference code in `cast_int_to_x.py` uses modulo for PostgreSQL in the multi-row path for deterministic reproducibility, not randomization. Use `random()` when randomized data values are the goal.

### SQL Server pattern — `RAND(CHECKSUM(NEWID()))` required, not `RAND()`

```sql
INSERT INTO target (fkr__unq, dt, ops)
SELECT
  a.pk + {OFFSET},
  DATEADD(second, {OFFSET} + FLOOR(RAND(CHECKSUM(NEWID())) * {BATCH_SIZE}), '1970-01-01 00:00:01'),
  RIGHT(REPLICATE('0', 10)
    + CAST({OFFSET} + FLOOR(RAND(CHECKSUM(NEWID())) * {BATCH_SIZE}) AS VARCHAR), 10)
FROM fkr__seed a;
```

**Critical:** `RAND()` in SQL Server is evaluated **once per query**, not per row. Every row in a multi-row INSERT SELECT gets the **same value** if you use plain `RAND()`. Fix: `RAND(CHECKSUM(NEWID()))` — `NEWID()` generates a unique UUID per row, `CHECKSUM()` converts it to an integer seed, `RAND(seed)` produces a different float for each row. ✓ SQL Azure 12.0.2000.8: confirmed `RAND()` = `0.65816…` identical for all 5 rows; `RAND(CHECKSUM(NEWID()))` = 5 distinct values.

### Per-engine random function behavior in INSERT SELECT

| Engine | Safe to use plain `RAND()`? | Multi-row INSERT SELECT formula | Verified by live test |
|--------|----------------------------|--------------------------------|----------------------|
| SQL Server | **No — same value all rows** | `RAND(CHECKSUM(NEWID()))` | ✓ SQL Azure 12.0.2000.8: `RAND()` gave `0.65816…` for all 5 rows; `RAND(CHECKSUM(NEWID()))` gave 5 different values |
| PostgreSQL | Yes (`random()` per row) | `floor(offset + random() * batch_size)` | ✓ PostgreSQL 16.13: 5 different values; `MIN=50000, MAX=59997` across 10k rows |
| MySQL | Yes (`RAND()` per row) | `FLOOR(offset + RAND() * batch_size)` | ✓ MySQL 8.0.44: 5 different values; `MIN=50000, MAX=59999` across 10k rows |
| Oracle | Yes (`dbms_random.value()` per row) | `FLOOR(dbms_random.value() * batch_size) + offset` | ⚠ not tested — no Oracle DB available |

### DataGenRange modes (from `my_config.py`)

The reference implementation defines three modes selectable per table/column:

| Mode | Formula | Notes |
|------|---------|-------|
| `MONOTONIC_INCREASE` | `x` (the seed directly) | Sequential, matches PK value exactly |
| `RANDOM_ROW_RANGE` *(default)* | `random() * batch_size + batch_offset` | Bounded to current batch window — slight skew to lower values is normal and considered realistic |
| `RANDOM_ROW_COUNT` | `random() * total_rows` | Spread across full table range — lower values over-represented in early batches |

**Default is `RANDOM_ROW_RANGE`** — each batch's data values are randomly drawn from its own window `[offset, offset+batch_size)`. This is the recommended mode for realistic-looking data with a flat distribution across the full range.

### Validation is unchanged

`fkr__unq` is still sequential, so range counting works exactly as before:
```sql
SELECT MIN(fkr__unq), MAX(fkr__unq),
  MAX(fkr__unq) - MIN(fkr__unq) + 1 AS estimated_count,
  COUNT(*) AS actual_count,
  MAX(fkr__unq) - MIN(fkr__unq) + 1 = COUNT(*) AS contiguous
FROM target WHERE fkr__unq IS NOT NULL;
```

## Foreign key constraints: populate parent first, bound child FK to parent range

When a child table has a FK referencing a parent, the FK column value must exist in the parent. Use the parent's already-inserted `fkr__unq` range to bound the random FK values so every generated row is guaranteed to satisfy the constraint.

### Steps

1. Populate the **parent** table first (any order within a batch).
2. Query `MIN(fkr__unq)` and `MAX(fkr__unq)` from the parent — this is the valid FK range.
3. For the **child** table's FK column, use `FLOOR(min_parent + RAND() * (max_parent - min_parent + 1))`.

### MySQL example

```sql
-- Step 1: populate parent (categories) — fkr__unq 0..99999
-- (already done via batch inserts)

-- Step 2: read parent range
SELECT MIN(fkr__unq) INTO @fk_min FROM categories WHERE fkr__unq IS NOT NULL;
SELECT MAX(fkr__unq) INTO @fk_max FROM categories WHERE fkr__unq IS NOT NULL;

-- Step 3: populate child, FK column bounded to parent range
INSERT INTO orders (fkr__unq, category_id, dt, ops)
SELECT
  a.pk + {OFFSET},                                                  -- child's own sequential key
  FLOOR(@fk_min + RAND() * (@fk_max - @fk_min + 1)),               -- FK → valid parent row
  date_add('1970-01-01 00:00:01',
    interval FLOOR({OFFSET} + RAND() * {BATCH_SIZE}) second),             -- batch-bounded random
  LPAD(FLOOR({OFFSET} + RAND() * {BATCH_SIZE}), 10, '0')
FROM fkr__seed a;
```

### Why this works even with randomization

- Parent `fkr__unq` is a contiguous range `[min_parent, max_parent]`, so any integer in that range is guaranteed to be a valid FK target.
- If the parent PK is auto_increment (with gaps), use the parent's `fkr__unq` column as the FK target and create a UNIQUE index on it — the child FK column then references `fkr__unq`, not the auto-assigned `pk`.
- If the parent already has holes in `fkr__unq` (partial deletes), query the actual min/max before each batch to avoid invalid FK values.

## Range maintenance and verification

**The data is always a contiguous integer range — only when you control the PK.** Inserts extend the high end; deletes shrink from the low or high end. There are never holes in the PK range.

**Exception — AUTO_INCREMENT / SERIAL / IDENTITY columns:** Do NOT use `MAX(pk) - MIN(pk) + 1` as the row count for auto-assigned PKs. MySQL 8 InnoDB with `innodb_autoinc_lock_mode=2` (the default) allocates AUTO_INCREMENT values in bulk during INSERT SELECT, producing gaps between batches. A table with 20,000 rows may show `MAX(pk) = 26,383` after two 10,000-row bulk inserts. Use the **seed-derived column** (the column whose value encodes the integer seed, e.g. `ops`) for contiguous-range counting instead.

### Never use COUNT(*) on large tables

`COUNT(*)` requires a full table scan and is expensive at scale. When the PK (or any indexed unique column) is a contiguous integer range, use `MAX - MIN + 1` instead — it is satisfied by two index lookups:

```sql
-- Fast: two index lookups only (valid ONLY for non-auto-increment PKs you control)
SELECT
  MIN(pk)                AS min_pk,
  MAX(pk)                AS max_pk,
  MAX(pk) - MIN(pk) + 1  AS estimated_count
FROM target;
```

This is valid only when the range is contiguous — no gaps. For AUTO_INCREMENT columns, use the **seed-derived column** instead:

```sql
-- For AUTO_INCREMENT PK tables: count via seed-derived column (e.g. ops = zero-padded seed int)
SELECT
  MIN(cast(ops AS signed))                                    AS min_seed,
  MAX(cast(ops AS signed))                                    AS max_seed,
  MAX(cast(ops AS signed)) - MIN(cast(ops AS signed)) + 1    AS estimated_count,
  COUNT(*)                                                    AS actual_count
FROM target;
-- estimated_count = actual_count confirms no duplicate/missing seeds
```

### Verify a step completed correctly

After any INSERT batch:
```sql
SELECT MIN(pk) AS min_pk, MAX(pk) AS max_pk, MAX(pk) - MIN(pk) + 1 AS estimated_count
FROM target;
-- estimated_count should equal the number of rows inserted
```

After any DELETE batch:
```sql
SELECT MIN(pk) AS min_pk, MAX(pk) AS max_pk, MAX(pk) - MIN(pk) + 1 AS estimated_count
FROM target;
-- min_pk should have increased (delete from low end)
-- OR max_pk should have decreased (delete from high end)
```

### Delete patterns

Deletes always operate on one end of the range to preserve contiguity.

**Delete from the low end (oldest rows):**
```sql
DELETE FROM target WHERE pk < :new_low_watermark;
-- or in batches:
DELETE FROM target WHERE pk BETWEEN :low AND :low + :batch_size - 1;
```

**Delete from the high end (most recent rows):**
```sql
DELETE FROM target WHERE pk > :new_high_watermark;
-- or in batches:
DELETE FROM target WHERE pk BETWEEN :high - :batch_size + 1 AND :high;
```

**Never delete from the middle** — it creates gaps and breaks the `MAX - MIN + 1` count invariant.

**Delete direction rule for AUTO_INCREMENT / IDENTITY columns:**

When the PK is auto-assigned (`IS_AUTOINCREMENT == 'YES'`), always delete from the **low end (head)** — not the high end — because identity values grow upward and deleting from the high end leaves the next INSERT starting at a much higher value:

```python
# From jdbc_client.py::getDeleteStart
if pk_df.iloc[0].IS_AUTOINCREMENT == 'YES':
    delete_from_head = True   # delete from min(pk) end
```

**Cannot UPDATE an AUTO_INCREMENT / IDENTITY PK:**

Identity PKs are DB-assigned; you cannot shift them. The reference code returns `None` from `updateRangePk` when `IS_AUTOINCREMENT == 'YES'`:

```python
# jdbc_client.py::updateRangePk
if pk_df.iloc[0].IS_AUTOINCREMENT == 'YES':
    return None   # identity PK values cannot be reassigned
```

**Cannot UPDATE columns used in UNIQUE keys:**

The update column pool uses `table_null_notnull_nouqifpk_columns` — unique-key columns are excluded from UPDATEs to avoid constraint violations when shifting values.

## Adding a new type mapping

1. **`cast_int_to_x.py`**: write `castIntToXxx(x, column_size, decimal_digits, db_type, cast_type, config, ...)` returning a SQL expression string per engine.
2. **`castFuncs` dict** (same file): add `JavaSqlTypes.XXX.name.upper(): castIntToXxx`.
3. **`cast_x_to_int.py`**: write `castXxxToBigInt(...)` and add to `castXToIntFuncs`.
4. **`convert_int_to_x.py`**: write `convertIntToXxx(...)` returning a Python value for prepared statements; add to `convertFuncs`.
5. **`java_sql_types.py`**: confirm the type exists in `JavaSqlTypes`; add if not.

For parameterized Oracle types like `TIMESTAMP(6) WITH TIME ZONE`, add a regex branch to `getCastFunction()` instead of polluting `castFuncs` with every parameter combination.

## Known gaps (JavaSqlTypes with no castFuncs entry)

| Type | Suggested approach |
|------|--------------------|
| `SERIAL4` / `SERIAL8` | Delegate to `castIntToInteger` / `castIntToBigInt` (SERIAL is an alias) |
| `ROWVERSION` | Treat as `BINARY(8)` — SQL Server auto-managed, skip in INSERT |
| `LONG_RAW` | Delegate to `castIntToBinary` with Oracle RAW semantics |
| `BFILE` | `BFILENAME('MY_DIR', lpad(to_char(x),20,'0') || '.bin')` — needs DIRECTORY object |
| `SQLXML` | Add key `SQLXML → castIntToXml` (synonym for XML at SQL level) |
| `NULL` | Return literal `NULL` |
| `ARRAY` | `ARRAY(SELECT castIntTo(v, elem) FROM generate_series(1,N) v)` for PostgreSQL |
| `STRUCT` | Engine-specific: `ROW(x, x+1)` (PG) or object constructor (Oracle) |
| `REF_CURSOR` | Not insertable — skip |
| `DATALINK` | `DATALINK(CAST('http://example.com/' || x AS VARCHAR(200)))` |
| `OTHER` / `DISTINCT` / `REF` | Fall back to `castIntToInteger`; log warning |

## Engine-specific gotchas

- **MySQL AUTO_INCREMENT gaps (`innodb_autoinc_lock_mode=2`):** ✓ MySQL 8.0.44: bulk INSERT SELECT allocates AUTO_INCREMENT in pre-allocated ranges per batch — two 10,000-row batches produced pk 1–10000 then 16384–26383 (not 10001–20000). `MAX(pk) - MIN(pk) + 1` overcounts. Use seed-derived column for verification, not the AUTO_INCREMENT pk. Behaviour tied to `innodb_autoinc_lock_mode=2` (default since MySQL 8.0).
- **MySQL TIMESTAMP epoch zero**: ✓ MySQL 8.0.44: `date_add('1970-01-01 00:00:00', interval 0 second)` is rejected on INSERT with `ERROR 1292 Incorrect datetime value` — `1970-01-01 00:00:00` is the reserved zero/NULL sentinel. Use `'1970-01-01 00:00:01'` as base epoch. Round-trip check: `TIME_TO_SEC(timediff(dt, '1970-01-01 00:00:01')) = cast(ops AS signed)`. **Side note:** columns with `ON UPDATE CURRENT_TIMESTAMP` have `dt` overwritten on any UPDATE (e.g. `fkr__unq` backfill) — epoch values only survive if the row is never UPDATEd after INSERT. ✓ MySQL 8.0.44: observed all dt values reset to `2026-04-29 15:06:07` after backfill UPDATE.
- **MySQL FLOAT/DOUBLE**: ✓ MySQL 8.0.44: `CAST(x AS FLOAT)` and `CAST(x AS DOUBLE)` both work correctly. `CAST AS FLOAT` was added in MySQL 8.0.17 — for MySQL < 8.0.17, use `(x * 1.0)`. The reference code uses `(x * 1.0)` for compatibility with older versions.
- **MySQL integer → type**: bare `mod(x, N)` without explicit cast. ⚠ not yet live-tested on a full column round-trip.
- **Oracle UUID**: `castIntToUniqueidentifier` calls `SYS_GUID()` — output is random, not derived from `x`. ⚠ not tested — no Oracle DB available.
- **Oracle types**: all integer subtypes map to `NUMBER(n)`. ⚠ not tested — no Oracle DB available.
- **Oracle timing `SET TIMING ON`**: ⚠ not tested — no Oracle DB available.
- **PostgreSQL BYTEA**: strips `\x` prefix from hex string before conversion. ⚠ not yet live-tested.
- **SQL Server string padding**: uses `replicate(cast('0' as varchar(max)), N)` not `repeat()`. ⚠ not yet live-tested — derived from source code reading.
- **Non-PK column randomness via modulo**: ✓ PostgreSQL 16.13: `buildCastRand` in reference code uses `(x % length) + start` for PostgreSQL multi-row path — **produces sequential data**, not random, when `x` is from `generate_series(offset, offset+batch-1)` and offset is a multiple of batch_size. Confirmed by live query returning `"SEQUENTIAL — modulo is a no-op"`. Use `floor(offset + random() * batch_size)` for actual randomization.
