---
name: lakeflow-connect
description: LakeFlow Connect (LFC) database configuration conventions across clouds — CDC_CT_MODE semantics, intpk/dtix table schema and purpose, required user grants per engine (MySQL/PostgreSQL/SQL Server), cross-cloud CDC entrypoints (sp_cdc vs rds_cdc vs gcloudsql_cdc), PostgreSQL replication slot cleanup, Databricks secret V2 format, UC connection JSON structure, gateway and ingestion pipeline naming. Use when configuring databases for LFC ingestion, setting up users/tables/CDC/CT, building UC connections, writing ingestion pipeline definitions, or understanding the reference table/schema/user naming conventions.
---

# LakeFlow Connect — Database Configuration

## CDC_CT_MODE

`CDC_CT_MODE` controls which replication mechanism and which demo tables are used:

| Mode | Description | Tables affected |
|------|-------------|-----------------|
| `BOTH` | CDC + CT; schema-wide ingestion | `intpk` (CT) + `dtix` (CDC) |
| `CT` | Change Tracking only | `intpk` only |
| `CDC` | Change Data Capture only | `dtix` only |
| `NONE` | No replication; read-only test | neither |

This single flag drives:
- **PostgreSQL**: `REPLICA IDENTITY` setting on each table (see below)
- **SQL Server**: which CT/CDC enablement steps run
- **GCP Cloud SQL**: instance edition (Express/Web for CT-only, Enterprise/Standard for CDC)
- **Ingestion pipeline**: `source_catalog`, table list, and `source_type` in the pipeline JSON

## Demo tables — intpk and dtix

| Table | PK | Purpose | Tracking type |
|-------|----|---------|--------------|
| `intpk` | Auto-increment integer | Change Tracking / row-level changes | CT-style |
| `dtix` | No explicit PK (datetime column) | CDC-style append / event stream | CDC-style |

Both tables are created in the `lfc_test_db` (or caller-specified `DB_SCHEMA`) schema. These are the standard test tables used across all clouds and engines in the reference implementation.

## MySQL — required user grants

The LFC user needs these grants for binlog CDC:

```sql
GRANT ALTER, CREATE, DROP, SELECT, INSERT, DELETE, UPDATE ON `<schema>`.* TO '<lfc_user>'@'%';
GRANT REPLICATION CLIENT ON *.* TO '<lfc_user>'@'%';
GRANT REPLICATION SLAVE  ON *.* TO '<lfc_user>'@'%';
```

Required server parameters:
| Parameter | Value |
|-----------|-------|
| `binlog_row_image` | `full` |
| `binlog_format` | `row` |
| `require_secure_transport` | `OFF` |
| `sql_generate_invisible_primary_key` | `OFF` |
| `binlog_expire_logs_seconds` | `≥ 604800` (7 days) |

`endless_dml_loop` stored procedure is created in the schema for continuous load generation.

## PostgreSQL — required grants and replica identity

```sql
-- User creation and grants
CREATE USER <lfc_user> WITH PASSWORD '<password>';
GRANT CONNECT ON DATABASE <catalog> TO <lfc_user>;
GRANT USAGE, CREATE ON SCHEMA public TO <lfc_user>;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA public TO <lfc_user>;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON TABLES TO <lfc_user>;
-- Replication role (Azure Flexible Server)
ALTER ROLE <lfc_user> WITH REPLICATION;
-- AWS RDS: GRANT rds_replication TO <lfc_user>;
```

Required server parameters:
| Parameter | Value |
|-----------|-------|
| `wal_level` | `logical` |
| `require_secure_transport` | `off` |

**Replica identity** drives what is included in CDC events:

| Table | Mode | CDC_CT_MODE |
|-------|------|-------------|
| `intpk` | `DEFAULT` (uses PK) | `BOTH` or `CT` |
| `dtix` | `FULL` (all columns) | `BOTH` or `CDC` |
| `dtix` | `NOTHING` | `CT` only (no CDC on dtix) |

```sql
ALTER TABLE intpk REPLICA IDENTITY DEFAULT;
ALTER TABLE dtix  REPLICA IDENTITY FULL;   -- or NOTHING for CT-only mode
```

**Table ownership:** Tables must be owned by `lfc_user` for `ALTER TABLE ... REPLICA IDENTITY` to succeed. Create tables as `lfc_user` (not DBA), or transfer ownership:
```sql
DO $$ DECLARE r RECORD;
BEGIN FOR r IN SELECT tablename FROM pg_tables WHERE schemaname='public'
  LOOP EXECUTE 'ALTER TABLE public.' || r.tablename || ' OWNER TO lfc_user'; END LOOP;
END $$;
```

## PostgreSQL — replication slot cleanup

LFC creates replication slots with the naming pattern `dbx_%_<GATEWAY_PIPELINE_ID>`. Clean them up after pipeline teardown:

```sql
SELECT slot_name FROM pg_replication_slots
  WHERE slot_name LIKE 'dbx_%_<GATEWAY_PIPELINE_ID>';

SELECT pg_drop_replication_slot(slot_name)
  FROM pg_replication_slots
  WHERE slot_name LIKE 'dbx_%_<GATEWAY_PIPELINE_ID>';
```

