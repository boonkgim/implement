# db

Illustrated against Postgres + Drizzle — the invariants (normalization, FK semantics, MVCC
transaction behavior, `pgvector`) are Postgres's; the syntax (`pgTable`, `$defaultFn`,
`onDelete: 'cascade'`) is Drizzle's convenience and translates directly to another ORM's
equivalent. Applies whenever a table, column, index, constraint, or migration changes.

## Rule 1: Normalize, but only against a real anomaly

Every table needs atomic columns and a unique row (1NF), non-key columns depending on the
whole key (2NF), and no non-key column depending on another non-key column (3NF). The failure
this prevents is concrete: a repeating-group column that can't be queried or indexed, an
update that has to touch N rows to change one fact, or a column that silently drifts from the
value it's actually derived from. Don't normalize past the point where a join replaces a fact
you'll always read together with its owner — that's over-normalization, not correctness.

```sql
-- DON'T: repeating group, not queryable
tags text  -- "urgent,billing,vip"
-- DO: one row per fact
CREATE TABLE lead_tag (lead_id text REFERENCES lead(id), tag text);
```

## Rule 2: One table, one responsibility

A table earns its name from one purpose. Split by provider or method, not by a shared parent
concept with a discriminator hiding underneath: `auth_password`, `auth_oauth_google`, and
`auth_oauth_apple` instead of one `auth` table with a `provider` column and nullable
provider-specific fields. The failure this prevents is a table where half the columns are
NULL for any given row and a query has to know which half applies before it can trust them.

## Rule 3: Separate tables for entity subtypes

When an entity has real subtypes with different fields, give each subtype its own table
instead of one "god table" with a `type` column and a wide skirt of nullable
type-specific columns. Exception: every subtype shares identical fields, subtypes are
user-defined at runtime, or there are only ever a handful of rows. The failure this prevents
is a table where no single row uses more than a third of its columns, and every query has to
filter by `type` before any column is safe to read.

## Rule 4: No redundant data, except deliberate snapshots

Don't duplicate a fact across tables — reference it. The one sanctioned exception is a
point-in-time snapshot on an order or invoice (price, address, name at the moment of the
transaction), because that value must resist later changes to its source, not track them. Any
other duplication means two rows can disagree about the same fact, and nothing in the schema
says which one is right.

## Rule 5: Model the cardinality that holds for the entity's lifetime

Before placing any FK, answer three questions, not just "what does the first write look
like": Can the child exist without the parent — if yes, nullable FK with `onDelete: 'set
null'`. Can the parent ever have more than one child (retakes, revisions, repeat purchases) —
if yes, it's 1:N and the FK goes on the "many" side. Will the two ever have independent
lifecycles (one created anonymously, attached later) — if yes, decouple them now. A `NOT NULL
UNIQUE` foreign key is a **forced 1:1**: it locks the parent to always having exactly that
child and the child to never having a sibling. Reserve it for a relationship that is genuinely
and permanently 1:1 (a pure extension row, `user` ↔ `user_profile`). When in doubt, use a
nullable FK on the many side — it behaves like 1:1 today but doesn't foreclose retakes,
attach-later, or other-source records tomorrow. Retrofitting 1:N onto a shipped forced-1:1 is
a data migration; choosing it up front costs nothing.

```typescript
// DON'T: forced 1:1 — blocks retakes, blocks an assessment with no lead yet
assessmentId: text('assessment_id').notNull().unique().references(() => assessment.id)

// DO: nullable FK on the many side — one lead, many assessments, independent lifecycles
leadId: text('lead_id').references(() => lead.id, { onDelete: 'set null' })
```

## Rule 6: Junction tables only when both sides are genuinely many

A many-to-many join needs a meaningfully named junction table, a unique constraint on the FK
pair, and explicit `ON DELETE` behavior on both sides. Don't reach for a junction to model a
1:N — if one side has at most one owner, a nullable FK on the many side is the whole answer,
and a junction there is unused generality that every future query has to join through anyway.

## Rule 7: ULID for record IDs

Primary keys are ULIDs, generated with `$defaultFn(() => ulid())` — not UUID, not
auto-increment. Auto-increment leaks row count and creation order to anyone who can see two
IDs; a bare UUIDv4 has no time ordering, which costs you sortability and index locality for
free. Exception: use UUIDv4 for records whose ID appears in a public URL and where a
guessable, time-ordered ID is itself a leak (medical records, abuse reports).

## Rule 8: Naming conventions

Tables and columns are `snake_case` in the database; the ORM maps to `camelCase` in code —
don't hand-write camelCase into the schema. Table names are singular (`user`, not `users`) and
follow `{app}_{domain}_{variant}`, domain first, so related tables sort and group together:
`ai_payment_stripe` / `ai_payment_apple`, not `ai_stripe_payment` / `ai_apple_payment`. A
sub-table keeps its parent's domain as the prefix — `art_lead_otp`, not `art_otp_lead` — and
is named for the flow it serves, not just the mechanism it uses: `art_lead_otp`, never a bare
`art_otp` that has nowhere to go the day a second OTP flow (password reset, phone verify)
shows up. Quote reserved words (`"user"`, `"order"`). The failure this prevents is a schema
where related tables are scattered alphabetically and a generic name has to be renamed —
a migration — the moment a second instance of the same mechanism appears.

