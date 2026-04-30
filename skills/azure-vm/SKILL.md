---
name: azure-vm
description: Azure VM provisioning for database hosting and testing — Ubuntu VM creation, static public IP, NSG rules (VirtualNetwork + serverless egress + local IP), Azure Policy NSG-at-NIC enforcement before public IP association, dual DNS exports (public FQDN vs internal hostname), PNG routing rules for VM-hosted databases. Use when creating Azure VMs for database hosting, adding public IPs to existing VMs, configuring NSG rules, or setting up PNG routing for VM databases.
---

# Azure VM — LFC Setup

## Required variables (from prior sourced scripts)

Before sourcing `customer-vm-base.sh`, the calling script must set:

| Variable | Source | Description |
|----------|--------|-------------|
| `VM_NAME` | caller | unique VM name (e.g. `${USER_PREFIX}-mysql-vm`) |
| `DB_PORT` | caller | database TCP port (3306 / 5432 / 1433) |
| `USER_PREFIX` | `customer-1-az-login.sh` | user-specific resource prefix |
| `RESOURCE_GROUP` | `customer-2-az-network.sh` | Azure resource group |
| `VNET_NAME` | `customer-2-az-network.sh` | VNet name |
| `SUBNET_NAME` | `customer-2-az-network.sh` | subnet name for the VM |
| `NSG_NAME` | `customer-2-az-network.sh` | NSG shared by all VMs |
| `LOCATION` | `customer-2-az-network.sh` | Azure region |

## VM creation

- Image: **Ubuntu 22.04** (`Ubuntu2204`), size `Standard_B2s`.
- Admin user: `azureuser`; password generated at creation time and printed once.
- Create with `--nsg ""` (no NSG at create-time) — NSG is attached at NIC level separately.
- VM creation is idempotent: `az vm show &>/dev/null` guards the `az vm create` call.

```bash
if az vm show --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" &>/dev/null; then
  echo "==> VM '$VM_NAME' already exists — skipping"
else
  VM_ADMIN_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
  az vm create \
    --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" --location "$LOCATION" \
    --image Ubuntu2204 --size Standard_B2s \
    --admin-username azureuser --admin-password "$VM_ADMIN_PASSWORD" \
    --vnet-name "$VNET_NAME" --subnet "$SUBNET_NAME" \
    --public-ip-address "$VM_PUBLIC_IP_NAME" \
    --nsg ""
fi
```

## Static public IP — create and set DNS label

Create the public IP **before** the VM (separate from `az vm create`) so the IP name is controlled:

```bash
az network public-ip create \
  --resource-group "$RESOURCE_GROUP" --name "${VM_NAME}-pip" \
  --location "$LOCATION" --sku Standard --allocation-method Static \
  --dns-name "$VM_NAME" --output none
```

- `--dns-name "$VM_NAME"` sets the DNS label so the FQDN is `<vm-name>.<region>.cloudapp.azure.com`.
- The FQDN resolves to the static public IP from anywhere on the internet.
- For existing public IPs without a DNS label, set it idempotently: `az network public-ip update --dns-name "$VM_NAME"`.

## Azure Policy: NSG must be attached to NIC before associating a public IP

Enterprise subscriptions enforce that a NIC must have an NSG attached **at the NIC level** (not just at the subnet level) before a public IP can be associated. `az network nic ip-config update` exits 0 even when the association is blocked by `RequestDisallowedByPolicy`.

**Order of operations:**
1. Attach NSG to NIC (`az network nic update --network-security-group`)
2. Associate public IP (`az network nic ip-config update --public-ip-address`)
3. **Verify** — re-query `az network public-ip show --query ipConfiguration.id`; exit 0 alone is not sufficient

```bash
# Check whether the public IP is already associated (more reliable than NIC query)
_EXISTING_PIP_NIC=$(az network public-ip show \
  --resource-group "$RESOURCE_GROUP" --name "${VM_NAME}-pip" \
  --query "ipConfiguration.id" -o tsv 2>/dev/null || echo "")

if [[ -z "${_EXISTING_PIP_NIC:-}" || "${_EXISTING_PIP_NIC:-}" == "None" ]]; then
  # Discover NIC and ip-config names dynamically — NOT always "ipconfig1"
  _VM_NIC_ID=$(az vm show --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" \
    --query "networkProfile.networkInterfaces[0].id" -o tsv)
  _VM_NIC="${_VM_NIC_ID##*/}"   # basename without xargs
  _VM_NIC_IPCONFIG=$(az network nic ip-config list \
    --resource-group "$RESOURCE_GROUP" --nic-name "$_VM_NIC" \
    --query "[0].name" -o tsv)
  _PIP_ID=$(az network public-ip show \
    --resource-group "$RESOURCE_GROUP" --name "${VM_NAME}-pip" \
    --query "id" -o tsv)

  # Step 1: NSG at NIC level (required by Azure Policy)
  az network nic update \
    --resource-group "$RESOURCE_GROUP" --name "$_VM_NIC" \
    --network-security-group "$NSG_NAME" --output none \
    || { echo "error: failed to attach NSG to NIC" >&2; kill -INT $$; }

  # Step 2: Associate public IP
  az network nic ip-config update \
    --resource-group "$RESOURCE_GROUP" --nic-name "$_VM_NIC" \
    --name "$_VM_NIC_IPCONFIG" --public-ip-address "$_PIP_ID" --output none

  # Step 3: Verify — exit 0 is not sufficient
  _VERIFY=$(az network public-ip show \
    --resource-group "$RESOURCE_GROUP" --name "${VM_NAME}-pip" \
    --query "ipConfiguration.id" -o tsv 2>/dev/null || echo "")
  if [[ -z "${_VERIFY:-}" || "${_VERIFY:-}" == "None" ]]; then
    echo "error: public IP still not associated after update" >&2; kill -INT $$
  fi
fi
```

