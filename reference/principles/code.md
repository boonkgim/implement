# code

Stack-agnostic rules for the shape of the code itself — naming, structure, typing discipline,
and failure handling. Applies to any language or framework a slice happens to be written in.

## Rule 1: No Magic Business Values

A business value — a quota, a limit, a bonus amount, a threshold, a price — must never be a
literal sitting inside UI or API code. It must come from one shared, named source. A literal
`if (count > 50)` scattered across three files means the business changes the limit once and
two of the three call sites keep the old one, silently, with no error to catch it. Styling
constants, animation durations, array indices, and genuinely one-off values with clear local
context are not business values and are fine as literals.

```
DON'T: if (user.uploads.length >= 50) { reject() }
DO:    if (user.uploads.length >= LIMITS.MAX_UPLOADS) { reject() }
```

## Rule 2: Config Lives Where It Changes

A value fixed to the product — a bundle ID, a URL scheme, a deep-link prefix — is a code
constant, not an environment variable; it does not vary by deploy, so hiding it in env config
just adds a place for it to go missing. A value that varies by environment — a key, a
connection string, a feature flag — is an environment variable, not a code constant; baking it
into code means the only way to change it is a new deploy, and the only way to leak it is a
public repo. Sort every config value by "does this change when the environment changes," not
by convenience.

## Rule 3: DRY Without Over-Abstracting

Extract genuinely repeated logic into one function, hook, or component so a fix made once
applies everywhere it's copied. But three similar lines used once each are not a pattern yet —
an abstraction built for a single caller adds a layer of indirection nobody needed and makes
the one real change harder to find. When a UI or logic pattern is fixed in one place, check
whether the same pattern is duplicated three or more times elsewhere; if so, extract it there
too in the same change, not as a follow-up that never happens.

## Rule 4: Single Responsibility

A function does one thing with clear inputs and outputs; a component owns one UI concern; a
file covers one domain. A function that both validates and persists and notifies is a function
nobody can change safely, because every caller now depends on a bundle of side effects instead
of the one thing it asked for. If describing what something does needs "and," it is at least
two things.

## Rule 5: Minimal Coupling

Keep components independent enough to move or reuse without dragging their neighbors along.
Deep prop drilling — passing a value through four layers that don't use it just to reach the
fifth that does — means every intermediate layer breaks when the shape of that value changes;
put shared state in a context, store, or equivalent instead. Prefer composing small pieces over
inheriting from a shared base, since a base class changed for one subclass silently changes
behavior for every other one.

## Rule 6: Don't Defeat the Type System

In a typed language, an escape-hatch type — `any`, an untyped blind spot, a cast that silences
an error instead of fixing the mismatch it's flagging — doesn't remove the bug, it just moves
the crash from compile time to whatever user hits it in production. Use the real, specific type
or an explicit unknown that forces a check. Where a library or a code generator already
produces a type, use it directly instead of hand-writing a mirror interface: a hand-written
copy silently drifts from the real shape (usually by forgetting a field is nullable) and the
compiler has no way to catch the gap because it's checking against the wrong source of truth.

```
DON'T: const data = response as UserProfile   // masks whatever response actually is
DO:    const data: GeneratedUserProfile = response
       const name = data.name ?? ''            // handles the nullability the type says is real
```

## Rule 7: Search Before You Add

Before writing a new utility, function, or component, check whether the codebase already has
one that does the same job — a second implementation of the same idea drifts from the first the
moment either one gets a bug fix the other doesn't. Keep code that belongs to one feature next
to that feature, and code genuinely shared across features in one shared location; code placed
by habit instead of by ownership is the code nobody remembers to delete when the feature that
used it is gone.

## Rule 8: Pure Functions for Logic

Calculations, transformations, validation, and formatting should be pure: same input, same
output, no hidden reads of the clock, a random source, or global state. Inject anything that
varies — the current time, a random seed — as a parameter with a sensible default. A function
that calls the system clock internally can't be tested for "what happens at midnight on a leap
year" without mocking the clock; a function that takes time as a parameter just gets called
with that timestamp.

```
DON'T: function isExpired(item) { return Date.now() > item.expiresAt }
DO:    function isExpired(item, now = Date.now()) { return now > item.expiresAt }
```

## Rule 9: Construct Dates Explicitly

