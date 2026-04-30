# terraform_common.sh — Orchestration and Lifecycle

## Full deployment lifecycle

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

---

## prepare_tfvars: env var → tfvars mapping

`prepare_tfvars` auto-populates `terraform.tfvars` from environment variables when the file doesn't exist:

| Env var | tfvars key | Notes |
|---------|-----------|-------|
| `WHOAMI` or `USER` | `whoami` | Resource naming prefix |
| `DBX_USERNAME` | `owner` | Owner email for tags |
| `RG_NAME` | `resource_group_name` | Azure only |
| `DB_CATALOG` | `db_catalog` | Optional — carry over from previous run |
| `DBA_USERNAME` | `dba_username` | Optional — carry over DBA credentials |
| `DBA_PASSWORD` | `dba_password` | Optional |
| `USER_USERNAME` | `username` | Optional |
| `USER_PASSWORD` | `user_password` | Optional |

**`setup_firewall_cidrs` auto-injects current IP** via `curl -s http://ipv4.icanhazip.com` and appends `/32`. This runs on every `prepare_tfvars` call (even if `terraform.tfvars` exists) to keep IPs fresh.

**Hard-coded CIDR correction** (mist-specific workaround):
```bash
# 128.77.97.250/30 is not a valid network address (host bits set)
if [[ "$cidr" == "128.77.97.250/30" ]]; then
    fixed_cidrs+=("128.77.97.248/30")
fi
```

---

## Azure region fallback

Azure regions restrict certain database services. `terraform_common.sh` implements automatic region fallback:

- Triggers when `CLOUD_DB_TYPE` contains `"azure"`
- Checks region support via `az provider show --namespace Microsoft.DBforPostgreSQL`
- Tries regions in geographic proximity order (East US → East US 2 → Central US → ...)
- Stops fallback on non-region errors (auth failure, quota, etc.)
- Detection: `"LocationIsOfferRestricted"` in apply output → try next region

This is handled in bash, not in Terraform HCL.
