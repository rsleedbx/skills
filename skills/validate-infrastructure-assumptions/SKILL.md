---
name: validate-infrastructure-assumptions
description: >-
  Validate infrastructure claims before writing code — check whether a GitHub issue is
  still open and unfixed, test in a clean environment (not a degraded/Broken state), and
  confirm version numbers before citing bugs. Use whenever a claim about a tool, runtime,
  or protocol behaviour is sourced from a blog post, GitHub issue, or forum thread rather
  than a live test, or when a workaround exists whose original justification has not been
  verified against the installed version.
---

# Validate Infrastructure Assumptions Before Writing Code

## The core rule

**Never write a workaround for a bug you have not confirmed exists in the version you are
actually running.**

Internet sources (blog posts, GitHub issues, Stack Overflow answers) describe the state of
the world at the time they were written. Bugs get fixed. Regressions get reverted. The
installed version on your machine may be months or years newer than the source describing
the problem.

---

## Case study 1 — Lima socket proxy (May 2026)

### What was claimed

> "On macOS, Podman 5.x's Unix socket proxy is unreliable — it produces EOF errors when
> the macOS Podman client tries to talk to a Lima VM over the forwarded socket. The reliable
> alternative is `limactl shell` (SSH-based)."

This claim was added to `container_helpers.py` as a code comment and drove the entire
`_base()` implementation:

```python
# Wrong workaround — added before validation
if connection and connection in _LIMA_VMS:
    return ["limactl", "shell", "--workdir=/tmp", connection, "--", "sudo", _RUNTIME]
```

### What was actually true

- Lima **1.0.1 (Nov 2024)** fixed the gRPC port forwarder that caused panics by reverting
  to SSH forwarding as the default.
- The installed version at time of testing was **Lima 2.1.1 (May 2026)** — 14 months past
  the fix.
- The GitHub issue used as evidence described a Lima v1.0.0 bug, not the current behaviour.

### Why the first test was invalid

The first attempt to test the socket ran on a VM in **"Broken"** state. Lima marks a VM
"Broken" when its hostagent process has died while the underlying QEMU/VZ process is still
running. In that state the SSH port forwarder is not running and any socket connection will
immediately get EOF — **regardless of Lima version**. The test confirmed "EOF exists" not
"Lima 2.1.1 has this bug."

### The clean test

After `limactl stop` + `limactl start` (VM status `Running`, not `Broken`):

```bash
CONTAINER_HOST=unix://$HOME/.lima/forgedb-vz/sock/podman.sock podman --remote version
# Server: Podman Engine  Version: 4.9.3  OS/Arch: linux/arm64  ← no EOF
```

**Result:** socket works; `limactl shell` workaround removed; 30+ lines of SSH overhead
per container command eliminated.

### What the validation should have been

Before writing the workaround:
1. Check Lima changelog: `limactl --version` → look up release notes for that version
2. Search the GitHub issue for "closed" / "fixed in" / "released in" labels
3. Run a clean test on a `Running` (not `Broken`) VM:
   ```bash
   limactl stop forgedb-vz && limactl start forgedb-vz
   limactl list forgedb-vz          # must show Running
   CONTAINER_HOST=unix://... podman --remote version
   ```

---

## Case study 2 — Podman DNS regression (May 2026)

### What was claimed

> "Podman 5.5+ with netavark/aardvark-dns 1.15+ has a known bug breaking
> container-to-container DNS resolution."

This was cited as a reason to prefer Docker Engine over Podman.

### What was actually true

- `netavark 1.15.0` (released ~May 2025) had a DNS search-domain regression.
- `netavark 1.15.1` fixed it **within days**.
- The GitHub issue (`containers/podman#26198`) was closed/fixed before it affected any
  stable release that reached most users.

### The consequence

The Podman DNS claim was used in `docs/container-runtime-decision.md` as a "Known Issue"
bullet point under Podman, influencing the decision to migrate to Docker Engine. The claim
was factually false for any system running netavark ≥ 1.15.1.

### What the validation should have been

