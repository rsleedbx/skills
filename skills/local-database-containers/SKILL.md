---
name: local-database-containers
description: Running databases locally on macOS via Podman, Lima, and Colima containers — the two-VM Apple Silicon architecture (VZ+Rosetta vs QEMU for DB2/Informix/Oracle), three backend options (Podman Machine / Lima+nerdctl / Colima), container images and required flags for each engine (PostgreSQL, MySQL, MariaDB, CockroachDB, SQL Server, Oracle Free, DB2, Informix, MongoDB), readiness probe patterns, named volume conventions, find_free_port pattern, DB2-specific requirements (--privileged, first-boot timing, catalog readiness), Informix requirements (OS/PAM auth, dbaccess CLI, WITH LOG for CDC), Oracle ARCHIVELOG timing, Lima socket stability (Lima 2.x SSH forwarder is reliable — use podman --connection, not limactl shell), registries.conf unqualified-search setup. Use when starting local database containers for development or testing, setting up statschema benchmarks, debugging DB2 IPC failures, choosing between Podman/Lima/Colima, understanding why DB2/Informix requires QEMU on Apple Silicon, or configuring the Podman socket connection for use with podman-compose and Testcontainers.
---

# Local Database Containers (Podman / Lima / Colima)

## Apple Silicon two-VM reality

On ARM64 Macs, you cannot run all databases in one VM. DB2 probes for x86-64-v2 CPU features (SSE4.2, POPCNT) at startup. **Rosetta 2 does not fully satisfy x86-64-v2** — DB2 fails under VZ/Rosetta but works reliably under QEMU emulation.

**You always need two VMs:**

| VM | vmType | Engines | Speed |
|----|--------|---------|-------|
| VM 1 (fast) — `forgedb-vz` | `vz` + Rosetta 2 | PostgreSQL, MySQL, MariaDB, MongoDB, SQL Server 2019/2022, Oracle 23ai | ~1.5–2× native |
| VM 2 (correct) — `forgedb-qemu` | `qemu` (x86_64) | DB2, Informix, Oracle 19c/21c | ~5–10× slower but correct |

The Lima VM names used by the `forgedb` project are `forgedb-vz` (VZ) and `forgedb-qemu` (QEMU). Commands use `podman --connection <vm> ...` (via a registered system connection that routes through the Lima Unix socket).

`vmType` is baked in at creation time and cannot be changed — delete and recreate if you need to change it.

## Three backend options

| Backend | VM 1 CLI | VM 1 manager | QEMU VM command |
|---------|----------|-------------|----------------|
| **Podman Machine** (`_common.sh`) | `podman run` | `podman machine` (transparent) | `limactl shell forgedb-qemu -- sudo podman ...` |
| **Lima + nerdctl** (`_common_lima.sh`) | `limactl shell forgedb-vz -- sudo nerdctl ...` | `limactl` | `limactl shell forgedb-qemu -- sudo podman ...` |
| **Colima** (`_common_colima.sh`) | `nerdctl run` | `colima` (wraps Lima) | `limactl shell forgedb-qemu -- sudo podman ...` |

All three setups use exactly two VMs for full coverage. The difference is only how VM 1 is managed. DB2 and Informix always use the Lima QEMU `forgedb-qemu` VM.

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
limactl create --name forgedb-vz \
  --set '.vmType = "vz"' \
  --set '.vmOpts.vz.rosetta.enabled = true' \
  --set '.vmOpts.vz.rosetta.binfmt = true' \
  template:default
limactl start forgedb-vz
limactl shell forgedb-vz -- sudo systemctl enable --now containerd

# Wrapper function used in scripts
_lnerdctl() { limactl shell "${LIMA_VM:-forgedb-vz}" -- sudo /usr/local/bin/nerdctl "$@"; }
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

### MariaDB

```bash
podman run -d --name "mariadb${VERSION}" \
  -v "mariadb${VERSION}-data:/var/lib/mysql" \
  -p "${PORT}:3306" \
  -e MARIADB_ROOT_PASSWORD="$dba_pass" \
  "mariadb:${VERSION}"
```

- Multi-arch image (ARM64 native on Apple Silicon) — runs in `forgedb-vz`
- **CLI binary**: MariaDB 11.4+ ships `mariadb` (not `mysql`). The `mysql` binary is still present as an alias in 11.4 but may be absent in future versions. Always invoke `mariadb` explicitly.
- **SQLAlchemy dialect**: use `mariadb+pymysql://` — the `mariadb://` dialect alone defaults to `MySQLdb` which is not installed; explicitly map `mariadb` → `pymysql` in driver config.
- Readiness: `podman exec $container mariadb -u root -p"${dba_pass}" -e "SELECT 1"` (use `mariadb` binary, not `mysql`)
- Env var for password in scripts: `MYSQL_PWD` is accepted by the `mariadb` binary (same as `mysql`)