Uncleared slots hold WAL and grow the disk indefinitely. Always drop slots when destroying a pipeline.

## SQL Server — CT and CDC setup

**Database-level:**
```sql
-- Change Tracking (CT)
ALTER DATABASE [<catalog>] SET CHANGE_TRACKING = ON
  (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON);

-- CDC (Azure SQL)
EXEC sys.sp_cdc_enable_db;

-- CDC (AWS RDS)
EXEC msdb.dbo.rds_cdc_enable_db '<catalog>';

-- CDC (GCP Cloud SQL)
EXEC msdb.dbo.gcloudsql_cdc_enable_db '<catalog>';
```

**Table-level:**
```sql
-- CT on intpk
ALTER TABLE [<schema>].[intpk] ENABLE CHANGE_TRACKING;

-- CDC on dtix (Azure SQL / on-premise)
EXEC sys.sp_cdc_enable_table
  @source_schema = '<schema>', @source_name = 'dtix',
  @role_name = NULL, @capture_instance = NULL;

-- CDC on dtix (AWS RDS)
EXEC msdb.dbo.rds_cdc_enable_table '<schema>', 'dtix';
```

**Schema evolution objects:** Inject `ddl_support_objects.sql` via `sed` to replace `@replicationUser` and `@mode` placeholders before executing with `sqlcmd`.

## SQL Server — LFC user grants

```sql
-- In master database
CREATE LOGIN [<lfc_user>] WITH PASSWORD = '<password>';

-- In application database
CREATE USER [<lfc_user>] FOR LOGIN [<lfc_user>];
ALTER ROLE db_owner ADD MEMBER [<lfc_user>];
ALTER ROLE db_ddladmin ADD MEMBER [<lfc_user>];
```

`db_owner` is required for CT and CDC operations. `db_ddladmin` supports schema evolution DDL.

## UC connection JSON structure

Canonical structure returned by `_build_uc_conn_json` in `uc-connection-helpers.sh`:

```json
{
  "name": "<connection-name>",
  "connection_type": "MYSQL|POSTGRESQL|SQLSERVER",
  "comment": "{\"secrets\":{\"scope\":\"<scope>\",\"key\":\"<key>\"}}",
  "options": {
    "host": "<host>",
    "port": "<port>",
    "user": "<lfc_user>",
    "password": "<lfc_password>"
  }
}
```

SQL Server adds `"trustServerCertificate": "true"` to `options` (self-signed cert on most VMs and Azure SQL).

PATCH operations (update) must omit `connection_type`:
```bash
_patch_json=$(echo "$_create_json" | jq 'del(.connection_type)')
```

## Dual UC connections — PNG vs public bypass

For every database, create **two** connections to enable side-by-side PNG vs non-PNG testing:

| Connection suffix | Host | Routes via PNG? |
|------------------|------|-----------------|
| `<prefix>-<db>` | private FQDN / internal DNS | Yes (NAT64 intercepts) |
| `<prefix>-<db>-public` | public IP or public FQDN | No (direct egress) |

The public connection uses the raw IP address (not FQDN) to bypass PNG's NAT64 interception — DNS resolution is what triggers PNG interception.

## Databricks secret V2 format

Canonical fields stored in the Databricks secret scope (JSON, base64-encoded):

| Field | Example | Notes |
|-------|---------|-------|
| `host_fqdn` | `robert-lee-mysql-vm.internal.cloudapp.net` | Primary hostname for LFC |
| `port` | `3306` | Database port |
| `user` | `lfc_user` | LFC app user |
| `password` | `<generated>` | LFC app password |
| `db_type` | `mysql` / `postgresql` / `sqlserver` | LFC engine type |
| `schema` | `lfc_test_db` / `public` / `dbo` | Default schema |
| `replication_mode` | `binlog` / `logicalreplication` / `ct` | Replication type |
| `dba__user` | `root` / `postgres` / `sa` | Admin user (double underscore prefix) |
| `dba__password` | `<generated>` | Admin password |
| `access_type` | `vm` / `flexible` / `azure_sql` | Where the DB is hosted |
| `vm_fqdn_public` | `robert-lee-mysql-vm.eastus2.cloudapp.azure.com` | VM only: public FQDN for local CLI |

## Naming conventions

| Object | Pattern | Example |
|--------|---------|---------|
| Database schema | Use underscores, not hyphens | `lfc_test_db` not `lfc-test-db` |
| LFC user | `lfc_user` | same across all engines |
| UC connection (PNG) | `{user_prefix}-{cloud}-{engine}` | `robert-lee-azure-mysql` |
| UC connection (public) | `{user_prefix}-{cloud}-{engine}-public` | `robert-lee-azure-mysql-public` |
| VM-hosted connection | `{user_prefix}-vm-{engine}` | `robert-lee-vm-mysql` |
| Secret scope key | `{cloud}-{engine}` or `vm-{engine}` | `azure-mysql`, `vm-postgres` |

Underscores in database names avoid quoting requirements in all three SQL dialects (MySQL backticks, PostgreSQL double-quotes, SQL Server square brackets).