## Rule 9: Every table gets `created_at`; mutable tables get `updated_at`

`created_at` is `timestamp with time zone`, defaulted via `$defaultFn(() => new Date())`, on
every table with no exception. Any table whose rows change after insert also gets
`updated_at`. Without this, there is no way to answer "when did this happen" or "is this stale"
after the fact — and it can't be backfilled accurately once the gap is noticed.

## Rule 10: Timestamp vs. date, and never hand-parse a date-only string

An instant (something that happened) is `timestamp with time zone`. A date-only business value
with no time component (birthday) is `date`. A date-only value that still needs time
arithmetic (a billing anchor) is `timestamp with time zone` stored at midnight UTC, not
`date`. Never construct a date from a bare `'YYYY-MM-DD'` string — `new Date('YYYY-MM-DD')`
parses as UTC midnight and prints as the wrong calendar day in any timezone behind UTC. Build
it explicitly instead. On the way out, APIs return full ISO datetime strings
(`toISOString()`), never a truncated date-only string — truncating server-side throws away the
timezone information the client needs to render the correct day, and the loss can't be undone
downstream.

```typescript
// DON'T
new Date('2024-03-15')            // UTC midnight — wrong day west of UTC
res.json({ date: d.toISOString().split('T')[0] })   // truncated, loses tz info
// DO
new Date(2024, 2, 15)             // explicit local construction
res.json({ date: d.toISOString() })
```

## Rule 11: Store the input, not the output — unless the output must resist its input

For any derived value, ask whether it should *track* its source or *resist* it. Tracking
values (age from birthday, a displayed total from its line items) are never stored — compute
them on read, because a stored copy goes stale the moment the source changes. Resisting values
(an invoice's price at purchase) get a snapshot, because the whole point is that they must NOT
follow later changes to the source (see Rule 13). When a value must be snapshotted, store only
the irreducible, version-sensitive part of it: a value re-derivable by a stable
version-independent function (a sum, an average) or by a static lookup keyed off a field you
already store stays derived, not stored. Only a verdict that would itself change if the
deriving logic were re-tuned (a bucket, a threshold, a confidence flag) is a snapshot
candidate — and even then, store it alongside the immutable input plus a version tag
(`engine_version`), not in place of them. Better still, when the computation is
`fixedLogic(inputs, params)` and only the params tune, snapshot the params (an input, keyed by
a queryable scalar version tag resolving to version-controlled config) and recompute the
verdict on read — a stored result silently disagrees with a fresh recompute after a param
change, while stored params always reconcile. Only snapshot the actual output once the logic
itself, not just its params, can change past results. A snapshot is justified by "this must
stay frozen," never by "this saves recomputing" — you still maintain the code that renders it.

## Rule 12: `null` means "no data"; never a sentinel

Use `null` for "not applicable / not measured." Never use `0`, `''`, or a magic string to mean
the same thing — a stored `0` is indistinguishable from a real zero, and every reader has to
know which meaning applies to which column, which fails silently the first time someone
doesn't.

## Rule 13: Snapshot tables for anything with billing or audit weight

Billing, usage, and audit-log data lives in its own append-only table, storing the value as it
was at the moment of the event (`price_at_purchase`), never derived at read time from a
record that can still be edited or deleted. Follow Rule 11's discipline inside the snapshot
too — store the minimal version-sensitive verdict plus its immutable input and a version tag,
not the whole computed output. The failure this prevents: a refund or an audit report changes
retroactively because someone edited the product's current price, and there is no way to
reconstruct what the customer actually saw.

## Rule 14: FK delete behavior follows the table's compliance weight

Compliance-bearing tables (payments, subscriptions, invoices) use `onDelete: 'restrict'` and
`onUpdate: 'restrict'` — deleting or repointing the parent must fail loudly rather than
silently take financial history with it.
User data and content (profiles, preferences, sessions, conversations, messages) use
`onDelete: 'cascade'` — deleting the owner should clean up what only they owned. Optional,
independently-owned references use `SET NULL`. Getting restrict and cascade backwards is the
common failure: cascade on a payment table lets an account deletion erase the financial record
it's legally required to keep; restrict on a session table blocks an otherwise-clean account
deletion on rows nobody needed to keep.

## Rule 15: Account deletion is two transactions, never one that swallows an FK error

Attempt a hard delete first, in its own transaction. If a `restrict` FK on a compliance table
blocks it, fall back — in a second, separate transaction — to a soft delete plus PII masking.
Never catch the FK error and keep going inside the same transaction: Postgres aborts a
transaction on the first error, so every statement after the catch silently no-ops until
rollback, and the operator sees a fake success on a delete that half-happened.

## Rule 16: Soft delete needs a filtered index and a real retention job

A soft-deletable table gets a nullable `deleted_at`. Every query against it filters
`isNull(table.deletedAt)` — a query that forgets this returns rows the product has told the
user are gone. Add a partial index `WHERE deleted_at IS NULL` so that filter stays cheap as the
table grows, and run automated cleanup that hard-deletes past the retention window (e.g. 30
days) — a soft delete with no cleanup job is a retention policy nobody enforces.

## Rule 17: Wrap related writes in a transaction; mutate counters atomically

Multiple inserts/updates that must succeed or fail together go in one transaction. Functions
that may run inside a caller's transaction accept an optional `tx` (`const dbOrTx = tx ??
db`) so they compose instead of forcing a new connection. Mutate a counter with atomic SQL
(`sql\`${table.count} + 1\``), never read-modify-write from application code — two concurrent
requests reading the same value and writing back independently will lose one of the
increments, and the bug only shows up under real concurrency, never in a single-user test.

