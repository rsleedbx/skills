---
name: sh-to-pulumi
description: >-
  Migrate bash setup scripts (az/aws/gcloud/databricks CLIs) to Python in a
  pulumiscripts/ directory. Three approach options: subprocess (call same CLIs),
  Azure SDK (azure-mgmt-*), or Pulumi Automation API. Default is subprocess for
  mixed workflows (Azure + database + Databricks). Covers conversion patterns
  (CMD wrapper → subprocess.run, source → import, env-var passing → function args,
  jq → json.loads, temp files → variables), observability conventions (==> prefix),
  idempotency patterns (check-then-create), and success criteria (same functionality,
  fewer lines, same observability). Use when porting .sh scripts to Python, choosing
  between subprocess/SDK/Pulumi, or reviewing pulumiscripts/ files.
---

# sh-to-pulumi Migration Skill

## Approach decision

Before creating any file, pick one approach:

| Option | When to use | State model |
|--------|-------------|-------------|
| **A — subprocess** | Mixed workflows (Azure + DB + Databricks); existing Databricks-secrets state | Databricks secrets (unchanged) |
| **B — Pulumi Automation API** | Pure Azure infra that benefits from drift detection / plan-apply | Pulumi stack + Databricks secrets |
| **C — Azure SDK + Databricks SDK** | No CLI dependency desired; maximum API coverage | Databricks secrets (unchanged) |

**Default**: Option A (subprocess) unless the script is pure Azure infra with no DB or Databricks calls.

Pulumi's state engine only covers the resources its provider understands. For scripts that mix Azure provisioning + MySQL admin + Databricks API, Pulumi covers one third of the work and adds a second source of truth alongside Databricks secrets. Use subprocess instead.

---

## Bash → Python conversion patterns

### CMD wrapper → `subprocess.run`

```bash
# Bash (CMD wrapper, temp file, jq)
CMD_EXIT_ON_ERROR=PRINT_EXIT CMD az mysql flexible-server show \
  --resource-group "$RG" --name "$SERVER" --query "id" -o tsv
SERVER_ID=$(cat /tmp/az_stdout.$$)
```

```python
# Python
import subprocess, json

def az(*args: str, check: bool = True) -> str:
    """Run az CLI, return stdout. Raises on non-zero exit if check=True."""
    cmd = ["az", *args]
    print(f"  + {' '.join(cmd)}")
    r = subprocess.run(cmd, capture_output=True, text=True)
    if r.returncode != 0 and check:
        print(r.stderr, end="")
        raise SystemExit(f"az failed (RC={r.returncode})")
    return r.stdout.strip()

server_id = az("mysql", "flexible-server", "show",
               "--resource-group", rg, "--name", server,
               "--query", "id", "-o", "tsv")
```

### Idempotency (check-then-create)

```bash
# Bash
if CMD az network vnet subnet show ... ; then
  echo "==> already exists — skipping"
else
  CMD_EXIT_ON_ERROR=PRINT_EXIT CMD az network vnet subnet create ...
fi
```

```python
# Python
def ensure_subnet(rg: str, vnet: str, name: str, cidr: str) -> None:
    r = subprocess.run(["az", "network", "vnet", "subnet", "show",
                        "--resource-group", rg, "--vnet-name", vnet, "--name", name],
                       capture_output=True)
    if r.returncode == 0:
        print(f"==> Subnet '{name}' already exists — skipping")
        return
    az("network", "vnet", "subnet", "create",
       "--resource-group", rg, "--vnet-name", vnet,
       "--name", name, "--address-prefixes", cidr,
       "--private-endpoint-network-policies", "Disabled", "--output", "none")
    print(f"==> Created subnet '{name}' ({cidr})")
```

### JSON parsing (jq → json.loads)

```bash
# Bash
_cur=$(echo "$_PARM_LIST" | jq -r --arg n "$name" \
  '.[] | select(.name==$n) | .currentValue // empty')
```

```python
# Python
params = json.loads(az("mysql", "flexible-server", "parameter", "list",
                       "--resource-group", rg, "--server-name", server))
cur = next((p["currentValue"] for p in params if p["name"] == name), None)
```

### source → import

Each sourced bash script becomes a Python module with functions:

| Bash | Python |
|------|--------|
| `source cmd-wrapper-helpers.sh` | `from helpers.az_runner import az, az_json` |
| `source databricks-secrets-helper.sh` | `from helpers.secrets import load_db_secret, save_db_secret` |
| `source mysql-configure.sh` | `from helpers.mysql_configure import mysql_configure` |
| `source azure-firewall-helpers.sh` | `from helpers.firewall import add_mysql_ip_rule` |
| `source uc-connection-helpers.sh` | `from helpers.uc_connections import uc_connection_upsert` |

### Env-var passing → function parameters / dataclass

```bash
# Bash: scripts communicate via exported env vars
export SQL_DNS="..."
source databricks-3-setup-uc-lakeflow.sh
# databricks-3 reads $SQL_DNS implicitly
```

```python
# Python: explicit parameters or a shared Config dataclass
@dataclass
class MysqlConfig:
    server_name: str
    sql_dns: str
    admin_user: str
    admin_password: str
    db_name: str
    resource_group: str
    vnet_name: str
    location: str
```

### Secret masking

```bash
CMD_MASK_SECRETS=("$MYSQL_ADMIN_PASSWORD")
CMD az mysql flexible-server create ... --admin-password "$MYSQL_ADMIN_PASSWORD"
```

```python
def az_masked(secret: str, *args: str) -> str:
    display = " ".join(a if a != secret else "***" for a in args)
    print(f"  + az {display}")
    r = subprocess.run(["az", *args], capture_output=True, text=True, check=True)
    return r.stdout.strip()
```

### Observability — keep the same `==>` style

```python
def section(title: str) -> None:
    print(f"\n{'=' * 40}")
    print(f"==> {title}")
    print('=' * 40)
```

---

## Directory structure

```
pulumiscripts/
├── requirements.txt          # subprocess approach: minimal deps
├── customer_3_setup_azure_mysql.py    # ported script
├── customer_3_setup_azure_postgres.py
├── customer_3_setup_azure_sqlserver.py
└── helpers/
    ├── __init__.py
    ├── az_runner.py          # az() wrapper (replaces CMD wrapper)
    ├── secrets.py            # load_db_secret / save_db_secret
    ├── mysql_configure.py    # replaces mysql-configure.sh
    ├── firewall.py           # replaces azure-firewall-helpers.sh
    └── uc_connections.py     # replaces uc-connection-helpers.sh
```

---

## Success criteria checklist

For every ported script, verify before closing:

- [ ] All 15 sections produce identical Azure / Databricks state
- [ ] Line count < original bash line count
- [ ] Every `==>` section header preserved
- [ ] Summary block at end (SQL_DNS, port, users, connections) identical
- [ ] Idempotent: running twice produces no errors and no duplicate resources
- [ ] Secrets loaded on re-run (no password rotation on second run)
- [ ] Dry-run mode available (`--dry-run` prints actions, makes no API calls)

---

## Plan docs location

Each migration project has a plan in `docs/<project-name>/`:
```
docs/sh-to-pulumi/
├── plan.md        # goal, key facts
├── tracker.md     # current status, design decision, section map
├── chronology.md  # one row per run
└── chronology-N.md
```
