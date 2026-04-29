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

**Decoding NAT64 → IPv4:** last 32 bits of the address hold the IPv4:

```
64:ff9b:1:0:ac1f:e0c3: 0a0a : 0304
                        ^^^^^^^^^^^
                        0x0a.0x0a.0x03.0x04  →  10.10.3.4
```

Multiple NAT64 addresses for the same FQDN = different NAT64 gateway nodes (load balanced), all encoding the same private IP.

See `databricks-private-network-gateway` skill for the full Python `decode_nat64` implementation, routing rules, and PNG verification.

## PNG routing (summary)

PNG routing is **DNS-based** — it intercepts at DNS resolution time, not at the packet level. FQDNs in the destination list get NAT64-encoded private IPs; FQDNs not in the list and raw IPs bypass PNG. Full routing table and destination rules are in the `databricks-private-network-gateway` skill.

## NCC egress subnets (authoritative source)

The NCC egress config lists all worker subnets for the serverless compute pool — the authoritative source for understanding what infrastructure serverless traffic comes from:

```bash
databricks api get \
  "/api/2.0/accounts/${ACCOUNT_ID}/network-connectivity-configs/${NCC_ID}" \
  --profile "$ACCOUNT_PROFILE" \
  | jq '.egress_config.default_rules.azure_service_endpoint_rule.subnets'
```

**Why VNet rules don't work:** the subnets are in **Databricks-owned Azure subscriptions** (`7eaeac6a-...`, `85736b19-...`, etc.) that customers have no access to. You cannot read their address prefixes, add them as VNet rules, or peer with them. **Use IP-based firewall rules with the `/24` range instead.**

See `databricks-network-connectivity-configuration` skill for full NCC API patterns and egress IP firewall rules.

## Discovering the egress IP

Run from a `%sh` cell on serverless compute:

```bash
curl -s https://ifconfig.me
# e.g. 20.96.41.141
```

The returned IP is the NAT64 gateway's public IPv4 — what Azure firewall rules see for non-PNG traffic. Serverless uses a pool within `20.96.41.0/24`; whitelist the entire `/24` rather than a single IP to avoid chasing pool rotations.

See `databricks-network-connectivity-configuration` skill for the `az` firewall rule commands and egress subnet details.

## Verification notebook

See `notebooks/png-verify-nat64.ipynb` in azure_png_demo for a full NAT64 decode + port check notebook.
