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
  → implement bead by bead in order
  → verify each bead before moving to next
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

## Stage 5 — Implement

Implement beads in dependency order. For each bead:
1. Read the bead doc
2. Write the code exactly as specified
3. Run the tests specified in the bead doc — all must pass
4. Do not move to the next bead until this bead's tests are green

**No skipping beads.** No implementing bead N+1 while bead N's tests are red.

## Hard Rules

- **Never implement before Stage 4 is clean.** A BLOCKER in the beads review means the code will be wrong.
- **Never combine beads.** Each bead is a discrete commit. If two beads touch the same file, they are still separate commits.
- **Human gates BLOCKERs and HOLEs.** The orchestrator resolves RISKs autonomously (add to arch doc trade-offs). Only BLOCKERs and HOLEs go to the human.
- **Arch doc is the source of truth.** If a bead doc conflicts with the arch doc, the arch doc wins unless the human explicitly overrides it.
