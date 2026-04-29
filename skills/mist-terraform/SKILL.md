---
name: mist-terraform
description: Mist database provisioning Terraform patterns — flat per-cloud stacks (no shared modules), environment_variables output contract, locals.common_tags, random credential generation with Oracle-safe override_special, clean_instance_id naming, firewall CIDR variable names by cloud (additional_cidrs/allowed_cidrs/authorized_networks), terraform_common.sh orchestration functions, deploy_database lifecycle, Azure region fallback, GCP delete via gcloud CLI, infra-vs-config split (Terraform provisions, Python/bash configures). Use when writing or reading mist Terraform stacks, consuming environment_variables output, setting up firewall CIDRs, orchestrating multi-cloud deploys, or handling cloud-specific deletion.
---

# Mist Terraform Patterns

## Stack layout: flat per-cloud, no shared modules

Mist uses **standalone root modules** for each engine+cloud combination. There are no shared HCL `module "..."` blocks:

```
mist/terraform_configs/
  oracle/
    aws-oracle/      main.tf  rds.tf  security.tf  user_setup.tf  variables.tf  outputs.tf
    oci-oracle/      main.tf  autonomous_database.tf  user_setup.tf  outputs.tf
  mysql/
    aws-mysql/       main.tf  rds.tf  security.tf  user_setup.tf  variables.tf  outputs.tf
    gcp-mysql/       main.tf  cloudsql.tf  ...
    azure-mysql/     main.tf  flexible_server.tf  ...
  postgres/
    aws-postgres/    main.tf  rds.tf  ...
    aws-aurora-postgres/
    gcp-postgres/    main.tf  cloudsql.tf  ...
    azure-postgres/  ...
  sqlserver/
    ...
  shared/
    terraform_common.sh  ← bash orchestration, not HCL
```

Conventions shared across stacks by **naming convention**, not by Terraform module references.

## environment_variables output: the stable API contract

Every stack must output `environment_variables` as a **map** of uppercase string keys. This is the contract consumed by `terraform_common.sh` and downstream Python code:

```hcl
output "environment_variables" {
  description = "Environment variables for database connection"
  sensitive   = true
  value = {
    DB_HOST_FQDN    = aws_db_instance.oracle.address
    DB_PORT         = tostring(aws_db_instance.oracle.port)
    DB_CATALOG      = aws_db_instance.oracle.db_name
    USER_USERNAME   = local.username
    USER_PASSWORD   = local.user_password
    DBA_USERNAME    = local.dba_username
    DBA_PASSWORD    = local.dba_password
    DB_TYPE         = "oracle"     # lowercase engine name
    DB_VERSION      = aws_db_instance.oracle.engine_version
    CLOUD_PROVIDER  = "aws"
    CONNECTION_TYPE = "ORACLE"     # uppercase for driver/connector selection
    SOURCE_TYPE     = "ORACLE"
    AWS_REGION      = var.aws_region
    JDBC_URL        = "jdbc:oracle:thin:@${...}:${...}:${...}"
    CLI_COMMAND     = "aws rds describe-db-instances --db-instance-identifier ${var.instance_id} ..."
  }
}
```

**Consuming in bash:**
```bash
eval "$(terraform output -json environment_variables \
  | jq -r 'to_entries[] | "export \(.key)=\(.value | @sh)"')"
# Now $DB_HOST_FQDN, $DB_PORT, $DB_CATALOG etc. are in the shell environment
```

**Standard keys** (all stacks must provide):

| Key | Example | Purpose |
|-----|---------|---------|
| `DB_HOST_FQDN` | `mydb.us-east-1.rds.amazonaws.com` | Hostname for connections |
| `DB_PORT` | `"5432"` (string) | Port — always string, never number |
| `DB_CATALOG` | `"mydb"` | Database name / Oracle SID |
| `USER_USERNAME` | `"u4a9b3c..."` | Regular (non-DBA) user |
| `USER_PASSWORD` | — | Regular user password (sensitive) |
| `DBA_USERNAME` | `"a9f2e1..."` | Admin/DBA user |
| `DBA_PASSWORD` | — | DBA password (sensitive) |
| `DB_TYPE` | `"oracle"` | Lowercase engine name |
| `CLOUD_PROVIDER` | `"aws"` | Cloud: aws, azure, gcp, oci |
| `CONNECTION_TYPE` | `"ORACLE"` | Uppercase, maps to driver selection |
| `CLOUD_CONSOLE_URL` | `https://console.aws.amazon.com/rds/...` | Direct link to cloud console resource |
| `CLI_COMMAND` | `aws rds describe-db-instances ...` | CLI command to inspect the instance |

**Oracle-specific keys** (AWS RDS):

