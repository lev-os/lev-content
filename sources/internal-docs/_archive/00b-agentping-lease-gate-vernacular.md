# AgentPing / AgentLease / AgentGate Vernacular Refinement

> **Source:** `00-vision.md` lines 20-51, 324-353 (voice transcript)
> **Status:** classified → needs alignment with `docs/design/vernacular.md` and `docs/design/graph-primitives.md`
> **Layer:** Site (core model — these are foundational primitives)
> **Overlaps:** `docs/design/vernacular.md` (partially captures Gate), `docs/design/graph-primitives.md` (has Gate, Proposal), `04-agentping-human-loop.md` (existing inbox file)

## Vernacular Resolution

### Resolved Terms

| Term | Technical Name | What It Is | Brand Name |
|------|---------------|-----------|------------|
| **Action Surface** | ActionSurface | Where interaction happens (browser, phone, SMS, stream deck, wall tablet, voice, VR) | AgentPing |
| **Interface** | Interface | The communication channel over an action surface | — |
| **Gate** | Gate | Validation checkpoint — enforces a lease, reminds the agent of constraints | AgentGate |
| **Lease** | Lease | A rule/contract the agent must fulfill before proceeding (e.g., before commit) | AgentLease |

### Key Insight from Transcript
> "Ping is the brand name, but the technical name is Action Surface. Now you have Interface, Gates, and Leases."

> "The only way you enforce a lease is with gates."

> "An action surface is part of the interaction model. Everything is an interface."

### Hierarchy
```
AgentPing (brand) = Action Surface (technical)
  └── Interface (communication channel)
      └── Gate (validation checkpoint)
          └── Lease (rule/contract to fulfill)
```

### Examples from Transcript
- **Git commit gate:** Agent doesn't know it needs a lease to commit → gate reminds it → must fulfill lease to proceed
- **Video editing gate:** Different gate than git — maybe fires on save
- **Data access:** "I want to give my SSN to fill out the driver's license" — attribute-based access control meets AgentPing
- **Physical surfaces:** Stream deck button, tablet on wall, voice command ("say this password")

### Action Items
- [ ] Merge into `docs/design/vernacular.md` — add Interface, ActionSurface, Gate, Lease as locked terms
- [ ] Cross-reference with `docs/design/graph-primitives.md` Gate primitive
- [ ] Reconcile with `04-agentping-human-loop.md` inbox file
