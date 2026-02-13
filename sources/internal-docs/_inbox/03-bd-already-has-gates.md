# BD Already Has Gates

## Discovery

We were designing a gate system. BD already has one.

```bash
bd gate --help
```

## Gate Types (Built-in)

| Type | Description |
|------|-------------|
| `human` | Manual close required |
| `timer` | Expires after timeout |
| `gh:run` | Waits for GitHub workflow |
| `gh:pr` | Waits for PR merge |
| `bead` | Waits for cross-rig bead to close |

## Commands

```bash
bd gate list           # Show open gates
bd gate list --all     # Show all gates including closed
bd gate check          # Evaluate all open gates
bd gate resolve <id>   # Close a gate manually
bd gate show <id>      # Show gate details
```

## Integration

Gates are created automatically when a formula step has a gate field.
They block workflow steps until resolved.

## What We DON'T Need to Build

- Gate types (bd has them)
- Gate resolution (bd has it)
- Gate checking (bd has it)

## What We MIGHT Need

- Custom gate types for our validation (spec approval, etc.)
- Integration with IAbstractHooks to emit events when gates resolve
- BDD-style gate definitions that map to bd gates
