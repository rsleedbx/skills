---
name: database-oracle
description: Oracle Database (RDS SE2, OCI Autonomous, on-premise) — SID vs service name, username-as-schema architecture, user creation and grants, LogMiner CDC setup (ARCHIVELOG, supplemental logging, permissions), table-level supplemental logging, heartbeat table, sqlplus conventions, Oracle password constraints, INTPK/DTIX test table definitions, platform differences (AWS RDS vs OCI Autonomous vs on-premise). Use when creating Oracle users, enabling LogMiner CDC, writing sqlplus scripts, understanding SID/catalog naming, debugging ORA- errors, or setting up Oracle replication.
---

# Oracle Database Reference

## Cloud platform support

| Platform | Managed Oracle service | Status |
|----------|----------------------|--------|
| **OCI** | Autonomous Database (Always Free) | ✅ Supported |
| **AWS** | RDS Oracle SE2 | ✅ Supported |
| **Azure** | None | ❌ No managed Oracle — would require VM |
| **GCP** | None | ❌ No Cloud SQL for Oracle — would require VM |

Azure and GCP deliberately chose not to offer managed Oracle services; they focus on PostgreSQL/MySQL. VM-based Oracle on Azure/GCP is possible but out of scope for quick-provisioning tools like Mist.

**Free tier comparison:**
- **OCI Always Free**: Forever, 2 instances max, 1 OCPU, 20GB, $0 permanently
- **AWS RDS Free Tier**: 12 months, `db.t3.micro`, after expiry ~$40/month for SE2
- Recommendation: OCI for long-term dev; AWS for short-term or AWS-integrated workloads

**Provisioning times:**
- OCI Autonomous Database: 3–5 min (fast — comparable to PostgreSQL)
- AWS RDS Oracle: 10–15 min (slow — enterprise heritage + `sleep 90` initial wait)

## SID vs service name: the "catalog"

Oracle uses different identifiers depending on version and deployment:

| Identifier | Format | Used in |
|-----------|--------|---------|
| SID | Short alphanumeric (≤8 chars), e.g. `ORCL4A2B` | Legacy, on-premise, RDS |
| Service name | Can be longer, often matches SID on RDS | JDBC thin, sqlplus service syntax |
| PDB name | Container database (12c+) — separate from SID | CDB/PDB architecture |

**AWS RDS Oracle** uses SID as the service name. sqlplus connection format:
```
sqlplus user/password@//host:port/SID
```

**SID naming rule**: 1–8 characters, uppercase alphanumeric only. Common pattern: `ORCL` + 4 random uppercase chars → `ORCL4A2B`.

## Username = schema architecture

In Oracle, **a user IS a schema**. When you create a user, Oracle automatically creates a schema with the same name. There is no `CREATE SCHEMA` or `USE database` equivalent — you create a user, then create objects in that user's schema:

```sql
-- Creating the user creates the schema
CREATE USER lfc_user IDENTIFIED BY "password";

-- Objects created as this user live in the lfc_user schema
CREATE TABLE intpk (...);          -- same as lfc_user.intpk
```

To access objects in another user's schema: `other_user.table_name`

**Implications for LFC:**
- `schema_name` in Oracle = username (always uppercase)
- There is no separate schema concept — use `USER_USERNAME.upper()` as the schema
- Tables created by DBA must be prefixed with the target schema: `CREATE TABLE lfc_user.intpk (...)`

## Oracle password constraints

Oracle passwords have strict rules. Characters that are **NOT allowed**:
- `/` (slash) — used as password/username separator in sqlplus
- `"` (double quote) — SQL string delimiter
- `@` (at sign) — connect string separator
- `&` (ampersand) — substitution variable in SQL*Plus
- space

**Minimum requirements** (AWS RDS):
- 8–30 characters
- At least 1 uppercase letter
- At least 1 lowercase letter
- At least 1 digit
- At least 1 special character

**Terraform `override_special` for Oracle-safe passwords:**
```hcl
resource "random_password" "oracle_pass" {
  length      = 16
  special     = true
  override_special = "!#$%*()-_=+[]{}|:;<>,.?"
  # Excludes: / " @ space &
}
```

**Usernames must start with a letter** (Oracle requirement):
```hcl
# BAD — may start with a digit
local.username = random_string.base.result

# GOOD — prefix with a letter
local.username     = "u${random_string.username_base.result}"
local.dba_username = "a${random_string.dba_base.result}"
```

## User creation and grants

