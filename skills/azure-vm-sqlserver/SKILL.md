---
name: azure-vm-sqlserver
description: SQL Server 2022 on Azure Ubuntu VM — installation via az vm run-command, mssql-conf setup, validate-then-reset SA password pattern (stop/mssql-conf/start via az vm run-command), SQL Server listens on 0.0.0.0:1433 by default (no bind-address change), direct source sqlserver-configure.sh. Use when setting up or debugging SQL Server on an Azure VM, fixing SA login failures, or configuring database user access. See database-sqlserver for SQL Server login/user architecture, CT/CDC setup, sqlcmd conventions, and trustServerCertificate.
---

# Azure VM SQL Server — LFC Setup

## Installation (idempotent)

Install SQL Server 2022 via `az vm run-command invoke`. Uses the Microsoft APT repository for Ubuntu 22.04:

```bash
SA_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    set -e
    curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
      | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
    curl -fsSL https://packages.microsoft.com/config/ubuntu/22.04/mssql-server-2022.list \
      -o /etc/apt/sources.list.d/mssql-server.list
    apt-get update -qq
    ACCEPT_EULA=Y apt-get install -y mssql-server
    MSSQL_SA_PASSWORD='${SA_PASSWORD}' MSSQL_PID=developer /opt/mssql/bin/mssql-conf -n setup
    systemctl enable mssql-server && systemctl start mssql-server
  " --output none
```

> **Note:** `az vm run-command` inline scripts run under `/bin/sh` (dash) — use POSIX `[ ]` not `[[ ]]`.

## No bind-address configuration needed

Unlike MySQL (which binds to `127.0.0.1` by default) and PostgreSQL (which listens only on `localhost`), SQL Server on Linux listens on `0.0.0.0:1433` immediately after `mssql-conf setup`. No `bind-address` change is required.

The only networking requirement is the NSG rule allowing inbound TCP on port 1433 (`VirtualNetwork` + serverless egress + local IP) — handled by `customer-vm-base.sh`.

## Install state probe — always probe via az vm run-command

Always check install state by probing the VM regardless of whether `SA_PASSWORD` was loaded from the secret. Skipping the probe when a password exists is wrong — the VM may already have SQL Server running.

```bash
echo "==> Checking SQL Server install state (az vm run-command takes ~30–60 s) ..."
_install_raw=$(az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "systemctl is-active --quiet mssql-server && echo RUNNING || echo NOT_INSTALLED" \
  --query "value[0].message" -o tsv)
INSTALL_STATE=$(grep -oE 'RUNNING|NOT_INSTALLED' <<< "$_install_raw" | head -1)
[[ -n "${INSTALL_STATE:-}" ]] || {
  echo "error: install-state probe returned empty output — VM may be unreachable" >&2
  kill -INT $$
}
```

## Validate-then-reset SA password via mssql-conf

SQL Server has no `debian-sys-maint`-style admin channel. The only out-of-band password reset is `mssql-conf -n setup` with `MSSQL_SA_PASSWORD`, which requires stopping the service:

```bash
if ! sqlcmd -S "${SQL_DNS},${DB_PORT}" -d master -U sa -P "$SA_PASSWORD" -C \
    -Q "SELECT 1" &>/dev/null; then
  echo "==> SA password mismatch — resetting via mssql-conf (takes ~30 s) ..."
  az vm run-command invoke \
    --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
    --command-id RunShellScript \
    --scripts "
      set -e
      systemctl stop mssql-server
      MSSQL_SA_PASSWORD='${SA_PASSWORD}' MSSQL_PID=developer \
        /opt/mssql/bin/mssql-conf -n setup
      systemctl start mssql-server
      sleep 5
      echo '==> SA password reset'
    " --query "value[0].message" -o tsv \
    || { echo "error: failed to reset SA password" >&2; kill -INT $$; }
  # Verify after reset
  if ! sqlcmd -S "${SQL_DNS},${DB_PORT}" -d master -U sa -P "$SA_PASSWORD" -C \
      -Q "SELECT 1" &>/dev/null; then
    echo "error: SA login still failing after reset — SA_PASSWORD in secret may be stale" >&2
    kill -INT $$
  fi
  echo "==> SA password synced and verified"
else
  echo "==> SA password verified"
fi
```

**Key points:**
- `systemctl stop` before `mssql-conf` — SQL Server must be stopped for password reset.
- `sleep 5` after `systemctl start` — SQL Server takes a few seconds to accept connections after start.
- `-n` flag on `mssql-conf` suppresses interactive prompts.

## Direct source sqlserver-configure.sh from laptop

SQL Server accepts connections on the public FQDN immediately (no `bind-address` change needed):

```bash
host_fqdn="$SQL_DNS"
port="$DB_PORT"
dba__user="sa"
dba__password="${SA_PASSWORD:?SA_PASSWORD is required}"
catalog="$VM_DB_NAME"
source "$(dirname "${BASH_SOURCE[0]}")/sqlserver-configure.sh"
```

After configure, save with `host_fqdn="$VM_INTERNAL_DNS"` for PNG routing:

```bash
host_fqdn="$VM_INTERNAL_DNS"
vm_fqdn_public="$SQL_DNS"
replication_mode="ct"
access_type="vm"
dba_user="sa"
dba_catalog="master"
save_db_secret "vm-sqlserver"
```

## Secret fields for VM SQL Server

| Field | Value |
|-------|-------|
| `db_type` | `sqlserver` |
| `replication_mode` | `ct` |
| `host_fqdn` | `VM_INTERNAL_DNS` (PNG routing) |
| `vm_fqdn_public` | `SQL_DNS` (local CLI) |
| `access_type` | `vm` |
| `dba__user` | `sa` |
| `dba__password` | `<SA password>` |
| `dba_catalog` | `master` |
| `schema` | `dbo` |

> **Double-underscore naming:** `save_db_secret` accepts `dba_user=` / `dba_password=` as input parameters. The JSON stores them nested as `dba.user` / `dba.password`. When `load_db_secret` exports to the environment, nested keys are flattened with `__`: `$dba__user`, `$dba__password`.

## UC connection — trustServerCertificate required

SQL Server VM connections require `trustServerCertificate: "true"` in the UC connection options (self-signed certificate):

```bash
_build_uc_conn_json "$name" SQLSERVER vm-sqlserver "$host" "$DB_PORT" \
  '{"trustServerCertificate":"true"}'
```
