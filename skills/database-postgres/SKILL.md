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

# AWS RDS — via parameter group
aws rds modify-db-parameter-group \
  --db-parameter-group-name "$PARAM_GROUP" \
  --parameters "ParameterName=rds.logical_replication,ParameterValue=1,ApplyMethod=pending-reboot"

# GCP Cloud SQL
gcloud sql instances patch "$INSTANCE" \
  --database-flags cloudsql.logical_decoding=on
```

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
