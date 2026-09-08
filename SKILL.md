---
name: implement
description: Turn a task into committed, tested code with no human between the plan and the result — a PRD, a data model and a mock UI if they exist, a one-line bug report if that's all there is. Plans, then runs test → code → critique → fix → gate as one recursive loop, breaking anything too big for one pass into vertical slices and running the same loop on each. Every test, code, critique and fix step scopes in the relevant do's and don'ts — general principles plus stack-specific library rules — from this skill's own reference files, so the critic checks the coder against exactly what the coder was told. Application- and stack-agnostic — it discovers the repo's layers and gates, and loads only the reference docs a slice's own scope calls for. Stops at the local gate; it never deploys.
license: MIT
---

# implement — plan, build, and prove it, slice by slice

**Shape: orchestration.** It owns the plan, the recursion, the gates, the commit and the
report, and no mechanics whatsoever — every rule about what good code, a good test or a good
critique looks like lives in a reference file this skill loads, not in prose repeated here.
(If your skill library sorts skills by shape — principle, practice, orchestration — this is
the orchestration kind: it owns process, never mechanics.)

A task — a PRD, a brief, a bug report, or just what you said in this conversation — is the
specification. This skill turns it into committed, tested code without stopping to ask
anything in between, and returns one report that says what landed, what didn't, and what was
never attempted.

## The recursion

One procedure, called on the whole task and then again on every slice it produces, as deep as
it needs to go:

```
implement(unit):
    plan := plan(unit)                      # §2 — read what exists, produce acceptance criteria
    if plan is horizontal:                  # a pipeline that has never run
        report as work remaining; stop
    if plan fits in one pass:               # the base case — see planning.md §3 for the test
        scope := scope-map lookup            # §3
        loop: test → code → critique → fix → gate   # §4, until green or failed
        return this unit's result
    else:
        slices := break plan into vertical slices, ordered   # planning.md §3-4
        for each slice, respecting order and independence:
            implement(slice)                 # the same procedure, on less work
        reconcile(unit, its slices' results) # §6
        return the aggregated result
```

A slice three levels deep runs exactly the same seven steps below as the top-level task did.
Nothing about §1-§7 changes with depth — only how much of the task is in front of it.

**Three words, one relationship.** The **task** is the whole input this skill was handed,
before any planning happens to it. A **unit** is whatever `implement()` is currently called
on — the task itself on the first call, a smaller piece of it on every call after. A **slice**
is what a unit gets broken into when it doesn't fit in one pass; a slice is itself the *next*
call's unit. Once a unit is small enough to build directly, `reference/planning.md` calls its
plan-file entry a **task item** — the leaf line a slice's own loop turns into a test. The word
"task" is doing double duty on purpose: the whole run's input and a plan's leaf line are the
same kind of thing at different scales, which is the point of the recursion.

## Owns / never touches

- **Owns:** the plan, the slice order at every depth, the loop, the commit, and the report.
- **Never writes code itself.** Every file change is made by a subagent that has loaded the
  reference docs this skill scoped in for it. Delegation is what keeps a rule enforceable
  across a long, deep run.
- **Never deploys.** The run ends at the local gate and the commit. A production gate needs
  credentials this run must not hold, and an observation only a person can make. Hand it back
  by name: if the repo has a skill that owns the production gate, the report says which one
  and which commits it should be given.
- **Never commits a slice that is not green end to end.**

## 0. Artifacts

**Establish the run-id once**, at the top of the very first call — today's date, plus a short
slug if a second run happens the same day — and thread it unchanged through every recursive
call this run makes. A slice three levels down writes into the *same* run's files as the
top-level call; it does not mint its own.

Two kinds of file, never confused:

- **Ephemeral, scratch.** Every subagent's raw output — test/code/critique/fix/gate results, a
  plan subagent's proposal, a reconcile subagent's comparison — goes under
  `.artifacts/implement/<run-id>/`, gitignored. It exists so the *next* subagent can read a
  prior one's work without the orchestrator holding it; it is not required to survive past the
  run.
