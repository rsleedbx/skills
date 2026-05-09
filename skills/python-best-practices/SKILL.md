---
name: python-best-practices
description: Apply Python best practices when writing or refactoring code: single source of truth, no duplicate code, no backward-compatibility complexity on new projects; use pytest for tests and maintain complete test coverage as code is written. Use when editing Python, adding constants, refactoring modules, or adding tests.
---

# Python Best Practices (Clean Code)

## Mandatory tool chain — run automatically after every Python edit

After writing or editing **any** Python file, run this sequence without being asked. Do not consider the change done until all four steps pass with 0 errors.

```
1. ruff format <edited-file>        ← formatting
2. ruff check --fix <edited-file>   ← auto-fix lint
3. ruff check <edited-file>         ← remaining lint  (must be 0 errors)
4. pyright <edited-file>            ← type errors     (must be 0 errors)
```

If errors remain after `--fix`, fix them manually and rerun steps 3–4. Repeat until clean.

**Never leave a Python edit with ruff or pyright errors.** Errors are blocking — not warnings to address later.

### Finding the binaries

`ruff` and `pyright` are installed into a venv, not the system `PATH`. Discover them before running:

```bash
# Find ruff (checks common venv locations)
RUFF=$(command -v ruff 2>/dev/null \
  || ls .venv*/bin/ruff .venv/bin/ruff 2>/dev/null | head -1 \
  || echo "")
if [[ -z "$RUFF" ]]; then
  echo "ruff not found — install with: python -m pip install ruff"; exit 1
fi

# Find pyright
PYRIGHT=$(command -v pyright 2>/dev/null \
  || ls .venv*/bin/pyright .venv/bin/pyright 2>/dev/null | head -1 \
  || echo "")
if [[ -z "$PYRIGHT" ]]; then
  echo "pyright not found — install with: python -m pip install pyright"; exit 1
fi

# Then run the chain
$RUFF format path/to/file.py
$RUFF check --fix path/to/file.py
$RUFF check path/to/file.py
$PYRIGHT path/to/file.py
```

In the **Shell tool**, always use the discovered path — never assume `ruff` or `pyright` are on `PATH`.

---

When writing or changing Python in this project, follow these rules. Assume the project is **new** — do not add backward-compatibility layers unless explicitly required.

## Always use `python -m pip`, never bare `pip`

Always invoke pip as `python -m pip`. Never use bare `pip` or `pip3`.

```bash
# correct
python -m pip install ruff
python -m pip install -r requirements.txt
.venv/bin/python -m pip install -r requirements.txt

# wrong
pip install ruff
pip3 install ruff
.venv/bin/pip install -r requirements.txt
```

Bare `pip` resolves to whichever pip is first on `PATH`. `python -m pip` guarantees the pip belonging to the active interpreter.

### Canonical venv setup sequence

Three steps, always in this order — never skip any:

```bash
python -m venv .venv
.venv/bin/python -m ensurepip --upgrade
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
```

| Step | Why it cannot be skipped |
|------|--------------------------|
| `python -m ensurepip --upgrade` | pip is not guaranteed to exist. Ubuntu/Debian system Python, CI images, and venvs created with `--without-pip` have no pip. `ensurepip` bootstraps from Python's bundled copy — no network needed. |
| `python -m pip install --upgrade pip` | The bundled copy is pinned to the Python release date — potentially months old. Stale pip silently resolves wrong versions or fails to build wheels. |
| `python -m pip install -r requirements.txt` | Install deps only after pip itself is current. |

### `ensurepip` is a module, not a command

```bash
# correct
python -m ensurepip --upgrade
.venv/bin/python -m ensurepip --upgrade

# wrong — always fails: bash: ensurepip: command not found
ensurepip --upgrade
```

### If `python -m ensurepip` itself fails

On **Debian / Ubuntu** the module is stripped from the system Python. Fix:

```bash
sudo apt-get install python3-venv python3-pip
python -m ensurepip --upgrade
python -m pip install --upgrade pip
```

