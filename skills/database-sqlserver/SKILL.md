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

A login must be created first in `master`, then a corresponding user must be created in each database it needs to access, linked with `FOR LOGIN`:

```sql
-- Step 1: create server-level login (in master)
USE master;
CREATE LOGIN [lfc_user] WITH PASSWORD = N'<password>';

-- Step 2: create database-level user linked to the login (in each DB)
USE [<catalog>];
CREATE USER [lfc_user] FOR LOGIN [lfc_user] WITH DEFAULT_SCHEMA = dbo;
```

## SA login: system administrator

`SA` is the built-in SQL Server admin login with full access to all databases.
- Password set during `mssql-conf setup` (Linux) or installation wizard (Windows).
- Connect: `sqlcmd -S "host,port" -d master -U sa -P "<password>" -C`
- On VM-hosted: `mssql-conf -n setup` with `MSSQL_SA_PASSWORD` env var sets SA password.

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

For the full login → user → roles → verify → create tables sequence, and multiple routing users: see [lfc-user-setup.md](lfc-user-setup.md).

## CT vs CDC: when to use each

| Mode | Mechanism | Table requirement | Overhead | Use for |
|------|-----------|------------------|----------|---------|
| `CT` | Tracks changed row PKs | Table must have PK | Low | PK-keyed tables (`intpk`) |
| `CDC` | Captures full row before/after | Any table | Higher | Tables without PK (`dtix`) |
| `BOTH` | Both CT and CDC | — | Both | Full demo / production |

Set `CDC_CT_MODE=CT|CDC|BOTH|NONE` in LFC setup scripts to control what is enabled.

## Change Tracking (CT) — quick reference

Database-level CT requires `EXEC()` with dynamic SQL because `ALTER DATABASE` doesn't accept a variable. Table-level CT must include `TRACK_COLUMNS_UPDATED = ON`.

For full DDL, verify queries, and idempotent guards: see [ct-cdc-setup.md](ct-cdc-setup.md).

```sql
-- Database CT — idempotent
IF NOT EXISTS (SELECT * FROM sys.change_tracking_databases WHERE database_id=DB_ID())
  EXEC ('ALTER DATABASE [<catalog>] SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON)');
go
-- Table CT
ALTER TABLE [dbo].[intpk] ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
```

## CDC — quick reference

CDC requires `autocommit=True`. Use the portable multi-platform ladder (one script works across Azure SQL, GCP Cloud SQL, AWS RDS). CDC auto-falls back to CT on Express/Web editions — always verify via `sys.databases.is_cdc_enabled`, not the stored proc's exit code.

For the full enable/disable ladders, `SET NOCOUNT ON` requirement, table-level CDC, and auto-fallback code: see [ct-cdc-setup.md](ct-cdc-setup.md).

## CDC availability: Express/Web editions

SQL Server Express and Web editions do not support CDC. Verify with:

```sql
SELECT is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
-- 1 = enabled, 0 = unavailable on this edition
```

If `is_cdc_enabled = 0`, fall back to CT mode. CT covers tables with PKs; CDC is only needed for non-PK tables.

## Schema evolution DDL objects

LFC requires `ddl_support_objects.sql` (CDC/BOTH modes) or `ddl_support_objects_ct_only.sql` (CT-only mode). Download from Databricks docs or use a local copy; inject `@replicationUser` and `@mode` via `sed` before piping to `sqlcmd`.

For the full download + sed + sqlcmd command and the utility script: see [schema-evolution.md](schema-evolution.md).

## Identifier quoting: brackets for hyphens

SQL Server uses `[square brackets]` for identifiers with hyphens. Critical inside dynamic SQL:

```sql
-- BAD — "Incorrect syntax near '-'"
EXEC('ALTER DATABASE robert-lee-db SET CHANGE_TRACKING = ON ...');

-- GOOD — brackets inside the dynamic string
EXEC('ALTER DATABASE [robert-lee-db] SET CHANGE_TRACKING = ON ...');
```

Use `robert_lee_db` (underscores) to avoid quoting in T-SQL, dynamic SQL, and `sqlcmd -d` arguments.

## sqlcmd conventions

```bash
sqlcmd -S "host,1433"               # comma-separated port (not colon)
sqlcmd -C                           # trust self-signed certificate
sqlcmd -l 30                        # login timeout (seconds); use 180 for Azure SQL Serverless
sqlcmd -Q "SELECT 1"                # inline query
sqlcmd -i script.sql                # run a T-SQL file
# go is the T-SQL batch separator; required between batches in heredocs
```

Full example:
```bash
sqlcmd -S "myserver.database.windows.net,1433" \
  -d mydb -U lfc_user -P "$LFC_PASSWORD" \
  -C -l 30 -Q "SELECT @@VERSION"
```

## trustServerCertificate

SQL Server uses TLS for all connections. Without a CA-signed certificate (VMs, developer instances), connections fail unless you bypass:

```bash
sqlcmd -C                                   # sqlcmd flag
{"trustServerCertificate": "true"}         # UC connection options
;TrustServerCertificate=Yes                # JDBC / Python connection string
```

Required for Azure VM SQL Server (self-signed cert from mssql-conf) and any SQL Server without a CA-signed TLS certificate.

## Testing a SQL Server connection

```bash
# Basic connectivity
sqlcmd -S "${HOST},${PORT}" -d master -U sa -P "$SA_PASSWORD" -C -l 10 -Q "SELECT 1"

# Check CT is enabled
sqlcmd -S "${HOST},${PORT}" -d "$CATALOG" -U sa -P "$SA_PASSWORD" -C \
  -Q "SELECT db_name(), is_change_tracking_on FROM sys.change_tracking_databases WHERE database_id = DB_ID()"

# Check if port is open
curl -v --connect-timeout 5 telnet://"${HOST}:${PORT}"
```
