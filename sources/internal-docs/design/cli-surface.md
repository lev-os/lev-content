# CLI Surface v1 (Locked Prerelease Contract)

**Status**: LOCKED
**Last Updated**: 2026-02-12
**Audience**: Core maintainers, plugin/runtimes maintainers, prerelease testers
**Scope**: Root CLI contract, `lev exec` task surface, provider token/accounting contract, routing/error contract

## Locked Decisions
1. `exec://` protocol flags are deferred to vNext and are not implemented in v1.
2. Root `lev search`, `lev build`, and `lev status` stay in v1.
3. `lev add` is removed from root; use `lev index add` only.
4. `lev ralph` is removed from root surface (not listed, routed, or documented).
5. `bd` is canonical for beads in this wave (`br` parity is separate work).
6. `lev exec` supports positional task mode and explicit `--prompt`.
7. No compatibility/deprecation shims are added in this lock wave.

## Root CLI Contract (v1)
### Allowed root commands
`lev get`, `lev find`, `lev search`, `lev build`, `lev status`, `lev work`, `lev exec`, `lev review`, `lev poly`, `lev index`

### Removed from root contract
`lev add`, `lev ralph`

### Namespaced path required
`lev index add`

### `lev get` (user-facing definition)
`lev get "<query>"` is the context gather primitive. It runs:
1. local find/search backends (`lev find` path)
2. optional research backend (if configured/available)

Output is a gather artifact in:
`/Users/jean-patricksmith/digital/leviathan/.lev/agentfs/gather/artifacts/<id>.json`

### Explicit non-commands
- `lev sidequest` is **not** a root CLI command.
- `lev harness auto` is **not** a root CLI command.

## `lev exec` Contract (v1)
### Accepted task entry modes
1. `lev exec "<task>"`
2. `lev exec --prompt "<task>"`

`--prompt` takes precedence over positional task when both are supplied.

### Preserved controls
`--epic`, `--status`, `--until`, `--max-iterations`, `--concurrency`

### Context-pressure controls (explicit)
`--context-pressure-threshold=<ratio>`  
`--context-pressure-action=handoff|stop`  
`--context-pressure-guard` / `--no-context-pressure-guard`

### Env mapping
`LEV_CONTEXT_PRESSURE_THRESHOLD` (default `0.5`)  
`LEV_CONTEXT_PRESSURE_ACTION` (default `handoff`)  
`LEV_CONTEXT_PRESSURE_GUARD` (default enabled)

## Provider Token Accounting Contract
Token usage is normalized across harness runners/SDKs to:

`provider`, `model`, `inputTokens`, `outputTokens`, `cachedInputTokens`, `totalTokens`, `costUSD`, `contextWindow`, `usageSource`, `confidence`, `raw`

### Source policy
1. Event/result payload usage fields are authoritative (`usageSource=event|result|model_usage`).
2. Invocation/config-derived values are marked as config-derived (`usageSource=config`).
3. Estimates are explicit only (`usageSource=estimate`, low confidence).
4. No silent synthetic totals without source labeling.

## Routing/Error Contract
### Discovered but unrunnable command
Structured envelope with `error_code: "UNIMPLEMENTED_COMMAND_SURFACE"` and exit code `2`.

### Unknown or removed root command
Structured root-router envelope with exit code `1`:
1. `ROOT_SURFACE_COMMAND_REMOVED` for commands removed from root contract (`add`, `ralph`).
2. `UNKNOWN_COMMAND` for all other unknown root commands.

## Deferred / Non-goals for v1
1. `exec://` resolver flags and protocol runtime in `lev exec`
2. Consumer migration shims/deprecation messaging
3. `br` tracker adapter parity at CLI contract level

## Tracking Lives in Beads
Epic: `lev-besy0`  
Chores: `lev-besy0.1`, `lev-besy0.2`, `lev-besy0.3`, `lev-besy0.4`, `lev-besy0.5`, `lev-besy0.6`, `lev-besy0.7`, `lev-besy0.8`

## Acceptance Gate
1. `pnpm run test:cli-surface` passes.
2. `lev`, `lev help`, `lev --routing` exit `0` and show locked surface.
3. `lev add` fails as removed root command; `lev index add` routes through index namespace.
4. `lev ralph` is not exposed in root help/routing.
5. `lev exec "task"` and `lev exec --prompt "task"` normalize to the same task input.
6. Provider usage mappers emit `usageSource` and `confidence`.
7. Context-pressure threshold crossing is deterministic with configured action (`handoff` or `stop`).
