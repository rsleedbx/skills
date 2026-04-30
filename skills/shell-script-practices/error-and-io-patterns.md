# Error and I/O Patterns — Deep Reference

## Never swallow errors with `2>/dev/null` on data-fetching commands

**`2>/dev/null` on a command whose output you depend on silently turns any error into empty data.**

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
# GOOD — existence check: only care about exit code
if az network private-endpoint show ... &>/dev/null; then
  echo "already exists — skipping"
fi

# GOOD — data fetch: let errors print to stderr
SECRET=$(databricks secrets get-secret "$SCOPE" "$KEY" \
  --profile "$PROFILE" --output json \
  | jq -r '.value // empty')

# GOOD — optional data: capture stderr separately to inspect it
ERR=$(cmd ... 2>&1 >/dev/null) || echo "warning: cmd failed: ${ERR}" >&2
```

### Verify CLI syntax before adding error suppression

Before adding `2>/dev/null` to any CLI call:
1. Run the command manually once and confirm it succeeds.
2. Test with a known-bad input to confirm the error message appears on stderr.
3. Only then decide whether to suppress (existence checks) or leave visible (data fetches).

A quick way to catch wrong flags: remove `2>/dev/null` temporarily, run the script, and see if any commands emit "unknown flag" or "invalid option" to stderr.

---

## Use `--output none` for Azure CLI commands, not `>/dev/null`

When you don't need the JSON response from an `az` command, suppress it with the Azure CLI flag `--output none`, not a shell redirect.

```bash
# BAD — >/dev/null suppresses both stdout AND stderr, hiding CLI errors
az network nic update … >/dev/null

# BAD — 2>/dev/null hides auth errors, policy errors, and "resource not found"
az network nic update … 2>/dev/null

# GOOD — --output none suppresses JSON response; errors still print to stderr
az network nic update … --output none
```

`--output none` tells the Azure CLI to produce no output on success, but still writes errors and warnings to stderr. `>/dev/null` silently hides `RequestDisallowedByPolicy` and `ResourceNotFound`.

**Exception:** Existence checks where you only care about the exit code may use `&>/dev/null`:
```bash
if az network private-endpoint show … &>/dev/null; then
  echo "already exists — skipping"
fi
```

---

## Capture pipeline output before piping — don't swallow exit codes

When a command's output drives a subsequent pipeline, capture it first. Piped commands suppress the earlier command's non-zero exit even with `set -o pipefail`.

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

---

## Validate credentials before use — reset only on failure

When a script stores a database (or service) password in a secret and restores it on re-runs, always **verify the credential works before using it**. Only trigger a password reset if the login actually fails.

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
- **Always verify after a reset.** The reset command itself can fail silently. A second login check catches this.
- **Both the DBA/admin credential and the LFC/app credential** should be verified. The DBA credential is validated before the configure step; the app-user credential is validated inside the configure script after `CREATE USER / ALTER USER`.
- **For VMs without a managed reset API**, use a privileged out-of-band channel (e.g. `mysql --defaults-file=/etc/mysql/debian.cnf` via `az vm run-command`) to issue `ALTER USER … IDENTIFIED BY`.

For engine-specific examples see: `azure-vm-mysql`, `azure-vm-postgres`, `azure-vm-sqlserver` skills.
