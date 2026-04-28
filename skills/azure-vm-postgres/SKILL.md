---
name: azure-vm-postgres
description: PostgreSQL 16 on Azure Ubuntu VM for LakeFlow Connect — installation via az vm run-command, listen_addresses=* + pg_hba.conf for remote access via az vm run-command, POSIX sh compatibility for inline scripts (/bin/sh not bash), generate-before-validate pattern for empty passwords, validate-then-reset postgres password via peer-auth + az vm run-command, direct source pg-configure.sh. Use when setting up or debugging PostgreSQL on an Azure VM, fixing connection-refused errors, syncing postgres passwords, or configuring LFC access. See database-postgres for PostgreSQL grants, WAL parameters, replica identity, peer auth, and scram-sha-256 details.
---

# Azure VM PostgreSQL — LFC Setup

## Installation (idempotent)

Install PostgreSQL 16 via `az vm run-command invoke`. The install script:
1. Adds the official PostgreSQL APT repository
2. Installs `postgresql-16`
3. Sets `postgres` user password
4. Configures `listen_addresses = '*'` and `pg_hba.conf` for remote access
5. Restarts PostgreSQL

```bash
PG_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    set -e
    apt-get update -qq && apt-get install -y gnupg curl
    curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
      | gpg --dearmor -o /usr/share/keyrings/postgresql.gpg
    echo 'deb [signed-by=/usr/share/keyrings/postgresql.gpg] https://apt.postgresql.org/pub/repos/apt jammy-pgdg main' \
      > /etc/apt/sources.list.d/pgdg.list
    apt-get update -qq && apt-get install -y postgresql-16
    systemctl enable postgresql && systemctl start postgresql
    sudo -u postgres psql -c \"ALTER USER postgres WITH PASSWORD '${PG_PASSWORD}';\"
    echo \"listen_addresses = '*'\" >> /etc/postgresql/16/main/postgresql.conf
    echo 'host all all 0.0.0.0/0 scram-sha-256' >> /etc/postgresql/16/main/pg_hba.conf
    systemctl restart postgresql
  " --output none
```

## POSIX shell compatibility for inline az vm run-command scripts

`az vm run-command invoke --scripts "..."` executes under `/bin/sh` (dash) on Ubuntu — **not** bash. Bash-specific syntax causes `"not found"` errors:

| Don't use | Use instead | Why |
|-----------|-------------|-----|
| `[[ condition ]]` | `[ condition ]` | `[[` is bash-only |
| `\s` in grep | `[[:space:]]` | `\s` is not POSIX |
| `${var:-default}` inside `[[ ]]` | standalone `[ ]` | mixed semantics |

```bash
# BAD — fails under /bin/sh (dash) on Ubuntu
if [[ "$CHANGED" -eq 1 ]]; then systemctl restart postgresql; fi

# GOOD — POSIX-compatible
if [ "$CHANGED" -eq 1 ]; then systemctl restart postgresql; fi
```

## listen_addresses + pg_hba: idempotent fix for existing installs

When a VM was installed before the public IP was added, PostgreSQL may listen only on `localhost`. Fix idempotently and restart only if a change was needed:

```bash
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    set -e
    ufw allow ${DB_PORT}/tcp 2>/dev/null || true
    CFG=\$(ls /etc/postgresql/*/main/postgresql.conf | head -1)
    HBA=\$(ls /etc/postgresql/*/main/pg_hba.conf | head -1)
    CHANGED=0
    if ! grep -qE \"^listen_addresses[[:space:]]*=[[:space:]]*'\\*'\" \"\$CFG\" 2>/dev/null; then
      sed -i \"s/^#*listen_addresses.*/listen_addresses = '*'/\" \"\$CFG\" \
        || echo \"listen_addresses = '*'\" >> \"\$CFG\"
      CHANGED=1
    fi
    if ! grep -q 'host all all 0.0.0.0/0 scram-sha-256' \"\$HBA\" 2>/dev/null; then
      echo 'host all all 0.0.0.0/0 scram-sha-256' >> \"\$HBA\"
      CHANGED=1
    fi
    if [ \"\$CHANGED\" -eq 1 ]; then
      systemctl restart postgresql
      echo '==> PostgreSQL reconfigured for external access'
    else
      echo '==> already configured — skipping'
    fi
  " --query "value[0].message" -o tsv \
  || { echo "error: failed to configure PostgreSQL network access" >&2; kill -INT $$; }
```

