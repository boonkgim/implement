# Scope map

One table, read at two different moments by two different subagents:

- **At plan time**, to decide which areas a slice's task list must cover — a slice that adds
  a table needs a `db` task, one that adds a screen needs a `ux` task.
- **At code, critique and fix time**, to decide which reference files a subagent loads before
  it touches a file — the same key, the same trigger, the same doc.

One row, one meaning, read twice. That is what keeps a plan's scope and a critic's checklist
from drifting apart.

| Key            | Kind      | Reference doc                                       | In scope when…                                                              |
| -------------- | --------- | ---------------------------------------------------- | ----------------------------------------------------------------------------- |
| `code`         | principle | `principles/code.md`                                  | Always. Every slice writes or touches code.                                   |
| `testing`      | principle | `principles/testing.md`                               | Always. Every slice's loop starts with a test.                                |
| `security`     | principle | `principles/security.md`                              | Auth, permissions, input validation, secrets, or user data are touched.       |
| `ux`           | principle | `principles/ux.md`                                    | Any user-facing UI is created or modified, on any platform.                   |
| `webhook`      | principle | `principles/webhook.md`                               | A handler receives a request from a system this app does not control.        |
| `db`           | library   | `libraries/db.md`                                     | A table, column, index, constraint, or migration changes.                     |
| `graphql`      | library   | `libraries/graphql.md`                                | A GraphQL schema, resolver, query, or mutation changes.                       |
| `frontend`     | library   | `libraries/frontend.md`                               | Web front-end code — components, styling, layout, accessibility.              |
| `react-native` | library   | `libraries/react-native.md`                           | A React Native / Expo screen or component changes.                            |
| `subscription` | library   | `libraries/subscription.md`                           | Billing, plans, quotas, or subscription state changes.                        |
| `landing-page` | library   | `libraries/landing-page.md`                           | A marketing or conversion-focused page is built or reworked.                  |
| `seo`          | library   | `libraries/seo.md`                                    | A page's metadata, structured data, sitemap, or crawlability changes.         |

**Detect scope from the capability, not from a guess.** A capability's screens and data, read
in §2 of `SKILL.md`, name their own files before a line of code exists — a capability that
"lets a guest join a waiting list" touches a table (`db`), a mutation (`graphql`), a screen
(`ux`, `frontend`), and stores no payment method (`security`, because it's still user data).
Read the row's trigger against what the capability actually does, not against its title.

**A key with no matching doc in this repo is still a signal.** If a slice touches something no
row covers — a queue, a search index, a native module — that gap goes in the plan's Open
questions, not into a silently unscoped slice. The absence of a library doc for a stack this
skill hasn't been taught yet is not permission to skip scoping; it's a reason to say so.

**Principles apply regardless of stack; libraries assume one.** A principle doc never names a
specific ORM, framework, or vendor — it states the invariant and lets the slice's own stack
satisfy it. A library doc is written against one real stack (illustrated in its own opening
line) because its rules are about that stack's actual seams — codegen, a query builder's
footguns, a component library's defaults. Install only the library docs your projects need;
delete the rest. The principle docs are never safe to delete — they are what makes a slice
without any library match still get reviewed.
