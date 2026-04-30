# Script Structure — Complete Examples

## Canonical script shape

```bash
#!/usr/bin/env bash
set -euo pipefail

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
    *)
      echo "Usage: $(basename "${BASH_SOURCE[0]}") <discover|create-s3-bucket|...>" >&2
      ;;
  esac
}

# Guard: only run main when executed, not sourced (for discover which must be sourced)
[[ "${BASH_SOURCE[0]}" == "$0" ]] && main "$@"
```

**Repo example:** `scripts/uctblextstorage/external_managed_location_uc.sh` — `discover` must be sourced (`BASH_SOURCE` vs `$0` guard); `discover-create-s3-bucket` runs `_metastore_summary_json` then `cmd_create_s3_bucket` in one `./` process; shared `check_prerequisites`.

---

## Conditional argv array pattern

```bash
# Declare stable prefix as array
local -a create=(aws s3api create-bucket --bucket "${BUCKET_NAME}" --region "${AWS_REGION}")
# Conditionally append optional args
[[ "${AWS_REGION}" != "us-east-1" ]] && \
  create+=(--create-bucket-configuration "LocationConstraint=${AWS_REGION}")
# Execute — each element is one argument; safe for spaces, no eval
if ! run "${create[@]}"; then
  echo "error: create-bucket failed" >&2
  kill -INT $$
fi
```

---

## Optional flag pattern (`${var:+word}`)

```bash
# One line; no duplicated databricks … api get …
databricks ${DATABRICKS_PROFILE:+ -p "${DATABRICKS_PROFILE}"} api get "${path}" -o json
```

Note the **leading space** inside `word` so tokens stay separate (e.g. `-p`, not `-pdatabricks`).

---

## Required env var helper (SIGINT on unset)

```bash
_require_nonempty() {
  local val=$1
  shift
  if [[ -z "${val}" ]]; then
    echo "error: $*" >&2
    kill -INT $$
  fi
}

# Usage
_require_nonempty "${BUCKET_NAME:-}" "set BUCKET_NAME (globally unique S3 bucket name)"
_require_nonempty "${AWS_REGION:-}"   "set AWS_REGION (e.g. from discover .region)"
```

Use when you want **SIGINT** (not exit 1) for sourced flows — see the `kill -INT $$` rule in SKILL.md.

---

## Shared helper extraction pattern

```bash
# uc-connection-helpers.sh  ← one canonical file
_uc_connection_upsert() { … }
_build_mysql_conn_json()     { … }
_build_pg_conn_json()        { … }
_build_sqlserver_conn_json() { … }

# customer-3-setup-azure-mysql.sh
source "$(dirname "${BASH_SOURCE[0]}")/uc-connection-helpers.sh"
_uc_connection_upsert "$CONN_NAME_PNG" "$(_build_mysql_conn_json "$HOST_PNG")"
```

**Rules:**
- A function used in 2+ files → move it to a shared helper.
- Shared helpers use the same `kill -INT $$` abort pattern.
- Don't redefine a sourced function to "override" it — pass different arguments or use a variable the function reads.
