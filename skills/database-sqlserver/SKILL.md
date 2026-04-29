---
name: database-sqlserver
description: SQL Server 2019/2022 and Azure SQL — logins vs users (master vs database-level), SA admin account, required LFC roles (db_owner + db_ddladmin), Change Tracking (CT) setup, CDC setup with platform variants (Azure/AWS RDS/GCP Cloud SQL), schema evolution DDL objects, identifier quoting with brackets, sqlcmd conventions (comma port, go separator, -C flag), trustServerCertificate, CT vs CDC mode for LFC. Use when creating SQL Server logins/users, enabling CT or CDC, writing sqlcmd scripts, understanding the login-user architecture, choosing CT vs CDC, or debugging privilege errors.
---

# SQL Server — Database Reference

## Logins vs Users architecture

SQL Server separates server-level identity from database-level identity:

| Object | Level | Location | Purpose |
|--------|-------|----------|---------|
| `LOGIN` | Server | `master` database | Authentication identity |
| `USER` | Database | Each application database | Authorization identity |

A login must be created first in `master`, then a corresponding user must be created in each database it needs to access. They must be linked with `FOR LOGIN`:

```sql
-- Step 1: create server-level login (in master)
USE master;
CREATE LOGIN [lfc_user] WITH PASSWORD = N'<password>';

-- Step 2: create database-level user linked to the login (in each DB)
USE [<catalog>];
CREATE USER [lfc_user] FOR LOGIN [lfc_user] WITH DEFAULT_SCHEMA = dbo;
```

## SA login: system administrator

`SA` (System Administrator) is the built-in SQL Server admin login:
- Full access to all databases and server-level operations
- Password is set during `mssql-conf setup` (Linux) or installation wizard (Windows)
- Connect: `sqlcmd -S "host,port" -d master -U sa -P "<password>" -C`

On VM-hosted SQL Server, `mssql-conf -n setup` with `MSSQL_SA_PASSWORD` env var sets the SA password.

## Required roles for LFC user

```sql
USE [<catalog>];
ALTER ROLE db_owner    ADD MEMBER [lfc_user];
ALTER ROLE db_ddladmin ADD MEMBER [lfc_user];
```

| Role | Why required |
|------|-------------|
| `db_owner` | Change Tracking, CDC, DML on all tables |
| `db_ddladmin` | Schema evolution DDL (ALTER TABLE, CREATE INDEX) |

## LakeFlow Connect DDL script URL

The schema evolution DDL script is versioned and hosted by Databricks:

```
https://docs.databricks.com/aws/en/assets/files/ddl_support_objects-06ebad393ea6bc7d853d5504dc6542de.sql
```

Download and inject with `sed` to replace placeholders before running via sqlcmd:

```bash
LAKEFLOW_DDL_SCRIPT_URL="https://docs.databricks.com/aws/en/assets/files/ddl_support_objects-06ebad393ea6bc7d853d5504dc6542de.sql"
curl -fsSL "$LAKEFLOW_DDL_SCRIPT_URL" \
  | sed "s/@replicationUser/${LFC_USER}/g; s/@mode/CT/g" \
  | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```

## CDC requires `autocommit=True` — cannot run in a transaction

`ALTER DATABASE SET CHANGE_TRACKING`, `EXEC sys.sp_cdc_enable_db`, and `EXEC sys.sp_cdc_enable_table` **cannot run inside a transaction**. If called via a database connection with autocommit=False (the default for most DB drivers), the commands fail silently or with:

```
The database principal owns a schema in the database, and cannot be dropped.
```

Always use `autocommit=True` for these operations:

```python
# Python pymssql / pyodbc — must set autocommit
conn.autocommit = True
cursor.execute("EXEC sys.sp_cdc_enable_db")
conn.autocommit = False  # restore
```

```bash
# sqlcmd — no explicit autocommit needed (runs each batch in its own implicit transaction)
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C -Q "EXEC sys.sp_cdc_enable_db"
```

## `sp_cdc_enable_table` — use `@supports_net_changes = 0`

```sql
EXEC sys.sp_cdc_enable_table
  @source_schema     = N'dbo',
  @source_name       = N'dtix',
  @role_name         = NULL,
  @supports_net_changes = 0;   -- 0 = don't create net changes function (simpler, compatible with all editions)
```

`@supports_net_changes = 1` creates an extra function that requires Enterprise/Developer editions. Set `0` for maximum compatibility.

Wrap with guard to only run when `cdc` schema exists:
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

## CDC availability: Express/Web editions — CT is sufficient fallback

SQL Server Express and Web editions do not support CDC. After attempting to enable CDC, **verify via `sys.databases` rather than trusting the stored proc's exit code**:

```sql
SELECT is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
-- Returns 1 if CDC is actually enabled, 0 if not
```

If `is_cdc_enabled = 0` after running `sp_cdc_enable_db`, CDC is unavailable on this edition. This is not an error — Change Tracking covers tables with primary keys. Return success with a note:

```
CDC not available on this SQL Server edition (Express/Web).
Change Tracking enabled — sufficient for tables with primary keys (intpk).
CDC required only for non-PK tables (dtix) on supported editions.
```



