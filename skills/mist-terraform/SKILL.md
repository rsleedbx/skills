---
name: mist-terraform
description: Mist database provisioning Terraform patterns — flat per-cloud stacks (no shared modules), environment_variables output contract, locals.common_tags, random credential generation with Oracle-safe override_special, clean_instance_id naming, firewall CIDR variable names by cloud (additional_cidrs/allowed_cidrs/authorized_networks), terraform_common.sh orchestration functions, deploy_database lifecycle, Azure region fallback, GCP delete via gcloud CLI, infra-vs-config split (Terraform provisions, Python/bash configures). Use when writing or reading mist Terraform stacks, consuming environment_variables output, setting up firewall CIDRs, orchestrating multi-cloud deploys, or handling cloud-specific deletion.
---

# Mist Terraform Patterns

## Stack layout: flat per-cloud, no shared modules

Mist uses **standalone root modules** per engine+cloud combination. No shared HCL `module "..."` blocks:

```
mist/terraform_configs/
  oracle/aws-oracle/     oracle/oci-oracle/
  mysql/aws-mysql/       mysql/gcp-mysql/      mysql/azure-mysql/
  postgres/aws-postgres/ postgres/gcp-postgres/ postgres/azure-postgres/
  sqlserver/...
  shared/terraform_common.sh   ← bash orchestration, not HCL
```

Conventions shared across stacks by **naming convention**, not Terraform module references.

## environment_variables output: the stable API contract

Every stack must output `environment_variables` as a **map of uppercase string keys**:

```hcl
output "environment_variables" {
  description = "Environment variables for database connection"
  sensitive   = true
  value = {
    DB_HOST_FQDN    = aws_db_instance.oracle.address
    DB_PORT         = tostring(aws_db_instance.oracle.port)   # always string, never number
    DB_CATALOG      = aws_db_instance.oracle.db_name
    USER_USERNAME   = local.username
    USER_PASSWORD   = local.user_password
    DBA_USERNAME    = local.dba_username
    DBA_PASSWORD    = local.dba_password
    DB_TYPE         = "oracle"     # lowercase engine name
    CLOUD_PROVIDER  = "aws"        # aws, azure, gcp, oci
    CONNECTION_TYPE = "ORACLE"     # uppercase, maps to driver selection
    DB_VERSION      = aws_db_instance.oracle.engine_version
  }
}
```

**Consuming in bash:**
```bash
eval "$(terraform output -json environment_variables \
  | jq -r 'to_entries[] | "export \(.key)=\(.value | @sh)"')"
```

**Standard keys all stacks must provide:** `DB_HOST_FQDN`, `DB_PORT` (string), `DB_CATALOG`, `USER_USERNAME`, `USER_PASSWORD`, `DBA_USERNAME`, `DBA_PASSWORD`, `DB_TYPE`, `CLOUD_PROVIDER`, `CONNECTION_TYPE`, `CLOUD_CONSOLE_URL`, `CLI_COMMAND`.

Oracle-specific: `JDBC_URL`, `SQLPLUS_CONNECT`. OCI-specific: `OCI_COMPARTMENT_ID`, `OCI_REGION`, `SERVICE_CONSOLE_URL`.

### `environment_variables` is not uniform across stacks ⚠️

Several stacks **omit** keys that automation expects:

| Stack | Missing keys | Notes |
|-------|-------------|-------|
| Azure MySQL/Postgres | `CONNECTION_TYPE`, `SOURCE_TYPE` | Not set — callers must assume from `CLOUD_DB_TYPE` |
| AWS Aurora Postgres | `CONNECTION_TYPE`, `SOURCE_TYPE` | Outputs `AURORA_CLUSTER_ARN`, `AURORA_READER_ENDPOINT` instead |
| GCP MySQL | `CONNECTION_TYPE`, `SOURCE_TYPE` | |
| GCP SQL Server | `DB_HOST`, `CLOUD_DB_TYPE`, `CLOUD_DB_SUFFIX` | `DBA_USERNAME` is hardcoded to `"sqlserver"` (literal string, not the random username) |
| AWS Aurora vs RDS | `DB_SG_ID` (Aurora uses `DB_SECURITY_GROUP_ID`) | Different key name for same concept |
| Aurora `db_suffix` default | Still `"aws-pg"` in variables.tf | Same as non-Aurora — can confuse logs/automation |

