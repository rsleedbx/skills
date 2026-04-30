---
name: azure-flexible-server-postgres
description: Azure PostgreSQL Flexible Server provisioning — idempotent bash scripts, parameter API field name (.value not .currentValue for Postgres), subnet cleanup, database user pre-seeding, configure-before-PE section ordering. Use when writing or editing setup scripts for Azure PostgreSQL Flexible Server, managing firewall rules, private endpoints, Unity Catalog connections, or debugging Azure parameter idempotency. See database-postgres for PostgreSQL grants, WAL parameters, replica identity, pg_hba.conf, and replication slots.
---

# Azure PostgreSQL Flexible Server — LFC Setup

## Parameter API field name

`az postgres flexible-server parameter list` returns **`value`** (NOT `currentValue` — that field does not exist for Postgres). MySQL uses `currentValue`; Postgres uses `value`.

```bash
cur=$(echo "$_PARM_LIST" | jq -r --arg n "$name" \
  '.[] | select(.name==$n) | .value // empty' | tr '[:upper:]' '[:lower:]')
[[ "$cur" == "$want" ]] && return 0   # already set — skip
```

## Required LFC parameters

| Parameter | Value | Notes |
|---|---|---|
| `wal_level` | `logical` | enables logical replication (CDC); requires server restart |
| `require_secure_transport` | `off` | LFC does not support SSL |

Both require a server restart to take effect. Track with `_PARAM_SET="1"` and restart once after all changes.


## Section ordering: configure before private endpoint

`pg-configure.sh` connects via the **public FQDN** — it has no dependency on the private endpoint. Always run save-secret + configure **before** creating the PE so LFC user setup is never blocked by a PE failure.

```
1   PE subnet
2   DNS zone + VNet link
3   Create server
4   Firewall rules (local IP + serverless egress)
4b  Admin password probe / reset
4c  Server parameters (wal_level, require_secure_transport)
5   Create database
6   Save secret (admin creds)       ← before PE
6b  pg-configure (LFC user)         ← before PE
7   Create private endpoint
8   Wire DNS A record
9   UC connections
```

## Stale VNet-integration subnet — CIDR overlap

When migrating from VNet-integration (`delegatedSubnetResourceId`) to public-access + PE, the delegated subnet is NOT deleted when the server is deleted. Its CIDR blocks the new PE subnet create with `NetcfgSubnetRangesOverlap`.

`az network vnet subnet show` returns non-zero for a subnet in certain transitional states, so the create is attempted — and fails. With `|| kill -INT $$` on the create, the script aborts immediately with a clear error.

Cleanup pattern:

```bash
# In section 0b — when a VNet-integrated server is detected and deleted:
_OLD_SUBNET="${_EXISTING_DELEGATED##*/}"
if az network vnet subnet show ... --name "$_OLD_SUBNET" &>/dev/null; then
  az network vnet subnet delete ... --name "$_OLD_SUBNET"
  echo "==> Deleted orphaned delegated subnet '$_OLD_SUBNET'"
fi
```

Diagnose a conflict:
```bash
az network vnet subnet list \
  --resource-group "$RESOURCE_GROUP" --vnet-name "$VNET_NAME" \
  --output table
```

A subnet with `Purpose` empty and no matching PE is likely the stale VNet-integration subnet. Safe to delete if `az postgres flexible-server show ... --query network.delegatedSubnetResourceId` returns `null`.

## LFC_PASSWORD pre-seeding (stable across re-runs)

Capture the loaded secret values immediately after `load_db_secret`, **before** the save-secret section overwrites `user`/`password`:

```bash
load_db_secret "azure-postgres" || true
_LOADED_USER="${user:-}"      # save before section 6 overwrites
_LOADED_PASSWORD="${password:-}"
PG_ADMIN_PASSWORD="${dba__password:-${password:-${PG_ADMIN_PASSWORD:-}}}"
```

Then in section 6b, pre-seed using the captured values:

```bash
LFC_USER="${LFC_USER:-lfc_user}"
[[ "${_LOADED_USER:-}" == "$LFC_USER" ]] && LFC_PASSWORD="${_LOADED_PASSWORD:-}"
source pg-configure.sh   # generates LFC_PASSWORD only if still empty
```

## Firewall rules — local machine IP required

Unlike MySQL, PostgreSQL Flexible Server does not create an allow-all rule when public access is enabled. Add the local machine's IP explicitly so `pg-configure.sh` can connect from the setup host:

```bash
_MY_IP=$(curl --silent --fail --max-time 5 https://api.ipify.org 2>/dev/null || true)
[[ -n "$_MY_IP" ]] && _add_pg_ip_rule "$PG_SERVER_NAME" "$_MY_IP" "local-machine-$(date +%Y%m%d)"
```

## Private endpoint NIC — async provisioning

The PE NIC ID is provisioned asynchronously after PE create completes. Poll up to 2 minutes before querying the private IP:

```bash
PE_NIC_ID=""
for _i in {1..12}; do
  PE_NIC_ID=$(az network private-endpoint show \
    --resource-group "$RESOURCE_GROUP" --name "$PG_PE_NAME" \
    --query "networkInterfaces[0].id" -o tsv 2>/dev/null || true)
  [[ -n "$PE_NIC_ID" && "$PE_NIC_ID" != "None" ]] && break
  echo "    Waiting for PE NIC to be ready ... ($_i/12)"
  sleep 10
done
[[ -z "$PE_NIC_ID" || "$PE_NIC_ID" == "None" ]] && { echo "error: PE NIC timeout" >&2; kill -INT $$; }
```


