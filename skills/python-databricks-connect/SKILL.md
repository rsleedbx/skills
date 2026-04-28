---
name: python-databricks-connect
description: Set up Python to connect to a Databricks workspace from a laptop using Databricks Connect — the ~/.databrickscfg serverless_compute_id = auto key, single vs multi-workspace profiles, pyspark verification, and the NameError: name 'spark' is not defined fix. Use when setting up Connect on a new machine, switching workspaces, or debugging why spark is unavailable in a local notebook or .py file.
---

# Python Databricks Connect — Setup and Verification

Databricks Connect lets local Python code (notebooks, `.py` files, `pyspark` REPL) run `spark` against a remote Databricks workspace without a local Spark installation.

---

## The key config: `serverless_compute_id = auto`

`~/.databrickscfg` controls which workspace and compute a profile targets.
The line `serverless_compute_id = auto` tells Connect to use serverless compute automatically — no cluster ID or cluster startup required.

```ini
# ~/.databrickscfg
[DEFAULT]
host                  = https://e2-demo-field-eng.cloud.databricks.com
serverless_compute_id = auto
auth_type             = databricks-cli
```

Without `serverless_compute_id = auto` (or an explicit cluster ID), Connect will fail to attach to compute.

---

## Setup

### 1. Install the Databricks CLI and log in with serverless

```bash
brew tap databricks/tap && brew install databricks
databricks auth login --configure-serverless --host https://<workspace-url>
```

`--configure-serverless` writes `serverless_compute_id = auto` to the profile automatically.

### 2. Install `databricks-connect` in your venv

`databricks-connect` and `pyspark` conflict — install only one.

```bash
pip uninstall pyspark -y
pip install "databricks-connect==17.3.*"   # match X.Y to your workspace runtime version
```

### 3. Verify — `pyspark` REPL

```bash
pyspark
```

Expected output confirms the connection:

```
Connected to the Databricks Connect server 'e2-demo-field-eng.cloud.databricks.com'.
SparkSession available as 'spark'.
```

### 4. Verify — standalone `.py` reading a Unity Catalog table

A minimal end-to-end check: create `verify.py` and run it locally.

```python
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder.getOrCreate()  # uses DEFAULT profile

df = spark.read.table("samples.nyctaxi.trips")
df.show(5)
```

```bash
python verify.py
```

Expected output — the query executes on serverless compute in the workspace and streams results back:

```
+--------------------+---------------------+-------------+-----------+----------+-----------+
|tpep_pickup_datetime|tpep_dropoff_datetime|trip_distance|fare_amount|pickup_zip|dropoff_zip|
+--------------------+---------------------+-------------+-----------+----------+-----------+
| 2016-02-13 21:47:53|  2016-02-13 21:57:15|          1.4|        8.0|     10103|      10110|
...
+--------------------+---------------------+-------------+-----------+----------+-----------+
```

`samples.nyctaxi.trips` is a built-in Databricks sample table available in all workspaces — no setup required. Replace with any Unity Catalog table the profile's identity has `SELECT` on.

---

## Multiple workspaces — named profiles

Each workspace gets its own `[PROFILE]` block with its own `host` and `serverless_compute_id = auto`.

```ini
# ~/.databrickscfg
[DEFAULT]
host                  = https://e2-demo-field-eng.cloud.databricks.com
serverless_compute_id = auto
auth_type             = databricks-cli

[DEV]
host                  = https://dev-workspace.azuredatabricks.net
serverless_compute_id = auto
auth_type             = databricks-cli
```

Use the profile in code:

```python
from databricks.connect import DatabricksSession
spark = DatabricksSession.builder.profile("DEV").getOrCreate()
```

Use the profile with the CLI:

```bash
databricks auth login --configure-serverless --host https://dev-workspace.azuredatabricks.net --profile DEV
```

---

## `NameError: name 'spark' is not defined`

This error appears in a `.py` file or a plain notebook cell when Connect is installed but `spark` has not been created yet:

```python
spark   # NameError: name 'spark' is not defined
```

