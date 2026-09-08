# Reconciliation — the plan and the code, checked against each other

Read once per recursive call, after every slice at that level has either landed or failed —
never mid-loop. A plan written before any code existed will have drifted from what actually
got built, in both directions, and neither document is automatically right.

## Compare

For every planned item at this level, and every file the slices under it actually touched,
find which of these it is:

| Category                    | Meaning                                                  |
| ---------------------------- | --------------------------------------------------------- |
| `PLANNED_NOT_IMPLEMENTED`    | In the plan; no slice built it (classified horizontal, failed, or simply never scheduled). |
| `IMPLEMENTED_DIFFERENTLY`    | Built, but the approach diverged from the plan's Design line. |
| `IMPLEMENTED_NOT_PLANNED`    | Code exists that no task in the plan called for.          |
| `PLAN_INCORRECT`             | The plan specified something that was wrong on its face — not a build failure, a planning one. |

## Recommend, don't silently pick

For each discrepancy:

- **UPDATE_PLAN** — the code is right; the plan should say what actually got built.
- **UPDATE_CODE** — the plan is right; recurse §4 of `SKILL.md` once more on just the affected
  files, under the same three-pass limit, to bring the code in line.
- **NEEDS_DISCUSSION** — neither document settles it. Put it in the report; do not guess.

## The brief still outranks both — but check this once, not at every level

If planning.md recorded a source brief, run one more pass rooted there rather than at the
plan: build the shipped surface list (routes, sections, screens actually built) and compare it
against the brief's, plus any PRD-approved additions. A shipped surface the brief never listed
and no approval covers is a discrepancy even when the code matches the plan exactly — the plan
itself may have drifted from the brief, and reconciling code to a drifted plan hides that.
Never resolve one of these by editing the brief or the PRD (`planning.md`, "The brief is
upstream"); it is always `UPDATE_CODE` when the approved intent is clear, `NEEDS_DISCUSSION`
otherwise.

**Run this pass only at the root call, after the whole tree has returned** — never at an
intermediate level. A nested call's shipped surface is only ever part of the brief by design
(the rest belongs to sibling subtrees, or hasn't run yet); comparing a subtree's own slice of
the work against the brief's *entire* page list produces false divergence on every surface
this subtree was never responsible for. Every other section on this page runs at every level,
including nested ones — this one section does not.

## When to stop reconciling

Once, at each level, after that level's slices are done. A single `UPDATE_CODE` pass is a
normal part of the run; a second round of the same discrepancy reopening is not a bigger
reconciliation pass — it's the halting rule in `SKILL.md` §5 applying to this level's own
work, and the level reports itself failed rather than reconciling a third time.
