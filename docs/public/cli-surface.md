---
title: Lev CLI Surface
description: User-facing command model and practical command map.
order: 20
section: cli
---

# Lev CLI Surface

Lev exposes a command-router model: discover, route, execute.

## Core Pattern

```bash
lev <command> [args]
```

## Common Commands

| Goal | Command |
|---|---|
| Show command surface | `lev help` |
| Gather context | `lev get "<query>"` |
| Legacy search path | `lev find "<query>"` |
| Semantic index search | `lev search "<query>"` |
| Route lifecycle work | `lev work auto "<task>"` |
| Run workflow | `lev workflow <...>` |
| Validate state | `lev validate <...>` |

## Alias Notes

- `ask` and `wiz` route to wizard behavior where available.
- `check` routes to validate.
- `go` routes to work.

## Public Docs Intent

This page documents **user-facing behavior**. Internal command implementation details
and subsystem design are intentionally kept in internal docs.
