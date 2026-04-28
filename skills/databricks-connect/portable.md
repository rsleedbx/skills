# Databricks Connect — Portable .py Pattern

Write `.py` files so the **same code runs unchanged** in a Databricks workspace notebook and locally via Databricks Connect. The two rules that make this work:

1. **Bootstrap guard at the top of every entrypoint** — creates `spark`/`dbutils` only when not already injected.
2. **Pass `spark` and `dbutils` as arguments** — functions and modules never create them internally.

---

## 0. First cell for a new interactive notebook

Every new interactive notebook (Jupyter / Cursor / Databricks workspace) needs **one** setup cell at the top. Copy this verbatim into any new notebook before writing business logic.

> **One cell, not two.** The env-var clear, `WorkspaceClient`, `spark`, and `dbutils` all serve the same purpose (initialising the runtime context) and belong together. Splitting them across two cells creates ordering confusion and makes the pattern harder to copy.

```python
import os
from databricks.sdk import WorkspaceClient

# Clear metadata-service auth set by Databricks Connect so WorkspaceClient uses ~/.databrickscfg locally.
if (os.environ.get("DATABRICKS_AUTH_TYPE") or "").strip().lower() == "metadata-service":
    os.environ.pop("DATABRICKS_AUTH_TYPE", None)
    os.environ.pop("DATABRICKS_METADATA_SERVICE_URL", None)
    print("Cleared metadata-service auth — WorkspaceClient will use CLI/profile auth.")

if "w" not in vars():
    w = WorkspaceClient()
if "spark" not in vars():
    from databricks.connect import DatabricksSession
    spark = DatabricksSession.builder.getOrCreate()
if "dbutils" not in vars():
    dbutils = w.dbutils
```

This cell is safe to run in a workspace notebook — the env var is not set there, so the `if` block is skipped and `w`/`spark`/`dbutils` are already injected.

| Environment | `spark` at cell entry | `dbutils` at cell entry | `w` at cell entry |
|---|---|---|---|
| Workspace notebook | ✅ injected | ✅ injected | ❌ not injected — guard runs |
| Databricks Connect (Cursor/Jupyter) kernel | ✅ usually injected | ✅ usually injected | ❌ not injected — guard runs |
| Plain Jupyter (no Connect kernel) | ❌ not injected — guard runs | ❌ not injected — guard runs | ❌ not injected — guard runs |

`w.dbutils` (from `WorkspaceClient`) is preferred for `dbutils` — see section 7 for why it has a fuller feature set than `DBUtils(spark)`.

> **`dbutils.widgets` works with Databricks Connect.** When `dbutils = w.dbutils` (from `WorkspaceClient`), calling `dbutils.widgets.dropdown()` and `dbutils.widgets.get()` works the same way both locally and in a workspace notebook. No `if/else` guard around widget calls is needed — do not add one.

---

## 0a. Package availability — when to add `%pip install`

### Rule

Before importing any third-party package in a notebook cell, check whether it is pre-installed by the runtime. If it is not, add a `%pip install` magic in a **dedicated cell that must be the very first code cell in the notebook** (before imports and before the Cell 1 `WorkspaceClient` setup). `%pip install` triggers a kernel restart; anything before it in the notebook is not re-executed.

```python
# Cell 0 — only present when non-standard packages are needed
%pip install databricks-zerobus-ingest-sdk "httpx[http2]" -q
```

### Serverless compute

Serverless notebooks currently run on the environment version that roughly corresponds to **Databricks Runtime 18.1** (released April 2026). Databricks upgrades the server-side runtime automatically. To confirm the current version:

1. Open [Serverless compute release notes](https://docs.databricks.com/aws/en/release-notes/serverless/) and note the latest version (e.g. `18.1`).
2. Open the matching runtime page: `https://docs.databricks.com/aws/en/release-notes/runtime/<version>` (e.g. `https://docs.databricks.com/aws/en/release-notes/runtime/18.1`) and scroll to **Installed Python libraries**.

**DBR 18.1 key pre-installed packages (no `%pip install` needed):**

| Package | Version | Package | Version |
|---|---|---|---|
| `pandas` | 2.2.3 | `numpy` | 2.1.3 |
| `matplotlib` | 3.10.0 | `seaborn` | 0.13.2 |
| `scipy` | 1.15.3 | `scikit-learn` | 1.6.1 |
| `pyarrow` | 21.0.0 | `plotly` | 5.24.1 |
| `requests` | 2.32.3 | `aiohttp` | 3.11.10 |
| `httpx` | 0.28.1 | `grpcio` | 1.67.0 |
| `databricks-sdk` | 0.67.0 | `mlflow-skinny` | 3.8.1 |
| `opentelemetry-api` | 1.39.1 | `opentelemetry-sdk` | 1.39.1 |
| `pydantic` | 2.10.6 | `nest-asyncio` | 1.6.0 |
| `typer-slim` | 0.21.1 | `fastapi` | 0.128.0 |
| `protobuf` | 5.29.4 | `cryptography` | 44.0.1 |

**Packages NOT pre-installed (must `%pip install`):**

| Package | Notes |
|---|---|
| `databricks-zerobus-ingest-sdk` | ZeroBus gRPC SDK — not part of the standard runtime |
| `httpx[http2]` | `httpx` is pre-installed but without the `h2` extra; HTTP/2 client support requires `%pip install "httpx[http2]"` |
| `h2` | HTTP/2 framing library (pulled in by `httpx[http2]`) |

### Shared / dedicated clusters

For shared or dedicated (non-serverless) clusters, check the cluster's **Databricks Runtime version** (visible in the cluster settings). Look up pre-installed packages at `https://docs.databricks.com/aws/en/release-notes/runtime/<version>` under **Installed Python libraries**.

### `%pip install` placement rule

```python
# ✅ correct — %pip cell is cell 0, before all imports and setup
# Cell 0
%pip install databricks-zerobus-ingest-sdk -q

# Cell 1 — WorkspaceClient setup
import os
from databricks.sdk import WorkspaceClient
...
```

```python
# ❌ wrong — %pip after an import triggers a restart that discards the import
import os
%pip install databricks-zerobus-ingest-sdk -q  # kernel restarts; os import is lost
```

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

## 4. Full entrypoint templates

Two guard variants — pick based on where the file runs:

### 4a. Notebook-portable guard — `if "spark" not in vars():`

Use when the same file must run unchanged in a **workspace notebook** (where `spark` is already injected) **and** locally via Connect.

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

### 4b. Standalone script guard — `if __name__ == "__main__":`

Use when the file is run **directly as a script** (`python myscript.py`) and may also be **imported as a module** without triggering Connect setup. The business-logic functions live at the top level; Connect setup is isolated in `__main__`.

```python
#!/usr/bin/env python3
"""One-line description of what this script does."""
from pyspark.sql import SparkSession


def run_something(spark: SparkSession) -> None:
    df = spark.read.table("samples.nyctaxi.trips")
    df.show(5)


def run_dbutils(dbutils) -> None:
    print(dbutils.fs.ls('/'))


if __name__ == "__main__":
    from databricks.connect import DatabricksSession
    from databricks.sdk import WorkspaceClient

    spark = DatabricksSession.builder.getOrCreate()   # uses DEFAULT profile
    run_something(spark)

    w = WorkspaceClient()        # also reads DEFAULT profile
    dbutils = w.dbutils          # preferred — full feature set (fs, secrets, widgets, jobs…)
    run_dbutils(dbutils)
```

`WorkspaceClient().dbutils` is preferred over `DBUtils(spark)` — see section 7 for the comparison.

**Expected warning** — harmless, emitted once per process when gRPC is active alongside `WorkspaceClient`:

```
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
I0000 … fork_posix.cc:71] Other threads are currently calling into gRPC, skipping fork() handlers
```

Key differences vs 4a:

| | `if "spark" not in vars():` | `if __name__ == "__main__":` |
|---|---|---|
| Works in workspace notebook | ✅ | ❌ (guard never runs) |
| Works as `python script.py` | ✅ | ✅ |
| Safe to `import` as module | ⚠️ (Connect setup runs at import) | ✅ (setup stays in `__main__`) |
| Idiomatic Python pattern | – | ✅ |

Use **4b** for pure scripts and CLI tools. Use **4a** for code that must be notebook-portable.

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

On the **workspace** the notebook's own directory is always on `sys.path`. In **Cursor / VS Code** the kernel CWD is the repo root (workspace folder).

> **Known VS Code Jupyter bug — do not rely on `jupyter.notebookFileRoot`:**
> Setting `"jupyter.notebookFileRoot": "${fileDirname}"` is silently ignored and the CWD stays at `${workspaceFolder}`. Reported as a regression in extension 2024.4.0, confirmed still broken in 2025.9.1 on both VS Code 1.116.0 and Cursor 3.0.12.
>
> - Original closed-stale report: [vscode-jupyter#15649](https://github.com/microsoft/vscode-jupyter/issues/15649)
> - Active bug report (filed Apr 2026, labelled `bug`): [vscode-jupyter#17376](https://github.com/microsoft/vscode-jupyter/issues/17376)
>
> **Why #15649 was not fixed:** The maintainer could not reproduce locally and asked for verbose logs. Reporters went quiet; auto-closed by bot after 60 days and locked. Cursor bundles the same Jupyter extension and inherits the bug.

Use a two-location probe in the notebook instead — checks CWD (workspace) then CWD/notebooks (Cursor):

```python
import sys
from pathlib import Path

# CWD = notebook dir on workspace; CWD = repo root in Cursor (notebookFileRoot bug).
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

| Situation | kernel CWD | `import zbhelper` works? |
|---|---|---|
| Workspace notebook | notebook dir (`notebooks/`) | ✅ always |
| Cursor / VS Code (with `${fileDirname}`) | notebook dir (`notebooks/`) | ✅ |
| pytest (via `conftest.py`) | repo root | ✅ |

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
# Option A — WorkspaceClient (PREFERRED — full feature set)
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()            # reads DEFAULT profile; add profile="DEV" for named profiles
dbutils = w.dbutils              # fs, secrets, widgets, jobs, notebook, library…

# Option B — DBUtils constructor (paired with DatabricksSession; fewer features)
from pyspark.dbutils import DBUtils
dbutils = DBUtils(spark)         # requires an active spark session; missing some dbutils namespaces

# Option C — SDK runtime shim (works when a Connect session is active)
from databricks.sdk.runtime import dbutils
```

**Option A (`WorkspaceClient().dbutils`) is preferred** — it exposes the full `dbutils` surface (all namespaces) and works independently of whether a Spark session is active. `DBUtils(spark)` (Option B) is missing some namespaces that `WorkspaceClient().dbutils` exposes.

Option A also works for scripts that do not use Spark at all:

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
secret = w.dbutils.secrets.get(scope="my-scope", key="my-key")
files  = w.dbutils.fs.ls("/Volumes/my_catalog/my_schema/")
```

### Reading secrets in a `.py` file

**Preferred — `w.dbutils.secrets.get()`** returns the plaintext string directly, no decoding step:

```python
import json
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Single string secret
token = w.dbutils.secrets.get(scope="my-scope", key="my-token")

# JSON blob secret — parse into a dict
config = json.loads(w.dbutils.secrets.get(scope="my-scope", key="my-config"))
# e.g. config["ZEROBUS_OAUTH_SECRET"], config["ZEROBUS_SERVER_ENDPOINT"], …
```

Example output when the secret value is a JSON object:

```
Secret: {
  "ZEROBUS_OAUTH_SECRET": "dose31354910…",
  "ZEROBUS_TABLE_NAME": "air_quality_otel",
  "DATABRICKS_WORKSPACE_URL": "https://e2-demo-field-eng.cloud.databricks.com",
  "ZEROBUS_SERVER_ENDPOINT": "https://1444828305810485.zerobus.us-west-2.cloud.databricks.com"
}
```

**Avoid `w.secrets.get_secret()`** — its `.value` is a base64-encoded `str`. Calling `.decode('utf-8')` on it raises `AttributeError: 'str' object has no attribute 'decode'` because it is already a `str`, not `bytes`. If you must use it:

```python
import base64
secret_response = w.secrets.get_secret(scope="my-scope", key="my-key")
actual = base64.b64decode(secret_response.value).decode('utf-8')   # extra step; avoid
```

---

## 8. Why this matters

| Pattern | Workspace notebook | Local .py (Connect) |
|---|---|---|
| Bootstrap guard + pass as args | ✅ | ✅ |
| Hard-coded `DatabricksSession` inside module | ❌ Creates second session | ⚠️ Works but wrong |
| `SparkSession.builder` inside module | ❌ Local/disconnected session | ❌ Local/disconnected session |
| `spark`/`dbutils` assumed as globals in module | ❌ NameError if called from `.py` | ❌ NameError if module imported first |
| `DBUtils(spark)` for `dbutils` | ⚠️ Limited namespaces | ⚠️ Limited namespaces |
| `WorkspaceClient().dbutils` | ✅ Full feature set | ✅ Full feature set |

---

## 9. Code promotion ladder — from inline notebook to reusable library to multi-workspace CLI

Start code in an inline demo notebook. Promote it to a reusable library. The same library then powers a thin driver notebook **and** a CLI script that can run across multiple workspaces. Do this in order so each step is easy.

```
Step 1: inline demo notebook          zerobus_grpc_http.ipynb
           ↓  extract functions
Step 2: reusable library              zbhelper/ingest_benchmark.py
                                      zbhelper/ingest_v2.py
                                      zbhelper/<one .py per concern>
           ↓  import library
Step 3: thin driver notebook          zerobus_benchmark_driver.ipynb
           ↓  same import, add --profile
Step 4: multi-workspace CLI script    benchmarks/run_benchmark.py
```

### Step 1 — inline demo notebook

Write all logic directly in notebook cells. No imports from an internal library. Keep it self-contained so it can be cloned into any workspace and run immediately.

Design cells so each logical operation is one cell and one function call — this makes extraction to a library function in Step 2 mechanical.

```python
# Cell 4a — inline in demo notebook
def ingest_singles(spark, dbutils, config, n=10):
    ...
    return metrics

metrics = ingest_singles(spark, dbutils, config)
display(metrics)
```

### Step 2 — reusable library

Create one `.py` file per concern under a helper package co-located with the notebooks (e.g. `notebooks/zbhelper/`). Each function mirrors a notebook cell, with the cell number documented:

```python
# zbhelper/ingest_benchmark.py
"""
Each function mirrors one notebook cell so notebooks can be retrofitted
to call these functions directly.

Cell 4a  → ingest_singles_grpc_sync / _http_sync / _http_async
Cell 4b  → ingest_batch_and_close_grpc_sync / _http_async
Cell 4c  → poll_visibility
Cell 4d  → print_metrics
"""

def ingest_singles(spark, dbutils, config, n: int = 10):
    # All Databricks runtime context comes from the caller.
    # Never import DatabricksSession or WorkspaceClient here.
    ...
    return metrics
```

Rules for library `.py` files:
- All runtime context (`spark`, `dbutils`, `w`) passed as arguments — never created inside
- No `DatabricksSession`, `WorkspaceClient`, or `dbutils` imports at module level
- Type-hint `spark` as `SparkSession` (the portable supertype)
- One `.py` per concern — keeps files small and independently testable

### Step 3 — thin driver notebook

The driver notebook imports from the library and only handles orchestration: widgets, iteration loops, result aggregation. No business logic in cells.

```python
# Driver notebook cell — imports library, calls functions
import zbhelper.ingest_benchmark as zb   # import as module, not from … import

results = zb.ingest_singles(spark, dbutils, config, n=iterations)
zb.print_metrics(results)
```

Use `import module as alias` (not `from module import fn`) so `%autoreload 2` — if used — updates the alias rather than a stale copied name. See section 2.

The driver notebook stays simple enough to run in a single Databricks workspace for demos. Multi-workspace testing moves to Step 4.

### Step 4 — multi-workspace CLI script

Import the same library. Add Typer `--profile` to target any workspace from `~/.databrickscfg`. `DatabricksSession` and `WorkspaceClient` are created in `__main__` and passed into library functions — the library code does not change.

```python
# benchmarks/run_benchmark.py
import typer
import zbhelper.ingest_benchmark as zb
from pyspark.sql import SparkSession


def run(spark: SparkSession, w, config: dict, iterations: int) -> None:
    results = zb.ingest_singles(spark, w.dbutils, config, n=iterations)
    zb.print_metrics(results)


def main(
    profile: str = typer.Option("DEFAULT", help="~/.databrickscfg profile name."),
    iterations: int = typer.Option(10, help="Number of rows to ingest."),
) -> None:
    from databricks.connect import DatabricksSession
    from databricks.sdk import WorkspaceClient

    spark = DatabricksSession.builder.profile(profile).getOrCreate()
    w = WorkspaceClient(profile=profile)
    config = _load_config(w)

    run(spark, w, config, iterations)


if __name__ == "__main__":
    typer.run(main)
```

```bash
python run_benchmark.py --profile DEFAULT        # AWS workspace
python run_benchmark.py --profile azurefe        # Azure workspace
python run_benchmark.py --profile gcpdev         # GCP workspace
```

### What Step 1 must get right for Steps 2–4 to be easy

| Rule | Why |
|---|---|
| Each cell is one logical operation, callable as one function | Extraction to library is copy-paste |
| All runtime context (`spark`, `dbutils`, `w`) passed as args | Library functions are portable without changes |
| No `DatabricksSession` / `WorkspaceClient` inside notebook helper functions | Same functions work in driver notebook and CLI |
| One concern per cell / per `.py` file | Individual functions are independently testable with pytest |
| Notebook cell docstring names the library function it will become | Library and notebook stay in sync |
