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

## Canary principle — parallel, fail-fast, validate the script first

These three ideas are one unified rule, not three separate ones:

**The canary run's purpose is to validate the script chain so that subsequent runs are reliable — not to measure timing.**

A 100K canary run that succeeds proves:
1. The source DB has the data and the LFC user can read it
2. The pipeline was created correctly (right connection, schema, table name)
3. The pipeline reached COMPLETED (not stuck or FAILED)
4. The actual rows in the Delta table match what the source had
5. The sheet write found the correct row and wrote the result
6. The script can be re-run reliably for larger tables

Once the canary passes, every larger run (1M, 10M, 100M, 1B) is just the same chain with more data. Timing from larger runs is the real benchmark — the 100K timing is irrelevant.

**If the canary fails, stop that DB's sequence.** Fix the failure before running 1M. The same failure will repeat at every size.

### Step 1 — Launch all canaries in parallel immediately

Do not wait for DB A to finish before starting DB B. Launch one canary per pipeline simultaneously the moment infrastructure is ready:

```bash
# Write each canary to its own script file (do NOT use nohup bash -c '...' with single
# quotes for multi-line scripts — nested quotes break bash parsing)
for spec in azure-mysql:png azure-postgres:public azure-postgres:png \
            azure-sqlserver:public azure-sqlserver:png \
            vm-mysql:public vm-mysql:png \
            vm-postgres:public vm-postgres:png \
            vm-sqlserver:public vm-sqlserver:png; do
  db="${spec%%:*}"; routing="${spec##*:}"
  cat > "/tmp/canary-${db}-${routing}.sh" << SCRIPT
#!/usr/bin/env bash
set -uo pipefail
cd /path/to/scripts
export WORKSPACE_PROFILE=... PNG_SECRET_SCOPE=... DATABRICKS_TOKEN=...
export SOURCE_TABLE=intpk_100k
source ./databricks-3-setup-uc-lakeflow.sh ${db} ${routing} qbc
source ./databricks-4-bench-run.sh 100000 "\$SS_ID"
SCRIPT
  nohup bash "/tmp/canary-${db}-${routing}.sh" > "/tmp/canary-${db}-${routing}.log" 2>&1 &
  echo "$db $routing PID=$!"
done
```

**Correct parallel layout:**
```
DB A (postgres public):   100K ─── 1M ─── 10M ─── 100M ─── 1B
DB B (sqlserver public):  100K ─── 1M ─── 10M ─── 100M ─── 1B
DB C (sqlserver png):     100K ─── 1M ─── 10M ─── 100M
DB D (mysql public):      (blocked: active update) ─── 100K ─── ...
DB E (vm-mysql public):   100K ─── ...
...all 12 at once
```

### Step 2 — Check all canary logs simultaneously

```bash
for log in /tmp/canary-*.log /tmp/bench-*.log; do
  label=$(basename "$log" .log)
  last=$(grep -E "COMPLETED|FAILED|ERROR|actual rows:|updated:|WARNING" "$log" 2>/dev/null | tail -1)
  printf "%-40s  %s\n" "$label" "${last:-<running...>}"
done
```

### Step 3 — Stop any DB whose canary failed; scale up the rest

- **COMPLETED + actual_rows > 0 + sheet written**: canary passed → launch full benchmark script (100K → 1M → 10M → 100M → 1B)
- **actual_rows = 0**: source empty or connection broken → fix before scaling
- **FAILED**: connection, SSL, or table issue → fix, re-run canary, only then scale
- **WARNING: no matching row**: sheet column A mismatch → fix sheet before continuing

### Still serialize within each DB

A single pipeline cannot have two active updates simultaneously. Run sizes in sequence: 100K → 1M → 10M → 100M → 1B for each DB. Parallelize across DBs, not within.

## Eliminate blockers by restructuring the test, not by fixing the blocker

When a pre-condition appears required but adds significant cost, ask first whether the test can be restructured to make the pre-condition unnecessary.

**General principle:** Before investing effort to fix a blocker, check whether changing _how_ you run the test removes the need for that blocker entirely. This is faster and produces a cleaner test design.

