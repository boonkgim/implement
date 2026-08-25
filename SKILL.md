---
name: implement
description: Build everything the design documents describe, slice by slice, with no human between the slices. Use when a PRD, a data model and a mock UI are committed and the application has to be made real. Owns discovery, scope, slice order, the per-slice loop of test → implement → critique → fix → gate, the commit, and the report; delegates every file change to the repo's own layer skills. Application-agnostic — it discovers the repo's layers, gates and skills rather than assuming them. Stops at the local gate; it never deploys.
---

# implement — hand over the work, get back a report

**Shape: orchestration.** It owns scope, order, gates, the commit and the report, and no
mechanics whatsoever — every rule about how code is written lives in a skill it names rather
than in this file. See `skill-shapes`.

The design documents are the specification. This skill turns them into committed, tested code
without being asked a question in between, and returns one report that says what landed, what
did not, and what was never attempted.

## Owns / never touches

- **Owns:** the scope, the slice order, the loop, the commit, and the report.
- **Never writes code itself.** Every file change is made by a subagent that has loaded the
  skill owning that file. Delegation is what keeps a rule enforceable across a long run.
- **Never deploys.** The run ends at the local gate and the commit. A production gate needs
  credentials this run must not hold, and an observation only a person can make.
- **Never commits a slice that is not green end to end.**

## 1. Discover the repo

Before scoping anything, establish four things and record them in the plan:

1. **The layers and their forced order.** Which skills own which directories, and what forces
   the order between them — usually a generated artifact, where one layer's output is the
   next layer's input. Never reorder a forced order; it is a dependency, not a preference.
2. **The two gates.** Most repos have both, under different script names, and running the
   slow one on every pass is what makes a run take all morning.
   - The **inner gate** — the slice's own tests, plus lint and types over the changed scope.
     This runs on every pass of the loop.
   - The **full gate** — the whole suite, integration included. This runs once, after the
     loop goes green, and the commit waits on it.

   Take both commands from the repo's scripts, verbatim, with the directory they run from.
   If the repo declares only one, that one is both gates, every pass pays for the whole
   suite, and the report says so.
3. **The cross-cutting and rules skills.** Two kinds load *in addition to* a layer skill
   rather than instead of it: cross-cutting skills that span layers — authentication,
   payments, mail — and rules skills that sit under them, holding what would survive a fork.
   Rules skills own no directory, so nothing about a slice's file paths will surface them;
   find them by following the references out of each layer skill, and name them in the brief
   for every subagent that touches that layer.
4. **What must be running.** Containers, seeded schemas, stub services, and the command that
   starts each.

If the repo declares none of this, **stop and say so.** This skill orchestrates a repo's own
rules; it does not invent them, and a run against an undeclared repo produces code nobody can
review.

## 2. Scope

Read three inputs. Their authority is not equal:

| Input        | Authority                                                          |
| ------------ | ------------------------------------------------------------------ |
| The PRD      | what should exist, and what each capability must do                 |
| The data model | what is stored, and which invariants the store itself must hold   |
| The mock UI, as built | what the user actually sees and can reach today            |

Where the PRD and the mock disagree, **record the disagreement in the report and follow the
PRD** — a screen missing from the mock is usually an unfinished mock, not a cancelled
requirement, and resolving it silently is how a capability disappears.

Produce a capability list. Each entry carries the capability in the user's words, the screens
it touches, the data it touches, and its **acceptance criteria** — which become the tests, so
a criterion that cannot be stated as an observable outcome is not yet a criterion.

Then classify each by where the risk lives:

| The risk lives in                                                                        | Slice      |
| ----------------------------------------------------------------------------------------- | ---------- |
| A pipeline that has never run — a new provider, a new secret in production, a new inbound endpoint, a new deploy target | horizontal |
| Integration between layers that already work                                              | vertical   |

**Horizontal slices are out of scope for an unattended run.** A pipeline is proven by running
it in isolation and observing it in production, which this run cannot do. List them in the
report as work remaining, and build every vertical slice.

## 3. Order the slices

