# graphql

Assumes GraphQL as the API layer, with typed codegen (e.g. graphql-code-generator) turning
`.graphql` documents into typed query/mutation objects and result types. That's a convenience
this doc leans on for its DO/DON'T examples, not a hard requirement — if a project queries with
raw strings, every rule below except the codegen-specific ones (1, 3, 5) still applies. Client
cache examples reference urql's document cache; Apollo's normalized cache has the same failure
shapes under different names.

## Rule 1: Use Typed Codegen, Never Raw Queries

Every query and mutation is a typed document generated from the schema, imported and called
through it — never a raw `fetch` with an inline GraphQL string. Raw strings lose field-level
type checking, so a typo or a removed field is a runtime error instead of a compile error.

```
DON'T: fetch('/graphql', { body: `query { me { id name } }` })
DO:    const { data } = await client.query(GetMeDocument, {})
```

## Rule 2: Organize Queries and Mutations in Dedicated Files

Define every query and mutation in its own `.graphql` file, grouped under `queries/` or
`mutations/`, never written inline inside a `.ts`/`.tsx` file as a template string handed to
codegen from there. Inline definitions scatter the operations codegen needs to find, make an
operation hard to reuse across components, and turn a schema diff into a full-repo grep instead
of a directory listing.

## Rule 3: Never Cast Codegen Types

Never use `as` to force a codegen query/mutation result into a manually-written interface, and
never hand-write an interface that mirrors a generated type. A manual type silently drops the
nullability the schema actually declares, so code reads a field as always-present, TypeScript
agrees, and it crashes at runtime the first time the field is null. Use the generated type
directly; derive component-prop subsets with `NonNullable<...>` and indexed access, and handle
real nullability with `?.` / `?? []` — configure codegen's `avoidOptionals` so fields the schema
actually marks non-null don't force optional chaining everywhere.

```
DON'T: const session = data.session as SessionType   // hand-written SessionType
DO:    type Item = NonNullable<GetSessionQuery['session']>['items'][number]
```

## Rule 4: Query Only What You Use

Every field in a query or mutation selection must be read somewhere in the component or
business logic that consumes the result. An unused field is a latent breaking change: when the
API later removes it, a query that never needed it breaks anyway. Don't add a field "for
debugging" or "just in case" — add it when a consumer needs it, and delete it the moment the
last consumer stops reading it. A field used by only one of several components sharing a query
is a sign the query should split.

## Rule 5: Keep Codegen in Sync with the Schema

Regenerate types every time the schema changes, and run codegen before typecheck in CI so a
stale generated file fails the build instead of shipping. TypeScript checks generated types
against each other, not against the live schema — if codegen didn't rerun, the types compile
clean while the server rejects the request. A "field not defined" error at runtime with green
types locally means regenerate first, debug second.

## Rule 6: Follow a Breaking Change Protocol for Schema Changes

Before renaming, removing, retyping, or changing the required/optional status of a field that
deployed clients still call, accept and return both the old and new shapes during a transition
window, add a test that pins the old contract, and document the change. Clients you don't
control — a shipped mobile build, a third-party integration — can't upgrade the instant the
schema does; an unversioned breaking change turns a backend deploy into an outage for whichever
clients haven't updated yet.

## Rule 7: Prefer ID-Based Queries over Context-Based Queries

Avoid queries scoped to "the current user's context" — `myFamily`, `myProfile` — in favor of
queries parameterized by an explicit id — `family(id: $id)`. Most GraphQL clients cache by
query + variables; a context-based query has no variable that varies per user, so its cache
entry is shared across whoever's session touches it first, and the next user sees someone
else's cached data. `me` is the sanctioned exception — it's the one query allowed to be
context-based, because its entire job is handing back the ids everything else should key on.
Every id-based query still needs server-side access control — the id being explicit doesn't
mean the request is authorized.

The same cache/correctness argument applies to writes, with a worse failure mode. A mutation
that infers its app or tenant from implicit request context instead of taking an explicit
id/scope parameter doesn't just risk a stale cache entry — it risks executing against the
wrong tenant's data outright. Take `appId`/`tenantId` (or the relevant scope id) as an explicit
mutation argument, validated server-side against the caller's access, never assumed from
session/request state.

```
DON'T: query { myFamily { members { id name } } }
DO:    query { family(id: $familyId) { members { id name } } }

DON'T: mutation { updateSettings(input: $input) }              # scope inferred from request context
DO:    mutation { updateSettings(tenantId: $tenantId, input: $input) }
```

## Rule 8: Never Derive Display Data from IDs on the Client

