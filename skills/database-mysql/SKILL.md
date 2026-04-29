---
name: database-mysql
description: MySQL 5.7 / 8.0 / MariaDB — authentication models (auth_socket vs password), root@localhost vs root@%, Ubuntu debian-sys-maint admin channel, required LFC user grants and binlog server parameters, bind-address configuration, identifier quoting with backticks, MySQL 8 vs 5.7 differences (caching_sha2_password, invisible PKs). Use when creating MySQL users, writing grants, configuring binlog replication, understanding auth_socket, finding the admin channel when root password is lost/stale, or debugging MySQL connectivity.
---

# MySQL — Database Reference

## Authentication models

MySQL has two separate user records for local vs remote root access:

| User | Auth method | How to connect |
|------|------------|----------------|
| `root@'localhost'` | `auth_socket` (Ubuntu) | `sudo mysql` — no password, matches Unix `root` user |
| `root@'%'` | `caching_sha2_password` / `mysql_native_password` | TCP with password from any host |

Both can exist simultaneously — they are separate user table rows. Creating `root@'%'` does not affect `root@'localhost'`. This is why `sudo mysql` may succeed while `mysql -h host -u root -p` fails: they hit different user records.

## Ubuntu admin channel: debian-sys-maint

On Debian/Ubuntu MySQL installations, `/etc/mysql/debian.cnf` stores credentials for the `debian-sys-maint` maintenance account, which always has full admin access regardless of whether root is using `auth_socket` or has an unknown password:

```bash
# Connect with admin privileges — bypasses any password auth issue
mysql --defaults-file=/etc/mysql/debian.cnf
```

Use this when:
- `sudo mysql` fails (`ERROR 1045` — root configured for password auth, not `auth_socket`)
- The DBA password in your secret is stale and you need to reset it
- Running admin commands on the VM when you don't have the current root password

The `debian.cnf` account is managed by the OS package manager (not by users) and survives MySQL reinstalls.

## Required LFC user grants

```sql
-- Schema-level: DML + DDL for LFC to create tables and run queries
-- NOTE: lakeflow_connect reference implementation uses global *.* not schema.*
GRANT ALTER, CREATE, DROP, SELECT, INSERT, DELETE, UPDATE ON *.* TO '<lfc_user>'@'%';

-- Server-level: binlog replication
GRANT REPLICATION CLIENT ON *.* TO '<lfc_user>'@'%';
GRANT REPLICATION SLAVE  ON *.* TO '<lfc_user>'@'%';
```

> **`*.*` vs `schema.*`**: The lakeflow_connect reference scripts grant on `*.*` globally (not per-schema). This is simpler for demo setups but broader than the principle of least privilege. For production, scope to the specific schema.

## Test tables: intpk and dtix (MySQL canonical DDL)

From the lakeflow_connect reference implementation (`mysql/02_mysql_configure.sh`):

```sql
CREATE TABLE IF NOT EXISTS <schema>.intpk (
  pk  SERIAL PRIMARY KEY,
  dt  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  ops VARCHAR(255) DEFAULT 'insert'
);

CREATE TABLE IF NOT EXISTS <schema>.dtix (
  pk  BIGINT,
  dt  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  ops VARCHAR(255) DEFAULT 'insert'
);
```

Key differences from other engines:
- `SERIAL` = `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT` (MySQL-specific)
- `ON UPDATE CURRENT_TIMESTAMP` — updates `dt` automatically on any row change
- `dtix` has a `pk BIGINT` column (non-primary-key — manually managed identifier for DML loops)

## endless_dml_loop stored procedure (full)