On **macOS (Homebrew)** and **Windows**, `ensurepip` ships with Python — if missing, reinstall Python.

**For named venvs** the same three steps apply:

```bash
python -m venv .venv_test
.venv_test/bin/python -m ensurepip --upgrade
.venv_test/bin/python -m pip install --upgrade pip
.venv_test/bin/python -m pip install -r requirements-test.txt
```

---

## Single Source of Truth

- Define each concept (valid options, config keys, formats) in **one place**.
- Derive everything else from that definition.

```python
class SchemaSource(str, Enum):
    YDATA = "ydata"
    SDV = "sdv"
    MYSQL = "mysql"

VALID = frozenset(s.value for s in SchemaSource)  # derived, not duplicated
```

## No Duplicate Code (DRY)

- Do not repeat the same logic or list of options in multiple places.
- Adding a new option should require editing **one** place.
- Do not introduce a one-line `def` whose only job is to call another single function once — inline at the call site.

## Prerequisites and environment (check once)

- **Validate prerequisites once** at startup, not inside every branch.
- **One global bootstrap** when subcommands share the same deps.
- **Prefer required deps** over optional dual code paths.
- Probe optional deps **once** at startup; reuse a flag — never call `shutil.which` repeatedly.

## Prefer Enums for Fixed Sets

- Use `Enum` (or `str, Enum`) for fixed sets of related string values.
- Avoid parallel string constants that mirror enum values.
- Normalize `Enum | str` input once at the boundary; use the enum internally.

## No Backward-Compat Complexity on New Projects

- No legacy aliases, deprecated paths, or "support both old and new" patterns unless explicitly requested.
- Update call sites and tests to the new API instead of keeping old names.

## Never swallow exceptions silently

| Situation | Pattern |
|-----------|---------|
| Expected, recoverable | Catch specific exception; log it; return a documented default |
| Unexpected | Let it propagate — or catch only to add context and re-raise |
| Optional operation | Catch specific exception; log at `WARNING`; document fallback |
| Cleanup | Use `finally`; do not suppress the original exception |

```python
# GOOD — specific, logged, documented
def get_host_fqdn(secret: dict) -> str | None:
    try:
        return secret["host_fqdn"]
    except KeyError:
        logger.warning("secret missing 'host_fqdn' — skipping: %s", secret.get("key"))
        return None

# GOOD — add context, re-raise
def load_config(path: str) -> dict:
    try:
        with open(path) as f:
            return json.load(f)
    except json.JSONDecodeError as exc:
        raise ValueError(f"invalid JSON in '{path}'") from exc
```

When a broad catch is unavoidable, always log with traceback:

```python
for item in items:
    try:
        process(item)
    except Exception:
        logger.exception("failed to process %r — skipping", item)
```

## Normalize Early, Use Strong Types

- Convert string input to the canonical type at the boundary.
- Use strong types everywhere inside the module.
- Raise `ValueError` with valid options when input is invalid.

---

## Use `@dataclass(frozen=True)` where possible

Prefer frozen dataclasses over raw dicts and positional tuples whenever two or
more related values travel together. The pattern applies in four concrete cases:

### 1. Repeated dict-extraction (blob → fields)

When the same `.get("key")` calls appear in multiple functions for the same
blob shape, extract once into a dataclass with a `from_blob` factory:

```python
# Instead of repeating these 5 lines in every function that reads a conn_blob:
db_type  = blob.get("db_type", "")
host     = blob.get("host_fqdn", "")
cloud    = blob.get("cloud") or {}
provider = cloud.get("provider", "local")
routing  = blob.get("routing") or {}

# Define once:
@dataclass(frozen=True)
class _ConnBlobMeta:
    db_type:  str
    host:     str
    provider: str
    routing:  dict

    @classmethod
    def from_blob(cls, blob: dict) -> "_ConnBlobMeta":
        cloud = blob.get("cloud") or {}
        return cls(
            db_type  = blob.get("db_type", ""),
            host     = blob.get("host_fqdn", ""),
            provider = cloud.get("provider", "local"),
            routing  = blob.get("routing") or {},
        )
```

