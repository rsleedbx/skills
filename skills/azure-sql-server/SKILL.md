---
name: azure-sql-server
description: Azure SQL Server setup for LakeFlow Connect — idempotent bash scripts, serverless auto-pause handling, configure-before-PE section ordering. Use when writing or editing setup scripts for Azure SQL Server, managing firewall rules, private endpoints, Unity Catalog connections, or handling Azure SQL Serverless auto-pause. See database-sqlserver for CT/CDC setup, login/user architecture, sqlcmd conventions, trustServerCertificate, and identifier quoting.
---

# Azure SQL Server — LFC Setup

## Serverless database auto-pause

Azure SQL Serverless databases auto-pause after idle. The first connection attempt returns "Database is not currently available" while Azure resumes it.

**Pattern (from lakeflow_connect `start_db`)**: check `az sql db show --query "state"` first. If `Online`, use a short timeout. If `Paused`/`Resuming`, use a long login timeout — `sqlcmd -l 180` will wait through the resume in a single attempt.

```bash
_sc_wait_for_db() {
  local db="$1"
  local _server="${_SC_HOST%%.*}"   # FQDN → server name
  local _state _timeout
  _state=$(az sql db show \
    --resource-group "$RESOURCE_GROUP" \
    --server "$_server" --name "$db" \
    --query "state" -o tsv 2>/dev/null || echo "Unknown")
  if [[ "$_state" == "Online" ]]; then _timeout=10; else _timeout=180; fi
  sqlcmd -S "${_SC_HOST},${_SC_PORT}" -d "$db" \
    -U "$_SC_USER" -P "$_SC_PASS" \
    -C -l "$_timeout" -Q "SELECT 1" &>/dev/null || {
    echo "error: database '$db' unavailable (state: '${_state}')" >&2
    kill -INT $$
  }
}
_sc_wait_for_db "$catalog"
```

Call `_sc_wait_for_db` before any catalog-level SQL — after DBA master connection is verified.

## Section ordering: configure before private endpoint

`sqlserver-configure.sh` connects via the **public FQDN** and does not depend on the private endpoint. Always save secret + run configure **before** creating the PE.

```
1   PE subnet
2   Create server + database
2b  Password probe
3   Save secret (admin creds)       ← before PE
3b  sqlserver-configure (LFC user)  ← before PE
4   Create private endpoint
5   Wire up private DNS
6   UC connections
```

## Firewall rules — IP-based idempotency

Check existing rules by IP range value (not rule name) to avoid duplicates:

```bash
_add_sql_ip_rule "$SQL_SERVER_NAME" "$_MY_IP" "local-machine-$(date +%Y%m%d)"
_add_sql_ip_rule "$SQL_SERVER_NAME" "$SERVERLESS_EGRESS_IP_START" \
  "databricks-serverless-egress-slash24" "$SERVERLESS_EGRESS_IP_END"
```

Always wrap IP variables in `${ip}` braces in echo strings — adjacent non-ASCII bytes (en-dash etc.) can bleed into the variable name in bash's locale-aware parser.