### SQL Server

```bash
podman run -d --name "sqlserver${VERSION}" \
  -v "sqlserver${VERSION}-data:/var/opt/mssql" \
  -p "${PORT}:1433" \
  --platform linux/amd64 \
  -e ACCEPT_EULA=Y \
  -e MSSQL_SA_PASSWORD="$dba_pass" \
  -e MSSQL_AGENT_ENABLED=true \
  "mcr.microsoft.com/mssql/server:${VERSION}"
```

- `--platform linux/amd64` required — no native ARM64 image; Rosetta 2 translates (SQL Server 2019/2022 run in `forgedb-vz`)
- Default version: `2022-latest` (SQL Server 2025 known not to work with macOS Rosetta at time of testing)
- Default base port: `1433`
- **`MSSQL_AGENT_ENABLED=true` is required for CDC** — SQL Server Agent runs the CDC capture jobs; without it, CDC is enabled at the database level but changes are never captured into the CDC change tables. Always include this env var when provisioning for CT+CDC (`BOTH` mode).
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

- Default PDB: `FREEPDB1` (Oracle Free) — derived automatically from the image version
- `--shm-size=1g` required for Oracle shared memory
- Oracle 23ai Free has a native ARM64 image → runs in `forgedb-vz`. Oracle 19c/21c are x86_64-only and Rosetta fails → must use `forgedb-qemu`.
- **Public gvenzl images** (no Oracle SSO login required): `gvenzl/oracle-xe:21`, `gvenzl/oracle-free:23`. Use `ORACLE_PASSWORD` env var (not `ORACLE_PWD`) with these images.
- Readiness probe must use `can_connect()` (TCP + actual SELECT), not just TCP — the TCP port opens before the PDB is registered (ORA-12514 false positive if you check TCP only). Poll until `SELECT 1 FROM DUAL` succeeds.
- Init takes 3–8 min; poll with 600 s timeout.
- **ARCHIVELOG + supplemental logging timing**: After running the SHUTDOWN/STARTUP MOUNT/ALTER DATABASE ARCHIVELOG/OPEN sequence inside the container, the CDB root service (e.g. `FREE`) takes additional time to accept connections even though the PDB accepts them immediately. **Must re-poll `can_connect()` on the PDB** after ARCHIVELOG restart before calling the supplemental logging configure step (`ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS`). Without this wait, the CDB connection fails silently and supplemental logging is skipped.

### DB2 (QEMU Lima VM — always)

DB2 always runs inside the Lima QEMU `forgedb-qemu` VM:

```bash
# Run DB2 container through the registered Podman connection
podman --connection forgedb-qemu run -d --name lfc-db2-115 \
  -v lfc-db2-115-data:/database:Z \
  -p 50000:50000 \
  --privileged \
  -e DB2INST1_PASSWORD="$dba_pass" \
  -e DBNAME="${catalog}" \
  -e LICENSE=accept \
  -e ARCHIVE_LOGS=false \
  -e AUTOCONFIG=false \
  "icr.io/db2_community/db2:latest"
```

**Critical flags:**
- `--privileged` — required by IBM for DB2 (grants System V IPC and other kernel capabilities needed by `db2ftok`)
- No `--platform linux/amd64` needed when running in `forgedb-qemu` (QEMU presents as x86_64)
- `:Z` suffix on the volume mount — required for SELinux relabeling inside the QEMU VM

**Three-stage readiness check — all three must pass before configuring:**

1. **Log sentinel**: Wait for `"Setup has completed"` in container logs (up to 35 min under QEMU). TCP port 50000 opens before DB2 is ready — polling TCP alone gives false positives.
2. **TCP reachability**: Poll `tcp_reachable("localhost", 15000)` until the port responds.
3. **Catalog availability**: Even after TCP is reachable, the `lfctest` catalog is not immediately accessible — DB2 registers it in its directory slightly after the port opens. Poll `can_connect(host, port, "db2inst1", pw, catalog)` (runs `SELECT 1 FROM SYSIBM.SYSDUMMY1`) until it succeeds. Skipping this step causes `SQL30061N: database not found` errors from provisioning code.

```python
# Correct readiness sequence (forgedb db2/podman.py)
podman_wait_for_log(container, "Setup has completed", timeout=2100)
# TCP poll ...
while not tcp_reachable("localhost", host_port): time.sleep(5)
# Catalog poll — critical, do not skip
while not can_connect("localhost", host_port, "db2inst1", pw, catalog): time.sleep(5)
```

**DB2 database name limit:** 8 characters maximum. Use `lfctest` (7 chars) not `lfctestdb`.

