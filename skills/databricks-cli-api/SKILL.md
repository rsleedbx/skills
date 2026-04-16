---
name: databricks-cli-api
description: Prefer the Databricks CLI `databricks api` subcommands with `--profile` and ~/.databrickscfg for workspace REST calls in scripts and automation. Use when replacing curl + DATABRICKS_HOST + DATABRICKS_TOKEN, adding shell helpers that hit the workspace API, aligning shell tools with Terraform/CLI profile auth, or Unity Catalog external storage / storage credentials / IAM trust by cloud.
---

# Databricks CLI for workspace APIs

## Default approach

For **workspace-scoped** HTTP APIs (Unity Catalog, workspace settings, jobs, etc.), use the **Databricks CLI** instead of raw `curl` with `DATABRICKS_HOST` and `DATABRICKS_TOKEN`.

- **Auth**: `host` and `token` (or OAuth where configured) live under a **`[profile]`** in `~/.databrickscfg`.
- **Profile**: pass **`-p` / `--profile`** on the CLI, or set **`DATABRICKS_PROFILE`** in the environment and have scripts pass `-p "$DATABRICKS_PROFILE"` when non-empty.
- **Do not** require callers to export `DATABRICKS_HOST` / `DATABRICKS_TOKEN` for flows that the CLI can perform.

## `databricks api` pattern

```bash
# Optional profile (omit -p to use the CLI default profile)
databricks -p "${DATABRICKS_PROFILE}" api get /api/2.1/unity-catalog/metastore_summary -o json

databricks api post /api/2.1/jobs/create --json @payload.json -o json
```

Subcommands: `get`, `post`, `put`, `patch`, `delete`, `head`. **PATH** is the URL path only (no host); the CLI resolves the workspace host from the profile.

## When curl is still reasonable

- **Non-workspace** endpoints (rare in this repo), or **headers/body** the CLI cannot express.
- **CI** that intentionally injects short-lived tokens without a config file (document that exception).

## Unity Catalog external tables / locations: who you trust (by cloud)

When wiring **external tables**, **external locations**, and **storage credentials** to customer cloud storage:

- **AWS:** The customer IAM role’s **trust policy** must allow **Databricks’ Unity Catalog master IAM role** (fixed **ARNs** published in **Databricks** docs for your **partition**, e.g. commercial vs GovCloud). There is a single documented UC master principal per partition, not “your workspace id.”
- **Azure / GCP:** You grant **your own** identity access in the storage layer (**Entra ID** service principal or **GCP** service account in ACLs / IAM), then **register that same identity** as the Unity Catalog **storage credential**. There is **no** single Databricks “master account id” to hard-code for those clouds the way there is for AWS UC master role trust.

## Repo examples

| Location | Pattern |
|----------|---------|
| `scripts/uctblextstorage/external_managed_location_uc.sh` | **`source … <command>`** for **`discover`**, **`bucket`**, **`role`** (cloud-specific; **`UCTB_CLOUD`** from **`discover`**); **`./… help`** only for usage (see `shell-script-practices`). |
| `scripts/uctblextstorage/terraform/` | Terraform `provider "databricks" { profile = … }` — same config file, different tool |

## Prerequisites

Install and upgrade the **unified** Databricks CLI (`databricks` with `api` subcommands). Legacy `databricks-cli` (Python) differs; prefer the current CLI in docs.
