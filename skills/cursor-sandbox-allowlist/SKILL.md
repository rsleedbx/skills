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

### Current allowlist (robert.lee's machine)

```json
{
  "terminalAllowlist": [
    "mysql",
    "psql",
    "databricks",
    "sqlcmd",
    "az",
    "gcloud",
    "curl"
  ]
}
```

### Adding a new command

Read the file, add the entry, write it back. Example — adding `docker`:

```json
{
  "terminalAllowlist": [
    "mysql",
    "psql",
    "databricks",
    "sqlcmd",
    "az",
    "gcloud",
    "curl",
    "docker"
  ]
}
```

Reload Cursor after editing: `Cmd+Shift+P → Reload Window`.

## How the allowlist works

| Source | Precedence | Notes |
|--------|-----------|-------|
| Team admin dashboard | Highest | Overrides everything |
| `~/.cursor/permissions.json` | Middle | Makes Cursor UI read-only for those entries |
| Cursor Settings UI → Agents → Auto-Run → Command Allowlist | Lowest | Use only when `permissions.json` doesn't exist |

Commands on the allowlist run **outside the sandbox** with no network or filesystem restrictions — no "Run" prompt, no `required_permissions` needed in tool calls.

Commands NOT on the allowlist require `required_permissions: ["full_network"]` (network only) or `required_permissions: ["all"]` (full access), each of which triggers a "Run" prompt.

## Agent behaviour after allowlist update

Once a command is on the allowlist, **do not** pass `required_permissions` for it — the command already runs unrestricted and the extra permission request is unnecessary noise.
