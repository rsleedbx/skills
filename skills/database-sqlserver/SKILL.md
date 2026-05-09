---
name: database-sqlserver
description: SQL Server 2019/2022 and Azure SQL — logins vs users (master vs database-level), SA admin account, required LFC roles (db_owner + db_ddladmin), Change Tracking (CT) setup, CDC setup with platform variants (Azure/AWS RDS/GCP Cloud SQL), cloud-specific CDC entrypoints (sp_cdc vs rds_cdc vs gcloudsql_cdc), AWS edition requirement (sqlserver-se for CDC), AWS option groups vs parameter groups, schema evolution DDL objects, identifier quoting with brackets, sqlcmd conventions (comma port, go separator, -C flag), trustServerCertificate, CT vs CDC mode for LFC. Use when creating SQL Server logins/users, enabling CT or CDC across platforms, writing sqlcmd scripts, understanding the login-user architecture, choosing CT vs CDC, or debugging privilege errors.
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

## Managed-platform login/user gotchas (discovered in mist testing)

Four operations that work on self-hosted SQL Server but are restricted or unreliable on managed cloud platforms:

### 1. `CREATE USER` in master — GCP Cloud SQL (and some Azure tiers)

`sqladmin` (GCP) and some restricted Azure admin accounts lack permission to create users in `master`. The step is **non-critical** — a login in `master` (for authentication) plus a user in the catalog (for authorization) is sufficient for LFC.

```python
try:
    exec_sql("IF NOT EXISTS (...) CREATE USER [{u}] FOR LOGIN [{u}] ...")
except Exception as e:
    print(f"[warn] skipped USER in master (not critical): {e}")
```

### 2. `ALTER LOGIN` — Azure SQL Database

`ALTER LOGIN [{user}] WITH PASSWORD = ...` can fail when the admin account doesn't hold `ALTER ANY LOGIN`. Non-critical because `CREATE LOGIN` already set the correct password. Run CREATE and ALTER as separate batches so a failure on ALTER doesn't block the CREATE:

```python
exec_sql("IF NOT EXISTS (...) CREATE LOGIN [{u}] WITH PASSWORD = N'{pw}';")
try:
    exec_sql(f"ALTER LOGIN [{u}] WITH PASSWORD = N'{pw}';")
except Exception as e:
    print(f"[warn] ALTER LOGIN skipped (not critical): {e}")
```

### 3. GCP user propagation delay (~30 s)

When the `sqladmin` user is created via the **GCP Cloud SQL Users API** (not T-SQL), there is a ~30 s propagation delay before the user can authenticate. LFC users created directly via `CREATE LOGIN` (T-SQL) are not affected — they are available immediately.

### 4. Username must start with a letter

SQL Server login and user names must begin with an alphabetic character. Names starting with digits or symbols produce a syntax error at `CREATE LOGIN`. Validate before running setup.

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

## Cloud-specific CDC entrypoints

| Platform | CDC enable stored proc | Catalog arg? |
|----------|----------------------|--------------|
| On-premise / VM | `EXEC sys.sp_cdc_enable_db` | No |
| Azure SQL | `EXEC sys.sp_cdc_enable_db` | No |
| AWS RDS | `EXEC msdb.dbo.rds_cdc_enable_db '<catalog>'` | **Yes** |
| GCP Cloud SQL | `EXEC msdb.dbo.gcloudsql_cdc_enable_db '<catalog>'` | **Yes** |

AWS and GCP wrappers require the database name as a positional argument. Azure's proc infers the current database. Omitting the catalog arg on AWS/GCP causes a silent no-op or error.

Pass `platform="aws"`, `platform="gcp"`, or `platform="azure"` to `sqlserver_configure()` so it uses the right entrypoint.

## AWS RDS SQL Server: edition and option groups

**Edition required for CDC:** `sqlserver-se` (Standard Edition). Express and Web editions do not support CDC. The forgedb AWS module always uses `sqlserver-se`.

**Option groups (not parameter groups):** SQL Server on RDS uses option groups to enable features like SQL Server Audit and native backup/restore. Parameter groups exist for RDS SQL Server but cover very few tunables — most SQL Server configuration is done via T-SQL, not parameter groups.

```python
# ensure_option_group pattern (AWS SQL Server)
rds.create_option_group(
    OptionGroupName=name,
    EngineName="sqlserver-se",
    MajorEngineVersion="15.00",   # from rds_value prefix
    OptionGroupDescription="forgedb LFC"
)
```

**Engine version resolution:** `describe_db_engine_versions(Engine="sqlserver-se", MajorEngineVersion="15.00")` → pick the latest patch string (e.g. `15.00.4345.5`). The `rds_value` in versions.py is the `MajorEngineVersion` prefix (`"15.00"` for 2019, `"16.00"` for 2022), not a full version string.

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
