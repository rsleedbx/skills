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

# Shell script practices

## Abort: prefer `kill -INT $$` over `exit 1` for manual errors

- After printing an error for a **recoverable / user** failure (missing tool, bad command, failed `aws`), use **`kill -INT $$`** instead of **`exit 1`**.
- **Why:** If the file is ever **`source`d** into an interactive shell, **`exit 1` closes that shell**; **`kill -INT $$`** sends **SIGINT** to the current process (usually “cancelled” behavior: stop the script path without logging out of the session).
- When the script is run as **`./script.sh`**, both stop the script process; exit status for SIGINT is typically **130** (`128 + 2`).
- **`${var:?msg}`** and **`set -e`** still use the shell’s own fatal paths; this rule targets **explicit** `exit 1` after `echo` / checks.

```bash
if ! "${create[@]}"; then
  echo "error: create-bucket failed" >&2
  kill -INT $$
fi
```

## Check prerequisites once

- Probe **required** external tools **one time** before real work (e.g. one `check_prerequisites` from `main` for every subcommand that needs them), not inside every `if`/`case` branch.
- **One global prerequisite function** for the script when subcommands share the same toolchain (e.g. `databricks`, `jq`, `aws` together). Avoid a separate `check_prerequisites_foo` per subcommand unless a command truly needs a **disjoint** set of binaries.
- **Prefer required dependencies** over optional ones: list them in the script header and `usage`, fail fast if missing, and **avoid** parallel code paths that differ only because a formatter or helper might be absent.

### Canonical loop: `command -v` over a fixed tool list

- Put every required binary name in **one** `for … in …` list—adding a tool means editing **one** line.
- Use **`command -v "$bin"`** (POSIX-friendly) to detect presence; redirect stdout/stderr to `/dev/null` so the check stays quiet.
- On first missing tool: **message to stderr**, then **`kill -INT $$`** (see “Abort” above)—no copy-pasted `if ! command -v databricks` / `if ! command -v jq` blocks.

```bash
check_prerequisites() {
  local bin
  for bin in databricks jq aws; do
    command -v "$bin" >/dev/null 2>&1 || {
      echo "error: '${bin}' not found (install and ensure it is on PATH)" >&2
      kill -INT $$
    }
  done
}
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` — `check_prerequisites databricks jq` for **`discover`**; **`check_prerequisites aws jq`** inside the **`bucket`** / **`role`** arms when **`UCTB_CLOUD`** is **`aws`**.

## Avoid optional-tool branching

- Do not add `if has_jq; then … else …` (or similar) when requiring the tool is acceptable; one path is easier to read and test.
- If a dependency is truly optional for a rare mode, isolate that in a subcommand or separate script so the default path stays single-track.

## Structure

- `usage` / `help` paths should **not** invoke tools that real commands need (only print text).
- Shared API calls live in one helper (`_foo`); branching only on profile/env, not on “is formatter installed.”

## Inline a single command

- Do **not** add `cmd_foo() { only_one_pipeline; }` when `foo` is used once and the body is a **single** pipeline or invocation—put that line **inline** in `main`’s `case` arm (or next to the sole caller).
- Keep a **named function** when the body is non-trivial, reused from multiple sites, or you need a stable name for testing/documentation.

## Required environment variables (avoid `: "${var:?msg}"` when you want SIGINT)

**`${parameter:?word}`** on unset/null writes `word` to stderr and uses the shell’s **fatal expansion** path (often **exit 1**, not SIGINT). That is fine in many **executed** scripts; for **`source`d** flows where you want the same **“cancelled”** behavior as other manual errors, **do not** rely on `:?` alone.

Prefer a tiny helper that **`echo`s to stderr** and then **`kill -INT $$`**:

```bash
_uctb_require_nonempty() {
  local val=$1
  shift
  if [[ -z "${val}" ]]; then
    echo "error: $*" >&2
    kill -INT $$
  fi
}
```

```bash
_uctb_bucket_aws() {
  _uctb_require_nonempty "${BUCKET_NAME:-}" "set BUCKET_NAME (globally unique S3 bucket name)"
  _uctb_require_nonempty "${AWS_REGION:-}" "set AWS_REGION (e.g. from discover .region)"
  …
}
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` (`_uctb_require_nonempty`, `_uctb_bucket_aws`, `_uctb_role_aws`).

