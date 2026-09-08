# implement

**A task is a specification, whatever shape it's in — a one-line bug report or a full PRD.
This turns it into committed, tested code: plan it, recurse into vertical slices as deep as
it takes, run each one through test → code → critique → fix → gate, reconcile the result against
the plan, and report exactly what landed — without stopping to ask you anything in between.**

An [agent skill](https://agentskills.io) for Claude Code, Codex, and any other AI coding
agent that reads `SKILL.md`. Point it at a repo with whatever's on hand — a brief, an
approved PRD, a data model, a mock UI, or nothing but what you said in this conversation —
and it plans the work, breaks anything too big for one pass into ordered slices, builds and
proves each one through its own five-subagent loop, and returns one report saying what's
real and what isn't.

This repo's own [commit history](https://github.com/boonkgim/implement/commits/main) *is*
the skill's design record: every commit that shaped it carries the real prompt that drove
the change, in order, from the first version through the project it was adopted into. Read
it before you install anything.

## Why you would want this

An agent told to "build the rest of it" will happily start writing code against whatever it
last read, with no record of what it skipped, no gate between one file and the next, and no
way to tell you afterwards what it actually finished versus what it assumed. The fix is not
a longer prompt; it's a skill that owns the process and refuses to touch what it can't prove.

- **One procedure calls itself on every slice.** `implement(unit)` plans a unit, and if it
  doesn't fit in one pass, breaks it into ordered vertical slices and calls itself on each —
  a slice three levels deep runs exactly the same plan → scope → loop → reconcile steps the
  top-level task did. There's no separate "slice mode"; the recursion is the whole mechanism,
  however big the task or however deep it has to go to fit.
- **It never writes code itself, and never guesses which rules apply.** Every file change is
  made by a subagent that has loaded the repo's own layer skills, in forced order, plus
  whatever reference docs `reference/scope-map.md` says this slice touches — five
  stack-agnostic principle docs (code, testing, security, ux, webhook) plus stack-specific
  library docs (db, graphql, frontend, react-native, subscription, landing-page, seo) for
  whatever this slice's stack actually is. The critic loads the identical list the coder did,
  so a finding is always a cited rule, never the critic's own taste.
- **It never fabricates a fact to fill a gap.** A capability with no source in the brief, the
  PRD, or a verifiable source gets marked `NEEDS:` in the plan instead of invented;
  user-facing text with an unsourceable claim gets `<!-- UNVERIFIED: ... -->` in the code
  instead of a plausible number, and the critic treats an unmarked implausible claim as
  blocking, not a style note.
- **It refuses the risky half of the work, at every depth.** A slice is horizontal — a
  pipeline that's never run, a new provider, a new production secret, a new inbound endpoint
  — or vertical — integration between layers that already work. Only vertical slices run
  unattended, at any level of the recursion; horizontal ones are proven by running in
  isolation and watching production, which no recursive call of this skill can do. They land
  in the report as work remaining, not a guess.
- **Two gates, not one.** A slice's own tests, lint and types run on every pass of the loop
  (the inner gate); the whole suite runs once, after a batch of slices is green, and the
  commit waits on that (the full gate) — so a three-hour run doesn't pay for the whole suite
  on every single retry.
- **Three passes and it branches off, not just stops.** A fourth attempt at the same defect
  is a loop, not progress: the slice's work moves to a branch off the last green commit, the
  failure is recorded verbatim, and the run continues with whatever doesn't depend on it. If
  more than a third of a single level's slices fail this way, that level's whole run stops
  and reports up as failed — at that rate the fault is one level up, in the plan, not in the
  slices. (A level with only two or three slices means one failure is already over that
  third — that's intentional, not a bug in the math.)
- **It never deploys, and says so by name.** The run ends at the local gate and the commit.
  If the repo has a skill that owns the production gate, the report says which one and which
  commits to hand it; this skill holds no credentials that could act on production. The same
  goes for a user-test script: if the repo has a skill that writes one and a skill that runs
  it, both run last and get named in the report — by role, never hardcoded to a skill name.
- **The brief outranks the plan, and neither is this run's to edit.** If a PRD names a source
  brief and marks it authoritative, the brief outranks the PRD and the PRD outranks the plan;
  a mismatch between what's built and what's briefed is a reported discrepancy, never a
  silent edit to the document above it — not for a genuine conflict, not for a typo.

## Reference docs: principles and libraries

`reference/scope-map.md` is one table, read twice: at plan time, to decide which areas a
slice's task list must cover, and again at code, critique and fix time, to decide which
reference files a subagent loads before it touches a file. Same key, same trigger, same doc,
read by both — that's what keeps a plan's scope and a critic's checklist from drifting apart.

- **`reference/principles/*.md`** — stack-agnostic do's and don'ts: `code`, `testing`,
  `security`, `ux`, `webhook`. A principle doc never names a specific ORM, framework, or
  vendor; it states the invariant and lets the slice's own stack satisfy it. `code` and
  `testing` are always in scope — every slice writes code, and every slice's loop starts with
  a failing test.
- **`reference/libraries/*.md`** — stack-specific do's and don'ts, each written against one
  real stack: `db`, `graphql`, `frontend`, `react-native`, `subscription`, `landing-page`,
  `seo`. Install only the ones your projects actually need and delete the rest — a library
  doc assumes a stack, so an irrelevant one is dead weight, not a safety net.

The principle docs are never safe to delete; they're what makes a slice with no library match
still get reviewed against something.

Every subagent this skill runs — test, code, critique, fix, gate — writes its result to a
file under `.artifacts/implement/<run-id>/`, gitignored. The orchestrator reads only that
file's frontmatter: a verdict, a count, a path, never the body. That's what keeps a run's
context flat no matter how many levels deep the recursion goes — a slice four levels down
writes its own files and returns four lines.

## Install

Paste this to your agent:

```
install the skill at https://github.com/boonkgim/implement
```

It clones the repo and puts `SKILL.md` and `reference/` where your tool looks for skills.
To update it later, ask the same way, or `git pull` in the clone.

<details>
<summary>By hand</summary>

```bash
git clone https://github.com/boonkgim/implement.git

# Claude Code
ln -s "$PWD/implement" ~/.claude/skills/implement

# Codex
ln -s "$PWD/implement" ~/.agents/skills/implement
```

Symlink into a project's `.claude/skills/` instead to scope it to one repo. Other tools
read skills from their own location, and some take an upload; check yours.

</details>

A skill is instructions your agent will follow, so read `SKILL.md` before installing this
or any other. It is one file, plus the `reference/` files it loads on demand.

## Works with

`SKILL.md` follows the [Agent Skills](https://agentskills.io) open standard, so it loads
directly in any agent that reads the format — **Claude Code**, from `~/.claude/skills/`,
**OpenAI Codex**, from `~/.agents/skills/`, and any other tool with its own skills
directory. Where a tool does not read `SKILL.md` natively, paste it into the session or
drop it into the rules file that tool already reads, such as `AGENTS.md`. Nothing in it is
tool-specific — the whole skill is prose, application-agnostic by design: it discovers a
repo's layers, gates and cross-cutting skills rather than assuming any of them.

## Usage

Point it at a repo and whatever's on hand. In order of authority: a source brief the PRD
marks authoritative, the PRD, the data model, the mock UI as built, or — if none of the
richer documents exist — a plain task, a bug report, or just the conversation itself. None
of these are required; the skill plans from whatever subset actually exists, and a task
described in one message is still plannable, just at a smaller scope. Tools that support
invoking a skill by name take `/implement` directly; otherwise just ask to fix the bug,
build out what's left, or make the app real.

It first discovers the repo's own layers and their forced order, the two gate commands, any
cross-cutting or rules skills the repo's layer skills depend on, and what has to be
running — and stops and says so if it can't establish those, rather than inventing rules the
repo never declared. From there it plans a capability list with acceptance criteria,
classifies each item as horizontal (always out of scope) or vertical, and — if the unit
doesn't fit in one pass — breaks it into ordered vertical slices and recurses on each,
running the same plan → scope → loop → reconcile steps at every depth. Once a level's slices
have all landed or failed, it reconciles the plan against what was actually built and
recurses once more on anything the plan clearly got right and the code doesn't yet match.

If the repo has a skill that writes a user-test script and one that runs it, both run last,
in that order, against what was built rather than what was specified, and their reports are
named in this skill's own. It writes one report — what got built, what failed, what was
classified horizontal and never attempted, what was planned and never scheduled, any
reconciliation discrepancy still open, and how to see it working — to `IMPLEMENT-REPORT-<run-id>.md`,
or wherever the repo's own docs convention puts it, committed alongside the code it describes.

If this is useful, a ⭐ helps other people find it.

## When not to use this

- **The work is entirely horizontal.** A new payment provider, a new production secret, a
  new deploy target, any pipeline that's never run — this skill refuses exactly that
  category, at every depth, and reports it as remaining work rather than attempting it.
- **You need a human mid-run.** This is built for an unattended run between fixed
  checkpoints — plan in, report out — not a pair-programming session, and it will not stop
  to ask you anything partway through.
- **You need it to deploy.** The run ends at the local gate and the commit. It holds no
  deploy credentials by design, and hands off to a production-gate skill by name rather than
  running one itself.

## Author

Built by **Khur Boon Kgim** at [boonkgim.com](https://boonkgim.com), where I write about
practical AI for builders: AI agents, coding workflows, and shipping software.

## License

MIT. See [LICENSE](LICENSE).