### 2. Typed fallback chain (blob → cfg)

When teardown reads values from a saved blob then falls back to `cfg`, use a
typed factory so the fallback is written once:

```python
@dataclass(frozen=True)
class AwsCtx:
    region:  str
    profile: str
    vpc_id:  str

def aws_ctx(blob: dict | str | None, cfg: Any) -> AwsCtx:
    c = blob_cloud_ctx(blob)
    return AwsCtx(
        region  = c.get("region")  or getattr(cfg.aws, "region",  ""),
        profile = c.get("profile") or getattr(cfg.aws, "profile", ""),
        vpc_id  = c.get("vpc_id")  or getattr(cfg.aws, "vpc_id",  ""),
    )

# Teardown becomes one line instead of four:
ctx = aws_ctx(lsec.read(scope, _server), cfg)
session = aws_session(ctx.profile or None)
rds = session.client("rds", region_name=ctx.region)
```

### 3. Internal tuple returns

When a private helper returns 2–4 positional values and both callers unpack
them, replace with a frozen dataclass. Positional tuples are silently wrong
when types match; named attributes are typo-safe and self-documenting:

```python
# Before — positional, easy to swap
def _cred_from_blob(...) -> tuple[str, str, str, str]:
    return user, password, catalog, host

user, password, catalog, host = _cred_from_blob(...)

# After — named, impossible to swap silently
@dataclass(frozen=True)
class _RouteCred:
    user: str; password: str; catalog: str; host: str

cred = _cred_from_blob(...)
url  = f"...{cred.user}:{cred.password}@{cred.host}/{cred.catalog}"
```

### 4. Repeated boilerplate blocks (load-or-generate + persist)

When the same load-or-generate-then-persist logic appears in multiple modules,
extract it to a helper in the module that owns the state:

```python
# Instead of this 4-line block in every */lfc.py:
lfc_pws = {k: state.get(f"{epoch}/{k}_password") or gen_password() for k in LFC_USERS}
for k, pw in lfc_pws.items():
    state.put(f"{epoch}/{k}_password", pw)

# Define once in state.py:
def materialize_lfc_passwords(state: State, epoch: int) -> dict[str, str]: ...

# Every lfc.py becomes one line:
lfc_pws = materialize_lfc_passwords(state, epoch)
```

### When NOT to use a dataclass

- Single call site with ≤3 fields — inline or use a local variable instead.
- The object crosses a JSON wire format boundary — keep it a `dict` or `TypedDict`;
  use a factory (`from_dict`) for construction but `asdict()` or manual `.get()`
  for serialization.
- A one-field wrapper — use an alias or `NewType` instead.

---

## Pytest and Test Coverage

- Use pytest. Tests in `tests/`; mirror package layout.
- Write tests as you write code, not after.
- One test file per module. `class TestX:` for grouping; `@pytest.mark.parametrize` for multiple inputs.
- Test success paths, invalid input (`pytest.raises`), and edge cases.
- Use `@pytest.fixture` for shared setup — no copy-pasted test bodies.

### Running pytest — always use `.venv_test`

```bash
# One-time setup
python -m venv .venv_test
.venv_test/bin/python -m ensurepip --upgrade
.venv_test/bin/python -m pip install --upgrade pip
.venv_test/bin/python -m pip install -r requirements-test.txt

# Run every time — must be 0 skipped
.venv_test/bin/python -m pytest tests/ -q
```

Fix skipped tests before considering the change done. To run against a real Databricks workspace use `.venv_3_11`.

---

## Static type checking — Pyright

Run after **every** Python file change. 0 errors required before a change is done.

### Setup

```bash
python -m ensurepip --upgrade
python -m pip install --upgrade pip
python -m pip install pyright
```

`pyrightconfig.json` at the root of the source tree (same directory scripts run from):