| Key | Example |
|-----|---------|
| `JDBC_URL` | `jdbc:oracle:thin:@host:1521:SID` |
| `SQLPLUS_CONNECT` | `user/pass@host:1521/SID` |

**OCI-specific keys** (Autonomous Database only — no `JDBC_URL`/`SQLPLUS_CONNECT`):

| Key | Example |
|-----|---------|
| `OCI_COMPARTMENT_ID` | `ocid1.compartment.oc1..aaaaa...` |
| `OCI_REGION` | `us-ashburn-1` |
| `SERVICE_CONSOLE_URL` | `https://cloud.oracle.com/db/adb/...` |

> **OCI note**: `USER_USERNAME` and `DBA_USERNAME` are both `"ADMIN"` for Autonomous Database. Create application users post-provision via SQL if needed.

## locals.common_tags

All stacks define a `common_tags` local for consistent tagging:

```hcl
locals {
  common_tags = merge(var.tags, {
    Name        = var.instance_id
    Owner       = var.owner
    Creator     = var.owner
    Environment = var.environment
    Project     = "lfcddemo"
    Service     = "mist"
    ManagedBy   = "terraform"
    Database    = "oracle"          # lowercase engine name
    DateCreated = var.date_created  # ISO date string — cloud-native cleanup hook
    RemoveAfter = var.remove_after  # ISO date string — cloud-native cleanup hook
  })
}
```

Standard variable inputs for tags:

```hcl
variable "instance_id"   {}  # unique identifier for this DB instance
variable "owner"         {}  # owner email (DBX_USERNAME from mist env)
variable "environment"   { default = "staging" }
variable "tags"          { default = {} type = map(string) }
variable "date_created"  {}
variable "remove_after"  {}
```

## clean_instance_id: safe resource name pattern

Cloud resource names often have restrictions (lowercase, no underscores, no dots). Derive a safe name:

```hcl
locals {
  clean_instance_id = lower(replace(replace(var.instance_id, "_", "-"), ".", "-"))
  # "my_db.prod" → "my-db-prod"
}
```

Use `local.clean_instance_id` for resource names (`aws_db_instance.id`, `google_sql_database_instance.name`, etc.) and `var.instance_id` for tags/display names.

## Random credential generation

### Username constraints by engine

Usernames must start with a letter (Oracle requirement; good practice for all engines):

```hcl
resource "random_string" "username_base" {
  length  = 14
  special = false
  upper   = false
  lower   = true
  numeric = true
}

locals {
  username     = "u${random_string.username_base.result}"    # regular user
  dba_username = "a${random_string.dba_base.result}"         # admin user
}
```

### Password constraints by engine

Oracle excludes `/`, `"`, `@`, `&`, space. Use `override_special`:

```hcl
resource "random_password" "oracle_pass" {
  length           = 16
  special          = true
  upper            = true
  lower            = true
  numeric          = true
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
  override_special = "!#$%*()-_=+[]{}|:;<>,.?"  # excludes / " @ & space
}
```

MySQL/PostgreSQL/SQL Server are less restrictive — `override_special` is optional.

### Conditional generation (only if not provided)

```hcl
# Generate only if var is null (allows bring-your-own credential)
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
resource "random_string" "db_name" {
  length  = 4
  special = false
  upper   = true
  lower   = false
  numeric = true
}

locals {
  db_name = "ORCL${upper(random_string.db_name.result)}"
  # e.g. "ORCL4A2B" — 8 chars, uppercase, alphanumeric
  sid     = local.db_name
}
```

SID limit: 8 characters. `ORCL` + 4 random = exactly 8.

## Firewall CIDR variable names by cloud

Each cloud uses a different variable name for the allowed IP list in `terraform.tfvars`:

| Cloud | Variable name | Terraform resource |
|-------|-------------|-------------------|
| AWS | `additional_cidrs` | `aws_security_group_rule` ingress |
| Azure | `allowed_cidrs` | `azurerm_firewall_policy_rule_collection` or NSG |
| GCP | `authorized_networks` | `google_sql_database_instance.settings.ip_configuration.authorized_networks` |

`terraform_common.sh` auto-injects the caller's current IP into the appropriate variable:

```bash
# DB_FIREWALL_CIDRS env var → injected into terraform.tfvars
# Format: space-separated CIDRs: "1.2.3.4/32 10.0.0.0/24"
export DB_FIREWALL_CIDRS="1.2.3.4/32"
```

The `setup_firewall_cidrs` function in `terraform_common.sh` fetches the caller's current IP via `curl -s http://ipv4.icanhazip.com` and appends it automatically.

## terraform_common.sh: prepare_tfvars details

`prepare_tfvars` auto-populates `terraform.tfvars` from environment variables when the file doesn't exist:

