---
name: databricks-cli-api
description: Prefer the Databricks CLI `databricks api` subcommands with `--profile` and ~/.databrickscfg for both workspace and account-level REST calls in scripts and automation. Use when replacing curl + DATABRICKS_HOST + DATABRICKS_TOKEN or curl + ACCOUNT_TOKEN, adding shell helpers that hit the workspace or account API, aligning shell tools with Terraform/CLI profile auth, Unity Catalog external storage / storage credentials / IAM trust by cloud, or configuring PNG (Private Network Gateway) destinations with correct private hostnames.
---

# Databricks CLI for workspace and account APIs

## Default approach

For **all** Databricks HTTP APIs — workspace-scoped (Unity Catalog, jobs, etc.) **and** account-level (NCC, PNG gateways, workspace list) — use the **Databricks CLI** instead of raw `curl` with manually obtained tokens.

- **Auth**: `host` and `token` (or OAuth where configured) live under a **`[profile]`** in `~/.databrickscfg`.
- **Profile**: pass **`-p` / `--profile`** on the CLI, or set **`DATABRICKS_PROFILE`** in the environment.
- **Do not** require callers to export `DATABRICKS_HOST` / `DATABRICKS_TOKEN`, and do not manually obtain `ACCOUNT_TOKEN` via `databricks auth token | jq .access_token` — the CLI manages auth and token refresh automatically.

## `databricks api` pattern

```bash
# Workspace-scoped API (profile host = workspace URL)
databricks -p "$WORKSPACE_PROFILE" api get /api/2.1/unity-catalog/metastore_summary

databricks -p "$WORKSPACE_PROFILE" api post /api/2.1/jobs/create --json @payload.json

# Account-level API (profile host = accounts.azuredatabricks.net)
# Path includes /api/2.0/accounts/${ACCOUNT_ID}/... — the CLI prepends the profile host.
databricks -p "$ACCOUNT_PROFILE" api get \
  "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}/private-network-gateways"

databricks -p "$ACCOUNT_PROFILE" api patch \
  "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}/private-network-gateways/${PNG_GATEWAY_ID}?update_mask=destinations" \
  --json "{\"destinations\": ${_MERGED_DESTS}}"
```

Subcommands: `get`, `post`, `put`, `patch`, `delete`, `head`. **PATH** is the URL path only (no host); the CLI resolves the host from the profile.

## Convenience alias for repeated account-level calls

When a script makes many account-level calls, define a one-liner alias to avoid repeating `databricks -p "$ACCOUNT_PROFILE" api`:

```bash
_acct() { databricks -p "$ACCOUNT_PROFILE" api "$@"; }

_acct get  "/api/2.0/accounts/${ACCOUNT_ID}/workspaces"
_acct post "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs" --json "$_BODY"
```

## When curl is still reasonable

- **CI** that intentionally injects short-lived tokens without a config file (document that exception).
- Endpoints outside `*.azuredatabricks.net` / `*.databricks.com` (e.g. Azure ARM, AWS APIs).

## Unity Catalog external tables / locations: who you trust (by cloud)

When wiring **external tables**, **external locations**, and **storage credentials** to customer cloud storage:

- **AWS:** The customer IAM role’s **trust policy** must allow **Databricks’ Unity Catalog master IAM role** (fixed **ARNs** published in **Databricks** docs for your **partition**, e.g. commercial vs GovCloud). There is a single documented UC master principal per partition, not “your workspace id.”
- **Azure / GCP:** You grant **your own** identity access in the storage layer (**Entra ID** service principal or **GCP** service account in ACLs / IAM), then **register that same identity** as the Unity Catalog **storage credential**. There is **no** single Databricks “master account id” to hard-code for those clouds the way there is for AWS UC master role trust.

## PNG (Private Network Gateway) destinations

PNG's NAT64 mechanism intercepts DNS queries for hostnames in the destination list. This only works correctly when the hostname resolves to a **private (RFC 1918) IP**.

### Rules

- **Only add hostnames that resolve to private IPs** as PNG destinations (Azure Flexible Server FQDNs via private endpoints, VM internal hostnames, custom private DNS zones)
- **Never add public FQDNs** or raw public IPs as PNG destinations
- For VMs with both a public FQDN and an internal hostname, use the **internal hostname** as the PNG destination

See `databricks-private-network-gateway` skill for the full PNG API, destination management, and verification patterns.

### SQL_DNS naming convention for PNG discovery scripts

Using a shared `SQL_DNS` risks the wrong value (e.g. a VM's public FQDN) being picked up by the PNG discovery loop. Export named per-engine vars (`SQL_DNS_MYSQL`, `SQL_DNS_PG`, `SQL_DNS_SQL`) pointing to the private/internal hostname only.

See `databricks-private-network-gateway` skill for the full naming convention and bash example.

### Verifying PNG is intercepting correctly

From a Databricks serverless notebook:
```bash
# Should return a 64:ff9b:1:0:.../3306 NAT64 address and "Connected"
curl -v --connect-timeout 5 telnet://<internal-hostname>:3306
# "Failure writing output to destination" = binary MySQL handshake = success
```

If `nslookup` returns the plain A record (not NAT64), the hostname is not in the PNG destination list or PNG has not yet reached ESTABLISHED state.

## Repo examples

| Location | Pattern |
|----------|---------|
| `scripts/uctblextstorage/external_managed_location_uc.sh` | **`source … <command>`** for **`discover`**, **`bucket`**, **`role`** (cloud-specific; **`UCTB_CLOUD`** from **`discover`**); **`./… help`** only for usage (see `shell-script-practices`). |
| `scripts/uctblextstorage/terraform/` | Terraform `provider "databricks" { profile = … }` — same config file, different tool |

## Prerequisites

Install and upgrade the **unified** Databricks CLI (`databricks` with `api` subcommands). Legacy `databricks-cli` (Python) differs; prefer the current CLI in docs.