```sql
DELIMITER $$

CREATE PROCEDURE endless_dml_loop(
    IN dml_interval_sec INT,
    IN max_iterations   INT
)
BEGIN
    DECLARE counter        INT DEFAULT 0;
    DECLARE sleep_interval INT DEFAULT COALESCE(dml_interval_sec, 60);
    DECLARE stop_after     INT DEFAULT COALESCE(max_iterations, 30);

    WHILE counter < stop_after DO
        INSERT INTO intpk (dt) VALUES (CURRENT_TIMESTAMP()), (CURRENT_TIMESTAMP()), (CURRENT_TIMESTAMP());
        COMMIT;
        DELETE FROM intpk WHERE pk = (SELECT min_pk FROM (SELECT MIN(pk) AS min_pk FROM intpk) AS temp);
        COMMIT;
        UPDATE intpk SET dt = CURRENT_TIMESTAMP()
          WHERE pk = (SELECT min_pk FROM (SELECT MIN(pk) AS min_pk FROM intpk) AS temp);
        COMMIT;

        INSERT INTO dtix (pk, dt) VALUES (1, CURRENT_TIMESTAMP()), (2, CURRENT_TIMESTAMP()), (3, CURRENT_TIMESTAMP());
        COMMIT;

        SELECT CONCAT('Counter ', counter, ' of ', stop_after,
                      ' (sleeping ', sleep_interval, 's)') AS notice;
        SET counter = counter + 1;
        DO SLEEP(sleep_interval);
    END WHILE;

    SELECT CONCAT('Completed ', counter, ' iterations') AS final_notice;
END $$
```

Setup and grant:
```sql
DROP PROCEDURE IF EXISTS endless_dml_loop;  -- DBA drops first so it can always be reset
-- (CREATE PROCEDURE as above)
GRANT EXECUTE ON PROCEDURE endless_dml_loop TO '<lfc_user>';
```

Invoke: `CALL <schema>.endless_dml_loop(NULL, NULL);` (defaults: 60s interval, 30 iterations)

## Required server parameters for LFC (binlog CDC)

| Parameter | Required value | Why |
|-----------|---------------|-----|
| `binlog_row_image` | `full` | include all column values in binlog events |
| `binlog_format` | `row` | row-based (not statement or mixed) binlog |
| `require_secure_transport` | `OFF` | LFC does not support SSL/TLS |
| `sql_generate_invisible_primary_key` | `OFF` | LFC needs explicit PKs, not auto-generated invisible ones |
| `binlog_expire_logs_seconds` | `≥ 604800` | 7-day binlog retention minimum |

Check current values:
```sql
SELECT @@binlog_row_image, @@binlog_format, @@require_secure_transport;
SHOW BINARY LOGS;
```

> `binlog_expire_logs_seconds` replaces `expire_logs_days` in MySQL 8.0. Use integer seconds (`604800` = 7 days). Some managed services (AWS RDS) set binlog format via parameter groups, not `SET GLOBAL`.

## bind-address: remote access configuration

MySQL on Ubuntu/Debian binds to `127.0.0.1` (loopback only) by default. To accept remote connections, change `bind-address`:

```
File: /etc/mysql/mysql.conf.d/mysqld.cnf
Setting: bind-address = 0.0.0.0
```

Idempotent check and fix:
```bash
CFG=/etc/mysql/mysql.conf.d/mysqld.cnf
if grep -qE '^bind-address[[:space:]]*=[[:space:]]*0\.0\.0\.0' "$CFG"; then
  echo "already 0.0.0.0 — skipping"
else
  sed -i 's/bind-address.*/bind-address = 0.0.0.0/' "$CFG"
  systemctl restart mysql
fi
```

> `[[:space:]]` instead of `\s` — `\s` is not POSIX and fails under `/bin/sh` (dash).

## root@'%' user: create for TCP access

After MySQL installation, `root@'%'` may not exist — only `root@'localhost'` with `auth_socket`. Create it explicitly:

