---
title: Getting Started
description: Quick orientation for Lev CLI and the public docs surface.
order: 10
section: start
---

# Getting Started

This public docs surface is intentionally curated.

- Public user docs live in `community/lev-content/docs/public`.
- Internal architecture/spec truth stays in the Leviathan repo `docs/` tree.
- Source material can be ingested into `community/lev-content/sources/*` for curation.

## Fast Start

```bash
# 1) show CLI surface
lev help

# 2) gather context
lev get "flowmind router"

# 3) run lifecycle router
lev work auto "set up docs scaffolding"
```

## Docs Boundaries

- Public docs: this site (`/docs/*`)
- Internal design/spec docs: not exposed on this site
- Skill runtime instructions: stay in `SKILL.md`
- Long-form user guidance: move into public docs pages
