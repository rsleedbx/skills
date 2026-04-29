---
name: firewall-ip-discovery
description: Discovering and managing IP addresses for database firewall rules — laptop IP vs serverless egress IP, public IP discovery services (ipify/ifconfig.me/icanhazip/checkip.amazonaws.com), split tunneling detection (when your VPN routes cloud API calls through a different IP than general internet), per-cloud firewall update commands (Azure flexible-server/NSG/SQL, AWS security groups, GCP authorized-networks), DB_FIREWALL_CIDRS pattern, serverless egress IP automation via Jobs API. Use when adding your IP to a database firewall, debugging connection failures after VPN change, setting up firewall rules for Databricks serverless, or understanding why your IP in the firewall still fails (split tunneling).
---

# Firewall IP Discovery

## Two separate IPs to whitelist

Any database accessed both from a local laptop and from Databricks serverless requires **two separate firewall rules**:

| Source | IP | Rule name pattern |
|--------|-----|-------------------|
| Laptop / VPN | current public IP (`/32`) | `my-ip-YYYYMMDD` |
| Databricks serverless | egress NAT IP pool (`/24`) | `databricks-serverless-egress` |

These are different IPs. The laptop IP changes when you switch VPN profiles or locations. The serverless IP rotates within a pool.

---

## Part 1: Laptop IP discovery

### Public IP discovery services — fallback order

| Service | URL | Notes |
|---------|-----|-------|
| `api.ipify.org` | `https://api.ipify.org` | Returns plain IPv4, no newline issues |
| `ifconfig.me` | `https://ifconfig.me` | Popular, reliable |
| `icanhazip.com` | `http://ipv4.icanhazip.com` | IPv4-specific URL; used by `terraform_common.sh` |
| `checkip.amazonaws.com` | `https://checkip.amazonaws.com` | AWS-hosted; see split tunneling section |

**Bash — with hard-fail on empty:**
```bash
MY_IP=$(curl --silent --fail --location https://api.ipify.org)
: "${MY_IP:?could not determine current machine IP — check internet connectivity}"
```

**Python — with fallback chain (mist `ip_checker.py` pattern):**
```python
import subprocess

def get_current_public_ip():
    services = [
        ['curl', '-s', '--max-time', '3', 'ifconfig.me'],
        ['curl', '-s', '--max-time', '3', 'api.ipify.org'],
        ['curl', '-s', '--max-time', '3', 'icanhazip.com'],
    ]
    for cmd in services:
        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=5)
            ip = result.stdout.strip()
            # Basic validation: 4-octet, max 15 chars
            if ip and '.' in ip and len(ip) <= 15:
                return ip
        except (subprocess.TimeoutExpired, FileNotFoundError):
            continue
    return None
```

**Terraform — data source:**
```hcl
data "http" "current_ip" {
  url = "http://ipv4.icanhazip.com"
}
locals {
  current_ip_cidr = "${chomp(data.http.current_ip.response_body)}/32"
}
```

---

## Part 2: Split tunneling — the hidden firewall problem

### What split tunneling is

Corporate VPNs sometimes route cloud API traffic (to `*.amazonaws.com`, `*.googleapis.com`) through the corporate network, while general internet traffic (to `ifconfig.me`) exits directly. This means **the IP that AWS/GCP sees for your API calls is different from the IP you discover with `curl ifconfig.me`** — your general-purpose firewall whitelist is wrong.

### Detection: multi-probe approach

Probe from multiple angles and compare. If you see more than one unique IP, split tunneling is active:

```python
def detect_split_tunneling():
    ips = {}

    # Source 1: General internet IP (may exit via VPN)
    ips['general'] = run_curl('ifconfig.me')

    # Source 2: AWS-hosted IP probe (routes via VPN for AWS traffic if split-tunneled)
    ips['aws_checkip'] = run_curl('checkip.amazonaws.com')

    # Source 3: What AWS actually receives — definitive via CloudTrail
    ips['aws_api'] = get_ip_from_aws_cloudtrail()

    # Source 4: What GCP actually receives — definitive via audit log
    ips['gcp_api'] = get_ip_from_gcp_audit()

    unique_ips = {v for v in ips.values() if v}
    split_tunneling = len(unique_ips) > 1
    return split_tunneling, ips
```

### What the cloud provider actually sees — definitive checks

**AWS — via CloudTrail:**
```bash
# 1. Trigger a CloudTrail event
aws sts get-caller-identity

# 2. Wait 2s, then query CloudTrail for the source IP
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetCallerIdentity \
  --max-results 1 \
  --query 'Events[0].CloudTrailEvent' \
  --output text | python3 -c "import json,sys; print(json.load(sys.stdin)['sourceIPAddress'])"
```