Never build a date from a date-only string with a generic date parser — the string carries no
timezone, so the same literal produces a different calendar day depending on where the code
runs, and a value that's correct in staging silently becomes "yesterday" in production. Build
local dates from explicit year/month/day components, and UTC dates through an explicit UTC
constructor. An API returning a date should return a full instant, not a truncated date-only
string that forces every caller to guess the timezone it meant.

## Rule 10: Bulk Operations Run Off the Request Path

A loop that deletes, emails, or processes many records synchronously inside a request handler
ties that request's success to the slowest item in the batch, and a timeout partway through
leaves the operation half-done with no record of where it stopped. Hand bulk work to a
background job, queue, or scheduled task instead, so the request returns immediately and the
work can be retried, resumed, or observed independently of the caller.

## Rule 11: Persist the Upload Before the Operation Completes

When a user uploads a file as one step of a multi-step operation, persist a reference to it as
soon as it lands — before the operation that uses it finishes — rather than only linking it in
at the end. If the later step fails, an upload persisted only at the end is simply gone with no
trace it ever existed; a reference created at upload time can be reconciled, retried, or cleaned
up as an orphan after a grace period instead of silently disappearing.

## Rule 12: Fail Loudly on Missing Configuration

Required configuration — a credential, a signing secret, a database connection — must throw on
startup when it's absent, not silently fall back to a mock or a default. A service that falls
back quietly looks like it's running correctly right up until the moment it processes real data
incorrectly, which is the worst time to discover the config was never set. Optional config —
analytics, a feature flag, cosmetic defaults — is the only kind allowed to default silently.

## Rule 13: Keep Business Logic Off the Client

Logic that decides an outcome — pricing, entitlement, anything that touches money or access —
belongs on a server the client can't tamper with, not in code running on a device the user
controls. A client that decides "this user gets the discount" and just tells the server the
result is a client an attacker can edit; the server must derive the decision itself from data
it trusts, using the client only to describe intent.

## Rule 14: Trust the Authoritative Source, Not a Local Assumption

When code processes a value that another system is authoritative for — an amount, a currency, a
timestamp, an identity — take that value from the authoritative response itself, never from a
seed value, an internal constant, or what the code assumed it would be. A handler that assumes
the amount instead of reading it from the source of truth is correct only until the two
disagree, and by then the mismatch has already been acted on.

## Rule 15: Sweep Every Reference When You Rename or Remove a Shared Identifier

A URL, route, exported symbol, or config key referenced from more than one place needs every
reference found and updated in the same change a rename or removal is made — not a matching
update to the first three places it's obviously used and a silent break in the fourth.
Navigation, footer, sitemap, and cross-link data files are the highest-leverage case: they
appear on every page, so one stale reference in one of them is a site-wide broken link, not a
local one. Search the whole project for the old identifier before calling the rename done.

## Rule 16: Split Monolithic Registries

A single data, config, or content file that grows past a few hundred lines or holds more than a
handful of similar entries becomes a file where every change collides with every other change
in progress, and finding one entry means scrolling past the rest. Split it into one file per
entry with a barrel file that re-exports the collection, so each change touches one small file
and consumers that need the whole set still get it from one import.

## Rule 17: No Silent Error Swallowing

A `catch` that discards its error, or logs it only to a sink nobody watches, is a failure that
never happened as far as anyone downstream can tell — "nothing failed loudly" is not the same
as "nothing failed." Best-effort side effects (an alert, an analytics call, a cache write) are
allowed to fail without rolling back the primary operation, but the failure itself must still
reach an observable, consistently-labeled log — not be swallowed because delivery was optional.
Confirm the sink it's logged to is actually enabled; a log written to a disabled pipeline is
worth exactly as much as no log. Cover every path that can fail the same way — a missing
config, a returned error result, and a thrown exception should all reach the same marker, not
just the one the happy path anticipated.

```
DON'T: try { await sendAlert(x) } catch { /* best effort */ }
DO:    try { await sendAlert(x) }
       catch (err) { logger.error('ALERT_FAILED', { stage: 'send', err }) }
```

## Not this

- A limit, price, or threshold typed directly into a component or handler.
- `any`, an unchecked cast, or a hand-rolled interface mirroring a type the codebase already generates.
- A new utility written without checking whether one already does the job.
- A bulk delete or bulk email loop running inline inside a request handler.
- A `catch` block with nothing in it, or logging that goes to a sink nobody monitors.
- Config that falls back to a mock instead of throwing when a required value is missing.
