---
name: databricks-lakebase
description: >-
  Databricks Lakebase (managed database instance on Databricks) provisioning
  and lifecycle management via the Python SDK (WorkspaceClient). Use when
  creating, deleting, or polling database instances on Databricks, integrating
  Lakebase with the mist/forgedb provisioning flow, or handling "already exists"
  idempotency.
---

# Databricks Lakebase

Lakebase is Databricks' managed database product. Instances are created and
managed via the Databricks SDK — there is no Terraform resource for it.

## Package

```bash
pip install databricks-sdk
```

## Create a database instance

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.database import DatabaseInstanceState

w = WorkspaceClient(profile="my-workspace-profile")

instance = w.database.create_database_instance(
    name="my-lakebase-instance",
    # stopped=False  # optional; defaults to running
)
# Returns a DatabaseInstance object immediately — creation is async.
# Poll until the instance reaches AVAILABLE state:
while True:
    inst = w.database.get_database_instance(name="my-lakebase-instance")
    if inst.state == DatabaseInstanceState.AVAILABLE:
        break
    if inst.state in (DatabaseInstanceState.FAILED, DatabaseInstanceState.DELETED):
        raise RuntimeError(f"Lakebase instance entered state {inst.state}")
    time.sleep(5)
```

## Idempotent create ("already exists" handling)

```python
from databricks.sdk.errors import ResourceAlreadyExists

try:
    w.database.create_database_instance(name=name)
except ResourceAlreadyExists:
    pass   # already provisioned — continue

inst = w.database.get_database_instance(name=name)
```

## Delete (with purge)

`purge=True` permanently destroys the data. Without it the instance enters a
soft-deleted state and can be recovered.

```python
w.database.delete_database_instance(name=name, purge=True)
```

## Credential validation

Before calling Lakebase APIs, validate that the profile resolves to a real workspace:

```python
try:
    w.current_user.me()
except Exception as e:
    raise RuntimeError(f"Workspace auth failed for profile '{profile}': {e}")
```

This catches misconfigured profiles (wrong host, expired token) before hitting
the Lakebase APIs.

## Notes

- `WorkspaceClient` reads the token cache at `~/.databricks/token-cache.json`
  written by `databricks auth login`. The CLI binary is not required at runtime.
- For GCP workspaces, `WorkspaceClient(host=...)` may need the numeric URL
  (`https://<number>.gcp.databricks.com`) rather than an alias.
- There is no Terraform provider resource for Lakebase — SDK is the only path.
- In the mist orchestration layer, Lakebase creation is a separate code path
  from traditional cloud databases: Terraform is not called; `WorkspaceClient`
  is called directly from `mist/core.py`.
