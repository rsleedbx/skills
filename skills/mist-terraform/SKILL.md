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

## GCP deletion: gcloud CLI preferred over terraform destroy

GCP Cloud SQL deletion via `terraform destroy` can fail with `403 Permission Denied`:

```bash
gcloud sql instances delete "$INSTANCE_NAME" --project="$PROJECT_ID" --quiet
terraform destroy -auto-approve   # clean up state
```

`delete_instance` in `terraform_common.sh` handles this automatically when `CLOUD_DB_TYPE=gcp-pg`.

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

## Cloud-specific networking and replication flags

For the AWS default VPC networking pattern, AWS RDS PostgreSQL replication parameter group, and GCP Cloud SQL PostgreSQL logical decoding flags: see [cloud-networking.md](cloud-networking.md).
