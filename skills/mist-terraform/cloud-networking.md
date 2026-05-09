# Cloud Networking and Replication Flags

## AWS networking pattern for RDS

```hcl
# Use default VPC for simplicity (public subnets)
data "aws_vpc" "default" { default = true }

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
  filter {
    name   = "map-public-ip-on-launch"
    values = ["true"]
  }
}

resource "aws_db_subnet_group" "main" {
  subnet_ids = data.aws_subnets.public.ids
}

resource "aws_db_instance" "oracle" {
  publicly_accessible  = true
  db_subnet_group_name = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

Security group: merge caller IP `/32` with `additional_cidrs` from tfvars.

---

## AWS RDS PostgreSQL: replication parameter group

```hcl
resource "aws_db_parameter_group" "postgres_lfc" {
  family = "postgres16"

  parameter {
    name         = "rds.logical_replication"
    value        = "1"
    apply_method = "pending-reboot"
  }
  parameter {
    name  = "max_replication_slots"
    value = "10"
  }
  parameter {
    name  = "max_wal_senders"
    value = "15"    # Both aws-postgres and aws-aurora-postgres use 15, not 10
  }
  parameter {
    name  = "rds.force_ssl"
    value = "0"    # LFC does not support SSL
  }
}
```

---

## GCP Cloud SQL PostgreSQL: replication flags

```hcl
resource "google_sql_database_instance" "postgres" {
  settings {
    edition = "ENTERPRISE"    # required for logical decoding / replication

    database_flags {
      name  = "cloudsql.logical_decoding"
      value = var.enable_replication ? "on" : "off"
    }
    backup_configuration {
      enabled                        = var.enable_replication
      point_in_time_recovery_enabled = var.enable_replication
    }
    ip_configuration {
      ssl_mode = "ALLOW_UNENCRYPTED_AND_ENCRYPTED"  # dev/test
      dynamic "authorized_networks" {
        for_each = var.authorized_networks
        content {
          value = authorized_networks.value
        }
      }
    }
  }
}
```

`backup_configuration.enabled` is required for logical replication on GCP Cloud SQL.

---

## AWS RDS MySQL: binlog parameter group

```hcl
resource "aws_db_parameter_group" "mysql_lfc" {
  family = "mysql8.0"

  parameter { name = "binlog_format";                     value = "ROW";    apply_method = "pending-reboot" }
  parameter { name = "binlog_row_image";                  value = "full";   apply_method = "pending-reboot" }
  parameter { name = "binlog_expire_logs_seconds";        value = "604800"; apply_method = "pending-reboot" }
  parameter { name = "require_secure_transport";          value = "0";      apply_method = "immediate" }
  parameter { name = "sql_generate_invisible_primary_key"; value = "0";     apply_method = "pending-reboot" }
}
```

Admin username: `mysqladmin` (`root` is reserved on RDS — set `username = "mysqladmin"` in the instance resource).

---

## AWS RDS SQL Server: option group (not parameter group)

SQL Server on RDS uses **option groups** for feature enablement, not parameter groups.
Must use `sqlserver-se` edition — Express/Web do not support CDC.

```hcl
resource "aws_db_option_group" "sqlserver_lfc" {
  name                 = "${local.clean_instance_id}-opts"
  engine_name          = "sqlserver-se"
  major_engine_version = "15.00"   # rds_value prefix: "15.00"=2019, "16.00"=2022
}

resource "aws_db_instance" "sqlserver" {
  engine               = "sqlserver-se"
  license_model        = "license-included"
  option_group_name    = aws_db_option_group.sqlserver_lfc.name
  # DBName not set — SQL Server creates master only; catalog created post-apply
}
```

EngineVersion for SQL Server must be resolved at apply time via `data.aws_rds_engine_version`:
```hcl
data "aws_rds_engine_version" "sqlserver" {
  engine         = "sqlserver-se"
  preferred_versions = ["15.00"]   # picks latest patch in 15.00.x line
}
```

---

## GCP Cloud SQL MySQL: binary log + flags

```hcl
resource "google_sql_database_instance" "mysql" {
  database_version = "MYSQL_8_0"   # or MYSQL_8_4

  settings {
    database_flags {
      name  = "binlog_row_image"
      value = "full"
    }
    database_flags {
      name  = "expire_logs_days"    # GCP flag name — NOT binlog_expire_logs_seconds
      value = "7"
    }
    database_flags {
      name  = "require_ssl"
      value = "off"
    }
    backup_configuration {
      enabled            = true
      binary_log_enabled = true   # required for binlog CDC on Cloud SQL MySQL
    }
  }
}
```

Instance names must be **lowercase** — Cloud SQL rejects uppercase characters.

---

## Aurora vs RDS in Terraform

Aurora uses `aws_rds_cluster` + `aws_rds_cluster_instance`, not `aws_db_instance`:

```hcl
# RDS (single instance)
resource "aws_db_instance" "main" { ... }

# Aurora (cluster + member)
resource "aws_rds_cluster" "main" {
  engine = "aurora-postgresql"   # or aurora-mysql
  ...
}
resource "aws_rds_cluster_instance" "member" {
  cluster_identifier = aws_rds_cluster.main.id
  ...
}
```

Oracle and SQL Server are **not available on Aurora** — only MySQL-compatible and PostgreSQL-compatible engines exist. Use `aws_db_instance` for SQL Server and Oracle.

### Aurora PostgreSQL: dual parameter groups + differences from RDS PG

Aurora requires **two** parameter groups — one for the cluster, one for each instance.
The instance group is typically empty; replication params go on the cluster group:

```hcl
resource "aws_rds_cluster_parameter_group" "pg" {
  family = "aurora-postgresql${var.major_version}"   # e.g. "aurora-postgresql14"
  # replication params on the cluster group:
  parameter { name = "rds.logical_replication"; value = "1"; apply_method = "pending-reboot" }
  parameter { name = "max_replication_slots";   value = "10" }
  parameter { name = "max_wal_senders";         value = "15" }
  parameter { name = "rds.force_ssl";           value = "0";  apply_method = "pending-reboot" }
  # NOTE: max_worker_processes is NOT in Aurora cluster group (differs from RDS PG)
}

