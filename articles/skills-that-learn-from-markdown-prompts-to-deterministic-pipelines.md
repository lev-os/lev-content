---
title: "Skills That Learn: From Markdown Prompts to Deterministic Pipelines"
description: "The progression from lossy LLM-interpreted skills to hardened FlowMind definitions. Why the path from markdown to determinism is the key to production-grade agent systems."
tags:
  - skills
  - flowmind
  - determinism
  - ai-agents
  - production-systems
date: 2026-04-10
---

# Skills That Learn: From Markdown Prompts to Deterministic Pipelines

Every agent system starts the same way: someone writes a prompt in a markdown file, tells the agent to follow it, and hopes for the best.

The prompt says: "When the user asks for a deployment, check that the staging environment is healthy, then deploy to production, then verify the deployment succeeded." The agent reads the prompt, interprets it through a 100-billion-parameter language model, and does... something roughly in the neighborhood of what you intended.

This is fine for a prototype. It is a liability for production.

LLMs are remarkably good at following instructions. But "following instructions" and "executing a deterministic pipeline" are different operations. Production systems need the latter. Prompts converge, harnesses differentiate.

## The Lossy Interpretation Problem

When an LLM reads a markdown prompt and executes a skill, it is performing interpretation, not execution. The model takes the natural language instructions, maps them to its understanding of what those instructions mean, and generates actions based on that understanding.

This interpretation is lossy. The model might:

- Reorder steps that the prompt lists sequentially because it believes the order does not matter
- Skip a step it considers redundant based on context
- Add steps it believes are implied but not stated
- Interpret an ambiguous instruction differently than intended

Each of these behaviors is reasonable from the model's perspective. Language is ambiguous. The model is doing its best to interpret ambiguous instructions in context. But "doing its best" is not the same as "doing what you specified," and the gap between interpretation and specification is where production failures live.

Here is a concrete example. Your deployment skill prompt says:

```markdown
## Deployment Skill

1. Check staging health
2. Deploy artifact to production
3. Run smoke tests
4. Notify the team
```

The agent reads this and decides that step 1 is unnecessary because it already checked staging health in a previous conversation turn. It skips to step 2. The staging environment has degraded since that previous check. The deployment goes to production against an unhealthy staging baseline. Smoke tests pass because the smoke tests do not catch the specific degradation. The team gets notified that the deployment succeeded. Three hours later, users report errors.

The agent did not disobey the prompt. It interpreted the prompt and made a reasonable inference about which steps were necessary in context. That reasonable inference was wrong, and the markdown prompt had no mechanism to enforce step ordering.

## The Hardening Spectrum

The solution is not to write better prompts. It is to move skills along a hardening spectrum, from fully interpreted to fully deterministic. In Leviathan, this spectrum maps to a concrete progression:

**Level 0 -- Markdown prompt**: Natural language instructions in a `.md` file. Fully LLM-interpreted. Maximum flexibility, minimum reliability.

**Level 1 -- Structured YAML**: Instructions encoded as YAML with explicit ordering, required fields, and typed parameters. The LLM still executes each step, but the structure constrains the order and completeness.

**Level 2 -- FlowMind declaration**: A `.flow.yaml` file that compiles into an execution contract. The flow defines nodes (agent steps, function calls, gate evaluations), edges (transitions between nodes), and constraints (budgets, timeouts, required outputs). The LLM executes within each node, but the graph structure is deterministic.

**Level 3 -- Deterministic pipeline**: All LLM-interpreted steps have been replaced by function calls, API invocations, and deterministic logic. The skill is now a compiled program that runs identically every time.

The key insight: you do not need to start at Level 3. You start at Level 0 and harden over time. The progression is natural and data-driven.

## The FlowMind Bridge

FlowMind is the compilation layer that makes this progression possible. It takes declarations -- whether simple structured YAML or complex multi-step workflows -- and compiles them into execution contracts that the runtime evaluates deterministically.

A FlowMind program has four irreducible node types (derived from the 7 graph primitives in `dna/graph.yaml`):

- **Agent**: An LLM invocation. Probabilistic by nature, bounded by constraints.
- **Function**: A tool or function call. Deterministic.
- **Gate**: A routing decision. Evaluates conditions and branches.
- **Subgraph**: A nested program. Composable.

The hardening process is, at its core, the progressive replacement of agent nodes with function nodes. Each time you observe that an agent node produces consistent, predictable output for a given input class, you can replace it with a function that produces that output deterministically.

Consider the deployment skill at FlowMind level:

```yaml
name: deploy-production
nodes:
  check_staging:
    type: function
    action: "http.get"
    params:
      url: "${staging_health_endpoint}"
    outputs: [health_status]

  evaluate_health:
    type: gate
    eval: "health_status.code == 200 AND health_status.body.status == 'healthy'"
    branches:
      pass: deploy
      fail: abort

  deploy:
    type: function
    action: "deploy.push"
    params:
      target: production
      artifact: "${artifact_id}"
    outputs: [deploy_receipt]

  smoke_test:
    type: function
    action: "test.smoke"
    params:
      environment: production
      receipt: "${deploy_receipt.id}"
    outputs: [test_results]

  notify:
    type: function
    action: "slack.post"
    params:
      channel: "#deployments"
      message: "Deployed ${artifact_id}: ${test_results.summary}"
```

