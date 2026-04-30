---
name: cursor-sandbox-allowlist
description: >-
  Fix repeated "Run" approval prompts when the Cursor agent runs shell commands.
  Commands not on the allowlist trigger a sandbox permission dialog every time.
  Use when the user is seeing repeated "Run" prompts, when adding a new CLI tool
  to the workflow (mysql, psql, databricks, gcloud, etc.), or when shell commands
  require full_network permission repeatedly.
---

# Cursor Sandbox Allowlist

## When to surface this

Stop and tell the user to update the allowlist when **any of these happen**:

- The same "Run" approval dialog appears 2+ times in a session for the same command
- A new CLI tool is being used for the first time in agent shell calls (mysql, psql, databricks, az, gcloud, sqlcmd, curl, etc.)
- Shell output contains "SANDBOXING: This command ran in a sandbox" for a command that will be called many more times

Say: *"You'll be asked to approve 'Run' for every `<command>` call this session. Add it to `~/.cursor/permissions.json` to skip these prompts permanently."*

## How to fix

The allowlist lives in **`~/.cursor/permissions.json`** (`terminalAllowlist` key).
When this file exists, the **Cursor UI allowlist becomes read-only** — edit the file directly.

### Adding a new command

Read `~/.cursor/permissions.json`, add the entry to `terminalAllowlist`, write it back.
Reload Cursor after editing: `Cmd+Shift+P → Reload Window`.

Current allowlist and full details are in the **How the allowlist works** section below.

## How the allowlist works

| Source | Precedence | Notes |
|--------|-----------|-------|
| Team admin dashboard | Highest | Overrides everything |
| `~/.cursor/permissions.json` | Middle | Makes Cursor UI read-only for those entries |
| Cursor Settings UI → Agents → Auto-Run → Command Allowlist | Lowest | Use only when `permissions.json` doesn't exist |

Commands on the allowlist run **outside the sandbox** with no network or filesystem restrictions — no "Run" prompt, no `required_permissions` needed in tool calls.

Commands NOT on the allowlist require `required_permissions: ["full_network"]` (network only) or `required_permissions: ["all"]` (full access), each of which triggers a "Run" prompt.

## Two common mistakes that still cause "Run" prompts

### Mistake 1 — wrapper scripts bypass allowlisted binaries

The allowlist matches on the **first token** of the command string only. If a shell script wrapper is called, the first token is `bash` (or the script path), not the binary inside.

```bash
# These trigger "Run" even though sqlcmd/psql/mysql are allowlisted:
/tmp/ss_run.sh -Q "..."       # first token = /tmp/ss_run.sh  ✗
bash /tmp/ss_run.sh -Q "..."  # first token = bash            → allowlist bash  ✓

# Fix: add "bash" to the allowlist to cover all wrapper scripts
```

**Always add `bash` to the allowlist** when any workflow uses wrapper scripts or multi-command pipelines.

### Mistake 2 — omitting `required_permissions` for commands that need network

The command allowlist prevents the "Run" prompt but does **not** automatically grant unrestricted network. Commands on the allowlist still run in a sandbox with limited network unless `required_permissions: ["all"]` is also specified.

| Command on allowlist | `required_permissions` | Result |
|---------------------|----------------------|--------|
| ✓ | none | No "Run" prompt, but **network still limited** (sandbox) |
| ✓ | `["all"]` | No "Run" prompt, **fully outside sandbox** ✓ |
| ✗ | `["all"]` | "Run" prompt shown, then outside sandbox |
| ✗ | none | "Run" prompt shown, runs in sandbox |

```
# Correct for commands needing full network access (no prompt if allowlisted):
required_permissions: ["all"]

# Wrong — still sandboxed, API calls to non-standard domains will fail:
(no required_permissions field)
```

**Rule**: for allowlisted commands that call external APIs (Databricks, Azure, GCP), always include `required_permissions: ["all"]`. This grants full access with no "Run" prompt as long as the command's first token is on the allowlist.

**Verified** (2026-04-30): `databricks pipelines get ... -p ...` with `required_permissions: ["all"]` ran "outside the sandbox (no restrictions)" with no prompt because `databricks` is allowlisted.

## Current allowlist (robert.lee's machine)

```json
{
  "terminalAllowlist": [
    "bash",
    "mysql",
    "psql",
    "databricks",
    "sqlcmd",
    "az",
    "gcloud",
    "curl",
    "python3",
    "jq"
  ]
}
```
