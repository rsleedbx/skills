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

`pyrightconfig.json` at project root:

```json
{
  "pythonVersion": "3.11",
  "typeCheckingMode": "standard",
  "exclude": [".venv", ".venv_test", ".venv_3_11", "**/__pycache__"]
}
```

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
| `reportMissingImports` | Install the package or add a stub |

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

## Checklist When Refactoring

- [ ] **Ruff: 0 errors** — `ruff format <file>` → `ruff check --fix <file>` → `ruff check <file>`.
- [ ] **Pyright: 0 errors** — `pyright <edited-files>`.
- [ ] No bare `pip` or `pip3` — all installs use `python -m pip`.
- [ ] Any new venv: `python -m ensurepip --upgrade` → `python -m pip install --upgrade pip` → `python -m pip install -r ...`.
- [ ] Prerequisites checked once at entry; required deps preferred over optional dual paths.
- [ ] Single definition of valid options/formats (enum or one list/dict); validation derived, not duplicated.
- [ ] No redundant aliases.
- [ ] Call sites and tests updated to new API; no backward-compat branches.
- [ ] Invalid inputs rejected at the boundary with a clear error.
- [ ] Existing tests still pass; new cases added to maintain coverage.
