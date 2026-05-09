---
name: aws-rds
description: >-
  AWS RDS provisioning via boto3 — subnet groups (≥2 AZs required), security groups,
  parameter groups vs option groups per engine, PubliclyAccessible networking, engine
  version resolution for SQL Server and Oracle, engine-specific admin usernames and gotchas,
  polling patterns, and RDS vs Aurora distinction. Use when provisioning or tearing down RDS
  instances for MySQL, PostgreSQL, SQL Server, or Oracle; choosing between RDS and Aurora;
  or understanding why forgedb targets RDS (not Aurora) for managed database provisioning.
---

# AWS RDS — Provisioning Reference

## Auth and required config

**SDK:** boto3 — `aws_session(cfg.aws.profile)` → standard AWS profile / env-var auth
(`~/.aws/credentials`, `AWS_ACCESS_KEY_ID`, or IAM role). SigV4 is handled automatically.

**Required profile fields:**

| Field | Used for |
|-------|---------|
| `cfg.aws.vpc_id` | SG creation, subnet group lookup |
| `cfg.aws.profile` | boto3 session profile name |
| `cfg.aws.region` | boto3 session region |

All three are auto-discovered from AWS CLI defaults if not set in `~/.cloudkit/<profile>.json`. Discovered values are saved to state so teardown targets the same account.

**Subnet group rule:** RDS requires a DB subnet group spanning **at least 2 subnets in different Availability Zones**. `resolve_subnet_group` reuses the existing `default` subnet group rather than creating one. If the `default` group is absent, create `forgedb-{vpc_id}` manually spanning ≥2 AZs before running setup.

---

## Networking model: PubliclyAccessible=True

forgedb uses `PubliclyAccessible=True` for all RDS instances — direct laptop + Databricks UC connection without a VPN or private endpoint. The security group restricts access to specific CIDRs.

**Firewall:** EC2 security group `authorize_security_group_ingress` per IP/CIDR.
`InvalidPermission.Duplicate` = rule already exists → treat as success (idempotent).

```python
try:
    ec2.authorize_security_group_ingress(GroupId=sg_id, IpPermissions=[rule])
except ec2.exceptions.ClientError as e:
    if e.response["Error"]["Code"] != "InvalidPermission.Duplicate":
        raise
```

---

## Resource creation order (idempotent)

```
1. ensure_parameter_group      (MySQL/PG: describe → create + apply params)
   OR ensure_option_group      (SQL Server: describe → create)
2. resolve_subnet_group        (reuse "default" or "forgedb-{vpc_id}"; never creates)
3. resolve_lfc_security_group  (find or create shared "forgedb-lfc-vpc-{short}" SG; add this engine's port rules)
4. ensure_security_group       (per-instance SG for laptop IP; describe → create if missing)
5. create_db_instance          (skip if describe_db_instances returns existing)
6. poll_rds_available          (wait for status=available)
7. refresh_laptop_firewall_aws (add current laptop IP to per-instance SG)
8. reset_master_password       (if saved_pw fails connectivity test)
```

Teardown deletes: RDS instance → per-instance SG → parameter/option group.
The shared LFC SG and subnet group are **never deleted**.

## Shared vs. per-instance resources

**Shared (one per VPC, never deleted):**

- `forgedb-lfc-vpc-{vpc_short}` — carries the 4 static Databricks LFC egress CIDRs for every engine port used in this VPC (3306 / 5432 / 1433 / 1521). Created by `resolve_lfc_security_group` on first setup; subsequent engines find it by name and add their port's rules via `ensure_sg_ingress`.
- `default` DB subnet group — the AWS-managed subnet group. `resolve_subnet_group` reuses it without modification. If absent, it looks for `forgedb-{vpc_id}`. Setup exits if neither exists (quota: 50 per account).

**Per-instance (deleted on teardown):**

- `{prefix}-{engine}-sg` — carries only the current laptop public IP. Deleted on teardown.
- `{prefix}-{engine}-params` / `{prefix}-sql-opts` — parameter or option group. Deleted on teardown.

## Security group limits