**GCP — via audit log:**
```bash
ACCOUNT=$(gcloud config get-value account)
gcloud logging read \
  "protoPayload.authenticationInfo.principalEmail=\"${ACCOUNT}\"" \
  --limit=1 --format=json \
  | python3 -c "import json,sys; logs=json.load(sys.stdin); print(logs[0]['httpRequest']['remoteIp'])"
```

### The fix for split tunneling

**Add ALL detected IPs to the firewall**, not just the one from `ifconfig.me`:

```bash
# Collect all your IPs
GENERAL_IP=$(curl -s ifconfig.me)
AWS_IP=$(aws cloudtrail lookup-events ... | jq -r '.sourceIPAddress')
# GCP equivalent...

# Add each one
for IP in "$GENERAL_IP" "$AWS_IP"; do
  [[ -n "$IP" ]] && _add_firewall_rule "$IP/32"
done
```

> **AWS checkip vs AWS API:** `checkip.amazonaws.com` and `aws_api` (CloudTrail) may still differ if VPN only splits on *signed* AWS traffic. Always use CloudTrail as the authoritative AWS source IP.

---

## Part 3: Serverless egress IP

### Method 1: Manual (quick, single observation)

Run from a `%sh` cell on Databricks serverless:
```bash
curl -s https://ifconfig.me
# or
curl -s https://api.ipify.org
```

The returned IP is the NAT64 gateway's public IPv4. **Whitelist the entire `/24`** — serverless uses a rotating pool within the range. Example: observed `20.96.41.141` → whitelist `20.96.41.0/24`.

### Method 2: Automated via Jobs API (`discover-egress-ip.sh` pattern)

Upload a one-cell notebook and run it as a serverless job from your laptop. This is the authoritative way — you observe the actual IP from within a real serverless execution.

**Notebook cell content:**
```python
import urllib.request, socket
public_ip  = urllib.request.urlopen("https://api.ipify.org").read().decode()
private_ip = socket.gethostbyname(socket.gethostname())
dbutils.notebook.exit(f"public={public_ip} private={private_ip}")
```

**Shell driver (`scripts/discover-egress-ip.sh`):**

```bash
# 1. Upload notebook via workspace API
curl --silent --show-error --fail \
  "${WORKSPACE_URL}/api/2.0/workspace/import" \
  --header "Authorization: Bearer ${_TOKEN}" \
  --data "$(jq -n \
    --arg path "/Shared/databricks-discover-egress-ip" \
    --arg content "$_NB_BASE64" \
    '{path:$path, format:"JUPYTER", language:"PYTHON", content:$content, overwrite:true}')"

# 2. Submit as serverless run (environment_key + client:"1" = serverless)
_RUN_ID=$(curl --silent --show-error --fail \
  "${WORKSPACE_URL}/api/2.1/jobs/runs/submit" \
  --header "Authorization: Bearer ${_TOKEN}" \
  --data "$(jq -n --arg path "/Shared/databricks-discover-egress-ip" '{
    run_name: "discover-egress-ip",
    tasks: [{
      task_key: "check",
      notebook_task: {notebook_path: $path},
      environment_key: "default"
    }],
    environments: [{environment_key: "default", spec: {client: "1"}}]
  }')" | jq -r '.run_id')

# 3. Poll life_cycle_state every 15s (serverless cold start ≈ 5–8 min on staging)
for i in $(seq 1 60); do
  _RESP=$(curl --silent --fail "${WORKSPACE_URL}/api/2.1/jobs/runs/get?run_id=${_RUN_ID}" \
    --header "Authorization: Bearer ${_TOKEN}")
  _STATE=$(echo "$_RESP" | jq -r '.state.life_cycle_state // empty')
  _TASK_RUN_ID=$(echo "$_RESP" | jq -r '.tasks[0].run_id // empty')
  [[ "$_STATE" == "TERMINATED" ]] && break
  sleep 15
done

# 4. Read notebook exit value
_OUTPUT=$(curl --silent --fail \
  "${WORKSPACE_URL}/api/2.1/jobs/runs/get-output?run_id=${_TASK_RUN_ID}" \
  --header "Authorization: Bearer ${_TOKEN}" \
  | jq -r '.notebook_output.result // empty')
# Output format: "public=20.96.41.141 private=10.x.x.x"

# 5. Export the IP and derive the /24 range
export SERVERLESS_EGRESS_IP=$(echo "$_OUTPUT" | sed 's/public=\([^ ]*\).*/\1/')
_BASE=$(echo "$SERVERLESS_EGRESS_IP" | sed 's/\.[0-9]*$//')
export SERVERLESS_EGRESS_IP_START="${_BASE}.0"
export SERVERLESS_EGRESS_IP_END="${_BASE}.255"
```

