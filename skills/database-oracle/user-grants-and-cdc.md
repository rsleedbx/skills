# Oracle User Grants and CDC Setup

## User creation (idempotent)

```sql
DECLARE
  user_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO user_count FROM dba_users WHERE username = UPPER('lfc_user');
  IF user_count = 0 THEN
    EXECUTE IMMEDIATE 'CREATE USER lfc_user IDENTIFIED BY "password"';
  END IF;
END;
/

-- Access grants
GRANT CONNECT, RESOURCE, CREATE SESSION TO lfc_user;
GRANT CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE TO lfc_user;
GRANT UNLIMITED TABLESPACE TO lfc_user;

-- Replication access
GRANT SELECT ANY DICTIONARY TO lfc_user;
```

## ARCHIVELOG mode (required for LogMiner)

Check:
```sql
SELECT LOG_MODE FROM V$DATABASE;
-- ARCHIVELOG or NOARCHIVELOG
```

| Platform | Status |
|----------|--------|
| AWS RDS Oracle | Always enabled — no action needed |
| OCI Autonomous Database | Always enabled, cannot be disabled |
| On-premise | Must be enabled manually (requires DB restart) |

On-premise enable:
```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

## Supplemental logging (required for LogMiner)

```sql
-- Database-level: three levels all required
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

Check:
```sql
SELECT SUPPLEMENTAL_LOG_DATA_MIN, SUPPLEMENTAL_LOG_DATA_PK, SUPPLEMENTAL_LOG_DATA_ALL
FROM V$DATABASE;
-- Should all be: YES
```

### Table-level supplemental logging

```sql
-- Tables WITH primary key
ALTER TABLE schema.INTPK ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;

-- Tables WITHOUT primary key (capture all columns)
ALTER TABLE schema.DTIX ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

`ORA-32589` = supplemental logging already enabled — treat as success (idempotent).

## LogMiner permissions

```sql
-- Oracle 11g / all versions
GRANT EXECUTE_CATALOG_ROLE TO lfc_user;
GRANT SELECT ANY TRANSACTION TO lfc_user;

-- View grants (use v_ not v$ — dollar sign has special meaning in SQL*Plus)
GRANT SELECT ON v_$logmnr_contents TO lfc_user;
GRANT SELECT ON gv_$archived_log   TO lfc_user;
GRANT SELECT ON v_$logfile         TO lfc_user;
GRANT SELECT ON v_$log             TO lfc_user;
GRANT SELECT ON v_$archived_log    TO lfc_user;

-- Consistent read
GRANT FLASHBACK ANY TABLE TO lfc_user;
```

> In SQL*Plus, `$` in identifiers requires escaping. Use `v_\$logmnr_contents` in shell heredocs.

## Heartbeat table

Required by some CDC tools (including Arcion-based systems):

```sql
CREATE TABLE lfc_user.REPLICATE_IO_CDC_HEARTBEAT (
    TIMESTAMP NUMBER PRIMARY KEY
);

GRANT SELECT, INSERT, UPDATE, DELETE
  ON lfc_user.REPLICATE_IO_CDC_HEARTBEAT TO lfc_user;

INSERT INTO lfc_user.REPLICATE_IO_CDC_HEARTBEAT (TIMESTAMP)
  VALUES (EXTRACT(EPOCH FROM SYSTIMESTAMP) * 1000);
COMMIT;
```

`ORA-00955` (name already used) = table already exists — treat as success (idempotent).

## Checking CDC readiness

```sql
-- 1. Verify ARCHIVELOG mode
SELECT LOG_MODE FROM V$DATABASE;

-- 2. Verify supplemental logging
SELECT SUPPLEMENTAL_LOG_DATA_MIN AS MINIMAL,
       SUPPLEMENTAL_LOG_DATA_PK  AS PRIMARY_KEY,
       SUPPLEMENTAL_LOG_DATA_ALL AS ALL_COLS
FROM V$DATABASE;

-- 3. Verify user has required privileges
SELECT GRANTED_ROLE FROM DBA_ROLE_PRIVS WHERE GRANTEE = UPPER('lfc_user');
SELECT PRIVILEGE    FROM DBA_SYS_PRIVS    WHERE GRANTEE = UPPER('lfc_user');

-- 4. Verify table-level supplemental logging
SELECT LOG_GROUP_TYPE, ALWAYS
FROM ALL_LOG_GROUPS
WHERE OWNER = UPPER('lfc_user')
AND TABLE_NAME IN ('INTPK', 'DTIX');
```
