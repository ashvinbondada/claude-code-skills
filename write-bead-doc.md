---
name: write-bead-doc
description: Use when a single bead from a verified beads document needs a standalone implementation document written before coding begins. Takes one bead and produces a complete per-bead doc — input/output contracts, exact code, runnable tests, real risks only. An engineer picks it up and executes with zero follow-up questions.
---

You are writing a per-bead implementation document. This is not architecture prose. It is the exact specification an engineer executes from.

## Inputs you need before starting

Ask for these if not provided:
1. The verified beads document (or the relevant bead extracted from it)
2. The bead number and total bead count (e.g. "Bead 2 of 5")
3. The project's architecture doc path (to reference, not repeat)

## Output

A single markdown file saved to `docs/architecture/beads/bead-{N:02d}-{slug}.md`.

Self-contained — everything the engineer needs is in this one file. A fenced ```mermaid block only if a diagram is genuinely needed.

## Mandatory sections

### 1. Bead title and position
- Which bead this is (e.g. "Bead 2 of 5")
- What it depends on (prior beads, their output artifacts by name)
- What depends on it (next beads, what they expect you to deliver)

### 2. Input contract
Not prose. TypeScript interfaces or a field table with columns: field, type, source, required.

### 3. Output contract
Not prose. TypeScript interfaces or a field table with columns: field, type, destination, guaranteed-by-when.

Must be specific enough that the next bead's author can write their input contract from this alone — without reading the code.

### 4. Exact code changes
For every file touched, in order:
- Full absolute file path
- Change type: new file / edit / delete
- The actual code — not pseudocode, not "something like this", the real implementation

If the bead is large, split into subsections per file. No elision.

### 5. Tests
The exact test code that must pass before this bead is done. Full, runnable. Not a description of what to test — the actual test file content, with import paths, setup, and assertions.

Include the command to run them.

### 6. Risks
Only include a risk if all three are true:
- There is a realistic failure path (not just theoretical)
- The consequence is non-trivial
- The engineer needs to act on it

Format each risk:
> **What:** [specific thing that breaks]
> **When:** [condition under which it occurs]
> **Consequence:** [concrete impact]
> **Mitigation:** [exact action, not "consider X"]

If there are no risks meeting this bar, write: "No risks above threshold for this bead."

### 7. File footprint
Every file this bead creates, modifies, or deletes, written per the `file-footprint` skill — exact normalized paths, delivered and incidental files both. Start from the footprint declared in the beads doc, then confirm or correct it against the exact code in section 4. If they differ, list which files were added or dropped and why.

When running as a subagent, return this footprint to the orchestrator alongside the doc — `parallel-implementation-plan` consumes these reports.

## What NOT to include
- Rationale already in the architecture doc — add a `See: [arch doc path]` reference instead
- Low-severity observations
- Anything "nice to have"
- Theoretical risks with no realistic failure path
- Repeated context from other beads

## Depth check before saving

Before writing the file, verify all three are true:
1. An engineer can write every line of code without asking a follow-up question
2. The exact tests to run to verify completion are present and runnable
3. The next bead's author can write their input contract from section 3 alone
4. The file footprint in section 7 matches exactly the files section 4 touches — nothing more, nothing less

If any are false, fill the gap before saving.
