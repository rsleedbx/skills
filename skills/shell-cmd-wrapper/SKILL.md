---
name: shell-cmd-wrapper
description: >-
  Generic CMD wrapper for all cloud and DB CLIs (az, aws, gcloud, databricks, mysql, psql, oci, etc.)
  that eliminates silent failures from $() and 2>/dev/null. CMD derives /tmp/<binary>_stdout.<PID>
  from basename $1, prints the command with CMD_MASK_SECRETS array masking (glob-safe), supports CMD_TIMEOUT for APIs
  that can hang indefinitely (regional outages), and delegates to CMD_CONT_OR_EXIT for three error
  modes: PRINT_EXIT (fatal, kill -INT $$), PRINT_RETURN (non-fatal, caller handles), or empty
  (continue). Read output from /tmp/<binary>_stdout.<PID> with jq — never use $() for REST API
  calls. Use CMD_OUT_SUFFIX to disambiguate sequential calls to the same binary. Use when writing
  any shell script that calls az, aws, gcloud, databricks, mysql, psql, sqlcmd, or any CLI that
  returns JSON or can fail/hang silently.
---

# Shell CMD Wrapper

One generic `CMD` function replaces per-CLI wrappers (`AZ`, `DBX`, `AWS`, …). The binary name
(`basename $1`) drives the temp file prefix automatically — no mapping table, no per-CLI duplication.

## Why `$()` and `2>/dev/null` are dangerous for REST API calls

| Failure mode | `$()` + `2>/dev/null` | `CMD` wrapper |
|---|---|---|
| API hangs (regional outage) | Blocks forever | `CMD_TIMEOUT=30` → RC=124, continues |
| Secret contains spaces | Word-splits on `$IFS`; no match | `CMD_MASK_SECRETS` is a bash array — one element per secret |
| Secret contains `*` `?` `[` | Treated as glob pattern; wrong mask | Four pure-bash substitutions escape them as literals before `${var//pat/repl}` |
| Auth error | Stderr swallowed; stdout `""` | Stderr printed; RC non-zero |
| Wrong flags / bad JSON | Silent empty result | Error visible; script aborted |
| Exit code lost in pipe (`$(cmd \| jq ...)`) | jq's RC reported, not cmd's | RC captured before any pipe |

## Implementation

Source this block once from a shared helper (e.g. `cmd-wrapper-helpers.sh`) and `export -f` both functions.

```bash
#!/usr/bin/env bash
# cmd-wrapper-helpers.sh — source this file; do not execute directly.
(return 0 2>/dev/null) || { echo "error: source this file, do not execute it" >&2; exit 1; }

CMD() {
  local _cli
  _cli=$(basename "$1")
  local _suffix="${CMD_OUT_SUFFIX:+_${CMD_OUT_SUFFIX}}"
  local CMD_EXIT_ON_ERROR="${CMD_EXIT_ON_ERROR:-}"
  local CMD_STDOUT="${CMD_STDOUT:-/tmp/${_cli}_stdout${_suffix}.$$}"
  local CMD_STDERR="${CMD_STDERR:-/tmp/${_cli}_stderr${_suffix}.$$}"
  local CMD_TIMEOUT="${CMD_TIMEOUT:-0}"
  local RC

  # Snapshot CMD_MASK_SECRETS into a local and immediately clear the global so
  # callers do not need to reset it between CMD invocations.
  local _mask=("${CMD_MASK_SECRETS[@]+"${CMD_MASK_SECRETS[@]}"}")
  CMD_MASK_SECRETS=()

  local PWMASK="$*"
  local _secret _escaped
  for _secret in "${_mask[@]+"${_mask[@]}"}"; do
    if [[ -n "$_secret" ]]; then
      _escaped="$_secret"
      _escaped="${_escaped//\\/\\\\}"   # backslash first (order matters)
      _escaped="${_escaped//\*/\\*}"
      _escaped="${_escaped//\?/\\?}"
      _escaped="${_escaped//\[/\\[}"
      PWMASK="${PWMASK//${_escaped}/***}"
    fi
  done
  if (( CMD_TIMEOUT > 0 )); then
    echo -n "  + [timeout=${CMD_TIMEOUT}s] ${PWMASK}"
    timeout "$CMD_TIMEOUT" "$@" >"${CMD_STDOUT}" 2>"${CMD_STDERR}"
  else
    echo -n "  + ${PWMASK}"
    "$@" >"${CMD_STDOUT}" 2>"${CMD_STDERR}"
  fi
  RC=$?

  if [[ "$RC" != "0" ]]; then
    if [[ "${CMD_EXIT_ON_ERROR}" == "PRINT_RETURN" ]]; then
      echo " failed (RC=${RC})" >&2
      [[ -s "${CMD_STDOUT}" ]] && cat "${CMD_STDOUT}" >&2
      [[ -s "${CMD_STDERR}" ]] && cat "${CMD_STDERR}" >&2
      return "$RC"
    elif [[ "${CMD_EXIT_ON_ERROR}" == "PRINT_EXIT" ]]; then
      echo " failed (RC=${RC})" >&2
      [[ -s "${CMD_STDOUT}" ]] && cat "${CMD_STDOUT}" >&2
      [[ -s "${CMD_STDERR}" ]] && cat "${CMD_STDERR}" >&2
      kill -INT $$
    else
      echo " failed (RC=${RC}) — continuing" >&2
      return "$RC"
    fi
  else
    echo ""   # newline after the echoed command on success
    if [[ "${CMD_EXIT_ON_ERROR}" == "RETURN_1_STDOUT_EMPTY" && ! -s "${CMD_STDOUT}" ]]; then
      return 1
    fi
  fi
  return 0
}
export -f CMD
```

