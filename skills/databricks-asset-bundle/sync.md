# DAB — File Sync

Controls which files `databricks bundle sync` uploads to the workspace.

## Key Rules

1. `sync` must be at the **top level** of `databricks.yml` — NOT nested under `bundle`
2. Use `sync.paths` as an explicit allowlist — it syncs only what is listed
3. Avoid `sync.include` + `sync.exclude: ["**"]` — excludes override includes and the config is silently ignored

## Template (with versioned path)

```yaml
bundle:
  name: <bundle-name>

variables:
  version:
    default: "0.0.1"
    description: "Bundle version — bump when releasing a new version"

sync:
  paths:
    - path/to/notebook.ipynb
    - path/to/another.ipynb

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<workspace-url>
      file_path: /Workspace/Users/${workspace.current_user.userName}/.bundle/${bundle.name}/v${var.version}/files
```

## Versioning Behaviour

- Each version syncs to its **own path** — old versions are never deleted
- Bumping `version` abandons the old path and creates a new one
- To keep `databricks.yml` version in sync with a Python package version automatically:

```python
databricks_yml = base_dir.parent / "databricks.yml"
if databricks_yml.exists():
    content = databricks_yml.read_text()
    content = re.sub(
        r'(version:\s+default:\s+")[^"]+(")',
        rf'\g<1>{version}\2',
        content,
    )
    databricks_yml.write_text(content)
```

## Steps to configure sync allowlist

1. Read the existing `databricks.yml`
2. Remove any `sync` block nested under `bundle`
3. Add a top-level `variables.version` block
4. Add a top-level `sync.paths` block listing only the files to sync
5. Add `file_path` with `v${var.version}` to the workspace target
6. Run `databricks bundle sync` and verify output

## Validation

After sync, terminal output should show:
```
Starting synchronization (N files)
Uploaded path/to/notebook.ipynb
Completed synchronization
```

If you see `Warning: unknown field: sync at bundle` — the `sync` block is still nested under `bundle`. Move it to the top level.
