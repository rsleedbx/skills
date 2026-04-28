---
name: databricks-secret-credentials
description: Store, retrieve, merge, and refresh credentials and config in Databricks secret scopes as JSON blobs. Use when any notebook or .py file needs to persist IDs, secrets, endpoints, tokens, or other config across runs and workspaces — Lakeflow Connect sources/targets, OAuth clients, API keys, database passwords, service principal secrets, etc.
---

# Databricks Secret Credentials — Create / Save / Retrieve / Refresh

Covers the full lifecycle for credentials and config stored as a JSON blob in a Databricks secret scope. Works identically in workspace notebooks, Databricks Connect (local), and standalone `.py` scripts.

---

## Scope and key naming conventions

Databricks workspaces have a **limit on the number of secret scopes** (typically 100 per workspace). Keep scopes coarse-grained and use keys to differentiate connections.

### Scope — one per project or team, not one per connection

```
# GOOD — one scope covers all connections for a project
scope: "lakeflow-connect"

# BAD — one scope per connection burns the quota fast
scope: "lakeflow-connect-salesforce-prod"   # ❌
scope: "lakeflow-connect-mysql-staging"      # ❌
```

### Key — use the FQDN for external sources

For any external system that has a hostname (database server, API endpoint, SaaS instance), use the **fully-qualified domain name (FQDN)** as the secret key. This is unambiguous, human-readable, and naturally unique per connection.

```
scope: "lakeflow-connect"

key: "choo9chu-sq.database.windows.net"          # Azure SQL / SQL Server
key: "prod-cluster.abcdef.us-east-1.rds.amazonaws.com"  # AWS RDS
key: "mycompany.my.salesforce.com"               # Salesforce
key: "mycompany.snowflakecomputing.com"          # Snowflake
key: "kafka-broker.internal.example.com"         # Kafka broker
```

For Databricks-internal resources that have no FQDN, use a descriptive kebab-case key:

```
key: "databricks-sp-lfcdemo"     # Databricks service principal
key: "otel-grafana-rslee6392"    # Grafana OTLP config
```

### Summary

| Resource type | Scope | Key |
|---|---|---|
| External DB / SaaS / API | one scope per project | FQDN of the server |
| Databricks SP OAuth | one scope per project | `databricks-sp-<name>` |
| Observability config | one scope per project | `otel-<provider>-<id>` |

---

## Storage model

One `scope / key` = one JSON string containing all fields for one connection or config set.

```
scope: "lakeflow-connect"
key:   "choo9chu-sq.database.windows.net"
value: {"HOST": "choo9chu-sq.database.windows.net", "PORT": "1433",
        "DATABASE": "mydb", "USERNAME": "svc_ingest", "PASSWORD": "…"}
```

- `dbutils.secrets.get()` / `w.dbutils.secrets.get()` returns the plaintext string (auto-decoded).
- `w.secrets.put_secret(string_value=…)` writes it.
- Updates **merge** into the existing blob — partial updates never lose other fields.

---

## Core helpers

Copy these into `zbhelper/` or inline. They cover every lifecycle operation.

```python
import json
from databricks.sdk import WorkspaceClient
from databricks.sdk.errors import ResourceAlreadyExists, ResourceDoesNotExist


def ensure_scope(w: WorkspaceClient, scope: str) -> None:
    try:
        w.secrets.create_scope(scope)
        print(f"Created scope {scope!r}")
    except ResourceAlreadyExists:
        pass


def load_blob(dbutils, scope: str, key: str) -> dict:
    """Read JSON blob. Returns {} if scope/key missing or empty."""
    try:
        return json.loads(dbutils.secrets.get(scope=scope, key=key))
    except Exception:
        return {}


def save_blob(w: WorkspaceClient, scope: str, key: str, blob: dict) -> None:
    """Write dict as JSON. Overwrites existing value."""
    w.secrets.put_secret(scope=scope, key=key, string_value=json.dumps(blob, indent=2))


def upsert_blob(
    w: WorkspaceClient,
    dbutils,
    scope: str,
    key: str,
    fields: dict,
) -> dict:
    """Idempotent: create scope+key if missing, merge fields, write only if changed.

    Returns the final merged blob.
    """
    ensure_scope(w, scope)
    try:
        w.secrets.get_secret(scope, key)
    except ResourceDoesNotExist:
        w.secrets.put_secret(scope=scope, key=key, string_value="{}")
        print(f"Initialised empty secret {scope!r}/{key!r}")

    current = load_blob(dbutils, scope, key)
    merged  = {**current, **fields}
    if merged != current:
        save_blob(w, scope, key, merged)
        print(f"Saved {scope!r}/{key!r}: updated {list(fields.keys())}")
    else:
        print(f"No change to {scope!r}/{key!r}")
    return merged
```

---

## Lifecycle patterns

### First-time bootstrap — store credentials

Run once locally (`.local.ipynb` or `.py`). Never commit credential values.

```python
SCOPE = "lakeflow-connect"

# External DB — key is the FQDN
blob = upsert_blob(w, dbutils, SCOPE, "choo9chu-sq.database.windows.net", {
    "HOST":     "choo9chu-sq.database.windows.net",
    "PORT":     "1433",
    "DATABASE": "mydb",
    "USERNAME": "svc_ingest",
    "PASSWORD": "<password>",
})

# SaaS — key is the instance FQDN
blob = upsert_blob(w, dbutils, SCOPE, "mycompany.my.salesforce.com", {
    "CLIENT_ID":     "<id>",
    "CLIENT_SECRET": "<secret>",
    "INSTANCE_URL":  "https://mycompany.my.salesforce.com",
})
```