```json
{
  "venvPath": "..",
  "venv": ".venv_3_11",
  "pythonVersion": "3.11",
  "typeCheckingMode": "standard",
  "exclude": [".venv", ".venv_test", ".venv_3_11", "**/__pycache__"]
}
```

**`venvPath` + `venv` are mandatory.** Without them, Pyright cannot resolve any installed package and every third-party import fires `reportMissingImports` as a false positive. This floods the output with noise, making it impossible to distinguish real errors from missing stubs.

`venvPath` is the directory that *contains* the venv folder; `venv` is the folder name:
- venv at `src/.venv_3_11` → `"venvPath": "."`, `"venv": ".venv_3_11"`
- venv at repo root `.venv_3_11`, config in `src/` → `"venvPath": ".."`, `"venv": ".venv_3_11"`

### Running

```bash
# Use the discovered binary (see "Finding the binaries" above)
$PYRIGHT path/to/file.py   # after each edit
$PYRIGHT                   # whole project
```

### Common errors

| Error | Fix |
|---|---|
| `reportMissingTypeArgument` | Add type params: `list[str]`, `dict[str, Any]` |
| `reportUnknownVariableType` | Annotate: `x: str = ...` |
| `reportOptionalMemberAccess` | Guard: `if x is not None:` |
| `reportReturnType` | Ensure all code paths return the declared type |
| `reportMissingImports` | Check venvPath/venv in pyrightconfig.json first; only then install stubs |

### Module naming — never shadow installed packages

**Never name a `.py` file the same as an installed package** that is on `sys.path`. Python resolves the local file first, turning the installed package into a broken namespace.

```
src/azure.py      ← shadows azure-identity → from azure.identity import ... fails at runtime
src/google.py     ← shadows google-auth    → from google.auth import ... fails at runtime
src/requests.py   ← shadows requests       → import requests fails at runtime
```

Pyright catches this **only when `pyrightconfig.json` has `venvPath`+`venv` configured**. Without the venv, Pyright already reports `reportMissingImports` for all installed packages, so the shadowing error looks identical to the pre-existing noise — the real error is invisible.

Safe naming when you need a helper module for a cloud:
- `azure.py` → `az_helpers.py`
- `google.py` → `gcp_helpers.py`
- `databricks.py` → `dbx_helpers.py`

### Type annotation conventions

- All function signatures must have parameter and return type annotations.
- Use `str | None`, not `Optional[str]`.
- Use `list[str]`, `dict[str, Any]` (lowercase generics), not `List`, `Dict`.
- Use `from __future__ import annotations` at the top of every file.

---

## Linting and formatting — Ruff

Ruff replaces flake8, Black, and isort in one tool. Run after **every** Python file change. 0 errors required before a change is done.

### Setup

```bash
python -m pip install ruff
```

`ruff.toml` at project root:

```toml
line-length = 100

[lint]
select = [
  "E", "W",   # pycodestyle — PEP 8 errors and warnings
  "F",        # Pyflakes — undefined names, unused imports
  "I",        # isort — import ordering
  "UP",       # pyupgrade — modern Python syntax
  "B",        # flake8-bugbear — likely bugs and design issues
  "SIM",      # flake8-simplify — unnecessary complexity
]
ignore = [
  "E501",     # line too long — ruff format handles this
]

[format]
quote-style = "double"
indent-style = "space"
```

### Running

```bash
# Use the discovered binary (see "Finding the binaries" above)
$RUFF format path/to/file.py        # format (replaces Black)
$RUFF check --fix path/to/file.py   # auto-fix lint issues
$RUFF check path/to/file.py         # remaining errors — must be 0
```

### What each rule set catches that Pyright misses

| Rule set | Example catches |
|---|---|
| `F` | Unused imports, undefined names, `import *` |
| `I` | Wrong import order (stdlib → third-party → local) |
| `UP` | `Optional[str]` → `str \| None`, old `%`-formatting, deprecated stdlib usage |
| `B` | Mutable default args `def f(x=[]):`, bare `except:`, `assert` in non-test code |
| `SIM` | `if x == True:`, needless `else` after `return`, `len(x) == 0` → `not x` |

