# Line-Shortening Patterns

Six techniques that eliminate the most repeated boilerplate. Measured across 9 real scripts
(~1500 lines total), applying all six saves **~102 lines** and shortens dozens more.

---

## 1. `_D` — define `dirname "${BASH_SOURCE[0]}"` once

Every `source` call repeats the same 29-character expression. Define it once at the top of every
script that sources other files.

```bash
# At top of script (after set -u / guard check):
_D="$(dirname "${BASH_SOURCE[0]}")"

# Every source call:
source "${_D}/cmd-wrapper-helpers.sh"
source "${_D}/secrets-databricks-helper.sh"
source "${_D}/mysql-configure.sh"
source "${_D}/uc-connection-helpers.sh"
```

Saves: shortens each source line by ~25 chars. 35 source calls across 9 scripts.

---

## 2. `_RG` args array — collapse repeated `--resource-group`

`--resource-group "$RESOURCE_GROUP"` appears on virtually every `az` CLI call as its own
continuation line. Capture it in an array once and expand with `"${_RG[@]}"`.

```bash
# At top of script (after RESOURCE_GROUP is validated):
_RG=(--resource-group "$RESOURCE_GROUP")

# Every az call saves one continuation line:
CMD az network vnet subnet show "${_RG[@]}" \
  --vnet-name "$VNET_NAME" --name "$MYSQL_PE_SUBNET_NAME"

CMD az network private-dns zone show "${_RG[@]}" \
  --name "$DNS_ZONE_MYSQL"
```

Saves: **~52 lines** across 6 azure scripts (78 occurrences × ~75% on own continuation line,
minus 6 for the `_RG=` definition).

Apply the same pattern for other repeated flag pairs when they appear 5+ times:
- `_SN=(--server-name "$MYSQL_SERVER_NAME")` for mysql param/db calls
- `_VN=(--vnet-name "$VNET_NAME")` when many subnet/PE calls share the same VNet

---

## 3. `declare -A` + loop for parameter check tables

When the same function is called 5+ times with parallel arguments, use an associative array and
loop. Exceptions that need a different operator (`ge`, `le`) stay as explicit calls below the loop.

```bash
# Before (5 separate calls, hard to extend):
_check_param require_secure_transport           off
_check_param binlog_row_image                   full
_check_param binlog_format                      row
_check_param sql_generate_invisible_primary_key off
_check_param binlog_expire_logs_seconds         604800 ge

# After (table-driven, trivial to extend):
declare -A _EQ_PARAMS=(
  [require_secure_transport]=off
  [binlog_row_image]=full
  [binlog_format]=row
  [sql_generate_invisible_primary_key]=off
)
for _p in "${!_EQ_PARAMS[@]}"; do _check_param "$_p" "${_EQ_PARAMS[$_p]}"; done
_check_param binlog_expire_logs_seconds 604800 ge   # kept separate: needs "ge" operator
```

Saves: ~2 lines per table. Scales cleanly when new parameters are added.

---

## 4. Heredoc for multi-line summary echo blocks

End-of-script summary blocks with 8+ consecutive `echo` lines are easier to edit as a heredoc.
No `"` quoting per line, no line-join issues.

```bash
# Before (12 separate echo lines):
echo ""
echo "==> SQL_DNS: $SQL_DNS"
echo "    Port:    3306"
echo "    Public:  firewall allows ${SERVERLESS_EGRESS_IP_START}-${SERVERLESS_EGRESS_IP_END} (/24)"
echo "    Private: endpoint '$MYSQL_PE_NAME' in subnet '$MYSQL_PE_SUBNET_NAME'"
echo "    LFC users:"
echo "      ${LFC_USER_PUBLIC}  → public IP"
echo "      ${LFC_USER_PNG}     → PNG FQDN"

# After:
cat <<EOF

==> SQL_DNS: ${SQL_DNS}
    Port:    3306
    Public:  firewall allows ${SERVERLESS_EGRESS_IP_START}-${SERVERLESS_EGRESS_IP_END} (/24)
    Private: endpoint '${MYSQL_PE_NAME}' in subnet '${MYSQL_PE_SUBNET_NAME}'
    LFC users:
      ${LFC_USER_PUBLIC}  → public IP
      ${LFC_USER_PNG}     → PNG FQDN
EOF
```

**Always use `${VAR}` braces** inside heredocs — guards against non-ASCII byte bleed into adjacent
variable names (same rule as in echo/string contexts; see main SKILL.md).

Line count impact is minimal (gains `cat <<EOF` and `EOF` lines), but the block is far easier
to extend and the quoting is simpler.

---

## 5. `jq -n env` / `$ENV` — replace `--arg` chains

`jq -n` exposes every **exported** shell variable as `$ENV.<NAME>`. For functions that build JSON
from many env vars, drop the `--arg` block entirely.

