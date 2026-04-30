---
name: developing-a-plan
description: >-
  How to build, present, and iterate on an implementation plan before executing.
  Use when the user asks for a plan, before starting multi-step work that crosses
  skill boundaries, or when any step requires an experiment because the relevant
  skill is untested or may not apply. Covers skill-gap assessment, blocker
  identification, design-decision surfacing, and plan iteration protocol.
---

# Developing a Plan

## When to invoke this skill

- User explicitly asks for a plan before execution
- Task spans multiple skills (data load + pipeline + spreadsheet write, etc.)
- Any step requires an experiment because the skill is marked ⚠ not tested
- Any prerequisite state is unknown (data loaded? connections exist? tokens valid?)

## Step 1 — Inventory skills honestly

For every capability the plan requires, classify it:

| Mark | Meaning |
|------|---------|
| ✓ tested | Skill covers it AND has been live-tested in this environment |
| ⚠ not tested | Skill has the pattern but it has never been run live here |
| ✗ gap | No skill covers it — must be written or decided |
| ✗ blocker | External state must be true before this step can run |

**Rule**: never claim a capability is available if the skill is marked ⚠ not tested. Surface it as an experiment step in Phase 0.

## Step 2 — Surface blockers and design decisions

Blockers and decisions must be listed explicitly before the phase list:

**Blockers** — things outside the agent's control that gate progress:
- Data that hasn't finished loading
- Infrastructure that may not exist (connections, schemas, VMs)
- Auth tokens that may be expired

**Design decisions** — ambiguous choices the user must make before the agent can proceed:
- How a parameter should be interpreted (e.g., "row count" means which table?)
- Which of two valid approaches to take
- Whether to reuse an existing resource or create a new one

Never silently pick a design default. State the ambiguity and which default you would use if the user says "go ahead".

## Step 3 — Structure the plan

```
### Phase 0 — Verify blockers and run experiments
  0a. <blocker check> — [command or query to run]
  0b. <experiment> — ⚠ not tested; run this to confirm before proceeding

### Phase N — <goal>
  Na. <step> — skill: <skill name> | status: ✓/⚠/✗
  ...

### What I cannot do without your input
  | Item | Why |
```

Rules:
- Phase 0 always comes first — no assumptions about state.
- Each step names the skill it uses and its test status.
- Steps blocked by Phase 0 outcomes are explicitly labelled "blocked on 0a" etc.
- "What I cannot do" table lists anything that requires user input or decision.

## Step 4 — Present the plan, wait for approval

Do not execute anything until the user approves the plan or says "go ahead".
If the user modifies the plan, update it and confirm before executing.

## Step 5 — Iterate on the plan during execution

Update the plan in-place as execution reveals new information:
- Mark steps ✓ complete, ✗ failed, or ⚡ modified.
- If an experiment (Phase 0) fails, re-plan the affected phases before continuing.
- If a blocker is unresolved, note it and skip ahead to unblocked phases.

## Real example — filling a benchmark spreadsheet (2026-04-30)

**Task**: Fill 18 empty rows in a Google Sheet tracking LFC pipeline ingestion time across 3 DB engines, 2 network routes, 4 row counts.

**Skill inventory produced:**

| Capability | Skill | Status |
|-----------|-------|--------|
| Read Google Sheets | google-workspace-api | ✓ tested |
| Write Google Sheets | google-workspace-api | ⚠ not tested |
| Create LFC pipeline | databricks-3-setup-uc-lakeflow.sh | ✓ works |
| Run pipeline + get update_id | databricks-pipeline-operations | ✓ skill covers it |
| Measure RUNNING→COMPLETED time | databricks-pipeline-operations | ✓ skill covers it |
| Select table by row count | databricks-3-setup-uc-lakeflow.sh | ✗ hardcodes "intpk" |
| Data load status 100M/1B rows | synthetic-data-in-database | ✗ unknown — needs check |
| UC connections for postgres/sqlserver | customer setup scripts | ✗ unknown — needs check |

**Design decision surfaced**: `databricks-3-setup-uc-lakeflow.sh` hardcodes `source_table: "intpk"`. The benchmarks require different table sizes (`intpk_1m`, `intpk_10m`, etc.). Decision needed: add a table-name arg to the script, or clarify that "time for 100M rows" means scanning a growing `intpk` table incrementally.

**Plan structure produced**: Phase 0 (verify blockers + Sheets write experiment) → Phase 1 (close script gaps) → Phase 2–4 (run pipelines, write results per engine).

## Presenting options — format and criteria

When multiple valid approaches exist, present them in a table with explicit criteria scoring before making a recommendation.

**Criteria should be defined by the user's stated goal** — e.g., "easy reproducibility" overrides "operational simplicity".

### Example: single vs multiple pipeline strategies for LFC benchmarking (2026-04-30)

**Task**: measure ingestion time for 1M / 10M / 100M / 1B rows across 3 DB engines, 2 network paths → fill a benchmark spreadsheet.

| | Option A — one pipeline per table | Option B — same as A, table-name arg | Option C — one pipeline, add tables progressively |
|--|--|--|--|
| Pipelines created | 24 (3 DB × 2 routes × 4 sizes) | 24 | 6 (3 DB × 2 routes) |
| First-run reproducibility | ✓✓ self-contained | ✓✓ self-contained | ✓ add table then trigger |
| Re-run reproducibility | ✓✓ just re-trigger | ✓✓ just re-trigger | ✓ `--full-refresh-selection <table>` |
| Per-table timing isolation | ✓✓ pipeline has one table | ✓✓ pipeline has one table | ✓ `flow_progress` events per-table |
| Operational overhead | High (24 pipelines to manage) | High (24 pipelines) | Low (6 pipelines) |
| Pipeline_id in spreadsheet | One per row | One per row | Shared across row-count rows |
| Matches existing data pattern? | ✗ | ✗ | ✓ (azure-mysql public rows 1M and 10M share the same pipeline_id) |

**Verdict (given "easy reproducibility")**: Option C is the best choice.
- Reproducibility is equal to A/B once you know the re-run mechanic (`--full-refresh-selection`)
- 6 pipelines vs 24 is dramatically simpler to maintain
- Already matches the established pattern in the existing benchmark data
- Single pipeline per connection tells a cleaner story in the Databricks UI

**Script change required for Option C**: add `source_table` as a 5th argument. On first call for a new table, `PATCH` the pipeline to append the table to `objects[]`. On re-run, trigger `--full-refresh-selection <table>`.

**Key insight**: when both row-count runs share a pipeline_id in the spreadsheet, the `update_id` column is what distinguishes individual runs — each update captures exactly one table's snapshot.

## Anti-patterns to avoid

- **Don't assume state** — never skip Phase 0 because "it probably worked last time".
- **Don't execute during planning** — only read/inspect; no writes until plan is approved.
- **Don't hide ⚠ gaps** — if a pattern is untested, say so even if you're confident it works.
- **Don't pick design defaults silently** — always surface ambiguous choices to the user.
- **Don't skip "What I cannot do"** — if anything requires user input, it must appear in that table.