---

## Python packages and VS Code / Cursor extensions

Some packages install Jupyter Lab extensions with `package.json` files inside `.venv`. Red Hat Dependency Analytics emits warnings like `component analysis error: package.json requires a lock file`. This is not a vulnerability — the extension skipped analysis of a read-only artifact.

**Known packages that cause this:** `ipywidgets`.

**Fix:**
```json
"redhat.dependency-analytics.excludedFolders": [".venv"]
```

---

## CLI — use Typer, not argparse

Typer is the standard for all CLI scripts. It uses the same type-hint pattern as FastAPI, so the same function works as a CLI command and as an importable FastAPI endpoint — zero code duplication.

```bash
python -m pip install typer
```

### Structure — two layers

```python
# Layer 1: data function — returns a dict, importable by FastAPI
def get_az_identity(tenant_id: str | None = None) -> dict:
    ...
    return {"userPrincipalName": ..., "displayName": ...}

# Layer 2: Typer CLI — formats and prints the dict
app = typer.Typer()

@app.command()
def whoami(
    tenant_id: Annotated[str | None, typer.Option(help="Azure AD tenant")] = None,
) -> None:
    data = get_az_identity(tenant_id=tenant_id)
    typer.echo(typer.style("✓", fg=typer.colors.GREEN) + f"  {data['displayName']}")
    raise typer.Exit(0)

if __name__ == "__main__":
    app()
```

FastAPI imports only layer 1 — never the CLI layer:

```python
from mymodule import get_az_identity   # data function only

@api.get("/identity/az")
def api_az(tenant_id: str | None = None) -> dict:
    return get_az_identity(tenant_id=tenant_id)
```

### Key conventions

| Pattern | Use |
|---------|-----|
| `Annotated[str \| None, typer.Option(help="...")]` | All CLI options — keeps type and metadata together |
| `typer.style(text, fg=typer.colors.GREEN/RED/YELLOW)` | Colored output — never raw ANSI codes |
| `raise typer.Exit(0)` / `raise typer.Exit(1)` | Exit codes — never `sys.exit()` |
| `typer.echo(..., err=True)` | Errors to stderr |
| `--only` repeated flag | `list[str] \| None` with `typer.Option()` |

### `--only` repeating option pattern

```python
@app.command()
def run(
    only: Annotated[list[str] | None, typer.Option(
        help="Run only these targets (repeat for multiple: --only az --only dbx)"
    )] = None,
) -> None:
    targets = set(only) if only else {"az", "dbx", "gcp", "aws"}
```

---

## Checklist When Refactoring

- [ ] **Ruff: 0 errors** — `ruff format <file>` → `ruff check --fix <file>` → `ruff check <file>`.
- [ ] **Pyright: 0 errors** — `pyright <edited-files>`. Confirm `pyrightconfig.json` has `venvPath`+`venv` so errors are real, not venv-resolution noise.
- [ ] **No module name shadows an installed package** — never name a `.py` file `azure`, `google`, `requests`, `databricks`, etc. Use `az_helpers`, `gcp_helpers`, `dbx_helpers` instead.
- [ ] No bare `pip` or `pip3` — all installs use `python -m pip`.
- [ ] Any new venv: `python -m ensurepip --upgrade` → `python -m pip install --upgrade pip` → `python -m pip install -r ...`.
- [ ] Prerequisites checked once at entry; required deps preferred over optional dual paths.
- [ ] Single definition of valid options/formats (enum or one list/dict); validation derived, not duplicated.
- [ ] No redundant aliases.
- [ ] Call sites and tests updated to new API; no backward-compat branches.
- [ ] Invalid inputs rejected at the boundary with a clear error.
- [ ] Existing tests still pass; new cases added to maintain coverage.
- [ ] CLI uses Typer, not argparse — data functions return dicts; Typer layer formats output.
- [ ] No `sys.exit()` in Typer commands — use `raise typer.Exit(code)` instead.
