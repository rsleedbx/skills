---
name: databricks-connect
description: Set up Databricks Connect for local notebooks and .py files — spark/dbutils injection, portable .py pattern (bootstrap guard + pass as args), what runs locally vs remotely, Spark Connect vs Databricks Connect, DatabricksSession, WorkspaceClient auth conflict, GCP URLs. Use when creating notebooks or .py scripts that run against a remote workspace, confused about spark/dbutils availability locally, or want code that works identically in workspace notebooks and local Connect.
---

# Databricks Connect

Runs `spark` and `dbutils` from a local IDE (Cursor, VS Code, PyCharm) or `.py` file against a remote Databricks workspace. No cluster needed — uses serverless compute.

---

## Spark Connect vs Databricks Connect

| | Spark Connect | Databricks Connect |
|---|---|---|
| What it is | Open-source gRPC protocol in Apache Spark for remote execution | Extension of Spark Connect for Databricks |
| Transport | Spark unresolved logical plans + Apache Arrow over gRPC | Same, plus Databricks auth, Unity Catalog, dbutils, serverless |
| Use with Databricks | Base protocol only | **Use this** — it includes all Databricks features |

**TL;DR:** When targeting Databricks, "Spark Connect" and "Databricks Connect" refer to the same library (`databricks-connect`). Databricks Connect *is* Spark Connect with Databricks extensions.

---

## spark and dbutils — just like a workspace notebook

When Databricks Connect is active, `spark` and `dbutils` are **automatically injected** into Jupyter notebooks and Cursor notebooks, exactly as they are in a Databricks workspace notebook.

```python
# These work out of the box — no import or setup needed in the notebook:
spark.sql("SELECT current_timestamp()").show()
dbutils.secrets.get(scope="my-scope", key="my-key")
dbutils.fs.ls("/mnt/")
```

The injected objects are remote proxies:
- `spark` → `pyspark.sql.connect.session.SparkSession` (remote session on Databricks compute)
- `dbutils` → `databricks.sdk.dbutils.RemoteDbUtils` (proxies to workspace)

**In `.py` files**, injection does not happen automatically. Use the bootstrap guard pattern so the same code works in both environments — see [portable.md](portable.md). For a standalone verification script see [test-template.md](test-template.md).

---

## What runs locally vs remotely

| Code | Runs |
|---|---|
| Python/Scala logic, loops, local variables | **Locally** on your machine |
| `spark.sql(...)`, DataFrame transformations | **Remotely** on Databricks serverless compute |
| `collect()`, `show()`, `toPandas()` | Materialises result locally |
| UDFs, `foreach`, `foreachBatch` | **Remotely** (serialised and sent to cluster) |
| `dbutils.secrets`, `dbutils.fs` | **Remotely** via workspace proxy |

This mirrors workspace notebook execution exactly — local Python, remote Spark.

---

## Portable .py files — bootstrap guard + pass as args

For code that must run in **both** a workspace notebook and a local `.py` file, see **[portable.md](portable.md)**. Key rules:

1. `if "spark" not in vars():` guard at the top of the entrypoint — skipped in notebooks, runs locally.
2. Pass `spark` and `dbutils` as arguments to every function and module — never create them inside.

---

## Do not call `SparkSession.builder` in notebooks

`SparkSession.builder` creates a disconnected local Spark session — **not** the remote Databricks session.

```python
# WRONG — creates a local, disconnected Spark session:
from pyspark.sql import SparkSession
spark = SparkSession.builder.getOrCreate()

# RIGHT — in notebooks, use the injected spark variable directly.
# In .py files, use DatabricksSession:
from databricks.connect import DatabricksSession
spark = DatabricksSession.builder.getOrCreate()          # reads ~/.databrickscfg DEFAULT
spark = DatabricksSession.builder.profile("DEV").getOrCreate()  # explicit profile
```

Pass `spark` into helper modules; never instantiate `SparkSession` or `DatabricksSession` inside `src/` library code.

---

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

`databricks-connect` conflicts with `pyspark` — uninstall PySpark first if present.

```bash
pip uninstall pyspark -y
pip install "databricks-connect==17.3.*"   # match X.Y to your cluster/runtime version
```

### 3. Verify

See [test-template.md](test-template.md) for the full verification script.

---

## `WorkspaceClient()` — metadata-service conflict

Databricks Connect exports `DATABRICKS_AUTH_TYPE=metadata-service`. `WorkspaceClient()` then fails with `metadata-service: Metadata Service returned empty token`.

**Fix (pick one):**
1. Unset before creating the client: `del os.environ["DATABRICKS_AUTH_TYPE"]`
2. Use explicit profile: `WorkspaceClient(profile="DEFAULT")`
3. Unset in shell before launching Jupyter: `unset DATABRICKS_AUTH_TYPE DATABRICKS_METADATA_SERVICE_URL`

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

| Mistake | Reality |
|---|---|
| `SparkSession.builder` in notebook | Creates local session; use injected `spark` |
| `dbutils` assumed unavailable locally | Wrong — Connect injects `RemoteDbUtils` |
| `WorkspaceClient()` fails with metadata-service | Unset `DATABRICKS_AUTH_TYPE` (see above) |
| `NameError: name 'dbutils'` | Only in plain `.py` without active Connect session; use `databricks.sdk.runtime.dbutils` |
| `pyspark` installed alongside `databricks-connect` | They conflict — uninstall `pyspark` |
| Wrong project root | Walk up from `Path.cwd()` to find `src/` directory |