## Rule 18: Don't put a `Date` inside a raw SQL template

Drizzle's `sql` template literal does not serialize a JS `Date` correctly against the
postgres.js driver — use Drizzle's operators (`gt`, `lt`, `eq`, `isNull`, `and`, `or`) for any
date comparison instead of hand-writing SQL with an interpolated `Date` variable. Raw SQL
stays fine for column references, atomic increments, string/number literals, and JSONB
operators — the failure is specific to interpolating a `Date`, not to raw SQL in general.

## Rule 19: Migrate with files; never `db:push` past local dev

Production and CI/CD changes go through versioned migration files, never `db:push` — push has
no history, no review artifact, and no rollback story. Rank a migration's risk before running
it: `CREATE TABLE` and a nullable `ADD COLUMN` are safe as-is; a `NOT NULL` `ADD COLUMN` needs
a default or a backfill in the same migration; an `ADD COLUMN` carrying a FK needs the
reference data seeded *before* the FK constraint is added, or the migration fails against
existing rows; `ALTER COLUMN`, `RENAME`, `DROP COLUMN`, and `DROP TABLE` all get tested against
a production clone first, and a drop gets a backup first. Review every generated migration by
eye — a tool that combines a rename with a nullability or default change can emit SQL that
drops and recreates the column instead of altering it, silently losing data that a diff review
would have caught. Never run the migration-generation command unprompted; a schema change that
becomes a migration file is a decision for whoever's driving, not an automatic side effect of
editing `schema.ts`.

```sql
-- DON'T: FK with no seed data — fails on any existing row
ALTER TABLE order ADD COLUMN status_id text REFERENCES status(id);
-- DO: seed first, then constrain
INSERT INTO status (id, name) VALUES ('draft', 'Draft'), ('paid', 'Paid');
ALTER TABLE order ADD COLUMN status_id text REFERENCES status(id) DEFAULT 'draft' NOT NULL;
```

## Rule 20: Own resources by team from day one, even in a single-user MVP

Unless the product is genuinely single-user forever (a CLI tool, a single-player game), give it
`user` (identity), `team` (ownership), and `team_member` (join + role) from the first schema,
and have every owned table reference `team_id`, not `user_id` directly. Auto-create a personal
team on signup and never surface "team" in a single-user MVP UI. This is a relational pattern —
tables plus FKs — so it costs the same on SQLite/D1 or MySQL as on Postgres. The failure this
prevents is specific and expensive: retrofitting multi-tenancy onto a schema that hard-wired
ownership to `user_id` means rewriting every query and backfilling every row, while building it
in up front is free because the MVP never has to show it.

## Rule 21: Vectors join relational rows by a stable ID, not the reverse

When the product needs semantic search, RAG, or similarity dedup, store the embedding next to
(or in a 1:1 table with) the row it describes and index it for ANN search — `pgvector` with an
HNSW index is the default on Postgres, because it keeps relational data and vectors in one
database and one transaction. If the engine can't host vectors in-process (Cloudflare D1 has no
`sqlite-vec` support), keep the relational row in the primary database with a stable `id` and
put the vector in the dedicated store (Vectorize, Pinecone, Qdrant), joined by that same ID —
never let the vector store be the source of truth for a fact the relational schema also claims
to own.

## Not this

- A `type` column plus a wide field of nullable siblings, instead of a table per subtype.
- A `NOT NULL UNIQUE` FK reached for out of convenience, locking two entities together forever.
- `cascade` on a payments/invoices table, or `restrict` on a sessions table — backwards per Rule 14.
- Storing a computed total, age, or score instead of deriving it on read.
- `new Date('YYYY-MM-DD')` anywhere near user-facing date logic.
- A soft-delete table with queries that forget `isNull(deletedAt)`, or no retention job at all.
- `db:push` used for anything beyond local schema experimentation.
- A migration that adds a defaulted FK before the referenced rows exist.
