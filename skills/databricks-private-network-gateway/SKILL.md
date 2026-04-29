---
name: databricks-private-network-gateway
description: Databricks Private Network Gateway (PNG) — how PNG routes serverless traffic to private endpoints via NAT64/DNS64, how to create and update a PNG (destinations, subnet, traffic_mode, private_dns_resolvers), how to poll for ESTABLISHED state, destination rules (private hostnames only, never public FQDNs), SQL_DNS naming conventions for discovery scripts, verifying PNG is intercepting correctly. Use when setting up PNG, adding database hostnames to PNG destinations, debugging PNG routing, checking NAT64 DNS resolution, or verifying private endpoint connectivity from Databricks serverless.
---

# Databricks Private Network Gateway (PNG)

## What PNG does

PNG creates an encrypted tunnel from Databricks serverless compute into a **customer-owned Azure VNet**, enabling serverless notebooks and jobs to reach private endpoints (databases, storage, etc.) without traversing the public internet.

Routing is **DNS-based**: PNG intercepts DNS queries for hostnames in its destination list, wraps the resolved private IPv4 in a NAT64 address (`64:ff9b:1::/96`), and routes the connection through the private tunnel.

## How NAT64 works

When PNG routes a connection, DNS64 encodes the private endpoint IPv4 in the last 32 bits of a `64:ff9b:1::/96` IPv6 address:

```
nslookup robert-lee-png-mysql.mysql.database.azure.com
→ Address: 64:ff9b:1:0:ac1f:e0c3:0a0a:0304
                         ^^^^^^^^^^^^^^^^^^^
                         0x0a.0x0a.0x03.0x04  →  10.10.3.4  (private endpoint IP)
```

Decoding in Python:
```python
import ipaddress

def decode_nat64(addr: str) -> str | None:
    NAT64_PREFIX = ipaddress.IPv6Network("64:ff9b:1::/96")
    try:
        v6 = ipaddress.IPv6Address(addr)
    except ValueError:
        return None
    if v6 not in NAT64_PREFIX:
        return None
    return str(ipaddress.IPv4Address(int(v6) & 0xFFFFFFFF))
```

Multiple NAT64 addresses for the same FQDN = different NAT64 gateway nodes (load balanced), all encoding the same private IP.

## PNG routing rules

| Connection type | Result |
|----------------|--------|
| FQDN in PNG destinations list | Routed through PNG tunnel → private endpoint |
| FQDN not in PNG destinations list | Public internet via NAT64 |
| Raw public IPv4 (no DNS resolution) | Public internet via NAT64 (PNG bypassed) |

PNG routing only activates during **DNS resolution**. If the client connects using a raw IP, no interception occurs.

## Destination rules — private hostnames only

PNG only works correctly when the hostnames in the destination list resolve to **private (RFC 1918) IPs**:

- **Add:** Azure Flexible Server FQDNs via private endpoints (e.g. `*.mysql.database.azure.com`)
- **Add:** Azure VM internal hostnames (e.g. `*.internal.cloudapp.net`)
- **Add:** Custom private DNS zone names linked to the customer VNet
- **Never add:** Public FQDNs (e.g. `*.eastus2.cloudapp.azure.com`)
- **Never add:** Raw public IPs

If a hostname resolves to a public IP, PNG will route that traffic over the private tunnel to a public address — this silently fails or produces unexpected results.

For VMs with both a public FQDN (local CLI access) and an internal hostname (PNG routing), use the **internal hostname** as the PNG destination.

## SQL_DNS naming convention for discovery scripts

When a session sources multiple DB setup scripts, each exports its own named variable for the PNG-routable hostname. Using a shared `SQL_DNS` risks the public FQDN being picked up.

```bash
# ✗ Bad — SQL_DNS may be the public FQDN from a VM setup script
for _extra in ${SQL_DNS:-} ${SQL_DNS_MYSQL:-} ...; do _add_dns "$_extra"; done

# ✓ Good — explicit named vars, each set to the private/internal hostname only
export SQL_DNS_MYSQL="$VM_INTERNAL_DNS"   # *.internal.cloudapp.net
export SQL_DNS_PG="$VM_INTERNAL_DNS"
export SQL_DNS_SQL="$VM_INTERNAL_DNS"
for _extra in ${SQL_DNS_MYSQL:-} ${SQL_DNS_PG:-} ${SQL_DNS_SQL:-}; do
  _add_dns "$_extra"
done
```

`SQL_DNS` (unqualified) is intentionally excluded from PNG discovery — VM setup scripts set it to the public FQDN.

## API: create a PNG gateway

All PNG APIs are account-level, under an NCC (see `databricks-network-connectivity-configuration` skill for NCC lookup/creation).

```bash
_acct() { databricks -p "$ACCOUNT_PROFILE" api "$@"; }
_PNG_BASE="/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}/private-network-gateways"

_PNG_BODY=$(jq -nc \
  --arg      name    "$GATEWAY_NAME" \
  --arg      subnet  "$CUSTOMER_SUBNET_ID" \
  --argjson  dests   "$_DEST_JSON" \
  '{
    gateway_name:  $name,
    traffic_mode:  "SPECIFIC_DESTINATIONS",
    azure_cloud_connection: {
      gateway_subnet: {resource_id: $subnet}
    },
    destinations: $dests,
    private_dns_resolvers: [{resolver_type: "IP_ADDRESS", value: "168.63.129.16"}]
  }')

PNG_RESPONSE=$(_acct post "$_PNG_BASE" --json "$_PNG_BODY")
PNG_GATEWAY_ID=$(echo "$PNG_RESPONSE" | jq -r '.gateway_id // empty')
```

