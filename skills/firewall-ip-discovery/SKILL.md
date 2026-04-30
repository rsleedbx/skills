---
name: firewall-ip-discovery
description: Discovering and managing IP addresses for database firewall rules — laptop IP vs serverless egress IP, public IP discovery services (ipify/ifconfig.me/icanhazip/checkip.amazonaws.com), split tunneling detection (when your VPN routes cloud API calls through a different IP than general internet), per-cloud firewall update commands (Azure flexible-server/NSG/SQL, AWS security groups, GCP authorized-networks), DB_FIREWALL_CIDRS pattern, serverless egress IP automation via Jobs API. Use when adding your IP to a database firewall, debugging connection failures after VPN change, setting up firewall rules for Databricks serverless, or understanding why your IP in the firewall still fails (split tunneling).
---

# Firewall IP Discovery

## Two separate IPs to whitelist

Any database accessed from both a local laptop and Databricks serverless requires **two separate firewall rules**:

| Source | IP | Rule name pattern |
|--------|-----|-------------------|
| Laptop / VPN | current public IP (`/32`) | `my-ip-YYYYMMDD` |
| Databricks serverless | egress NAT IP pool (`/24`) | `databricks-serverless-egress` |

The laptop IP changes when you switch VPN profiles or locations. The serverless IP rotates within a pool.

---

## Laptop IP discovery — bash

```bash
MY_IP=$(curl --silent --fail --location https://api.ipify.org)
: "${MY_IP:?could not determine current machine IP — check internet connectivity}"
```

Public IP services — fallback order:

| Service | URL |
|---------|-----|
| `api.ipify.org` | `https://api.ipify.org` — returns plain IPv4 |
| `ifconfig.me` | `https://ifconfig.me` |
| `icanhazip.com` | `http://ipv4.icanhazip.com` — used by `terraform_common.sh` |
| `checkip.amazonaws.com` | `https://checkip.amazonaws.com` — see split tunneling |

> **Split tunneling warning:** If on a corporate VPN, the IP you see from `ifconfig.me` may differ from the IP AWS/GCP sees for your API calls. Always verify. For detection code and the fix: see [split-tunneling.md](split-tunneling.md).

For the Python fallback chain and Terraform data source: see [split-tunneling.md](split-tunneling.md).

---

## Serverless egress IP

### Method 1 — Manual (quick, single observation)

Run from a `%sh` cell on Databricks serverless:
```bash
curl -s https://api.ipify.org
```

The returned IP is the NAT64 gateway's public IPv4. **Whitelist the entire `/24`** — serverless uses a rotating pool within the range. Example: observed `20.96.41.141` → whitelist `20.96.41.0/24`.

### Method 2 — Automated via Jobs API

Upload a notebook and run it as a serverless job from your laptop — the authoritative approach. Extracts `SERVERLESS_EGRESS_IP` and derives the `/24` range automatically.

For the full Jobs API script: see [serverless-egress-automation.md](serverless-egress-automation.md).

---

## Per-cloud firewall commands — quick reference

| Cloud | Resource | Command |
|-------|----------|---------|
| Azure | MySQL Flexible Server | `az mysql flexible-server firewall-rule create` |
| Azure | PostgreSQL Flexible Server | `az postgres flexible-server firewall-rule create` |
| Azure | SQL Server | `az sql server firewall-rule create` (idempotent — overwrites by name) |
| Azure | VM (NSG) | `az network nsg rule create` |
| AWS | RDS | `aws ec2 authorize-security-group-ingress` |
| GCP | Cloud SQL | `gcloud sql instances patch --authorized-networks` (replaces entire list — merge first) |

For full helper functions with idempotency checks: see [per-cloud-commands.md](per-cloud-commands.md).

---

## DB_FIREWALL_CIDRS pattern (mist/Terraform)

`DB_FIREWALL_CIDRS` is a space-separated list of CIDRs passed to setup scripts and Terraform:

```bash
export DB_FIREWALL_CIDRS="${MY_IP}/32 20.96.41.0/24"
```

Per-cloud Terraform variable names:

| Cloud | Terraform variable |
|-------|-------------------|
| AWS | `additional_cidrs` |
| Azure | `allowed_cidrs` |
| GCP | `authorized_networks` |

**CIDR correction:** Terraform sometimes receives a host address instead of a network address. Fix before passing to cloud APIs: `128.77.97.250/30` → `128.77.97.248/30`.

---

## Idempotency and rule naming

```bash
# Creates a new rule each day; old rules accumulate but don't cause errors
RULE_NAME="my-ip-$(date +%Y%m%d)"    # e.g. my-ip-20260428
```

Periodic cleanup: list rules matching `my-ip-*` and delete those older than N days.

---

## Why "add my IP" doesn't always work

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Rule added but connection still fails | Split tunneling — cloud sees a different IP | Detect with CloudTrail/GCP audit; add all IPs |
| Works from laptop, fails from serverless | Serverless egress IP not whitelisted | Add `/24` range or run `discover-egress-ip.sh` |
| Works via PNG, fails without PNG | PNG routing active; public path IP not whitelisted | Add serverless egress IP to public-path firewall |
| Azure Flexible Server: no VNet rules | `Microsoft.Sql` endpoint missing from Databricks subnets | Use IP rules only |
| IP rule added but rule shows empty IP | `${MY_IP}` adjacent to non-ASCII dash character | Use `${MY_IP}` with braces; see `shell-script-practices` |
