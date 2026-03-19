# cursor-config

Shared Cursor AI configuration — skills and rules used across multiple repos.

## Usage as a git submodule

Add to a repo:
```bash
git submodule add https://github.com/robert-lee/cursor-config .cursor
git commit -m "Add shared Cursor config as submodule"
```

Clone a repo that already has this submodule:
```bash
git clone --recurse-submodules <repo-url>
# or, if already cloned:
git submodule update --init
```

Pull latest cursor-config updates into a repo:
```bash
git submodule update --remote .cursor
git add .cursor
git commit -m "Update cursor config"
```

## Contents

| Path | Purpose |
|------|---------|
| `skills/python-best-practices/` | Python coding conventions for AI agents |
| `skills/databricks-connect-notebook/` | Create and test notebooks with Databricks Connect |
| `skills/zerobus-ingest/` | ZeroBus Protobuf ingest patterns |
| `skills/docs-writing-style/` | Documentation writing style guide |

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` in this repo
2. Commit and push
3. In each consuming repo: `git submodule update --remote .cursor && git add .cursor && git commit -m "Add <skill-name> skill"`
