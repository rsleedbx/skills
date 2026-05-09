---
name: database-postgres
description: PostgreSQL 14/15/16 — authentication models (peer, scram-sha-256, md5), pg_hba.conf, listen_addresses, required LFC user grants and WAL parameters, replica identity (DEFAULT/FULL/NOTHING), replication slot cleanup, scram-sha-256 rejects empty passwords, identifier quoting with double quotes, AWS RDS vs Azure vs GCP REPLICATION grant differences, cross-platform wal_level. Use when creating PostgreSQL users, configuring replication, writing grants, managing pg_hba.conf, debugging authentication, understanding replica identity settings, or cleaning up replication slots.
---

# PostgreSQL — Database Reference

## Authentication models

PostgreSQL authentication is controlled per-connection-type by `pg_hba.conf`:

| Method | When used | How to connect |
|--------|-----------|----------------|
| `peer` | Local Unix socket, Unix user matches DB user | `sudo -u postgres psql` — no password |
| `scram-sha-256` | TCP connections (PostgreSQL 14+ default) | `PGPASSWORD=... psql -h host` |
| `md5` | TCP connections (legacy, pre-14 default) | `PGPASSWORD=... psql -h host` |
| `trust` | Local or specified CIDR, no auth | Development only, never production |

**Peer auth is always available on the local socket.** `sudo -u postgres psql` always works regardless of the TCP password — this is the admin fallback channel.

## scram-sha-256 rejects empty passwords

`ALTER USER postgres WITH PASSWORD ''` will set an empty password, but scram-sha-256 authentication rejects empty passwords on connection. The effect is that the user appears to exist but can never authenticate via TCP.

**Always generate a non-empty password before any `ALTER USER ... WITH PASSWORD` call:**

```bash
if [[ -z "${PG_PASSWORD:-}" ]]; then
  PG_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 15; echo)_
fi
# Now safe to use PG_PASSWORD in ALTER USER
```

## pg_hba.conf: remote TCP access

Default Ubuntu/Debian installations only allow local connections. To allow remote TCP:

```
# Allow all users, all databases, from any IPv4, using scram-sha-256
host  all  all  0.0.0.0/0  scram-sha-256
```

File location: `/etc/postgresql/<version>/main/pg_hba.conf`

Use a glob to find it across versions:
```bash
HBA=$(ls /etc/postgresql/*/main/pg_hba.conf | head -1)
```

Restart PostgreSQL to apply pg_hba.conf changes (or `SELECT pg_reload_conf()` for many changes, but host additions require restart when combined with `listen_addresses`).

## listen_addresses: accept remote connections

Default: `localhost` (loopback only). Change to `'*'` to accept connections from all interfaces:

```
# In postgresql.conf
listen_addresses = '*'
```

File: `/etc/postgresql/<version>/main/postgresql.conf`

```bash
CFG=$(ls /etc/postgresql/*/main/postgresql.conf | head -1)
sed -i "s/^#*listen_addresses.*/listen_addresses = '*'/" "$CFG"
# or append if the line doesn't exist:
echo "listen_addresses = '*'" >> "$CFG"
systemctl restart postgresql
```

Both `listen_addresses` and `pg_hba.conf` must be changed together — listening on all interfaces without a permissive `pg_hba.conf` rule still rejects remote connections.

## Required LFC user grants

```sql
-- Step 1: DBA grants (run connected to postgres or admin DB)
CREATE USER <lfc_user> WITH PASSWORD '<password>';
GRANT CONNECT ON DATABASE "<catalog>" TO <lfc_user>;
GRANT ALL PRIVILEGES ON DATABASE "<catalog>" TO <lfc_user>;

-- Step 2: Schema-level grants (must be run connected to the catalog DB)
GRANT USAGE, CREATE ON SCHEMA public TO <lfc_user>;

-- Step 3: Existing tables — full DML
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
  ON ALL TABLES IN SCHEMA public TO <lfc_user>;

-- Step 4: Future tables — default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON TABLES TO <lfc_user>;

-- Step 5: Replication role (platform-specific — see below)
ALTER ROLE <lfc_user> WITH REPLICATION;   -- Azure / on-premise
-- AWS RDS: GRANT rds_replication TO <lfc_user>;
```