| Limit | Default |
|---|---|
| Security groups per VPC | 2,500 |
| Inbound rules per SG | 60 |
| DB subnet groups per account | 50 |

Orphaned per-instance SGs (old naming: `rble-{epoch}-{port}-sg`) can be detected by checking which SGs are not attached to any active RDS instance and deleted if unattached. A `DependencyViolation` on delete means another SG references it — find and remove that rule first.

---

## Engine-specific details

### MySQL

| Item | Value |
|------|-------|
| Engine | `mysql` |
| Admin user | **`mysqladmin`** — `root` is reserved on RDS |
| Parameter group | Required — set binlog params (`binlog_format=ROW`, `binlog_row_image=full`, `binlog_expire_logs_seconds=604800`, `require_secure_transport=0`, `sql_generate_invisible_primary_key=0`) |
| Apply method | `pending-reboot` for most binlog params; `immediate` for `require_secure_transport` |
| Min storage | 20 GB (gp3) |
| EngineVersion | `rds_value` major string (e.g. `"8.0"` or `"8.4"`); RDS selects latest patch |
| Free tier | `db.t3.micro` (12 months) |

### PostgreSQL

| Item | Value |
|------|-------|
| Engine | `postgres` |
| Admin user | **`pgadmin`** — `postgres` is reserved on RDS |
| Parameter group | Required — `rds.logical_replication=1`, `rds.force_ssl=0`, `max_replication_slots=10`, `max_wal_senders=15`, `max_worker_processes=10`, `max_slot_wal_keep_size=10240` |
| Apply method | All `pending-reboot` |
| Post-create | After `poll_rds_available`, check `PendingModifiedValues` — **reboot** if non-empty (params won't apply without reboot) |
| EngineVersion | `rds_value` major string (e.g. `"16"`) |
| Replication grant | `GRANT rds_replication TO lfc_user;` (not `ALTER ROLE WITH REPLICATION` — that requires superuser) |
| Free tier | `db.t3.micro` (12 months) |

### SQL Server

| Item | Value |
|------|-------|
| Engine | **`sqlserver-se`** (Standard Edition) — required for CDC; Express/Web do not support CDC |
| License | `license-included` |
| Admin user | SA-equivalent master user set at create time |
| Parameter group | **None** — SQL Server uses option groups, not parameter groups |
| Option group | Required for CDC: create option group → add `SQLSERVER_BACKUP_RESTORE` or similar options |
| EngineVersion | Resolved at runtime: `describe_db_engine_versions(Engine="sqlserver-se", MajorEngineVersion=ver.rds_value)` → pick latest patch (e.g. `rds_value="15.00"` → `15.00.4345.5`) |
| DBName | **Not set at create** — SQL Server creates `master` database; catalog is created in configure step |
| Timeout | 2400s (`poll_rds_available`) — SQL Server instances take longer to become available |
| CDC platform | `platform="aws"` → `rds_cdc_enable_db` (not `sp_cdc_enable_db` or `gcloudsql_cdc_enable_db`) |
| Free tier | None |

### Oracle

| Item | Value |
|------|-------|
| Engine | **`oracle-se2`** (Standard Edition 2 only) |
| License | `license-included` |
| Admin user | `admin` (master user) |
| SID | Fixed `"ORCL"` |
| EngineVersion | Resolved at runtime: `describe_db_engine_versions(Engine="oracle-se2", MajorEngineVersion=ver.rds_value)` |
| Supported versions | **19c and 21c only** — 23ai has `rds_value=""` and is **not available on RDS** |
| lts/latest alias | ⚠️ Both resolve to `23ai` → `rds_value=""` → setup exits. Always pass `--version 19c` or `--version 21c` explicitly for RDS Oracle |
| Timeout | 2400s |
| CDC platform | `platform="rds"` → Oracle RDS utilities |
| Free tier | None (db.t3.micro available but not free after 12 months) |

---

## RDS vs Aurora

| | **RDS** | **Aurora** |
|---|---|---|
| Architecture | Single instance (or Multi-AZ standby) | DB cluster with shared storage + cluster members |
| API | `create_db_instance` / `DBInstanceIdentifier` | `create_db_cluster` + `create_db_instance` (cluster member) |
| Endpoints | Single instance endpoint | Cluster endpoint (writer) + reader endpoint(s) |
| Engines | MySQL, PostgreSQL, SQL Server, Oracle, MariaDB | MySQL-compatible, PostgreSQL-compatible only |
| Storage | Per-instance gp2/gp3/io1 | Shared, auto-scales 10GB–128TB |
| Failover | Minutes (Multi-AZ: standby promote) | ~30s (cluster member promotion) |
| Pricing | Per instance-hour + storage | Per ACU-hour (Serverless v2) or instance-hour |
| Oracle / SQL Server | ✅ Available on RDS | ❌ Not available on Aurora |

**forgedb targets RDS**, not Aurora, for all engines. Aurora would require a different resource model (`create_db_cluster` + cluster member) and is MySQL/PostgreSQL only — Oracle and SQL Server cannot run on Aurora.

**When you'd choose Aurora over RDS:**
- Need sub-30s failover for MySQL or PostgreSQL
- Want serverless auto-scaling compute (Aurora Serverless v2)
- Running read-heavy workloads that benefit from reader endpoints

---

## Polling patterns

```python
# poll_rds_available — waits for DBInstanceStatus = "available"
# MySQL/PG: timeout 1200s; SQL Server/Oracle: 2400s
def poll_rds_available(rds, identifier, timeout=1200):
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = rds.describe_db_instances(DBInstanceIdentifier=identifier)
        status = r["DBInstances"][0]["DBInstanceStatus"]
        if status == "available":
            return
        if status in ("failed", "incompatible-restore", "incompatible-network"):
            raise RuntimeError(f"RDS instance entered terminal state: {status}")
        time.sleep(30)
    raise TimeoutError(f"RDS instance not available after {timeout}s")
```

After `poll_rds_available` for PostgreSQL — check and reboot if needed:
```python
inst = rds.describe_db_instances(DBInstanceIdentifier=identifier)["DBInstances"][0]
if inst.get("PendingModifiedValues"):
    rds.reboot_db_instance(DBInstanceIdentifier=identifier)
    poll_rds_available(rds, identifier)
```

---

## SSM bastion for VPN-blocked RDS

When RDS is `PubliclyAccessible=True` but your client IP is behind a VPN that blocks
direct TCP to port 3306/5432/1433, use AWS SSM port-forwarding via a bastion EC2 instance
in the same VPC (no open SSH port needed):

```bash
# Start SSM port-forward tunnel in background
aws ssm start-session \
  --target "$BASTION_INSTANCE_ID" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'$RDS_ENDPOINT'"],"portNumber":["3306"],"localPortNumber":["13306"]}' &

# Connect via localhost tunnel
mysql -h 127.0.0.1 -P 13306 -u mysqladmin -p"$PASSWORD" "$CATALOG"
```

Requirements: bastion EC2 instance in the VPC with `ssm:StartSession` IAM permission + SSM agent installed; caller needs `ec2:DescribeInstances` + `ssm:StartSession`.

Stop/start RDS instances via boto3 when not in use:

```python
rds.stop_db_instance(DBInstanceIdentifier=identifier)
rds.start_db_instance(DBInstanceIdentifier=identifier)
# Poll describe_db_instances until DBInstanceStatus == "available" after start
```

---

## Parameter group apply methods

| Param | ApplyMethod | Engine |
|-------|------------|--------|
| binlog params | `pending-reboot` | MySQL |
| `require_secure_transport` | `immediate` | MySQL |
| `rds.logical_replication` | `pending-reboot` | PostgreSQL |
| `rds.force_ssl` | `pending-reboot` | PostgreSQL |
| `max_replication_slots` etc. | `pending-reboot` | PostgreSQL |

`pending-reboot` params are applied at next restart, not immediately. Always reboot after applying PostgreSQL params — forgedb checks `PendingModifiedValues` and reboots when non-empty.
