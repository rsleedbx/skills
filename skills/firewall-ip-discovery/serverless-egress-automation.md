# Serverless Egress IP — Automated Discovery

## Method 2: Automated via Jobs API

Upload a one-cell notebook and run it as a serverless job. This is the authoritative way — you observe the actual egress IP from within a real serverless execution.

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

# 2. Submit as serverless run — environments.spec.client="1" = serverless
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

# 5. Export IP and derive /24 range
export SERVERLESS_EGRESS_IP=$(echo "$_OUTPUT" | sed 's/public=\([^ ]*\).*/\1/')
_BASE=$(echo "$SERVERLESS_EGRESS_IP" | sed 's/\.[0-9]*$//')
export SERVERLESS_EGRESS_IP_START="${_BASE}.0"
export SERVERLESS_EGRESS_IP_END="${_BASE}.255"
```

> `client: "1"` in `environments.spec` makes this run on serverless. Remove it and add `existing_cluster_id` or `new_cluster` to run on classic compute instead.

- `private_ip` is the internal NAT64 node address — useful for diagnosing routing
- `SERVERLESS_EGRESS_IP_START`/`END` are the `/24` range — pass to `_add_*_ip_rule` helpers
- Cold start on staging ≈ 5–8 min; production is typically faster

---

## Method 3: Classic compute (non-serverless clusters)

Classic Databricks clusters use IPv4 (not IPv6/NAT64).

**Option A — Interactive notebook on an existing cluster:**
```python
# Run in a %sh cell on the target cluster
import urllib.request
ip = urllib.request.urlopen("https://api.ipify.org").read().decode()
print(f"Cluster egress IP: {ip}")
```

**Option B — Submit a job to a specific cluster:**
```bash
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

**Classic vs serverless egress IPs:**

| Compute type | Networking | Egress IP source |
|---|---|---|
| Serverless | IPv6 + NAT64, Databricks-managed VNet | NCC pool (rotating, use `/24`) |
| Classic (no VNet injection) | IPv4, Databricks-managed VNet | Databricks NAT gateway |
| Classic (VNet injection) | IPv4, customer VNet | Customer NAT gateway / public IP |

Classic clusters with VNet injection egress from the customer's own NAT gateway — this IP is stable and known from the Azure portal, no job needed.
