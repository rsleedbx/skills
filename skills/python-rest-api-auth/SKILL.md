---
name: python-rest-api-auth
description: >-
  Python REST API authentication for Azure, Databricks, and GCP using delegated
  user credentials (CLI-cached OAuth) or Managed Identity — no SDK, no PATs, no
  Service Principals. Use when writing Python scripts that call Azure ARM, Databricks
  account/workspace APIs, or GCP APIs; when replacing bash scripts that use az/databricks
  CLI subprocess calls; or when deciding between Managed Identity vs CLI-delegated auth
  for a given environment.
---

# Python REST API Auth — No SDK, No PAT, No Service Principal

## The two tiers

| Environment | Method | CLI required? |
|-------------|--------|---------------|
| Azure VM / ACI / Functions / AKS | **Managed Identity** (IMDS) | No |
| Laptop / CI with human login | **CLI-delegated** (subprocess) | Yes — must be installed |

**Never use PATs** — long-lived, not rotated, being retired by all vendors in 2026.  
**Avoid Service Principals unless you have Databricks account-admin** — SP assignment requires admin; CLI-delegated OAuth does not.

---

## CLI requirements by environment

| Environment | Azure | Databricks | GCP | CLI required |
|-------------|-------|-----------|-----|--------------|
| Laptop | `az` subprocess | `WorkspaceClient` reads cache | `gcloud` subprocess | `az login` once + `databricks auth login` once + `gcloud auth application-default login` once |
| Notebook | N/A | `dbutils` built-in global | N/A | Nothing |
| Azure VM / ACI | `mi_token()` IMDS | `WorkspaceClient` + MI config | N/A | Nothing |
| CI/CD | `az` subprocess | env vars `DATABRICKS_HOST` + `DATABRICKS_TOKEN` | `gcloud` subprocess | Depends on runner |

**Databricks is the only cloud where the CLI binary is not required at runtime.** `WorkspaceClient` reads the token cache file that `databricks auth login` wrote — the CLI binary does not need to be on `PATH` for subsequent runs.

For Azure and GCP, the subprocess calls (`az account get-access-token`, `gcloud auth print-access-token`) are still needed at runtime — reading their internal cache files directly is not safe (formats change between CLI versions; GCP ADC files have multiple credential types requiring different handling).

### Initial login commands (once per machine/session)

```bash
az login
databricks auth login --host https://accounts.azuredatabricks.net -p my-account-profile
databricks auth login --host https://adb-<id>.<n>.azuredatabricks.net -p my-workspace-profile
gcloud auth application-default login \
  --scopes=https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/documents,\
https://www.googleapis.com/auth/drive
```

---

## `auth.py` — complete module (~70 lines)

All three clouds follow the same pattern: use a **lightweight, first-party auth-only package** that reads the CLI token cache in-process, handles all credential types, and refreshes without subprocess. All actual API calls still go through `requests`.

