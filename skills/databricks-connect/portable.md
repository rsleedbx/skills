# Databricks Connect — Portable .py Pattern

Write `.py` files so the **same code runs unchanged** in a Databricks workspace notebook and locally via Databricks Connect. The two rules that make this work:

1. **Bootstrap guard at the top of every entrypoint** — creates `spark`/`dbutils` only when not already injected.
2. **Pass `spark` and `dbutils` as arguments** — functions and modules never create them internally.

---

## 1. Bootstrap guard

```python
if "spark" not in vars():
    from databricks.connect import DatabricksSession
    from pyspark.dbutils import DBUtils
    spark = DatabricksSession.builder.getOrCreate()   # reads DEFAULT profile
    dbutils = DBUtils(spark)
```

| Environment | What happens |
|---|---|
| Workspace notebook | `spark` already injected → guard is **skipped** |
| Local `.py` via Connect | `spark` not yet defined → guard **runs**, creates remote session |

No code changes when moving between environments.

**Profile variants:**

```python
# Explicit profile
spark = DatabricksSession.builder.profile("DEV").getOrCreate()

# Explicit serverless (for one-off scripts; avoid in library/production code)
spark = DatabricksSession.builder.serverless().profile("DEFAULT").getOrCreate()
```

`serverless_compute_id = auto` in `~/.databrickscfg` DEFAULT profile makes `.getOrCreate()` use serverless without spelling it out.

---

## 2. Always import as module, never import individual names

In notebooks, always import the module and call through it. Never bind individual names with `from … import`.

```python
# GOOD — zb.func always resolves through the reloaded module
import zbhelper.ingest_benchmark as zb
zb.run_benchmark(spark, dbutils)

# BAD — run_benchmark is bound at import time; %autoreload 2 updates zb.run_benchmark
# but the notebook-level name run_benchmark still points to the old object
from zbhelper.ingest_benchmark import run_benchmark
run_benchmark(spark, dbutils)   # stale after edits to the .py file
```

This pairs with `%autoreload 2`: autoreload replaces attributes on the module object, but it cannot update names that were already copied into the notebook's namespace via `from … import`.

---

## 3. Pass spark and dbutils as arguments — never create inside modules

All functions and `src/` modules receive `spark` (and `dbutils` when needed) as parameters.

```python
# GOOD — portable; works in notebook and .py
def process(spark, dbutils, table: str):
    df = spark.read.table(table)
    secret = dbutils.secrets.get(scope="my-scope", key="my-key")
    df.show(5)

process(spark, dbutils, "samples.nyctaxi.trips")
```

```python
# BAD — creates a second session; fails or behaves unexpectedly in a workspace notebook
def process(table: str):
    from databricks.connect import DatabricksSession
    spark = DatabricksSession.builder.getOrCreate()   # wrong: new session, not the notebook session
    spark.read.table(table).show(5)
```

**The Internet pattern of creating `spark` inside functions only works for standalone scripts and breaks portability.** Always inject.

---

## 4. Full entrypoint template

```python
#!/usr/bin/env python3
"""One-line description of what this script does."""

if "spark" not in vars():
    from databricks.connect import DatabricksSession
    from pyspark.dbutils import DBUtils
    spark = DatabricksSession.builder.getOrCreate()
    dbutils = DBUtils(spark)


def main(spark, dbutils):
    # All Spark / Unity Catalog logic here.
    # spark and dbutils come from the caller — never created here.
    df = spark.read.table("samples.nyctaxi.trips")
    df.show(5)


main(spark, dbutils)
```

---

## 5. Library / `src/` module pattern

Type-hint `spark` as `SparkSession` (the Connect-compatible supertype). Never import or call `DatabricksSession` inside a library module.

```python
# src/mymodule.py
from pyspark.sql import SparkSession


def count_rows(spark: SparkSession, table: str) -> int:
    return spark.read.table(table).count()


def read_secret(dbutils, scope: str, key: str) -> str:
    return dbutils.secrets.get(scope=scope, key=key)
```