> `GRANT ALL PRIVILEGES ON DATABASE` covers only database-level rights (CONNECT, CREATE SCHEMA, TEMP) — it does NOT grant DML on tables. Steps 2–4 are required separately.

## Table ownership: tables must be owned by lfc_user

`ALTER TABLE ... REPLICA IDENTITY` requires table ownership. If tables were created by the DBA (e.g. in an earlier run), transfer ownership:

```sql
DO $$ DECLARE r RECORD;
BEGIN
  FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  LOOP
    EXECUTE 'ALTER TABLE public.' || r.tablename || ' OWNER TO lfc_user';
  END LOOP;
END $$;
```

The safest pattern is to **create tables as `lfc_user`** from the start:

```bash
# Connect as lfc_user for table creation
PGPASSWORD="$LFC_PASSWORD" psql "postgresql://${LFC_USER}@${HOST}:${PORT}/${CATALOG}" <<'EOF'
CREATE TABLE IF NOT EXISTS public.intpk (...);
CREATE TABLE IF NOT EXISTS public.dtix  (...);
ALTER TABLE public.intpk REPLICA IDENTITY DEFAULT;
ALTER TABLE public.dtix  REPLICA IDENTITY FULL;
EOF
```

## Create as DBA, do everything else as LFC user

```bash
# Step 1: DBA creates user and grants
_pc_sql postgres <<EOF
CREATE USER ${LFC_USER} WITH PASSWORD '${LFC_PASSWORD}';
GRANT CONNECT ON DATABASE "${catalog}" TO ${LFC_USER};
GRANT ALL PRIVILEGES ON DATABASE "${catalog}" TO ${LFC_USER};
ALTER ROLE ${LFC_USER} WITH REPLICATION;
EOF

# Step 2: switch to catalog DB for schema-level grants
_pc_sql "$catalog" <<EOF
GRANT USAGE, CREATE ON SCHEMA public TO ${LFC_USER};
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA public TO ${LFC_USER};
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON TABLES TO ${LFC_USER};
EOF

# Step 3: verify lfc_user login
PGPASSWORD="$LFC_PASSWORD" psql "postgresql://${LFC_USER}@${HOST}:${PORT}/${catalog}" \
  --command "SELECT 1"

# Step 4: tables and replica identity as lfc_user (owner required for REPLICA IDENTITY)
PGPASSWORD="$LFC_PASSWORD" psql "postgresql://${LFC_USER}@${HOST}:${PORT}/${catalog}" <<'EOF'
CREATE TABLE IF NOT EXISTS public.intpk (...);
CREATE TABLE IF NOT EXISTS public.dtix  (...);
ALTER TABLE public.intpk REPLICA IDENTITY DEFAULT;
ALTER TABLE public.dtix  REPLICA IDENTITY FULL;
EOF
```

## Required server parameters for LFC

| Parameter | Value | Notes |
|-----------|-------|-------|
| `wal_level` | `logical` | enables logical decoding for CDC; requires server restart |
| `require_secure_transport` | `off` | LFC does not support SSL/TLS |

Both require a server restart on most platforms. Batch parameter changes and do a single restart.

**Platform-specific commands:**
```bash
# Azure Flexible Server
az postgres flexible-server parameter set \
  --resource-group "$RG" --server-name "$SERVER" \
  --name wal_level --value logical --output none

# On-premise / VM
echo "wal_level = logical" >> /etc/postgresql/16/main/postgresql.conf
systemctl restart postgresql

# AWS RDS — via parameter group (full set required for LFC)
aws rds modify-db-parameter-group \
  --db-parameter-group-name "$PARAM_GROUP" \
  --parameters "[
    {\"ParameterName\":\"rds.logical_replication\",\"ParameterValue\":\"1\",\"ApplyMethod\":\"pending-reboot\"},
    {\"ParameterName\":\"rds.force_ssl\",\"ParameterValue\":\"0\",\"ApplyMethod\":\"pending-reboot\"},
    {\"ParameterName\":\"max_replication_slots\",\"ParameterValue\":\"10\",\"ApplyMethod\":\"pending-reboot\"},
    {\"ParameterName\":\"max_wal_senders\",\"ParameterValue\":\"15\",\"ApplyMethod\":\"pending-reboot\"},
    {\"ParameterName\":\"max_worker_processes\",\"ParameterValue\":\"10\",\"ApplyMethod\":\"pending-reboot\"},
    {\"ParameterName\":\"max_slot_wal_keep_size\",\"ParameterValue\":\"10240\",\"ApplyMethod\":\"pending-reboot\"}
  ]"

# GCP Cloud SQL
gcloud sql instances patch "$INSTANCE" \
  --database-flags cloudsql.logical_decoding=on
```

