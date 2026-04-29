---
name: local-database-containers
description: Running databases locally on macOS via Podman, Lima, and Colima containers — the two-VM Apple Silicon architecture (VZ+Rosetta vs QEMU for DB2), three backend options (Podman Machine / Lima+nerdctl / Colima), container images and required flags for each engine (PostgreSQL, MySQL, CockroachDB, SQL Server, Oracle Free, DB2), readiness probe patterns, named volume conventions, find_free_port pattern, DB2-specific requirements (--ipc=host --privileged, first-boot timing, db2profile patch). Use when starting local database containers for development or testing, setting up statschema benchmarks, debugging DB2 IPC failures, choosing between Podman/Lima/Colima, or understanding why DB2 requires QEMU on Apple Silicon.
---

# Local Database Containers (Podman / Lima / Colima)

## Apple Silicon two-VM reality

On ARM64 Macs, you cannot run all databases in one VM. DB2 probes for x86-64-v2 CPU features (SSE4.2, POPCNT) at startup. **Rosetta 2 does not fully satisfy x86-64-v2** — DB2 fails under VZ/Rosetta but works reliably under QEMU emulation.

**You always need two VMs:**

| VM | vmType | Engines | Speed |
|----|--------|---------|-------|
| VM 1 (fast) | `vz` + Rosetta 2 | PostgreSQL, MySQL, CockroachDB, SQL Server, Oracle | ~1.5–2× native |
| VM 2 (correct) | `qemu` (x86_64) | DB2 only | ~5–10× slower but correct |

`vmType` is baked in at creation time and cannot be changed — delete and recreate if you need to change it.

## Three backend options

| Backend | VM 1 CLI | VM 1 manager | DB2 command |
|---------|----------|-------------|------------|
| **Podman Machine** (`_common.sh`) | `podman run` | `podman machine` (transparent) | `limactl shell db2 -- sudo podman ...` |
| **Lima + nerdctl** (`_common_lima.sh`) | `limactl shell statschema -- sudo nerdctl ...` | `limactl` | `limactl shell db2 -- sudo podman ...` |
| **Colima** (`_common_colima.sh`) | `nerdctl run` | `colima` (wraps Lima) | `limactl shell db2 -- sudo podman ...` |

All three setups use exactly two VMs for full coverage. The difference is only how VM 1 is managed. DB2 always uses the Lima QEMU `db2` VM.

### Podman Machine (simplest)

```bash
podman machine init            # creates applehv (VZ + Rosetta) VM
podman machine start
podman run ...                 # proxied transparently to the VM
```

**Limitation:** Podman 5.x removed `--vmtype`/`--arch` flags — you cannot create a QEMU Podman machine. DB2 must use Lima directly.

### Lima + nerdctl

```bash
# Create the VZ VM with Rosetta (arm64)
limactl create --name statschema \
  --set '.vmType = "vz"' \
  --set '.vmOpts.vz.rosetta.enabled = true' \
  --set '.vmOpts.vz.rosetta.binfmt = true' \
  template:default
limactl start statschema
limactl shell statschema -- sudo systemctl enable --now containerd

# Wrapper function used in scripts
_lnerdctl() { limactl shell "${LIMA_VM:-statschema}" -- sudo /usr/local/bin/nerdctl "$@"; }
```

### Colima

```bash
colima start --vmtype vz --arch aarch64 --vz-rosetta --runtime containerd
# Internally creates a Lima VZ VM

# Target a named Colima instance
nerdctl --address ~/.colima/arm/containerd.sock run ...
```

---

## Container images and required flags

### PostgreSQL

```bash
podman run -d --name "pg${VERSION}" \
  -v "pg${VERSION}-data:/var/lib/postgresql" \
  -p "${PORT}:5432" \
  -e POSTGRES_PASSWORD="$dba_pass" \
  "postgres:${VERSION}"
```

- Mount `/var/lib/postgresql` (parent dir) — allows `pg_upgrade --link` across major versions without crossing mount boundaries
- Readiness: `podman exec $container pg_isready -U postgres`

### MySQL

```bash
podman run -d --name "mysql${VERSION}" \
  -v "mysql${VERSION}-data:/var/lib/mysql" \
  -p "${PORT}:3306" \
  -e MYSQL_ROOT_PASSWORD="$dba_pass" \
  "mysql:${VERSION}"
```

- Default base port: `3384` (avoids clash with system MySQL on 3306)
- Readiness: `podman exec $container mysqladmin ping -u root -p"${dba_pass}" --silent`

### CockroachDB

```bash
podman run -d --name "crdb${VERSION}" \
  -v "crdb${VERSION}-data:/cockroach/cockroach-data" \
  -p "${PORT}:26257" \
  "cockroachdb/cockroach:${VERSION}" \
  -- start-single-node --insecure
```

