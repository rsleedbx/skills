# Databricks Connect — Test Template

## Test file: `test_databricks_connect.py`

```python
#!/usr/bin/env python3
"""Databricks Connect + Serverless Spark verification.

Usage:
    python test_databricks_connect.py
    python test_databricks_connect.py --profile DEFAULT
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

    print("\n✅ ALL TESTS PASSED\n")
    return True


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--profile", default="DEFAULT")
    args = parser.parse_args()
    sys.exit(0 if test_spark_session(args.profile) else 1)


if __name__ == "__main__":
    main()
```

## Run

```bash
.venv/bin/python test_databricks_connect.py --profile DEFAULT
```

## `dbutils` in `.py` files

`spark.dbutils` does not exist in `databricks-connect 15.x`. Use the runtime shim:

```python
from databricks.sdk.runtime import dbutils   # works when connect session is active
dbutils.fs.ls("/")
dbutils.secrets.listScopes()
```

Or via `WorkspaceClient`:
```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient(profile="DEFAULT")
files = list(w.dbutils.fs.ls("/"))
secret = w.secrets.get_secret(scope="my-scope", key="my-key")
```

Cross-environment pattern (`.py` and `.ipynb`):
```python
try:
    dbutils   # injected in Databricks workspace notebooks
except NameError:
    from databricks.sdk.runtime import dbutils  # local via databricks-connect
```

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `databricks-connect is not installed` | Wrong Python / not in venv | Use `.venv/bin/python` |
| `ConnectionError` / auth failure | Token expired or profile not found | `databricks auth login --profile DEFAULT` |
| `serverless_compute_id` missing | Profile missing serverless | Add `serverless_compute_id = auto` to `~/.databrickscfg` |
| Hangs at session creation | Workspace unreachable | Check VPN / network |
