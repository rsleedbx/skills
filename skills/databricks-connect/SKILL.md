---
name: databricks-connect
description: Set up and test Databricks Connect for local notebooks and Python scripts — spark, dbutils, WorkspaceClient, ~/.databrickscfg auth, GCP workspace URLs, metadata-service conflicts. Use when creating notebooks that run against a remote Databricks workspace, verifying serverless Spark works locally, or debugging auth errors with WorkspaceClient or databricks-connect.
---

# Databricks Connect

Runs `spark` and `dbutils` against a remote Databricks workspace from a local IDE or `.py` file. No cluster needed — uses serverless compute.

## Setup

### 1. Install Databricks CLI and log in

```bash
brew tap databricks/tap && brew install databricks
databricks auth login --configure-serverless --host https://<workspace>.azuredatabricks.net
```

Resulting `~/.databrickscfg`:
```ini
[DEFAULT]
host                  = https://<workspace>.azuredatabricks.net
serverless_compute_id = auto
auth_type             = databricks-cli
```

### 2. Install `databricks-connect` in your venv

```bash
pip install databricks-connect
```

### 3. Verify (see [test-template.md](test-template.md) for a full test script)

```bash
.venv/bin/python test_databricks_connect.py --profile DEFAULT
```

---

## In Notebooks

**Do not call `SparkSession.builder`** — the session is injected as `spark` and `dbutils` by Databricks Connect.

```python
# Both are available locally:
print(spark)    # <pyspark.sql.connect.session.SparkSession ...>
print(dbutils)  # <databricks.sdk.dbutils.RemoteDbUtils ...>

val = dbutils.secrets.get(scope="my-scope", key="my-key")
```

`NameError: name 'dbutils' is not defined` only occurs in a plain Python environment with no Databricks context — not in a Databricks Connect environment.

Pass `spark` into helper modules; never instantiate `SparkSession` inside `src/`.

---

## `WorkspaceClient()` — metadata-service conflict

Databricks Connect exports `DATABRICKS_AUTH_TYPE=metadata-service`. `WorkspaceClient()` then fails with `metadata-service: Metadata Service returned empty token`.

**Fix (pick one):**
1. Unset before creating the client: `del os.environ["DATABRICKS_AUTH_TYPE"]`
2. Use explicit profile: `WorkspaceClient(profile="DEFAULT")`
3. Unset in the shell before launching Jupyter: `unset DATABRICKS_AUTH_TYPE DATABRICKS_METADATA_SERVICE_URL`

---

## GCP Workspaces

- URL shape: `https://<WORKSPACE_ID>.<shard>.gcp.databricks.com`
- Set this URL as `targets.<name>.workspace.host` in `databricks.yml` for bundle/VS Code.
- Connect reads `~/.databrickscfg`, not `databricks.yml`.
- `get_workspace_id()` may return 400 on GCP — parse workspace ID from the first hostname label instead.

---

## How `~/.databrickscfg` is shared

| Component | Auth source |
|---|---|
| Databricks Connect (`spark`) | `~/.databrickscfg` |
| Databricks SDK (`WorkspaceClient`) | same file |
| ZeroBus gRPC SDK (`IngestConfig`) | `WorkspaceClient` → same file |

---

## Common Pitfalls

- **`SparkSession.builder` in notebook** — don't; use injected `spark`.
- **`dbutils` assumed unavailable locally** — wrong; Connect injects `RemoteDbUtils`.
- **`WorkspaceClient()` fails with metadata-service** — unset `DATABRICKS_AUTH_TYPE` (see above).
- **Wrong project root** — walk up from `Path.cwd()` to find `src/` directory.