- Single-node insecure mode — no passwords for the app user in insecure mode
- Readiness: `podman exec $container /cockroach/cockroach sql --insecure -e "SELECT 1"`
- Grant: `CREATE USER IF NOT EXISTS user; GRANT ALL ON DATABASE catalog TO user; ALTER USER user CREATEDB;`

### SQL Server

```bash
podman run -d --name "sqlserver${VERSION}" \
  -v "sqlserver${VERSION}-data:/var/opt/mssql" \
  -p "${PORT}:1433" \
  --platform linux/amd64 \
  -e ACCEPT_EULA=Y \
  -e MSSQL_SA_PASSWORD="$dba_pass" \
  "mcr.microsoft.com/mssql/server:${VERSION}"
```

- `--platform linux/amd64` required — no native ARM64 image; Rosetta 2 translates
- Default version: `2022-latest` (SQL Server 2025 known not to work with macOS Rosetta at time of testing)
- Default base port: `1433`
- Readiness: `podman exec $container /opt/mssql-tools18/bin/sqlcmd -C -S localhost -U sa -P "$dba_pass" -Q "SELECT 1" -b`
- `-C` = trust server certificate (self-signed inside container)

### Oracle Free

```bash
# ARM64 Mac: force native image (faster, avoids Rosetta overhead)
PLATFORM_FLAG=()
[[ "$(uname -m)" == "arm64" ]] && PLATFORM_FLAG=(--platform linux/arm64)

podman run -d --name "oracle${VERSION}" \
  -v "oracle${VERSION}-data:/opt/oracle/oradata" \
  -p "${PORT}:1521" \
  --shm-size=1g \
  "${PLATFORM_FLAG[@]}" \
  -e ORACLE_PWD="$pass" \
  -e APP_USER="$app_user" \
  -e APP_USER_PASSWORD="$app_pass" \
  "container-registry.oracle.com/database/free:${VERSION}"
```

- Default PDB: `FREEPDB1` (env var chain: `DB_ORA_CATALOG` → `DB_ORA_DATABASE` → `ORACLE_PDB` → `FREEPDB1`)
- `--shm-size=1g` required for Oracle shared memory
- Oracle Free has a native ARM64 image — Podman defaults to pulling x86_64 without `--platform linux/arm64`
- Readiness probe must use TCP + `-L` flag: `sqlplus -L -S "system/${pass}@//localhost:1521/${PDB}"` — without `-L`, sqlplus reads EXIT from stdin and exits 0 even when the PDB is not yet registered (ORA-12514 false positive)
- Init takes ~60s; use `wait_db` with 180s timeout

### DB2 (QEMU Lima VM — always)

DB2 always runs inside the Lima QEMU `db2` VM:

```bash
# Run DB2 container inside the Lima db2 VM
limactl shell db2 -- sudo podman run -d --name db2ce \
  -v db2_data:/database \
  -p 50000:50000 \
  --platform linux/amd64 \
  --privileged \
  --ipc=host \
  -e DB2INST1_PASSWORD="$dba_pass" \
  -e DBNAME="${catalog}" \
  -e LICENSE=accept \
  "icr.io/db2_community/db2:${VERSION}"
```

**Critical flags:**
- `--ipc=host` — required: DB2 uses System V IPC shared memory (`db2ftok`); isolated IPC namespace returns `ENOTSUP` under Rosetta 2
- `--privileged` — required by IBM for DB2 container
- `--platform linux/amd64` — no native ARM64 image

