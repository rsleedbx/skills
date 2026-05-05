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

## Monitoring long-running shell commands — use `block_until_ms: 0` + Await, never `nohup … &`

**Root cause pattern to avoid:** Running `nohup <cmd> > /tmp/logfile &` inside a Shell tool call, then reading `/tmp/logfile` in a subsequent Shell call.

This fails for two reasons:
1. **No Shell ID** — backgrounding with `&` inside the shell gives a PID but not a Shell tool ID. The Await tool cannot monitor it. You are flying blind.
2. **Sandbox mismatch** — if the `nohup` call used `required_permissions: ["all"]` (non-sandboxed) but the follow-up `cat /tmp/logfile` uses default sandbox, the sandboxed shell may block reading the file, causing `cat` to hang indefinitely (appearing as "interrupted after Nms" with N = hours, because the Cursor UI timed out waiting for it — not because `cat` actually ran that long).

**Correct pattern:** Set `block_until_ms: 0` on the Shell tool call itself. The tool moves the entire command to the background and returns a Shell ID. Then use Await to monitor completion via regex.

```
# Shell tool params:
block_until_ms: 0          ← backgrounds the command; returns Shell ID
required_permissions: [all] ← consistent permission for any follow-up reads

# Then:
Await(task_id=<Shell ID>, pattern="Done\\.|exit_code: 0", block_until_ms=300000)
```

**Rule:** `nohup … &` inside a Shell tool call is ONLY acceptable when you do not need to detect completion — e.g. launching a daemon whose output you will never read. For any long-running command where you need to know when it finishes, always use `block_until_ms: 0` on the Shell tool.

**Follow-up reads:** Always use `required_permissions: ["all"]` for any Shell call that reads files created by a non-sandboxed process. Sandbox and non-sandbox processes do not share `/tmp` visibility in the same way.

## Duration estimates — set `block_until_ms` from expected range, not arbitrarily

**The problem:** Without a duration expectation, the agent sets `block_until_ms` arbitrarily and cannot distinguish "still running normally" from "hung." History files are not needed — rough order-of-magnitude estimates per operation type are sufficient to detect a hung command.

**Rule:** Before backgrounding any command, look up its expected range below. Set `block_until_ms` in Await to `max_expected × 1.5`. If Await returns without matching the success pattern, the command has exceeded its expected range — **investigate immediately; do not keep waiting.**

### Database index creation — `CREATE INDEX` / `ALTER TABLE … ADD INDEX`

Estimates assume a mid-range Azure VM or Flexible Server (8–16 vCores, SSD). Multiply by 3–5× for spinning disk or heavily loaded servers.

| Rows | PostgreSQL `CONCURRENTLY` | SQL Server (blocking) | MySQL `INPLACE LOCK=NONE` |
|------|--------------------------|----------------------|--------------------------|
| 100K | < 5 s | < 5 s | < 5 s |
| 1M | 5–30 s | 5–30 s | 5–30 s |
| 10M | 30 s – 3 min | 30 s – 3 min | 30 s – 3 min |
| 100M | 3 – 15 min | 3 – 15 min | 3 – 15 min |
| 1B | 15 – 60 min | 15 – 60 min | 15 – 60 min |

SQL Server also requires **temp sort space** ≈ 10–20% of table size. If the filegroup or tempdb is full, the command fails immediately (not a timeout).

### LFC QBC snapshot ingestion — snapshot phase only

Estimates assume Databricks serverless, standard Azure network path. PNG adds < 5% overhead.

| Rows | Expected range | Notes |
|------|---------------|-------|
| 100K | 10 – 60 s | canary; should be fast |
| 1M | 30 s – 5 min | |
| 10M | 3 – 15 min | |
| 100M | 10 – 60 min | |
| 1B | 30 min – 4 hr | varies widely by engine and row width |

A QBC snapshot that exceeds 2× the upper bound without reaching COMPLETED is hung or has a resource bottleneck — check pipeline events before waiting further.

### Azure VM / disk operations

| Operation | Expected range |
|-----------|---------------|
| `az vm deallocate` | 1 – 5 min |
| `az disk update --size-gb` (offline) | 10 – 30 s |
| `az vm start` | 1 – 3 min |
| `growpart` + `resize2fs` / `xfs_growfs` | < 10 s (online resize) |
| `az vm run-command invoke` (simple script) | 30 s – 3 min |

### How to apply

```
# Example: CREATE INDEX CONCURRENTLY on 100M rows → max ~15 min → block_until_ms = 15*60*1000*1.5 = 1350000
Await(task_id=<id>, pattern="CREATE INDEX|Done", block_until_ms=1350000)

# If Await returns without pattern match → exceeded expected range → investigate:
#   - Check if process is still running (ps aux)
#   - Check DB logs or pg_stat_activity / sys.dm_exec_requests
#   - Do NOT just set a larger block_until_ms and keep waiting
```

**Why not a history file?** The agent only needs to distinguish *hung* from *running slowly*. Rough ranges make that distinction. The benchmark results sheet serves as historical LFC timing data for anomaly detection once a baseline exists. Maintaining a separate history file adds overhead and becomes stale as hardware and load change.

## Cloud CLI wrappers — use `CMD`, not `$()`

See the `shell-cmd-wrapper` skill for the full `CMD` implementation. Key principle: never use `$()` or `2>/dev/null` for REST API calls (`az`, `aws`, `gcloud`, `databricks`, `mysql`, etc.) — they silently swallow errors and can hang on API outages.

## Canonical script shape and examples

For the full script skeleton (`run` + `run_log` + `check_prerequisites` + `main` + `BASH_SOURCE` guard), argv array examples, optional flag examples, and shared helper extraction: see [script-structure-examples.md](script-structure-examples.md).
