# BD Fields and Templates

## Available Fields (from bd show --json)

```
.ID
.Title
.Notes (body)
.Status
.Priority
.IssueType
.Owner
.CreatedAt
.CreatedBy
.UpdatedAt
```

## Go Template Examples

```bash
# Simple list
bd list --format='{{.ID}}: {{.Title}}'

# Markdown
bd list --format='# {{.Title}}

{{.Notes}}

---
'

# Checkbox list
bd list --format='- [ ] {{.ID}}: {{.Title}} [P{{.Priority}}]'
```

## Local Type + ADR Template Pattern

```bash
# Register custom types for this repo (once)
bd config set types.custom "spec,decision"

# Decision/ADR issues use custom type from .beads/types/decision.yaml
bd create "Adopt deterministic deploy runtime boundary" \
  --type decision \
  --priority P1

# Validate available types (core + persisted custom types)
bd types --json
```

> Note: Current BD CLI does not expose `--from-template`. Repository templates in
> `.beads/templates/*.yaml` are for local workflow/tooling composition.
> In this repo, `bd create --type decision --dry-run` is accepted even when
> `bd types --json` only shows persisted custom types.

## Export Formats

```bash
bd export --format=jsonl    # JSON Lines (default)
bd export --format=obsidian # Markdown with checkboxes
```

## Useful Filters

```bash
--type=epic|task|bug|feature|chore
--status=open|in_progress|blocked|closed
--priority=0|1|2|3|4 (or P0-P4)
--label=foo
--assignee=bar
--limit=N (default 50, 0=unlimited)
```
