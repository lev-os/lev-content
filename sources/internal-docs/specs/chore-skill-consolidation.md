---
id: spec-skill-registry-consolidation
title: Skill Registry Consolidation
status: draft
created: 2026-02-11
updated: 2026-02-11
origin: cursor-plan
---

# Skill Registry Consolidation Plan

## Current State Analysis

### Three Fragmented Systems

1. **POC Lookup CLI** ([`workshop/poc/lookup/cli.js`](leviathan/workshop/poc/lookup/cli.js))
   - Hardcoded `SCAN_LOCATIONS` array with 17 glob patterns
   - Local `./index.json` file for embeddings
   - No XDG awareness - uses `~/` and `./` literals

2. **Core Index SDK** ([`core/index/src/index.js`](leviathan/core/index/src/index.js))
   - Vector search via LEANN gRPC on port 50052
   - Backend registry pattern exists but not for skills
   - No skill-specific functionality

3. **Core Build Skills Command** ([`core/build/src/commands/skills.ts`](leviathan/core/build/src/commands/skills.ts))
   - XDG-aware (`getConfigDir()`, `XDG_CONFIG_HOME`)
   - Writes to `~/.config/lev/skills-index.json`
   - Has `sources.yaml` pattern for external skills
   - Imports from `lev-skills-registry` (external package)

4. **Harness Skill Injector** ([`core/harness/src/skill-injector.ts`](leviathan/core/harness/src/skill-injector.ts))
   - Runtime `SkillRegistry` interface with `load()`, `list()`, `findByTrigger()`
   - Single-directory based (`createSkillRegistry(skillsDir?)`)
   - No multi-source support

### The Problem

```
poc/lookup              core/build/skills         harness/skill-injector
    |                        |                           |
    v                        v                           v
SCAN_LOCATIONS          sources.yaml              single skillsDir
(17 hardcoded)          (XDG config)              (no XDG)
    |                        |                           |
    v                        v                           v
./index.json           skills-index.json         runtime cache
(local to POC)         (XDG_CONFIG_HOME)         (in-memory)
```

**Result**: Skills generated in POC never reach `~/.claude/skills` where Claude Code looks.

---

## Proposed Architecture

### Single Source of Truth: XDG Config

Add `skill_sources` to [`~/.config/lev/config.yaml`](~/.config/lev/config.yaml):

```yaml
# ~/.config/lev/config.yaml
skill_sources:
  # Priority order: first match wins
 - type: project
    path: './.lev/skills'
 - type: user
    path: '${XDG_DATA_HOME}/lev/skills'
 - type: claude
    path: '${HOME}/.claude/skills'
 - type: plugins
    path: '${HOME}/.claude/plugins/*/skills'
 - type: external
    ref: 'sources.yaml'
```

### Resolution Chain

```
lev find "research"
     |
     v
SkillResolver.resolve(query)
     |
     v
1. Load skill_sources from config.yaml
2. Expand env vars (XDG_DATA_HOME, HOME)
3. Glob each source path
4. Search index (embeddings + keywords)
5. Return ranked results with source metadata
```

### Unified Index Location

- **Index file**: `~/.config/lev/skills-index.json` (XDG_CONFIG_HOME)
- **Embeddings cache**: `~/.cache/lev/skills-embeddings/` (XDG_CACHE_HOME)
- **Skills data**: `~/.local/share/lev/skills/` (XDG_DATA_HOME)

---

## What Breaks

| Component | Breakage | Fix |

| ----------------------------------- | --------------------------- | ------------------------------------ |

| `poc/lookup/cli.js` | Hardcoded `SCAN_LOCATIONS` | Replace with config loader |

| `poc/lookup/src/scanner.js` | `expandTilde()` ignores XDG | Use `xdg-paths.ts` utilities |

| `core/harness/skill-injector.ts` | Single `skillsDir` param | Accept `SkillResolver` or config |

| `core/build/src/commands/skills.ts` | Already mostly correct | Add `skill_sources` to config schema |

---

## Implementation Steps

### Step 1: Add `getSkillSources()` to xdg-paths.ts

Extend [`core/build/src/utils/xdg-paths.ts`](leviathan/core/build/src/utils/xdg-paths.ts) with:

```typescript
export interface SkillSource {
  type: 'project' | 'user' | 'claude' | 'plugins' | 'external'
  path: string
  priority: number
}

export function getSkillSources(): SkillSource[] {
  // Load from config.yaml or return defaults
}
```

### Step 2: Create SkillResolver in core/harness

New file: `core/harness/src/skill-resolver.ts`

```typescript
export class SkillResolver {
  constructor(private sources: SkillSource[]) {}

  async resolve(query: string): Promise<Skill[]> {
    // Multi-source resolution
  }

  async getAllPaths(): Promise<string[]> {
    // Expand all sources, glob, dedupe
  }
}
```

### Step 3: Update scanner.js to use SkillResolver

Replace hardcoded `SCAN_LOCATIONS` with:

```javascript
import { getSkillSources } from '@leviathan/core-build'

export async function scanAllLocations() {
  const sources = getSkillSources()
  // Use SkillResolver pattern
}
```

### Step 4: Update config.yaml schema

Add to [`core/build/src/sources/schema.ts`](leviathan/core/build/src/sources/schema.ts):

```typescript
skill_sources: {
  type: 'array',
  items: {
    type: 'object',
    properties: {
      type: { enum: ['project', 'user', 'claude', 'plugins', 'external'] },
      path: { type: 'string' },
    }
  }
}
```

### Step 5: Migrate poc/lookup to use core utilities

- Import XDG paths from `@leviathan/core-build`
- Remove hardcoded `SCAN_LOCATIONS`
- Keep embedding/search logic (reusable)

---

## Validation Fitness Functions

1. **Config Compliance**: `lev doctor` verifies `XDG_CONFIG_HOME/lev/config.yaml` exists and parses
2. **Resolution Order**: Test that `./lev/skills/foo` overrides `~/.local/share/lev/skills/foo`
3. **Portability**: Move project folder, verify `lev find` still works
4. **Claude Interop**: Skills in XDG paths appear in Claude Code's skill picker

---

## Risk Mitigation

- **Backward Compatibility**: Keep `~/.claude/skills` in default sources
- **Gradual Migration**: POC lookup can coexist during transition
- **Fallback**: If config.yaml missing, use hardcoded defaults matching current behavior