When consuming `environment_variables` in bash automation, guard against missing keys:
```bash
CONNECTION_TYPE="${CONNECTION_TYPE:-${DB_TYPE^^}}"  # fallback if not set
```

> OCI ADB: `USER_USERNAME = DBA_USERNAME = "ADMIN"`. Create application users post-provision if needed.

## locals.common_tags

```hcl
locals {
  common_tags = merge(var.tags, {
    Name        = var.instance_id
    Owner       = var.owner
    Environment = var.environment
    Project     = "lfcddemo"
    Service     = "mist"
    ManagedBy   = "terraform"
    Database    = "oracle"        # lowercase engine name
    DateCreated = var.date_created
    RemoveAfter = var.remove_after
  })
}
```

## clean_instance_id: safe resource name

```hcl
locals {
  clean_instance_id = lower(replace(replace(var.instance_id, "_", "-"), ".", "-"))
  # "my_db.prod" → "my-db-prod"
}
```

Use `local.clean_instance_id` for cloud resource names; `var.instance_id` for tags/display names.

## Random credential generation

### Usernames: must start with a letter (Oracle requirement; good practice for all)

```hcl
locals {
  username     = "u${random_string.username_base.result}"    # regular user
  dba_username = "a${random_string.dba_base.result}"         # admin user
}
```

### Passwords: Oracle-safe `override_special`

```hcl
resource "random_password" "oracle_pass" {
  length           = 16
  special          = true
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
  override_special = "!#$%*()-_=+[]{}|:;<>,.?"  # excludes / " @ & space
}
```

MySQL/PostgreSQL/SQL Server are less restrictive — `override_special` optional.

### Conditional generation (bring-your-own credential)

```hcl
resource "random_password" "user_password" {
  count = var.user_password == null ? 1 : 0
  ...
}
locals {
  user_password = var.user_password != null ? var.user_password : random_password.user_password[0].result
}
```

## Oracle SID naming

```hcl
locals {
  db_name = "ORCL${upper(random_string.db_name.result)}"   # e.g. "ORCL4A2B"
  sid     = local.db_name
}
```

SID limit: 8 characters. `ORCL` + 4 random = exactly 8.

## Firewall CIDR variable names by cloud

| Cloud | Variable name | Terraform resource |
|-------|-------------|-------------------|
| AWS | `additional_cidrs` | `aws_security_group_rule` ingress |
| Azure | `allowed_cidrs` | NSG / Azure Firewall policy |
| GCP | `authorized_networks` | `google_sql_database_instance.settings.ip_configuration` |

`terraform_common.sh` auto-injects the caller's IP via `curl -s http://ipv4.icanhazip.com`. For split tunneling and full per-cloud firewall commands: see `firewall-ip-discovery` skill.

## GCP Cloud SQL tier and edition constraints

These are hard API rules — violations return 400 with no useful fallback.

