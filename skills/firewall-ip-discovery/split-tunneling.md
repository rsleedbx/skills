# Split Tunneling — Detection and Fix

## What split tunneling is

Corporate VPNs sometimes route cloud API traffic (to `*.amazonaws.com`, `*.googleapis.com`) through the corporate network, while general internet traffic (to `ifconfig.me`) exits directly. This means **the IP that AWS/GCP sees for your API calls is different from the IP you discover with `curl ifconfig.me`** — your general-purpose firewall whitelist is wrong.

## Detection: multi-probe approach

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

> **AWS checkip vs AWS API:** `checkip.amazonaws.com` and CloudTrail may still differ if VPN only splits on *signed* AWS traffic. Always use CloudTrail as the authoritative AWS source IP.

## What the cloud provider actually sees — definitive checks

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

## The fix: add ALL detected IPs to the firewall

```bash
GENERAL_IP=$(curl -s ifconfig.me)
AWS_IP=$(aws cloudtrail lookup-events ... | jq -r '.sourceIPAddress')

for IP in "$GENERAL_IP" "$AWS_IP"; do
  [[ -n "$IP" ]] && _add_firewall_rule "${IP}/32"
done
```

## Laptop IP discovery — Python and Terraform

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
