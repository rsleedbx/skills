---
name: docs-organization
description: Three-tier documentation structure — current-state docs, options comparison docs, and per-option deep-dive docs. Use when creating or reorganizing project documentation, deciding where new content belongs, splitting a doc that mixes current behavior with alternatives, or linking between doc tiers.
---

# Docs Organization — Three-Tier Pattern

Documentation is split into three tiers. Each tier has a single job.

---

## Tier 1 — `docs/<name>.md` (current state)

Describes **what the code does now**. Nothing else.

- No "why we didn't choose X" content.
- No comparison tables between alternatives.
- If a decision is still pending (multiple options being evaluated), add a single link at
  the bottom pointing to the options doc:
  ```markdown
  See [docs/options/<name>.md](options/<name>.md) for options under evaluation.
  ```
- Once a decision is made, the options doc moves to **See also** — it is not removed, but
  it is no longer surfaced as a pending decision.

**Sections that belong here:** architecture diagram, routing table, port assignment,
bootstrap steps, aliases, VM details, troubleshooting, CLI reference.

**Sections that do not belong here:** runtime alternatives, rejected approaches, deep
internals explaining *why* the mechanism works, historical notes about previous
implementations.

---

## Tier 2 — `docs/options/<name>.md` (options)

Summarizes the viable choices **side by side** and states which one was chosen and why.

- One comparison table at the top covering all options.
- One paragraph per rejected option explaining the specific failure reason.
- The chosen option is marked (e.g. `**Default**`, `**Current**`).
- Links down to `docs/options/alternatives/<option-name>.md` for per-option depth.
- Links back up to the current-state doc in **See also**.

**Rule:** if the reason for rejection fits in one sentence, it goes here. If it requires
a benchmark, log excerpt, or multi-step explanation, it goes in the alternatives doc.

---

## Tier 3 — `docs/options/alternatives/<option-name>.md` (per-option deep dive)

One file per option — including rejected options. Rejected options are kept permanently;
they may become relevant again when tool versions change.

Each file covers:
- What the option is and how it works
- Setup/install steps (if it were chosen)
- Specific test results, log excerpts, or benchmarks that informed the decision
- Known issues or version-specific behavior
- Status: `Chosen`, `Rejected — <one-line reason>`, or `Under evaluation`

---

## Naming convention

Use the decision area as the shared prefix across all three tiers:

| Tier | Path | Example |
|---|---|---|
| Current state | `docs/<area>.md` | `docs/local-container.md` |
| Options | `docs/options/<area>.md` | `docs/options/local-container.md` |
| Per-option | `docs/options/alternatives/<area>-<option>.md` | `docs/options/alternatives/local-container-docker-engine.md` |

---

## What moves between tiers

When editing any doc, apply this test to each section:

| Section content | Tier |
|---|---|
| What the code does today | 1 — current state |
| Comparison of multiple approaches | 2 — options |
| Deep internals explaining *why* a mechanism works | 2 — options, or 3 — alternatives |
| Full evaluation of a specific alternative (pros, cons, test results) | 3 — alternatives |
| Historical note about a previous implementation | 2 — options or 3 — alternatives |
| Forward-looking guidance (feature not yet in code) | 2 — options |

---

## Example — forgedb local container docs

```
docs/local-container.md              ← Lima + Podman, two VMs, routing, ports, aliases
docs/local-container-alternatives.md ← runtime comparison table, Docker Compose with Podman,
                                        Lima port forwarding internals, cross-VM networking options
docs/container-runtime-decision.md  ← full May 2026 evaluation (Podman vs Docker Engine vs nerdctl)
```

The `local-container-alternatives.md` serves as both Tier 2 (options summary) and a
holding area for Tier 3 content until the `docs/options/alternatives/` subtree is
created. When that subtree is created, split it:
- Comparison table + one-paragraph rejections → `docs/options/local-container.md`
- Lima port forwarding internals → `docs/options/alternatives/local-container-lima-internals.md`
- Cross-VM container networking → `docs/options/alternatives/local-container-cross-vm-networking.md`
- Docker Compose with Podman → `docs/options/alternatives/local-container-compose-podman.md`
