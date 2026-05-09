---
name: databricks-create-workspace
description: >-
  Create a Databricks workspace on Azure, AWS, or GCP — first-party vs third-party
  service model differences, why az databricks workspace create is required on Azure
  (not the Databricks Account API), required vars, RG quota handling, region
  Default Storage validation, managed resource group purpose, polling, and
  post-creation profile registration. Use when creating a Databricks workspace,
  debugging workspace creation failures (subscription type errors, RG quota,
  unsupported region), or understanding the Azure-first-party vs AWS/GCP-ISV model.
---

# Databricks Workspace Creation

## Cloud service model — critical context

**Azure**: Databricks is a **first-party Microsoft service** (Azure Databricks).
Sold, billed, and surfaced through the Azure portal as part of the native Azure catalog.
The workspace is an Azure ARM resource (`Microsoft.Databricks/workspaces`) in the customer's
Azure subscription. Creating it via the **Databricks Account API fails** for
internal/staging Databricks accounts with:
> `INVALID_PARAMETER_VALUE: Subscription ae7b6b87-... is not a third party subscription`

The Account API expects a customer-owned Azure subscription. Internal Databricks
engineering accounts are backed by Microsoft-owned subscriptions that the public API
rejects. Use `az databricks workspace create` instead — it creates the ARM resource
directly in any subscription you control via `az login`.

**AWS / GCP**: Databricks is a third-party ISV. Workspaces are created via the
Databricks Account API or the cloud provider's marketplace. The "not a third-party
subscription" error does not apply.

---

## Azure workspace creation — script

Reference: `scripts/databricks-0-create-workspace.sh`

### Required vars (all must be set before sourcing)

```bash
export ACCOUNT_PROFILE="ACCOUNT-{uuid}"          # Databricks account profile in ~/.databrickscfg
export NEW_WORKSPACE_NAME="robert-lee-png-eastus2"
export LOCATION="eastus2"                         # NOT eastus — see region note below
export SUBSCRIPTION_ID="596df088-..."             # Azure subscription you control via az login
export RESOURCE_GROUP="robert-lee-png-customer-rg" # Must pre-exist when sub is at 980 RG quota
```

Run:
```bash
. ./scripts/databricks-0-create-workspace.sh
```

The script checks all vars up-front, prints every missing one, and aborts via `return 1`
before running any az or databricks command.

### Resource group quota

Azure limits subscriptions to **980 resource groups**. Every workspace creation needs:
- 1 RG for the workspace ARM resource itself → reuse an existing RG to save a slot
- 1 RG for the managed resource group → must be net-new, Databricks-controlled

Set `RESOURCE_GROUP` to an existing RG. Set `NEW_WORKSPACE_NAME` to a name that hasn't
been used — the managed RG is derived as `{NEW_WORKSPACE_NAME}-managed`.

Check count:
```bash
az group list --subscription "${SUBSCRIPTION_ID}" --query "length(@)" -o tsv
```

### Managed resource group

Databricks creates and locks a second RG (`{workspace-name}-managed`) to hold all
workspace infrastructure: VNet, NSGs, storage accounts (DBFS root), cluster VMs
(ephemeral), load balancers, private endpoints. You name it; Databricks owns it.
It must not pre-exist.

### Region and Default Storage

Not every Azure region supports Databricks Default Storage (DBFS root).
`eastus` (East US) does **not** support it. `eastus2` (East US 2) does.
These are two distinct regions despite the same geography.

The script calls `GET /api/2.0/accounts/{id}/deployments/supported-regions` to validate
`LOCATION` before creating. If that endpoint is unavailable it falls back to a built-in
list. To bypass validation: `export LOCATION_OVERRIDE_ALLOW=1`.

Known-good regions: `eastus2 westus2 westeurope northeurope australiaeast
southeastasia japaneast uksouth canadacentral centralus brazilsouth`

### SKU

Always use `--sku premium`. Standard SKU lacks Unity Catalog and NCC/PNG support.

### Polling

`az databricks workspace show --query provisioningState` returns:
- `Updating` / `Creating` → still in progress
- `Succeeded` → workspace is ready
- `Failed` / `Canceled` → creation failed

The Databricks Account API uses different status values (`RUNNING`, `FAILED`, `BANNED`).

### Post-creation

After `provisioningState = Succeeded`:
1. Script writes `[WORKSPACE_PROFILE]` + `host = https://...` to `~/.databrickscfg`
2. Authenticate: `databricks auth login --host "${WORKSPACE_HOST}" --profile "${WORKSPACE_PROFILE}"`
3. Script adds the profile entry to `scripts/workspace-config.json`
4. Optionally attach an NCC: `scripts/databricks-2-setup-png.sh`

The workspace automatically joins the Databricks account for its Azure tenant — no
explicit account attachment step required.

---

## AWS workspace creation

Use the Databricks Account API (no az equivalent):
```bash
databricks -p "${ACCOUNT_PROFILE}" api post \
  "/api/2.0/accounts/${ACCOUNT_ID}/workspaces" \
  --json '{"workspace_name":"...","deployment_name":"...","aws_region":"us-east-1",
           "credentials_id":"...","storage_configuration_id":"..."}'
```
Poll `GET /api/2.0/accounts/{id}/workspaces/{id}` for `workspace_status = RUNNING`.

## GCP workspace creation

Same Account API pattern as AWS but with `gcp_managed_network_config` and
`cloud_resource_container` fields. GCP workspace URLs use numeric IDs
(see databricks-asset-bundle skill for GCP URL quirks).
