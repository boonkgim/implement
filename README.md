# implement

**Design documents are a specification. This turns them into committed, tested code, slice
by slice, without stopping to ask you anything in between.**

An [agent skill](https://agentskills.io) for Claude Code, Codex, and any other AI coding
agent that reads `SKILL.md`. Point it at a repo with an approved PRD, a data model and a
mock UI already committed, and it scopes the remaining work into slices, classifies each by
where the risk actually lives, and runs every safe one through a test → implement → critique
→ fix → gate loop — each step a fresh-context subagent — until the repo is real or the run
has told you exactly why it isn't.

This repo's own [commit history](https://github.com/boonkgim/implement/commits/main) *is*
the skill's design record: every commit that shaped it carries the real prompt that drove
the change, in order, from the first version through the project it was adopted into. Read
it before you install anything.

## Why you would want this

An agent told to "build the rest of it" will happily start writing code against whatever it
last read, with no record of what it skipped, no gate between one file and the next, and no
way to tell you afterwards what it actually finished versus what it assumed. The fix is not
a longer prompt; it's a skill that owns the process and refuses to touch what it can't prove.

- **It never writes code itself.** Every file change goes through a subagent that has loaded
  the skill that owns that file — a database change through the database skill, a UI change
  through the UI skill. Delegation is what keeps a rule enforceable across a long,
  unattended run; a single skill that tried to know every layer would drift from all of them.
- **It refuses the risky half of the work.** A slice is horizontal — a new payment provider,
  a new production secret, a new inbound endpoint — or vertical — integration between layers
  that already work. Only vertical slices run unattended; horizontal ones get proven by
  running in isolation and watching production, which no unattended run can do. They land in
  the report as work remaining, not as a guess.
- **Two gates, not one.** A slice's own tests, lint and types run on every pass of the loop;
  the whole suite runs once, after the loop is green, and the commit waits on that — so a
  three-hour run doesn't pay for the full suite on every single retry.
- **A test that passes immediately is treated as broken**, same as one that fails on a bad
  import. The critic reads the diff and the acceptance criteria and nothing else — not the
  implementer's summary — so a critique of the story never substitutes for a critique of the
  code.
- **Three passes and it stops.** A fourth attempt at the same defect is a loop, not progress:
  the slice's work moves to a branch, the failure is recorded verbatim, and the run continues
  with whatever doesn't depend on it. Past a third of the plan failing, the whole run stops —
  at that rate the fault is in the inputs, not the slices.
- **It never deploys, and says so by name.** The run ends at the local gate and the commit.
  If your repo has a skill that owns the production gate, the report says which one and
  which commits to hand it; this skill holds no credentials that could act on production.

## Install

Paste this to your agent:

```
install the skill at https://github.com/boonkgim/implement
```

It clones the repo and puts `SKILL.md` where your tool looks for skills. To update it
later, ask the same way, or `git pull` in the clone.

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
or any other. It is one file.

## Works with

`SKILL.md` follows the [Agent Skills](https://agentskills.io) open standard, so it loads
directly in any agent that reads the format — **Claude Code**, from `~/.claude/skills/`,
**OpenAI Codex**, from `~/.agents/skills/`, and any other tool with its own skills
directory. Where a tool does not read `SKILL.md` natively, paste it into the session or
drop it into the rules file that tool already reads, such as `AGENTS.md`. Nothing in it is
tool-specific — the whole skill is prose, application-agnostic by design: it discovers a
repo's layers, gates and companion skills rather than assuming any of them.

## Usage

Have a PRD, a data model and a mock UI already committed — this skill turns design
decisions into code, it doesn't make them. Tools that support invoking a skill by name take
`/implement` directly; otherwise just ask to build out what's left, wire up the backend, or
make the app real.

It discovers your repo's layers and their forced order, the two gate commands, any
cross-cutting or rules skills, and what has to be running, then stops and says so if it
can't establish those. From there it scopes the PRD into a capability list, classifies each
by risk, orders the vertical ones, and runs the loop — delegating to your project's own
slice skill if you have one, or running the five-subagent loop itself. It writes exactly one
report: what got built, what failed, what was never attempted, and what's left to deploy.

If this is useful, a ⭐ helps other people find it.

## Companion skills

This skill was written and tested alongside a small set of siblings, and names them where
it expects one:

- A **test-writing skill** for step 1 of the loop, and a **diff-review skill** for step 3 —
  illustrated in this skill's own history as `code-test` and `critique`. Any skill shaped
  the same way works; the loop doesn't depend on the name.
- A **slice skill**, if your project has one that already owns layer order, the gates and
  the commit for a single vertical slice — this skill delegates to it instead of
  reimplementing the same rules twice.
- A **production-gate skill**, named in the report rather than run, since this skill holds
  no deploy credentials by design.
- `create-user-test` and `run-user-test`, run last and in that order if present, so a human
  tester opens a report instead of a diff.

None of these are required to install; the skill degrades to running its own loop and
naming nothing in the report when they're absent.

## When not to use this

- **There's no PRD, data model or mock UI yet.** This skill builds what design documents
  already describe; it doesn't produce the decisions themselves.
- **You want to watch each change land.** This is built for an unattended run between fixed
  checkpoints, not a pair-programming session — it will not stop to ask you anything.
- **The work is entirely horizontal.** A new payment provider, a new production secret, a
  new deploy target — this skill refuses exactly that category and reports it as remaining
  work rather than attempting it.

## Author

Built by **Khur Boon Kgim** at [boonkgim.com](https://boonkgim.com), where I write about
practical AI for builders: AI agents, coding workflows, and shipping software.

## License

MIT. See [LICENSE](LICENSE).
