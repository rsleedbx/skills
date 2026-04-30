# Cloud Networking and Replication Flags

## AWS networking pattern for RDS

```hcl
# Use default VPC for simplicity (public subnets)
data "aws_vpc" "default" { default = true }

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
  filter {
    name   = "map-public-ip-on-launch"
    values = ["true"]
  }
}

resource "aws_db_subnet_group" "main" {
  subnet_ids = data.aws_subnets.public.ids
}

resource "aws_db_instance" "oracle" {
  publicly_accessible  = true
  db_subnet_group_name = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

Security group: merge caller IP `/32` with `additional_cidrs` from tfvars.

---

## AWS RDS PostgreSQL: replication parameter group

```hcl
resource "aws_db_parameter_group" "postgres_lfc" {
  family = "postgres16"

  parameter {
    name         = "rds.logical_replication"
    value        = "1"
    apply_method = "pending-reboot"
  }
  parameter {
    name  = "max_replication_slots"
    value = "10"
  }
  parameter {
    name  = "max_wal_senders"
    value = "15"    # Both aws-postgres and aws-aurora-postgres use 15, not 10
  }
  parameter {
    name  = "rds.force_ssl"
    value = "0"    # LFC does not support SSL
  }
}
```

---

## GCP Cloud SQL PostgreSQL: replication flags

```hcl
resource "google_sql_database_instance" "postgres" {
  settings {
    edition = "ENTERPRISE"    # required for logical decoding / replication

    database_flags {
      name  = "cloudsql.logical_decoding"
      value = var.enable_replication ? "on" : "off"
    }
    backup_configuration {
      enabled                        = var.enable_replication
      point_in_time_recovery_enabled = var.enable_replication
    }
    ip_configuration {
      ssl_mode = "ALLOW_UNENCRYPTED_AND_ENCRYPTED"  # dev/test
      dynamic "authorized_networks" {
        for_each = var.authorized_networks
        content {
          value = authorized_networks.value
        }
      }
    }
  }
}
```

`backup_configuration.enabled` is required for logical replication on GCP Cloud SQL.
