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
