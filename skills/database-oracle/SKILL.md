---
name: database-oracle
description: Oracle Database (RDS SE2, OCI Autonomous, on-premise) — SID vs service name, username-as-schema architecture, user creation and grants, LogMiner CDC setup (ARCHIVELOG, supplemental logging, permissions), table-level supplemental logging, heartbeat table, sqlplus conventions, Oracle password constraints, INTPK/DTIX test table definitions, platform differences (AWS RDS vs OCI Autonomous vs on-premise). Use when creating Oracle users, enabling LogMiner CDC, writing sqlplus scripts, understanding SID/catalog naming, debugging ORA- errors, or setting up Oracle replication.
---

# Oracle Database Reference

## Cloud platform support

| Platform | Managed Oracle service | Versions available | Status |
|----------|----------------------|--------------------|--------|
| **OCI** | Autonomous Database (Always Free) | 19c, 21c, 23ai | ✅ Supported |
| **AWS** | RDS Oracle SE2 | **19c, 21c only** | ✅ Supported |
| **Azure** | None | N/A | ❌ No managed Oracle — VM required |
| **GCP** | None | N/A | ❌ No Cloud SQL for Oracle — VM required |

- **OCI Always Free**: Forever, 2 instances, 1 OCPU, 20GB, $0 — best for long-term dev
- **AWS RDS Free Tier**: 12 months, `db.t3.micro`; after expiry ~$40/month for SE2
- Provisioning: OCI 3–5 min; AWS RDS 10–15 min + `sleep 90` wait

### AWS RDS Oracle: version alias pitfall ⚠️

The `lts` and `latest` version aliases both resolve to **23ai**, which has `rds_value=""` and is **not available on AWS RDS**. The setup function exits if `rds_value` is empty.

Always pass an explicit version for AWS RDS Oracle:
```bash
python cli/setup_oracle_aws.py --version 19c   # RDS SE2, Oracle 19c
python cli/setup_oracle_aws.py --version 21c   # RDS SE2, Oracle 21c
python cli/setup_oracle_aws.py                 # WRONG — lts → 23ai → exits with error
```

**Why Azure and GCP require a VM for Oracle:**
Oracle's licensing model (SE2 requires CPU pinning and hard CPU limits) cannot be guaranteed by multi-tenant managed services. Neither Azure nor GCP offers a managed Oracle service. Use an Azure VM or GCP Compute Engine VM with Oracle Free or SE2, then configure via `oracle_configure(platform="vm")`.

## SID vs service name: the "catalog"

| Identifier | Format | Used in |
|-----------|--------|---------|
| SID | ≤8 chars, uppercase alphanumeric, e.g. `ORCL4A2B` | Legacy, on-premise, RDS |
| Service name | Can be longer, often matches SID on RDS | JDBC thin, sqlplus service syntax |
| PDB name | Container database (12c+) | CDB/PDB architecture |

**AWS RDS** uses SID as the service name. Connection: `sqlplus user/password@//host:port/SID`

**SID naming rule**: 1–8 chars, uppercase alphanumeric only. Pattern: `ORCL` + 4 random = `ORCL4A2B`.

## Username = schema architecture

In Oracle, **a user IS a schema**. Creating a user automatically creates a schema with the same name. There is no `CREATE SCHEMA` or `USE database` equivalent:

```sql
CREATE USER lfc_user IDENTIFIED BY "password";
CREATE TABLE intpk (...);   -- same as lfc_user.intpk
```

- `schema_name` in Oracle = username (always uppercase)
- Tables created by DBA must be prefixed: `CREATE TABLE lfc_user.intpk (...)`
- Access other user's objects: `other_user.table_name`

## Oracle password constraints

Characters **not allowed**: `/` `"` `@` `&` space

**Minimum (AWS RDS)**: 8–30 chars, ≥1 uppercase, ≥1 lowercase, ≥1 digit, ≥1 special character.

**Terraform Oracle-safe `override_special`:**
```hcl
override_special = "!#$%*()-_=+[]{}|:;<>,.?"   # excludes / " @ & space
```

**Usernames must start with a letter:**
```hcl
local.username     = "u${random_string.username_base.result}"
local.dba_username = "a${random_string.dba_base.result}"
```

## ARCHIVELOG mode (required for LogMiner CDC)

```sql
SELECT LOG_MODE FROM V$DATABASE;   -- ARCHIVELOG or NOARCHIVELOG
```

AWS RDS and OCI ADB: always enabled automatically. On-premise: requires DB restart.

For on-premise enable commands, full supplemental logging setup, LogMiner permissions, heartbeat table, and CDC readiness check: see [user-grants-and-cdc.md](user-grants-and-cdc.md).

## Supplemental logging — quick reference

Three levels required (database-level) + one per table:

```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
-- Per table:
ALTER TABLE schema.INTPK ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER TABLE schema.DTIX  ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

`ORA-32589` = already enabled — treat as success.

For full supplemental logging detail, LogMiner grants, heartbeat table, and CDC readiness queries: see [user-grants-and-cdc.md](user-grants-and-cdc.md).

## JDBC connection URL

```
jdbc:oracle:thin:@host:port:SID          -- legacy SID format
jdbc:oracle:thin:@//host:port/SID        -- EZCONNECT format (preferred)
jdbc:oracle:thin:@//host:port/service    -- service name format
```

## Platform differences

| Feature | AWS RDS SE2 | OCI Autonomous DB | On-premise |
|---------|-------------|-------------------|------------|
| ARCHIVELOG | Always on | Always on | Manual, restart needed |
| DBA user | Master user (e.g. `admin`) | `ADMIN` | `SYS`, `SYSTEM` |
| CDB/PDB | No (SE2 non-CDB) | Yes (ADB is PDB) | Optional (12c+) |
| Max SID length | 8 chars | Service name | 8 chars |
| Wallet required | No | Yes | No |

For OCI Autonomous Database specifics (wallet, service name, password rules, API key auth), AWS RDS Terraform config, ready-check pattern, test table DDL, and sqlplus conventions: see [platform-specifics.md](platform-specifics.md).

## Common ORA- errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `ORA-00972` | Identifier too long | SID exceeds 8 chars — truncate |
| `ORA-00988` | Missing or invalid password | Username starts with digit, OR password contains `/` `@` `"` space |
| `ORA-12170` | TNS: Connect timeout | Oracle still initializing — wait 90s initial + retry 15×30s |
| `ORA-00955` | Name already used by existing object | Treat as success (idempotent) |
| `ORA-32589` | Supplemental logging already enabled | Treat as success (idempotent) |
| `ORA-01917` | User does not exist | REVOKE/GRANT on non-existent user — Oracle names are uppercase |
| `ORA-01950` | No privileges on tablespace | User needs `GRANT UNLIMITED TABLESPACE` |
