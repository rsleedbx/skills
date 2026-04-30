# Comparison Testing — Separate Identifiers Per Dimension

When testing different configurations side by side (routing paths, DB engines, network modes, compute tiers, etc.), **assign a distinct identifier at every layer** so metrics stay clean and auto-detection works without extra arguments.

## Why separate identifiers

- `performance_schema.status_by_account` (MySQL) buckets bytes by `USER` — a shared user conflates both paths.
- The pipeline spec stores exactly one `connection_name` — the connection name must encode the configuration.
- `run-logs/byte-capture.jsonl` records `conn_name` and `routing` — grouping by those fields gives per-configuration trends with no post-hoc labelling.
- Concurrent runs of two pipelines against different endpoints produce clean, non-interleaving byte counters only when the users are distinct.

## Identifier stack per comparison axis

For each variable dimension (routing, DB engine, etc.) create one dedicated identifier at each layer:

| Layer | Public path | PNG path | Why |
|-------|-------------|----------|-----|
| MySQL user | `lfc_user_public` | `lfc_user_png` | `performance_schema` buckets bytes by USER |
| UC connection | `<prefix>-<key>-public` | `<prefix>-<key>-png` | Pipeline spec stores one connection name; suffix encodes routing |
| Connection comment | `{"secrets":{"scope":"…","key":"…"}}` | same format | `databricks-run-with-byte-capture.sh` reads this to auto-detect the secret |
| Databricks secret | `<key>` (shared) | same | Single secret stores all routing user credentials under `routing.public.*` / `routing.png.*` |
| LakeFlow pipeline | `<prefix>-<key>-qbc-public` | `<prefix>-<key>-qbc-png` | One pipeline per connection; system tables correlate by pipeline ID |
| UC schema (destination) | `db_ingest_public` | `db_ingest_png` | Separate Delta tables per path; prevents MERGE conflicts on concurrent runs |

## Naming convention rule

> **Always encode the variable dimension as a suffix in the name**, using a consistent vocabulary:
> - routing path: `-public` / `-png`
> - mode: `-qbc` / `-cdc`
> - other axes: add a new suffix token (e.g. `-tls` / `-notls`, `-classic` / `-serverless`)

## Auto-detection chain (how identifiers flow at run time)

```
./databricks-run-with-byte-capture.sh <PIPELINE_ID>
  │
  ├─ GET /api/2.0/pipelines/<id>
  │    → spec.ingestion_definition.connection_name  (e.g. "robert-lee-vm-mysql-png")
  │
  ├─ suffix "-png" → routing = "png"
  │
  ├─ GET /api/2.1/unity-catalog/connections/robert-lee-vm-mysql-png
  │    → comment = {"secrets":{"scope":"png-testing","key":"vm-mysql"}}
  │
  ├─ load_db_secret "vm-mysql"
  │    → routing__png__user     = "lfc_user_png"
  │    → routing__png__password = "…"
  │    → host_fqdn, dba__user, dba__password, …
  │
  └─ snapshot performance_schema WHERE USER = 'lfc_user_png'
       → bytes are clean for this path only
```

No arguments beyond the pipeline ID are needed because the pipeline encodes its own configuration.

## Adding a new comparison dimension

When you add a new variable (e.g. TLS on vs off):

1. **MySQL**: create `lfc_user_tls` / `lfc_user_notls` (same grants as `lfc_user`).
2. **Secret**: save their passwords under `routing.tls.*` / `routing.notls.*` in the existing secret.
3. **UC connections**: create `<prefix>-<key>-tls` and `<prefix>-<key>-notls`.
4. **Pipelines**: create `<prefix>-<key>-qbc-tls` and `<prefix>-<key>-qbc-notls`.
5. **`_routing_from_conn_name`**: add `*-tls) echo "tls"` / `*-notls) echo "notls"` cases.
6. **`databricks-run-with-byte-capture.sh`**: add the new routing token to the `lfc_user` selection block.

## When a single user is acceptable

Use a shared `lfc_user` (no routing split) only when:
- Runs are strictly sequential (never concurrent), AND
- You don't need per-path byte isolation, AND
- The dimension being compared is not the routing path

Even then, encode the dimension in the connection name suffix so the auto-detection chain remains intact.
