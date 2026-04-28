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

## Create as DBA, do everything else as LFC user

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

```sql
-- Database-level CT
ALTER DATABASE [<catalog>]
  SET CHANGE_TRACKING = ON
  (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON);

-- Table-level CT
ALTER TABLE [dbo].[intpk]
  ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
```

Check:
```sql
SELECT * FROM sys.change_tracking_databases WHERE database_id = DB_ID();
SELECT * FROM sys.change_tracking_tables WHERE object_id = OBJECT_ID('dbo.intpk');
```

## CDC (Change Data Capture)

CDC captures before/after row images. Requires `sysadmin` at database-level.

### Database-level CDC — platform variants

```sql
-- Azure SQL / on-premise SQL Server
EXEC sys.sp_cdc_enable_db;

-- AWS RDS
EXEC msdb.dbo.rds_cdc_enable_db '<catalog>';

-- GCP Cloud SQL
EXEC msdb.dbo.gcloudsql_cdc_enable_db '<catalog>';
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

LFC uses `ddl_support_objects.sql` to capture DDL changes (ALTER TABLE). Inject via `sed` before running with `sqlcmd`:

```bash
sed "s/@replicationUser/${LFC_USER}/g; s/@mode/CT/g" ddl_support_objects.sql \
  | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```

`utility_script.sql` installs companion stored procedures for table discovery and CT/CDC status.

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
