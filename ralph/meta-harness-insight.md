---
title: "The Meta-Harness Insight"
date: 2026-04-10
type: insight
source: session-voice-memo
tags: [constraint-engineering, meta-harness, recursive, thesis]
---

# The Meta-Harness Insight

> A constraint is a flowmind definition. A harness is the tooling to run it.
> Leviathan is a meta-harness — it defines and runs its own constraint harnesses.

## The Recursive Structure

```
1. ALIGN    — How aligned am I with the user? Know their intent.
2. PLAN     — Make a plan (SDLC or otherwise). Present it.
3. ACCEPT   — User accepts. Now we need adversarial flowminds.
4. SCAFFOLD — Write the flowminds + constraints. Dry run.
5. PROMOTE  — If success, promote code. Kick off next safe slice.
6. REPEAT   — After each set of rounds, offer the same experience:
              expand the harness, write new flowminds, tighten constraints.
```

The more the user front-loads constraints and harness definitions,
the more we can schedule autonomous loops to manifest the work.

## The 90/10 Split

- **90% of cases**: Lev IS the harness. Define the DNA, write the flowmind,
  Lev executes and validates against its own constraint surface.
- **10% of cases**: Custom harness needed (AppTestr for iOS, NAAC for ATC).
  But you STILL define the DNA first. The DNA cascade builds the code
  deterministically around the behavioral contract.

## What This Means for /levloop

The loop's onboarding phase IS alignment:
- Capture user intent → tick types
- Write adversarial flowminds → validation between ticks
- Dry run 1-3 items → test the harness
- If harness works → schedule the loop
- Each round expands the harness with new constraints discovered

## Recommended Workflows → Tooling

We're structuring these as recommended workflows first.
Then we build tooling to support the structure.
The structure IS the product. The tooling serves it.