> `client: "1"` in `environments.spec` is what makes this run on serverless. Remove it and add a `cluster_id` or `new_cluster` spec to run on classic compute instead.

Key points:
- The notebook also returns `private_ip` (the internal NAT64 node address) — useful for diagnosing routing
- `SERVERLESS_EGRESS_IP_START`/`END` are the `/24` range — pass them to `_add_*_ip_rule` helpers
- Cold start on staging can take 5–8 min; production is typically faster

### Method 3: Classic compute (non-serverless clusters)

Classic Databricks clusters use IPv4 (not IPv6/NAT64). Their egress IP is the public IP or NAT gateway of the cluster's VNet.

**Option A — Interactive notebook on an existing cluster:**
```python
# Run in a %sh cell on the target cluster
import urllib.request
ip = urllib.request.urlopen("https://api.ipify.org").read().decode()
print(f"Cluster egress IP: {ip}")
```

**Option B — Submit a job to a specific cluster:**
```bash
# Run against an existing cluster (replace environment_key/environments with cluster_id)
_RUN_ID=$(curl --silent --show-error --fail \
  "${WORKSPACE_URL}/api/2.1/jobs/runs/submit" \
  --header "Authorization: Bearer ${_TOKEN}" \
  --data "$(jq -n \
    --arg path "/Shared/databricks-discover-egress-ip" \
    --arg cluster_id "$CLUSTER_ID" \
    '{run_name: "discover-egress-ip-classic",
      tasks: [{
        task_key: "check",
        notebook_task: {notebook_path: $path},
        existing_cluster_id: $cluster_id
      }]}')" | jq -r '.run_id')
```

**Classic vs serverless egress IPs differ:**

| Compute type | Networking | Egress IP source |
|---|---|---|
| Serverless | IPv6 + NAT64, Databricks-managed VNet | NCC pool (rotating, use `/24`) |
| Classic (no VNet injection) | IPv4, Databricks-managed VNet | Databricks NAT gateway |
| Classic (VNet injection) | IPv4, customer VNet | Customer NAT gateway / public IP |

Classic clusters with VNet injection egress from the customer's own NAT gateway — this IP is stable and known from the Azure portal, no job needed.

---

## Part 4: Per-cloud firewall commands

### Azure — MySQL Flexible Server

```bash
# Check if rule exists (idempotent)
_add_mysql_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  if az mysql flexible-server firewall-rule show \
       --resource-group "$RESOURCE_GROUP" --name "$server" \
       --rule-name "$rule_name" --output none &>/dev/null; then
    echo "  rule $rule_name already exists — skipping"
    return
  fi
  az mysql flexible-server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --name "$server" \
    --rule-name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
}
_add_mysql_ip_rule "$MYSQL_SERVER" "$MY_IP"               "my-ip-$(date +%Y%m%d)"
_add_mysql_ip_rule "$MYSQL_SERVER" "$SERVERLESS_EGRESS_IP" "databricks-serverless-egress"
```

**Note:** MySQL Flexible Server in public-access mode does **not** support VNet rules. IP rules only.

### Azure — PostgreSQL Flexible Server

```bash
_add_pg_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  if az postgres flexible-server firewall-rule show \
       --resource-group "$RESOURCE_GROUP" --name "$server" \
       --rule-name "$rule_name" --output none &>/dev/null; then
    echo "  rule $rule_name already exists — skipping"
    return
  fi
  az postgres flexible-server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --name "$server" \
    --rule-name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
}
```

**Idempotency check:** compare `startIpAddress` and `endIpAddress` from `firewall-rule list` output, not rule name (rule names vary across runs).

### Azure — SQL Server (logical server)

```bash
_add_sql_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  az sql server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --server "$server" \
    --name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
  # create is idempotent — overwrites if rule name exists
}
```

**VNet rules for Azure SQL:** Databricks worker subnets only have `Microsoft.Storage` endpoint — **not** `Microsoft.Sql`. Without `Microsoft.Sql` endpoint, serverless traffic exits as a public IP and VNet rules never match. Use IP rules.

### Azure — NSG (for VMs)

```bash
_add_nsg_ip_rule() {
  local nsg="$1" ip="$2" priority="$3" rule_name="$4"
  az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$nsg" \
    --name "$rule_name" \
    --priority "$priority" \
    --direction Inbound \
    --access Allow \
    --protocol Tcp \
    --source-address-prefixes "$ip" \
    --destination-port-ranges "$DB_PORT" \
    --output none
}
# Priority scheme: VirtualNetwork=1000, serverless /24=1010, laptop /32=1020
_add_nsg_ip_rule "$NSG_NAME" "20.96.41.0/24"  1010 "databricks-serverless-egress"
_add_nsg_ip_rule "$NSG_NAME" "${MY_IP}/32"    1020 "my-ip-$(date +%Y%m%d)"
```