1. Check the issue status: open or closed? When was it closed?
2. Check the fixed version: `netavark --version` on the test machine
3. Actually test DNS between two containers before citing it as a blocker:
   ```bash
   # Two-container DNS test — does 'client' resolve 'postgres' by name?
   podman-compose -f /tmp/test-compose.yml up --abort-on-container-exit
   # Check if client can SELECT 1 via hostname 'postgres'
   ```

---

## Validation checklist — before writing any workaround

Run through this before adding a workaround, a comment citing a bug, or a migration
rationale based on a third-party source.

| Step | Action |
|------|--------|
| 1 | **Check installed version** — `tool --version`. Note the exact version. |
| 2 | **Find the source** — locate the GitHub issue, blog post, or forum thread. |
| 3 | **Check issue status** — is it Open or Closed? If closed: what version fixed it? |
| 4 | **Compare versions** — two paths: |
| | **Path A — Bug is fixed:** Is the fix version older than what is installed? If yes → do not add the workaround; validate with a live test instead. |
| | **Path B — Bug may still apply:** Is your installed version older than the fix? If yes → upgrade first, then re-test before writing a workaround. A workaround written against an old version may become dead code the moment you upgrade. |
| 5 | **Clean environment** — stop/restart the service/VM to eliminate degraded state. |
| 6 | **Run the actual test** — reproduce the failure (or confirm it cannot be reproduced). |
| 7 | **Test the alternative too** — confirm the proposed replacement actually works. |

If steps 3–4 show the bug is fixed and step 6 cannot reproduce it: **do not add the
workaround**.

If step 4 (Path B) shows your version predates the fix: **upgrade first**. Writing a
workaround for a bug that a version bump would resolve adds permanent technical debt and
masks the real action needed.

---

## Case study 3 — Path B example: old version, bug is real

Suppose `podman-compose` 1.0.6 has a known bug where it ignores `depends_on` ordering.
The bug is fixed in 1.1.0.

**Wrong response:** write a workaround in the Python setup code that manually sequences
container starts.

**Right response:**
```bash
podman-compose --version          # 1.0.6 — older than fix
pip install --upgrade podman-compose
podman-compose --version          # 1.1.0
# Re-test — bug gone. No workaround needed.
```

A workaround added before checking the installed version becomes permanently entangled
in the codebase even after the upgrade. The version check takes 10 seconds; the
workaround removal and its side effects may take hours.

---

When a Lima VM shows `Broken` status, its hostagent is down. This makes the Podman socket
unreachable — not because of any Podman or Lima bug, but because the forwarder process
is not running.

Any test run against a `Broken` VM produces misleading results:
- Socket connections → EOF (looks like a Podman/Lima bug)
- `podman --connection` → "cannot connect" (looks like a Podman bug)
- `limactl shell` may or may not work depending on whether the VM process is alive

**Always verify VM state before running socket tests:**

```bash
limactl list <vm>          # must show "Running"
# If Broken:
limactl stop <vm>
limactl start <vm>
limactl list <vm>          # confirm Running before any socket test
```

---

## Pattern: state-invalidated tests

A test that appears to confirm a bug but was run in a degraded system state is a
**state-invalidated test**. Signs that a test may be state-invalidated:

- The service/VM/daemon was recently restarted, killed, or in an unknown state
- The test was run immediately after a crash or error without a clean restart
- The error is "connection refused" or "EOF" — these can be caused by the process
  simply not being up, independent of any bug

When a test appears to confirm a serious bug (especially one that would justify
rewriting production code), repeat the test from a clean state before acting on it.

---

## Citing sources in code comments and documentation

When a code comment or doc cites a bug or issue as the reason for a workaround:

**Include:**
- The exact URL of the issue or source
- The affected version range (e.g., "Lima v1.0.0 only")
- The fixed version (e.g., "fixed in Lima v1.0.1")
- The installed version at time of writing (e.g., "verified broken on Lima v1.0.0")

**Bad:**
```python
# limactl shell used because podman --connection is unreliable on macOS
```

**Good:**
```python
# Lima <1.0.1 gRPC forwarder caused EOF on macOS (github.com/lima-vm/lima/issues/2859).
# Fixed in Lima 1.0.1 (Nov 2024). Validated working on Lima 2.1.1 — no workaround needed.
```

If you cannot fill in the fixed version and a validation test, do not add the workaround
yet — run the validation first.
