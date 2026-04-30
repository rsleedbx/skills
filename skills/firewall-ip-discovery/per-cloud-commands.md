# Per-Cloud Firewall Commands

## Azure — MySQL Flexible Server

```bash
_add_mysql_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  if az mysql flexible-server firewall-rule show \
       --resource-group "$RESOURCE_GROUP" --name "$server" \
       --rule-name "$rule_name" --output none &>/dev/null; then
    echo "  rule ${rule_name} already exists — skipping"
    return
  fi
  az mysql flexible-server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --name "$server" \
    --rule-name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
}
_add_mysql_ip_rule "$MYSQL_SERVER" "$MY_IP"                "my-ip-$(date +%Y%m%d)"
_add_mysql_ip_rule "$MYSQL_SERVER" "$SERVERLESS_EGRESS_IP" "databricks-serverless-egress"
```

> MySQL Flexible Server in public-access mode does **not** support VNet rules — IP rules only.

## Azure — PostgreSQL Flexible Server

```bash
_add_pg_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  if az postgres flexible-server firewall-rule show \
       --resource-group "$RESOURCE_GROUP" --name "$server" \
       --rule-name "$rule_name" --output none &>/dev/null; then
    echo "  rule ${rule_name} already exists — skipping"
    return
  fi
  az postgres flexible-server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --name "$server" \
    --rule-name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
}
```

**Idempotency check:** compare `startIpAddress` and `endIpAddress` from `firewall-rule list`, not rule name (rule names vary across runs).

## Azure — SQL Server (logical server)

```bash
_add_sql_ip_rule() {
  local server="$1" ip="$2" rule_name="$3"
  az sql server firewall-rule create \
    --resource-group "$RESOURCE_GROUP" --server "$server" \
    --name "$rule_name" \
    --start-ip-address "$ip" --end-ip-address "$ip" --output none
  # create is idempotent — overwrites if rule name exists
}
```

> Azure SQL: Databricks worker subnets only have `Microsoft.Storage` endpoint, not `Microsoft.Sql`. Without `Microsoft.Sql`, serverless traffic exits as a public IP and VNet rules never match. Use IP rules.

## Azure — NSG (for VMs)

```bash
_add_nsg_ip_rule() {
  local nsg="$1" ip="$2" priority="$3" rule_name="$4"
  az network nsg rule create \
    --resource-group "$RESOURCE_GROUP" \
    --nsg-name "$nsg" \
    --name "$rule_name" \
    --priority "$priority" \
    --direction Inbound \
    --access Allow \
    --protocol Tcp \
    --source-address-prefixes "$ip" \
    --destination-port-ranges "$DB_PORT" \
    --output none
}
# Priority scheme: VirtualNetwork=1000, serverless /24=1010, laptop /32=1020
_add_nsg_ip_rule "$NSG_NAME" "20.96.41.0/24"  1010 "databricks-serverless-egress"
_add_nsg_ip_rule "$NSG_NAME" "${MY_IP}/32"    1020 "my-ip-$(date +%Y%m%d)"
```

> Always skip NSG logic for `access_type=vm` — VMs use NSG rules added separately via `customer-vm-base.sh`.

## AWS — RDS security group

```bash
SG_ID=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_INSTANCE_ID" --region "$REGION" \
  --query 'DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port "$DB_PORT" \
  --cidr "${MY_IP}/32" \
  --region "$REGION"
```

To update (remove stale, add new):
```bash
aws ec2 revoke-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port "$DB_PORT" --cidr "$OLD_IP/32" --region "$REGION"
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port "$DB_PORT" --cidr "${MY_IP}/32" --region "$REGION"
```

## GCP — Cloud SQL authorized networks

```bash
CURRENT=$(gcloud sql instances describe "$INSTANCE" --format=json \
  | jq -r '.settings.ipConfiguration.authorizedNetworks[].value' | tr '\n' ',')
gcloud sql instances patch "$INSTANCE" \
  --authorized-networks "${CURRENT}${MY_IP}/32" \
  --quiet
```

> `gcloud sql instances patch --authorized-networks` **replaces** the entire list — always merge existing + new. `--quiet` suppresses the interactive prompt. Allow up to 300 seconds for propagation.
