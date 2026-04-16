---
name: databricks-asset-bundle
description: Databricks Asset Bundle (DAB) — databricks.yml structure, workspace host binding, GCP numeric URLs, auth, targets, and bundle operations. Use when creating or editing databricks.yml, configuring bundle targets, setting workspace.host for GCP, syncing files to Databricks, deploying jobs or pipelines, or debugging bundle sync and workspace binding issues.
---

# Databricks Asset Bundle (DAB)

DAB packages Databricks resources (notebooks, jobs, pipelines, models) into a `databricks.yml` file for versioned deployment across workspaces.

## Sub-topics

- **File sync** (allowlist, versioned paths): see [sync.md](sync.md)

---

## `databricks.yml` top-level structure

```yaml
bundle:
  name: <bundle-name>

variables:
  my_var:
    default: "value"

sync:          # top-level only — NOT nested under bundle
  paths: [...]

resources:
  jobs:
    my_job: ...
  pipelines:
    my_pipeline: ...

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<workspace-url>
```

---

## Workspace host binding

`targets.<name>.workspace.host` binds the bundle to a specific workspace. Used by:
- `databricks bundle sync` / `deploy` / `run`
- The Databricks VS Code extension ("pick workspace")

**Not a substitute for notebooks:** Python / SDK / Connect read `~/.databrickscfg` (or env vars). `databricks.yml` `host` is for bundle/IDE workspace binding only — not for `WorkspaceClient()` in code.

### GCP workspaces

GCP workspace URLs are numeric:

```
https://<WORKSPACE_ID>.<shard>.gcp.databricks.com
```

Example: `https://8259565162423548.8.gcp.databricks.com`

Set this under `targets.<name>.workspace.host`. The VS Code extension and `databricks bundle sync` on GCP **require** the explicit `host` — they do not reliably infer it from `workspace.profile` alone.

**When `host` can be omitted:** `databricks bundle` can sometimes resolve the workspace from `workspace.profile` if the named profile in `~/.databrickscfg` already defines `host`. Reliable on AWS/Azure; unreliable on GCP — always set explicit `host` for GCP.

The workspace ID is not a secret (public to all users of that workspace). Keep OAuth tokens only in `~/.databrickscfg`.

---

## Auth

DAB targets use `~/.databrickscfg` for authentication — same file as Databricks Connect and the SDK.

```yaml
targets:
  dev:
    workspace:
      host: https://<workspace-url>
      profile: DEFAULT          # optional; omit to use DEFAULT profile
```

Or set `DATABRICKS_CONFIG_PROFILE` in the environment.

---

## Common commands

```bash
databricks bundle validate          # check databricks.yml syntax
databricks bundle sync              # sync files to workspace
databricks bundle deploy            # deploy resources (jobs, pipelines)
databricks bundle run <resource>    # run a deployed job or pipeline
```

Add `-t <target>` to use a non-default target (e.g. `-t prod`).
