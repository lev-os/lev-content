# LinkedIn — 2026-04-14 — Constraint engineering

**Campaign:** Web Summit GTM sprint  
**Topic:** Constraints aren’t guardrails on creativity — they’re the product  
**Canonical surface:** `lev.here.now` / `docs/MANIFESTO.md`

---

## Hook

If your “agent safety” is a paragraph in a system prompt, you don’t have constraints. You have vibes.

---

## Body

Probability is not policy. When a model can always “try again,” the failure mode isn’t the wrong answer — it’s that nothing in your stack refused the transition.

We treat constraints the way you treat types and auth: tiered, explicit, enforced by something that isn’t the model. DNA-style tiers (think C1 through C5) exist so execution can fail closed: alignment passes or it doesn’t; the graph patch is declared or it isn’t; the gate ratchets or you don’t ship. The runtime owns that story — not a longer system message.

That’s how you put agents next to production systems without pretending “please be careful” is architecture. Conversation is where understanding forms; alignment is the gate; execution follows a patch you can inspect. Strip any of those layers and you’re not building an agent stack — you’re narrating one.

Web Summit is a month out. We’re shipping proof, not positioning paragraphs. If you’re wiring gates this sprint, the useful question isn’t “how smart is the model” — it’s “what does the runtime do when alignment fails?”

---

## CTA

If you’re wiring gates this month, reply with the one constraint you refuse to compromise — or read `docs/MANIFESTO.md` and tell me what clashes with your stack.

---

## Hashtags

#AgentRuntime #ConstraintEngineering #AIInfrastructure #WebSummit #BuildInPublic
