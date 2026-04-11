---
title: "AgentFS: The Sandbox Your AI Agents Can't Escape"
description: "The case for filesystem-level isolation in AI agent systems. Copy-on-write overlays, BLAKE3 audit trails, reactive hooks, and why agents must not bypass constraints."
tags: [agentfs, sandbox, filesystem, security, isolation, copy-on-write, audit-trail]
date: 2026-04-10
series: leviathan-engineering
---

# AgentFS: The Sandbox Your AI Agents Can't Escape

Every serious agent system eventually faces the same question: what happens when the agent does something wrong?

Not "wrong" in the hallucination sense. Wrong in the "it deleted the production database" sense. Wrong in the "it wrote credentials to a public file" sense. Wrong in the "it modified 47 files and we cannot figure out which changes were intentional" sense.

The standard answer is to restrict what tools the agent can call. Deny lists. Approval workflows. Permission prompts. These are necessary but insufficient. They operate at the intent level -- they try to prevent the agent from wanting to do bad things. They do not prevent the consequences when an approved action has unintended side effects.

AgentFS takes a different approach. It operates at the filesystem level. The agent cannot bypass it because there is nothing to bypass -- every file operation goes through a virtual filesystem backed by SQLite, with copy-on-write isolation, reactive hooks, and an append-only audit trail.

## The Architecture: One File to Rule Them All

AgentFS is a single SQLite file that provides three interfaces:

| Interface | Purpose |
|---|---|
| Virtual Filesystem | POSIX-compliant inodes, directories, symlinks, overlay CoW, whiteouts |
| Key-Value Store | Namespaced JSON with automatic timestamps |
| Tool Call Audit Trail | Append-only, tamper-resistant execution history |

The virtual filesystem mounts via FUSE on Linux and NFS on macOS. To any process running inside the mount -- including the AI agent -- it looks like a normal filesystem. `open()`, `read()`, `write()`, `unlink()` all work exactly as expected. The agent does not know it is sandboxed. It does not need to use special APIs. It just reads and writes files.

But every one of those operations goes through SQLite. Every write is checksummed with BLAKE3. Every mutation fires reactive hooks that can inspect, modify, or block the operation before it completes.

## Copy-on-Write: Fork, Validate, Commit or Discard

The defining feature of AgentFS is its overlay copy-on-write system. Here is how it works in practice:

1. **Fork state.** Create an overlay on top of the current filesystem state. The overlay sees all existing files but writes go to a separate delta layer.

2. **Let the agent work.** The agent reads and writes files through the overlay. From its perspective, nothing is different. But the base layer remains untouched.

3. **Validate.** Run constraint checks against the overlay. Did the agent produce valid output? Did it respect file size limits? Did it modify files it should not have touched? Did the frontmatter schema pass validation?

4. **Commit or discard.** If validation passes, merge the overlay delta into the base layer. If it fails, discard the overlay entirely. The base layer never saw the invalid state.

This is not theoretical. The overlay implementation lives in `crates/lev-agentfs/sdk/rust/src/filesystem/overlayfs.rs`. It uses a delta table in SQLite to track modifications, and whiteout entries to track deletions. The base layer's checksums are preserved through copy-up operations.

The implication: you can let an agent run with full filesystem access inside its sandbox, knowing that nothing it does is permanent until you explicitly commit it. If the agent goes off the rails -- writes garbage, deletes critical files, creates an inconsistent state -- you discard the overlay and lose nothing.

## Reactive Hooks: Policy at the Filesystem Boundary

AgentFS integrates with Leviathan's reactive hooks system to enforce policy at the filesystem level. Every mutating operation -- create, write, unlink, rename, mkdir, rmdir, symlink, link -- can trigger sync and async hooks.

Sync hooks fire before the operation completes. They can inspect the operation and return one of three decisions:

- **Allow**: The operation proceeds normally.
- **Block**: The operation is denied. The agent receives `EPERM`. No data is written.
- **Warn**: The operation proceeds, but a warning is logged for human review.

Consider a concrete example. You want to prevent an agent from writing files larger than 1MB, from creating files with certain extensions, or from modifying files that match a protected path pattern. You register sync hooks:

```yaml
hooks:
  - event: "file:write"
    match: "*.secret"
    action: block
    reason: "Protected file pattern"
  - event: "file:create"
    max_size: 1048576
    action: block
    reason: "File exceeds 1MB limit"
```

When the agent tries to write to `config.secret`, the sync hook fires, evaluates the policy, and returns `Block`. The agent gets a permission error. No data reaches SQLite. The hook decision is logged with the reason.

This is fundamentally different from application-level permission checks. The agent cannot circumvent filesystem hooks by using a different API, calling a subprocess, or exploiting a code path that skips the check. Every file operation -- regardless of how it is initiated -- passes through the same hook pipeline.

