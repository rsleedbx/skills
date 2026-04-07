---
name: databricks-connect-test
description: Create or run a Python test file that verifies databricks-connect and serverless Spark work from a local .py file. Use when the user wants to test Databricks Connect, verify serverless_compute_id is working, check that Spark is reachable from a local Python script, or debug databricks-connect setup.
---

# Databricks Connect — Serverless Spark Test

## Key Facts

- `databricks-connect` must be installed in the **project `.venv`**, not system Python
- Run with: `.venv/bin/python <test_file>.py --profile <profile>`
- Profile names come from `~/.databrickscfg`
- `serverless_compute_id = auto` in the profile enables serverless Spark (no cluster needed)

## Verified Working Setup (this repo)

| Item | Value |
|---|---|
| Python | `.venv/bin/python` (Python 3.12) |
| Package | `databricks-connect 15.1.0` |
| Profile | `e2dogfood` |
| Spark version returned | `4.1.0` |
| Test file | `lfc/db/test_databricks_connect.py` |

## Test File Template

Create `test_databricks_connect.py`:

```python
#!/usr/bin/env python3
"""
Databricks Connect + Serverless Spark Test

Usage:
    python test_databricks_connect.py
    python test_databricks_connect.py --profile e2dogfood
"""

import argparse
import sys
import time


def test_spark_session(profile: str) -> bool:
    print(f"\n{'='*60}")
    print(f"  Databricks Connect — Serverless Spark Test")
    print(f"{'='*60}")
    print(f"  Profile : {profile}")

    try:
        from databricks.connect import DatabricksSession
    except ImportError:
        print("\n❌ databricks-connect is not installed.")
        print("   Run: pip install databricks-connect")
        return False

    print("\n[1/4] Creating Spark session...")
    t0 = time.time()
    try:
        spark = DatabricksSession.builder.profile(profile).getOrCreate()
        print(f"      ✅ Session created in {time.time()-t0:.1f}s")
    except Exception as e:
        print(f"      ❌ Failed: {e}")
        return False

    print("\n[2/4] Spark version...")
    try:
        print(f"      ✅ {spark.version}")
    except Exception as e:
        print(f"      ⚠️  {e}")

    print("\n[3/4] spark.range(1000).count()...")
    try:
        t0 = time.time()
        count = spark.range(1000).count()
        print(f"      ✅ count={count}  ({time.time()-t0:.1f}s)")
    except Exception as e:
        print(f"      ❌ {e}")
        return False

    print("\n[4/4] SQL query...")
    try:
        t0 = time.time()
        row = spark.sql("SELECT current_timestamp() AS ts").collect()[0]
        print(f"      ✅ ts={row['ts']}  ({time.time()-t0:.1f}s)")
    except Exception as e:
        print(f"      ❌ {e}")
        return False

    print(f"\n✅ ALL TESTS PASSED\n")
    return True


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--profile", default="e2dogfood")
    args = parser.parse_args()
    sys.exit(0 if test_spark_session(args.profile) else 1)


if __name__ == "__main__":
    main()
```

## How to Run

```bash
# From repo root
.venv/bin/python lfc/db/test_databricks_connect.py --profile e2dogfood
```

## dbutils Availability

`spark.dbutils` does **not** exist in `databricks-connect 15.x`. Use the Databricks SDK instead:

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient(profile="e2dogfood")

# dbutils.fs equivalent
files = list(w.dbutils.fs.ls("/"))

# dbutils.secrets equivalent
scopes = list(w.secrets.list_scopes())
secret = w.secrets.get_secret(scope="my-scope", key="my-key")
```

Or import the runtime shim (works in `.py` files with databricks-connect active):

```python
from databricks.sdk.runtime import dbutils  # works when connect session is active
dbutils.fs.ls("/")
dbutils.secrets.listScopes()
```

### Cross-environment pattern (works in both `.py` and `.ipynb`)

```python
try:
    dbutils  # already injected in Databricks notebooks (richer — includes widgets, notebook utils)
except NameError:
    from databricks.sdk.runtime import dbutils  # local .py / local Jupyter via databricks-connect
```

| Feature | Available | How |
|---|---|---|
| `dbutils.fs` | ✅ | `WorkspaceClient.dbutils.fs` or SDK runtime |
| `dbutils.secrets` | ✅ | `WorkspaceClient.secrets` or SDK runtime |
| `dbutils.widgets` | ❌ | Notebook-only |
| `dbutils.notebook` | ❌ | Notebook-only |

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `databricks-connect is not installed` | Wrong Python / not in `.venv` | Use `.venv/bin/python`, not `python3` |
| `ConnectionError` / auth failure | Profile not found or token expired | Run `databricks auth login --profile e2dogfood` |
| `serverless_compute_id` missing | Profile doesn't have serverless enabled | Add `serverless_compute_id = auto` to `~/.databrickscfg` |
| Hangs at session creation | Workspace unreachable | Check VPN / network access to workspace host |

## ~/.databrickscfg Profile Format

```ini
[e2dogfood]
host                  = https://<workspace>.cloud.databricks.com
serverless_compute_id = auto
auth_type             = databricks-cli
```
