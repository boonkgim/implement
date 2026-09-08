# testing

This doc governs how to write and structure a good test — what makes it real, isolated,
specific, and worth keeping. It does not own verification, proof, or the gate; that
mechanics lives in `reference/loop.md` — read this doc for how to write the test that
mechanics runs.

## Rule 1: Prove the test can fail

Write the test before the code it tests, run it, and read the failure. A test that has
never failed for the reason you expect is unverified — it might pass regardless of what
the code does, because of a typo in the assertion, an import that resolved to a stub, or
a mocked boundary that swallowed the real call. Then write the minimal code that makes
it pass, nothing more. A bug fix starts the same way: reproduce the bug as a failing test
before touching the fix, or the "fix" is a guess about what was wrong. A test that passes
on its first, unmodified run is not evidence of anything — break it on purpose, confirm
it goes red, then restore it, before you trust it. This is the same discipline loop.md's
gate enforces structurally, one level down: at the level of a single test, not a run.

```
// red flag: has this ever actually failed?
expect(parseAmount("12.50")).toBe(1250)
```

## Rule 2: Draw the unit/integration line by dependency, not by file

A unit test exercises logic behind mocked boundaries — no real network call, database
connection, filesystem access, or clock — and runs in milliseconds; if it needs a
timeout, something is missing a mock. An integration test crosses a real boundary (a
real database, a real call to a real or containerized service) and proves the seam
itself, not the logic behind it — the logic is already proven by the unit tests under
it. Testing a database operation as a "unit test" gives you the worst of both: slow and
flaky, while a mocked version would have been faster and an honest integration test
would have actually proven the seam. Decide what a test is by what it touches, not by
which directory it sits in or what its file is named.

## Rule 3: Isolate every test from every other test

A test produces the same result regardless of what ran before it, after it, or beside
it. Give test data identifiers that cannot collide under parallel runs — random or
time-based, never hardcoded — and build fresh state per test instead of sharing a
suite-level variable that later tests mutate. Anything a test creates in shared state
(rows, files, queued jobs) it deletes when done, in dependency order — child records
before parents — because data left behind corrupts every test that runs after it, not
just itself. Tests that mutate a shared resource (a real database, a shared queue) run
sequentially unless they were deliberately designed to be parallel-safe. A suite that's
flaky under parallelism doesn't have bad luck, it has an isolation bug.

```
// collides the instant two test runs overlap
const userId = "test-user-1"

// safe under parallel runs
const userId = `test-${randomId()}`
```

## Rule 4: Fail loud when a dependency is missing, never skip silently

A test that needs an environment, service, or credential it doesn't have throws an
actionable error — naming what's missing and how to provide it — rather than skipping.
A skipped test still reports green, and a green suite that quietly ran fewer checks than
it claims is a false signal: the gap surfaces only once the untested path breaks for
real. This is distinct from the gate's own count check in loop.md, which catches a suite
that silently dropped a whole file; this rule is about why no individual test should let
itself become the dropped one.

```
// silent — reports green either way
if (!process.env.DATABASE_URL) return

// loud — fails with a next action, not just a red mark
if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL not set. Run: <the command that sets it>")
}
```

## Rule 5: Automate before you concede "manual"

Before marking a scenario manual, check whether the same state is reachable by
injecting it directly: insert a row with an already-past timestamp instead of waiting
for something to expire, generate a token with a past expiry instead of waiting for one,
read a side-effect (a stored code, a queued message) from wherever it actually landed
instead of eyeballing it. Most scenarios that look like they need a human are state
nobody tried hard enough to construct. What's left after that — a flow with no
programmatic hook, an external dashboard, a physical device — is genuinely manual; keep
it out of the automated suite rather than let a faked-automated version buy false
confidence. Rule 11 covers how that remainder gets written up.

## Rule 6: Assert exact inputs and exact outputs

A test's name and its assertion name the specific value going in and the specific value
expected out, not a category. "Handles invalid input" tells the next reader nothing
about what was tried; `parseAmount("abc") throws InvalidAmountError` tells them exactly
what broke the day it stops being true. One assertion per case — a test that checks five
things under one description reports a single pass/fail for five different failure
modes, and whichever breaks first hides the other four.

