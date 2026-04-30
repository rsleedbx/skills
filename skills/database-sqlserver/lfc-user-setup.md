# LFC User Setup — Full Flow

Complete sequence: login in `master` → user + roles in catalog → verify → create tables as lfc_user.

```bash
# Step 1: DBA creates login in master
sqlcmd -S "$HOST,$PORT" -d master -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = N'${LFC_USER}')
  CREATE LOGIN [${LFC_USER}] WITH PASSWORD = N'${LFC_PASSWORD}';
go
EOF

# Step 2: DBA creates user and grants roles in catalog
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = N'${LFC_USER}')
BEGIN
  CREATE USER [${LFC_USER}] FOR LOGIN [${LFC_USER}] WITH DEFAULT_SCHEMA = dbo;
  ALTER ROLE db_owner    ADD MEMBER [${LFC_USER}];
  ALTER ROLE db_ddladmin ADD MEMBER [${LFC_USER}];
END
go
EOF

# Step 3: verify lfc_user login — fail fast here before any further setup
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U "$LFC_USER" -P "$LFC_PASSWORD" -C -l 10 -Q "SELECT 1"

# Step 4: create tables as lfc_user (db_owner — can enable CT/CDC on them)
sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U "$LFC_USER" -P "$LFC_PASSWORD" -C <<'EOF'
IF OBJECT_ID(N'dbo.intpk', N'U') IS NULL
  CREATE TABLE [dbo].[intpk] (pk INT IDENTITY NOT NULL PRIMARY KEY, ...);
go
IF OBJECT_ID(N'dbo.dtix', N'U') IS NULL
  CREATE TABLE [dbo].[dtix] (created_at DATETIME, ...);
go
EOF
```

## Multiple routing users (public vs PNG)

When separate users are needed per network path, repeat the login+user+roles block for each. Use a helper function to avoid duplication:

```bash
_create_lfc_user() {
  local user="$1" pass="$2"

  # Login in master
  sqlcmd -S "$HOST,$PORT" -d master -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = N'${user}')
  CREATE LOGIN [${user}] WITH PASSWORD = N'${pass}';
go
EOF

  # User + roles in catalog
  sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C <<EOF
IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = N'${user}')
BEGIN
  CREATE USER [${user}] FOR LOGIN [${user}] WITH DEFAULT_SCHEMA = dbo;
  ALTER ROLE db_owner    ADD MEMBER [${user}];
  ALTER ROLE db_ddladmin ADD MEMBER [${user}];
END
go
EOF

  # Verify
  sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U "$user" -P "$pass" -C -l 10 -Q "SELECT 1"
}

_create_lfc_user "$LFC_USER_PUBLIC" "$LFC_USER_PUBLIC_PASSWORD"
_create_lfc_user "$LFC_USER_PNG"    "$LFC_USER_PNG_PASSWORD"
```

## Verify logins exist

```sql
-- Server-level logins
SELECT name FROM sys.server_principals WHERE name IN (N'lfc_user_public', N'lfc_user_png');

-- Database-level users and their roles
SELECT dp.name AS principal, r.name AS role
FROM sys.database_role_members rm
JOIN sys.database_principals dp ON rm.member_principal_id = dp.principal_id
JOIN sys.database_principals r  ON rm.role_principal_id   = r.principal_id
WHERE dp.name IN (N'lfc_user_public', N'lfc_user_png');
```