### Read in any notebook or .py

```python
import json
from databricks.sdk import WorkspaceClient

w     = WorkspaceClient()               # or WorkspaceClient(profile="…")
creds = json.loads(w.dbutils.secrets.get(
    scope="lakeflow-connect",
    key="choo9chu-sq.database.windows.net",
))

HOST     = creds["HOST"]
PASSWORD = creds["PASSWORD"]
```

### Rotate / refresh a single field

Only the changed fields are written; all others in the blob are preserved.

```python
upsert_blob(w, dbutils, "lakeflow-connect", "choo9chu-sq.database.windows.net", {
    "PASSWORD": "<new-password>",
    "LAST_ROTATED_AT": "2026-04-24T00:00:00Z",
})
```

### Validate before use + auto-refresh (OAuth client_credentials)

```python
import urllib.request
from urllib.parse import urlencode


def _oidc_valid(client_id: str, client_secret: str, token_url: str) -> bool:
    if not client_id or not client_secret:
        return False
    try:
        body = urlencode({
            "grant_type": "client_credentials",
            "client_id": client_id,
            "client_secret": client_secret,
            "scope": "all-apis",
        }).encode()
        req = urllib.request.Request(
            token_url, data=body, method="POST",
            headers={"Content-Type": "application/x-www-form-urlencoded"},
        )
        with urllib.request.urlopen(req, timeout=15) as r:
            return 200 <= getattr(r, "status", 200) < 300
    except Exception:
        return False


SCOPE, KEY = "my-project", "databricks-sp"
creds     = load_blob(dbutils, SCOPE, KEY)
TOKEN_URL  = f"{WORKSPACE_URL}/oidc/v1/token"

if not _oidc_valid(creds.get("CLIENT_ID",""), creds.get("CLIENT_SECRET",""), TOKEN_URL):
    new_secret = _mint_sp_secret(w, creds["SP_ID"])          # see databricks-service-principal skill
    creds = upsert_blob(w, dbutils, SCOPE, KEY, {"CLIENT_SECRET": new_secret})
```

---

## Bash bootstrap (CI/CD or shell scripts)

```bash
SCOPE="lakeflow-connect"
KEY="choo9chu-sq.database.windows.net"   # FQDN as key
PROFILE="<your-profile>"

# Create scope (idempotent)
databricks secrets create-scope "$SCOPE" --profile "$PROFILE" 2>/dev/null || true

# Write blob using jq to build safe JSON (handles quotes / backslashes in values)
SECRET_JSON=$(jq -n \
  --arg host "$HOST" \
  --arg port "$PORT" \
  --arg db   "$DATABASE" \
  --arg user "$USERNAME" \
  --arg pass "$PASSWORD" \
  '{HOST: $host, PORT: $port, DATABASE: $db, USERNAME: $user, PASSWORD: $pass}')

databricks secrets put-secret "$SCOPE" "$KEY" \
  --string-value "$SECRET_JSON" --profile "$PROFILE"
```

Read back (CLI output `.value` is base64 — always decode):

```bash
databricks secrets get-secret "$SCOPE" "$KEY" --profile "$PROFILE" --output json \
  | jq -r '.value | @base64d'
```

---

## Recommended JSON field conventions

Use `UPPER_SNAKE_CASE` keys so fields are grep-able across notebooks.

| Use case | Suggested fields |
|---|---|
| OAuth client_credentials | `CLIENT_ID`, `CLIENT_SECRET`, `TOKEN_URL` |
| Database | `HOST`, `PORT`, `DATABASE`, `USERNAME`, `PASSWORD` |
| API key | `API_KEY`, `BASE_URL` |
| Service principal (Databricks) | `CLIENT_ID`, `CLIENT_SECRET`, `SP_ID`, `APP_ID` |
| Any | `LAST_ROTATED_AT` (ISO-8601) — track when credentials were last changed |

---

## Key facts

| Topic | Detail |
|---|---|
| CLI decode | `.value` from CLI is base64 — pipe through `\| @base64d` |
| Notebook decode | `dbutils.secrets.get()` and `w.dbutils.secrets.get()` return plaintext — no decode needed |
| Merge pattern | Read → update dict → write; never replace whole blob from partial fields |
| Rotation | Call `upsert_blob` with only changed fields; other fields are preserved |
| Scope creation | `create_scope` raises `ResourceAlreadyExists` — catch and ignore |
| Cross-workspace | One scope per workspace; use `WorkspaceClient(profile="…")` to target each workspace |
| SP OAuth secret limit | 5 per SP — see `databricks-service-principal` skill for mint/validate pattern |

## Real examples in this repo

| File | What it shows |
|---|---|
| `notebooks/zbhelper/setup.py` — `ensure_service_principal()` | Full lifecycle: ensure scope, load blob, create SP, OIDC-validate, mint + save OAuth secret |
| `notebooks/otel/grafana/secrets-bootstrap.local.ipynb` | Merge-write pattern for Grafana OTLP config into a scope |
| `newskill/multiworkspace4.py` | Reading secrets from `.py` with `w.dbutils.secrets.get()` |