```sql
-- Create user (check if exists first for idempotency)
DECLARE
  user_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO user_count FROM dba_users WHERE username = UPPER('lfc_user');
  IF user_count = 0 THEN
    EXECUTE IMMEDIATE 'CREATE USER lfc_user IDENTIFIED BY "password"';
  END IF;
END;
/

-- Basic access grants
GRANT CONNECT TO lfc_user;
GRANT RESOURCE TO lfc_user;
GRANT CREATE SESSION TO lfc_user;
GRANT CREATE TABLE TO lfc_user;
GRANT CREATE VIEW TO lfc_user;
GRANT CREATE SEQUENCE TO lfc_user;
GRANT CREATE PROCEDURE TO lfc_user;

-- Storage
GRANT UNLIMITED TABLESPACE TO lfc_user;

-- For replication access to data dictionary
GRANT SELECT ANY DICTIONARY TO lfc_user;
```

## ARCHIVELOG mode (required for LogMiner CDC)

LogMiner requires the database to be in ARCHIVELOG mode. Check:

```sql
SELECT LOG_MODE FROM V$DATABASE;
-- Returns: ARCHIVELOG or NOARCHIVELOG
```

| Platform | ARCHIVELOG status |
|----------|-------------------|
| AWS RDS Oracle | Always enabled automatically — no action needed |
| OCI Autonomous Database | Always enabled, cannot be disabled |
| On-premise | Must be enabled manually (requires DB restart) |

To enable on-premise:
```sql
-- Requires database restart
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

## Supplemental logging (required for LogMiner CDC)

Supplemental logging captures additional column data in redo logs for CDC. Three levels required:

```sql
-- Database-level: minimal supplemental logging (required base)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Database-level: primary key columns (for PK tables)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;

-- Database-level: all columns (for non-PK tables)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

Check current status:
```sql
SELECT SUPPLEMENTAL_LOG_DATA_MIN,
       SUPPLEMENTAL_LOG_DATA_PK,
       SUPPLEMENTAL_LOG_DATA_ALL
FROM V$DATABASE;
-- Should all be: YES
```

### Table-level supplemental logging

In addition to database-level, each tracked table needs table-level supplemental logging:

```sql
-- For tables WITH primary key
ALTER TABLE schema.INTPK ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;

-- For tables WITHOUT primary key (capture all columns)
ALTER TABLE schema.DTIX ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

Idempotent: `ORA-32589` means supplemental logging is already enabled — treat as success.

## LogMiner permissions for CDC user

```sql
-- Oracle 11g / all versions
GRANT EXECUTE_CATALOG_ROLE TO lfc_user;
GRANT SELECT ANY TRANSACTION TO lfc_user;

-- View grants (use v_ prefix, not v$ — dollar sign has special meaning in SQL*Plus)
GRANT SELECT ON v_$logmnr_contents TO lfc_user;
GRANT SELECT ON gv_$archived_log   TO lfc_user;
GRANT SELECT ON v_$logfile         TO lfc_user;
GRANT SELECT ON v_$log             TO lfc_user;
GRANT SELECT ON v_$archived_log    TO lfc_user;

-- Consistent read
GRANT FLASHBACK ANY TABLE TO lfc_user;
```

> Note the `v_$` prefix: in SQL*Plus, `$` in identifiers requires escaping. Use `v_\$logmnr_contents` in shell heredocs or `v_$logmnr_contents` when not inside a shell variable expansion context.

## Heartbeat table for CDC

A heartbeat table is required by some CDC tools (including Arcion-based systems):

```sql
CREATE TABLE lfc_user.REPLICATE_IO_CDC_HEARTBEAT (
    TIMESTAMP NUMBER PRIMARY KEY
);

GRANT SELECT, INSERT, UPDATE, DELETE
  ON lfc_user.REPLICATE_IO_CDC_HEARTBEAT TO lfc_user;

-- Insert initial record
INSERT INTO lfc_user.REPLICATE_IO_CDC_HEARTBEAT (TIMESTAMP)
  VALUES (EXTRACT(EPOCH FROM SYSTIMESTAMP) * 1000);