## NSG rules — three required ingress rules per VM

| Rule | Source | Priority formula | Purpose |
|------|--------|-----------------|---------|
| `Allow-{port}-Inbound` | `VirtualNetwork` | `1000 + (port % 1000)` | PNG routing via private IP |
| `Allow-{port}-Serverless-Inbound` | `<egress-cidr>/24` | `1100 + (port % 1000)` | Serverless egress without PNG (bypass test) |
| `Allow-{port}-My-IP` | `<local-ip>/32` | `800 + (port % 100)` | Local CLI (`mysqlcli`, `pgcli`, `sqlcmd`) |

Priority formula avoids conflicts when MySQL, Postgres, and SQL Server share the same NSG:
- MySQL 3306 → 1306 / 1406 / 806
- Postgres 5432 → 1432 / 1532 / 832
- SQL Server 1433 → 1433 / 1533 / 833

```bash
# VirtualNetwork rule (PNG routing)
az network nsg rule create \
  --resource-group "$RESOURCE_GROUP" --nsg-name "$NSG_NAME" \
  --name "Allow-${DB_PORT}-Inbound" \
  --priority $(( 1000 + DB_PORT % 1000 )) \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes "VirtualNetwork" --source-port-ranges "*" \
  --destination-address-prefixes "*" --destination-port-ranges "$DB_PORT" --output none
```

Use `az network nsg rule show &>/dev/null` to guard idempotency before creating each rule.

## Dual DNS exports — public FQDN vs internal hostname

After provisioning, every VM DB script exports two distinct names:

| Export | Value | Used for |
|--------|-------|----------|
| `SQL_DNS` | `<vm>.eastus2.cloudapp.azure.com` | Secret `host_fqdn` field, local CLI, `-public` UC connection |
| `VM_INTERNAL_DNS` | `<vm>.internal.cloudapp.net` | PNG UC connection, `SQL_DNS_*` for PNG discovery |

```bash
export SQL_DNS="${VM_PUBLIC_FQDN}"            # public FQDN — internet-resolvable
export VM_INTERNAL_DNS="${VM_NAME}.internal.cloudapp.net"   # private NIC IP via 168.63.129.16
```

**Why two names?**
- `VM_INTERNAL_DNS` resolves to the VM's RFC1918 NIC IP inside Azure DNS (168.63.129.16).
- PNG intercepts hostnames that resolve to private IPs via NAT64 — `VM_INTERNAL_DNS` is intercepted correctly.
- `SQL_DNS` (public FQDN) resolves to the static public IP — PNG's NAT64 cannot route public IPs over the private tunnel.

## PNG destination discovery — use SQL_DNS_* not SQL_DNS

Set `SQL_DNS_MYSQL` / `SQL_DNS_PG` / `SQL_DNS_SQL` to `VM_INTERNAL_DNS`, not to `SQL_DNS`.
The `databricks-2-setup-png.sh` discovery loop reads these named variables and adds them to PNG destinations.

```bash
export SQL_DNS_MYSQL="$VM_INTERNAL_DNS"   # customer-3-setup-vm-mysql.sh
export SQL_DNS_PG="$VM_INTERNAL_DNS"      # customer-3-setup-vm-postgres.sh
export SQL_DNS_SQL="$VM_INTERNAL_DNS"     # customer-3-setup-vm-sqlserver.sh
```

**Never** export `SQL_DNS` (public FQDN) for PNG discovery — adding a public FQDN to PNG destinations causes NAT64 to attempt routing over the private tunnel to a public IP, which fails.

## UC connections — two per VM database

Create two Unity Catalog connections per VM database:

| Connection name | Host | Purpose |
|-----------------|------|---------|
| `{prefix}-vm-{db}-public` | `VM_PUBLIC_IP` (or `SQL_DNS` as fallback) | Bypasses PNG — baseline test |
| `{prefix}-vm-{db}` | `VM_INTERNAL_DNS` | Routes via PNG — production path |

Both are created/updated via `uc-connection-helpers.sh`:
```bash
source "$(dirname "${BASH_SOURCE[0]}")/uc-connection-helpers.sh"
_uc_connection_upsert "${USER_PREFIX}-vm-mysql-public" \
  "$(_build_uc_conn_json "…-public" MYSQL vm-mysql "${VM_PUBLIC_IP:-$SQL_DNS}" "$DB_PORT")"
_uc_connection_upsert "${USER_PREFIX}-vm-mysql" \
  "$(_build_uc_conn_json "…" MYSQL vm-mysql "$VM_INTERNAL_DNS" "$DB_PORT")"
```

## Secret fields for VM databases

VM database secrets include extra fields vs Flexible Server secrets:

| Field | Value | Purpose |
|-------|-------|---------|
| `host_fqdn` | `VM_INTERNAL_DNS` | PNG routing (private) |
| `vm_fqdn_public` | `SQL_DNS` | Local CLI host (`sqlcli-helper.sh` uses this when `access_type=vm`) |
| `access_type` | `vm` | Signals `sqlcli-helper.sh` to use `vm_fqdn_public` |
| `vm_name` | `VM_NAME` | For `az vm run-command` operations |
| `vm_admin_user` | `azureuser` | VM SSH/RDP admin |
| `vm_admin_password` | generated | VM admin credential |