### Informix (QEMU Lima VM — always)

IBM Informix Developer Database is x86_64-only → runs in `forgedb-qemu`:

```bash
podman --connection forgedb-qemu run -d --name lfc-informix-1410 \
  -v lfc-informix-1410-data:/opt/ibm/data \
  -p 9088:9088 \
  --privileged \
  -e LICENSE=accept \
  -e INFORMIX_PASSWORD="$adm_pass" \
  -e DB_INIT=1 \
  -e DBNAME=lfctest \
  "icr.io/informix/informix-developer-database:latest"
```

- Container port `9088` is forwarded to macOS host port `19088` via Lima portForward in `forgedb-qemu.yaml`
- Readiness: Wait for `"Informix Dynamic Server Started"` in logs (up to 10 min), then poll TCP
- **OS / PAM authentication**: Informix authenticates users by OS identity — every database user must exist as an OS user inside the container (`useradd -m -g informix <user>`). There is no password-based SQL auth.
- **`dbaccess` CLI for SQL**: IfxPy Python driver requires the Informix CSDK which has no pre-built macOS ARM64 binary. Execute all SQL via `dbaccess` inside the container using `podman exec -u <os_user>`.
- **Environment variables must be injected**: `dbaccess` is only on PATH when Informix environment variables are set. Either use a login shell (`bash -l -c`) to source the informix user's profile, or explicitly inject:
  ```
  INFORMIXDIR=/opt/ibm/informix/v15.0.1.0.3
  INFORMIXSERVER=informix
  PATH=$INFORMIXDIR/bin:/usr/local/bin:/usr/bin:/bin
  ```
- **`CONNECT TO ... USING 'password'` is NOT supported** by `dbaccess` — it uses OS user identity only. Attempting this syntax produces `USING clause unsupported. DB-Access will prompt you for a password.`
- **`WITH LOG` required for CDC**: Databases must be created `WITH LOG` to enable transaction logging. A database created without `WITH LOG` cannot support CDC (`is_logging=0` in `sysmaster:sysdatabases`).
- **User grants for LFC**: `GRANT CONNECT TO <user>; GRANT RESOURCE TO <user>; GRANT DBA TO <user>;`

---

## Lima socket stability — Lima 2.x uses SSH forwarder (reliable)

**Lima 1.0.1 (Nov 2024) fixed the socket proxy.** Before 1.0.1, Lima used a gRPC port forwarder that panicked under load, forcing use of `limactl shell` (SSH) as a workaround. Since 1.0.1, Lima reverted to SSH port forwarding as the default, which is stable.

**Confirmed working in Lima 2.1.1 (tested May 2026):** After a clean `limactl stop` + `limactl start`, the socket at `~/.lima/<vm>/sock/podman.sock` is stable and `podman --connection <vm>` works reliably with no EOF errors.

**Verified behaviours:**
- `CONTAINER_HOST=unix://$HOME/.lima/forgedb-vz/sock/podman.sock podman --remote version` → returns server version with no EOF
- `podman --connection forgedb-vz ps` → works reliably
- `podman-compose` via `CONTAINER_HOST` → resolves container-to-container DNS (hostname `postgres` → IP), pulls images, `SELECT 1` returns correctly
- The Lima boot log shows `Forwarding "/run/podman/podman.sock" (guest) to "~/.lima/forgedb-vz/sock/podman.sock" (host)` when the socket is active

**The correct approach — use `podman --connection`:**

```bash
# Register the connection once (setup_local_env.py does this automatically)
podman system connection add forgedb-vz "unix://$HOME/.lima/forgedb-vz/sock/podman.sock"
podman system connection add forgedb-qemu "unix://$HOME/.lima/forgedb-qemu/sock/podman.sock"
podman system connection default forgedb-vz

# Then use normally — no limactl shell needed
podman --connection forgedb-vz ps
podman --connection forgedb-qemu images
```

**Shell aliases (add to `~/.zshrc`):**

```bash
alias pvz='podman --connection forgedb-vz'      # MySQL / PostgreSQL / Oracle 23ai
alias pqemu='podman --connection forgedb-qemu'  # Oracle 19c/21c / SQL Server / DB2
```

**For `podman-compose` and Testcontainers**, set `CONTAINER_HOST`:

```bash
export CONTAINER_HOST=unix://$HOME/.lima/forgedb-vz/sock/podman.sock
podman-compose -f docker-compose.yml up
```

**Python `_base()` pattern** in `container_helpers.py`:

```python
def _base(connection: Optional[str]) -> list[str]:
    if _RUNTIME == "docker":
        return ["docker"]      # relies on DOCKER_HOST or default context
    cmd = ["podman"]
    if connection:
        cmd += ["--connection", connection]
    return cmd
```

