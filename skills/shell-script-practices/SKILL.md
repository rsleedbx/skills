---
name: shell-script-practices
description: >-
  Bash script conventions — run() / run_log() ("$@" never "${*}"); abort with kill -INT $$;
  prerequisites for-bin + command -v loop; ${VAR:?msg}, ${VAR:+ …}, argv arrays + run "${cmd[@]}",
  inline single-command bodies; always use ${var} braces in echo/string contexts to guard against
  non-ASCII byte bleed into variable names; never use 2>/dev/null on data-fetching commands; set -u
  in all scripts including sourced helpers; safe for-loops over arrays not unquoted vars; capture
  pipeline output before piping to avoid swallowed exit codes; use --output none not >/dev/null for
  az CLI; extract shared helpers instead of duplicating across scripts; validate DB credentials
  before use and reset only on failure. For REST API CLI wrappers (az, aws, gcloud, databricks,
  mysql, psql) — CMD_EXIT_ON_ERROR / CMD_TIMEOUT / CMD_OUT_SUFFIX / CMD_MASK_SECRETS — see the
  shell-cmd-wrapper skill. Use when writing or refactoring any .sh scripts, CLI wrappers, or shell
  automation.
---

# Shell Script Practices

## Abort: prefer `kill -INT $$` over `exit 1`

Use `kill -INT $$` instead of `exit 1` after printing a manual error. If the file is ever `source`d into an interactive shell, `exit 1` closes the shell; `kill -INT $$` sends SIGINT (script cancels, session stays).

```bash
if ! "${create[@]}"; then
  echo "error: create-bucket failed" >&2
  kill -INT $$
fi
```

## Check prerequisites — one `command -v` loop

Put all required binaries in one loop. Never copy-paste `if ! command -v foo` per tool:

```bash
check_prerequisites() {
  local bin
  for bin in databricks jq aws; do
    command -v "$bin" >/dev/null 2>&1 || {
      echo "error: '${bin}' not found" >&2
      kill -INT $$
    }
  done
}
```

- One global prerequisite function per script; only split when subcommands need disjoint toolchains.
- Do not add `if has_jq; then … else …` branches — require the tool instead.
- `usage`/`help` paths must not invoke tools.

## `set -u` in every script — including sourced helpers

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Add to **every** shell file, including library files. A script with `set -u` sourcing a library without it creates a two-tier safety net — the library is the dangerous tier.

Guard optional variables with `${var:-}` to make library files `-u`-safe by construction.

## Never swallow errors with `2>/dev/null` on data-fetching commands

`2>/dev/null` on a command whose output you depend on silently turns any error into empty data. Full anti-patterns, rules table, and "verify CLI syntax before suppressing" guidance: see [error-and-io-patterns.md](error-and-io-patterns.md).

**Quick rules:**
- Existence check (only care about exit code) → `cmd &>/dev/null`
- Data fetch (you use the output) → never `2>/dev/null`
- Azure CLI commands → `--output none`, not `>/dev/null`

## Always use `${var}` braces in echo/string contexts

Without braces, bash's locale-aware `isalpha()` can include the first byte of a multi-byte character (e.g. `0xE2` from an en-dash) as part of the variable name, silently reading the wrong variable:

```bash
# BAD — en-dash byte 0xE2 directly follows $start_ip → wrong variable name
echo "Rule for $start_ip–$end_ip"

# GOOD — closing brace terminates the name
echo "Rule for ${start_ip} - ${end_ip}"
```

Required when adjacent to em/en dashes, ellipses, or any non-ASCII literal. Always use `${var}` in echo/printf strings.

## Safe `for` loops — arrays over unquoted variables

Never iterate over an unquoted variable expansion:

```bash
# BAD
for _key in $_SECRET_KEYS; do …; done

# GOOD — when list comes from a command
while IFS= read -r _key; do …; done <<< "$_SECRET_KEYS"

# GOOD — for a fixed set of optional variables
_dns_targets=()
[[ -n "${SQL_DNS_MYSQL:-}" ]] && _dns_targets+=("$SQL_DNS_MYSQL")
[[ -n "${SQL_DNS_PG:-}"    ]] && _dns_targets+=("$SQL_DNS_PG")
for _extra in "${_dns_targets[@]}"; do _add_dns "$_extra"; done
```

## Capture pipeline output before piping

```bash
# BAD — az failure is swallowed; grep and head still exit 0
INSTALL_STATE=$(az vm run-command invoke … | grep "RUNNING" | head -1)

# GOOD — capture first, then process
_raw=$(az vm run-command invoke …)
INSTALL_STATE=$(grep "RUNNING" <<< "$_raw" | head -1)
```

For detailed explanation see [error-and-io-patterns.md](error-and-io-patterns.md).

## Validate credentials before use — reset only on failure

Try the stored credential first. Only reset if login fails. Always re-verify after reset.

Full pattern with code and rules: see [error-and-io-patterns.md](error-and-io-patterns.md).

## Optional CLI flags — `${var:+word}`

```bash
# Passes -p only when DATABRICKS_PROFILE is set and non-empty
databricks ${DATABRICKS_PROFILE:+ -p "${DATABRICKS_PROFILE}"} api get "${path}" -o json
```

Note the **leading space** inside `word` so tokens stay separate. Do not use `:-` — that substitutes a default, which is wrong semantics for "optional flag."

## Required env var helper (SIGINT on unset)

Use a small helper instead of `${var:?msg}` when you need SIGINT rather than exit 1:

```bash
_require_nonempty() {
  local val=$1; shift
  [[ -z "${val}" ]] && { echo "error: $*" >&2; kill -INT $$; }
}
_require_nonempty "${BUCKET_NAME:-}" "set BUCKET_NAME"
```

## `run()` / `run_log()` — `"$@"` never `"${*}"`

```bash
run()     { "$@"; }
run_log() { printf '%q ' "$@" >&2; echo >&2; "$@"; }
```

`"${*}"` joins parameters into one string — do not use it. Use `run "${cmd[@]}"` with an argv array for conditional flags.

## Build CLI as an argv array

```bash
local -a create=(aws s3api create-bucket --bucket "${BUCKET_NAME}" --region "${AWS_REGION}")
[[ "${AWS_REGION}" != "us-east-1" ]] && \
  create+=(--create-bucket-configuration "LocationConstraint=${AWS_REGION}")
run "${create[@]}"
```

## Extract shared helpers — never duplicate functions

A function used in 2+ scripts → move to a shared helper file and `source` it. Duplicated functions drift independently — a bug fix in one copy is missed in all others.

```bash
source "$(dirname "${BASH_SOURCE[0]}")/uc-connection-helpers.sh"
```

Rules and a complete helper extraction example: see [script-structure-examples.md](script-structure-examples.md).

## Inline a single command

Do not add `cmd_foo() { only_one_pipeline; }` when the body is a single pipeline used once — put it inline in the `case` arm. Keep a named function when the body is non-trivial, reused, or needs a stable name for testing.

## Cloud CLI wrappers — use `CMD`, not `$()`

See the `shell-cmd-wrapper` skill for the full `CMD` implementation. Key principle: never use `$()` or `2>/dev/null` for REST API calls (`az`, `aws`, `gcloud`, `databricks`, `mysql`, etc.) — they silently swallow errors and can hang on API outages.

## Canonical script shape and examples

For the full script skeleton (`run` + `run_log` + `check_prerequisites` + `main` + `BASH_SOURCE` guard), argv array examples, optional flag examples, and shared helper extraction: see [script-structure-examples.md](script-structure-examples.md).