| Constraint | Rule | Fix |
|---|---|---|
| **Binary logging requires backups** | MySQL: `binaryLogEnabled=true` is rejected unless `backupConfiguration.enabled=true`. Applies to all MySQL versions on standalone instances. | Force `enabled: true` when binary_log is needed; set `transactionLogRetentionDays: 1` + `backupRetentionSettings: {retentionUnit: COUNT, retainedBackups: 1}` for dev to minimise cost. |
| **Backup retention ≥ transaction log retention** | GCP enforces `backupRetentionDays >= transactionLogRetentionDays` (default 7). Setting `retainedBackups: 1` (COUNT) while `transactionLogRetentionDays` is 7 is rejected. | Set both to 1 for dev. |
| **ENTERPRISE_PLUS default** | Some GCP projects/orgs default new instances to `ENTERPRISE_PLUS` edition. `db-f1-micro` and `db-custom-*` are ENTERPRISE-only tiers and are rejected for ENTERPRISE_PLUS. | Always set `"edition": "ENTERPRISE"` explicitly in the settings body. |
| **SQL Server requires custom machine type** | SQL Server cannot use `db-f1-micro` (shared vCPU). Minimum is `db-custom-{cpu}-{ram_mb}`. The free-tier `db-f1-micro` shortcut must be skipped for SQL Server. | In tier resolution: only use `db-f1-micro` when `db_type != "sqlserver"`. Treat `has_free_tier("gcp", "sqlserver")` as False for sizing purposes. |
| **SQL Server minimum RAM** | SQL Server requires at least 3840 MiB (3.75 GB) total memory. `db-custom-1-2048` (2 GB) is rejected. | Clamp `ram_mb = max(ram_mb, 3840)` for SQL Server before building the tier string. |

**Summary of valid GCP Cloud SQL tier+edition combinations:**

| Tier | Engine | Edition |
|---|---|---|
| `db-f1-micro` | MySQL, PostgreSQL only | ENTERPRISE |
| `db-custom-{N}-{M}` | MySQL, PostgreSQL, SQL Server | ENTERPRISE |
| `db-perf-optimized-N-*` | MySQL, PostgreSQL, SQL Server | ENTERPRISE_PLUS |

## GCP Cloud SQL: mist defaults and gotchas (from production Terraform)

These are the concrete values used in the mist Terraform stacks (`mist/mist/terraform_configs/`).

### Per-engine defaults

| Setting | MySQL | PostgreSQL | SQL Server |
|---------|-------|-----------|------------|
| `db_tier` default | `db-custom-1-3840` | `db-standard-1` (mist generator uses `db-custom-*`) | `db-custom-2-7680` |
| `disk_type` | `PD_HDD` | `PD_SSD` | `PD_SSD` |
| `disk_size` | 10 GB | 10 GB | 20 GB |
| `edition` in Terraform | _(not set — API default)_ | `"ENTERPRISE"` (explicit) | _(not set — inferred from `databaseVersion`)_ |
| `database_version` default | `MYSQL_8_0` | `POSTGRES_15` | `SQLSERVER_2019_STANDARD` (mist generates `SQLSERVER_2022_STANDARD`) |
| `availability_type` | `ZONAL` | `ZONAL` | `ZONAL` |
| `disk_autoresize` | `false` | `false` | `false` |
| `deletion_protection` | `false` | `false` | `false` |

**Key cost notes:**
- MySQL uses `PD_HDD` (cheaper). PostgreSQL and SQL Server use `PD_SSD`.
- Mist SQL Server minimum is `db-custom-2-7680` (2 vCPU / 7.5 GB). The absolute GCP API minimum for SQL Server is `db-custom-1-3840` (1 vCPU / 3.75 GB).
- `disk_autoresize = false` prevents unbounded disk cost in dev.

### `sslMode`

All three mist GCP stacks explicitly set:
```hcl
ssl_mode = "ALLOW_UNENCRYPTED_AND_ENCRYPTED"
```
GCP's API default may differ based on org policy. Always set `sslMode` explicitly to avoid surprises:
- dev/staging: `ALLOW_UNENCRYPTED_AND_ENCRYPTED`
- prod: `ENCRYPTED_ONLY`

### Backup configuration pattern

Mist links backup to the `enable_replication` flag (not only to `backup_enabled`):

```
MySQL:       enabled = backup_enabled || enable_replication
             binary_log_enabled = enable_replication
PostgreSQL:  enabled = backup_enabled || enable_replication
             point_in_time_recovery_enabled = enable_replication || backup_enabled
SQL Server:  enabled = backup_enabled              (no enable_replication variable)
             point_in_time_recovery_enabled = backup_enabled
```