## Generate PG_PASSWORD before validate-then-reset

`scram-sha-256` rejects empty passwords. On existing VMs where the secret may not yet have a password (e.g. first run after adding the VM to an existing setup), generate a password **before** the validate step:

```bash
if [[ -z "${PG_PASSWORD:-}" ]]; then
  PG_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
  echo "==> Generated PG_PASSWORD — will be saved to secret"
fi
```

Generating first ensures `ALTER USER postgres WITH PASSWORD '${PG_PASSWORD}'` never sets an empty password, which would be rejected by scram-sha-256 on the next connection attempt.

## Admin access: peer authentication channel (sudo -u postgres psql)

PostgreSQL on Ubuntu uses peer authentication for `postgres` on the local socket — `sudo -u postgres psql` always has admin access regardless of the TCP password:

```bash
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    sudo -u postgres psql -c \"ALTER USER postgres WITH PASSWORD '${PG_PASSWORD}';\"
    echo '==> postgres password synced'
  " --query "value[0].message" -o tsv \
  || { echo "error: failed to sync postgres password" >&2; kill -INT $$; }
```

## Validate-then-reset postgres password

```bash
if ! PGPASSWORD="$PG_PASSWORD" psql \
    -h "$SQL_DNS" -p "$DB_PORT" -U postgres -d postgres \
    --connect-timeout 10 -c "SELECT 1" &>/dev/null; then
  echo "==> postgres password mismatch — syncing via peer auth ..."
  az vm run-command invoke \
    --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
    --command-id RunShellScript \
    --scripts "
      sudo -u postgres psql -c \"ALTER USER postgres WITH PASSWORD '${PG_PASSWORD}';\"
      echo '==> postgres password synced'
    " --query "value[0].message" -o tsv \
    || { echo "error: failed to sync postgres password" >&2; kill -INT $$; }
  # Verify — capture stderr to distinguish auth vs network failure
  _PG_ERR=$(PGPASSWORD="$PG_PASSWORD" psql \
    -h "$SQL_DNS" -p "$DB_PORT" -U postgres -d postgres \
    --connect-timeout 10 -c "SELECT 1" 2>&1 >/dev/null || true)
  if [[ -n "${_PG_ERR}" ]]; then
    echo "error: postgres login still failing after sync" >&2
    echo "       ${_PG_ERR}" >&2
    echo "       connection refused/timeout: check ufw/NSG and listen_addresses" >&2
    echo "       password auth failed: re-run to rotate PG_PASSWORD" >&2
    kill -INT $$
  fi
  echo "==> postgres password synced and verified"
else
  echo "==> postgres password verified"
fi
```

## Direct source pg-configure.sh from laptop

Once PostgreSQL listens on `0.0.0.0` and has a public IP, source the configure script directly:

```bash
host_fqdn="$SQL_DNS"
port="$DB_PORT"
dba__user="postgres"
dba__password="${PG_PASSWORD:?PG_PASSWORD is required}"
catalog="$VM_DB_NAME"
source "$(dirname "${BASH_SOURCE[0]}")/pg-configure.sh"
```

After configure, save with `host_fqdn="$VM_INTERNAL_DNS"` for PNG routing:

```bash
host_fqdn="$VM_INTERNAL_DNS"
vm_fqdn_public="$SQL_DNS"
replication_mode="logicalreplication"
access_type="vm"
dba_user="postgres"
dba_catalog="postgres"
save_db_secret "vm-postgres"
```

## Secret fields for VM PostgreSQL

| Field | Value |
|-------|-------|
| `db_type` | `postgresql` |
| `replication_mode` | `logicalreplication` |
| `host_fqdn` | `VM_INTERNAL_DNS` (PNG routing) |
| `vm_fqdn_public` | `SQL_DNS` (local CLI) |
| `access_type` | `vm` |
| `dba_user` | `postgres` |
| `dba_catalog` | `postgres` |
| `schema` | `public` |
