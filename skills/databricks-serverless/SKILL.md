---
name: databricks-serverless
description: Quirks and gotchas of Databricks serverless compute — IPv6-only networking, NAT64 DNS, PNG routing, and port-checking tools. Use when writing code or shell commands that run on Databricks serverless compute, debugging connectivity from serverless notebooks, or working with PNG (Private Network Gateway) DNS resolution.
---

# Databricks Serverless Compute

## IPv6-only networking

Serverless compute nodes have no IPv4 address. All traffic uses IPv6 with NAT64 translation.

**Consequences:**
- `nc -6` flag is NOT available — traditional `netcat` is installed, not `netcat-openbsd`
- `nc hostname port` fails with `No address associated with name` when DNS returns only AAAA records
- Direct IPv4 connections still work via the OS NAT64 layer, but PNG routing is bypassed

## Port reachability checks

Do NOT use `nc`. Use one of these instead:

```bash
# Option 1: bash /dev/tcp (preferred — no external dependency)
timeout 5 bash -c 'echo >/dev/tcp/<host>/<port>' 2>/dev/null \
  && echo "open" || echo "closed"

# Option 2: curl telnet mode (works; expect "Binary output" warning on DB ports — that's normal)
curl -v --connect-timeout 5 telnet://<host>:<port>
```

Multi-host check pattern:
```bash
for entry in "host1.example.com 3306" "host2.example.com 5432"; do
  host="${entry% *}"; port="${entry#* }"
  timeout 5 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null \
    && echo "OPEN   $host:$port" || echo "CLOSED $host:$port"
done
```

## NAT64 DNS (PNG routing)

When PNG routes a connection, DNS64 wraps the private endpoint IPv4 inside a NAT64 address (`64:ff9b:1::/96` prefix). `nslookup` works normally.

**Decoding NAT64 → IPv4:** last 32 bits of the address hold the IPv4.

```
64:ff9b:1:0:ac1f:e0c3: 0a0a : 0304
                        ^^^^^^^^^^^
                        0x0a.0x0a.0x03.0x04  →  10.10.3.4
```

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

| Connection | Result |
|---|---|
| FQDN in PNG destinations | Routed through PNG tunnel → private endpoint |
| FQDN not in PNG destinations | Public internet via NAT64 |
| Raw public IPv4 (no DNS) | Public internet via NAT64 (PNG bypassed) |

PNG routing is **DNS-based** — it intercepts at DNS resolution time, not at the packet level.

## NCC egress subnets (authoritative source)

The NCC egress config lists all worker subnets for the serverless compute pool — the authoritative source for understanding what infrastructure serverless traffic comes from:

```bash
databricks api get \
  "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}" \
  --profile "$ACCOUNT_PROFILE" \
  | jq '.egress_config.default_rules.azure_service_endpoint_rule.subnets'
```

Example output (staging eastus2 — 11 pools across multiple subscriptions):
```json
[
  ".../staging-eastus2-snp-2-compute-5/subnets/worker-subnet",
  ".../staging-azure-eastus2-nephos10/subnets/worker-subnet",
  ...
]
```

**Why VNet rules don't work:** the subnets are in **Databricks-owned Azure subscriptions** (`7eaeac6a-...`, `85736b19-...`, etc.) that customers have no access to. You cannot read their address prefixes, add them as VNet rules, or peer with them. The NCC API only gives you the resource ID strings — not actionable for firewall configuration. **Use IP-based firewall rules with the `/24` range instead.**

## Discovering the egress IP

Run from a `%sh` cell on serverless compute:

```bash
curl -s https://ifconfig.me
# e.g. 20.96.41.141
```

The returned IP is the NAT64 gateway's public IPv4 — what Azure firewall rules see for non-PNG traffic. Serverless uses a pool within `20.96.41.0/24`; whitelist the entire `/24` rather than a single IP to avoid chasing pool rotations.

To whitelist the `/24` in Azure firewall rules (run from your laptop, not serverless):

```bash
# MySQL
az mysql flexible-server firewall-rule create \
  --resource-group "$RESOURCE_GROUP" --name "$MYSQL_SERVER_NAME" \
  --rule-name "databricks-serverless-egress-slash24" \
  --start-ip-address 20.96.41.0 --end-ip-address 20.96.41.255

# PostgreSQL
az postgres flexible-server firewall-rule create \
  --resource-group "$RESOURCE_GROUP" --name "$PG_SERVER_NAME" \
  --rule-name "databricks-serverless-egress-slash24" \
  --start-ip-address 20.96.41.0 --end-ip-address 20.96.41.255

# SQL Server
az sql server firewall-rule create \
  --resource-group "$RESOURCE_GROUP" --server "$SQL_SERVER_NAME" \
  --name "databricks-serverless-egress-slash24" \
  --start-ip-address 20.96.41.0 --end-ip-address 20.96.41.255
```

The setup scripts (`customer-3-setup-azure-*.sh`) use `SERVERLESS_EGRESS_IP_START` / `SERVERLESS_EGRESS_IP_END` variables (defaulting to the full `/24`) and pass them as a range to the `_add_*_ip_rule` helpers.

## Verification notebook

See `notebooks/png-verify-nat64.ipynb` in azure_png_demo for a full NAT64 decode + port check notebook.
