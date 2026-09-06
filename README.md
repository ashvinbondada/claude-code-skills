# claude-code-skills

Architecture-to-implementation loop skills for [Claude Code](https://claude.com/claude-code). These skills take a project from brainstorm to running code through a gated pipeline: architecture doc → beads (work units) doc → hostile review → per-bead implementation docs → bead-by-bead implementation.

## Skills

| Skill | What it does | When to use |
|---|---|---|
| `architecture-document` | Writes an architecture document | After brainstorming concludes, for new features, greenfield projects, or changes to an existing codebase |
| `architecture-to-html` | Converts an architecture markdown doc (`docs/architecture/*.md`) to a styled HTML file | Optional, outside the loop — only when you want to render or publish an architecture doc |
| `beads-document` | Creates a beads document — a migration plan where each bead is an atomic unit of work with dependency ordering, contract tests, and E2E validation steps | Once the architecture doc is reviewed and stable |
| `hostile-beads-review` | Adversarially reviews a beads document against the arch doc — dispatches the review dimensions as parallel subagents to find contract mismatches, ordering errors, structural assumptions, and scope creep | Before any per-bead docs are written, and again after they are |
| `write-bead-doc` | Turns a single bead into a standalone implementation doc with input/output contracts, exact code, and runnable tests | After the beads doc passes hostile review |
| `parallel-implementation-plan` | Extracts each bead's file footprint and computes a validated wave schedule so dependency-free, file-disjoint beads can be implemented by parallel agents | After the per-bead docs pass final hostile review, before implementation |
| `arch-to-implementation` | Orchestrates the entire loop above, end to end, with human gates at each review stage | When an architecture doc exists and you're ready to build |

## Installation

Each skill is a single markdown file with YAML frontmatter. Claude Code discovers skills as directories containing a `SKILL.md` file, so install each file into its own directory named after the skill.

**Per-user (available in every project):**

```bash
git clone https://github.com/ashvinbondada/claude-code-skills
cd claude-code-skills
for f in *.md; do
  [ "$f" = "README.md" ] && continue
  name="${f%.md}"
  mkdir -p ~/.claude/skills/"$name"
  cp "$f" ~/.claude/skills/"$name"/SKILL.md
done
```

**Per-project (checked into the repo, shared with your team):** do the same but target `.claude/skills/` inside your project instead of `~/.claude/skills/`.

Restart Claude Code (or start a new session) and the skills will appear in the skills list.

## Usage

### The full loop (recommended)

With an architecture document already written and reviewed, just ask:

```
Use the arch-to-implementation skill with docs/architecture/my-feature.md
```

The orchestrator runs every stage in order and stops at each gate:

1. Writes the beads doc (`beads-document`)
2. Hostile-reviews it against the arch doc (`hostile-beads-review`) — you resolve BLOCKERs and HOLEs before it proceeds
3. Writes one implementation doc per bead (`write-bead-doc`)
4. Re-reviews the per-bead docs for contradictions
5. Computes a wave schedule (`parallel-implementation-plan`) — a deterministic validator must pass before any code is written
6. Implements wave by wave — beads within a wave run as parallel agents on disjoint file sets; each wave's combined tests must be green before the next wave starts

### Individual skills

You can also invoke any stage on its own — either by asking naturally ("write a beads document for this arch doc") or by naming the skill explicitly:

```
Use the architecture-document skill to design the notification system
```
```
Use the beads-document skill on docs/architecture/notifications.md
```
```
Run hostile-beads-review on the beads doc against the arch doc
```
```
Use write-bead-doc for bead 3 in the beads document
```
```
Run parallel-implementation-plan on the per-bead docs
```
```
Use architecture-to-html to render docs/architecture/notifications.md
```

Because each skill's frontmatter describes when it applies, Claude Code will also pick them up automatically when the task matches — you don't always need to name them.

## Hard rules baked into the loop

- Nothing is implemented until the beads review is clean — a BLOCKER means the code would be wrong.
- Nothing is implemented until the parallel plan's validator exits 0 — the model's confidence in a schedule is not a gate; the script is.
- Beads are never combined; each bead is a discrete commit — even when implemented in parallel waves.
- Beads in the same wave never share a file; footprint drift during implementation is a hard failure, not a warning.
- BLOCKERs and HOLEs are resolved with a human; only RISKs are resolved autonomously.
- The architecture doc is the source of truth when documents conflict.