The client must never parse, split, or pattern-match an id to produce a display value —
`planId.split('_')[0]` to get a plan name. If the UI needs a name, slug, or category, the API
returns it as its own field; the backend owns the id format and is free to change it without
every client's parsing logic breaking in sync.

```
DON'T: const tier = planId.split('_')[0]
DO:    query { plan(id: $id) { id displayName } }
```

## Rule 9: Don't Require Redundant IDs in Inputs

Don't ask a mutation for two ids when the second is derivable from the first on the server —
`familyId` alongside `childProfileId` when the child already belongs to a family. Redundant ids
let a client send a mismatched pair, and once the server accepts both instead of deriving one,
it's trusting a client assertion instead of validating an actual relationship. Take the most
specific id and look up the rest server-side.

## Rule 10: Separate Schema Definitions from Shared Helpers

Keep files under the schema layer limited to type definitions, resolvers, and logic that's
actually schema-specific; move authorization checks, input validation, and shared DB-access
patterns into a helpers layer that resolvers import from. A helper duplicated inside multiple
resolver files drifts — one gets a bug fix the others don't — and there's no single place to
audit "how do we check permission for X."

## Rule 11: Handle All Three Query States

Every query consumer destructures and handles `fetching`, `error`, and `data` — not just
`data`. Compute an initial-load flag as `fetching && !data` rather than `fetching` alone, so a
background refetch (data present, fetching true) shows the existing content instead of
flashing a skeleton over it. Give the error state a retry path, not just a message.

```
DON'T: if (fetching) return <Skeleton />   // also fires on background refetch
DO:    if (fetching && !data) return <Skeleton />
```

## Rule 12: Invalidate Cache Explicitly After Mutations

A document/normalized cache does not know a mutation changed data it should invalidate — that
wiring is the client's job. Use an explicit invalidation signal (a version counter incremented
on mutation success, compared against a previous value to trigger refetch) rather than
refetching on every screen focus, which refetches even when nothing changed. Skip this and a
mutation succeeds while every open screen keeps rendering the pre-mutation cached value.

## Rule 13: Share Typed Error Codes Between Backend and Frontend

Error codes live in a package both sides import, never as ad hoc string literals on either
side. Backend resolvers throw a typed code with a log-only message (masked in production);
frontend maps every code to a user-facing message through a `Record<ErrorCodeType, string>` so
the compiler flags a code with no message. A string-literal error code can drift — the backend
renames it, the frontend's string comparison silently stops matching, and the user gets the
fallback message forever instead of the real one.

## Rule 14: Abstract Infrastructure from the API Surface

Nothing in a schema field, response, or error message names a specific vendor, model, or
provider — no "Claude," "S3," "GPT-4," no provider-specific response shape. Expose
business-level concepts instead: "AI Assistant," "storage," "Service is temporarily busy."
Swapping the underlying provider must be an API-layer change only; if a client depends on a
provider name or a provider's response format, that swap becomes a client release.

## Rule 15: Never Leak Raw Errors or Provider-Specific Data to Clients

Never forward `error.message` from an infrastructure or provider call straight to the client,
and never pass a provider's response through unnormalized — model names, provider-specific
stop reasons, provider usage formats. Map every infrastructure error to a business error code
with a predefined message on the backend, log the raw detail server-side, and return only
normalized, business-relevant fields. A raw provider error string is both a leak (it names the
vendor) and a UX failure (it's not written for a user).

```
DON'T: throw new UserError(providerError.message)
DO:    throwAppError(ErrorCode.SERVICE_UNAVAILABLE, providerError.message)  // message logged, not returned
```

## Rule 16: Map Errors to User-Friendly Messages on the Frontend

The frontend never renders a raw GraphQL error to a user — no "Field 'x' is not defined by
type 'Y'." Known error codes map through the shared message table (Rule 13); `INVALID_INPUT`
is the one code allowed to show the backend's message verbatim, because that message is written
to be user-facing. Everything unmapped falls back to a generic message, and network failures
get their own specific one. An unmapped or raw error shown as-is leaks schema structure and
tells the user nothing they can act on.

## Not this

- Inline `fetch` with a GraphQL string instead of a generated typed document.
- `as SomeHandType` on a codegen result, or a hand-rolled interface duplicating one.
- A query fetching fields nothing on screen reads.
- Shipping a schema change that renames/removes a field with no transition window.
- A context-scoped query (`myFamily`) instead of an id-scoped one, causing cross-user cache
  collisions.
- Raw `error.message` or a provider name/model name reaching the client, in data or in errors.