**First-boot timing:** DB2 runs a ~300s initialization (Task #3: `db2iupdt`). TCP port 50000 opens before DB2 is ready — `wait_port` alone gives false positives. Always wait for the init-done sentinel file before running provisioning:

```bash
# Wait for IBM's setup_db2_instance.sh to complete (writes this file as last step)
podman exec db2ce test -f /database/config/.shared-data/setup_complete

# Then probe with real connection
su - db2inst1 -c ". ~/sqllib/db2profile && db2 connect to ${catalog^^}"
```

**Rosetta 2 db2profile patch** — during init, some `su - db2inst1` invocations fail with "db2: command not found" due to transient IPC state. Patch every script in `/var/db2_setup/` to prepend `. ~/sqllib/db2profile &&`:

```bash
limactl shell db2 -- sudo podman exec db2ce \
  bash -c "sed -i 's|su - db2inst1 -c|su - db2inst1 -c \". ~/sqllib/db2profile \&\& |g' /var/db2_setup/*.sh"
```

**DB2 database name limit:** 8 characters maximum. Use `statsch` not `statschema`.

---

## Reusable `_start_container` pattern

The statschema `_start_container` helper encapsulates the idempotent create-or-reuse logic:

```bash
_start_container() {
    local container="$1" image="$2" guest_port="$3" default_port="$4"
    local port_var="$5" data_mount="$6"
    shift 6
    # ... split remaining args on '--' into run_opts and cmd_args

    local _state port
    _state=$(podman inspect --format '{{.State.Status}}' "$container" 2>/dev/null)

    if [[ -n "$_state" ]]; then
        # Already exists — read its provisioned port
        port=$(podman inspect --format \
            "{{(index (index .HostConfig.PortBindings \"${guest_port}/tcp\") 0).HostPort}}" \
            "$container" 2>/dev/null)
        [[ "$_state" != "running" ]] && podman start "$container"
    else
        # Create new — find a free port first
        port=$(find_free_port "${!port_var:-$default_port}")
        podman run -d --name "$container" \
            -v "${container}-data:${data_mount}" \
            "${run_opts[@]}" \
            -p "${port}:${guest_port}" "$image" "${cmd_args[@]}"
    fi
    printf -v "$port_var" '%s' "$port"
    export "$port_var"
}
```

## `find_free_port` — avoids port conflicts with stopped containers

TCP port checking alone (`nc -z`) misses ports reserved by stopped Podman containers (they don't bind but Podman owns the reservation):

```bash
find_free_port() {
    local port="$1"
    local _podman_ports
    _podman_ports=$(podman inspect \
        --format '{{range $p, $b := .HostConfig.PortBindings}}{{(index $b 0).HostPort}} {{end}}' \
        $(podman ps -aq 2>/dev/null) 2>/dev/null)
    while true; do
        nc -z 127.0.0.1 "$port" 2>/dev/null && { port=$(( port + 1 )); continue; }
        [[ " $_podman_ports " == *" $port "* ]] && { port=$(( port + 1 )); continue; }
        break
    done
    echo "$port"
}
```

## `wait_db` — real connection probe (not just TCP)

TCP open is not sufficient — MySQL, DB2, and SQL Server all accept TCP before they're ready. Always probe with a real engine connection:

```bash
wait_db() {
    local label="$1" max_s="${2:-120}" container="${3:-}"
    shift 3
    # $@ = the probe command
    local elapsed=0
    until "$@" &>/dev/null; do
        (( elapsed >= max_s )) && { podman logs "$container" | tail -20; die "$label timeout"; }
        sleep 10; elapsed=$(( elapsed + 10 ))
        # Show snapshot (not -f stream — concurrent podman logs -f + exec hangs Podman)
        [[ -n "$container" ]] && podman logs --tail 3 "$container" 2>&1
    done
}
```

> **Never use `podman logs -f` concurrently with `podman exec`** — this combination causes Podman to hang. Use `podman logs --tail N` for snapshots.

## Default port assignments

| Engine | Default base port | Note |
|--------|------------------|------|
| PostgreSQL | `5432` | |
| MySQL | `3384` | avoids clash with system MySQL on 3306 |
| CockroachDB | `26257` | |
| SQL Server | `1433` | |
| Oracle | `1521` | |
| DB2 | `50000` | |

## Volume naming convention

Named volumes use `${container_name}-data`:
- `pg18-data` → `/var/lib/postgresql`
- `mysql8-data` → `/var/lib/mysql`
- `sqlserver2022-data` → `/var/opt/mssql`
- `oraclelatest-data` → `/opt/oracle/oradata`
- `db2_data` (inside Lima VM) → `/database`

Container names are derived from image version: `${engine}${version}` with dots/dashes stripped, e.g. `postgres:18` → `pg18`, `mssql/server:2022-latest` → `sqlserver2022latest`.

## Default credentials and env var chain

The common pattern uses a fallback chain: `DB_<ENGINE>_DBA_PASSWORD` → `DB_DBA_PASSWORD` → `Statsch3ma!`

```bash
dba_pass="${DB_PG_DBA_PASSWORD:-${DB_DBA_PASSWORD:-Statsch3ma!}}"
user="${DB_PG_USERNAME:-${DB_USERNAME:-statschema}}"
app_pass="${DB_PG_PASSWORD:-${DB_PASSWORD:-Statsch3ma!}}"
catalog="${DB_PG_CATALOG:-${DB_CATALOG:-statschema}}"
```

Set in `.env` to override:
```bash
DB_DBA_PASSWORD=MyStrongPass!
DB_USERNAME=myapp
DB_PASSWORD=MyAppPass!
DB_CATALOG=mydb
```

## Provisioning: basic user + schema (not LFC)

These containers provision a basic app user and schema for benchmarks — **not** full LFC configuration. There is no `wal_level=logical`, no binlog CDC params, no CT/CDC enablement. For LFC-specific setup see `databricks-lakeflow-connect` + database engine skills.

## QEMU performance on Apple Silicon

Running multiple QEMU VMs (DB2, SQL Server, Oracle in separate QEMU VMs) simultaneously saturates the host CPU. For benchmarks, use two-wave scheduling:

- **Wave 1** — Podman engines (PostgreSQL, MySQL, CockroachDB): run first, ~5 min
- **Wave 2** — QEMU engines (SQL Server, Oracle, DB2): run after Wave 1 finishes, uncontested, ~33 min

Running all QEMU engines simultaneously causes ~1.7× slowdown each due to shared CPU. See the statschema repo's `setup-lima-databases` skill for memory budgets and OOM caps per engine.
