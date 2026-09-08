# Planning — turn a task into a plan, a plan into slices

Read by the plan step of every recursion, at every depth. The same rules apply whether the
task in front of you is the whole PRD or one slice three levels down — the only thing that
changes is how much of it there is to plan.

## 1. Read whatever exists

In order of authority, read what the repo actually has — none of these are required, and this
skill plans from whatever subset exists:

| Input                  | Authority                                                          |
| ----------------------- | ------------------------------------------------------------------ |
| A source brief, if the PRD marks one authoritative | Highest. See "The brief is upstream" below. |
| The PRD                | What should exist, and what each capability must do.               |
| The data model         | What is stored, and which invariants the store itself must hold.   |
| The mock UI, as built  | What the user actually sees and can reach today.                   |
| A plain task, bug report, or the conversation itself | Lowest, and the only input a small task needs. |

If none of the richer documents exist, the conversation *is* the brief — a task described in
one message is still plannable, just at a smaller scope. Do not demand a PRD before doing a
bugfix.

Where the PRD and the mock disagree, **record the disagreement in the report and follow the
PRD** — a screen missing from the mock is usually an unfinished mock, not a cancelled
requirement, and resolving it silently is how a capability disappears.

### The brief is upstream

If a PRD names a `Source:` brief and marks it authoritative, that brief outranks the PRD, and
the PRD outranks this plan. Read it in full, not summarized — a brief is short, and a summary
is exactly how a load-bearing sentence goes missing.

Two rules follow, and they hold at every depth of recursion, not just the top:

- **Structure conformance.** The plan's surfaces — pages, sections, endpoints — must match
  the brief's. A surface the brief doesn't list is allowed only if the PRD records it as an
  approved addition. An unbriefed surface with no approval is not a plan detail to fix later;
  it doesn't get planned.
- **Upstream documents are read-only.** No task, no fix, no reconciliation may edit the brief
  or the PRD to make the code agree with it — not for a genuine conflict, not for a typo. A
  conflict between what's built and what's briefed is a reported discrepancy for the user to
  resolve (see `reconciliation.md`), never a silent edit upstream. A subagent that finds
  itself editing a brief to make its own work pass has inverted which document is the source
  of truth — stop and say so in the report instead.

## 2. Produce a capability list

Each entry carries the capability in the user's words, the screens it touches, the data it
touches, and its **acceptance criteria** — stated as an observable outcome, because that is
what becomes the test in §4 of `SKILL.md`. A criterion that cannot be checked by running
something is not yet a criterion.

**Never fabricate a fact to fill a slot.** A capability's copy, numbers, or claims that aren't
in the brief, the PRD, or a verifiable source don't get invented to make the plan look
complete — mark them `NEEDS: <what's missing>` and carry that into the report. A number typed
to sound plausible is a defect the reader has no way to catch later.

## 3. Classify, then decide: one slice, or many

Classify by where the risk lives:

| The risk lives in                                                                          | Slice      |
| ----------------------------------------------------------------------------------------- | ---------- |
| A pipeline that has never run — a new provider, a new secret in production, a new inbound endpoint, a new deploy target | horizontal |
| Integration between layers that already work                                              | vertical   |

**Horizontal work is out of scope for an unattended run**, at any depth. A pipeline is proven
by running it in isolation and watching production, which no recursive call of this skill can
do. List it in the report as work remaining; never attempt it.

Then size what's left:

- **Small enough to build in one pass** — go straight to §4 of `SKILL.md` (test → code →
  critique → fix → gate) with this plan.
- **Too big for one pass** — break it into vertical slices and recurse §2 of `SKILL.md` once
  per slice. A slice that is *still* too big gets broken again, by the same rule, at the next
  depth. There is no separate "slice planning" procedure — it is this section, run again on a
  smaller input.

"Too big" has no fixed line; use the right-sizing test in §5.

## 4. Order the slices

1. **Dependency first.** A slice that reads what another slice creates comes after it.
2. **Breadth before depth.** Complete one path through the application before deepening any
   part of it, so a run cut short leaves a usable app rather than a set of half-features.
3. **Mark independent slices as independent.** That is what lets a run continue past one
   slice's failure, and what lets independent slices run in parallel instead of in sequence.

## 5. Write the plan to a file

Not a gate — what survives a context reset, and what a person reads afterward to find out
what happened. There is **one plan file for the whole run**, at the location `SKILL.md` §0
names, shared across every depth of the recursion — a slice three levels down adds its section
to the same file the top-level call started, under its own heading, rather than starting one
of its own. **Only the orchestrator ever writes to it.** A planning subagent proposes its
unit's section and returns it via the normal file-based handoff (`loop.md`'s frontmatter
convention applies here too); the orchestrator is what actually appends it to the plan file,
sequentially, one unit at a time — which is what keeps two independent slices planning in
parallel from racing on the same file.

One task per leaf item:

```markdown
- [ ] **S{depth}.{seq}** <capability, stated as an action>
  - Source: <PRD section / brief section — an item that cites nothing is either infrastructure,
    labeled as such, or an invention, and inventions don't get planned>
  - Scope: <keys from scope-map.md — code, testing, db, ux, ...>
  - Files: <paths this slice will touch, to scope a subagent's exploration>
  - Acceptance: <the observable outcome that becomes the failing test>
  - [ ] Verify: <how — CLI, a rendered check, a test file>
  - Proof: <what a reviewer can check without re-running the work>
```

- **Right-size each leaf.** Completable in one pass of the loop — roughly hours, not days. A
  task whose acceptance criteria need three unrelated tests is undersliced; break it again.
- **Dependencies are explicit**, by ID: "requires S1.2", not implied by reading order.
- **A task with no Source line is either infrastructure or an invention.** Say which. An
  invention doesn't get planned — it goes to Open questions instead.

### Structural rules for repetitive content

Two patterns catch problems that only show up once a plan has several similar entries in it —
check for them before the plan is final, not after the tenth near-duplicate file exists:

- **Shared constants, at three or more.** If the plan creates three or more entries that share
  identical field values — the same category, the same template, the same default copy —
  plan a shared-constants file first, and have every entry import from it. A duplicated
  literal in entry four is a typo waiting to diverge from entry one.
- **Individual files, past five.** A registry, config, or content collection with more than
  five entries gets one file per entry plus a barrel index from the start, not a single
  growing file. This is what keeps diffs reviewable and lets independent slices touch
  different entries without conflicting.

## 6. Open questions

Everything that didn't get planned: an unbriefed addition with no approval, a claim with no
source, a scope no reference doc in this skill covers, a genuine ambiguity. This section is
not a failure of planning — it's the plan being honest about what it didn't decide for you.