## Optional CLI flags without verbose `if`/`else`

When a flag (e.g. `-p profile`) is **omitted when unset** and **passed when set**, use **one** command line instead of duplicating the whole invocation.

- Use **`${parameter:+word}`** (if *set and non-empty*, substitute `word`; otherwise nothing). Do **not** use **`:-`** here—that chooses a default when unset, which is the wrong semantics for “optional flag.”
- Put a **leading space** inside `word` so the previous token and the flag stay separate words (e.g. `databricks` then `-p`, not `databricks-p`).
- **Quote** the variable inside `word` so values with spaces stay a single argument.

```bash
# One line; no duplicated databricks … api get …
databricks ${DATABRICKS_PROFILE:+ -p "${DATABRICKS_PROFILE}"} api get "${path}" -o json
```

Same pattern for other tools: optional `--region`, `--namespace`, etc., when unset means “use tool default.”

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` (`_uctb_metastore_discover`).

## Always use `${var}` braces in echo/string contexts

**Always wrap variable references in curly braces (`${var}`) when they appear adjacent to non-ASCII characters, punctuation, or in echo/string output lines.** Without braces, bash's locale-aware `isalpha()` can include the first byte of a multi-byte character (e.g. `0xE2` from an en-dash `–` is `â` in ISO-8859-1, which `isalpha()` returns true for) as part of the variable name, silently reading the wrong (unbound) variable.

```bash
# BAD — en-dash byte 0xE2 directly follows $start_ip; bash may read
# "start_ip\xE2\x80" as the variable name → unbound variable
echo "Rule for $start_ip–$end_ip"

# GOOD — closing brace terminates the name before any non-ASCII byte
echo "Rule for ${start_ip} - ${end_ip}"
```

- **In echo/printf strings**: always use `${var}` rather than `$var`.
- **In command arguments** (where word-splitting is not a concern): `"$var"` is acceptable, but `"${var}"` is never wrong.
- **Adjacent to em/en dashes, ellipses, or any non-ASCII literal**: curly braces are **required**.

## Never swallow errors with `2>/dev/null` on data-fetching commands

**`2>/dev/null` on a command whose output you depend on silently turns any error into empty data.** The script keeps running as if the command succeeded, with wrong or missing values downstream. This is the most common source of subtle, hard-to-diagnose bugs in shell scripts.

### The anti-pattern

```bash
# BAD — if get-secret fails (wrong flags, auth error, missing key),
# _secret_json is silently empty; the loop skips every secret with no warning.
_secret_json=$(databricks secrets get-secret \
  --scope "$SCOPE" --key "$KEY" --output json 2>/dev/null \
  | jq -r '.value // empty')

# BAD — if az command fails, RESULT is empty; subsequent code sees wrong state.
RESULT=$(az resource show ... 2>/dev/null)
```

This exact pattern caused the PNG discovery loop to find 0 secrets from the scope — `--scope`/`--key` are not valid flags for `get-secret` (only positional args are), but the error was hidden. The script appeared to work while silently producing wrong data.

### Rules

| Situation | Pattern |
|-----------|---------|
| **Existence check** (you only care if it succeeds) | `cmd ... &>/dev/null && echo "exists"` |
| **Data fetch** (you use the output) | Never `2>/dev/null`; let stderr print so errors are visible |
| **Optional data** (empty result is acceptable) | Redirect stderr to a file or variable, check it explicitly |
| **Known noisy warning** you want to hide | Redirect only that specific FD after verifying the command works correctly first |

```bash
# GOOD — existence check: only care about exit code, not output
if az network private-endpoint show ... &>/dev/null; then
  echo "already exists — skipping"
fi

# GOOD — data fetch: let errors print to stderr; script aborts on failure
SECRET=$(databricks secrets get-secret "$SCOPE" "$KEY" \
  --profile "$PROFILE" --output json \
  | jq -r '.value // empty')