> **AWS RDS parameter values:** `max_wal_senders=15` (not 10), `max_slot_wal_keep_size=10240` (10 GiB cap to prevent unbounded WAL growth). Apply all six params together and reboot once.

## Replica identity: what goes into CDC events

| Setting | CDC event includes | Use when |
|---------|-------------------|----------|
| `DEFAULT` | Changed columns + PK | table has a primary key (intpk) |
| `FULL` | All columns (before + after image) | table has no PK (dtix, event tables) |
| `NOTHING` | Nothing | table is excluded from CDC |
| `INDEX idx` | Columns in specified unique index | table has unique index but no PK |

```sql
ALTER TABLE intpk REPLICA IDENTITY DEFAULT;  -- uses PK
ALTER TABLE dtix  REPLICA IDENTITY FULL;     -- all columns
```

> LFC requires `FULL` or `DEFAULT` on every tracked table. `NOTHING` disables CDC for that table.

## Replication slot cleanup

LFC creates named replication slots. Abandoned slots hold WAL and grow disk indefinitely.

**Naming pattern:** `dbx_%_<GATEWAY_PIPELINE_ID>`

```sql
-- List LFC slots
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
  AS lag
FROM pg_replication_slots
WHERE slot_name LIKE 'dbx_%';

-- Drop all slots for a specific gateway pipeline
SELECT pg_drop_replication_slot(slot_name)
FROM pg_replication_slots
WHERE slot_name LIKE 'dbx_%_<GATEWAY_PIPELINE_ID>';
```

Always drop slots when destroying or recreating a gateway pipeline.

## REPLICATION role: platform differences

| Platform | Grant |
|----------|-------|
| PostgreSQL on-premise / Azure Flexible Server | `ALTER ROLE lfc_user WITH REPLICATION;` |
| AWS RDS | `GRANT rds_replication TO lfc_user;` (the REPLICATION attribute is not available to non-superusers) |
| GCP Cloud SQL | `ALTER ROLE lfc_user WITH REPLICATION;` (available to `cloudsqlsuperuser`) |

