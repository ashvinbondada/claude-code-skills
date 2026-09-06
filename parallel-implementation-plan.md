---
name: parallel-implementation-plan
description: Use after per-bead docs pass final hostile review and before implementation — analyzes each bead's file footprint, computes a wave schedule where dependency-free, file-disjoint beads run in parallel, and gates execution behind a deterministic validator.
---

# Parallel Implementation Plan

## Overview

The beads DAG encodes *logical* dependency order. That alone is not enough to parallelize implementation: two beads with no edge between them can still touch the same file. This skill adds the second, orthogonal constraint — **file-set disjointness** — and turns the DAG into an execution plan of **waves**. Beads in the same wave have all dependencies satisfied by earlier waves and pairwise-disjoint file sets, so they can be implemented by concurrent subagents without stepping on each other.

The plan is not trusted because the model believes it is correct. It is trusted because a small deterministic validator checks it. **The validator passing is the gate — not judgment.**

## What You Need Before Starting

- The beads doc (full DAG, all "Depends on" fields)
- All per-bead implementation docs (they specify exact code, so file paths are derivable)
- Stage 4 of the loop (final hostile review) must be clean. Footprints extracted from unreviewed bead docs are not trustworthy.

## Step 1 — Extract File Footprints

For each bead, from its per-bead doc, list **every file it will create or modify**. Two categories, both mandatory:

- **Delivered files** — the modules, tests, and schemas the bead exists to produce.
- **Incidental files** — files the bead touches on the way. This is where hidden conflicts live. Check explicitly for: barrel/index files, route or plugin registries, `package.json` / lockfiles, migration indexes, config files, generated files, shared type files.

If you cannot determine a bead's full footprint from its doc, that is a HOLE in the bead doc — stop and fix the doc, do not guess.

Any file shared between two beads makes them **same-wave incompatible**, no exceptions. There is no "they touch different parts of the file."

## Step 2 — Compute Waves

Greedy scheduling over the DAG:

1. A bead is *eligible* for the current wave when every bead it depends on is assigned to an earlier wave.
2. Add an eligible bead to the current wave only if its file set is disjoint from every bead already in that wave. Prefer lower bead numbers first so the plan is deterministic.
3. When no more beads fit, close the wave and start the next.
4. Repeat until every bead is scheduled. If eligible beads remain but none can ever be scheduled, the DAG has a cycle — that is a BLOCKER against the beads doc.

Record every pairwise conflict that forced a bead into a later wave, so a human can audit why the plan is shaped the way it is.

**Degenerate plans are findings.** If every wave has one bead, there is a serialization bottleneck — usually one god-file every bead touches. Report which file(s) cause it; that is architectural feedback, and possibly a reason to split the file into its own bead first.

## Step 3 — Emit the Plan as JSON

Save to `docs/architecture/beads/parallel-plan.json`:

```json
{
  "beads": [
    { "id": 1, "title": "schema types", "depends_on": [], "files": ["src/types.ts", "src/index.ts"] },
    { "id": 2, "title": "parser", "depends_on": [1], "files": ["src/parser.ts", "test/parser.test.ts"] },
    { "id": 3, "title": "renderer", "depends_on": [1], "files": ["src/render.ts", "test/render.test.ts"] }
  ],
  "waves": [[1], [2, 3]],
  "conflicts": [
    { "beads": [1, 2], "reason": "dependency: 2 depends on 1" }
  ]
}
```

`beads` is the input (DAG + footprints), `waves` is the schedule, `conflicts` documents why beads were separated (dependency or shared file — name the file).

## Step 4 — Validate

Write this validator to `docs/architecture/beads/validate-plan.py` and run it against the plan. Do not paraphrase or "improve" it; its simplicity is the point.

```python
#!/usr/bin/env python3
"""Validate a parallel implementation plan. Exit 0 iff the plan is sound."""
import json, sys

def main(path):
    plan = json.load(open(path))
    beads = {b["id"]: b for b in plan["beads"]}
    errors = []

    # Every bead scheduled exactly once
    scheduled = [i for wave in plan["waves"] for i in wave]
    if sorted(scheduled) != sorted(beads):
        errors.append(f"scheduled beads {sorted(scheduled)} != declared beads {sorted(beads)}")

    wave_of = {i: w for w, wave in enumerate(plan["waves"]) for i in wave}

    for b in plan["beads"]:
        # Dependencies exist and land in strictly earlier waves
        for dep in b["depends_on"]:
            if dep not in beads:
                errors.append(f"bead {b['id']} depends on unknown bead {dep}")
            elif b["id"] in wave_of and dep in wave_of and wave_of[dep] >= wave_of[b["id"]]:
                errors.append(f"bead {b['id']} (wave {wave_of[b['id']]}) depends on bead {dep} (wave {wave_of[dep]})")
        if not b["files"]:
            errors.append(f"bead {b['id']} has an empty file footprint")

    # File sets pairwise disjoint within each wave
    for w, wave in enumerate(plan["waves"]):
        for i, a in enumerate(wave):
            for c in wave[i + 1:]:
                shared = set(beads[a]["files"]) & set(beads[c]["files"])
                if shared:
                    errors.append(f"wave {w}: beads {a} and {c} share files {sorted(shared)}")

    for e in errors:
        print(f"INVALID: {e}")
    print("OK" if not errors else f"{len(errors)} error(s)")
    sys.exit(1 if errors else 0)

if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "docs/architecture/beads/parallel-plan.json")
```

**Gate:** the validator must exit 0 before any implementation starts. If it fails, fix the plan (or the beads doc) and re-run. Never hand-wave past a validator error.

Note that all-beads-scheduled plus every-dependency-in-an-earlier-wave together imply the DAG is acyclic — the validator covers cycles without checking them explicitly.

## Execution Semantics

These replace the strictly sequential Stage 5 rules of `arch-to-implementation` when a validated plan exists:

1. **Within a wave:** dispatch one subagent per bead, all in a single message so they run concurrently. Each subagent gets its bead doc and its declared file footprint, and must not touch any file outside it.
2. **Commits stay discrete:** each bead is still exactly one commit. Apply/commit within a wave in bead-number order (parallel drafting, serialized commit), or use isolated worktrees merged in bead-number order.
3. **Footprint enforcement:** after each bead, run `git diff --name-only` for its commit and compare against the declared footprint. Any file outside the footprint is a **hard failure** — the plan's disjointness guarantee is void. Revert or amend, fix the footprint in the plan, re-validate, re-plan the remaining waves.
4. **Wave gate:** before the next wave starts, run the **combined** test suite — every test specified by every bead in the wave, plus the project's full suite if one exists. Two beads can each pass in isolation and still conflict semantically. A red wave gate stops everything, exactly as a red bead stops the sequential loop.
5. **No bead is implemented outside the plan.** A newly discovered bead means: update the beads doc, re-run the affected reviews, re-plan.

## Hard Rules

- **The validator gates execution.** Claude's confidence in the plan is not a substitute for exit code 0.
- **Shared file ⇒ different waves.** No same-file parallelism, ever, regardless of how far apart the edits are.
- **Footprint drift is a hard failure**, not a warning.
- **Wave green before next wave** — the combined suite, not per-bead suites alone.
- **Degenerate plan ⇒ report the bottleneck file(s)** rather than silently running sequentially.