- **Persistent, committed.** The plan file, the final report, and every proof screenshot a
  `Proof:` line points to never live under `.artifacts/`. Write them to the repo's own docs
  convention if it has one; otherwise to the project root as `IMPLEMENT-PLAN-<run-id>.md`,
  `IMPLEMENT-REPORT-<run-id>.md`, and `implement-proof-<run-id>/`. These are what a person
  reads and checks, and what git remembers.

**Whichever file a subagent writes to, the orchestrator reads only its frontmatter** — a
verdict, a count, a file path, never the body. See `reference/loop.md` for the exact
convention. This, not which directory a file lives in, is what keeps a run's context flat no
matter how deep the recursion goes: §2 and §6 below delegate to a subagent under this same
rule, exactly like §4 does; §1 and §3 are cheap, bounded lookups the orchestrator does inline
because there's nothing there large enough to threaten the budget; and §7 needs no subagent of
its own, because every step before it already handed back only a summary — assembling the
report is concatenating summaries you already have.

## 1. Discover the repo

Before planning anything, establish four things and record them in the run's plan file (§0):

1. **The layers and their forced order.** Which skills or conventions own which directories,
   and what forces the order between them — usually a generated artifact, where one layer's
   output is the next layer's input. Never reorder a forced order; it is a dependency, not a
   preference.
2. **The two gates.** Most repos have both, under different script names.
   - The **inner gate** — the slice's own tests, plus lint and types over the changed scope.
     Runs on every pass of the loop.
   - The **full gate** — the whole suite, integration included. Runs once a batch of slices
     is green, and the commit waits on it.

   Take both commands from the repo's own scripts, verbatim. If the repo declares only one,
   that one is both gates, and every pass pays for the whole suite — say so in the report.
3. **The cross-cutting and rules skills.** Skills that load *in addition to* whatever this
   skill scopes in — authentication, payments, mail, or any project-specific rule that would
   survive a fork. They own no directory, so nothing about a file path surfaces them; find
   them by following the references out of the repo's own layer skills, and name them in
   every subagent's brief for the layer they cover.
4. **What must be running**, and how to tell. Containers, seeded schemas, stub services, a dev
   server — check `package.json` scripts, framework config, or `.env*` files for the port
   before ever assuming `localhost:3000`. The command that starts each, and the check that
   confirms it already is.

If the repo declares none of this, **stop and say so.** This skill orchestrates a repo's own
rules; it does not invent them, and a run against an undeclared repo produces code nobody can
review. If the gap is missing tooling rather than an undeclared repo — no typecheck script, no
linter — and the repo has a skill that owns project setup, name it in the report instead of
inventing the missing configuration yourself.

## 2. Plan

Delegate to a fresh subagent that reads `reference/planning.md`. This is the recursive entry
point: it reads whatever inputs exist, produces a capability list with acceptance criteria,
classifies horizontal versus vertical, and — if the unit is too big for one pass — breaks it
into ordered vertical slices. It writes this unit's own section of the plan file (§0) and
returns a frontmatter summary; **only the orchestrator ever edits the plan file directly** —
a subagent proposes, the orchestrator applies, so two independent slices planning in parallel
never race on the same file.

**Horizontal work stops here, at every depth.** A pipeline is proven by running it in
isolation and watching production, which no recursive call of this skill can do. List it in
the report as work remaining; never attempt it, however small it looks.

## 3. Scope this unit of work

Read `reference/scope-map.md`. Before any subagent touches a file, know which reference
docs — general principles plus the library docs for whatever stack this unit actually
touches — apply to it, and name every one of them in every subagent's brief for §4. A slice
that skips this step is a slice whose critic doesn't know what it was supposed to check.

## 4. The loop: test → code → critique → fix → gate

Read `reference/loop.md` for the full mechanics — the five subagents, the file-based handoff,
verification and proof, the two-gate split. The rules that most shape the run:

- **Three passes maximum**, then this unit has failed (see §5).
- **The critic checks the coder against exactly the docs §3 scoped in** — never its own
  taste, never a doc nobody named.
- **Commit only when the full gate is green**, and never start the next slice, or a sibling
  in the same parallel batch, with this one half-green.

## 5. When a unit will not go green

1. Move its work to a branch off the last green commit, named for the unit, and reset the
   working line back to that commit. The diagnostic survives; the main line stays green.