For retention: **7 days** when backups enabled, **1 day** when disabled (applies to both `transactionLogRetentionDays` and `retainedBackups`).

### PostgreSQL database flags (when `enable_replication=true`)

```hcl
database_flags { name = "cloudsql.logical_decoding"  value = "on" }
```
`max_replication_slots` and `max_wal_senders` are not set in mist Terraform — Cloud SQL defaults are sufficient.

### SQL Server admin user

SQL Server instances require `root_password` at the top level of the instance resource (not inside `settings`). The built-in admin login is **`sqlserver`** and uses that root password. Additional users are created separately via the Cloud SQL Users API. `DBA_USERNAME` in mist outputs is hardcoded to `"sqlserver"` (not the randomly generated `dba_username`).

### PostgreSQL `cloudsqlsuperuser` grant

On GCP, `ALTER ROLE {user} WITH REPLICATION` is sufficient for the LFC user to connect in replication mode. For replication slot management (create/drop), the LFC user additionally needs the `cloudsqlsuperuser` role:

```sql
GRANT cloudsqlsuperuser TO "lfc_user";
```

`cloudsqlsuperuser` is the GCP equivalent of PostgreSQL's superuser role within Cloud SQL's access model. The admin user created via the Cloud SQL API already has this role. `rds_replication` (AWS) is the equivalent pattern on RDS.

### GCP SQL Server CDC

Use `msdb.dbo.gcloudsql_cdc_enable_db` instead of `sys.sp_cdc_enable_db`:
```sql
EXEC msdb.dbo.gcloudsql_cdc_enable_db N'<catalog>'
EXEC msdb.dbo.gcloudsql_cdc_disable_db N'<catalog>'
```

### Firewall: `gcloud sql instances patch` + 5-minute timeout

Authorized network updates are async and can take up to 5 minutes. Mist's `firewall_manager.py` uses a 5-minute timeout when polling the patch operation.

## AWS RDS defaults (from mist production Terraform)

Sources: `mist/mist/terraform_configs/{mysql,postgres,sqlserver}/aws-{engine}/`

### Instance classes

| Engine | Default `instance_class` | Notes |
|--------|--------------------------|-------|
| MySQL | `db.t4g.medium` | ARM Graviton; not available in all regions |
| PostgreSQL | `db.t4g.medium` | Same |
| SQL Server | `db.t3.small` | Express editions only; SE/EE need `db.m5.large` or higher |

**Gotcha:** Mist defaults `engine = "sqlserver-se"` (Standard Edition) but `instance_class = "db.t3.small"`. SE on t3.small may fail in some regions — use `db.m5.large` for SE/EE or switch to `sqlserver-ex` for t3.small.

### Backup / deletion

| Setting | MySQL | PostgreSQL | SQL Server |
|---------|-------|-----------|------------|
| `backup_retention_period` | **hard-coded `1`** (required for binlog/CDC) | **hard-coded `1`** (required for WAL replication) | **variable default `0`** — not forced to 1 |
| `deletion_protection` | `false` | `false` | `false` |
| `skip_final_snapshot` | `true` | `true` | `true` |
| `storage_encrypted` | `false` | `false` | `false` |
| `multi_az` | `false` (variable) | `false` | `false` |
| `storage_type` | `gp2` | `gp2` | `gp2` |
| `allocated_storage` | 20 GB | 20 GB | 20 GB |
| `publicly_accessible` | `true` | `true` | `true` |

### Parameter groups

**MySQL** (`family = "mysql8.0"`):
- `binlog_format = ROW` (immediate)
- `binlog_row_image = FULL` (immediate)
- `require_secure_transport = 0` (immediate)

**PostgreSQL** (`family = "postgres15"`):
- `rds.logical_replication = 1` (**`pending-reboot`** — instance must restart)
- `max_replication_slots = 10` (pending-reboot)
- `max_wal_senders = 15` (pending-reboot)
- `max_worker_processes = 10` (pending-reboot)
- `rds.force_ssl = 0` (immediate)
- **No explicit `wal_level` parameter** — `rds.logical_replication=1` implies `wal_level=logical`.

