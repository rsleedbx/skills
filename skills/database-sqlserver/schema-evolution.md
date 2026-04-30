# Schema Evolution DDL Objects and Utility Script

## Schema evolution DDL objects

LFC requires DDL support objects downloaded from the Databricks docs. Choose the script based on `CDC_CT_MODE`:

| CDC_CT_MODE | Script |
|-------------|--------|
| `BOTH` / `CDC` | `ddl_support_objects.sql` |
| `CT` | `ddl_support_objects_ct_only.sql` |

```bash
ddl_script_url="https://docs.databricks.com/aws/en/assets/files/ddl_support_objects-06ebad393ea6bc7d853d5504dc6542de.sql"

case "${CDC_CT_MODE}" in
  "BOTH"|"CDC") ddl_script=./ddl_support_objects.sql ;;
  "CT")         ddl_script=./ddl_support_objects_ct_only.sql ;;
  *)            echo "CDC_CT_MODE must be BOTH, CDC, or CT for schema evolution"; return 1 ;;
esac

# Use local file if present, else download
if [[ -f "$ddl_script" ]]; then
    cat "$ddl_script"
else
    wget -qO- "$ddl_script_url"
fi | sed -e "s/SET \@replicationUser = '';/SET \@replicationUser = '${LFC_USER}';/" \
        -e "s/\@mode = '.*';/\@mode = '$CDC_CT_MODE';/" \
  | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```

The `sed` substitutions inject the LFC user and CDC_CT_MODE into the downloaded script before execution. Run as SA (DBA), not lfc_user.

---

## Utility script

A companion script for table discovery and CT/CDC status verification:

```bash
utility_script_url="https://docs.databricks.com/aws/en/assets/files/utility_script-a4544ba646de3f6f3fd03eb3dcba563e.sql"

if [[ -f "$utility_script_path" ]]; then
    cat "$utility_script_path"
else
    wget -qO- "$utility_script_url"
fi | sqlcmd -S "$HOST,$PORT" -d "$CATALOG" -U sa -P "$SA_PASS" -C
```
