---
name: hostile-beads-review
description: Use when a beads document exists and needs adversarial review before per-bead implementation docs are written — specifically to find contract mismatches, arch doc violations, ordering errors, structural assumptions, and scope creep before any code is touched.
---

# Hostile Beads Review

## Overview

An adversarial review of a beads document cross-referenced against the architecture document. The reviewer's job is to find every way the bead chain can fail — not to validate it.

**Default posture:** Every "depends on" claim is wrong until you verify the dependency's output contract matches this bead's required input. Every bead is doing too much until you confirm it delivers exactly one logical thing.

## What You Need Before Starting

- The beads document (full bead chain with all "Depends on" fields)
- The architecture document (commandments, trade-off decisions, constraints, performance targets)

Read both in full before writing a single finding.

## How to Run This Review — Parallel Dispatch

The seven dimensions below are independent: each needs only the beads doc and the arch doc as input, and none consumes another dimension's findings. **Default to dispatching them as parallel subagents**, all launched in a single message so they run concurrently:

1. One subagent per dimension (or group cheap adjacent ones — e.g. 4+6, 5+7 — into a shared agent; never more than two dimensions per agent).
2. Each subagent receives: both documents (paths, or full content), the instructions for exactly its dimension(s), the severity labels, the "What NOT to Surface" rules, and the output format. It returns only findings in that format.
3. The orchestrator merges the results: dedupe overlapping findings (dimensions 1 and 3 routinely catch the same edge problem — keep one finding, note both dimensions), then order BLOCKERs → HOLEs → RISKs, by bead number within each group.

Fall back to running all dimensions inline, sequentially, only when subagent dispatch is unavailable. The re-review pass in "After the Review" (changed beads only) is small enough to run inline.

Parallel dispatch changes who does the work, not the bar: every dimension still runs, every red flag below still applies to the merged result.

## Review Dimensions

Run every dimension. Do not skip one because it seems fine.

### 1. Input/Output Contract Continuity

For every bead with a "Depends on" entry:
- What does the predecessor bead's output contract deliver? (exact type, shape, fields)
- What does this bead's implementation require as input?
- Do they match exactly — same type, same nullability, same field names?

For every bead that has dependents:
- What does this bead deliver?
- What do each of its dependents expect?
- Any mismatch is a BLOCKER.

Do not assume "close enough." A function that returns `string | null` feeding a bead that expects `string` is a BLOCKER.

### 2. Arch Doc Violations

Read every commandment, constraint, and trade-off decision in the arch doc. For each bead, check:
- Does this bead introduce I/O in a module declared pure?
- Does this bead add a dependency the arch doc explicitly excluded?
- Does this bead's approach contradict a stated trade-off decision?
- Does this bead import a layer it isn't supposed to touch?

Any violation is a BLOCKER.

### 3. Ordering Violations

Walk the DAG. For each dependency edge:
- Can bead N actually be implemented before bead N+1 starts, given what each delivers?
- Is there a circular dependency hidden in the "Depends on" fields?
- Does bead N+1 use something from bead N that bead N doesn't actually produce yet?

An ordering violation is a BLOCKER.

### 4. Big Structural Assumptions

For each bead, identify what it assumes exists:
- A schema shape or DB table
- A file or module at a specific path
- A function signature or exported type
- An environment variable or config key

For each assumption: which prior bead delivers it? Is it in the existing codebase? If neither, it's a HOLE.

### 5. Performance Assumptions

Identify every performance target in the arch doc (latency, throughput, concurrency). For each bead:
- Does it introduce sequential awaits where the arch doc requires concurrent execution?
- Does it add a blocking call in a hot path?
- Does it batch where the arch doc says stream, or stream where it says batch?

Any pattern that would violate a performance target is a RISK (or BLOCKER if the target is hard).

### 6. Missing Beads

Read the arch doc for every deliverable it requires. For each one:
- Which bead implements it?
- If no bead covers it, that is a HOLE.

Common misses: shared utility extraction, error boundary setup, config validation, teardown/cleanup logic the arch doc mentions but no bead owns.

### 7. Scope Creep

Each bead must deliver exactly one logical thing. A bead that does more than one of the following is multiple beads:
- Changes a type or schema
- Adds or rewrites a module
- Updates a pipeline or orchestration layer
- Adds tests for a contract that doesn't exist yet in the chain

If a bead is doing two logical things, split it. If it is doing three, it is three beads. Flag as HOLE.

## What NOT to Surface

- Naming preferences or style choices
- Low-severity cosmetic issues
- "Consider" or "might want to" observations
- Anything that doesn't affect correctness, contracts, ordering, or performance

Every finding you write must have a concrete Fix. If you can't write a Fix, it's not a finding.

## Severity Labels

- **BLOCKER** — contract mismatch, arch violation, ordering error. Must be fixed before any per-bead implementation doc is written.
- **HOLE** — underspecified; the bead chain will require a decision that isn't recorded here. Must be resolved before the affected bead is implemented.
- **RISK** — known failure condition not acknowledged. Must be added to arch doc trade-offs before implementation.

## Output Format

Each finding:

```
N. [DIMENSION] Short title — SEVERITY
   Bead(s): Which bead number(s) are affected.
   Claim: What the beads doc asserts or assumes.
   Problem: Why it is wrong or risky.
   Consequence: What breaks at implementation time.
   Fix: Exactly how to resolve it in the beads doc (or arch doc).
```

Present BLOCKERs first, then HOLEs, then RISKs. Within each group, order by affected bead number.

## After the Review

1. Present all findings grouped by severity.
2. Do not proceed to writing per-bead implementation docs until all BLOCKERs and HOLEs are resolved.
3. Update the beads doc in place — do not create a separate reviewed version.
4. If a fix reveals an issue in the arch doc (missing trade-off, false constraint), update the arch doc too.
5. Re-run the review on changed beads only. Confirm no new holes were introduced by the fixes.

## Red Flags — You Are Being Too Soft If

- You have zero BLOCKERs on a beads doc with 4+ beads
- You didn't check every "Depends on" edge (both directions: output of predecessor, input of dependent)
- You didn't cross-reference every bead against every arch doc commandment
- You surfaced a finding without a concrete Fix
- You wrote "looks reasonable" anywhere
- You skipped a dimension because the beads doc "seemed clean"