**SQL Server:** empty `aws_db_option_group` shell; no option blocks (CDC is not enabled via Terraform — done in Python post-apply scripts).

### Subnet / networking

- Default subnet: public subnets only (`map-public-ip-on-launch = true` filter) — required when `publicly_accessible = true`.
- Security group: per-instance, ingress on `db_port` from `local.all_cidrs`; egress TCP 0→65535.
- **Gotcha:** Using private subnets with `publicly_accessible = true` (or vice versa) is a common 400-class failure on RDS.

### Engine versions and SQL Server editions

| Engine | Terraform default | Mist-generated |
|--------|------------------|---------------|
| MySQL | `8.0` | `8.0` |
| PostgreSQL | `15` | `15` |
| SQL Server | `15.00` + `sqlserver-se` | same |

SQL Server `license_model = "license-included"` (not BYOL).

### AWS SQL Server CDC

AWS uses `msdb.dbo.rds_cdc_enable_db` instead of `sys.sp_cdc_enable_db`. No Terraform option group setup required for CDC — done via Python post-apply scripts.

---

## Azure Flexible Server / Azure SQL defaults (from mist production Terraform)

Sources: `mist/mist/terraform_configs/{mysql,postgres,sqlserver}/azure-{engine}/`

**Scope:** No Azure SQL Managed Instance in mist. SQL Server = `azurerm_mssql_server` + `azurerm_mssql_database` (Azure SQL Database DTU/vCore).

### SKU / tier

| Service | Default `sku_name` | Tier |
|---------|-------------------|------|
| MySQL Flexible Server | `B_Standard_B1ms` | Burstable |
| PostgreSQL Flexible Server | `GP_Standard_D2s_v3` | General Purpose (2 vCPU / 8 GB) |
| Azure SQL Database | `Basic` | DTU model |

**Gotcha:** PostgreSQL default is General Purpose, not Burstable. GCP has a `db-f1-micro` equivalent; Azure PostgreSQL Burstable is cheaper but has feature restrictions for logical replication.

### Storage / backup

| | MySQL | PostgreSQL | Azure SQL |
|--|-------|-----------|-----------|
| Storage | `storage.size_gb` default 20 | `storage_mb` default 32768 (32 GB) | SKU-defined |
| Backup retention | 7 days | 7 days | SKU-defined |
| Geo-redundant backup | `false` | `false` | N/A |

**Gotcha:** PostgreSQL `storage_mb` is a **string** in `variables.tf` (`"32768"`), not a number. The Terraform resource expects a number — Terraform coerces it, but callers that read the variable as a string will get `"32768"`.

### Server parameters / flags

**MySQL Flexible Server:**
- `binlog_format` and `binlog_row_image` are **read-only** — always ROW/FULL by platform default. Do not try to set them.
- `binlog_expire_logs_seconds = "86400"` (1 day) — the only configurable binlog retention parameter.

**PostgreSQL Flexible Server:**
- `wal_level = logical` — required for logical decoding.
- `max_replication_slots = 10`, `max_wal_senders = 15`, `max_worker_processes = 10`
- Unlike RDS, there is no `rds.logical_replication` knob; `wal_level=logical` is the direct parameter.

**Azure SQL Database:**
- `minimum_tls_version = "1.2"` on the server.
- No Terraform-level CDC flags — CT/CDC enabled via post-apply SQL scripts.

### Private endpoint vs firewall

- **No private endpoint resources** in mist Terraform — access is via `azurerm_*_firewall_rule` with start/end IP pairs.
- Open-internet pattern: start_ip = `0.0.0.0`, end_ip = `255.255.255.255`.

### Azure SQL serverless / auto-pause