No `limactl shell` branch needed. `podman --connection forgedb-vz` routes through the forwarded socket.

**`limactl shell` is still needed for system administration** (not container operations):
- Checking systemd status: `limactl shell forgedb-vz -- sudo systemctl is-active podman.socket`
- Editing guest files: `limactl shell forgedb-vz -- sudo bash -c "..."`
- `setup_local_env.py` uses it for `_ensure_podman_socket()`, `_ensure_registries_conf()`, `_configure_storage_conf()`

**"Broken" state invalidates the socket.** If `limactl list` shows `Broken`, the Lima hostagent is down and the socket forwarder is not active — `podman --connection` will EOF. Always do a clean stop/start before testing the socket:

```bash
limactl stop forgedb-vz      # fully stops hostagent + VM
limactl start forgedb-vz     # clean start; socket forwarder re-initializes
limactl list forgedb-vz      # must show "Running", not "Broken"
```

A VM that shows "already running" after `limactl start` (because the VM process survived but hostagent died) may still have an inactive socket forwarder — verify with `limactl list` status.

**Full delete/recreate** (only if stop/start fails — destroys all container volumes):

```bash
limactl delete forgedb-vz
limactl start --name=forgedb-vz config/lima/forgedb-vz.yaml --tty=false
```

`vmType` and provisioned Podman configuration survive a stop/start. Named Podman volumes inside the VM are lost only on `limactl delete`.

---

## registries.conf — required for unqualified image names

Podman inside the Lima guest (Ubuntu 24.04 apt ships Podman 4.9.3) has no unqualified-search registries configured by default. Pulling `postgres:16-alpine` (without `docker.io/library/` prefix) fails:

```
Error: short-name "postgres:16-alpine" did not resolve to an alias
and no unqualified-search registries are defined in "/etc/containers/registries.conf"
```

**Fix (idempotent, run via `limactl shell`):**

```bash
limactl shell forgedb-vz -- sudo bash -c "
  if ! grep -q 'docker.io' /etc/containers/registries.conf 2>/dev/null; then
    printf '\n[registries.search]\nregistries = [\"docker.io\", \"quay.io\"]\n' \
      >> /etc/containers/registries.conf
  fi"
```

`setup_local_env.py` applies this automatically via `_ensure_registries_conf()` on every `step_lima_vm()` call. The Lima YAML provision scripts also include this step for new VMs.

**Alternative: always use fully-qualified image names (safer long-term):**

```bash
docker.io/library/postgres:16-alpine   # instead of postgres:16-alpine
docker.io/library/mysql:8.4            # instead of mysql:8.4
```

---

## Podman client vs guest version mismatch

The macOS Homebrew Podman client (5.8.1) and the Podman server inside an existing Lima VM (e.g. 4.9.3 from Ubuntu apt) may have different versions. The Podman API is generally backward compatible for basic operations. To upgrade the guest:

```bash
limactl shell forgedb-vz -- sudo apt-get update
limactl shell forgedb-vz -- sudo apt-get install -y podman
limactl shell forgedb-vz -- podman --version   # verify
```

For new VMs, the Lima YAML provision scripts install Podman 5.x from the kubic unstable repo.

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

| Engine | Container port | macOS host port | Note |
|--------|---------------|-----------------|------|
| PostgreSQL | `5432` | dynamic | |
| MySQL | `3306` | dynamic | |
| MariaDB | `3306` | dynamic | |
| CockroachDB | `26257` | dynamic | |
| SQL Server | `1433` | dynamic | |
| Oracle | `1521` | dynamic | |
| MongoDB | `27017` | dynamic | |
| DB2 | `50000` | `15000` | Lima portForward in forgedb-qemu.yaml: guestPort 50000 → hostPort 15000 |
| Informix | `9088` | `19088` | Lima portForward in forgedb-qemu.yaml: guestPort 9088 → hostPort 19088 |

DB2 and Informix use fixed Lima portForwards because `ibm_db_dbi` / `dbaccess` connect to known ports. All other engines use `find_free_port()` at provision time.

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

Running multiple containers in `forgedb-qemu` simultaneously (DB2, Informix, Oracle 19c/21c) shares the single QEMU process's CPU budget. For full-suite `--rebuild` test runs, engines are provisioned sequentially within each VM. The VZ engines (MySQL, MariaDB, PostgreSQL, SQL Server, MongoDB, Oracle 23ai) all run in `forgedb-vz` and provision much faster.

Approximate first-boot times under QEMU:
- DB2 first-boot: 20–30 min
- Informix first-boot: 2–5 min
- Oracle 19c/21c first-boot: 5–10 min
- Oracle 23ai (VZ, native): 3–5 min