| Env var | tfvars key | Notes |
|---------|-----------|-------|
| `WHOAMI` or `USER` | `whoami` | Resource naming prefix |
| `DBX_USERNAME` | `owner` | Owner email for tags |
| `RG_NAME` | `resource_group_name` | Azure only — reuse existing resource group |
| `DB_CATALOG` | `db_catalog` | Optional — carry over from previous run |
| `DBA_USERNAME` | `dba_username` | Optional — carry over DBA credentials |
| `DBA_PASSWORD` | `dba_password` | Optional — carry over DBA credentials |
| `USER_USERNAME` | `username` | Optional — regular user |
| `USER_PASSWORD` | `user_password` | Optional — regular user password |

**`setup_firewall_cidrs` auto-injects current IP** via `curl -s http://ipv4.icanhazip.com` and appends `/32`. This runs on every `prepare_tfvars` call (even if `terraform.tfvars` exists) to keep IPs fresh.

**Hard-coded CIDR correction** (mist-specific bug workaround):
```bash
# 128.77.97.250/30 is not a valid network address (host bits set)
# corrected to the actual network address:
if [[ "$cidr" == "128.77.97.250/30" ]]; then
    fixed_cidrs+=("128.77.97.248/30")
fi
```

## terraform_common.sh: orchestration lifecycle

`terraform_common.sh` provides the full deployment lifecycle as bash functions:

```bash
source mist/terraform_configs/shared/terraform_common.sh

# Full lifecycle in one call
deploy_database "aws" "Oracle RDS" "$0" "$(dirname $0)/configure.sh"

# Or step-by-step:
check_prerequisites "aws"         # verifies terraform + aws CLI installed
setup_terraform_environment "$0"  # cd to terraform/ dir
prepare_tfvars "aws"              # creates from .example, injects CIDRs + owner
terraform_init                    # terraform init
terraform_plan                    # terraform plan -out=tfplan
confirm_deployment "Oracle RDS"   # interactive y/N (skip with AUTO_CONFIRM_DEPLOY=true)
terraform_apply                   # terraform apply tfplan (Azure adds region fallback)
setup_environment_from_terraform  # exports environment_variables to shell
setup_auto_deletion               # optional: DELETE_DB_AFTER_SLEEP=2h → background kill
run_database_configuration "configure.sh"  # sources config script after Terraform
```

### AUTO_CONFIRM_DEPLOY: skip interactive prompt

```bash
AUTO_CONFIRM_DEPLOY=true source terraform_common.sh && deploy_database ...
```

### DELETE_DB_AFTER_SLEEP: auto-teardown

```bash
DELETE_DB_AFTER_SLEEP=2h deploy_database ...
# Starts background job that calls delete_instance after 2 hours
```

## Azure region fallback

Azure regions restrict certain database services. `terraform_common.sh` implements automatic region fallback:

```bash
# Triggers when CLOUD_DB_TYPE contains "azure"
# Checks region support via: az provider show --namespace Microsoft.DBforPostgreSQL
# Tries regions in geographic proximity order (East US → East US 2 → Central US → ...)
# Stops fallback on non-region errors (auth failure, quota, etc.)
# Detection: "LocationIsOfferRestricted" in apply output → try next region
```

This is an infrastructure concern handled in bash, not in Terraform HCL.

## GCP deletion: gcloud CLI preferred over terraform destroy

GCP Cloud SQL deletion via `terraform destroy` can fail with `403 Permission Denied`. Use `gcloud sql instances delete` first:

```bash
gcloud sql instances delete "$INSTANCE_NAME" --project="$PROJECT_ID" --quiet
# Then clean up Terraform state:
terraform destroy -auto-approve
```

`delete_instance` in `terraform_common.sh` handles this automatically when `CLOUD_DB_TYPE=gcp-pg`.

## Infra vs configuration split

Terraform provisions infrastructure only. Database user setup and LFC configuration run as **post-apply** steps:

```
Terraform apply
  → RDS/Cloud SQL/Flexible Server provisioned
  → null_resource.setup_users (sqlplus/psql/sqlcmd) — basic user + test tables

Post-apply (Python or bash):
  → setup_mysql_user.py / setup_postgres_user.py
  → replication provider setup (LogMiner grants, supplemental logging, CDC)
  → LFC/Arcion configuration
```

This split exists because Terraform `local-exec` provisioners are brittle for complex SQL — they run once and failures require `terraform taint`. Python/bash post-apply scripts are easier to retry.

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
  publicly_accessible  = true     # required for external CDC tool access
  db_subnet_group_name = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  ...
}
```

Security group: merge caller IP `/32` with `additional_cidrs` from tfvars.

## AWS RDS PostgreSQL: replication parameter group

```hcl
resource "aws_db_parameter_group" "postgres_lfc" {
  family = "postgres16"

  parameter {
    name  = "rds.logical_replication"
    value = "1"
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
      enabled            = var.enable_replication  # required for logical replication
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