# GOOD — optional data: capture stderr separately to inspect it
ERR=$(cmd ... 2>&1 >/dev/null) || echo "warning: cmd failed: $ERR" >&2
```

### Verify CLI syntax before adding error suppression

Before adding `2>/dev/null` to any CLI call:
1. Run the command manually once and confirm it succeeds.
2. Test with a known-bad input to confirm the error message appears on stderr.
3. Only then decide whether to suppress (existence checks) or leave visible (data fetches).

A quick way to catch wrong flags: remove `2>/dev/null` temporarily, run the script, and see if any commands emit "unknown flag" or "invalid option" to stderr.

## Validate credentials before use — reset only on failure

When a script stores a database (or service) password in a secret and restores it on re-runs, always **verify the credential works before using it**. Only trigger a password reset if the login actually fails. This avoids a costly reset operation on every re-run and surfaces real auth failures immediately.

### Pattern

```bash
# 1. Try the credential loaded from the secret.
if ! SOME_PWD="$STORED_PASSWORD" some-client \
    --user "$USER" --host "$HOST" --port "$PORT" \
    --connect-timeout 10 --execute "SELECT 1" &>/dev/null; then

  # 2. Mismatch detected — reset the credential to match the secret.
  echo "==> Password mismatch detected — resetting to match secret ..."
  <reset command, e.g. az flexible-server update --admin-password / az vm run-command>
  echo "==> Password reset"

  # 3. Verify the fix took effect — fail fast if still broken.
  if ! SOME_PWD="$STORED_PASSWORD" some-client \
      --user "$USER" --host "$HOST" --port "$PORT" \
      --connect-timeout 10 --execute "SELECT 1" &>/dev/null; then
    echo "error: login still failing after reset — stored password may be wrong" >&2
    kill -INT $$
  fi
  echo "==> Password synced and verified"
else
  echo "==> Password verified"
fi
```

### Rules

- **Try first, reset only on failure.** A password reset on every run is wasteful and slow (Azure Flexible Server takes ~1 min; `az vm run-command` takes ~30 s).
- **Always verify after a reset.** The reset command itself can fail silently (e.g. Azure CLI exits 0 on some policy errors). A second login check catches this.
- **Both the DBA/admin credential and the LFC/app credential** should be verified. The DBA credential is validated before the configure step; the app-user credential is validated inside the configure script after `CREATE USER / ALTER USER`.
- **For VMs without a managed reset API**, use a privileged out-of-band channel (e.g. `mysql --defaults-file=/etc/mysql/debian.cnf` via `az vm run-command`) to issue `ALTER USER … IDENTIFIED BY`.

For engine-specific examples see: **azure-vm-mysql**, **azure-vm-postgres**, **azure-vm-sqlserver** skills.

## `set -u` in every script — including sourced helpers

Add `set -u` (or `set -euo pipefail`) at the top of **every** shell file, including library files that are sourced by other scripts. Typos in variable names silently expand to empty in scripts without `set -u`, and fail fast with a clear message in scripts that have it. This asymmetry makes bugs impossible to reproduce reliably.

```bash
#!/usr/bin/env bash
set -euo pipefail
```

**For sourced library files** (e.g. `azure-firewall-helpers.sh`, `databricks-secrets-helper.sh`) that don't set `set -u` themselves, either:
1. Add `set -u` to the library, or
2. Guard every reference to an optional variable with `${var:-}` so the file is `-u`-safe by construction.

**Why it matters:** A script with `set -u` sourcing a library without it creates a two-tier safety net — the library is the dangerous tier. Bugs introduced in the library are the ones that reach production.

```bash
# BAD — typo silently expands to "" (no set -u in library)
SCOPE="$SCOPE_NANE"   # SCOPE_NANE is undefined → SCOPE is ""

# GOOD — set -u catches it immediately
set -u
SCOPE="$SCOPE_NANE"   # → "bash: SCOPE_NANE: unbound variable"
```

## Safe `for` loops — arrays over unquoted variable expansion

Never iterate over an unquoted variable expansion. Values that contain spaces or glob characters word-split unpredictably.

```bash
# BAD — if _SECRET_KEYS contains a key with a space or *, it breaks
for _key in $_SECRET_KEYS; do …; done