Caller (entrypoint or notebook):

```python
from src.mymodule import count_rows, read_secret

n = count_rows(spark, "my_catalog.my_schema.my_table")
token = read_secret(dbutils, "my-scope", "my-token")
```

---

## 6. Importing helper modules from notebooks

On a Databricks workspace, `"."` (the notebook's own directory) is always on `sys.path`, but `".."` (the parent) is not. This means a helper package imported by notebooks must be **co-located in the same directory** as the notebooks — not under `src/`.

**Preferred layout:**

```
notebooks/
  zbhelper/          # helper package lives here, next to the notebooks
    __init__.py
    ingest_benchmark.py
  my_notebook.ipynb
src/
  statschema/        # packages used by pytest & scripts stay in src/
```

On the **workspace** the notebook's own directory is always on `sys.path`, so `import zbhelper` works with no setup. **Locally** (Cursor, VS Code), the Jupyter kernel CWD is typically the repo root, not `notebooks/`, so a one-line probe is needed:

```python
import sys
from pathlib import Path

# Try CWD (workspace — notebook dir is CWD) then CWD/notebooks (Cursor — repo root is CWD)
_nb = next((str(p) for p in [Path.cwd(), Path.cwd() / "notebooks"] if (p / "zbhelper").is_dir()), None)
if _nb and _nb not in sys.path:
    sys.path.insert(0, _nb)
```

For **pytest** to find the package, add `notebooks/` to `sys.path` once in `conftest.py`:

```python
# conftest.py (repo root)
import sys
from pathlib import Path

_notebooks = str(Path(__file__).parent / "notebooks")
if _notebooks not in sys.path:
    sys.path.insert(0, _notebooks)
```

| Situation | `"."` resolves to | `import zbhelper` works? |
|---|---|---|
| Workspace notebook | notebook dir = `notebooks/` | ✅ |
| Local kernel CWD = `notebooks/` | `notebooks/` | ✅ |
| pytest (via `conftest.py`) | N/A — `conftest.py` adds `notebooks/` | ✅ |

If a package genuinely belongs in `src/` (shared with non-notebook code), use the walk-up fallback instead:

```python
import sys
from pathlib import Path

def _find_src(marker: str = "mypackage") -> str:
    for parent in [Path.cwd(), *Path.cwd().parents]:
        candidate = parent / "src"
        if (candidate / marker).is_dir():
            return str(candidate)
    raise RuntimeError(f"Cannot find src/{marker} — run from within the repo")

_src = _find_src()
if _src not in sys.path:
    sys.path.insert(0, _src)
```

---

## 7. `dbutils` options in `.py` files

```python
# Option A — via DBUtils constructor (pairs naturally with DatabricksSession)
from pyspark.dbutils import DBUtils
dbutils = DBUtils(spark)

# Option B — SDK runtime shim (works when a Connect session is active)
from databricks.sdk.runtime import dbutils

# Option C — WorkspaceClient (for scripts that don't use spark at all)
from databricks.sdk import WorkspaceClient
w = WorkspaceClient(profile="DEFAULT")
secret = w.secrets.get_secret(scope="my-scope", key="my-key").value
```

Option A is preferred when `spark` is already created by the bootstrap guard. Option C is preferred for non-Spark scripts (no session needed).

---

## 8. Why this matters

| Pattern | Workspace notebook | Local .py (Connect) |
|---|---|---|
| Bootstrap guard + pass as args | ✅ | ✅ |
| Hard-coded `DatabricksSession` inside module | ❌ Creates second session | ⚠️ Works but wrong |
| `SparkSession.builder` inside module | ❌ Local/disconnected session | ❌ Local/disconnected session |
| `spark`/`dbutils` assumed as globals in module | ❌ NameError if called from `.py` | ❌ NameError if module imported first |
