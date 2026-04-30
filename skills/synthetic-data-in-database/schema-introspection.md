# Schema Introspection — PK, UNIQUE, INDEX, FK

## Batching principle — one round trip per operation

Never query one table at a time. Retrieve metadata for **all tables in one SQL call**:

| Operation | Pattern |
|-----------|---------|
| Column discovery | `WHERE TABLE_SCHEMA = :catalog` — no table filter; returns all tables at once. Filter in application code. |
| Index / PK / UNIQUE | `WHERE TABLE_SCHEMA = :catalog AND TABLE_NAME IN ('t1','t2',…)` — all tables in one query. |
| Stats collection | `UNION ALL` subqueries per table, chunked at **max 300** per execution. |

**IN-list construction** (no prepared statements for index queries):
```python
table_names_sql = ",".join([f"'{t}'" for t in table_names])
where = f"TABLE_NAME IN ({table_names_sql})"
```

**Prepared statement param limits by engine:**
| Engine | Param limit |
|--------|------------|
| SQL Server ≥ v8 | 2,100 |
| SQL Server v7 | 1,024 |
| PostgreSQL | 32,767 (use `unnest()` for larger arrays) |
| MySQL / Oracle | No hard limit documented |

## Case sensitivity per engine

| Engine | Default | Quote char | Meaning |
|--------|---------|------------|---------|
| MySQL | `DEFAULT` (case-insensitive) | `` ` `` | Unquoted identifiers are case-insensitive |
| PostgreSQL (raw) | `DEFAULT` (case-insensitive unquoted) | `"` | Unquoted = case-insensitive; `"Name"` = case-sensitive |
| PostgreSQL (managed/Databricks) | `QUOTE` | `"` | Always double-quote; triggers case-sensitive matching |
| SQL Server | `CASE_INSENSITIVE` | `"` | Always case-insensitive |
| Oracle | `QUOTE` | `"` | Unquoted identifiers stored as UPPERCASE |

- INFORMATION_SCHEMA `WHERE` clauses use literal equality — no `LOWER()`/`UPPER()` in SQL
- Case-sensitivity applied **after fetch** using pandas `.str.match('^name$', case=case_sensitive_search)`
- Oracle: unquoted names stored as UPPERCASE — use `UPPER(:name)` when filtering

## SQL templates — one query for all tables

### MySQL
```sql
-- Columns: one call for entire schema
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = ':catalog'
ORDER BY TABLE_NAME, ORDINAL_POSITION;

-- PK / UNIQUE / INDEX: all tables
SELECT TABLE_SCHEMA AS TABLE_CAT, 'Null' AS TABLE_SCHEM,
    TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX AS KEY_SEQ,
    CASE WHEN INDEX_NAME = 'PRIMARY' THEN 'PRIMARY KEY'
         WHEN NON_UNIQUE = 0         THEN 'UNIQUE'
         ELSE '' END AS CONSTRAINT_TYPE,
    INDEX_TYPE AS TYPE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = ':catalog'
  AND TABLE_NAME IN (':t1',':t2', …)
ORDER BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME;

-- Foreign keys
SELECT COLUMN_NAME, CONSTRAINT_NAME,
       REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = ':catalog'
  AND TABLE_NAME IN (':t1',':t2', …)
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

### PostgreSQL
```sql
-- Columns
SELECT * FROM information_schema.columns
WHERE table_catalog = ':catalog' AND table_schema = ':schema'
ORDER BY table_name, ordinal_position;

-- PK / UNIQUE / INDEX
SELECT ':catalog' AS TABLE_CAT, tnsp.nspname AS TABLE_SCHEM,
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
WHERE a.attnum = ANY(ix.indkey)
  AND trel.relkind = 'r'
  AND tnsp.nspname = ':schema'
  AND trel.relname IN (':t1',':t2', …)
ORDER BY tnsp.nspname, trel.relname, i.relname,
         array_position(ix.indkey, a.attnum);
-- Large IN list: WHERE trel.relname = ANY(ARRAY[':t1',':t2',…])
```

### SQL Server
```sql
-- Columns
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_CATALOG = ':catalog' AND TABLE_SCHEMA = ':schema'
ORDER BY TABLE_NAME, ORDINAL_POSITION;

-- PK / UNIQUE / INDEX
SELECT db_name() AS TABLE_CAT, s.name AS TABLE_SCHEM,
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
  AND t.name IN (':t1',':t2', …)
ORDER BY s.name, t.name, ind.name, ic.key_ordinal;
```

### Oracle
```sql
-- Columns
SELECT * FROM ALL_TAB_COLUMNS
WHERE OWNER = UPPER(':schema')
ORDER BY TABLE_NAME, COLUMN_ID;