# BAD — optional FQDN vars could contain globs (unlikely, but the pattern is wrong)
for _extra in ${SQL_DNS_MYSQL:-} ${SQL_DNS_PG:-} ${SQL_DNS_SQL:-}; do …; done
```

**Pattern 1:** Use `while IFS= read -r` when the list comes from a command:
```bash
while IFS= read -r _key; do
  …
done <<< "$_SECRET_KEYS"
```

**Pattern 2:** Build a bash array for a fixed set of optional variables:
```bash
_dns_targets=()
[[ -n "${SQL_DNS_MYSQL:-}" ]] && _dns_targets+=("$SQL_DNS_MYSQL")
[[ -n "${SQL_DNS_PG:-}"    ]] && _dns_targets+=("$SQL_DNS_PG")
[[ -n "${SQL_DNS_SQL:-}"   ]] && _dns_targets+=("$SQL_DNS_SQL")

for _extra in "${_dns_targets[@]}"; do
  _add_dns "$_extra"
done
```

**Why:** `for x in $var` is fine when you know the value is a single, safe word — but it is an invisible landmine once a value is dynamic. Use arrays and `"${arr[@]}"` from the start.

## Capture pipeline output before piping — don't swallow exit codes

When a command's output drives a subsequent pipeline, capture it first. Piped commands suppress the earlier command's non-zero exit in most shells (even with `set -o pipefail`, the pipeline exit is the last command's exit).

```bash
# BAD — if az vm run-command fails, INSTALL_STATE is empty;
#        grep and head still exit 0; the install block runs on an already-configured VM.
INSTALL_STATE=$(az vm run-command invoke … | grep "RUNNING" | head -1)

# GOOD — capture az output first; check it; then extract
_raw=$(az vm run-command invoke …)
INSTALL_STATE=$(grep "RUNNING" <<< "$_raw" | head -1)
[[ -n "$INSTALL_STATE" ]] || {
  echo "error: install-state probe returned empty output" >&2
  kill -INT $$
}
```

**Why:** `cmd | grep | head` appears to work because `head` exits 0 and its output is empty — but the script proceeds as if the probe succeeded with no match, triggering a re-install on an already-running service.

## Use `--output none` for Azure CLI commands, not `>/dev/null`

When you don't need the JSON response from an `az` command, suppress it with the Azure CLI flag `--output none`, not a shell redirect.

```bash
# BAD — >/dev/null suppresses both stdout AND stderr, hiding CLI errors
az network nic update … >/dev/null

# BAD — 2>/dev/null hides auth errors, policy errors, and "resource not found"
az network nic update … 2>/dev/null

# GOOD — --output none suppresses the JSON response; errors still print to stderr
az network nic update … --output none
```

**Why:** `--output none` tells the Azure CLI to produce no output on success, but still writes errors and warnings to stderr. `>/dev/null` hides errors too, turning `RequestDisallowedByPolicy` and `ResourceNotFound` into silent no-ops.

**Exception:** Existence checks where you only care about the exit code may use `&>/dev/null`:
```bash
# OK — you only want to know if the resource exists; errors mean "doesn't exist"
if az network private-endpoint show … &>/dev/null; then
  echo "already exists — skipping"
fi
```

## Cloud CLI wrappers — save stdout/stderr, never swallow

See the **`shell-cmd-wrapper`** skill for the full `CMD` + `CMD_CONT_OR_EXIT` implementation, usage patterns, and temp file reference.

Key principle: never use `$()` or `2>/dev/null` for REST API calls (`az`, `aws`, `gcloud`, `databricks`, `mysql`, etc.) — they silently swallow errors and can hang forever on API outages. Use `CMD` instead, which derives `/tmp/<binary>_stdout.<PID>` from `basename $1`, prints the command with masking, supports `CMD_TIMEOUT` for hang prevention, and routes errors through `CMD_CONT_OR_EXIT`.

---

## Extract shared helpers — never duplicate functions across scripts

If the same function appears in more than one script, extract it to a shared helper file and `source` it. Duplicated functions drift independently — a bug fix in one copy is missed in all others.

**Common offenders in this repo:**
- `_uc_connection_upsert` + `_build_*_conn_json` — copied into all six `customer-3-setup-*.sh` scripts
- SQL CLI helpers (`_mc_sql`, `_pc_sql`, `_sc_sql`) — defined in configure scripts and re-created in wrappers

**Pattern:**
```bash
# uc-connection-helpers.sh  ← one canonical file
_uc_connection_upsert() { … }
_build_mysql_conn_json() { … }
_build_pg_conn_json() { … }
_build_sqlserver_conn_json() { … }