## Usage

### Reading output — always from the temp file, never `$()`

```bash
# GOOD — output saved to file; jq reads from file; exit code is clean
CMD_EXIT_ON_ERROR="PRINT_EXIT" CMD az account show
az_id=$(jq -r '.id' /tmp/az_stdout.$$)
az_user=$(jq -r '.user.name' /tmp/az_stdout.$$)

# BAD — $() discards stderr; exit code is lost if piped
az_id=$(az account show 2>/dev/null | jq -r '.id')
```

### Error modes

```bash
# Fatal — script aborts on any failure
CMD_EXIT_ON_ERROR="PRINT_EXIT" CMD az group show --resource-group "$RG"

# Non-fatal — caller checks return code; error is still printed
if ! CMD_EXIT_ON_ERROR="PRINT_RETURN" CMD az group show --resource-group "$RG"; then
  echo "==> Resource group missing — creating ..."
  CMD_EXIT_ON_ERROR="PRINT_EXIT" CMD az group create \
    --resource-group "$RG" --location "$LOC" --output none
fi

# Continue on failure — prints warning, script keeps going
CMD az some-optional-check ...
```

### Timeout — for APIs that can hang

Always pair `CMD_TIMEOUT` with `CMD_EXIT_ON_ERROR=PRINT_EXIT`. A timeout is a failure — the
existing `PRINT_EXIT` branch handles it: prints stdout+stderr and aborts. No special timeout
handling is needed in the wrapper or the caller. Do not set `CMD_TIMEOUT` on existence probes
where a non-zero exit is a normal first-run outcome.

```bash
# Long-running command that must succeed — pair CMD_TIMEOUT with PRINT_EXIT.
# Timeout is a failure; PRINT_EXIT prints stdout+stderr and aborts. No special handling needed.
CMD_TIMEOUT=60 CMD_EXIT_ON_ERROR="PRINT_EXIT" \
  CMD az mysql flexible-server restart \
      --resource-group "$RG" --name "$SERVER" --output none

# Existence probe — do NOT set CMD_TIMEOUT; failure (not found) is a normal first-run outcome.
CMD_EXIT_ON_ERROR="PRINT_RETURN" \
  CMD az mysql flexible-server show \
      --resource-group "$RG" --name "$SERVER" \
      --query 'network.delegatedSubnetResourceId' -o tsv
if [[ $? -ne 0 ]]; then
  _DELEGATED=""     # server does not exist yet — normal on first run
else
  _DELEGATED=$(cat /tmp/az_stdout.$$)
fi
```

### Sequential calls to the same binary

Use `CMD_OUT_SUFFIX` so files don't overwrite each other:

```bash
CMD_OUT_SUFFIX="before" CMD az mysql flexible-server show --name "$SERVER_A" ...
CMD_OUT_SUFFIX="after"  CMD az mysql flexible-server show --name "$SERVER_B" ...
fqdn_a=$(jq -r '.fullyQualifiedDomainName' /tmp/az_stdout_before.$$)
fqdn_b=$(jq -r '.fullyQualifiedDomainName' /tmp/az_stdout_after.$$)
```

### Array syntax

The binary name is derived from `$1` regardless of how args are passed:

```bash
cmd=(az account show)
CMD "${cmd[@]}"
sub_id=$(jq -r '.id' /tmp/az_stdout.$$)   # prefix = "${cmd[0]}" = "az"
```

### Password masking

`CMD_MASK_SECRETS` must be a **bash array** — one element per secret. Never use a space-separated string: secrets can contain spaces, and the `${var//pat/repl}` substitution treats the pattern as a glob, so `*`, `?`, `[` in a secret would match unintended substrings.

`CMD_MASK_SECRETS` **must be set as a standalone statement** before calling `CMD` — NOT as a command prefix (`VAR=(...) CMD ...`). When used as a command prefix, bash passes the array as a temporary scalar string including the literal `(` and `)` characters, so the value never matches the actual secret. The `CMD` function snapshots and clears `CMD_MASK_SECRETS` at the start of each invocation, so the mask is automatically cleaned up and does not leak to subsequent calls.

```bash
# CORRECT — separate statement; CMD snapshots and clears it automatically
CMD_MASK_SECRETS=("$DBA_PASSWORD" "$USER_PASSWORD")
CMD_EXIT_ON_ERROR=PRINT_EXIT CMD az mysql flexible-server create \
    --admin-password "$DBA_PASSWORD" ...

# WRONG — prefix syntax; bash passes (value) as scalar string, masking never fires
CMD_MASK_SECRETS=("$DBA_PASSWORD") \
CMD_EXIT_ON_ERROR=PRINT_EXIT CMD az ...  # do not use

# WRONG — space-separated string; breaks if any secret contains a space
CMD_MASK_SECRETS="$DBA_PASSWORD $USER_PASSWORD" CMD az ...   # do not use
```

