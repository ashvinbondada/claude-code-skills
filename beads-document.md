---
name: beads-document
description: Use when asked to create a beads document — a migration plan where each bead is an atomic unit of work with dependency ordering, input/output contracts, a file footprint, and Playwright E2E validation steps.
---

# Beads Document

## Overview

A Beads Document expresses an architectural migration as a sequenced chain of atomic deliverables. Each bead has a precise contract: what it delivers, what it requires, how to verify it, and whether it is browser-testable.

## Bead Contract Structure

| Field | Rule |
|---|---|
| **Title** | Short imperative phrase — "Initialise Next.js project" |
| **Accomplishes** | 2–3 sentences: what is delivered and why it matters for the migration |
| **Depends on** | Bead numbers/titles that must be complete before this bead starts |
| **Input contract** | Exact types/shapes this bead consumes, and which prior bead (or existing code) delivers each — precise enough to diff against the predecessor's output contract |
| **Output contract** | Exact types/shapes this bead delivers — precise enough that every dependent can write its input contract from this alone |
| **File footprint** | Every file this bead will create, modify, or delete, written per the `file-footprint` skill — delivered AND incidental files, exact normalized paths |
| **Implementation outline** | Ordered planning-level steps: what changes, where. NOT full code — exact code is written once, later, by `write-bead-doc` |
| **Test intent** | Each behavior that must be proven: setup state, the exact call/condition, the assertion. Runnable test code is written later by `write-bead-doc` |
| **E2E validation** | Playwright MCP steps if browser-testable; otherwise "N/A — not browser-testable at this step" |

**Contracts and footprint are mandatory.** A bead whose contracts can't be diffed against its neighbours' is not a bead — it is a wish. This document is the planning source of truth: `write-bead-doc` must be able to produce the full implementation doc from a bead's entry plus the arch doc, with no guesswork. Exact code and runnable tests live only in the per-bead docs — never duplicate them here.

## Identifying Bead Boundaries

Split at every point where:
- A new deployable artifact appears (schema, route, component, config)
- A dependency chain would block parallel work
- A contract test boundary exists (unit → integration → browser)
- Rollback scope changes

Do NOT split by file type or by team. Split by **deliverable**.

## Writing Test Intent

Test intent is **precise, not runnable**. Each entry pins down setup state, the exact call under test, and the exact assertion — specific enough that `write-bead-doc` can turn it into runnable code without making a single decision.

Good intent (fully pinned down):
- Given one `MaintenanceTicket` created 24h ago (room 501) and one created now (room 502), `GET /api/shift/maintenance-tickets/` returns room 502 and does NOT return room 501

Bad intent (a wish):
- "the endpoint only returns tickets from the current shift" — names no setup, no exact call, no assertion

The runnable code — imports, fixtures, assertions — is written once, in the per-bead doc, by `write-bead-doc`. Never paste test code into the beads doc.

## Determining E2E Testability

A bead is browser-testable when a user-visible UI state exists after it completes.

**E2E validation format (when testable):**
1. `browser_navigate` to the relevant URL
2. `browser_snapshot` or `browser_take_screenshot` to capture state
3. Assert visible elements, text, or network responses

**Mark N/A when:** the bead only adds infrastructure (DB migrations, env vars, CI config, non-rendered modules).

## Parallelism DAG

**Every beads document MUST include a DAG** at the top, before the first bead card, showing which beads can run in parallel.

**Rule: no more than 5 beads in any parallel wave.** If a natural wave has more than 5, split it into two sequential sub-waves.

Render the DAG as a fenced ```mermaid `flowchart LR` block. Layout rules:
- Waves run left-to-right — group each wave's beads in a `subgraph` labeled `WAVE 1`, `WAVE 2`, etc.
- Each node shows the bead number and short title, e.g. `B03["03 · parser"]`.
- Edges connect dependency → dependent.

Example DAG structure for a 10-bead doc with waves [01] → [02,03,04,05,06] → [07,08,09] → [10]:
```
WAVE 1    WAVE 2              WAVE 3        WAVE 4
  01   →  02, 03, 04, 05, 06  →  07, 08, 09  →  10
```

## Output Spec

Save to: `docs/architecture/YYYY-MM-DD-<topic>-beads.md`

**Structure rules:**
- The DAG mermaid block renders at the top of the document, before the first bead
- Each bead is a `## Bead NN — Title` section
- Subsections labeled: **Accomplishes**, **Depends On**, **Input Contract**, **Output Contract**, **File Footprint**, **Implementation Outline**, **Test Intent**, **E2E Validation**
- **Input/Output Contract** as type definitions or field tables — exact, diffable
- **File Footprint** as a plain list of exact paths per the `file-footprint` skill
- **Implementation Outline** as an ordered list of planning-level steps — no code blocks
- **Test Intent** as a bulleted list of setup → call → assertion entries
- E2E steps are a numbered list; N/A stated plainly

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bead too large — spans multiple deployables | Split at each new artifact |
| Full code pasted into the beads doc | Code lives only in the per-bead docs — keep the outline at planning level |
| Contracts described in loose prose | Exact types/shapes, diffable against neighbouring beads' contracts |
| Test intent too vague to implement | Pin down setup state, the exact call, and the assertion |
| File footprint incomplete, or uses globs/directories | Follow the `file-footprint` skill: incidental files included, exact normalized paths only |
| E2E step says "check it works" | Name the exact element, URL, and assertion |
| Depends On left empty for non-root beads | Trace every prerequisite; omit only for bead 1 |
| No DAG at the top of the document | Every beads doc requires a parallelism DAG (mermaid) before the first bead |
| More than 5 beads in a parallel wave | Split into sub-waves; cap every wave at 5 |