# customer-3-setup-azure-mysql.sh
source "$(dirname "${BASH_SOURCE[0]}")/uc-connection-helpers.sh"
_uc_connection_upsert "$CONN_NAME_PNG" "$(_build_mysql_conn_json "$HOST_PNG")"
```

**Rules:**
- A function used in 2+ files → move it to a shared helper.
- Shared helpers use the same `kill -INT $$` abort pattern.
- Don't redefine a sourced function to "override" it — pass different arguments instead, or use a variable the function reads.

## `run() { "$@"; }` for a static argv (not `"${*}"`)

- A one-line wrapper **`run() { "$@"; }`** runs the caller’s arguments as a normal command with **word boundaries preserved**.
- Do **not** use **`"${*}"`**: it joins parameters into **one** string (IFS-separated), so `run aws s3api …` becomes a **single** unusable token.
- Use **`run aws …`** for fixed commands; use an **argv array** when you **`+=`** optional flags, then **`run "${cmd[@]}"`**.

```bash
run() { "$@"; }

if run aws s3api head-bucket --bucket "${BUCKET_NAME}" >/dev/null 2>&1; then
  echo "already exists"
  return 0
fi
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` (`run`, `_uctb_bucket_aws` head check).

## `run_log`: print argv then run

- When you would **`echo … >&2`** immediately before the same **`"$@"`** invocation, use **`run_log`** instead: **`printf '%q ' "$@"`** on stderr (re-executable word list), newline, then **`"$@"`**.
- Keeps **`run`** silent for checks (e.g. `head-bucket` to `/dev/null`); use **`run_log`** only where stderr tracing is intended.

```bash
run_log() {
  printf '%q ' "$@" >&2
  echo >&2
  "$@"
}
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` (`run_log`, `_uctb_metastore_discover`).

## Build a CLI as an argv array (`"${cmd[@]}"`)

- Declare **`local -a cmd=(program fixed-arg …)`** for the stable prefix, then **`cmd+=(extra args)`** for conditional flags or values.
- Execute with **`run "${cmd[@]}"`** (or **`"${cmd[@]}"`** alone) so each array element is one argument (safe for spaces; no `eval`, no fragile string concat).
- Prefer this over two near-duplicate full command lines when only a tail segment differs.

```bash
local -a create=(aws s3api create-bucket --bucket "${BUCKET_NAME}" --region "${AWS_REGION}")
[[ "${AWS_REGION}" != "us-east-1" ]] && create+=(--create-bucket-configuration "LocationConstraint=${AWS_REGION}")
if ! run "${create[@]}"; then
  echo "error: create-bucket failed" >&2
  kill -INT $$
fi
echo "ok: created"
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` (`_uctb_bucket_aws`).

## Example shape

```bash
run() { "$@"; }
run_log() {
  printf '%q ' "$@" >&2
  echo >&2
  "$@"
}

check_prerequisites() {
  local bin
  for bin in databricks jq aws; do
    command -v "$bin" >/dev/null 2>&1 || {
      echo "error: '${bin}' not found (install and ensure it is on PATH)" >&2
      kill -INT $$
    }
  done
}

main() {
  case "${1:-help}" in
    discover)
      check_prerequisites
      _metastore_summary_json
      ;;
    discover-create-s3-bucket)
      check_prerequisites
      _metastore_summary_json
      cmd_create_s3_bucket
      ;;
    create-s3-bucket)
      check_prerequisites
      cmd_create_s3_bucket
      ;;
    …
  esac
}
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` — **`discover`** must be **`source`d** (`BASH_SOURCE` vs `$0` guard); **`discover-create-s3-bucket`** runs `_metastore_summary_json` then **`cmd_create_s3_bucket`** in one **`./`** process; shared `check_prerequisites`.
