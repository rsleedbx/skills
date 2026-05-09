---
name: gcp-cloud-sql
description: >-
  GCP Cloud SQL provisioning via REST API and Application Default Credentials (ADC) —
  httpx client setup, operations polling, authorized networks read-modify-write,
  engine-specific databaseVersion enums, instance naming constraints, Users/Databases APIs,
  engine gotchas (MySQL lowercase names, PostgreSQL PITR, SQL Server gcloudsql_cdc),
  and which engines are NOT available on Cloud SQL (Oracle, DB2). Use when provisioning
  or tearing down Cloud SQL instances for MySQL, PostgreSQL, or SQL Server; configuring
  replication parameters; or understanding GCP-specific vs AWS/Azure differences.
---

# GCP Cloud SQL — Provisioning Reference

## Auth and required config

**API:** Cloud SQL Admin REST v1beta4 — `https://sqladmin.googleapis.com/sql/v1beta4`

Two Python approaches to the same API surface:

| | forgedb | mist |
|---|---|---|
| HTTP client | **httpx** | **`googleapiclient.discovery.build('sqladmin', 'v1')`** |
| Auth | `google.auth.default()` → `Authorization: Bearer` | `google.auth.default()` → passed to discovery client |
| Style | Manual REST calls | Resource-style: `service.instances().patch(...).execute()` |

Both use ADC. Choose based on preference — the API surface is identical.

```python
# discovery client style (mist pattern)
from googleapiclient.discovery import build
from google.auth import default as gauth_default

creds, _ = gauth_default(scopes=["https://www.googleapis.com/auth/sqlservice.admin"])
service = build("sqladmin", "v1", credentials=creds)
service.instances().get(project=project, instance=name).execute()
service.instances().patch(project=project, instance=name, body={...}).execute()
```

No `gcloud` subprocess in either approach.

```bash
# One-time host setup: get ADC credentials
gcloud auth application-default login
```

**Required profile fields:**

| Field | Used for |
|-------|---------|
| `cfg.gcp.project` | All API calls (`/projects/{project}/...`) |
| `cfg.gcp.region` | Instance `settings.locationPreference.region` |

**`gcp_client()`** builds an httpx client with an ADC `Authorization: Bearer` token
refreshed per-request via `google.auth.default()`. No boto3, no `gcloud` subprocess.

---

## Networking model: Public IP + authorized networks

Cloud SQL instances use a public IPv4 address. Access is controlled by `authorizedNetworks`
— a list of CIDR entries in `settings.ipConfiguration`.

**Critical:** authorized networks use **read-modify-write** — always GET the current list,
append your new CIDR, then PATCH. Patching with only your new entry overwrites all existing rules.

```python
# refresh_laptop_firewall_gcp pattern
inst = gcp_instance_get(project, instance_name)
current = inst["settings"]["ipConfiguration"].get("authorizedNetworks", [])
new_entry = {"value": f"{my_ip}/32", "name": f"laptop-{date}"}
if not any(e["value"] == new_entry["value"] for e in current):
    current.append(new_entry)
    gcp_instance_patch(project, instance_name,
                       {"settings": {"ipConfiguration": {"authorizedNetworks": current}}})
    poll_gcp_op(project, op)
```

---

## Operations polling

Most mutating calls return a long-running operation. Poll until `status=DONE`:

```python
def poll_gcp_op(project, op):
    url = f"https://sqladmin.googleapis.com/sql/v1beta4/projects/{project}/operations/{op['name']}"
    while True:
        r = gcp_client().get(url)
        r.raise_for_status()
        data = r.json()
        if data["status"] == "DONE":
            if "error" in data:
                raise RuntimeError(data["error"])
            return data
        time.sleep(10)
```

---

## Resource creation order (idempotent)

```
1. GET  instance → skip if exists
2. POST instances  (create with databaseVersion, tier, flags, backupConfig, rootPassword)
3. poll_gcp_op     (wait for operation DONE)
4. poll instance ready (status=RUNNABLE + ipAddress present)
5. gcp_upsert_user  (PUT /users/{name} — admin user separate from root)
6. gcp_ensure_database (POST /databases — MySQL/PG only; SQL Server uses configure)
7. refresh_laptop_firewall_gcp (read-modify-write authorizedNetworks)
8. mysql_configure / pg_configure / sqlserver_configure
```

---

## `databaseVersion` enum (gcp_value in versions.py)

| Engine / Version | `databaseVersion` enum |
|------------------|------------------------|
| MySQL 8.0 | `MYSQL_8_0` |
| MySQL 8.4 | `MYSQL_8_4` |
| PostgreSQL 14 | `POSTGRES_14` |
| PostgreSQL 15 | `POSTGRES_15` |
| PostgreSQL 16 | `POSTGRES_16` |
| PostgreSQL 17 | `POSTGRES_17` |
| SQL Server 2019 Standard | `SQLSERVER_2019_STANDARD` |
| SQL Server 2022 Standard | `SQLSERVER_2022_STANDARD` |
| Oracle | ❌ Not available on Cloud SQL |
| DB2 | ❌ Not available on Cloud SQL |

---

## Engine-specific details

### MySQL