### AWS — RDS security group

```bash
# Get VPC security group from RDS instance
SG_ID=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_INSTANCE_ID" --region "$REGION" \
  --query 'DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId' \
  --output text)

# Add inbound rule for current IP
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port "$DB_PORT" \
  --cidr "${MY_IP}/32" \
  --region "$REGION"
```

To update (remove stale, add new):
```bash
# Revoke old
aws ec2 revoke-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port "$DB_PORT" --cidr "$OLD_IP/32" --region "$REGION"
# Authorize new
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port "$DB_PORT" --cidr "${MY_IP}/32" --region "$REGION"
```

### GCP — Cloud SQL authorized networks

```bash
# Get current networks, add new IP, patch in one call (merge in Python/jq)
CURRENT=$(gcloud sql instances describe "$INSTANCE" --format=json \
  | jq -r '.settings.ipConfiguration.authorizedNetworks[].value' | tr '\n' ',')
gcloud sql instances patch "$INSTANCE" \
  --authorized-networks "${CURRENT}${MY_IP}/32" \
  --quiet
```

**Note:** `gcloud sql instances patch --authorized-networks` **replaces** the entire list — always merge existing + new. `--quiet` suppresses the interactive prompt. Timeout: allow up to 300 seconds for the patch to propagate.

---

## Part 5: DB_FIREWALL_CIDRS pattern (mist/Terraform)

`DB_FIREWALL_CIDRS` is a space-separated list of CIDRs passed into setup scripts and Terraform:

```bash
# Default (wide open for demos)
export DB_FIREWALL_CIDRS="${DB_FIREWALL_CIDRS:-"0.0.0.0/0"}"

# For production: caller's /32 + serverless pool
export DB_FIREWALL_CIDRS="${MY_IP}/32 20.96.41.0/24"
```

The `terraform_common.sh` `setup_firewall_cidrs` function:
```bash
setup_firewall_cidrs() {
  current_ip=$(curl -s http://ipv4.icanhazip.com 2>/dev/null || echo "")
  if [[ -n "$current_ip" ]]; then
    DB_FIREWALL_CIDRS="${current_ip}/32"
  fi
  # Writes to per-cloud tfvars
}
```

Per-cloud Terraform variable names:
| Cloud | Terraform variable |
|-------|-------------------|
| AWS | `additional_cidrs` |
| Azure | `allowed_cidrs` |
| GCP | `authorized_networks` |

**CIDR correction pattern:** Terraform sometimes receives a host address instead of a network address. Fix before passing to cloud APIs:
```
128.77.97.250/30  →  128.77.97.248/30   (host → network)
```

---

## Part 6: Idempotency and rule naming

### Date-based rule names

```bash
# Creates a new rule each day; old rules accumulate but don't cause errors
RULE_NAME="my-ip-$(date +%Y%m%d)"    # e.g. my-ip-20260428
```

Periodic cleanup: list rules matching `my-ip-*` and delete those older than N days.

### `sqlcli-helper.sh` `_sqlcli_ensure_my_ip` pattern

```bash
_sqlcli_ensure_my_ip() {
  local access_type="${access_type:-}"
  # Skip NSG/firewall logic for VM databases — handled separately
  [[ "$access_type" == "vm" ]] && return

  local my_ip
  my_ip=$(curl --silent --fail --max-time 5 https://api.ipify.org) || return
  local rule_name="my-ip-$(date +%Y%m%d)"
  # Call _add_*_ip_rule for the detected DB type
}
```

> Always skip firewall logic for `access_type=vm` — VMs use NSG rules added separately via `customer-vm-base.sh`.

---

## Part 7: Why "add my IP" doesn't always work

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Rule added but connection still fails | Split tunneling — cloud sees a different IP | Detect with CloudTrail/GCP audit; add all IPs |
| Works from laptop, fails from serverless | Serverless egress IP not whitelisted | Add `/24` range or run `discover-egress-ip.sh` |
| Works from serverless via PNG, fails without PNG | PNG routing active; public path IP not whitelisted | Add serverless egress IP to public-path firewall |
| Azure Flexible Server: no VNet rules | `Microsoft.Sql` endpoint missing from Databricks subnets | Use IP rules only |
| IP rule added but rule shows start=end= "no-IP" | `${MY_IP}` echo adjacent to non-ASCII dash character (en-dash) | Use `${MY_IP}` with braces; see `shell-script-practices` |
