---
name: azure-flexible-server-mysql
description: Azure MySQL Flexible Server provisioning — idempotent bash scripts, parameter API field name (.currentValue not .value), subnet cleanup, database user pre-seeding, configure-before-PE section ordering. Use when writing or editing setup scripts for Azure MySQL Flexible Server, managing firewall rules and private endpoints, or debugging Azure parameter idempotency. See database-mysql for MySQL grants, binlog parameters, auth_socket, debian-sys-maint, and bind-address.
---

# Azure MySQL Flexible Server — LFC Setup

## Parameter API field name

`az mysql flexible-server parameter list` returns **`currentValue`** (not `value`) for the currently active value. Use `.currentValue // empty` in `jq` to check idempotency.

```bash
cur=$(echo "$_PARM_LIST" | jq -r --arg n "$name" \
  '.[] | select(.name==$n) | .currentValue // empty' | tr '[:upper:]' '[:lower:]')
[[ "$cur" == "$want" ]] && return 0   # already set — skip
```

## Required LFC parameters

| Parameter | Value | Notes |
|---|---|---|
| `require_secure_transport` | `OFF` | LFC does not support SSL |
| `binlog_row_image` | `full` | full row images for CDC |
| `binlog_format` | `row` | row-based replication |
| `sql_generate_invisible_primary_key` | `OFF` | LFC needs explicit PKs |
| `binlog_expire_logs_seconds` | `≥604800` | 7-day binlog retention |

Check `≥` params with `op=ge` and integer comparison (`[[ "${cur:-0}" -ge "$want" ]]`).

## Section ordering: configure before private endpoint

`mysql-configure.sh` connects via the **public FQDN** — it has no dependency on the private endpoint. Always run save-secret + configure **before** creating the PE so LFC user setup is never blocked by a PE failure.

```
1  PE subnet
2  DNS zone + VNet link
3  Create server
4  Firewall rules + params + password probe
5  Create database
6  Save secret (admin creds)       ← before PE
6b mysql-configure (LFC user)      ← before PE
7  Create private endpoint
8  Wire DNS A record
9  UC connections
```

## Stale VNet-integration subnet cleanup

When migrating from VNet-integration to public-access + PE, the delegated subnet is NOT automatically deleted when the server is deleted. Its CIDR blocks the new PE subnet. Clean it up in **two places**:

```bash
# In section 0b — when deleting a VNet-integrated server:
_OLD_SUBNET="${_EXISTING_DELEGATED##*/}"
if az network vnet subnet show ... --name "$_OLD_SUBNET" &>/dev/null; then
  az network vnet subnet delete ... --name "$_OLD_SUBNET"
fi

# In section 1 — safety net for known old subnet name before create:
_OLD_SUBNET="${USER_PREFIX}-mysql-subnet"
if az network vnet subnet show ... --name "$_OLD_SUBNET" &>/dev/null; then
  az network vnet subnet delete ... --name "$_OLD_SUBNET"
fi
```

Subnet create must have `|| { echo "error: CIDR overlap ..." >&2; kill -INT $$; }` so a missed cleanup fails loudly.

## LFC_PASSWORD pre-seeding (stable across re-runs)

Capture the loaded secret values immediately after `load_db_secret`, **before** the save-secret section overwrites `user`/`password`:

```bash
load_db_secret "azure-mysql" || true
_LOADED_USER="${user:-}"      # save before section 6 overwrites
_LOADED_PASSWORD="${password:-}"
MYSQL_ADMIN_PASSWORD="${dba__password:-${password:-${MYSQL_ADMIN_PASSWORD:-}}}"
```

Then in section 6b, pre-seed using the captured values:

```bash
LFC_USER="${LFC_USER:-lfc_user}"
[[ "${_LOADED_USER:-}" == "$LFC_USER" ]] && LFC_PASSWORD="${_LOADED_PASSWORD:-}"
source mysql-configure.sh   # generates LFC_PASSWORD only if still empty
```

Without this, `user` is always `mysqladmin` by the time the check runs, so `LFC_PASSWORD` is never restored and `mysql-configure.sh` generates a new password on every run.


## Firewall rules — IP-based idempotency

Check existing rules by IP value (not name) to avoid duplicates across different callers:

```bash
_fw_ip_exists() {
  local list_json="$1" start_ip="$2" end_ip="$3"
  local match
  match=$(echo "$list_json" | jq -r \
    --arg s "$start_ip" --arg e "$end_ip" \
    '.[] | select(.startIpAddress == $s and .endIpAddress == $e) | .name' \
    2>/dev/null | head -1)
  [[ -n "$match" ]]
}
```

Always use `${start_ip}` (braces) in echo strings — an unbraced `$start_ip–` causes bash to include the en-dash byte in the variable name lookup.

## Private endpoint NIC — async provisioning

The PE NIC ID is available asynchronously. Poll before querying the NIC IP (MySQL PE is typically faster than Postgres but the same pattern applies):

```bash
for _i in {1..12}; do
  PE_NIC_ID=$(az network private-endpoint show \
    --resource-group "$RESOURCE_GROUP" --name "$PE_NAME" \
    --query "networkInterfaces[0].id" -o tsv 2>/dev/null || true)
  [[ -n "$PE_NIC_ID" && "$PE_NIC_ID" != "None" ]] && break
  sleep 10
done
```