```bash
# Before (23 --arg lines in save_db_secret):
new_json=$(jq -n \
  --arg host_fqdn         "${host_fqdn:?}" \
  --arg port              "${port:?}" \
  --arg catalog           "${catalog:-}" \
  --arg user              "${user:?}" \
  --arg password          "${password:?}" \
  --arg db_type           "${db_type:?}" \
  --arg schema            "${schema:-}" \
  --arg replication_mode  "${replication_mode:-}" \
  --arg access_type       "${access_type:-}" \
  --arg cloud_provider    "${cloud_provider:-azure}" \
  --arg cloud_location    "${cloud_location:-}" \
  --arg dba_user          "${dba_user:-${user}}" \
  --arg dba_password      "${dba_password:-${password}}" \
  --arg dba_catalog       "${dba_catalog:-}" \
  --arg server_name       "${server_name:-}" \
  --arg vm_name           "${vm_name:-}" \
  --arg vm_admin_user     "${vm_admin_user:-}" \
  --arg vm_admin_password "${vm_admin_password:-}" \
  --arg vm_fqdn_public    "${vm_fqdn_public:-}" \
  --arg routing_user_public      "${routing_user_public:-}" \
  --arg routing_password_public  "${routing_password_public:-}" \
  --arg routing_user_png         "${routing_user_png:-}" \
  --arg routing_password_png     "${routing_password_png:-}" \
  '{ version: "v2", host_fqdn: $host_fqdn, ... }')

# After (0 --arg lines — variables must be exported):
new_json=$(jq -n '{
  version:          "v2",
  host_fqdn:        $ENV.host_fqdn,
  port:             ($ENV.port | tonumber),
  catalog:          $ENV.catalog,
  user:             $ENV.user,
  password:         $ENV.password,
  db_type:          $ENV.db_type,
  schema:           $ENV.schema,
  replication_mode: $ENV.replication_mode,
  access_type:      $ENV.access_type,
  cloud:            {provider: $ENV.cloud_provider, location: $ENV.cloud_location},
  dba:              {user: $ENV.dba_user, password: $ENV.dba_password, catalog: $ENV.dba_catalog}
}
+ if $ENV.server_name != "" then {server_name: $ENV.server_name} else {} end
+ if $ENV.vm_name != "" then {vm_name: $ENV.vm_name} else {} end
...')
```

**Trade-offs:**
- `$ENV.var` returns `""` when the variable is unset or unexported — no `:?` error from jq
- Move required-field validation to a bash function (e.g. `_validate_db_secret`) that runs before `jq`
- Variables **must be exported** (`export foo=bar` or `export foo` after assignment) — `$ENV` does not see unexported locals

Saves: **~23 lines** in `save_db_secret` alone; applies anywhere `jq -n` builds objects from env vars.

---

## 6. `${!var}` indirect expansion — collapse parallel guard blocks

When 4+ variables need the same guard/validation, use bash indirect expansion `${!var}` in a loop.
`${!_v}` reads the variable whose name is stored in `_v`.

```bash
# Before (6 separate guard lines — same error message stem):
: "${LFC_USER:?mysql-configure did not produce LFC_USER — re-run setup}" || kill -INT $$
: "${LFC_PASSWORD:?mysql-configure did not produce LFC_PASSWORD — re-run setup}" || kill -INT $$
: "${LFC_USER_PUBLIC:?mysql-configure did not produce LFC_USER_PUBLIC — re-run setup}" || kill -INT $$
: "${LFC_USER_PUBLIC_PASSWORD:?mysql-configure did not produce LFC_USER_PUBLIC_PASSWORD — re-run setup}" || kill -INT $$
: "${LFC_USER_PNG:?mysql-configure did not produce LFC_USER_PNG — re-run setup}" || kill -INT $$
: "${LFC_USER_PNG_PASSWORD:?mysql-configure did not produce LFC_USER_PNG_PASSWORD — re-run setup}" || kill -INT $$

# After (3 lines):
for _v in LFC_USER LFC_PASSWORD LFC_USER_PUBLIC LFC_USER_PUBLIC_PASSWORD \
          LFC_USER_PNG LFC_USER_PNG_PASSWORD; do
  [[ -n "${!_v:-}" ]] || { echo "error: mysql-configure did not set ${_v} — re-run setup" >&2; kill -INT $$; }
done
```

Apply whenever 4+ consecutive lines share the same error message stem. Saves ~3 lines per
6-variable guard block; ~18 lines across 6 scripts.

---

## Summary — measured savings (azure_png_demo, 9 scripts, ~1500 lines)

| Technique | Occurrences | Net lines saved |
|---|---|---|
| `_D` dirname variable | 35 source calls / 9 scripts | lines shortened, not removed |
| `_RG` args array | 78 `az` calls / 6 scripts | **~52** |
| `declare -A` param table | 5 params / 3 scripts | ~9 |
| Heredoc summary blocks | 72 echo lines / 6 scripts | readability only |
| `jq -n env` / `$ENV` | 23 `--arg` lines / 1 function | **~23** |
| `${!var}` indirect loop | 6 guards × 6 scripts | **~18** |
| **Total** | | **~102 lines** |