> **Run both grants together** — the lakeflow_connect reference implementation runs `ALTER ROLE ... WITH REPLICATION` AND `GRANT rds_replication` in the same block. On non-RDS platforms, the `GRANT rds_replication` fails silently (role doesn't exist). On RDS, `ALTER ROLE WITH REPLICATION` fails silently (not permitted). Running both makes the script portable across platforms without conditionals.

```sql
ALTER ROLE <lfc_user> WITH REPLICATION;   -- succeeds on Azure / on-premise / GCP
GRANT rds_replication TO <lfc_user>;      -- succeeds on AWS RDS (fails silently elsewhere)
```

## Cross-platform grant compatibility probe (GCP Cloud SQL)

GCP Cloud SQL requires a `cloudsqlsuperuser` intermediary step before `REPLICATION` can be granted. A portable setup script issues both and uses the results as a platform probe:

```python
# Step 1: attempt cloudsqlsuperuser grant (GCP-specific; no-op / error on other platforms)
try:
    cursor.execute(f"GRANT cloudsqlsuperuser TO {lfc_user}")
except Exception:
    pass   # not GCP — continue

# Step 2: run both replication grants (each silently succeeds on the right platform)
cursor.execute(f"ALTER ROLE {lfc_user} WITH REPLICATION")       # Azure / on-premise / GCP
try:
    cursor.execute(f"GRANT rds_replication TO {lfc_user}")      # AWS RDS
except Exception:
    pass
```

The sequential pattern — try `cloudsqlsuperuser`, then try both replication grants — works on all three platforms without any `IF`/platform-detection logic in the caller.

## Test tables: intpk and dtix (PostgreSQL canonical DDL)

From the lakeflow_connect reference implementation (`postgres/02_postgres_configure.sh`):

```sql
CREATE TABLE IF NOT EXISTS <schema>.intpk (
  pk SERIAL PRIMARY KEY,
  dt TIMESTAMP
);

CREATE TABLE IF NOT EXISTS <schema>.dtix (
  dt TIMESTAMP
);
```

Key PostgreSQL specifics:
- `SERIAL` = auto-increment integer PK (PostgreSQL-specific)
- `dtix` has **no `pk` column** and **no `ops` column** — just `dt TIMESTAMP`
- No `DEFAULT CURRENT_TIMESTAMP` — `dt` must be supplied in INSERT
- These are intentionally minimal for CDC testing

## Replica identity: per-table based on CDC_CT_MODE

Set via `pg_class.relreplident`. The lakeflow_connect reference uses `CDC_CT_MODE` to decide:

| CDC_CT_MODE | dtix | intpk |
|-------------|------|-------|
| `BOTH` or `CDC` | `FULL` (`f`) | `DEFAULT` (`d`) |
| `BOTH` or `CT` | `NOTHING` (`n`) — no CDC on dtix | `DEFAULT` (`d`) |
| `NONE` | `NOTHING` (`n`) | `NOTHING` (`n`) |

```bash
# Check current replica identity for both tables
psql -c "
  SELECT nspname, relname, relreplident
  FROM pg_class AS c
  JOIN pg_namespace AS ns ON c.relnamespace = ns.oid
  WHERE nspname = '$DB_SCHEMA' AND relname IN ('dtix', 'intpk')
"
# relreplident values: d=DEFAULT, f=FULL, n=NOTHING, i=INDEX
# Grep patterns used in lakeflow_connect reference:
#   "${DB_SCHEMA},dtix,f"   → FULL (expected when CDC_CT_MODE=BOTH|CDC)
#   "${DB_SCHEMA},dtix,n"   → NOTHING (expected when CDC_CT_MODE=NONE)
#   "${DB_SCHEMA},intpk,d"  → DEFAULT (expected when CDC_CT_MODE=BOTH|CT)
#   "${DB_SCHEMA},intpk,n"  → NOTHING (expected when CDC_CT_MODE=NONE)
```

Set replica identity:
```sql
ALTER TABLE <schema>.dtix  REPLICA IDENTITY FULL;     -- for CDC_CT_MODE=BOTH|CDC
ALTER TABLE <schema>.dtix  REPLICA IDENTITY NOTHING;  -- for CDC_CT_MODE=NONE
ALTER TABLE <schema>.intpk REPLICA IDENTITY DEFAULT;  -- for CDC_CT_MODE=BOTH|CT
ALTER TABLE <schema>.intpk REPLICA IDENTITY NOTHING;  -- for CDC_CT_MODE=NONE
```

## Identifier quoting: double quotes for hyphens

PostgreSQL uses double quotes for identifiers with hyphens or reserved words:

```sql
-- BAD — syntax error at "-"
GRANT CONNECT ON DATABASE robert-lee-db TO lfc_user;

-- GOOD — double-quoted identifier
GRANT CONNECT ON DATABASE "robert-lee-db" TO lfc_user;
```

Safe alternative: use underscores (`robert_lee_db`) to avoid quoting.

## Testing a PostgreSQL connection

```bash
# TCP test with password
PGPASSWORD="$PASSWORD" psql \
  -h "$HOST" -p "$PORT" -U "$USER" -d postgres \
  --connect-timeout 10 -c "SELECT 1"

# Check WAL level
PGPASSWORD="$PASSWORD" psql -h "$HOST" -U postgres -d postgres \
  -c "SHOW wal_level;"

# Check if port is open
curl -v --connect-timeout 5 telnet://"${HOST}:${PORT}"
```
