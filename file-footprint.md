---
name: file-footprint
description: Reference skill — the one canonical definition of a bead's file footprint. Consulted by beads-document, write-bead-doc, and parallel-implementation-plan whenever a footprint is written, confirmed, or collected; not invoked as a standalone task.
---

# File Footprint

## Overview

A bead's file footprint is the complete list of files it will create, modify, or delete. It is the input to wave scheduling: `parallel-implementation-plan` puts two beads in the same wave only if their footprints don't intersect, and its validator checks that intersection **as a string set operation**. Every rule below exists so that check is sound. Do not restate these rules in other skills — reference this one.

## Format Rules

The validator compares paths as strings. Two spellings of the same file silently fail to intersect, and the plan will schedule two conflicting beads in the same wave. Therefore:

- **Repo-relative, normalized paths only.** `src/api/routes.ts` — never `./src/api/routes.ts`, never absolute, always forward slashes. One spelling, used identically in the beads doc, the per-bead doc, and the plan JSON.
- **Exact file paths only — no globs, no directories.** `src/api/*` or `src/api/` intersects with nothing as a string, which quietly defeats the disjointness guarantee. If a bead touches five files in a directory, list five paths.
- **One path per entry**, as a plain list.

## What Counts as Touched

Created, modified, and deleted files all count. People remember source files and forget:

- Test files the bead adds or edits, and snapshot files those tests regenerate
- `package.json` **and the lockfile** when the bead adds, removes, or bumps a dependency
- Generated files: codegen output, migration indexes, API client stubs, compiled schemas — if the bead's steps regenerate it, it's in the footprint
- Config files the bead edits (CI config, env schema, build config)

## Delivered vs Incidental

Both categories are mandatory; label them if the consuming doc's format has room, but every path must appear either way:

- **Delivered files** — the modules, tests, and schemas the bead exists to produce.
- **Incidental files** — files the bead touches on the way. This is where hidden same-wave conflicts live: barrel/index files, route or plugin registries, `package.json`/lockfiles, migration indexes, config files, generated files, shared type files.

## How to Derive a Footprint

Don't introspect — check:

1. Walk the bead's implementation (outline or exact code): every file named there is in.
2. For each module the bead adds or renames, grep for barrel files and registries that would re-export or register it — those are incidental entries.
3. If the bead adds a dependency, add `package.json` and the lockfile.
4. If the bead touches anything a codegen step consumes, add the codegen outputs.
5. If the bead's tests snapshot rendered output, add the snapshot files.

## Drift

A footprint appears twice in the loop: declared in the beads doc (from the outline) and confirmed by `write-bead-doc` (from the exact code). The confirmed one wins, but drift must be **explained in the per-bead doc** (which files were added or dropped, and why) — unexplained drift is a finding for `parallel-implementation-plan` to raise, not something to silently adopt. At implementation time, `git diff --name-only` outside the confirmed footprint is a hard failure.
