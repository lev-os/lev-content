---
id: spec-ubs-sdlc-integration
title: UBS Lev SDLC Integration
status: draft
created: 2026-02-11
updated: 2026-02-11
origin: cursor-plan
---

# UBS Integration into Lev SDLC

## Decisions

- **Skill-first**: Create the UBS skill and validator now. Git hook wiring deferred.
- **agent-lease stays dead config for now**: `.agent-lease.json` gets the UBS runner added, but hooks won't fire until a future task wires agent-lease into `.git/hooks/`.
- **agent-lease becomes a workspace member**: Add `community/agent-lease` to `pnpm-workspace.yaml` so it gets a bin link and can be invoked via `pnpm exec agent-lease`.

## agent-lease Status (as investigated)

- Binary works: `node community/agent-lease/bin/agent-lease.js status` runs correctly
- `.agent-lease.json` is valid but nothing reads it (no git hook integration)
- Not in pnpm workspace; has its own npm `package-lock.json`
- Validator scripts all exist and pass (with minor grep regex warnings)
- `lev-code-review.sh` silently fails on large diffs ("Argument list too long" with claude CLI)

---

## 1. Create UBS Skill

**Location:** `~/.agents/skills/ubs-bug-scan/`

**Structure:**

```
~/.agents/skills/ubs-bug-scan/
  SKILL.md
  references/
    ubs-quickref.md
```

**SKILL.md** will follow the standard skill format with:

- **Frontmatter:** name, description, triggers, related_skills, allowed_tools
- **Triggers:** "ubs", "bug scan", "static analysis", "code review bugs", "scan for bugs", "ultimate bug scanner", "catch bugs"
- **Related skills:** pr-review, security-best-practices, clean-code
- **Workflow:**
  1. Check `command -v ubs`; if missing, emit install one-liner and stop.
  2. Determine scope: staged files (`ubs --staged`), changed files (`ubs $(git diff --name-only)`), specific files, or full project (`ubs .`).
  3. Run with `--format=json` for agent parsing or `--format=toon` for token efficiency.
  4. Parse output: critical = must fix, warning = should fix, info = optional.
  5. For each finding: navigate to `file:line`, verify not false positive, fix root cause, re-run `ubs <file>`.
- **Golden rule:** `ubs <changed-files>` before every commit. Exit 0 = safe. Exit >0 = fix and re-run.
- **Speed tip:** Scope to changed files (`ubs src/file.ts` <1s) instead of full project (`ubs .` ~30s).

**references/ubs-quickref.md:** Condensed from UBS README "Agent Quickstart" section -- commands, output parsing, severity guide, anti-patterns.

---

## 2. Create UBS Validator

**Location:** `[.lev/validators/ubs-scan.sh](.lev/validators/ubs-scan.sh)`

**Contract:** Exit 0 = pass, 1 = fail. Same pattern as existing validators in `.lev/validators/`.

```bash
#!/bin/bash
# ubs-scan.sh - Ultimate Bug Scanner validation gate
# Exit 0 = pass, Exit 1 = critical issues found
set -e

# Skip if explicitly disabled
if [ -n "$UBS_SKIP" ]; then
  echo "[ubs] Skipped (UBS_SKIP set)"
  exit 0
fi

# Check ubs is installed
if ! command -v ubs &>/dev/null; then
  echo "[ubs] Not installed. Install: brew install dicklesworthstone/tap/ubs"
  if [ -n "$UBS_STRICT_MISSING" ]; then exit 1; fi
  exit 0
fi

# Scan staged files only (fast, scoped)
ubs --staged --ci 2>&1 | tail -20
EXIT_CODE=${PIPESTATUS[0]}

if [ "$EXIT_CODE" -eq 0 ]; then
  echo "[ubs] No critical issues found"
else
  echo "[ubs] Critical issues detected. Fix before committing."
fi

exit "$EXIT_CODE"
```

---

## 3. Register in Agent-Lease Config

**File:** `[.agent-lease.json](.agent-lease.json)`

Add after the existing `namespace` runner:

```json
{
  "name": "ubs-scan",
  "command": ".lev/validators/ubs-scan.sh",
  "on": "commit"
}
```

Note: This is prep config. It won't fire until agent-lease is wired into git hooks (deferred).

---

## 4. Add agent-lease to pnpm Workspace

**File:** `[pnpm-workspace.yaml](pnpm-workspace.yaml)`

Add under the existing sections:

```yaml
# Community packages (submodules)
- 'community/agent-lease'
```

Then run `pnpm install` to link the bin entry. After this, `pnpm exec agent-lease status` will work from the repo root.

Note: The existing `package-lock.json` inside `community/agent-lease/` should be removed or gitignored once pnpm manages it.

---

## Files to Create/Modify

| Action | Path                                                       |
| ------ | ---------------------------------------------------------- |
| Create | `~/.agents/skills/ubs-bug-scan/SKILL.md`                   |
| Create | `~/.agents/skills/ubs-bug-scan/references/ubs-quickref.md` |
| Create | `.lev/validators/ubs-scan.sh`                              |
| Edit   | `.agent-lease.json` (add ubs-scan runner)                  |
| Edit   | `pnpm-workspace.yaml` (add community/agent-lease)          |

---

## Future Work (not in scope)

- Wire agent-lease into `.git/hooks/pre-commit` and `.git/hooks/pre-push`
- Fix grep regex warnings in `security-scan.sh` and `xdg-compliance.sh`
- Fix `lev-code-review.sh` "Argument list too long" on large diffs (use temp file + truncation)
- Add UBS step to `pr-review` skill
- Create `.ubsignore` at repo root