## The Audit Trail: What Did the Agent Actually Do?

The tool call audit trail is append-only and tamper-resistant. Every tool invocation the agent makes is recorded with:

- The tool name and arguments
- The timestamp
- The result (success or failure)
- The files that were read or modified as a consequence

This serves two purposes. First, debugging. When an agent produces unexpected output, you can reconstruct exactly what happened: which files it read, in what order, what it wrote, and when. Second, compliance. In regulated environments, you need to prove what an AI system did and did not do. The audit trail provides that proof.

The append-only property matters. The agent cannot modify its own audit trail. It cannot delete entries, backdate timestamps, or alter recorded results. The trail is a faithful record of what actually happened, not what the agent claims happened.

Combined with BLAKE3 checksums on every stored chunk, you get data integrity verification built into the filesystem itself. The `agentfs scrub` command validates all chunks against their stored checksums and reports any corruption -- silent or otherwise.

## Multi-Surface Mount: Everywhere the Agent Runs

AgentFS mounts on four surfaces:

- **FUSE** on Linux: full kernel-level filesystem interception
- **NFS** on macOS: network filesystem mount with hook parity
- **WASM** in browsers: for agent-in-browser use cases
- **Durable Objects** on edge: for Cloudflare Workers and similar runtimes

This means the same sandbox model works regardless of where the agent runs. A local development agent gets the same isolation guarantees as an edge-deployed agent. The SQLite backing store is the same. The hooks are the same. The audit trail is the same.

## Why Application-Level Sandboxing Is Not Enough

The standard alternative to filesystem-level sandboxing is application-level access control. You define a list of allowed tools, each tool checks permissions before executing, and you hope that every code path respects those checks.

This fails in practice for three reasons:

**1. Bypass paths exist.** Any tool that can execute shell commands, write files through a library, or invoke a subprocess can circumvent application-level checks. The agent does not need to be malicious -- it just needs to find a code path that does not go through the permission layer.

**2. Composition breaks isolation.** Tool A is safe. Tool B is safe. Tool A followed by Tool B produces an unsafe state. Application-level checks evaluate each tool in isolation. Filesystem-level isolation evaluates the cumulative state.

**3. Audit is incomplete.** Application-level logging records what the tool's API layer saw. It does not record what actually happened at the filesystem level. If a tool writes a temp file, reads it back, and deletes it, the application log might only show the final result. The filesystem audit trail shows every operation.

AgentFS eliminates all three failure modes. There is no bypass path because every file operation goes through the virtual filesystem. Composition is handled by the overlay -- validate the cumulative state, not individual operations. And the audit trail records every filesystem operation, not just the ones the application layer decided to log.

## The Constraint Engineering Connection

In Leviathan's architecture, AgentFS is not a safety feature bolted on after the fact. It is the sandbox layer for constraint engineering -- constraints are the product, and AgentFS is where they execute at the filesystem boundary.

The eval harness uses AgentFS overlays as its testing ground. Fork state, run the agent, validate the output against gates (gates = loss function), commit or discard. This is the same pattern whether you are running a unit test, a full integration test, or a production agent workload.

The reactive hooks are the enforcement mechanism for validation gates. A gate that says "no file larger than 100KB" is not enforced by hoping the agent respects the limit. It is enforced by a sync hook that blocks the write at the filesystem level. The gate ratchet -- aspirational to declared to enforced to auto-enforced (61 module gates across the current system, all tracked in `dna/gates.yaml`) -- applies to filesystem policies just as it applies to code quality standards.

And the audit trail feeds the eval harness with ground truth data about what the agent did. When you are comparing two agent strategies (did strategy A or strategy B produce better results?), you need accurate records of what each strategy actually did. The filesystem audit trail provides that, unfakeably.

## Getting Started

AgentFS ships as part of Leviathan's Rust crate workspace. The core components:

- `crates/lev-agentfs/cli` -- the `agentfs` binary with mount, init, run, exec, and scrub commands
- `crates/lev-agentfs/sdk/rust` -- the Rust SDK with `FileSystem` trait, `AgentFS` (SQLite-backed), `OverlayFS` (CoW delta), and `HostFS` (passthrough)
- `crates/lev-agentfs/sandbox` -- Linux ptrace-based syscall interception via Reverie

TypeScript and Python SDKs are also available for integration into existing agent frameworks.

The simplest way to start: mount an AgentFS instance, point your agent at it, and watch the audit trail. You will learn more about your agent's actual behavior in ten minutes than you would from a week of log analysis.

---

*This article is part of the Leviathan engineering series exploring the design principles behind the universal agent runtime. Leviathan is open source at [github.com/lev-os/lev](https://github.com/lev-os/lev).*