This is the same deployment skill, but now every step is a function node. The staging health check is an HTTP call, not an LLM deciding whether to check. The health evaluation is a deterministic gate, not an LLM judging "healthy enough." The deployment is an API call. The notification is a function.

The LLM is entirely absent. The skill runs the same way every time. The markdown prompt has hardened into a deterministic pipeline.

## How Skills Learn

The progression from Level 0 to Level 3 is not manual. It is driven by the system's own execution history.

Every time a skill runs, it produces a receipt -- an immutable record of what happened, what constraints were evaluated, and what output was produced. Over time, these receipts form a dataset that reveals where the skill is deterministic in practice.

If an agent node produces the same output structure for the same input class across 100 executions, that node is a candidate for hardening into a function. The node has learned its own behavior, and that learned behavior can be crystallized into deterministic code.

In Leviathan, this learning process follows the entity lifecycle:

- **Ephemeral**: The skill is a markdown prompt. Ideas and instructions, loosely held.
- **Captured**: The skill is documented with inputs, outputs, and observed behavior patterns.
- **Classified**: The skill's behavior is categorized -- which steps are deterministic, which require LLM judgment, which have variable outputs.
- **Crystallizing**: Deterministic steps are being converted to function nodes. The skill is partially hardened.
- **Crystallized**: The skill is a FlowMind declaration. Structure is locked. Probabilistic nodes are explicitly marked and bounded.
- **Manifesting**: The skill is running in production with constraint enforcement.
- **Completed**: The skill is fully hardened. All nodes are deterministic or explicitly bounded.

This lifecycle is not aspirational. It is enforced by the gate system -- gates = loss function for entity maturity. A skill cannot move from "crystallizing" to "crystallized" without passing validation gates that verify the FlowMind declaration compiles, the execution contract is valid, and the constraint coverage is complete. The 8 entity types in `dna/entities.yaml` all follow this same lifecycle ratchet.

## The Translator Boundary

There is a hard architectural line in this progression: the translator boundary. On one side, LLM classification and interpretation -- probabilistic, best-effort, context-dependent. On the other side, kernel evaluation -- deterministic, rule-based, reproducible.

The translator boundary is not a gradual transition. It is a hard cut. When a FlowMind program compiles, the compilation output is a pure data structure -- an execution contract -- that the orchestration engine evaluates without any LLM involvement. The LLM's work is done at authoring time (writing the flow declaration) and at agent-node execution time (within bounded, gated nodes). The flow structure itself is deterministic.

This matters because it means the flow's behavior is auditable, reproducible, and provable. You can run the same execution contract against the same inputs and get the same structural behavior every time. The agent nodes within the flow may produce different outputs (they are LLMs, after all), but the flow's routing, gating, and constraint enforcement is identical across runs.

This is the property that makes FlowMind programs suitable for production. Not that they eliminate LLM variability -- that is impossible and undesirable. But they contain it. The variability is boxed into explicit agent nodes with declared budgets, output schemas, and constraint evaluations. The rest of the pipeline is deterministic.

## Why This Matters for Production Agent Systems

The markdown-to-determinism progression solves the three problems that kill agent systems in production:

**Reliability**: A deterministic pipeline does the same thing every time. A markdown prompt does approximately the same thing most of the time. The difference between these two statements is the difference between a system you can operate and a system you can only observe.

**Auditability**: A FlowMind execution contract produces a complete trace of every node execution, every gate evaluation, and every state transition. A markdown prompt produces a chat log. When regulators or customers ask "what did your system do and why?", the execution trace is an answer. The chat log is a story.

**Evolution**: A hardened skill can be composed into larger workflows with confidence. A markdown prompt can only be composed with hope. When your deployment skill is a deterministic pipeline, you can wire it into a release workflow, a rollback workflow, and an emergency procedure, knowing it will behave identically in each context. When it is a markdown prompt, each context might trigger a different interpretation.

## Starting the Progression

You do not need to rewrite all your skills today. Start with the one that matters most -- the skill that runs most frequently, or the one where failure is most expensive.

1. Run it 20 times and collect the receipts. Look at the outputs.
2. Identify which steps produce consistent outputs. Those are your hardening candidates.
3. Convert those steps to function nodes in a FlowMind declaration. Keep the variable steps as agent nodes.
4. Run the hybrid flow 20 more times. Compare reliability.
5. Repeat: identify consistent agent nodes, convert to functions, validate.

Within a few iterations, your most critical skill will be significantly hardened. The LLM is still doing the creative work -- the parts that genuinely require judgment and context. But the scaffolding around that creative work is deterministic, auditable, and constrained.

Skills start as markdown prompts because that is the fastest way to express intent. They end as deterministic pipelines because that is the only way to operate at scale. Prompts converge, harnesses differentiate -- and the path between those two points is the path from prototype to production.

The system learns. The skills harden. The constraints tighten.

That is how you build agent skills that survive contact with reality.