After LFC user creation and login verification, switch to the LFC user for all subsequent operations:

```bash
# Step 1: DBA creates login in master
sqlcmd -S "$HOST,$PORT" -d master -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = N'${LFC_USER}')
  CREATE LOGIN [${LFC_USER}] WITH PASSWORD = N'${LFC_PASSWORD}';
go
EOF

# Step 2: DBA creates user and grants roles in catalog
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = N'${LFC_USER}')
BEGIN
  CREATE USER [${LFC_USER}] FOR LOGIN [${LFC_USER}] WITH DEFAULT_SCHEMA = dbo;
  ALTER ROLE db_owner    ADD MEMBER [${LFC_USER}];
  ALTER ROLE db_ddladmin ADD MEMBER [${LFC_USER}];
END
go
EOF

# Step 3: verify lfc_user login
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U "$LFC_USER" -P "$LFC_PASSWORD" -C -l 10 -Q "SELECT 1"

# Step 4: create tables as lfc_user (db_owner — can enable CT/CDC on them)
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U "$LFC_USER" -P "$LFC_PASSWORD" -C <<'EOF'
IF OBJECT_ID(N'dbo.intpk', N'U') IS NULL
  CREATE TABLE [dbo].[intpk] (pk INT IDENTITY NOT NULL PRIMARY KEY, ...);
go
IF OBJECT_ID(N'dbo.dtix', N'U') IS NULL
  CREATE TABLE [dbo].[dtix] (created_at DATETIME, ...);
go
EOF
```

## Change Tracking (CT)

CT is lightweight row-level change tracking. Does not require sysadmin.

**Database-level CT** — requires dynamic SQL because `ALTER DATABASE` doesn't accept a variable for the DB name:

```sql
-- Idempotent: only enable if not already enabled
IF EXISTS (SELECT * FROM sys.change_tracking_databases WHERE database_id=DB_ID())
    SELECT 'CT already enabled'
ELSE
  BEGIN
    SELECT 'CT enabled on database';
    EXEC ('ALTER DATABASE <catalog> SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON)');
  END
go
```

Verify:
```sql
SELECT * FROM sys.change_tracking_databases WHERE database_id = DB_ID();
-- Non-empty output = CT enabled
```

**Table-level CT** — always include `TRACK_COLUMNS_UPDATED = ON`:

```sql
ALTER TABLE [<schema>].[intpk]
  ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
```

> `TRACK_COLUMNS_UPDATED = ON` — required for LFC to identify which columns changed. Always include this flag.

Verify CT on tables:
```sql
SELECT DB_NAME()                 AS TABLE_CAT,
       SCHEMA_NAME(t.schema_id)  AS TABLE_SCHEM,
       t.name                    AS TABLE_NAME
FROM sys.change_tracking_tables ctt
LEFT JOIN sys.tables t ON ctt.object_id = t.object_id
WHERE t.schema_id = SCHEMA_ID('<schema>')
-- Non-empty = tables are CT-enabled
```

## CDC (Change Data Capture)

CDC captures before/after row images. Requires `sysadmin` at database-level.

### Database-level CDC — portable multi-platform ladder

Run all three procedures in one script guarded by `is_cdc_enabled` checks. Each tries only if the previous didn't succeed — making the script work across all platforms without conditionals:

```sql
SET NOCOUNT ON
go
-- Check/report current state
IF EXISTS (SELECT name FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1)
    SELECT 'CDC already enabled'
ELSE
    SELECT 'CDC enabled on database'
go
-- Azure SQL / on-premise SQL Server (vm and azure sql)
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

> **`SET NOCOUNT ON` before CDC stored procs** — required to fix "Invalid cursor state, SQL state 24000 in SQLExecDirect" that occurs when row-count messages interfere with cursor state.

Verify after enable:
```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name=DB_NAME() AND is_cdc_enabled=1
-- Non-empty output = success
```

### CDC disable — portable multi-platform ladder

```sql
-- First disable table-level CDC
EXEC sys.sp_cdc_disable_table
  @source_schema = N'<schema>', @source_name = N'<table>', @capture_instance = N'all'
go
-- Then disable database-level
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

### CDC availability — auto-fallback to CT

If CDC cannot be enabled (Express/Web edition, or insufficient permissions), the lakeflow_connect reference implementation **auto-falls back to CT mode**:

```bash
# After running set_cdc_on_catalog:
if [[ ! -s /tmp/sqlcmd_stdout.$$ ]]; then
    echo "ERROR: CDC COULD NOT BE ENABLED. CHANGING TO CT ONLY MODE"
    CDC_CT_MODE=CT
fi
```

Verify CDC availability before deciding mode:
```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = DB_NAME() AND is_cdc_enabled = 1
-- Empty result = CDC not available on this edition/permissions
```

### Table-level CDC

```sql
-- Azure SQL / on-premise
EXEC sys.sp_cdc_enable_table
  @source_schema = N'dbo',
  @source_name   = N'dtix',
  @role_name     = NULL,
  @capture_instance = NULL;

-- AWS RDS
EXEC msdb.dbo.rds_cdc_enable_table 'dbo', 'dtix';
```

