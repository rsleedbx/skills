---
name: databricks-lakeflow-connect
description: Databricks LakeFlow Connect (LFC) database configuration conventions across clouds — CDC_CT_MODE semantics, intpk/dtix table schema and purpose, required user grants per engine (MySQL/PostgreSQL/SQL Server), cross-cloud CDC entrypoints (sp_cdc vs rds_cdc vs gcloudsql_cdc), PostgreSQL replication slot cleanup, Databricks secret V2 format, UC connection JSON structure, gateway and ingestion pipeline naming. Use when configuring databases for LFC ingestion, setting up users/tables/CDC/CT, building UC connections, writing ingestion pipeline definitions, or understanding the reference table/schema/user naming conventions.
---

# Databricks LakeFlow Connect — Database Configuration

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

The LFC user needs binlog CDC grants (`REPLICATION CLIENT`, `REPLICATION SLAVE`, DDL) and specific server parameters (`binlog_row_image=full`, `binlog_format=row`, `require_secure_transport=OFF`, etc.). Full SQL and parameter table: see `database-mysql` skill.

`endless_dml_loop` stored procedure is created in the schema for continuous load generation.

## PostgreSQL — required grants and replica identity

Full grant SQL, idempotent user creation, and platform variants (`ALTER ROLE WITH REPLICATION` vs `GRANT rds_replication`): see `database-postgres` skill.

Required server parameters: `wal_level=logical`, `require_secure_transport=off`.

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

**Table ownership:** Tables must be owned by `lfc_user` for `ALTER TABLE ... REPLICA IDENTITY` to succeed. Create tables as `lfc_user` (not DBA), or transfer ownership with `ALTER TABLE ... OWNER TO lfc_user`.

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

Full T-SQL for database-level CT, database-level CDC (multi-platform ladder: `sp_cdc_enable_db` / `gcloudsql_cdc_enable_db` / `rds_cdc_enable_db`), table-level CT (`TRACK_COLUMNS_UPDATED = ON` required), and table-level CDC: see `database-sqlserver` skill.

**LFC-specific flags:**
- `CDC_CT_MODE=BOTH|CT` → enable CT on `intpk`; disable CDC on `dtix` (set `REPLICA IDENTITY NOTHING`)
- `CDC_CT_MODE=BOTH|CDC` → enable CDC on database and `dtix`
- **Auto-fallback:** If CDC cannot be enabled (Express/Web edition), fall back to `CDC_CT_MODE=CT`

**Schema evolution objects:**
- Use `ddl_support_objects.sql` for `CDC_CT_MODE=BOTH|CDC`
- Use `ddl_support_objects_ct_only.sql` for `CDC_CT_MODE=CT`
- Inject `@replicationUser` and `@mode` via `sed` before running with `sqlcmd`.

## SQL Server — LFC user grants

`db_owner` + `db_ddladmin` roles on the application database. Full SQL: see `database-sqlserver` skill.

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

## UC connection options — verify before assuming

**Always probe the API to discover the current supported options list.** The options whitelist is enforced server-side and changes across Databricks versions. Do not assume an option is available; do not assume it is absent.

### How to probe supported options for a connection type

Send an intentionally unknown option. The error response lists every supported option:

```bash
databricks -p $PROFILE api post "/api/2.1/unity-catalog/connections" \
  --json '{
    "name": "_probe_options",
    "connection_type": "MYSQL",
    "options": {"host":"x","port":"3306","user":"x","password":"x","_probe_":"true"}
  }' 2>&1
# Error: CONNECTION/CONNECTION_MYSQL does not support the following option(s): _probe_.
# Supported options: userProvidedServerCertificate,host,port,trustServerCertificate,user,password.
```

If the probe accidentally creates a connection (a non-conflicting valid payload), delete it:
```bash
databricks -p $PROFILE api delete "/api/2.1/unity-catalog/connections/_probe_options"
```

Replace `"MYSQL"` with `"POSTGRESQL"` or `"SQLSERVER"` to discover options for those types.

### Verified options — last checked Apr 2026

> **WARNING — ASSUMED CURRENT:** The table below reflects a live API probe on Apr 30 2026.
> Options lists change as Databricks adds features. **Re-run the probe above** before relying on
> this table, especially before concluding that a desired option is unsupported.

| connection_type | Supported options | Notes |
|-----------------|-------------------|-------|
| `MYSQL` | `host`, `port`, `user`, `password`, `trustServerCertificate`, `userProvidedServerCertificate` | No JDBC pass-through. `useCompression`, `sslMode`, `connectTimeout` are **not** accepted. |
| `POSTGRESQL` | probe to verify | — |
| `SQLSERVER` | probe to verify | `trustServerCertificate` confirmed accepted |

### Consequences of the MYSQL options restriction

> **ASSUMED TRUE (verified Apr 2026):** LFC connects to MySQL without protocol-level compression
> because `useCompression` is not in the supported options. If Databricks adds compression support
> in future, re-probe and update the byte measurement methodology accordingly.

Because compression is unavailable:
- `Bytes_sent` in `performance_schema.status_by_account` equals actual pre-TLS wire bytes — no compression layer between MySQL and the JDBC client.
- `Bytes_sent` is therefore a direct measure of JDBC read volume and is comparable across runs without a decompression factor.
- TLS (if negotiated) encrypts bytes *after* MySQL counts them — `Bytes_sent` is always pre-TLS. The counter is accurate regardless of `trustServerCertificate` setting.

PATCH operations (update) must omit `connection_type`:
```bash
_patch_json=$(echo "$_create_json" | jq 'del(.connection_type)')
```

## Dual UC connections — PNG vs public bypass

For every database, create **two** connections to enable side-by-side PNG vs non-PNG testing:

| Connection suffix | Host | Routes via PNG? |
|------------------|------|-----------------|
| `<prefix>-<db>` | private FQDN / internal DNS | Yes (NAT64 intercepts private IP) |
| `<prefix>-<db>-public` | public IP **or** public FQDN | No (resolves to public IP, bypasses NAT64) |

**IP vs FQDN for the public connection:** PNG's NAT64 intercepts hostnames that resolve to **private IP addresses**. A public FQDN that resolves to a public IP is **not intercepted** and works for bypass. A raw IP (e.g. `20.97.1.2`) also works. Either form is acceptable for the `-public` connection. The only requirement is that the address does not resolve to a private RFC1918 address.

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