**Signs that restructuring is the right move:**
- The fix is orthogonal to what you're actually measuring (e.g., adding an index to speed up polling when you're benchmarking snapshots, not polling)
- The fix would permanently change the environment in ways that affect other tests
- The fix takes longer to implement than redesigning the test

**When a specific product blocks you:** consult the product's skill for the canonical workaround. The solution is product-specific knowledge, not plan-building knowledge. Record the lesson in that product skill, not here.

## Plan directory structure

Each plan lives in its own subdirectory under `docs/`. Files within the directory use short names — the directory provides the namespace.

```
docs/
└── <plan-name>/
    ├── plan.md           ← goal, key facts, links (≤ 30 lines)
    ├── tracker.md        ← per-item status, active processes, resume commands
    ├── chronology.md     ← index: one row per run with ✓/✗/⚠ and link to detail
    ├── chronology-1.md   ← detail for run 1 (immutable once closed)
    ├── chronology-2.md   ← detail for run 2
    └── ...
```

### File responsibilities

**`plan.md`** — static reference: goal, key facts, links to tracker and chronology, quick status block.

**`tracker.md`** — current system state: per-item status tables, data load status, active PIDs, resume commands.

**`chronology.md`** — append-only index: one row per run, `| [N](chronology-N.md) | date | title | ✓/✗/⚠ result |`.

**`chronology-N.md`** — immutable run record: actions, results, mistakes, fix applied, next step → link to N+1.

### Rules

- **Never put detail in `chronology.md`** — one line per run only. Detail goes in `chronology-N.md`.
- **Never edit a closed `chronology-N.md`** — corrections go in the next entry.
- **New session = new chronology entry** — even if you only verified state and did nothing.
- **When closing `chronology-N.md`, immediately: (1) create `chronology-(N+1).md` as a blank stub, (2) add run N's row to `chronology.md`.** The link `→ [N+1](chronology-N+1.md)` must point to a file that exists — never leave it dangling.
- **`tracker.md` reflects now; `chronology-N.md` records what happened.** Don't merge them.

### Blank template for a new run

```markdown
# Run N — [title]

**Date**: _fill in_
**Back to index**: [chronology.md](chronology.md)

---

## Actions
_What was launched or changed this session?_

## Results
| item | outcome | artifact |
|------|---------|----------|
| | | |

## Mistakes
_What went wrong and why?_

## Fix applied
_What was changed in response?_

## Next step decided
_What should run N+1 do?_

→ [N+1](chronology-N+1.md)
```

## Write one generic script, not many custom scripts

Every time a new variant appears — a retry, a subset re-run, a canary pass — the instinct is to create a new file. This generates a proliferation of near-identical scripts:

```
bench-run-mysql.sh                 # original
bench-run-postgres.sh              # original
bench-run-postgres-public-rerun.sh # retry after failures
bench-run-postgres-public-10m-to-1b.sh  # resume from 10M
bench-run-sqlserver-1m-rerun.sh    # single row fix
bench-run-100k-canaries.sh         # canary pass
bench-run-vm-mysql-retry.sh        # retry variant
bench-run-vm-postgres-retry.sh     # retry variant
bench-run-vm-sqlserver-retry.sh    # retry variant
bench-run-all.sh                   # orchestrator
```

Each file is a one-off. None can be reused. The next variant creates another file. **This is the wrong pattern.**

### The correct pattern: one generic script with arguments

Extract the invariant logic into a single parameterized script. All variants become invocations:

```bash
# One generic script: bench-run.sh <db> <routing> <table> <rows>
bash bench-run.sh azure-postgres public intpk_1m   1000000
bash bench-run.sh azure-postgres public intpk_10m  10000000
bash bench-run.sh vm-sqlserver   public intpk_1m   1000000   # retry: same invocation
bash bench-run.sh azure-mysql    public intpk_100k 100000     # canary: same script
```