resource "aws_db_parameter_group" "pg_instance" {
  family = "aurora-postgresql${var.major_version}"
  # empty — instance group required but unused for replication
}
```

**Key differences from RDS PostgreSQL parameter group:**
- Family prefix is `aurora-postgresql` not `postgres`
- `max_worker_processes` is **not set** on Aurora (RDS PG sets it to `10`)
- Both cluster and instance parameter groups must be created and referenced

---

## AWS Oracle: networking and lifecycle details

```hcl
# Client IP via http provider (no manual CIDR injection)
data "http" "my_ip" {
  url = "https://ipv4.icanhazip.com"
}
locals {
  my_cidr = "${chomp(data.http.my_ip.response_body)}/32"
}

# Public subnets only (filter map-public-ip-on-launch)
data "aws_subnets" "public" {
  filter { name = "vpc-id";                    values = [data.aws_vpc.default.id] }
  filter { name = "map-public-ip-on-launch";   values = ["true"] }
}

resource "aws_db_instance" "oracle" {
  engine               = "oracle-se2"
  license_model        = "license-included"
  character_set_name   = "AL32UTF8"
  db_name              = local.sid    # "ORCL" + 4 chars, ≤8 total
  publicly_accessible  = true

  lifecycle {
    ignore_changes = [password]   # prevent Terraform drift on externally-rotated password
  }
}
```

Extra `environment_variables` keys for Oracle (not present on other engines):
`JDBC_URL`, `SQLPLUS_CONNECT`, `CLOUD_CONSOLE_URL`, `CLI_COMMAND`, `AWS_REGION`.

---

## OCI Autonomous Database (Always Free)

```hcl
resource "oci_database_autonomous_database" "main" {
  is_free_tier              = true
  cpu_core_count            = 1        # Always Free limit
  data_storage_size_in_tbs  = 1        # Always Free limit — verify current OCI limits
  db_name                   = local.db_name   # uppercase, ≤14 chars, no hyphens
  admin_password            = local.dba_password   # Oracle-safe special chars
  whitelisted_ips           = [local.my_cidr]   # different naming from DB_FIREWALL_CIDRS
  compartment_id            = var.compartment_id
}
```

**OCI-specific conventions:**
- `db_name` ≤14 uppercase alphanumeric chars (strip hyphens, prepend letter prefix if needed)
- `DBA_USERNAME` = `USER_USERNAME` = `"ADMIN"` — ADB has a single ADMIN superuser; create app users post-provision if needed
- Firewall uses `whitelisted_ips` list (not `authorized_networks` / `additional_cidrs`)
- Extra output keys: `OCI_COMPARTMENT_ID`, `OCI_REGION`, `SERVICE_CONSOLE_URL`

---

## Azure SQL Database: DTU vs serverless vCore

mist uses `azurerm_mssql_server` + `azurerm_mssql_database` (Azure SQL Database PaaS), **not** a VM. Default SKU is Basic (DTU model). Serverless is enabled when the SKU matches `^GP_S_`:

```hcl
resource "azurerm_mssql_database" "main" {
  sku_name = var.sku_name   # e.g. "Basic", "S1", "GP_S_Gen5_1"

  # Serverless-only settings (applied conditionally)
  auto_pause_delay_in_minutes = startswith(var.sku_name, "GP_S_") ? var.auto_pause_delay : -1
  min_capacity                = startswith(var.sku_name, "GP_S_") ? var.min_capacity      : null
}
```

| SKU prefix | Model | Notes |
|------------|-------|-------|
| `Basic`, `S*`, `P*` | DTU | Fixed compute; no auto-pause |
| `GP_Gen5_*`, `BC_Gen5_*` | vCore provisioned | Per-vCore pricing |
| `GP_S_Gen5_*` | vCore serverless | Auto-pause after idle; `sqlcmd -l 180` needed |

The `azure-sql-server` skill covers the **serverless auto-pause `sqlcmd` pattern**. The DTU vs serverless distinction is the Terraform-level control.

---

## GCP SQL Server: local-exec user setup

GCP Cloud SQL SQL Server cannot run `CREATE LOGIN` / `ALTER SERVER ROLE` via the standard Users API. mist uses a `null_resource` + `local-exec` to run Python/pymssql after the instance is ready:

```hcl
resource "null_resource" "sqlserver_user_setup" {
  depends_on = [google_sql_database_instance.main]

  provisioner "local-exec" {
    command = <<-EOF
      python3 -c "
      import pymssql, json
      conn = pymssql.connect(host='${google_sql_database_instance.main.ip_address[0].ip_address}',
                             user='sqlserver', password='${local.root_password}', database='master',
                             tls_disable_verify=True)
      # CREATE USER + roles
      "
    EOF
  }
}
```

**Why:** Cloud SQL SQL Server's Users API creates database-level users, not server-level logins. The full login → user → `db_owner` + `db_ddladmin` sequence requires a direct SQL connection.

`DBA_USERNAME` in `environment_variables` is hardcoded to the string `"sqlserver"` (the built-in Cloud SQL admin) — not the random `dba_username` variable used by other stacks.