**Cause:** In Cursor/Jupyter notebooks `spark` is injected automatically when the kernel starts — but only if Databricks Connect is the active kernel. In `.py` files it is never injected.

**Fix — bootstrap guard at the top of every `.py` entrypoint:**

```python
if "spark" not in vars():
    from databricks.connect import DatabricksSession
    from pyspark.dbutils import DBUtils
    spark = DatabricksSession.builder.getOrCreate()   # uses DEFAULT profile
    dbutils = DBUtils(spark)
```

The guard is skipped in workspace notebooks (where `spark` is already injected) and runs locally where it is not.

To target a specific profile locally:

```python
if "spark" not in vars():
    from databricks.connect import DatabricksSession
    from pyspark.dbutils import DBUtils
    spark = DatabricksSession.builder.profile("DEV").getOrCreate()
    dbutils = DBUtils(spark)
```

See the `databricks-connect` skill for the full portable `.py` pattern (passing `spark`/`dbutils` as arguments to functions).

---

## CLI scripts with `--profile` — mirroring the Databricks CLI pattern

Use **Typer** to add a `--profile` option that behaves the same way as the Databricks CLI's `--profile` flag. Typer derives the option name and help text from the type annotation — no decorators on the function itself.

```python
import typer
from pyspark.sql import SparkSession


def run_something(spark: SparkSession) -> None:
    df = spark.read.table("samples.nyctaxi.trips")
    df.show(5)


def run_secrets(w) -> None:
    import json
    secret_json = w.dbutils.secrets.get(scope="my-scope", key="my-key")
    config = json.loads(secret_json)
    print(f"Secret: {json.dumps(config, indent=2)}")


def main(
    profile: str = typer.Option("DEFAULT", help="~/.databrickscfg profile name."),
) -> None:
    from databricks.connect import DatabricksSession
    from databricks.sdk import WorkspaceClient

    spark = DatabricksSession.builder.profile(profile).getOrCreate()
    w = WorkspaceClient(profile=profile)

    run_something(spark)
    run_secrets(w)


if __name__ == "__main__":
    typer.run(main)
```

Usage — identical pattern to the Databricks CLI:

```bash
python script.py                        # uses DEFAULT profile
python script.py --profile azurefe      # uses [azurefe] block
python script.py --help                 # auto-generated from type hints
```

```
Usage: script.py [OPTIONS]

Options:
  --profile TEXT  ~/.databrickscfg profile name.  [default: DEFAULT]
  --help          Show this message and exit.
```

### `Exception: Cluster id or serverless are required but were not specified`

This error means the target profile is missing `serverless_compute_id = auto`:

```
Exception: Cluster id or serverless are required but were not specified.
```

Fix — add the line to every named profile you intend to use with Connect:

```bash
databricks auth login --configure-serverless --host https://<workspace-url> --profile azurefe
```

Or edit `~/.databrickscfg` manually:

```ini
[azurefe]
host                  = https://adb-984752964297111.11.azuredatabricks.net
serverless_compute_id = auto
auth_type             = databricks-cli
```

`DatabricksSession.builder.profile(name)` and `WorkspaceClient(profile=name)` both read from the same `[name]` block — pass the same string to both so they target the same workspace.

---

## Quick reference

| Goal | Command / setting |
|---|---|
| Write `serverless_compute_id = auto` automatically | `databricks auth login --configure-serverless --host <url>` |
| Verify connection | `pyspark` — look for "Connected to … SparkSession available as 'spark'" |
| Target a named profile in code | `DatabricksSession.builder.profile("DEV").getOrCreate()` |
| Fix `NameError: name 'spark'` in `.py` | Bootstrap guard: `if "spark" not in vars(): spark = DatabricksSession.builder.getOrCreate()` |
| Use serverless without specifying it in code | Set `serverless_compute_id = auto` in `~/.databrickscfg` |
| Add `--profile` CLI option to a `.py` script | `typer.Option("DEFAULT", help="~/.databrickscfg profile name.")` |
| `Exception: Cluster id or serverless are required` | Named profile missing `serverless_compute_id = auto` — run `databricks auth login --configure-serverless --profile <name>` |