```
// vague — tells you nothing when it fails
test("validates email")

// specific — the failure is legible from the name alone
test('validateEmail("user@example.com") -> true')
test('validateEmail("not-an-email") -> false')
```

## Rule 7: Cover the boundary and the error path, not just the happy path

Every branch, every boundary (empty, one item, the maximum, one past it), and every
error path (invalid input, missing data, a dependency that fails) gets its own case.
Derive the list from the code and the spec — the actual branches, the actual domain
rules — never from the tests that already exist; summarizing an existing suite just
re-certifies whatever gap it already had. A suite that only exercises the path a
developer was thinking about when they wrote the code has, by construction, never
exercised the path they weren't thinking about — which is where the bugs live.

## Rule 8: Never hand-copy the value under test

A test that checks a constant, a schema shape, or a computed value imports that value
from the source rather than retyping it. A test with a hand-copied expected value keeps
passing after the source value silently drifts, because both sides changed together —
they were never really two independent sides. If importing the real value drags in
something a test shouldn't pay for (a database connection, a network client), the fix is
to move that value into a dependency-free module, not to fork a second copy into the
test file.

## Rule 9: Colocate tests with the source they cover

A test file sits next to the source file it tests, one to one — never a centralized file
that tests everything in a directory. A centralized test file is the first thing that
goes stale: nobody thinks to update `all.test.ts` when they touch a function three
directories away, and coverage quietly rots without anyone noticing. Extend an existing
test file as a capability grows; don't fork a new one per version or per flag — a
version-pinned test file is wrong the moment the next version ships. Naming convention
is illustrative, not prescriptive: this skill's own tests, for example, might use a
`.test.ts` / `.integration.ts` suffix split so each kind can run independently — adopt
whatever convention your project already has, and never assume a specific runner.

```
src/billing/charge.ts
src/billing/charge.test.ts          # colocated, one to one

src/billing/charge.ts
test/all.test.ts                    # centralized — avoid
```

## Rule 10: When two paths can produce the same effect, test both orderings

If a result is reachable by more than one path — an async callback and a synchronous
confirmation, an original request and its retry — a test proves the effect applies
exactly once no matter which path lands first. Cover both orderings explicitly (A then
B, and B then A), and confirm replaying either path again is a no-op. Testing only one
ordering proves the convenient race, not the real one — production doesn't get to
choose which path arrives first. `principles/webhook.md`, when in scope, owns the design
rules for building a handler this way; this rule is what proves the design holds.

## Rule 11: A human-checkable script is linear and makes no judgment call

For the scenarios Rule 5 leaves genuinely manual, what a person runs by hand must be
executable top to bottom — no scrolling back, no "if this, then that," no choice of path
left open to the tester. The plan already decided which path, which input, which button
text; the script states it as one action and one expected result per line. A script that
hands the tester a decision the plan never made isn't a test, it's a request for the
tester to finish planning. If this repo has a companion skill that produces this
document, hand it what was actually built, per SKILL.md §7 — that skill owns the
document's full shape; this rule is the one property of it that belongs here, because
it's a property of the test, not of the report format.

## Rule 12: Confirm you're testing the build you actually produced

A green test can be testing stale output — a bundle that didn't rebuild, a codegen
artifact that didn't rerun, a dependency still pinned to the wrong version. Before
trusting a result, confirm the thing under test is the thing you just changed — a
timestamp, a content hash, a version string — not the previous run's leftovers. The same
check applies one level up, after a deploy: confirm the version that's live is the
version that was built, rather than assuming a deploy step that claimed success actually
shipped it. Release process itself — version bumps, store submission, rollout
percentages — is out of scope for this doc; only this one check, that the build matches
intent, is general enough to belong here.

## Not this

- A test suite that's green because it's skipping, not because it's passing.
- A "unit" test that opens a real database connection and just runs fast enough to pass.
- An assertion that only checks that the code didn't throw.
- A manual test script that tells the tester to "verify it works as expected."
- A centralized `all.test.ts` nobody remembers to update.
- A fix applied to reach green without ever having watched the test fail red first.