Key fields:
| Field | Value | Notes |
|-------|-------|-------|
| `traffic_mode` | `SPECIFIC_DESTINATIONS` | only route listed hostnames through PNG |
| `azure_cloud_connection.gateway_subnet.resource_id` | customer subnet resource ID | the VNet subnet PNG injects into |
| `private_dns_resolvers` | `168.63.129.16` | Azure's magic IP — resolves private DNS zones within the VNet |
| `destinations` | array of `{destination_type, value}` | DNS_NAME entries; see destination rules above |

## API: update PNG destinations (PATCH)

`update_mask=destinations` is **required** in the URL — otherwise the API ignores the destinations field:

```bash
_acct patch "${_PNG_BASE}/${PNG_GATEWAY_ID}?update_mask=destinations" \
  --json "{\"destinations\": ${_MERGED_DESTS}}"
```

Merge new destinations with existing to avoid overwriting:
```bash
_EXISTING_DESTS=$(_acct get "${_PNG_BASE}/${PNG_GATEWAY_ID}" | jq -c '.destinations // []')
_MERGED_DESTS=$(printf '%s\n' "$_EXISTING_DESTS" "$_NEW_DEST_JSON" \
  | jq -sc 'add | unique_by(.value)')
```

## API: list PNG gateways

```bash
_acct get "$_PNG_BASE"
# Response key: .private_network_gateways (array) or .items (fallback)
_GATEWAYS=$(echo "$_PNG_LIST" | jq -c '(.private_network_gateways // .items // [])[]')

# Match by name
PNG_GATEWAY_ID=$(echo "$_GATEWAYS" | jq -r --arg name "$GATEWAY_NAME" \
  'select(.gateway_name==$name) | .gateway_id // empty' | head -1)
```

## Polling for ESTABLISHED state

PNG provisioning takes up to ~5 minutes (cold start). Poll until `state == "ESTABLISHED"` before using:

```bash
for i in $(seq 1 60); do
  STATE=$(_acct get "${_PNG_BASE}/${PNG_GATEWAY_ID}" | jq -r '.state // empty')
  echo "  [${i}/60] state: ${STATE:-(unknown)}"
  [[ "$STATE" == "ESTABLISHED" ]] && break
  [[ "$STATE" == "FAILED" ]] && { echo "error: PNG reached FAILED state" >&2; kill -INT $$; }
  sleep 10
done
[[ "$STATE" != "ESTABLISHED" ]] && { echo "error: timeout — state=$STATE" >&2; kill -INT $$; }
```

States: `PROVISIONING` → `ESTABLISHED` (normal) or `FAILED` (subnet capacity / VNet config issue).

## Build the destinations JSON array

From an array of FQDNs:
```bash
_DEST_JSON=$(printf '%s\n' "${_dns_seen[@]}" \
  | jq -Rc '{"destination_type":"DNS_NAME","value":.}' | jq -sc '.')
```

## Verifying PNG is intercepting correctly

From a Databricks serverless notebook (`%sh` cell):

```bash
# 1. DNS check — should return 64:ff9b:1::.../96 address (not a plain A record)
nslookup <internal-hostname>
# Expected: Address: 64:ff9b:1:0:...

# 2. Connectivity check — curl telnet in NAT64 environment
curl -v --connect-timeout 5 telnet://<internal-hostname>:3306
# "Connected to ... (64:ff9b:1:...)" = PNG routing active
# "Failure writing output to destination" = binary DB handshake = port open, success
# "Could not resolve host" = hostname not in PNG destinations
# Plain A record in nslookup = hostname not in destinations (or PNG not ESTABLISHED)
```

If `nslookup` returns a plain A record (not NAT64), the hostname is either not in the PNG destination list or the gateway hasn't reached ESTABLISHED state.

## Idempotent setup pattern (find-or-create)

```bash
# 1. Try to find by name
PNG_GATEWAY_ID=$(echo "$_GATEWAYS" | jq -r --arg name "$GATEWAY_NAME" \
  'select(.gateway_name==$name) | .gateway_id // empty' | head -1)

# 2. Fallback: find by matching destination
if [[ -z "$PNG_GATEWAY_ID" ]]; then
  PNG_GATEWAY_ID=$(echo "$_GATEWAYS" | jq -r --arg dns "${_dns_seen[0]}" \
    'select((.destinations[]?.value == $dns) or (.dns_names[]? == $dns)) | .gateway_id // empty' | head -1)
fi

# 3. Create if still not found
if [[ -z "$PNG_GATEWAY_ID" ]]; then
  # ... POST as above
fi

# 4. PATCH if found but destinations are stale
```

## Save state to Databricks secret scope

After setup, save the NCC and PNG IDs so other scripts can look them up:

```bash
save_db_secret "png-gateway" NCC_ID PNG_GATEWAY_ID
```

Load back:
```bash
load_db_secret "png-gateway"
# exports $NCC_ID and $PNG_GATEWAY_ID
```
