---
title: "The Universal SDLC: Why This Cycle Works for Everything, Not Just Code"
description: "The 6-step universal cycle -- Converse, Propose, Plan, Execute, Feedback, Learn -- applied to content creation, research, design, and operations. Software is where you prove it first."
tags: [sdlc, lifecycle, universal-cycle, flowmind, constraint-engineering, workflow]
date: 2026-04-10
series: leviathan-engineering
---

# The Universal SDLC: Why This Cycle Works for Everything, Not Just Code

The lifecycle that works for building software also works for building everything else. Almost nobody has noticed.

Leviathan's universal SDLC is a 6-step cycle: **Converse, Propose, Plan, Execute, Feedback, Learn.** It looks like a software development lifecycle because that is where it was designed and proven across 30+ flows and 490 benchmark runs. But it is not about software. It is about how any creative work moves from vague intent to validated output through constraint-satisfying steps.

## The Six Steps

### 1. Converse

Raw intent enters the system. A human says "I need a blog post about agent sandboxing." An agent observes "the test suite has 3 failing tests." A cron job fires and delivers "the nightly build broke."

This is the intake layer. The system classifies the incoming signal: what kind of work is this? What domain does it belong to? What is the current state of related entities? In Leviathan's architecture, every message triggers the Tick loop -- INGEST, OBSERVE, PROPOSE, GATE, APPLY, UPDATE, EMIT -- across all 7 graph primitives defined in `dna/graph.yaml`. Even at this earliest stage, the conversation is not idle chatter. It is graph mutation.

The key property of this step: it is read-only. The conversational agent gathers context, classifies entities, and observes state. It does not execute. It does not commit. It asks clarifying questions if the intent is ambiguous. The bias is toward understanding before action.

### 2. Propose

The classified intent crystallizes into a concrete proposal. "Write a 1200-word blog post about AgentFS, covering CoW overlays, audit trails, and reactive hooks." The proposal has a specific scope, specific deliverables, and specific acceptance criteria.

In Leviathan's graph model, a proposal is a candidate mutation. It describes what will change and what conditions must be satisfied for the change to be accepted. This is the equivalent of a pull request for any kind of work, not just code.

The proposal stage is where constraints first engage. The system evaluates: Is this within scope? Are the required capabilities available? Does this conflict with other active work? Gates run against the proposal before any execution begins. Catching a bad idea at the proposal stage costs almost nothing. Catching it after execution costs everything.

### 3. Plan

The accepted proposal decomposes into an execution plan. Not a vague roadmap -- a concrete sequence of bounded steps, each with its own acceptance criteria and exit conditions.

For software: write the spec, implement the handler, write tests, validate against gates, create the PR. For a blog post: research the topic, outline the structure, write the draft, run the editorial checklist, validate against the style guide. For a design: frame the problem, define JTBD, create wireframes, run UX review, produce final assets.

The plan is itself a proposal-shaped artifact. It can be reviewed, amended, or rejected before execution starts. And critically, each step in the plan is bounded. No step takes infinite time, consumes infinite tokens, or produces unbounded output. This is C1 Finitude applied to planning: every step must declare its budget.

### 4. Execute

The plan runs. Each step fires the tick loop: ingest the step context, observe current state, propose the step's output, gate the output against acceptance criteria, apply if it passes, update state, emit events.

Execution in Leviathan is always delegated to specialized workers with scoped capabilities. The conversational agent does not write code or draft blog posts. It submits structured jobs to execution agents that have the specific tools and context required for the task.

This separation matters because execution agents operate under explicit constraints: time budgets, token budgets, tool access lists, and filesystem sandboxes (via AgentFS). The execution layer cannot exceed its bounds, and every action it takes is recorded in an append-only audit trail.

### 5. Feedback

The output of execution passes through validation. This is not "does the human like it?" (though that is one feedback surface). It is structured evaluation against the acceptance criteria defined in the proposal and plan stages.

For software: do the tests pass? Does the linter agree? Do the validation gates clear? For a blog post: does it hit the word count? Does it cover the required topics? Does it pass the editorial checklist? For a design: does it satisfy the JTBD? Does it meet accessibility standards?

Leviathan uses an eval harness with four strategy kinds: gate-based (binary pass/fail), metric-based (numeric thresholds), LLM-as-judge (subjective quality), and adversarial (did the agent game the system?). Gates = loss function. The eval strategy is defined before execution begins, not after -- eliminating post-hoc criteria adjustment.

### 6. Learn

Feedback produces artifacts that feed back into the system. Successful patterns become templates. Failed patterns become constraints. The skill that produced a good blog post gets reinforced. The approach that produced a bad one gets a new adversarial check.

In Leviathan's architecture, learning manifests as:

- **Skill evolution**: Successful execution patterns crystallize into reusable FlowMind declarations.
- **Gate expansion**: Failures that slipped through validation become new enforcement gates.
- **Template updates**: Entity templates (spec templates, plan templates, report templates) update based on what produced good outcomes.
- **Experience capture**: The system records what worked and what did not, building institutional memory that persists across sessions.

The learn step is what makes the cycle compound rather than repeat. Each iteration through the cycle produces not just the output (the code, the blog post, the design) but also improvements to the cycle itself.

## Applied to Content Creation

Here is the cycle applied to writing a series of technical blog posts:

**Converse**: "We need 5 articles about Leviathan's capabilities for the open source project." The system classifies this as a content creation workstream. It observes the existing content, the project documentation, and the target audience.

**Propose**: Each article gets a proposal with title, target word count, required topics, and tone guidelines. The proposals are reviewed for scope (are 5 articles enough?), feasibility (do we have sufficient source material?), and standards (do they align with the brand voice?).

**Plan**: Each article decomposes into research (gather technical details from DNA files, specs, and architecture docs), outline (structure the argument), draft (write the full text), and validate (editorial checklist, accuracy check against source material).

**Execute**: The writer (human or agent) produces each article following the plan. Each article is bounded: target word count, required sections, specific technical claims that must be sourced from documentation.

**Feedback**: Each article passes through editorial validation. Does it hit the word count? Does it cover the required topics? Are the technical claims accurate? Does the tone match the style guide? Are the code examples correct?

**Learn**: The articles that performed well (by whatever metric -- reader engagement, accuracy, editorial quality) feed back as templates for future content. The patterns that did not work become constraints on future proposals.

## Applied to Research

**Converse**: "We need to understand how competitor X implements feature Y." Classification: research workstream, competitive analysis domain.

**Propose**: Research scope defined. Specific questions to answer. Sources to consult. Time budget. Deliverable format (report with structured findings).

**Plan**: Decompose into source gathering (documentation, code, benchmarks), analysis (compare against our architecture), synthesis (produce findings), and validation (are the findings supported by evidence?).

**Execute**: Researcher executes the plan. Each step is bounded by the time and source budget.

**Feedback**: Eval against the original questions. Were they answered? Is the evidence sufficient? Are the conclusions supported?

**Learn**: Research methodology that worked gets captured as a reusable research flow. Sources that proved valuable get indexed for future use.

## Applied to Operations

**Converse**: "The nightly build broke." Classification: incident, execution domain.

**Propose**: Root cause hypothesis. Proposed fix. Scope of impact.

**Plan**: Investigate (gather logs, reproduce), fix (implement the correction), validate (run the build), document (update runbook if needed).

**Execute**: Engineer applies the fix within the scoped plan.

**Feedback**: Build passes. No regression. Runbook is current.

**Learn**: If this is a recurring failure mode, a new gate gets created to prevent it. The gate enters the ratchet at aspirational and progresses toward enforcement.

## Why Software First

Software engineering is the proof ground for the universal cycle because it has the richest infrastructure for structured feedback. Tests pass or fail. Linters produce deterministic results. Type checkers enforce constraints. CI pipelines run automatically.

This infrastructure makes the feedback step concrete and measurable. You do not have to argue about whether the output is good. You run the tests. They pass or they do not.

But the cycle itself is not about tests or CI. It is about a pattern: intake, crystallization, decomposition, bounded execution, structured feedback, and learning. That pattern applies whenever you are transforming intent into validated output under constraints.

Constraints are the product here too. C1 Finitude says every operation must be bounded. C2 Non-Commutation says order matters and history is irreversible. These are not software engineering principles. They are physics -- the two axioms in `dna/graph.yaml` from which everything else derives. They apply to writing, research, design, operations, and any other domain where finite beings do work in irreversible time.

## The Compound Effect

The most important property of the universal cycle is not any individual step. It is that the learn step feeds back into every other step. Each iteration makes the next iteration better. Templates improve. Gates accumulate. Skills sharpen. Constraints tighten.

Over time, a team running this cycle does not just produce better output. They produce output faster, with fewer iterations, at lower cost, with higher reliability. Not because they work harder, but because the cycle compounds.

This is why Leviathan encodes the cycle as a system primitive rather than a process document. Process documents get ignored. Encoded primitives execute. Every workstream, every session, every tick follows the same pattern: observe, propose, gate, apply, update, emit. The cycle is not something you decide to follow. It is what the runtime does.

Software engineering is just where you prove it first. Everything else follows.

---

*This article is part of the Leviathan engineering series exploring the design principles behind the universal agent runtime. Leviathan is open source at [github.com/lev-os/lev](https://github.com/lev-os/lev).*