**Rules:**
- Before creating a new script file, ask: "can I call the existing script with different arguments?"
- If the answer is yes — do that. Do not create a new file.
- If a genuine new capability is needed (different pre/post steps), add an optional argument or flag to the existing script rather than copying it.
- Orchestrators (run-all, parallel launchers) are legitimate — they call the generic script in parallel/sequence. They do not duplicate its logic.

### Identifying the generic interface

For a benchmark workflow, the generic parameters are always:
```
<db-key>  <routing>  <table>  <row-count>  [<spreadsheet-id>]
```
Everything else (connection name, pipeline name, UC catalog/schema, polling logic, sheet write) is derived from those inputs by the existing setup/bench scripts.

**Before writing any new file:** map the intended action onto these parameters. If it fits — use the existing script.

## Verify idempotency before scaling up

**A script is not ready to run unsupervised until it has been verified idempotent: run it at least 3 times with the same inputs and confirm the end state is identical each time.**

### What idempotency means for a bench script

| Run | Expected behavior |
|-----|-------------------|
| 1st | Fresh snapshot → timing captured → sheet written |
| 2nd | Same — `full_refresh_selection` resets checkpoint → another snapshot → sheet overwritten with same result ±network variance |
| 3rd | Same again — no drift, no accumulation, no error |

A script is **not** idempotent if:
- Run 2 fails because run 1 left the pipeline in a bad state
- Run 2 writes 0 rows because the cursor watermark was not reset
- Run 2 writes to a different sheet row than run 1 (lookup broke)
- Run 3 reports a different row count than runs 1 and 2 (source changed unexpectedly)

### How to verify

```bash
# Run the script three times; compare actual_rows and sheet row on each pass
for i in 1 2 3; do
  echo "=== Run $i ==="
  bash bench-run.sh azure-postgres public intpk_100k 100000
  # Read sheet cell back: expected, timing, actual all populated
done
```

After all three runs, read the sheet. The `actual rows` value should be the same across all three. The `duration_s` may vary by ±10–30% — that is expected noise, not a bug.

**If run 2 or run 3 differs structurally (0 actual rows, FAILED state, wrong row written), stop and fix before scaling to larger tables.** The same idempotency bug will silently corrupt every subsequent run.

### Apply idempotency checks before parallel scale-out

The canary principle says "launch all DBs in parallel once the canary passes." The idempotency principle says "the canary must pass at least 3 times before you call it 'passed'."

Correct sequence:
```
1. Run 100K canary once → verify D/E/H all populated correctly
2. Run 100K canary again (same invocation) → same result
3. Run 100K canary third time → same result
4. Only then: launch 1M → 10M → 100M → 1B for that DB
```

## Two-actor execution model — Reviewer and Implementer

Long-running benchmarks need two distinct mental roles operating simultaneously:

### Reviewer (the "user" perspective)
The Reviewer's job is to continuously answer: **"Does what is actually running match what should be running?"**

**Check at every opportunity:**
- What pipeline updates are RUNNING right now? (`/api/2.0/pipelines/{id}/updates?max_results=1`)
- Do the running tables/sizes match the current goal state (e.g., "we are doing 100K verification")?
- Are any background processes from a previous session still alive and consuming pipeline capacity?
- Does the sheet reflect what the pipelines actually produced?

**The Reviewer blocks forward progress when:**
- An unrelated run occupies a needed pipeline (e.g., a 1B run blocks a 1M re-run)
- A background process is looping on a CANCELED update instead of the intended one
- The sheet row shows data from a stale/wrong pipeline

**Concrete Reviewer checks before launching any re-run:**
```bash
# 1. Check for running processes from prior sessions
ps aux | grep "bench-run\|databricks"

# 2. Check pipeline idle state for each pipeline you intend to use
databricks api get /api/2.0/pipelines/{PIPELINE_ID}/updates?max_results=1 \
  --profile "$WORKSPACE_PROFILE" | jq '.updates[0] | {state, update_id}'

# 3. Read sheet to confirm goal state matches actual state
# Sheet rows you expect to be empty should be empty; rows with stale data should be identified
```