Mist uses the `Basic` DTU SKU (no serverless). Serverless path (`GP_S_*`) is available but requires:
- `sku_name` matching `^GP_S_` pattern
- `min_capacity = 0.5`
- `auto_pause_delay_in_minutes = 60`

With `Basic` sku, `auto_pause_delay_in_minutes = -1` (disabled).

### MySQL Flexible Server version gotcha

Mist default `version = "8.0.21"` (patch-level string). GCP and AWS accept major version only (`"8.0"`); Azure Flexible Server requires a specific patch version like `"8.0.21"`. Using the wrong format causes a 400 API error.

### Azure SQL version

`azurerm_mssql_server.version = "12.0"` — this is the fixed API version string for Azure SQL (maps to SQL Server 2014+ syntax). It does not mean SQL Server 2012; all modern SQL Server features are available.

### Azure CDC

Azure SQL Database uses `sys.sp_cdc_enable_db` (standard — no platform wrapper). Azure SQL Managed Instance is the same. Azure Flexible Server PostgreSQL uses `wal_level=logical` (same as on-prem).

---

## GCP deletion: gcloud CLI preferred over terraform destroy

GCP Cloud SQL deletion via `terraform destroy` can fail with `403 Permission Denied`:

```bash
gcloud sql instances delete "$INSTANCE_NAME" --project="$PROJECT_ID" --quiet
terraform destroy -auto-approve   # clean up state
```

`delete_instance` in `terraform_common.sh` handles this **only for a subset of stacks** — not all. The gcloud-first path triggers only when `CLOUD_DB_TYPE` is one of: `aws-pg`, `aws-aurora-pg`, `azure-pg`, `gcp-pg`, `gcp-mysql`, `gcp-sqlserver`. All other stacks fall through to plain `terraform destroy`. ⚠️ If you add a new GCP stack, add it to the `delete_instance` case list or deletion will silently fall back to `terraform destroy` which may fail.

## Infra vs configuration split

Terraform provisions infrastructure only. User setup and LFC configuration run as **post-apply** steps:

```
Terraform apply → RDS/Cloud SQL/Flexible Server provisioned
                → null_resource.setup_users (basic user + test tables)

Post-apply (Python or bash):
  → setup_mysql_user.py / setup_postgres_user.py
  → replication provider setup (LogMiner grants, supplemental logging, CDC)
  → LFC/Arcion configuration
```

This split exists because Terraform `local-exec` provisioners are brittle for complex SQL — failures require `terraform taint`. Python/bash post-apply scripts are easier to retry.

## terraform_common.sh orchestration and Azure region fallback

For the full `deploy_database` lifecycle, `prepare_tfvars` env var mapping, `AUTO_CONFIRM_DEPLOY`, `DELETE_DB_AFTER_SLEEP`, and Azure region fallback logic: see [orchestration.md](orchestration.md).

## Post-apply user setup: Terraform → Python

MySQL and SQL Server user setup was migrated out of `user_setup.tf` (Terraform `local-exec`) into Python scripts invoked from `mist/core.py` after apply. The `.tf` files in these stacks contain only a comment documenting the migration.

```
mist/terraform_configs/scripts/
  setup_mysql_user.py       # post-apply: LFC user + grants + binlog verify
  setup_postgres_user.py    # post-apply: LFC user + WAL + replica identity
  setup_sqlserver_user.py   # post-apply: login + user + CT/CDC
```

**⚠️ Broken references in `terraform_common.sh`:** `run_database_configuration` attempts to source `mysql/shared/setup_environment.sh` and `sqlserver/shared/setup_environment.sh` — these files do **not exist** in the repo. Only `postgres/shared/setup_environment.sh` (and per-stack variants) exist. MySQL and SQL Server configuration must be run manually or via `mist/core.py`.

## Cloud-specific networking and replication flags

For per-engine HCL patterns, Aurora vs RDS parameter groups, OCI ADB, Azure SQL serverless, GCP SQL Server local-exec: see [cloud-networking.md](cloud-networking.md).