COMMIT;
```

Idempotent: `ORA-00955` (name already used by existing object) is acceptable — skip creation.

## Test tables: INTPK and DTIX — two DDL variants

### Mist Terraform seed tables (Terraform `user_setup.tf`)
Created by the DBA as the regular user's first tables for initial connectivity testing:

```sql
CREATE TABLE intpk (
  pk         NUMBER PRIMARY KEY,
  data       VARCHAR2(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE SEQUENCE intpk_seq START WITH 1 INCREMENT BY 1;

CREATE TABLE dtix (
  id   NUMBER,
  data VARCHAR2(255),
  ts   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Replication test schema (Python `OracleReplicationProvider`)
Richer columns for CDC/LFC validation:

```sql
CREATE TABLE SCHEMA.INTPK (
  PK  NUMBER PRIMARY KEY,
  DT  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  OPS VARCHAR2(255) DEFAULT 'insert'
);

CREATE TABLE SCHEMA.DTIX (
  PK  NUMBER,
  DT  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  OPS VARCHAR2(255) DEFAULT 'insert'
);
```

Use the replication variant when setting up LFC/CDC — it includes the `OPS` column used by DML loop scripts and matches other engine conventions.

## OCI Autonomous Database specifics

OCI ADB is "always on" ARCHIVELOG and supplemental logging is available, but several details differ from AWS RDS:

### db_name: max 14 chars, uppercase, no hyphens

```hcl
# OCI imposes a 14-char max (vs 8-char SID on RDS)
locals {
  db_name = upper(substr(replace(local.clean_instance_id, "-", ""), 0, 14))
  # "my-instance-id" → "MYINSTANCEID" (12 chars, uppercase, hyphens stripped)
}
```

### Service name: `{db_name}_high`

OCI ADB exposes named services, not a plain SID. The "high priority" service is `{db_name}_high`:

```bash
# Connect using the service name (requires wallet)
SERVICE_NAME="${DB_CATALOG}_high"
sqlplus -S ADMIN/"${ADMIN_PASSWORD}"@"${SERVICE_NAME}"
```

### Wallet required for OCI ADB

OCI ADB connections require a wallet. Download with:
```bash
oci db autonomous-database generate-wallet \
  --autonomous-database-id "$ADB_OCID" \
  --password "$WALLET_PASSWORD" \
  --file wallet.zip
unzip wallet.zip -d wallet/
export TNS_ADMIN="$(pwd)/wallet"
```

### OCI: single ADMIN user for both DBA and regular access

Unlike AWS RDS (where you create a separate regular user), OCI ADB `environment_variables` output sets `USER_USERNAME = DBA_USERNAME = "ADMIN"`. Create application users post-provision if needed.

### OCI password: `min_*=2`, and `&` is allowed

OCI ADB requires at least 2 uppercase, 2 lowercase, 2 digits, 2 special chars. The `&` character **is allowed** in OCI passwords (unlike on-premise Oracle where it's a SQL*Plus substitution variable):

```hcl
resource "random_password" "oci_admin_password" {
  min_upper   = 2
  min_lower   = 2
  min_numeric = 2
  min_special = 2
  override_special = "!#$%&*()-_=+[]{}|:;<>,.?"  # & is OK for OCI
}
```

AWS RDS Oracle: `min_*=1`, do NOT include `&` (SQL*Plus treats it as substitution variable).

### OCI auth: API key pairs required

OCI uses RSA key pairs, not username/password or bearer tokens:

```ini
# ~/.oci/config
[DEFAULT]
user=ocid1.user.oc1..aaaaa...
fingerprint=aa:bb:cc:dd:ee:ff:...
tenancy=ocid1.tenancy.oc1..aaaaa...
region=us-ashburn-1
key_file=~/.oci/oci_api_key.pem
```

Or set env vars: `OCI_TENANCY`, `OCI_USER`, `OCI_FINGERPRINT`, `OCI_REGION`, `OCI_KEY_FILE`.

## AWS RDS Oracle specifics

```hcl
resource "aws_db_instance" "oracle" {
  engine             = "oracle-se2"     # Oracle Standard Edition 2
  license_model      = "license-included"
  character_set_name = "AL32UTF8"       # Unicode character set (required for international data)
  storage_encrypted  = false            # Disabled for cost optimization (dev/test)
  publicly_accessible = true            # Required for external CDC/LFC access

  lifecycle {
    ignore_changes = [password]         # Prevent drift from post-apply password changes
  }
}
```

**Ready-check after create** — Oracle RDS takes up to 8 minutes to accept connections after Terraform reports "applied":

```bash
sleep 90  # Initial wait — Oracle initialization
for i in {1..15}; do
  if echo "SELECT 1 FROM DUAL; EXIT;" \
     | sqlplus -S "${DBA_USER}/${DBA_PASS}@//${HOST}:${PORT}/${SID}" > /dev/null 2>&1; then
    echo "Oracle RDS ready"
    break
  fi
  sleep 30  # 15 × 30s = up to 7.5 more minutes
done
```

## sqlplus conventions

**Two valid sqlplus connection formats** — both work, choose based on context:

```bash
# Format 1: EZCONNECT with double-slash (recommended, explicit)
sqlplus -S user/password@//host:port/SID

# Format 2: host:port/SID without double-slash (also common in mist scripts)
sqlplus -S user/password@host:port/SID
```

Mist code uses Format 2 in the Python replication provider (`f"{host}:{port}/{catalog}"`). Format 1 is the "official" Easy Connect Plus style. Both work for AWS RDS; OCI ADB requires wallet + service name regardless.

```bash
# Connection format — note double-slash before host (EZCONNECT syntax)
sqlplus -S user/password@//host:port/SID

# Silent mode (-S): suppresses banner/feedback for scripting
# Non-silent: connects and shows banner

# Run SQL from heredoc in bash
sqlplus -S "${DBA_USER}/${DBA_PASS}@//${HOST}:${PORT}/${SID}" <<'EOSQL'
SET ECHO OFF
SET FEEDBACK ON
SELECT 1 FROM DUAL;
EXIT;
EOSQL

# Error detection: check stdout for ORA- prefix (Oracle errors go to stdout, not stderr)
if echo "$output" | grep -q 'ORA-'; then
  echo "Oracle error: $output"
fi

# Allow specific errors (idempotency):
# ORA-00955: name already used (table/object exists) → CREATE TABLE IF NOT EXISTS equivalent
# ORA-32589: supplemental logging already enabled → ALTER TABLE ... ADD SUPPLEMENTAL already done
# ORA-01917: user does not exist (REVOKE from non-existent user)

# Wait for Oracle RDS to be ready (Oracle takes longer than MySQL/PostgreSQL):
for i in {1..15}; do
  if echo "SELECT 1 FROM DUAL; EXIT;" \
     | sqlplus -S "${DBA_USER}/${DBA_PASS}@//${HOST}:${PORT}/${SID}" > /dev/null 2>&1; then
    break
  fi
  sleep 30
done
```

## sqlplus SQL*Plus settings for scripting

```sql
SET ECHO OFF        -- don't echo SQL statements
SET FEEDBACK ON     -- show row counts
SET PAGESIZE 0      -- suppress header/page breaks
SET HEADING OFF     -- suppress column headers
SET FEEDBACK OFF    -- also suppress row counts for pure data extraction
```

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
| Supplemental logging | SQL commands | SQL commands | SQL commands |
| DBA user | Master user (e.g. `admin`) | `ADMIN` | `SYS`, `SYSTEM` |
| Supplemental logging scope | Full control | Full control | Full control |
| CDB/PDB | No (SE2 is non-CDB) | Yes (ADB is PDB) | Optional (12c+) |
| Max SID length | 8 chars | Service name | 8 chars |

**AWS RDS-specific:**
- `publicly_accessible = true` required for external CDC tool connections
- Security group must allow inbound port 1521 from CDC tool's egress IP
- ARCHIVELOG automatic — skip the check in setup scripts
- Master user has DBA-equivalent privileges

**OCI Autonomous Database-specific:**
- Connection requires wallet download: `oci db autonomous-database generate-wallet`
- sqlplus requires wallet directory: `sqlplus /nolog` then `CONNECT user/pass@service`
- Always Free tier supports LogMiner

## Common ORA- errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `ORA-00972` | Identifier too long | SID exceeds 8 chars — truncate to `ORCL` + 4 chars |
| `ORA-00988` | Missing or invalid password | Username starts with a digit, OR password contains forbidden chars (`/` `@` `"` space) |
| `ORA-12170` | TNS: Connect timeout | Oracle still initializing — wait 90s initial + retry 15×30s |
| `ORA-00955` | Name already used by existing object | Table/sequence already exists — treat as success (idempotent) |
| `ORA-32589` | Supplemental logging already enabled | `ALTER TABLE ... ADD SUPPLEMENTAL LOG DATA` already done — treat as success |
| `ORA-01917` | User does not exist | REVOKE/GRANT on non-existent user — check spelling, remember Oracle names are uppercase |
| `ORA-01950` | No privileges on tablespace | User needs `GRANT UNLIMITED TABLESPACE` or quota on specific tablespace |

## Checking CDC readiness

```sql
-- 1. Verify ARCHIVELOG mode
SELECT LOG_MODE FROM V$DATABASE;

-- 2. Verify supplemental logging
SELECT SUPPLEMENTAL_LOG_DATA_MIN AS MINIMAL,
       SUPPLEMENTAL_LOG_DATA_PK  AS PRIMARY_KEY,
       SUPPLEMENTAL_LOG_DATA_ALL AS ALL_COLS
FROM V$DATABASE;

-- 3. Verify user exists and has required privileges
SELECT GRANTED_ROLE FROM DBA_ROLE_PRIVS WHERE GRANTEE = UPPER('lfc_user');
SELECT PRIVILEGE FROM DBA_SYS_PRIVS    WHERE GRANTEE = UPPER('lfc_user');

-- 4. Verify table-level supplemental logging
SELECT LOG_GROUP_TYPE, ALWAYS
FROM ALL_LOG_GROUPS
WHERE OWNER = UPPER('lfc_user')
AND TABLE_NAME IN ('INTPK', 'DTIX');
```
