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

## Step 2b — Define the end state for every phase

**Before writing any phase, state what "done" looks like for that phase.** This is not a log message — it is the actual state of the system that must be true for the phase to be complete.

Write the end state as a checkable condition:

| Phase goal | End state (checkable) | How to verify |
|------------|----------------------|---------------|
| Load 1M rows into intpk | `MAX(pk) = 1,000,000` in the DB | `SELECT MAX(pk) FROM intpk;` |
| Pipeline ingests a table | Destination Delta table has the expected row count | `SELECT COUNT(*) FROM <catalog>.<schema>.<table>` |
| Pipeline completed | Update reached COMPLETED state | Events API shows COMPLETED event |
| Sheet write succeeded | Cell H18 contains a non-empty row count | Read cell H18 back from the Sheets API |

**Rules:**
- End state is the actual system state, not a log message or exit code.
- **If the goal involves rows, verify row counts.** A pipeline that says COMPLETED but ingested 0 rows is a failure — duration alone cannot detect it. Always check actual rows in the destination.
- If no rows expected (e.g. cleanup), verify the table is empty — not just that the DELETE ran.
- Write the end state into the phase description before executing. If you cannot define a checkable end state, the goal is too vague.
- For LFC benchmarks: `duration_s` measures pipeline wall time; `actual rows` measures whether data actually moved. Both are required. A long duration with 0 actual rows means the source was empty, the connection failed silently, or the table name was wrong.

**Monitoring validates the end state, not the script output.** Log lines like `updated: Sheet1!E18:G18` confirm the API call was made — they do not confirm the cell has the correct value. End-state verification reads back the actual value to confirm it is correct.

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

## Monitor first run before waiting — applies to ALL systematic testing

**Rule: the first run after any script or configuration change must be monitored to completion before backgrounding or waiting for subsequent iterations.**

"First run after touching" means: any run that follows a code change, config fix, new connection, new pipeline, or new DB key — even if prior runs existed. The second run after a fix is still the *first validated run* of the fixed version.

**Why**: failures in systematic tests (LFC benchmarks, data loads, API integrations) repeat identically for every iteration. Catching a failure in iteration 1 takes 2 minutes; discovering it after 8 backgrounded iterations have all silently failed wastes hours and corrupts output (wrong data written to sheets, pipelines stuck in FAILED state).

**Pattern for any scripted test loop:**

```
1. Run iteration 1 in the FOREGROUND — watch output live.
2. Verify the success criteria are met:
   - Correct config printed (right host, right schema, right table)
   - First action succeeded (pipeline RUNNING, not stuck; insert started; API returned 200)
   - First iteration reached the expected end state (COMPLETED, rows written, file created)
3. Only then background the remaining iterations.
```

**Applies to:**
- LFC pipeline benchmarks (watch first `COMPLETED` before `nohup bash bench-run-all.sh`)
- Bulk data loads (watch first batch complete before leaving unattended)
- API integrations (watch first write succeed before looping)
- Any `for` loop or `while` loop over a set of inputs

**What "monitoring" means — verify the end state, not just the log:**

Monitoring is not reading log lines. Monitoring is confirming that the actual system state matches the end state defined in the plan. Log lines are hints; the end state is the truth.

**Before starting any test loop, write down the expected end state for iteration 1:**

| Goal | Expected end state after iteration 1 | How to verify (not just the log) |
|------|--------------------------------------|----------------------------------|
| Ingest 1M rows via LFC | Destination Delta table has 1,000,000 rows AND sheet H18 = 1000000 | `SELECT COUNT(*) FROM <catalog>.<schema>.intpk_1m`; read H18 from sheet |
| Load 1M rows into DB | `MAX(pk) = 1,000,000` in the table | Run `SELECT MAX(pk) FROM intpk_1m` |
| Pipeline completed | Update in COMPLETED state with non-zero actual rows | Events API shows COMPLETED; count query returns expected rows |
| API write succeeded | Cell value matches what was written | Read the cell back after writing |
| Table is empty after cleanup | `COUNT(*) = 0` or table does not exist | Run count query or check metadata |

**If the expected end state is not reached after iteration 1, stop immediately.** Do not background the remaining iterations — the same failure will repeat for every one.

**Stop conditions — if any of these appear in the first run, stop before backgrounding:**
- `WARNING: no matching row found` — sheet lookup failed (column mismatch, wrong DB key name)
- `Result NOT written to sheet` — means the output was silently discarded
- Empty `duration_s` — means timestamp parsing failed (pipeline likely FAILED, not COMPLETED)
- `FAILED` pipeline state — connection, SSL, or schema problem
- Any `WARNING` or `ERROR` that isn't expected noise
- End-state read-back shows wrong or empty value even though the log said it succeeded

**Example — LFC bench:**

End state for iteration 1: **Sheet cell E18 contains a positive integer (duration in seconds).**

```bash
# WRONG: background immediately after launching
nohup bash bench-run-sqlserver.sh > /tmp/bench.log 2>&1 &
# failures repeat silently; sheet stays empty

# RIGHT: run first iteration inline, define and verify the end state
SOURCE_TABLE=intpk_1m . ./databricks-3-setup-uc-lakeflow.sh vm-sqlserver public qbc
. ./databricks-4-bench-run.sh 1000000
# Watch for: "COMPLETED" in log AND "updated: Sheet1!E18:G18" in log

# Then verify the END STATE — read the cell back:
SS_TOKEN=$(gcloud auth application-default print-access-token)
curl -s -H "Authorization: Bearer $SS_TOKEN" \
  "https://sheets.googleapis.com/v4/spreadsheets/17ODQ.../values/Sheet1!E18" \
  | jq -r '.values[0][0]'
# Expected: a positive integer like "6"
# If empty or missing → end state not reached → stop and fix before backgrounding

# Only after end state is confirmed: background the rest
nohup bash bench-run-sqlserver.sh > /tmp/bench.log 2>&1 &
```

**Example — bulk data load:**

End state: **`MAX(pk) = 1,000,000` in the target table.**

```bash
# After watching the first batch complete, verify:
mysql -u dba_user -p"$PW" -e "SELECT MAX(pk) FROM intpk_1m;" robert_lee_png_db
# Expected: 1000000
# If 0 or NULL → rows were not written → stop and diagnose before continuing
```

## Anti-patterns to avoid

- **Don't assume state** — never skip Phase 0 because "it probably worked last time".
- **Don't execute during planning** — only read/inspect; no writes until plan is approved.
- **Don't hide ⚠ gaps** — if a pattern is untested, say so even if you're confident it works.
- **Don't pick design defaults silently** — always surface ambiguous choices to the user.
- **Don't skip "What I cannot do"** — if anything requires user input, it must appear in that table.
- **Don't background without monitoring first run** — the first run after any change must complete successfully before waiting for subsequent iterations.