The implementation escapes `\`, `*`, `?`, `[` in each secret before the substitution so glob special chars are treated as literals.

Add every value that could appear literally in a CLI argument — passwords, subscription IDs, tenant IDs, API tokens.

**Passwords embedded in JSON blobs** — `--string-value` and `--json` arguments frequently contain passwords inline. Masking the password variable masks the occurrence inside the JSON too, since `CMD` performs plain string substitution on the full command line:

```bash
# databricks secrets put-secret with JSON payload containing password
local _mask=()
[[ -n "${password:-}" ]]      && _mask+=("${password}")
[[ -n "${dba_password:-}" ]]  && _mask+=("${dba_password}")
CMD_MASK_SECRETS=("${_mask[@]}") \
CMD_EXIT_ON_ERROR=PRINT_EXIT CMD databricks secrets put-secret \
  "$SCOPE" "$KEY" --string-value "$json_with_passwords" --profile "$PROFILE"

# UC connection JSON — extract the password from the JSON first
local _conn_pass; _conn_pass=$(echo "$_create_json" | jq -r '.options.password // empty')
CMD_MASK_SECRETS=("${_conn_pass}") \
CMD_EXIT_ON_ERROR=PRINT_EXIT CMD databricks api post /api/2.1/unity-catalog/connections \
  --profile "$WORKSPACE_PROFILE" --json "$_create_json"
```

### Existence checks

Existence checks only care about the exit code. Default mode (no `CMD_EXIT_ON_ERROR`) is correct — error is printed but the script continues:

```bash
if CMD az network private-endpoint show \
    --resource-group "$RG" --name "$PE_NAME" ; then
  echo "==> Private endpoint already exists — skipping"
else
  # create it
  CMD_EXIT_ON_ERROR="PRINT_EXIT" CMD az network private-endpoint create ...
fi
```

For truly silent existence checks where the "not found" error is expected and noisy, capture stderr:

```bash
# Only suppress stderr for well-understood "resource not found" cases
"$@" >"${CMD_STDOUT}" 2>"${CMD_STDERR}"   # CMD handles this internally
# After the CMD call, CMD_STDERR is still available for inspection if needed
cat /tmp/az_stderr.$$   # always readable — never swallowed
```

## Rules summary

| Situation | Pattern |
|-----------|---------|
| REST API call whose output you use | `CMD <cli> ...` + read `/tmp/<cli>_stdout.$$` |
| Fatal failure | `CMD_EXIT_ON_ERROR="PRINT_EXIT"` |
| Tolerated failure | `CMD_EXIT_ON_ERROR="PRINT_RETURN"` |
| Long-running command that must succeed | `CMD_TIMEOUT=<N>` — any failure aborts with visible output |
| Existence probe (failure is normal) | No `CMD_TIMEOUT` — use `CMD_EXIT_ON_ERROR=PRINT_RETURN` only |
| Sequential calls to same binary | `CMD_OUT_SUFFIX="a"` / `CMD_OUT_SUFFIX="b"` |
| Secrets in arguments | `CMD_MASK_SECRETS=("$s1" "$s2")` — bash array, one secret per element |
| No JSON output wanted | `--output none` for az; never `>/dev/null` |
| Never | `$()` for REST API calls; `2>/dev/null` on data-fetching commands |

## Refactoring measure

A successful refactor using CMD reduces total line count. Each conversion from raw CLI calls
to CMD eliminates:
- the `2>/dev/null` or `&>/dev/null` redirect
- the `|| { echo "error..."; kill -INT $$; }` guard
- the `$()` capture + separate error check
- manual RC checks for timeouts or specific exit codes

If line count goes up after introducing CMD, the refactor is not done — look for redundant
`if`/`else` blocks, duplicate error messages, or leftover manual RC handling that CMD already
covers.

## Temp file reference

| Binary | stdout | stderr |
|--------|--------|--------|
| `az` | `/tmp/az_stdout.$$` | `/tmp/az_stderr.$$` |
| `databricks` | `/tmp/databricks_stdout.$$` | `/tmp/databricks_stderr.$$` |
| `aws` | `/tmp/aws_stdout.$$` | `/tmp/aws_stderr.$$` |
| `gcloud` | `/tmp/gcloud_stdout.$$` | `/tmp/gcloud_stderr.$$` |
| `mysql` | `/tmp/mysql_stdout.$$` | `/tmp/mysql_stderr.$$` |
| `psql` | `/tmp/psql_stdout.$$` | `/tmp/psql_stderr.$$` |
| `sqlcmd` | `/tmp/sqlcmd_stdout.$$` | `/tmp/sqlcmd_stderr.$$` |
| `oci` | `/tmp/oci_stdout.$$` | `/tmp/oci_stderr.$$` |
| With suffix | `/tmp/az_stdout_pre.$$` | `/tmp/az_stderr_pre.$$` |
