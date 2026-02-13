# Plugin: CMS

> Type-driven filesystem mirroring for beads

## Status

| Attribute | Value |
|-----------|-------|
| Location | `plugins/cms/` |
| Version | 2.0.0 |
| Event Provider | `bd.ts`, `router.js` (via `emit()`) |
| Event Bus | `core/events/src/events/event-bus.ts` |
| Install | `initCmsPlugin()` at startup |

## Purpose

Owns the `mirror_to_filesystem` hook declared in every `.beads/types/*.yaml`. When a bead is created or updated, the CMS plugin reads the type's `file_template`, substitutes variables, and writes a markdown file to the appropriate output directory.

Before this plugin existed, this logic was tangled inside `ideas.js` middleware — tightly coupled, hardcoded to one type, and blocking the operation. Now it's fire-and-forget via events.

## Features

### Current State

| Feature | Status | Notes |
|---------|--------|-------|
| Type-driven templates | ✅ Live | Reads `file_template` from `.beads/types/*.yaml` |
| Fallback mirror | ❌ Removed | **Strict Mode**: Fails if type/template missing |
| Variable substitution | ✅ Live | `{title}`, `{id}`, `{dimensions.layer}`, etc. |
| Deep var resolution | ✅ Live | Dot notation supported for nested properties |
| Slug generation | ✅ Live | `slugify()` for filesystem-safe names |
| Bead validation | ✅ Live | `validateBead()` checks title + type schema required fields |
| Slug collision check | ✅ Live | `checkSlugCollision()` detects existing dirs |
| BD template regen | ✅ Live | Updating a bead re-renders the markdown template |
| BD create events | ✅ Live | Subscribes to `bd:create:complete` |
| BD update events | ✅ Live | Subscribes to `bd:update:complete` |
| Legacy bead events | ✅ Live | Subscribes to `beads:*:create:complete` |
| `index_for_search` | ⏳ Stub | Logs but does not index yet |
| Type cache | ✅ Live | Cached after first load, `clearTypeCache()` for testing |
| Teardown | ✅ Live | `teardown()` unsubscribes all events |

### Ideal State

| Feature | Status | Description |
|---------|--------|-------------|
| All type YAMLs mirrored | ✅ | Every type in `.beads/types/` auto-mirrors on create/update |
| `index_for_search` | 🔲 | Integrate with LEANN/cass for semantic indexing post-create |
| Archive on close | 🔲 | Move markdown to archive dir on `bd:close:complete` |
| Bidirectional sync | 🔲 | Edits to markdown files propagate back to beads |
| `lev install` scaffold | 🔲 | `lev install @lev/cms` copies type YAMLs to `.beads/types/` |
| Hot reload types | 🔲 | Watch `.beads/types/` for changes, reload without restart |
| Type extension | 🔲 | `extends: entity` inheritance resolved at load time |
| Lifecycle guards | 🔲 | Enforce `guards` from type YAML transitions |
| Hook orchestration | 🔲 | Execute all `hooks.post_create` actions, not just mirror |

## Output Directory Mapping

| Bead Type | Output Directory | Env Override |
|-----------|-----------------|--------------|
| idea | `ideas/` | `LEV_PM_IDEAS` |
| proposal | `.lev/pm/proposals/` | `LEV_PM_PROPOSALS` |
| plan | `.lev/pm/plans/` | `LEV_PM_PLANS` |
| spec | `.lev/pm/specs/` | `LEV_PM_SPECS` |
| session | `sessions/` | `LEV_PM_SESSIONS` |
| report | `.lev/pm/reports/` | `LEV_PM_REPORTS` |
| system | `docs/systems/` | `LEV_PM_SYSTEMS` |
| learning | `.lev/pm/learnings/` | `LEV_PM_LEARNINGS` |
| decision | `.lev/pm/decisions/` | `LEV_PM_DECISIONS` |
| (other) | `.lev/pm/misc/` | — |

## Supported Types (from `.beads/types/`)

| Type | Fields (required) | Lifecycle States | `file_template` |
|------|-------------------|------------------|-----------------|
| entity | title | — | ✅ |
| idea | title | ideation → analysis → specification → ... | ✅ |
| proposal | title, intent | draft → submitted → under_review → ... | ✅ |
| plan | title | draft → validated → implementing → ... | ✅ |
| session | semantic_summary | — | ✅ |
| spec | — | draft → review → approved → implementing | ✅ |
| report | title | draft → review → published → archived | ✅ |
| system | title | draft → active → deprecated | ✅ |
| learning | title | draft → published → archived | ✅ |
| decision | — | *(no YAML yet)* | ❌ (Will Fail) |

## Architecture

```
BD operation
  → Zod validate (schema)
  → executeHandler (bd binary)
  → emit('bd:create:complete', { result, ctx })
       │
       └── CMS plugin subscribes
             ├── resolveBeadType(result || ctx)
             ├── loadTypes() → .beads/types/*.yaml
             ├── buildTemplateVars(ctx) (resolves deep props)
             ├── renderTemplate(file_template, vars)
             └── writeFileSync(outputDir/slug.md)
```

## API

```javascript
import { initCmsPlugin } from './plugins/cms/cms-plugin.js'

// At startup
const cms = initCmsPlugin()
// cms.name → 'cms'
// cms.supportedTypes() → ['entity', 'idea', 'proposal', ...]
// cms.teardown() → unsubscribe all

// Direct use (no events)
import { mirrorToFilesystem, validateBead, checkSlugCollision } from './plugins/cms/template-scaffolder.js'

mirrorToFilesystem(bead, 'proposal', '.lev/pm/proposals/')
validateBead({ title: 'Fix build' }, 'idea')
checkSlugCollision('Fix Build', 'ideas/')
```

## Tests

```bash
node --test plugins/cms/cms-plugin.test.js
# 15 tests, 8 suites, 0 failures
```

| Suite | Tests | Coverage |
|-------|-------|----------|
| slugify | 1 | slug generation |
| renderTemplate | 3 | substitution, unresolved vars, null handling |
| buildTemplateVars | 1 | context → vars mapping |
| mirrorToFilesystem | 2 | fallback + type-driven |
| validateBead | 2 | missing title, valid bead |
| checkSlugCollision | 2 | collision detected, no collision |
| Deep Variable Resolution | 3 | nested props, deep nested, missing props |
| Regeneration | 1 | update bead → re-render same file |

## Migration Notes

This plugin replaces the middleware system:

| Before | After |
|--------|-------|
| `core/index/middleware/beads/ideas.js` | **Deleted** |
| `core/polyglot-runners/src/middleware/registry.js` | **Deleted** |
| `runMiddleware('bd:create:before', ctx)` in `bd.ts` | **Removed** |
| `runMiddleware(hookPoint, ctx)` in `router.js` | **Removed** |
| Hardcoded template in middleware | Type YAML `file_template` |
