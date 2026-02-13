# Entity Lifecycle Cleanup

## Kill List

**BD Types to DELETE:**
- `synth.yaml` - dead concept
- `artifact.yaml` - overengineered
- `capability.yaml` - redundant with spec
- `initiative.yaml` - wtf is this
- `memory.yaml` - different system
- `message.yaml` - different system

**Keep:**
- `entity.yaml` (base)
- `idea.yaml`
- `proposal.yaml`
- `plan.yaml`
- `spec.yaml`
- `session.yaml`

## Types to ADD

```yaml
claim.yaml    # Atomic assertion (subject, predicate, object, truth_state)
evidence.yaml # Proof for claim (kind, source, payload_ref)
gate.yaml     # BUT BD ALREADY HAS bd gate - check if we need custom type
```

## FSM Cleanup

**Kill:**
- Synth FSM - dead

**Keep:**
- Idea FSM → rename to Entity FSM
- Session FSM (impl detail)

**Simplify Idea FSM:**
```
idle → captured → crystallizing → committed → planning → executing → complete
```

No "manifesting" bullshit. Executing = BD tasks. That's it.

## The Mud We Fixed

OLD (overloaded):
```
crystallized → manifesting → creatingIssue → manifested → manifestation plugin → ???
```

NEW (clean):
```
committed → planning → executing (BD) → complete (gates pass)
```
