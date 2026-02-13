# Deterministic Deploy Adapter Conformance Matrix

Date: 2026-02-10

## Adapter Matrix

| Provider | Adapter file | Contract methods | Deterministic IDs | Simulated apply | Rollback action path | Tests |
|---|---|---|---|---|---|---|
| docker | `plugins/platforms/src/adapters/docker.ts` | `plan/apply/verify/rollback` | Yes | Yes | Pull -> stop/remove -> run target revision | `plugins/platforms/tests/adapters/docker.test.ts` |
| coolify | `plugins/platforms/src/adapters/coolify.ts` | `plan/apply/verify/rollback` | Yes | Yes | `coolify rollback` + redeploy target revision | `plugins/platforms/tests/adapters/coolify.test.ts` |
| dokku | `plugins/platforms/src/adapters/dokku.ts` | `plan/apply/verify/rollback` | Yes | Yes | App rollback + image release | `plugins/platforms/tests/adapters/dokku.test.ts` |
| heroku | `plugins/platforms/src/adapters/heroku.ts` | `plan/apply/verify/rollback` | Yes | Yes | Release rollback + health verification | `plugins/platforms/tests/adapters/heroku.test.ts` |

## Provider Alias Normalization

`levd` normalizes provider aliases before adapter resolution.

| Alias | Canonical provider |
|---|---|
| `awd` | `aws` |
| `hertzel` | `hetzner` |

Project and environment aliases can be supplied through deploy config:

- Project/global config section: `deploy.aliases`
- Environment override: `LEV_DEPLOY_ALIASES` (JSON object)

## Deterministic Triads

- Triad A (clean path): `plan` -> `deploy` succeeds with deterministic plan and healthy verify.
- Triad B (idempotent rerun): repeated `deploy --plan <id>` yields same release id and command set.
- Triad C (override/no-sync): config overrides + alias mapping + dry-run (`--execute` omitted) path is stable.

Primary test file: `tests/integration/levd-triad.test.ts`.

Unified local test command:

`pnpm run test:deploy-runtime`

## Rollback Runbook Checks

1. Confirm target release state with `levd status --release <id> --json`.

2. Execute rollback with provider auto-detection or explicit provider:
   - `levd rollback --release <id>`
   - `levd rollback --release <id> --provider <provider>`

3. Validate rollback artifact and event log:
   - `.lev/agentfs/deploy/status/<release>.json`
   - `.lev/agentfs/deploy/events.jsonl` (event type `deploy.rollback`)

4. Re-run `levd deploy --plan <plan-id>` only after verify checks pass.

5. Run readiness checks for adapter resolution, CLI availability, credential hints, and rollback artifact presence:
   - `levd doctor --provider <provider> --json`