1. **Dependency first.** A slice that reads what another slice creates comes after it.
2. **Then breadth before depth.** Complete one path through the application before deepening
   any part of it, so a run cut short leaves a usable app rather than a set of half-features.
3. **Mark independent slices as independent.** That is what lets the run continue past a
   failure.

Write the plan to a file before the first slice runs, and update it as slices land. It is not
a gate — it is what survives a context reset, and what a person reads afterwards to find out
what happened.

## 4. The loop, per slice

**If the repo has a slice skill** — one that already owns layer order, the gates and the
commit for a single vertical slice — delegate each slice to it and let it run its own loop.
Do not reimplement what it owns; a second copy of the rules is a second thing to keep true.
Your job around it is the slice list, the halting rules in §5, and the report.

Otherwise run the loop yourself. Five subagents, each in fresh context, in order:

| #   | Subagent  | Loads                                                                            | Returns                                       | May write        |
| --- | --------- | -------------------------------------------------------------------------------- | --------------------------------------------- | ---------------- |
| 1   | test      | `code-test`                                                                       | a failing test, and the failure message        | test files       |
| 2   | implement | the layer skills for this slice in the repo's forced order, plus any cross-cutting skill it touches | code that makes the test pass | source files     |
| 3   | critique  | `critique`                                                                        | a ranked verdict                               | nothing          |
| 4   | fix       | the blocking findings, plus the same layer skills                                 | fixes                                          | test and source  |
| 5   | gate      | —                                                                                 | the inner gate's output                        | nothing          |

Rules that make the loop worth running:

- **Subagents are how "never commits" becomes structural.** Only this skill commits; a
  subagent cannot, because it is not the one holding the slice.
- **Hand the critic the diff and the acceptance criteria, and nothing else.** Not the
  implementer's summary, not its reasoning. A critic that reads the story reviews the story.
- **Assert counts, not exit codes**, at the gate. A suite that stopped collecting a file looks
  identical to one that passes.
- If the critic returns blocking, run 4, then 5. If the inner gate is red, run 3 and 4 again.
- **The inner gate runs every pass. The full gate runs once, after it.** The loop needs
  feedback in seconds, and the whole suite on every pass proves nothing about this slice that
  the slice's own tests did not already prove. Red at the full gate is a regression in a slice
  that was already green: hand it to the critic as a blocking finding and run 4 and 5 again,
  against the same three-pass limit.
- **Three passes maximum**, then the slice has failed. A fourth pass on the same defect is a
  loop, not progress.
- **Commit only when the full gate is green**, and never start the next slice with this one
  half-green.

## 5. When a slice will not go green

1. Move the slice's work to a branch off the last green commit, named for the slice, and
   reset the working line back to that commit. The diagnostic survives; the main line stays
   green.
2. Record the slice as failed, with the critic's last verdict and the gate's last output
   verbatim. A summary of a failure is not a failure report.
3. Skip every slice that depends on it. Continue with the independent ones.
4. **If more than a third of the planned slices fail, stop the whole run.** At that rate the
   fault is in the inputs rather than in the slices, and further attempts will not find it.

## 6. Report

One file, and the only thing a person should have to read:

- **What got built** — each slice, its commit, and what it now does.
- **What failed** — the branch, the last verdict, the last gate output.
- **Not attempted** — what was classified horizontal, with the pipeline each one would have to
  prove.
- **Never started** — what was in the PRD and never became a slice. Nothing else in the run
  will surface a capability that was simply not seen.
- **Non-blocking findings**, unfixed by design.
- **How to see it working** — the commands, in order, including whatever must be running.

If the repo has a `create-user-test` skill, run it last, against **what was built** rather
than what was specified. That script is how a person checks the run without reading the code.

## Never

- **Never deploy**, and never run a command whose name marks it as acting on production.
- **Never hand-edit a generated file.** Regenerate it. A generated file missing from the tree
  means codegen has not run.
- **Never widen scope mid-run.** A capability discovered during a slice goes in the report,
  not into the slice.
- **Never fix a non-blocking finding.** That is what makes the loop terminate.
- **Never weaken a test to reach green.** A failing gate is a result; a green gate bought by
  editing the assertion is a lie the report will repeat.
