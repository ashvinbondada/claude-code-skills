---
name: arch-to-implementation
description: Use when an architecture document is ready and you need to move from design to implementation — runs the full loop: beads doc, hostile reviews, per-bead docs, then implementation bead by bead.
---

# Arch to Implementation

## Overview

Orchestrates the full loop from architecture document to running code. Each stage gates the next — nothing moves forward until the current stage is clean.

## The Loop

```
architecture doc (exists, reviewed)
  → write beads doc
  → hostile-beads-review (cross-ref arch doc)
  → resolve all BLOCKERs and HOLEs with human
  → fix beads doc + arch doc if needed
  → delete beads doc, write one bead doc per bead (write-bead-doc)
  → hostile-beads-review again (against per-bead docs)
  → resolve any new BLOCKERs with human
  → parallel-implementation-plan (wave schedule + validator)
  → implement wave by wave (beads within a wave in parallel)
  → verify each wave before moving to next
```

## Stage 1 — Write Beads Doc

Invoke `beads-document` skill. The beads doc must reference the arch doc by path. Every bead must have: title, depends-on, delivers, input contract, output contract, implementation steps, tests.

**Gate:** Do not proceed until the beads doc is written and saved.

## Stage 2 — Hostile Beads Review

Invoke `hostile-beads-review` skill with both the beads doc and the arch doc as context.

Present findings to human grouped by severity: BLOCKERs → HOLEs → RISKs.

**Gate:** Do not write any per-bead docs until all BLOCKERs and HOLEs are resolved. For each resolution: update the beads doc in place, update the arch doc if the fix reveals an arch-level issue.

## Stage 3 — Per-Bead Docs

For each bead in order, invoke `write-bead-doc`. Each doc is saved to `docs/architecture/beads/bead-{N:02d}-{slug}.html`.

Run all beads in parallel if they have no dependencies between them. Run sequentially if bead N's doc requires knowing what bead N-1 delivers.

**Gate:** All per-bead docs must exist before Stage 4.

## Stage 4 — Final Review

Re-run `hostile-beads-review` using the per-bead docs as primary input and the arch doc as reference. At this stage the review focuses on: do the per-bead docs contradict each other? Does any bead doc introduce something not in the beads doc?

**Gate:** Resolve any new BLOCKERs with human before implementing.

## Stage 5 — Parallel Implementation Plan

Invoke `parallel-implementation-plan`. It extracts each bead's file footprint from the per-bead docs, computes a wave schedule (all dependencies in earlier waves, pairwise-disjoint file sets within a wave), emits the plan as JSON, and validates it with a deterministic script.

**Gate:** The validator must exit 0. If the plan is degenerate (every wave has one bead), review the reported bottleneck file(s) with the human before proceeding — it may be architectural feedback.

## Stage 6 — Implement

Implement wave by wave per the validated plan. For each wave:
1. Dispatch one subagent per bead in the wave, concurrently. Each reads its bead doc, writes the code exactly as specified, and stays inside its declared file footprint.
2. Commit beads in bead-number order — each bead is still one discrete commit.
3. After each bead's commit, check `git diff --name-only` against its declared footprint. Any file outside it is a hard failure: revert or amend, fix the plan, re-validate.
4. Run the combined test suite for the wave (every bead's tests, plus the project suite) — all must pass before the next wave starts.

**No skipping beads, no skipping waves.** No wave N+1 while wave N is red. If subagent dispatch is unavailable, implement the plan's beads sequentially in wave order — the plan still fixes the order and the footprint checks still apply.

## Hard Rules

- **The plan validator gates implementation.** No code is written until `parallel-implementation-plan`'s validator exits 0.
- **Never implement before Stage 4 is clean.** A BLOCKER in the beads review means the code will be wrong.
- **Never combine beads.** Each bead is a discrete commit. If two beads touch the same file, they are still separate commits.
- **Human gates BLOCKERs and HOLEs.** The orchestrator resolves RISKs autonomously (add to arch doc trade-offs). Only BLOCKERs and HOLEs go to the human.
- **Arch doc is the source of truth.** If a bead doc conflicts with the arch doc, the arch doc wins unless the human explicitly overrides it.
