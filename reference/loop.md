# The loop — test, code, critique, fix, gate

Read by every recursion's base case: the point where a slice is small enough to build
directly rather than sliced again. §4 of `SKILL.md` names the five subagents; this file is
what each of them, and the orchestrator between them, actually does. `<run-id>` below is the
one `SKILL.md` §0 establishes once, at the top of the run — never re-derived per slice.

## File-based handoff

Every subagent writes its result to a file under `.artifacts/implement/<run-id>/`, gitignored,
one file per subagent call — this is scratch space for inter-subagent handoff, not evidence;
nothing here is expected to survive past the run. The orchestrator reads only that file's
frontmatter — never the body, never a subagent's raw output pasted back into its own context.

```yaml
---
type: test | code | critique | fix | gate
slice: S1.2
round: 1
verdict: PASS | FAIL
output_file: .artifacts/implement/<run-id>/S1.2-critique-r1.md
---
```

**The orchestrator holds at most ~40-50 lines of any subagent's output at a time.** Verdicts,
counts, and file paths stay inline; everything else is a path the next subagent reads for
itself. This is what lets the recursion go arbitrarily deep without the top-level context
filling up — a slice four levels down writes its own files and returns four lines.

## The five subagents

Each runs in fresh context. None sees another's reasoning — only its file output.

| # | Subagent | Loads | Returns | May write |
| - | -------- | ----- | ------- | --------- |
| 1 | test | `principles/testing.md`, plus any in-scope library doc's own testing notes | a failing test, and the failure message | test files |
| 2 | code | the repo's own layer skills, in forced order, plus every in-scope reference doc from `scope-map.md` | code that makes the test pass | source files |
| 3 | critique | the same reference docs subagent 2 loaded — the critic checks against exactly what the coder was told to follow | a ranked verdict, one line per rule | nothing |
| 4 | fix | the blocking findings, plus the same layer skills and reference docs | fixes | test and source |
| 5 | gate | — | the inner gate's output | nothing |

### Rules that make the loop worth running

- **Subagents are how "never commits" becomes structural.** Only the orchestrator commits; a
  subagent cannot, because it never holds the whole slice.
- **Hand the critic the diff and the acceptance criteria, and nothing else.** Not the
  implementer's summary, not its reasoning. A critic that reads the story reviews the story.
- **The critic cites a rule, not a feeling.** Every finding names the reference doc and the
  rule it checked, quotes the exact code or text it's about, and states the failure scenario —
  concrete input or state that produces a wrong result. A finding with no rule behind it is a
  preference, and only rule-backed, blocking findings reach the fixer.

  ```
  security.md Rule 4: FAIL — `db.query(`WHERE id = ${id}`)` — string interpolation into a
  query; a crafted id reads another account's row. Fix: parameterize the query.
  ```

- **Assert counts, not exit codes**, at the gate. A suite that stopped collecting a file looks
  identical to one that passes.
- **Two gates.** The **inner gate** — this slice's own tests, lint and types over the changed
  scope — runs on every pass. The **full gate** — the whole suite — runs once, after the loop
  is green, and the commit waits on it. Take both commands from the repo's own scripts,
  verbatim; if the repo declares only one, that one is both gates, and every pass pays for the
  whole suite.
- If the critic returns blocking, run 4 then 5. If the inner gate is red, run 3 and 4 again.
- **Red at the full gate is a regression in a slice that was already green.** Hand it to the
  critic as a blocking finding and run 4 and 5 again, against the same pass limit below — it
  does not get a fresh count.
- **Three passes maximum**, then the slice has failed. A fourth pass on the same defect is a
  loop, not progress.
- **Commit only when the full gate is green**, and never start the next slice — or the next
  sibling in a parallel batch — with this one half-green.
- **Never fabricate.** A code subagent asked to produce user-facing text with a factual claim
  it cannot source — a statistic, a capability, a price, a quote — writes `<!-- UNVERIFIED:
  <claim> -->` next to it instead of inventing a plausible number, and the critic treats an
  unmarked implausible claim as a blocking finding, not a style note.

## Verification and proof

Every task's `Verify:` line gets attempted by whatever method actually applies, in this order
of preference:

1. **A command** — a build, a typecheck, a targeted `curl`, a grep over generated output.
   Cheapest, and preferred whenever the criterion is checkable this way.
2. **A test** — the one subagent 1 wrote, or one like it, run and shown passing.
3. **A rendered check** — for anything with a browser-observable outcome: a headless browser
   script for DOM state, meta tags, computed style, or a screenshot; an interactive browser
   tool only when the check needs a real session (auth, cookies) or a person needs to see it
   live.
4. **Code review** — read the modified files and confirm they match the plan's Design line.
   Weakest; use it only when nothing above applies.

**Proof is what a reviewer checks without re-running the work.** Every verified task gets a
`Proof:` line in the plan file — a reproduce command and its key output, a screenshot path, or
both. A screenshot is evidence, not scratch — it lives next to the plan and the report (§0 of
`SKILL.md`: the repo's docs convention, or the project root), under `implement-proof-<run-id>/`,
committed, never under the gitignored `.artifacts/` tree a run's own handoff files use. Write
proof the reviewer can act on, not a description of having checked:

```
- Proof: `curl -s http://localhost:3001 | grep canonical` → `<link rel="canonical" .../>`
- Proof: screenshot at implement-proof-<run-id>/S2.1-hero.png, page: /pricing
- Proof: _Manual — requires the payment provider's dashboard; not auto-provable_
```

If a criterion needs a person (an external dashboard, a physical device, a judgment call),
say so and mark it manual rather than inventing a check that doesn't actually verify it.

## The gate, project-wide

This fires once every slice in the current parallel batch is individually green — not after
every single slice, and not only once for the whole run. Run the project's typecheck and lint
across the whole tree, not just the changed files. A violation in a file this batch touched is
this batch's bug, fixed now. A violation in a file nothing here touched is pre-existing debt —
fix it too, and say so under a separate heading, rather than let it silently accumulate under
the excuse that it predates the run.