### Implementer (the agent)
The Implementer executes steps but **is expected to make mistakes**, especially:
- Attaching to the wrong active update when launching re-runs
- Not checking whether a pipeline is clear before starting
- Not realizing an old script's background process is still running
- Missing that the current goal is "100K verification" while accidentally launching 1B runs

**Implementer discipline:**
1. Before any `bench-run.sh` invocation, confirm the pipeline is IDLE (not RUNNING/INITIALIZING)
2. Before launching parallel re-runs, check which processes are already running (`ps aux`)
3. Keep the current goal visible — write it as a `TODO` comment at the top of every session
4. When a background log shows unexpected behavior, stop and report to Reviewer before continuing

### Goal-state tracking
At the start of every session, write the current goal explicitly:

```
CURRENT GOAL: Verify all 12 × 100K rows in sheet have correct pipeline_id,
              duration_s, and actual_rows before proceeding to 1M.
```

Check every running and queued action against this goal. If an action (e.g., a 1B re-run) doesn't serve the current goal, stop it.

## Fail-fast: never advance to a larger workload until the smaller one is fully verified

This is the single most important sequencing rule.

**Rule: ALL rows at size N must be correct before ANY row at size N+1 is started.**

"Correct" means every row at size N has:
- Column E (duration_s): non-empty
- Column F (pipeline_id): from the correct catalog/pipeline
- Column G (update_id): non-empty
- Column H (actual_rows): populated OR confirmed as a known-fast-run gap

If even one row at size N is empty, partial, or from a wrong pipeline, stop. Fix size N first.

**Why this matters:**
- A script bug at 100K will silently corrupt every 1M → 10M → 100M → 1B run after it
- Debugging a corrupt 100M run is 1000× harder than debugging the same bug at 100K
- If you fix the bug at 100K first, all larger sizes get the correct version for free

**Concrete gate check before advancing from 100K to 1M:**
```bash
# Read sheet, show only 100K rows — every row must have columns E, F, G non-empty
curl -s ... "Sheet1!A1:H61" | jq -r '
  .values | to_entries[]
  | select(.value[3] == "100,000")
  | "\(.key+1)\t\(.value[4] // "MISSING-E")\t\(.value[5] // "MISSING-F")\t\(.value[7] // "no-H")"'
```

If any row shows `MISSING-E` or `MISSING-F`, that row must be re-run before proceeding.

**The ordering error to avoid:**
```
❌ Wrong: row 7 (100K) empty → launch 1M re-runs anyway to fix stale data
✅ Right:  row 7 (100K) empty → fix row 7 (100K) first → verify → then launch 1M
```

Even if the 1M re-runs are motivated by a different issue (wrong pipeline catalog), do not launch them while any 100K row is incomplete. The fix cost at 100K is seconds; the debugging cost at 1M+ is minutes to hours.

## Anti-patterns to avoid

- **Don't assume state** — never skip Phase 0 because "it probably worked last time".
- **Don't execute during planning** — only read/inspect; no writes until plan is approved.
- **Don't hide ⚠ gaps** — if a pattern is untested, say so even if you're confident it works.
- **Don't pick design defaults silently** — always surface ambiguous choices to the user.
- **Don't skip "What I cannot do"** — if anything requires user input, it must appear in that table.
- **Don't background without monitoring first run** — the first run after any change must complete successfully before waiting for subsequent iterations.
- **Don't leave `chronology-N.md` pointing to a missing `chronology-(N+1).md`** — always create the stub immediately when closing a run entry.
- **Don't create a new script file for every variant** — add an argument to the existing generic script instead.
- **Don't call a canary "passed" after one run** — verify idempotency (3 identical runs) before scaling up.
- **Don't advance to a larger workload while any smaller workload row is incomplete** — fix 100K before touching 1M, fix 1M before touching 10M. No exceptions.
- **Don't let test objects accumulate in a pipeline** — configure `objects[]` with exactly what the test requires (one for a single-table benchmark, two for a two-table test). Appending instead of replacing leaves stale objects that pollute every subsequent run. Always replace, never append.