| Item | Value |
|------|-------|
| Instance name | **Must be lowercase** alphanumeric + hyphens; Cloud SQL rejects uppercase |
| Root user | `root` with `rootPassword` in create body |
| Admin user | Separate `gcp_upsert_user` call for `mysqladmin` or app user |
| Binlog flag name | **`expire_logs_days`** (not `binlog_expire_logs_seconds` — that's the MySQL 8 server param name, but Cloud SQL uses the old flag name) |
| Binary log | Must enable in `backupConfiguration`: `{"backupConfiguration": {"binaryLogEnabled": true, "enabled": true}}` |
| Database flags | `{"name": "binlog_row_image", "value": "full"}`, `{"name": "binlog_format", "value": "ROW"}`, etc. |
| Min storage | 10 GB |
| Idempotent user | `PUT /projects/{project}/instances/{instance}/users/{name}` (upsert) |

### PostgreSQL

| Item | Value |
|------|-------|
| Logical decoding flag | `cloudsql.logical_decoding=on` (not `wal_level=logical` — Cloud SQL uses a flag wrapper) |
| WAL archiving / PITR | Enable `pointInTimeRecoveryEnabled: true` in `backupConfiguration` |
| Replication flag | Also set: `max_replication_slots`, `max_wal_senders`, `max_slot_wal_keep_size` |
| Admin user | `cloudsqlsuperuser` — can run `ALTER ROLE lfc_user WITH REPLICATION` (unlike AWS RDS where `rds_replication` grant is required) |
| Database charset | `charset="UTF8"`, `collation=""` in `gcp_ensure_database` |

### SQL Server

| Item | Value |
|------|-------|
| Root user | Built-in **`sqlserver`** user; set `rootPassword` in create body |
| Admin user | Additional `sqladmin` created via `gcp_upsert_user` (sysadmin path) |
| Database | **Not** created via `gcp_ensure_database` — catalog creation is in `sqlserver_configure` |
| CDC platform | `platform="gcp"` → **`gcloudsql_cdc_enable_db`** (not `sp_cdc_enable_db` / `rds_cdc_enable_db`) |
| TLS | `trustServerCertificate: true` required in UC connection and sqlcmd (`-C` flag) |
| UC options | `{"trustServerCertificate": "true"}` |

---

## Users API (gcp_upsert_user)

Cloud SQL uses a Users API separate from the instance. For idempotency, use `PUT` (upsert):

```
PUT  /projects/{project}/instances/{instance}/users/{name}
Body: {"name": "user", "password": "...", "host": "%"}
```

`POST /users` creates and errors if the user exists. `PUT /users/{name}` upserts.
For MySQL, include `"host": "%"` to allow remote connections.

---

## Engine availability: Cloud SQL vs VM

| Engine | Cloud SQL | Requires VM |
|--------|-----------|-------------|
| MySQL | ✅ | No (Azure/AWS/GCP all have managed MySQL) |
| PostgreSQL | ✅ | No |
| SQL Server | ✅ | No (but no ARM64 container — x86 VM or managed service) |
| Oracle | ❌ | **Yes** — No managed Oracle on GCP or Azure. AWS RDS SE2 only (19c/21c). |
| DB2 | ❌ | **Yes** — No managed DB2 on any public cloud. Always VM or local container. |
| MariaDB | ❌ | Yes |

**Why VM for Oracle on Azure/GCP:**
- Neither Azure nor GCP offers a managed Oracle service.
- Oracle requires strict licensing compliance + specific CPU pinning that cloud-managed
  services cannot guarantee for SE2.
- On AWS, RDS Oracle SE2 is available (19c/21c) but 23ai is not on RDS.

**Why VM for DB2:**
- IBM does not license DB2 for third-party cloud managed services.
- IBM Cloud has managed Db2 Warehouse, but not standard CE/EE on AWS/Azure/GCP.
- Local: Lima QEMU VM required (DB2 needs x86-64-v2 CPU features that Rosetta 2 does not satisfy).

---

## Stop / start a Cloud SQL instance

Cloud SQL start/stop is controlled by patching `settings.activationPolicy`:

```python
# Stop
service.instances().patch(project=project, instance=name,
    body={"settings": {"activationPolicy": "NEVER"}}).execute()

# Start
service.instances().patch(project=project, instance=name,
    body={"settings": {"activationPolicy": "ALWAYS"}}).execute()
```

Poll the returned operation until `status=DONE`. After start, wait for instance
`state=RUNNABLE` before attempting connections — the IP address is present even
while the instance is resuming.

---

## Instance naming constraints

Cloud SQL instance names:
- Lowercase letters, numbers, and hyphens only
- Must start with a letter
- 1–98 characters
- **Cannot be reused** within 7 days after deletion (Cloud SQL keeps the name reserved)

In forgedb: `n.server.lower()` derives the instance name from the naming convention.
If you delete and immediately recreate with the same name, use a different epoch or wait 7 days.

---

## Tier naming (sizing)

```
db-f1-micro          # smallest, free tier eligible
db-g1-small
db-custom-{cpu}-{ram_mb}    # e.g. db-custom-1-3840 = 1 vCPU, 3.75 GB
```

`sizing.resolve_sku("gcp", ...)` returns `{"tier": "db-custom-1-3840"}` for the given size.