-- PK / UNIQUE / INDEX
SELECT ':catalog' AS TABLE_CAT, t.OWNER AS TABLE_SCHEM,
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
WHERE t.OWNER = UPPER(':schema')
  AND t.TABLE_NAME IN (UPPER(':t1'), UPPER(':t2'), …)
ORDER BY t.TABLE_NAME, i.INDEX_NAME, ic.COLUMN_POSITION;
```

## JDBC fields vs INFORMATION_SCHEMA

The reference code uses JDBC metadata (engine-agnostic). Key mappings:

| JDBC field | MySQL | PostgreSQL | SQL Server |
|------------|-------|------------|------------|
| `IS_AUTOINCREMENT = 'YES'` | `EXTRA LIKE '%auto_increment%'` | `is_identity = 'YES'` (SERIAL is NOT this — use `column_default LIKE 'nextval(%'`) | `COLUMNPROPERTY(...,'IsIdentity') = 1` |
| `IS_GENERATEDCOLUMN = 'YES'` | `EXTRA LIKE '%GENERATED%'` | `is_generated = 'ALWAYS'` | `is_computed = 1` in `sys.columns` |
| `COLUMN_DEF != 'Null'` | `COLUMN_DEFAULT IS NOT NULL` | `column_default IS NOT NULL` | **Insufficient for SQL Server** — identity + rowversion have NULL here |
| SQL Server rowversion | n/a | n/a | `DATA_TYPE = 'timestamp'` |

**PostgreSQL critical:** SERIAL sets `column_default = nextval(...)` with `is_identity = 'NO'`. Only `GENERATED AS IDENTITY` sets `is_identity = 'YES'`. Both must be excluded from INSERT but need different detection predicates.

**`COLUMN_DEF` covers most cases:** `COLUMN_DEF != 'Null'` already excludes `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, static defaults, and sequence defaults portably — the reference code prefers it over parsing engine-specific `EXTRA` strings.

## Decision rules from introspection output

| Finding | Action |
|---------|--------|
| `IS_AUTOINCREMENT = 'YES'` | **Omit from INSERT** — DB assigns |
| `IS_GENERATEDCOLUMN = 'YES'` | **Must omit from INSERT** — DB computes |
| `COLUMN_DEF != 'Null'` (any default) | **Omit from INSERT** — let DB use default |
| SQL Server `TYPE_NAME = 'timestamp'` | **Must omit** — rowversion auto-managed |
| PK with `IS_AUTOINCREMENT = 'NO'` | Use as seed: `castIntTo(a.value)` |
| `CONSTRAINT_TYPE = 'UNIQUE'` | Derive from PK seed to guarantee uniqueness |
| Regular index | Use `buildCastRand` range |
| FK | Parent table must have rows first |
| `IS_NULLABLE = 'NO'` and `COLUMN_DEF = 'Null'` and not index | **Must generate** — INSERT fails otherwise |
| All columns omitted (non-Oracle) | Use `INSERT INTO table DEFAULT VALUES` |

**Fallback — all columns DB-managed:**
| Engine | Syntax | Works? |
|--------|--------|--------|
| PostgreSQL | `INSERT INTO t DEFAULT VALUES` | ✓ tested (16.13) |
| SQL Server | `INSERT INTO t DEFAULT VALUES` | ✓ tested (Azure SQL 12.0) |
| MySQL | `INSERT INTO t DEFAULT VALUES` | **FAILS** — use `INSERT INTO t VALUES ()` |
| Oracle | Must specify at least one column | raises if empty |

**The actual pandas query from `db_info.py::set_regular_columns`:**
```python
null_cols = df.query(
    "TABLE_NAME.str.match('^{table}$', case=case_sensitive) "
    "and COLUMN_DEF == 'Null' "
    "and IS_AUTOINCREMENT != 'YES' "
    "and IS_GENERATEDCOLUMN != 'YES' "
    "and CATSCHTABCOL not in @index_cols"
    "and IS_NULLABLE == 'YES' "
    "and not(USED_IN_PKUQ > 0)"
)
# SQL Server adds: "and TYPE_NAME != 'timestamp'"
```

**Python skip helper:**
```python
def should_skip_column(col, db_type):
    if col.IS_AUTOINCREMENT == 'YES':   return True
    if col.IS_GENERATEDCOLUMN == 'YES': return True
    if col.COLUMN_DEF != 'Null':        return True
    if db_type == DbType.SQLSERVER and col.TYPE_NAME == 'timestamp': return True
    return False
```
