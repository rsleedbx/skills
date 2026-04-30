# Oracle Platform Specifics

## OCI Autonomous Database

### db_name: max 14 chars, uppercase, no hyphens

```hcl
locals {
  db_name = upper(substr(replace(local.clean_instance_id, "-", ""), 0, 14))
  # "my-instance-id" → "MYINSTANCEID" (12 chars, uppercase, hyphens stripped)
}
```

### Service name: `{db_name}_high`

OCI ADB exposes named services, not a plain SID:
```bash
SERVICE_NAME="${DB_CATALOG}_high"
sqlplus -S ADMIN/"${ADMIN_PASSWORD}"@"${SERVICE_NAME}"
```

### Wallet required

```bash
oci db autonomous-database generate-wallet \
  --autonomous-database-id "$ADB_OCID" \
  --password "$WALLET_PASSWORD" \
  --file wallet.zip
unzip wallet.zip -d wallet/
export TNS_ADMIN="$(pwd)/wallet"
```

OCI ADB connections require a wallet. `sqlplus` requires wallet directory: `sqlplus /nolog` then `CONNECT user/pass@service`.

### Single ADMIN user for both DBA and regular access

Unlike AWS RDS, OCI ADB `environment_variables` sets `USER_USERNAME = DBA_USERNAME = "ADMIN"`. Create application users post-provision if needed.

### OCI password: `min_*=2`, `&` is allowed

```hcl
resource "random_password" "oci_admin_password" {
  min_upper   = 2
  min_lower   = 2
  min_numeric = 2
  min_special = 2
  override_special = "!#$%&*()-_=+[]{}|:;<>,.?"  # & is OK for OCI
}
```

AWS RDS: `min_*=1`, do NOT include `&` (SQL*Plus treats it as substitution variable).

### OCI auth: API key pairs required

```ini
# ~/.oci/config
[DEFAULT]
user=ocid1.user.oc1..aaaaa...
fingerprint=aa:bb:cc:dd:ee:ff:...
tenancy=ocid1.tenancy.oc1..aaaaa...
region=us-ashburn-1
key_file=~/.oci/oci_api_key.pem
```

Or env vars: `OCI_TENANCY`, `OCI_USER`, `OCI_FINGERPRINT`, `OCI_REGION`, `OCI_KEY_FILE`.

### OCI Always Free tier

- Forever, 2 instances max, 1 OCPU, 20GB, $0 permanently
- Supports LogMiner
- Provisioning time: 3–5 min

---

## AWS RDS Oracle specifics

```hcl
resource "aws_db_instance" "oracle" {
  engine             = "oracle-se2"
  license_model      = "license-included"
  character_set_name = "AL32UTF8"       # Unicode — required for international data
  storage_encrypted  = false            # dev/test
  publicly_accessible = true            # required for external CDC/LFC access

  lifecycle {
    ignore_changes = [password]         # prevent drift from post-apply password changes
  }
}
```

- Security group must allow inbound port 1521 from CDC tool's egress IP
- ARCHIVELOG automatic — skip the check in setup scripts
- Master user has DBA-equivalent privileges
- AWS Free Tier: 12 months, `db.t3.micro`; after expiry ~$40/month for SE2
- Provisioning time: 10–15 min (slow — enterprise heritage + `sleep 90` initial wait)

**Ready-check after create** — Oracle RDS takes up to 8 minutes to accept connections:

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

---

## Test tables: INTPK and DTIX

### Mist Terraform seed tables (initial connectivity testing)

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

### Replication test schema (for CDC/LFC validation)

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

Use the replication variant for LFC/CDC — it includes the `OPS` column used by DML loop scripts and matches other engine conventions.

---

## sqlplus conventions

**Two valid connection formats:**

```bash
# Format 1: EZCONNECT (recommended, explicit)
sqlplus -S user/password@//host:port/SID

# Format 2: also common in mist scripts
sqlplus -S user/password@host:port/SID
```

Both work for AWS RDS; OCI ADB requires wallet + service name regardless.

```bash
# Silent mode (-S): suppresses banner/feedback for scripting
# Run SQL from heredoc
sqlplus -S "${DBA_USER}/${DBA_PASS}@//${HOST}:${PORT}/${SID}" <<'EOSQL'
SET ECHO OFF
SET FEEDBACK ON
SELECT 1 FROM DUAL;
EXIT;
EOSQL

# Error detection: Oracle errors go to stdout, not stderr
if echo "$output" | grep -q 'ORA-'; then
  echo "Oracle error: ${output}"
fi
```

**Idempotent error codes — treat as success:**
- `ORA-00955`: name already used (table/object exists)
- `ORA-32589`: supplemental logging already enabled
- `ORA-01917`: user does not exist (REVOKE from non-existent user)

**SQL*Plus settings for scripting:**
```sql
SET ECHO OFF        -- don't echo SQL statements
SET FEEDBACK ON     -- show row counts
SET PAGESIZE 0      -- suppress header/page breaks
SET HEADING OFF     -- suppress column headers
```