```python
import time
import requests

# Auth-only packages — lightweight, first-party, no full SDK pulled in
from azure.identity import AzureCliCredential          # pip install azure-identity
import google.auth                                      # pip install google-auth
import google.auth.transport.requests
from databricks.sdk import WorkspaceClient             # pip install databricks-sdk

_cache: dict = {}
_ws_clients: dict[str, WorkspaceClient] = {}
_gcp_creds = None

# ── Azure — AzureCliCredential reads MSAL cache in-process, handles all credential types ──
# Prereq: az login
# Also works with Managed Identity, Service Principal — same call, credential chain auto-detects.

def az_token(resource: str = "https://management.azure.com/") -> str:
    c = _cache.get(f"az:{resource}", {})
    if c.get("exp", 0) > time.time() + 60:
        return c["tok"]
    scope = resource.rstrip("/") + "/.default"
    tok = AzureCliCredential().get_token(scope)
    _cache[f"az:{resource}"] = {"tok": tok.token, "exp": tok.expires_on}
    return tok.token

# ── Databricks — WorkspaceClient reads ~/.databricks/token-cache.json in-process ──
# Prereq: databricks auth login --host <host> -p <profile>
# CLI binary not required after initial login. Works in notebook and on laptop.
# Two profiles needed:
#   dbx_token(ACCOUNT_PROFILE)   → account-level APIs (NCC, PNG, workspace mgmt)
#   dbx_token(WORKSPACE_PROFILE) → workspace-level APIs (pipelines, UC, secrets)

def dbx_token(profile: str) -> str:
    if profile not in _ws_clients:
        _ws_clients[profile] = WorkspaceClient(profile=profile)
    w = _ws_clients[profile]
    tok = w.config.token          # populated for PAT and M2M OAuth profiles
    if tok:
        return tok
    # CLI-delegated auth (auth_type=databricks-cli): config.token is empty.
    # config.authenticate() → Dict[str, str] triggers the CLI subprocess internally
    # and returns {"Authorization": "Bearer <token>"}.
    headers = w.config.authenticate()
    bearer = headers.get("Authorization", "")
    if not bearer.startswith("Bearer "):
        raise ValueError(f"no Bearer token for profile '{profile}' (auth_type={w.config.auth_type})")
    return bearer.removeprefix("Bearer ")

# ── Databricks notebook — dbutils is a built-in global, zero imports needed ──
# Returns (host, token). Call only when running inside a Databricks notebook.

def dbx_token_notebook() -> tuple[str, str]:
    ctx = dbutils.notebook.entry_point.getDbutils().notebook().getContext()  # noqa: F821
    return ctx.apiUrl().get(), ctx.apiToken().get()

# ── GCP — google.auth.default() reads ADC file, handles all credential types ──
# Prereq: gcloud auth application-default login
# Handles authorized_user, service_account, workload identity — subprocess was fragile.

def gcp_token(scopes: list[str] | None = None) -> str:
    global _gcp_creds
    if scopes is None:
        scopes = ["https://www.googleapis.com/auth/cloud-platform"]
    if _gcp_creds is None:
        _gcp_creds, _ = google.auth.default(scopes=scopes)
    if not _gcp_creds.valid:
        _gcp_creds.refresh(google.auth.transport.requests.Request())
    return _gcp_creds.token

# ── Azure Managed Identity — IMDS, no CLI, no package needed ─────────────────
# Use on Azure VM / ACI / Functions / AKS instead of az_token().

def mi_token(resource: str = "https://management.azure.com/") -> str:
    c = _cache.get(f"mi:{resource}", {})
    if c.get("exp", 0) > time.time() + 60:
        return c["tok"]
    r = requests.get("http://169.254.169.254/metadata/identity/oauth2/token",
                     headers={"Metadata": "true"},
                     params={"api-version": "2018-02-01", "resource": resource},
                     timeout=5)
    r.raise_for_status()
    d = r.json()
    _cache[f"mi:{resource}"] = {"tok": d["access_token"], "exp": int(d["expires_on"])}
    return d["access_token"]
```

### Why these are the right exceptions to "no full SDK"

| Package | What it is | What it avoids |
|---------|-----------|----------------|
| `azure-identity` | Microsoft auth-only library (~500 KB) | `azure-mgmt-*` resource management SDKs |
| `google-auth` | Google auth-only library (~300 KB) | `google-cloud-*` service SDKs |
| `databricks-sdk` | Used only for `.config.token` | SDK object models, pagination wrappers |

All actual API calls go through `requests` directly. These packages are used for exactly one thing: resolving and refreshing credentials from whatever the CLI cached.

---

## Making REST calls

```python
import requests
from auth import az_token, dbx_token

ACCOUNT_HOST      = "https://accounts.azuredatabricks.net"
WORKSPACE_HOST    = "https://adb-<id>.<n>.azuredatabricks.net"
ACCOUNT_ID        = "<account-id>"
ACCOUNT_PROFILE   = "my-account-profile"
WORKSPACE_PROFILE = "my-workspace-profile"

def az(method, path, **kw):
    r = requests.request(method,
        f"https://management.azure.com{path}",
        headers={"Authorization": f"Bearer {az_token()}"}, **kw)
    r.raise_for_status(); return r.json()

def dbx_acct(method, path, **kw):
    r = requests.request(method, ACCOUNT_HOST + path,
        headers={"Authorization": f"Bearer {dbx_token(ACCOUNT_PROFILE)}"}, **kw)
    r.raise_for_status(); return r.json()

def dbx_ws(method, path, **kw):
    r = requests.request(method, WORKSPACE_HOST + path,
        headers={"Authorization": f"Bearer {dbx_token(WORKSPACE_PROFILE)}"}, **kw)
    r.raise_for_status(); return r.json()

# Usage — tokens resolved once per process, cached until expiry
nccs  = dbx_acct("GET", f"/api/2.0/accounts/{ACCOUNT_ID}/network-connectivity-configs")
pipes = dbx_ws("GET", "/api/2.0/pipelines")
vnets = az("GET", f"/subscriptions/{SUB}/providers/Microsoft.Network/virtualNetworks?api-version=2024-01-01")
```

