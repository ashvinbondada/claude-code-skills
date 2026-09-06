---
name: beads-document
description: Use when asked to create a beads document — a migration plan where each bead is an atomic unit of work with dependency ordering, contract tests, and Playwright E2E validation steps.
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
| **Implementation** | All code required to complete this bead — full file contents or exact diffs, not pseudocode or descriptions |
| **Tests** | Full runnable test code (not descriptions) — every test that must pass before this bead is done |
| **E2E validation** | Playwright MCP steps if browser-testable; otherwise "N/A — not browser-testable at this step" |

**Implementation and Tests are mandatory.** A bead with no code is not a bead — it is a wish. An engineer must be able to implement and verify the bead entirely from what is written here, with no guesswork.

## Identifying Bead Boundaries

Split at every point where:
- A new deployable artifact appears (schema, route, component, config)
- A dependency chain would block parallel work
- A contract test boundary exists (unit → integration → browser)
- Rollback scope changes

Do NOT split by file type or by team. Split by **deliverable**.

## Writing Tests

Tests are **full runnable code**, not descriptions. Copy-paste into the test file and run — they must work.

Good test (full code):
```python
@pytest.mark.django_db
def test_maintenance_tickets_scoped_to_shift(api_client):
    old = timezone.now() - timedelta(hours=24)
    MaintenanceTicket.objects.create(room_number="501", description="Old", priority="low", status="open", created_at=old)
    MaintenanceTicket.objects.create(room_number="502", description="Current", priority="medium", status="open", created_at=timezone.now())
    room_numbers = [t["room_number"] for t in api_client.get("/api/shift/maintenance-tickets/").json()]
    assert "502" in room_numbers
    assert "501" not in room_numbers
```

Bad test (description only):
- "`GET /api/shift/maintenance-tickets/` only returns tickets from the current shift" — **this is a wish, not a test**

Every test must include: imports, setup, the call under test, and the assertion. No placeholders.

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
- Subsections labeled: **Accomplishes**, **Depends On**, **Implementation**, **Tests**, **E2E Validation**
- **Implementation** is a fenced code block with a language tag — full file contents or exact diff
- **Tests** is a fenced code block — full runnable test code, not descriptions
- E2E steps are a numbered list; N/A stated plainly

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bead too large — spans multiple deployables | Split at each new artifact |
| Implementation is pseudocode or prose | Write the actual code — full file or exact diff |
| Tests are descriptions, not runnable code | Write complete test functions with imports, setup, and assertions |
| Test missing imports or fixtures | Every test must be copy-paste runnable |
| E2E step says "check it works" | Name the exact element, URL, and assertion |
| Depends On left empty for non-root beads | Trace every prerequisite; omit only for bead 1 |
| No DAG at the top of the document | Every beads doc requires a parallelism DAG (mermaid) before the first bead |
| More than 5 beads in a parallel wave | Split into sub-waves; cap every wave at 5 |