```sql
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Use `CREATE USER IF NOT EXISTS` so the statement is idempotent on re-runs. Follow with `ALTER USER` to reset the password if it may be stale:

```sql
ALTER USER 'root'@'%' IDENTIFIED BY '<password>';
FLUSH PRIVILEGES;
```

## Create as DBA, do everything else as LFC user

After LFC user creation and login verification, define an `lfc_user` SQL helper and use it for all subsequent operations — table creation, stored procedures. This surfaces privilege errors at setup time rather than at ingest time.

```bash
# Step 1: DBA creates user and grants
mysql --user root --host "$host" --port "$port" ... --database mysql <<EOF
CREATE USER IF NOT EXISTS '${LFC_USER}'@'%' IDENTIFIED BY '${LFC_PASSWORD}';
GRANT ALTER, CREATE, DROP, SELECT, INSERT, DELETE, UPDATE ON \`${schema}\`.* TO '${LFC_USER}'@'%';
GRANT REPLICATION CLIENT ON *.* TO '${LFC_USER}'@'%';
GRANT REPLICATION SLAVE  ON *.* TO '${LFC_USER}'@'%';
FLUSH PRIVILEGES;
EOF

# Step 2: verify lfc_user can log in (fail fast if misconfigured)
MYSQL_PWD="$LFC_PASSWORD" mysql \
  --user "$LFC_USER" --host "$host" --port "$port" \
  --connect-timeout 10 --execute "SELECT 1"

# Step 3: all subsequent SQL runs as lfc_user (catches privilege gaps at setup time)
MYSQL_PWD="$LFC_PASSWORD" mysql \
  --user "$LFC_USER" --host "$host" --port "$port" \
  --database "$schema" <<'EOF'
CREATE TABLE IF NOT EXISTS intpk (...);
CREATE TABLE IF NOT EXISTS dtix  (...);
EOF
```

**DBA-only operations** (always run as admin, not LFC user):
- `CREATE USER`, `GRANT`, `FLUSH PRIVILEGES`
- `SET GLOBAL binlog_format = ...` (parameter changes)

## Identifier quoting: backticks for hyphens

MySQL uses backticks to quote identifiers with hyphens or reserved words:

```sql
-- BAD — syntax error
GRANT SELECT ON robert-lee-db.* TO 'user'@'%';

-- GOOD — backtick-quoted
GRANT SELECT ON `robert-lee-db`.* TO 'user'@'%';
```

Safe alternative: use underscores (`robert_lee_db`) to avoid quoting entirely.

## Testing a MySQL connection

```bash
# TCP test with password
MYSQL_PWD="$PASSWORD" mysql \
  --user "$USER" --host "$HOST" --port "$PORT" \
  --connect-timeout 10 --execute "SELECT 1"

# Check if port is open (no MySQL client needed)
curl -v --connect-timeout 5 telnet://"${HOST}:${PORT}"
```

## MySQL 8 vs MySQL 5.7 differences

| Feature | MySQL 5.7 | MySQL 8.0 |
|---------|-----------|-----------|
| Default auth plugin | `mysql_native_password` | `caching_sha2_password` |
| Invisible PKs | Not available | `sql_generate_invisible_primary_key` (must set OFF for LFC) |
| Binlog expiry | `expire_logs_days` (days) | `binlog_expire_logs_seconds` (seconds) |
| `require_secure_transport` default | `OFF` | `OFF` (but some managed services enable it) |

> **`caching_sha2_password` compatibility**: Older MySQL clients may not support it. If clients get "Authentication plugin not supported" errors, either upgrade the client or set `default_authentication_plugin=mysql_native_password` in the server config.

## MariaDB differences

MariaDB uses the same binlog parameters as MySQL for LFC but:
- Uses `MASTER_LOG_FILE` / `MASTER_LOG_POS` (not `SOURCE_LOG_FILE` / `SOURCE_LOG_POS`)
- `REPLICATION SLAVE` grant (MySQL 8 also accepts `REPLICATION REPLICA`)
- No `sql_generate_invisible_primary_key` (the parameter does not exist)
- `expire_logs_days` instead of `binlog_expire_logs_seconds`