---

## Databricks account-level vs workspace-level tokens

This is the most common mistake. They use different OAuth endpoints and different profiles:

| API path prefix | Token source | Profile |
|----------------|--------------|---------|
| `accounts.azuredatabricks.net/api/2.0/accounts/...` | `dbx_token(ACCOUNT_PROFILE)` | account-level profile |
| `adb-<id>.azuredatabricks.net/api/2.0/...` | `dbx_token(WORKSPACE_PROFILE)` | workspace-level profile |

NCC, PNG, and workspace management are account-level. Pipelines, UC, secrets, and clusters are workspace-level.

---

## Login commands (one-time per session)

```bash
# Azure
az login

# Databricks — separate login for account and workspace profiles
databricks auth login --host https://accounts.azuredatabricks.net -p my-account-profile
databricks auth login --host https://adb-<id>.<n>.azuredatabricks.net -p my-workspace-profile

# GCP
gcloud auth application-default login \
  --scopes=https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/documents,\
https://www.googleapis.com/auth/drive
```

---

## GCP project ID — never use `google.auth.default()` for the project

`google.auth.default()` returns a tuple `(credentials, project)`. The project is populated
**only** for service account key files or well-configured Workload Identity. On a laptop with
`gcloud auth application-default login`, the second element is `None`.

Use `gcloud` or an explicit env var instead:

```python
import os, subprocess, json

def gcp_project() -> str:
    # 1. Explicit override (CI, ADC configured env)
    if p := os.getenv("CLOUDSDK_CORE_PROJECT") or os.getenv("GOOGLE_CLOUD_PROJECT"):
        return p
    # 2. gcloud active config
    r = subprocess.run(["gcloud", "config", "get", "project"],
                       capture_output=True, text=True, check=True)
    return r.stdout.strip()
```

Do **not** write `_, project = google.auth.default()` and then use `project` without
checking for `None` — it will fail silently on most developer machines.

---

## Azure subscription ID — env var before CLI before DefaultAzureCredential

When writing `azure-mgmt-*` code that needs a subscription ID, resolve it in this order:

```python
import os, subprocess, json

def azure_subscription_id() -> str:
    if sid := os.getenv("AZURE_SUBSCRIPTION_ID"):
        return sid
    r = subprocess.run(["az", "account", "show", "--output", "json"],
                       capture_output=True, text=True, check=True)
    return json.loads(r.stdout)["id"]
```

`DefaultAzureCredential` for auth is fine; for subscription resolution it returns the
**credential object**, not the subscription ID. Always resolve the subscription ID
separately before instantiating any `azure-mgmt-*` client.

---

## Caching behaviour

- `_cli_token` is called once per token per Python process. All REST calls within that run use the in-memory dict.
- Subprocess overhead: ~200ms per token, paid once at startup — not per API call.
- Tokens are refreshed automatically on next process start if expired.
- For long-running processes (daemons), use Managed Identity instead — IMDS responses cache for the token lifetime with no subprocess involved.

---

## Line count vs bash equivalent

| Scope | Bash (non-blank/comment lines) | Python (REST, this auth module) |
|-------|-------------------------------|----------------------------------|
| Auth helpers across all scripts | spread across 4 helper scripts, ~375 lines | `auth.py` ~55 lines |
| Per-script business logic | 1,238 lines across all scripts | ~495 lines |
| **Total** | **~1,238** | **~550** |

Primary savings come from: native JSON (replaces `jq`), `r.raise_for_status()` (replaces CMD wrapper), and function return values (replaces env-var passing via `export`).