Check:
```sql
SELECT * FROM sys.dm_cdc_log_scan_sessions;
SELECT * FROM cdc.change_tables;
```

## CT vs CDC: when to use each

| Mode | Mechanism | Table requirement | Overhead | Use for |
|------|-----------|------------------|----------|---------|
| `CT` | Tracks changed row PKs | Table must have PK | Low | PK-keyed tables (`intpk`) |
| `CDC` | Captures full row before/after | Any table | Higher | Tables without PK (`dtix`) |
| `BOTH` | Both CT and CDC | — | Both | Full demo / production |

Set `CDC_CT_MODE=CT|CDC|BOTH|NONE` in LFC setup scripts to control what is enabled.

## Schema evolution DDL objects

LFC uses `ddl_support_objects.sql` (for CDC+BOTH modes) or `ddl_support_objects_ct_only.sql` (for CT-only mode). Choose the right file based on `CDC_CT_MODE`:

```bash
ddl_script_url="https://docs.databricks.com/aws/en/assets/files/ddl_support_objects-06ebad393ea6bc7d853d5504dc6542de.sql"

case "${CDC_CT_MODE}" in
  "BOTH"|"CDC") ddl_script=./ddl_support_objects.sql ;;
  "CT")         ddl_script=./ddl_support_objects_ct_only.sql ;;
  *)            echo "CDC_CT_MODE must be BOTH, CDC, or CT for schema evolution"; return 1 ;;
esac

# Use local file if present, else download
if [[ -f "$ddl_script" ]]; then
    cat "$ddl_script"
else
    wget -qO- "$ddl_script_url"
fi | sed -e "s/SET \@replicationUser = '';/SET \@replicationUser = '${LFC_USER}';/" \
        -e "s/\@mode = '.*';/\@mode = '$CDC_CT_MODE';/" \
  | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```

## Utility script

A companion script for table discovery and CT/CDC status:

```bash
utility_script_url="https://docs.databricks.com/aws/en/assets/files/utility_script-a4544ba646de3f6f3fd03eb3dcba563e.sql"

if [[ -f "$utility_script_path" ]]; then
    cat "$utility_script_path"
else
    wget -qO- "$utility_script_url"
fi | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```

## Identifier quoting: brackets for hyphens

SQL Server uses `[square brackets]` for identifiers with hyphens. Critical inside dynamic SQL strings where the name is embedded in a string literal:

```sql
-- BAD — "Incorrect syntax near '-'"
EXEC('ALTER DATABASE robert-lee-db SET CHANGE_TRACKING = ON ...');

-- GOOD — brackets inside the dynamic string
EXEC('ALTER DATABASE [robert-lee-db] SET CHANGE_TRACKING = ON ...');
```

This silent failure pattern: CT appears enabled (the `SELECT` succeeds) but the `ALTER DATABASE` inside `EXEC` was never applied, so table-level CT fails later.

Use `robert_lee_db` (underscores) to avoid all quoting in T-SQL, dynamic SQL, and `sqlcmd -d` arguments.

## sqlcmd conventions

```bash
# Port: comma-separated from host (not colon)
sqlcmd -S "host,1433"

# Trust self-signed certificate
sqlcmd -C

# Login timeout (seconds) — important for Azure SQL Serverless auto-resume
sqlcmd -l 30      # short for already-running DB
sqlcmd -l 180     # long for paused DB resuming

# Run a query inline
sqlcmd -Q "SELECT 1"

# Run a T-SQL file
sqlcmd -i script.sql

# T-SQL batch separator (required between GO-separated batches)
go

# Full example
sqlcmd -S "myserver.database.windows.net,1433" \
  -d mydb -U lfc_user -P "$LFC_PASSWORD" \
  -C -l 30 -Q "SELECT @@VERSION"
```

## trustServerCertificate

SQL Server uses TLS for all connections. Without a CA-signed certificate (VMs, developer instances), connections fail with certificate validation errors unless you bypass:

```bash
# sqlcmd
sqlcmd -C

# UC connection options (LFC / Databricks)
{"trustServerCertificate": "true"}

# JDBC / Python connection string
;TrustServerCertificate=Yes
```

Required for:
- Azure VM SQL Server (self-signed cert installed by mssql-conf)
- Azure SQL Serverless with default certificates (depends on client)
- Any SQL Server without a CA-signed TLS certificate

## Testing a SQL Server connection

```bash
sqlcmd -S "${HOST},${PORT}" -d master -U sa -P "$SA_PASSWORD" -C -l 10 -Q "SELECT 1"

# Check CT is enabled
sqlcmd -S "${HOST},${PORT}" -d "$CATALOG" -U sa -P "$SA_PASSWORD" -C \
  -Q "SELECT db_name(), is_change_tracking_on FROM sys.change_tracking_databases WHERE database_id = DB_ID()"

# Check if port is open
curl -v --connect-timeout 5 telnet://"${HOST}:${PORT}"
```
