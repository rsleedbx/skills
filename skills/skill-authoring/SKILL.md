---
name: skill-authoring
description: >-
  Personal addenda to Cursor's built-in create-skill guidance — additional conventions
  learned from authoring and maintaining skills over time. Use when creating, editing,
  or reviewing any SKILL.md file, alongside the built-in create-skill skill.
---

# Skill Authoring — Personal Conventions

These extend the built-in create-skill guidance. Apply both together.

## Descriptions must be implementation-agnostic

The description is the agent's discovery mechanism. It should describe **capabilities**, not the specific use case that motivated the skill.

**Bad** — encodes implementation details that limit reuse:
```yaml
description: >-
  Compare MySQL public vs PNG routing using lfc_user_public / lfc_user_png users
  and databricks-run-with-byte-capture.sh with -public / -png connection suffixes.
```

**Good** — describes the capability generically:
```yaml
description: >-
  Compare pipeline configurations (network path, DB engine, compute tier, etc.)
  by creating separate source users, UC connections, and pipelines per test dimension
  so metrics never interleave and auto-detection works from connection name suffixes.
```

**Rule**: if the description would break or mislead when the skill is applied to a different project, database, or team, it is too specific. Replace proper nouns and tool-specific names with the role they play.

## Skills grow — check size periodically and split proactively

Skills accumulate content across many conversations. A skill that starts at 100 lines can silently grow past 1,000 lines before anyone notices.

**Trigger a size check and refactor when**:
- A new section is added and the file feels long
- The skill has not been reviewed in several sessions
- An agent reads the skill and seems to miss earlier sections (context window pressure)

**Check**:
```bash
wc -l ~/.cursor/skills/*/SKILL.md
```

Any file over 400 lines is a candidate for splitting. Over 500 lines requires splitting per the built-in guideline.

**Split strategy** (progressive disclosure):
1. Keep decision-level content in `SKILL.md`: rules, tables, short code snippets, pointers
2. Move how-to detail to named reference files: `byte-metrics.md`, `system-tables.md`, etc.
3. Add a one-line pointer at the end of each abbreviated section: "For detail, see [reference.md](reference.md)"
4. Name reference files after the section they replace — not generic names like `reference.md`

**Measure success**: total line count across all files should decrease vs the monolithic version (progressive disclosure eliminates duplication, not just moves it).

## Reference file naming

Use the section heading as the filename, lowercased and hyphenated:

| Section heading | Reference file |
|----------------|----------------|
| Byte Metrics — MySQL Source Side | `byte-metrics.md` |
| Compute Resource Metrics — System Tables | `system-tables.md` |
| Notebook Pattern for Pipeline Analysis | `notebook-analysis.md` |
| Latency Triage | `latency-triage.md` |
| Comparison Testing | `comparison-testing.md` |

Avoid generic names (`reference.md`, `details.md`) — they don't tell the agent what they contain.

## Skill taxonomy — five strategies

Use the strategy that best matches *what the skill is fundamentally about*. A skill may mix strategies at the edges, but it should have one dominant axis.

| # | Strategy | Axis | Named after | Use when |
|---|----------|------|-------------|----------|
| 1 | **Product/platform-centric** | a specific product's API, CLI, or config quirks | `<cloud>-<service>` or `<platform>-<feature>` | The product has non-obvious auth, syntax, or edge cases. Answers "how do I use *this thing*." |
| 2 | **Engine/technology-centric** | a technology's mechanics, independent of where it runs | `<domain>-<engine>` (e.g. `database-mysql`) | The same engine runs in multiple environments and the SQL/config is identical everywhere. Answers "how does *this engine* work." |
| 3 | **Cross-cutting pattern** | a behavioral convention applied everywhere | `<domain>-<convention>` (e.g. `shell-cmd-wrapper`) | A pattern (error handling, secret masking, doc style) is not tied to any single product. Answers "how should I *always* do X." |
| 4 | **Problem/solution-centric** | a recurring problem with a known solution | `<noun>-<action>` (e.g. `firewall-ip-discovery`) | The same problem recurs across projects/clouds/engines. Answers "how do I solve *this problem* regardless of product." |
| 5 | **Workflow/process-centric** | an end-to-end sequence spanning multiple products | `<noun>-<verb>-workflow` or `<process>-end-to-end` | The *ordering* of steps is the knowledge; individual steps delegate to product/engine skills. Answers "how do I *accomplish this goal* end to end." |

**Composability rule**: Product skills and engine skills overlap on the same subject (e.g., MySQL appears in `azure-flexible-server-mysql` AND `database-mysql`). Resolve by split-and-cross-reference: the cloud provisioning skill handles cloud-specific steps, then says "See `database-mysql` for grants/binlog." Never duplicate the SQL content.

**Workflow skills degrade faster** than other strategies — they chain across APIs that evolve independently. Keep them as high-level checklists and let them delegate details to product/engine skills. Review any workflow skill when a major product API version changes.

## Description anti-patterns

Beyond the implementation-agnostic rule above, watch for these common mistakes:

**"for [specific project]" in descriptions** — `"for LakeFlow Connect testing"` or `"for LFC ingestion"` incorrectly implies the skill only applies to that project. Replace with the general use case: `"for database hosting"`, `"for ingestion pipeline setup"`.

**Strategy mismatch in the name** — if a skill is named `product-feature` but the content is really about a cross-cutting problem (e.g., `firewall-ip-discovery` is a problem-centric skill, not a product skill), the name is fine — but the description should not say "Use when working with [product]." It should describe the problem.

**Empty descriptions** — `description: >-` with no content means the agent never auto-selects the skill. Always fill in the description; it is the agent's only discovery signal.

**Duplicate coverage** — two skills covering the same subject from slightly different angles (e.g., `databricks-connect` and `python-databricks-connect`) should be merged unless they truly serve different triggering conditions. Check by asking: "would I read both before starting a task?" If yes, merge them.

## Commit before each skill change

Before editing any skill file, check for unstaged changes and commit them first. This keeps skill edits in their own commits, making it easy to revert or review individual changes.

```bash
# Check for unstaged changes in the skills directory
git -C ~/.cursor status --short

# If there are changes, commit them before proceeding
git -C ~/.cursor add -A
git -C ~/.cursor commit -m "skill: <describe what changed>"
```

**When to apply this rule:**
- Before creating a new skill
- Before editing an existing SKILL.md or reference file
- Before splitting, merging, or renaming skills
- Before updating a description

**One logical change per commit** — do not bundle a description fix with a content split in the same commit. If multiple skills need changes, commit each one separately so each commit is independently revertable.

## Periodic audit checklist

Run this to catch size issues and empty descriptions in one pass:

```bash
# Line counts — anything over 400 is a split candidate
wc -l ~/.cursor/skills/*/SKILL.md | sort -n

# Truly empty descriptions (description key with no value on the same or next line)
# Note: "description: >-" alone is valid multi-line YAML; check the line after it instead
awk '/^description:/{if ($0 !~ /[^ \t].*[^ \t]/ && $0 ~ /^description: *$/){print FILENAME}} FNR==1{FILENAME=ARGV[ARGIND]}' ~/.cursor/skills/*/SKILL.md
# Simpler: manually spot-check any SKILL.md under 20 lines — those are likely stubs
awk 'END{if(NR<20) print FILENAME}' ~/.cursor/skills/*/SKILL.md
```

Also check for:
- Skills whose name includes a specific project name (rename to general capability)
- Skills whose description says "for [project]" (generalize per description rules above)
- Two skills covering the same engine/product from nearly-identical angles (merge candidates)
