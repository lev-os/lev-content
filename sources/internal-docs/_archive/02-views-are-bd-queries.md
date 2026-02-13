# Views = BD Queries + Templates

## Core Insight

View is NOT a type. It's a render mechanism.

```
BD Content (source) → Template → Markdown File (view)
```

## BD Already Has This

```bash
# Go templates
bd list --type=epic --format='# {{.Title}}\n{{.Notes}}'

# Obsidian export
bd export --format obsidian

# JSON
bd list --json
bd show <id> --json
```

## View Definition (simple)

A view is:
1. **Query** - `bd list --type=X --status=Y`
2. **Template** - Go template string or file
3. **Output** - file glob

## Current Views (already exist as directories)

| View | Glob | BD Query |
|------|------|----------|
| reports | `~/.lev/pm/reports/*.md` | `--label=report` |
| handoffs | `~/.lev/pm/handoffs/*.md` | `--type=session` |
| proposals | `~/.lev/pm/proposals/*.md` | `--label=proposal` |
| specs | `~/.lev/pm/specs/*.md` | `--label=spec` |
| plans | `~/.lev/pm/plans/*.md` | `--label=plan` |

## Sync Command

```bash
#!/bin/bash
# lev-sync-views

bd list --type=epic --label=spec --limit=0 --json | \
  jq -r '.[] | "# \(.title)\n\nStatus: \(.status)\n\n\(.notes)\n\n---\n"' \
  > ~/.lev/pm/specs/index.md
```

Or just use bd export:
```bash
bd export --type=epic --label=spec --format=obsidian -o ~/.lev/pm/specs/specs.md
```
