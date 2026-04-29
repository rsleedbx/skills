---
name: databricks-network-connectivity-configuration
description: Databricks Network Connectivity Configuration (NCC) — what an NCC is, how to look up the NCC assigned to a workspace, how to create one, how to read the egress config (subnets), why VNet rules don't work for serverless, and how to discover the serverless egress IP. Use when working with NCC objects in the Databricks account API, looking up egress subnets, configuring firewall rules for serverless traffic, or understanding the relationship between an NCC and a Private Network Gateway (PNG).
---

# Databricks Network Connectivity Configuration (NCC)

## What an NCC is

An NCC is an account-level object that defines the network connectivity settings for Databricks **serverless** compute. Each workspace is assigned at most one NCC. The NCC:

- Contains the **egress config** — the set of Azure subnets from which serverless workers originate traffic
- Is the parent container for **Private Network Gateways (PNG)** — each PNG is created under an NCC

> **NCC is serverless-only.** Classic clusters (Job Clusters, All-Purpose Clusters) use a separate networking model — either Databricks-managed VNet or customer VNet injection. They are not covered by the NCC or PNG, and their egress IPs come from a different source. See `firewall-ip-discovery` skill for how to discover classic cluster egress IPs.

## Two ways to discover the serverless egress IP

| Method | When to use | How |
|--------|------------|-----|
| NCC API (subnets) | Understand infrastructure topology | `jq '.egress_config...subnets'` — returns subnet resource IDs, **not IP ranges** |
| Run a job | Get actual observed public IP for firewall rules | `discover-egress-ip.sh` — submits a notebook as a serverless job, reads `api.ipify.org` output |

The NCC API gives you subnet resource IDs in Databricks-owned subscriptions. You **cannot** convert those to IP ranges. For firewall rules, always use the job-based approach to observe the actual IP.

See `firewall-ip-discovery` skill for the full automated pattern (`discover-egress-ip.sh`), including how to derive `SERVERLESS_EGRESS_IP_START`/`END` and whitelist the `/24` range.

## Resolve the NCC assigned to a workspace

Look up the workspace list at the account level and match by workspace URL:

```bash
_acct() { databricks -p "$ACCOUNT_PROFILE" api "$@"; }

NCC_ID=$(_acct get "/api/2.0/accounts/${ACCOUNT_ID}/workspaces" \
  | jq -r --arg host "$WORKSPACE_HOST" '
      .[] | select(
        .workspace_url == $host or
        .workspace_url == ($host | ltrimstr("https://")) or
        ("https://" + .deployment_name + ".azuredatabricks.net")         == $host or
        ("https://" + .deployment_name + ".staging.azuredatabricks.net") == $host
      ) | .network_connectivity_config_id // empty' \
  | head -1)
```

> Resolve `$WORKSPACE_HOST` with:
> ```bash
> WORKSPACE_HOST=$(databricks auth describe --profile "$WORKSPACE_PROFILE" --output json | jq -r '.details.host // empty')
> ```

## Create an NCC

If the workspace has no NCC assigned, create one:

```bash
NCC_RESPONSE=$(_acct post "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs" \
  --json "{\"name\": \"${NCC_NAME}\", \"region\": \"${LOCATION}\"}")
NCC_ID=$(echo "$NCC_RESPONSE" | jq -r '.network_connectivity_config_id // empty')
```

Naming convention: `{user_prefix}-png-ncc` (e.g. `robert-lee-png-ncc`).

## Read the egress config (subnets)

```bash
_acct get "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}" \
  | jq '.egress_config.default_rules.azure_service_endpoint_rule.subnets'
```

Returns an array of Azure subnet resource IDs — the subnets that serverless workers use for outbound traffic:

```json
[
  "/subscriptions/7eaeac6a-.../resourceGroups/staging-eastus2-snp-2-compute-5/providers/Microsoft.Network/virtualNetworks/.../subnets/worker-subnet",
  "/subscriptions/85736b19-.../resourceGroups/staging-azure-eastus2-nephos10/subnets/worker-subnet",
  ...
]
```

> These IDs tell you **which subnets** serverless workers live in, but **not their IP ranges**. The subnets are in Databricks-owned subscriptions — you cannot read their CIDRs. This API call is useful for topology understanding, not for constructing firewall rules.

## Why VNet rules don't work for serverless firewall access

The subnets are in **Databricks-owned Azure subscriptions** (`7eaeac6a-...`, `85736b19-...`, etc.). You cannot:
- Read their IP address prefixes (no access to those subscriptions)
- Add them as VNet service endpoint rules (they're not in your subscription)
- Peer with them

The NCC API returns only resource ID strings, not CIDR ranges. They are **not actionable** for database firewall configuration.

**Use IP-based firewall rules instead** — discover the actual public egress IP by running a job on serverless compute. See `firewall-ip-discovery` skill for the full pattern.

## Discovering the serverless egress IP

Quick check — run from a `%sh` cell on serverless compute:

```bash
curl -s https://ifconfig.me   # or https://api.ipify.org
# e.g. 20.96.41.141
```

This is the NAT64 gateway's **public IPv4**. Whitelist the entire `/24` — serverless uses a rotating pool within that range. For the fully automated approach (running as a job from your laptop, exporting `SERVERLESS_EGRESS_IP_START`/`END`, deriving the `/24`), see the `firewall-ip-discovery` skill (`discover-egress-ip.sh` pattern).

## NCC vs PNG

| Object | Level | Purpose |
|--------|-------|---------|
| NCC | Account | Container for egress config + PNG gateways |
| PNG | Under NCC | Tunnel from serverless to private endpoints in customer VNet |

A workspace has one NCC; an NCC can have multiple PNG gateways (e.g. one per region/VNet).

See `databricks-private-network-gateway` skill for PNG creation and destination management.
