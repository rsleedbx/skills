---
name: databricks-service-principal
description: Create and maintain Databricks workspace service principals and their OAuth client secrets stored in Databricks secret scopes, using the Databricks CLI (bash) or Python SDK (notebooks). Use when creating a service principal, managing client secrets with a 5-secret-per-SP limit, storing SP credentials as JSON in a secret scope, reading secrets back with base64 decode, or implementing idempotent SP setup scripts.
---

# Databricks Service Principal

Covers idempotent SP creation and secret management in both bash (sourced scripts) and Python (notebooks/SDK).

---

## Secret scope: what is stored and how it is encoded

Secrets are stored as a JSON blob under one `scope/key`. The value is **base64-encoded** at rest — `dbutils.secrets.get()` decodes it automatically in notebooks; the CLI does not.

```bash
# CLI: must base64-decode manually
databricks secrets get-secret $SCOPE $KEY --profile $PROFILE --output json \
  | jq -r '.value | @base64d'   # .value is base64 — always pipe through @base64d
```

```python
# Python notebook: dbutils decodes automatically
import json
blob = json.loads(dbutils.secrets.get(scope=SCOPE, key=KEY))
```

---

## Bash: idempotent SP + secret (sourced script)

### Prerequisites

Must be sourced (`.` / `source`), not executed, so `export` propagates to the shell.

```bash
(return 0 2>/dev/null) || { echo "error: run with source: . ./script.sh" >&2; kill -INT $$; }
```

### Step 1 — Load state from secret scope

```bash
# Ensure scope exists (get-first pattern, no silent swallow)
if ! databricks secrets list-secrets "$SCOPE" --profile "$WS_PROFILE" &>/dev/null; then
  databricks secrets create-scope "$SCOPE" --profile "$WS_PROFILE"
fi

# Load state (base64-decode required)
declare -A SP=([sp_name]="" [sp_scim_id]="" [client_id]="" [client_secret]="" [secret_id]="")

SAVED=$(databricks secrets get-secret "$SCOPE" "$KEY" --profile "$WS_PROFILE" \
  --output json 2>/dev/null | jq -r 'if .value then .value | @base64d else empty end')

if [[ -n "$SAVED" ]]; then
  for key in sp_name sp_scim_id client_id client_secret secret_id; do
    val=$(echo "$SAVED" | jq -r --arg k "$key" '.[$k] // empty')
    [[ -n "$val" ]] && SP[$key]="$val"
  done
fi
```

### Step 2 — Create SP (workspace-level, no account admin needed)

```bash
if [[ -z "${SP[sp_scim_id]}" ]]; then
  # Check API first (handles SPs created via UI)
  EXISTING=$(databricks service-principals list --profile "$WS_PROFILE" \
    --filter "displayName eq \"$SP_NAME\"" --output json | jq -r '.[0] // empty')
  if [[ -n "$EXISTING" ]]; then
    SP[sp_scim_id]=$(echo "$EXISTING" | jq -r '.id')
    SP[client_id]=$(echo "$EXISTING"  | jq -r '.applicationId')
  else
    RESULT=$(databricks service-principals create --profile "$WS_PROFILE" \
      --json "{\"displayName\": \"$SP_NAME\"}")
    SP[sp_scim_id]=$(echo "$RESULT" | jq -r '.id')
    SP[client_id]=$(echo "$RESULT"  | jq -r '.applicationId')
  fi
  SP[sp_name]="$SP_NAME"
fi
```

### Step 3 — Create OAuth client secret (limit: 5 per SP)

Use `service-principal-secrets-proxy` — workspace-level, no account admin required.

```bash
if [[ -z "${SP[client_secret]}" ]]; then
  RESULT=$(databricks service-principal-secrets-proxy create "${SP[sp_scim_id]}" \
    --profile "$WS_PROFILE" --output json)
  SP[client_secret]=$(echo "$RESULT" | jq -r '.secret')
  SP[secret_id]=$(echo "$RESULT"     | jq -r '.id')
fi
```

### Step 4 — Save state (skip write if unchanged)

Build JSON from the associative array without special-character issues:

```bash
SECRET_JSON=$(for key in "${!SP[@]}"; do
  printf "%s\n%s\n" "$key" "${SP[$key]}"
done | jq -Rn '[inputs | {(.): input}] | add')

# Only write if content changed
if [[ "$(echo "$SECRET_JSON" | jq -cS .)" != "$(echo "$SAVED" | jq -cS . 2>/dev/null)" ]]; then
  databricks secrets put-secret "$SCOPE" "$KEY" \
    --string-value "$SECRET_JSON" --profile "$WS_PROFILE"
fi
```

### Step 5 — Mint bearer token (1-hour TTL, unlimited mints)

```bash
export DATABRICKS_TOKEN=$(curl --silent --fail \
  "${ACCOUNT_CONSOLE}/oidc/accounts/${ACCOUNT_ID}/v1/token" \
  --header "Authorization: Basic $(printf '%s:%s' "$CLIENT_ID" "$CLIENT_SECRET" | base64)" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "scope=all-apis" \
  | jq -r '.access_token')
```

---

## Python: idempotent SP + secret (notebook / SDK)

```python
import json
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()  # auth from env / .databrickscfg / cluster context

SCOPE, KEY = "my-scope", "sp-credentials"

# --- ensure scope and key exist ---
try:
    w.secrets.create_scope(SCOPE)
except Exception:
    pass  # already exists

try:
    w.secrets.get_secret(SCOPE, KEY)
except Exception:
    w.secrets.put_secret(scope=SCOPE, key=KEY, string_value="{}")

# --- load state ---
blob = json.loads(dbutils.secrets.get(scope=SCOPE, key=KEY))  # auto-decoded

# --- create SP if missing ---
sp_id = blob.get("sp_scim_id")
if not sp_id:
    sps = list(w.service_principals.list(filter=f'displayName eq "{SP_NAME}"'))
    if sps:
        sp = sps[0]
    else:
        sp = w.service_principals.create(display_name=SP_NAME)
    blob["sp_scim_id"] = str(sp.id)
    blob["client_id"]  = str(sp.application_id)

# --- validate or mint client secret ---
def _token_valid(client_id, client_secret, ws_url):
    import requests
    r = requests.post(f"{ws_url}/oidc/v1/token",
        data={"grant_type": "client_credentials", "scope": "all-apis"},
        auth=(client_id, client_secret), timeout=10)
    return r.status_code == 200

if not _token_valid(blob.get("client_id",""), blob.get("client_secret",""), DATABRICKS_WORKSPACE_URL):
    secret = w.service_principal_secrets.create(blob["sp_scim_id"])  # limit: 5 per SP
    blob["client_secret"] = secret.secret
    blob["secret_id"]     = secret.id
    w.secrets.put_secret(scope=SCOPE, key=KEY, string_value=json.dumps(blob))
```

---

## Key facts

| Topic | Detail |
|-------|--------|
| Secret encoding | CLI: `\| @base64d` required; `dbutils.secrets.get()` auto-decodes |
| SP creation | `databricks service-principals` (workspace-level, workspace admin only) |
| Client secret creation | `databricks service-principal-secrets-proxy create <sp_id>` (workspace-level) |
| Client secret limit | **5 per SP** — check/reuse before creating |
| Token TTL | **1 hour** — cheap to re-mint, no reason to cache in scope |
| JSON builder (bash) | Use `jq -n --arg` or the `printf key\nval | jq -Rn '[inputs\|{(.):input}]\|add'` pattern — never raw heredoc interpolation (breaks on `"` or `\` in secret values) |
