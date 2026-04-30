---
name: azure-vm-mysql
description: MySQL 8 on Azure Ubuntu VM — installation via az vm run-command, bind-address=0.0.0.0 (Azure VM networking), root@% user creation, debian-sys-maint admin channel via az vm run-command, validate-then-reset root password pattern, database user pre-seeding, direct source mysql-configure.sh from laptop. Use when setting up or debugging MySQL on an Azure VM, fixing connection-refused errors, syncing root passwords, or configuring database user access. See database-mysql for MySQL grants, binlog parameters, bind-address config, and auth_socket details.
---

# Azure VM MySQL — LFC Setup

## Installation (idempotent)

Install MySQL 8 via `az vm run-command invoke`. The install script:
1. `apt-get install mysql-server`
2. Creates `root@'%'` with a password (keeps `root@'localhost'` on auth_socket for `sudo mysql`)
3. Sets `bind-address = 0.0.0.0` so the database accepts remote connections immediately

```bash
ROOT_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    set -e
    apt-get update -qq
    DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
    systemctl enable mysql && systemctl start mysql
    # root@localhost stays on auth_socket (sudo mysql still works).
    # root@'%' gets a password for remote/TCP access.
    mysql -e \"CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY '${ROOT_PASSWORD}';\"
    mysql -e \"GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;\"
    mysql -e \"FLUSH PRIVILEGES;\"
    sed -i 's/bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
    systemctl restart mysql
  " \
  --output none
```

> **Note:** `az vm run-command` inline scripts run under `/bin/sh` (dash) on Ubuntu — use POSIX `[ ]` and `[[:space:]]`, not bash `[[ ]]` or `\s` regex.

## bind-address: idempotent fix for existing installs

When a VM was installed before the public IP was added, MySQL may still bind to `127.0.0.1`. Fix idempotently:

```bash
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    CFG=/etc/mysql/mysql.conf.d/mysqld.cnf
    if grep -qE '^bind-address[[:space:]]*=[[:space:]]*0\.0\.0\.0' \"\$CFG\" 2>/dev/null; then
      echo 'already 0.0.0.0 — skipping'
    else
      sed -i 's/bind-address.*/bind-address = 0.0.0.0/' \"\$CFG\"
      systemctl restart mysql
      echo 'bind-address set to 0.0.0.0'
    fi
  " --query "value[0].message" -o tsv
```

## Admin access: debian-sys-maint channel (not sudo mysql)

On VMs where MySQL is configured for password authentication, `sudo mysql` may fail (`ERROR 1045`). Use `mysql --defaults-file=/etc/mysql/debian.cnf` instead — this uses the `debian-sys-maint` account which always has admin access:

```bash
# On the VM via az vm run-command — used when direct TCP login fails
az vm run-command invoke \
  --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "
    mysql --defaults-file=/etc/mysql/debian.cnf <<'SQL'
ALTER USER 'root'@'%' IDENTIFIED BY '${ROOT_PASSWORD}';
FLUSH PRIVILEGES;
SQL
    echo '==> root@% password synced'
  " --query "value[0].message" -o tsv
```

> The inner `<<'SQL'` heredoc is single-quoted (literal) — `${ROOT_PASSWORD}` is expanded by the **outer** laptop bash before the string is sent to the VM, which is intentional.

## Validate-then-reset root@'%' password

On re-runs, always validate the stored password first before attempting a reset. A reset via `az vm run-command` takes ~30 s — skip it on every run if the credential is already correct.

```bash
if ! MYSQL_PWD="$ROOT_PASSWORD" mysql \
    --user root --host "$SQL_DNS" --port "$DB_PORT" \
    --connect-timeout 10 --execute "SELECT 1" &>/dev/null; then
  echo "==> root@'%' password mismatch — syncing via debian-sys-maint ..."
  az vm run-command invoke \
    --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
    --command-id RunShellScript \
    --scripts "
      mysql --defaults-file=/etc/mysql/debian.cnf <<'SQL'
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY '${ROOT_PASSWORD}';
ALTER  USER 'root'@'%' IDENTIFIED BY '${ROOT_PASSWORD}';
GRANT  ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH  PRIVILEGES;
SQL
      echo '==> root@% password synced'
    " --query "value[0].message" -o tsv \
    || { echo "error: failed to sync root@'%' password" >&2; kill -INT $$; }
  # Verify after reset — don't trust the az command's exit code alone
  if ! MYSQL_PWD="$ROOT_PASSWORD" mysql \
      --user root --host "$SQL_DNS" --port "$DB_PORT" \
      --connect-timeout 10 --execute "SELECT 1" &>/dev/null; then
    echo "error: root@'%' still failing after sync — ROOT_PASSWORD in secret may be stale" >&2
    kill -INT $$
  fi
  echo "==> root@'%' password synced and verified"
else
  echo "==> root@'%' password verified"
fi
```

## LFC user pre-seeding

Restore `LFC_PASSWORD` from the loaded secret when the loaded user matches `LFC_USER`, to avoid rotating the password on every re-run:

```bash
load_db_secret "vm-mysql" || true
_LOADED_USER="${user:-}"
_LOADED_PASSWORD="${password:-}"
ROOT_PASSWORD="${dba__password:-${password:-${ROOT_PASSWORD:-}}}"

# ... (later, before sourcing mysql-configure.sh)
LFC_USER="${LFC_USER:-lfc_user}"
[[ "${_LOADED_USER:-}" == "$LFC_USER" ]] && LFC_PASSWORD="${_LOADED_PASSWORD:-}"
if [[ -z "${LFC_PASSWORD:-}" ]]; then
  LFC_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
fi
export LFC_USER LFC_PASSWORD
```

## Direct source mysql-configure.sh from laptop

Once MySQL listens on `0.0.0.0` and has a public IP, source the configure script from the laptop exactly like Azure Flexible Server. No `az vm run-command` wrapper is needed:

```bash
host_fqdn="$SQL_DNS"          # public FQDN — reachable from laptop
port="$DB_PORT"
dba__user="root"
dba__password="${ROOT_PASSWORD:?ROOT_PASSWORD is required}"
catalog="$VM_DB_NAME"
source "$(dirname "${BASH_SOURCE[0]}")/mysql-configure.sh"
```

After configure completes, save the secret with `host_fqdn="$VM_INTERNAL_DNS"` (private DNS for PNG):

```bash
host_fqdn="$VM_INTERNAL_DNS"    # PNG routing — private DNS
vm_fqdn_public="$SQL_DNS"       # local CLI — public FQDN
replication_mode="binlog"
access_type="vm"
save_db_secret "vm-mysql"
```

## Secret fields for VM MySQL

| Field | Value |
|-------|-------|
| `db_type` | `mysql` |
| `replication_mode` | `binlog` |
| `host_fqdn` | `VM_INTERNAL_DNS` (private DNS, PNG routing) |
| `vm_fqdn_public` | `SQL_DNS` (public FQDN, local CLI) |
| `access_type` | `vm` |
| `dba__user` | `root` |
| `dba__password` | `<root password>` |
| `dba_catalog` | `mysql` |

> **Double-underscore naming:** `save_db_secret` accepts `dba_user=` / `dba_password=` as input parameters. The JSON stores them nested as `dba.user` / `dba.password`. When `load_db_secret` exports to the environment, nested keys are flattened with `__`: `$dba__user`, `$dba__password`.
