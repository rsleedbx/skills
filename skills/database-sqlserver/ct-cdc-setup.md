# Change Tracking and CDC Setup

## Change Tracking (CT)

CT is lightweight row-level change tracking. Does not require sysadmin — `db_owner` is sufficient.

### Database-level CT — requires dynamic SQL

`ALTER DATABASE` does not accept a variable for the DB name. Wrap in `EXEC()`:

```sql
-- Idempotent: only enable if not already enabled
IF EXISTS (SELECT * FROM sys.change_tracking_databases WHERE database_id=DB_ID())
    SELECT 'CT already enabled'
ELSE
  BEGIN
    SELECT 'CT enabled on database';
    EXEC ('ALTER DATABASE [<catalog>] SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON)');
  END
go
```

> **Always bracket the database name inside `EXEC()`** — bare hyphens cause `Incorrect syntax near '-'`. See the identifier quoting section in SKILL.md.

Verify:
```sql
SELECT * FROM sys.change_tracking_databases WHERE database_id = DB_ID();
-- Non-empty = CT enabled
```

### Table-level CT

```sql
ALTER TABLE [<schema>].[intpk]
  ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
```

`TRACK_COLUMNS_UPDATED = ON` — required for LFC to identify which columns changed. Always include.

Verify:
```sql
SELECT DB_NAME() AS TABLE_CAT, SCHEMA_NAME(t.schema_id) AS TABLE_SCHEM, t.name AS TABLE_NAME
FROM sys.change_tracking_tables ctt
LEFT JOIN sys.tables t ON ctt.object_id = t.object_id
WHERE t.schema_id = SCHEMA_ID('<schema>');
-- Non-empty = tables are CT-enabled
```

---

## CDC (Change Data Capture)

CDC captures full before/after row images. Requires `sysadmin` at database-level (or equivalent platform role).

### CDC requires `autocommit=True` — cannot run inside a transaction

`ALTER DATABASE SET CHANGE_TRACKING`, `EXEC sys.sp_cdc_enable_db`, and `EXEC sys.sp_cdc_enable_table` **cannot run inside a transaction**. If called with autocommit=False (the default for most DB drivers), commands fail silently or with misleading errors.

```python
# Python pymssql / pyodbc
conn.autocommit = True
cursor.execute("EXEC sys.sp_cdc_enable_db")
conn.autocommit = False  # restore
```

```bash
# sqlcmd — no explicit autocommit needed (each batch runs in its own implicit transaction)
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C -Q "EXEC sys.sp_cdc_enable_db"
```

### Database-level CDC — portable multi-platform ladder

Run all three procedures guarded by `is_cdc_enabled`. Each tries only if the previous didn't succeed — works across Azure SQL, GCP Cloud SQL, and AWS RDS without conditionals:

```sql
SET NOCOUNT ON
go
-- Report current state
IF EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1)
    SELECT 'CDC already enabled'
ELSE
    SELECT 'CDC enabled on database'
go
-- Azure SQL / on-premise SQL Server
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1)
  EXEC sys.sp_cdc_enable_db
go
-- GCP Cloud SQL SQL Server
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1)
  EXEC msdb.dbo.gcloudsql_cdc_enable_db '<catalog>'
go
-- AWS RDS SQL Server
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1)
  EXEC msdb.dbo.rds_cdc_enable_db '<catalog>'
go
```

> **`SET NOCOUNT ON` before CDC stored procs** — required to fix `Invalid cursor state, SQL state 24000 in SQLExecDirect` caused by row-count messages interfering with cursor state.

Verify:
```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1;
-- Non-empty = success
```

### Database-level CDC — portable disable ladder

```sql
-- First disable table-level CDC
EXEC sys.sp_cdc_disable_table
  @source_schema = N'<schema>', @source_name = N'<table>', @capture_instance = N'all'
go
-- Azure SQL / on-premise
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=0)
  EXEC sys.sp_cdc_disable_db
go
-- GCP Cloud SQL
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=0)
  EXEC msdb.dbo.gcloudsql_cdc_disable_db '<catalog>'
go
-- AWS RDS
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=0)
  EXEC msdb.dbo.rds_cdc_disable_db '<catalog>'
go
```

### Table-level CDC

```sql
-- Azure SQL / on-premise
EXEC sys.sp_cdc_enable_table
  @source_schema     = N'dbo',
  @source_name       = N'dtix',
  @role_name         = NULL,
  @supports_net_changes = 0;   -- 0 = don't create net changes function; compatible with all editions
```

`@supports_net_changes = 1` requires Enterprise/Developer edition. Use `0` for maximum compatibility.

Idempotent guard:
```sql
IF EXISTS (SELECT * FROM sys.schemas WHERE name = 'cdc')
BEGIN
  IF NOT EXISTS (
    SELECT * FROM cdc.change_tables
    WHERE source_object_id = OBJECT_ID(N'dbo.dtix')
  )
    EXEC sys.sp_cdc_enable_table @source_schema = N'dbo', @source_name = N'dtix',
      @role_name = NULL, @supports_net_changes = 0;
END
```

```sql
-- AWS RDS
EXEC msdb.dbo.rds_cdc_enable_table 'dbo', 'dtix';
```

Check:
```sql
SELECT * FROM sys.dm_cdc_log_scan_sessions;
SELECT * FROM cdc.change_tables;
```

### CDC auto-fallback to CT

If CDC cannot be enabled, verify via `sys.databases` rather than trusting the stored proc's exit code:

```sql
SELECT is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
-- Returns 1 if CDC is actually enabled, 0 if not
```

If `is_cdc_enabled = 0` after running `sp_cdc_enable_db`, CDC is unavailable (Express/Web edition, or insufficient permissions). This is not an error — auto-fall back to CT:

```bash
if [[ ! -s /tmp/sqlcmd_stdout.$$ ]]; then
    echo "ERROR: CDC COULD NOT BE ENABLED. CHANGING TO CT ONLY MODE"
    CDC_CT_MODE=CT
fi
```

CT covers tables with primary keys (`intpk`). CDC is only required for non-PK tables (`dtix`) on supported editions.