2. Record it as failed, with the critic's last verdict and the gate's last output verbatim.
   A summary of a failure is not a failure report.
3. Skip every sibling that depends on it. Continue with the independent ones.
4. **If more than a third of a level's slices fail, stop that level's whole run** and report
   up to the caller as failed. At that rate the fault is in the plan one level up, not in the
   slices, and further attempts at this depth will not find it. At a small slice count this
   reduces to near-zero tolerance — two slices means one failure already stops the level — and
   that is intended, not a rounding error: a level small enough to have two or three slices is
   small enough that any failure in it is a large fraction of that level's own work.

## 6. Reconcile

Once, after a level's slices have all either landed or failed — never mid-loop. Delegate to a
fresh subagent that reads `reference/reconciliation.md`: compare what was planned against what
was actually built, categorize every discrepancy, and recurse §4 once more, under the same
pass limit, for anything the plan clearly got right and the code didn't yet match. Everything
else is a report line, not a silent edit to either the plan or the code.

**The brief-conformance pass — reconciliation.md's check against the brief's full surface
list — runs once, at the root call only**, after the entire tree has returned. A nested
call's own subtree covers only part of the brief by design; comparing it against the whole
brief's page list produces noise, not a finding.

## 7. Report

One file per top-level run — `IMPLEMENT-REPORT-<run-id>.md` or wherever the repo's own docs
convention puts it (§0), committed, never under the gitignored `.artifacts/` tree — and the
only thing a person should have to read:

- **What got built** — each slice, its commit, and what it now does.
- **What failed** — the branch, the last verdict, the last gate output, at whatever depth it
  failed.
- **Not attempted** — what was classified horizontal, with the pipeline each one would have
  to prove.
- **Never started** — what was in the plan and never became a slice. Nothing else in the run
  surfaces a capability that was simply not seen.
- **Reconciliation discrepancies** left as `NEEDS_DISCUSSION`.
- **Non-blocking findings**, unfixed by design.
- **How to see it working** — the commands, in order, including whatever must be running.
- **What is left to deploy** — the commits, and the skill that owns the production gate. A
  run that ends green and says nothing about the deploy reads as finished when it is not.

If the repo has a skill that writes a user-test script, run it last, against **what was
built** rather than what was specified — that script is how a person checks the run without
reading the code. Then, if the repo also has a skill that runs that script, run that: the same
check done by a machine, and what it finds is what nobody has to spend a tester's hour on.
Name its report in your own — a report written and never opened is the same as no report.

## Reference files

Loaded on demand, not upfront — a run that never touches GraphQL never opens `graphql.md`.

| File | Read at | Covers |
| ---- | ------- | ------ |
| `reference/planning.md` | §2, every depth | Intake, capability list, horizontal/vertical, slicing, the plan file format |
| `reference/scope-map.md` | §3, every depth | Which reference doc applies to which kind of change |
| `reference/loop.md` | §4, every depth | The five subagents, file-based handoff, verification, proof, the two gates |
| `reference/reconciliation.md` | §6, every depth | Plan-vs-code comparison and when to recurse once more |
| `reference/principles/*.md` | §4, when in scope | Stack-agnostic do's and don'ts: code, testing, security, ux, webhook |
| `reference/libraries/*.md` | §4, when in scope | Stack-specific do's and don'ts: db, graphql, frontend, react-native, subscription, landing-page, seo |

## Never

- **Never deploy**, and never run a command whose name marks it as acting on production.
- **Never hand-edit a generated file.** Regenerate it. A generated file missing from the tree
  means codegen has not run.
- **Never widen scope mid-run.** A capability discovered during a slice goes in the report,
  not into the slice.
- **Never fix a non-blocking finding.** That is what makes the loop terminate.
- **Never weaken a test to reach green.** A failing gate is a result; a green gate bought by
  editing the assertion is a lie the report will repeat.
- **Never edit a brief or a PRD to make the code, the plan, or a review pass.** See
  `reference/planning.md`, "The brief is upstream." A document above this run in authority is
  not this run's to change.
- **Never fabricate a fact in generated content.** A claim with no source becomes an
  `UNVERIFIED` marker in the code and a line in the report, never a plausible invention.
