# subscription

Illustrated against Stripe (with notes for Apple/Google IAP where the pattern must generalize
across providers). Swap the provider specifics; keep the invariants.

## Rule 1: Store immutable facts, compute current state — no cron

Store only what happened (a timestamp, an event, a user action). Never store a value that is
true today but needs a background job to keep true tomorrow. Effective status, current usage,
and "is this expired" are all queries over immutable facts, evaluated at read time.

Prevents: a subscription that reads `active` for hours or days after it actually lapsed
because the cron that flips it to `expired` hasn't run yet — a user gets paid features for
free, or worse, stale state drives a billing decision.

## Rule 2: Version plans immutably

A plan definition is never updated in place. Changing a quota creates a new plan row with a new
versioned id (`plus_2025_01` → `plus_2025_06`); the old version is deactivated
(`isActive: false`), never deleted. Existing subscribers keep the version they signed up under.

Prevents: silently downgrading (or upgrading) every existing subscriber's quota the moment
product changes a number on the current plan — a support incident, not a pricing change.

## Rule 3: Preserve the original free quota forever

Capture the free quota at signup and store it on the parent record (org/user), not on the
subscription. On downgrade or expiry, grant `max(currentFreeQuota, originalFreeQuota)`, never
just "whatever the free plan is today." The parent record outlives any single subscription.

Prevents: a day-one user who signed up when free meant 1,000 credits getting silently cut to
today's free tier of 50 credits the moment their paid plan lapses.

## Rule 4: Record usage as events, not counters

Usage is INSERT-only, append-only rows (`usage_event(org_id, quantity, created_at)`), never a
mutable counter that gets incremented and periodically reset. Current-period usage is
`SUM(quantity) WHERE created_at BETWEEN periodStart AND periodEnd`. Index
`(organization_id, created_at)`.

Prevents: a reset job that fires early, late, or twice — either wiping usage a customer paid
to keep, or letting them run over quota because the counter never rolled over.

## Rule 5: Never store computed status

The database stores only user-driven states — `active` or `canceled`. `expired` is derived,
never written. Free plans are always `active`. A canceled user keeps paid benefits until
`periodEnd`, which the query, not a stored flag, enforces.

Prevents: two competing writers (a webhook and a cron job) racing to set `status='expired'`,
one of them clobbering a concurrent `reactivated` write.

## Rule 6: Audit every subscription change

Every change — `created`, `upgraded`, `downgraded`, `canceled`, `reactivated`, `renewed`,
`expired`, `scheduled_downgrade_applied` — writes a `subscription_history` row with `action`,
`fromPlanId`, `toPlanId`, `reason`, `source` (`stripe_webhook`, `api`, `system`,
`on_demand_check`), `createdBy`, `metadata`.

Prevents: a billing dispute nobody can resolve because there's no record of whether a
downgrade came from the user, a failed payment, or an internal script.

## Rule 7: Frontend fetches plan data from the API — never hardcodes it

Prices, quotas, and plan names live in the API response, not in frontend constants. Use an
`isActive` flag on prices to control visibility (hide web checkout during app-store review)
and platform flags (`isAvailableForIos`, `isAvailableForAndroid`) for per-platform control.

Prevents: a price change requiring an app store release to take effect, or a stale hardcoded
quota telling a user they're capped at a number that changed weeks ago.

## Rule 8: Separate pricing from plan features

The plan table holds features and quotas only — no price columns. Prices live in
`subscription_price`, one row per `plan + channel + interval`; channel is a provider name
(`stripe`, `apple`, `google`), never `web`. Yearly price follows the 11-month convention
(`monthlyPrice * 11`). Payment records store `pricePaidCents`, the actual amount, never list
price.

Prevents: Apple's 30% cut forcing a different price than Stripe's for the "same" plan, with
nowhere to store that difference because price was baked into the plan row.

## Rule 9: One table per payment provider — never a shared payment table

Each provider gets its own table (`stripe_payment`, `apple_payment`, `google_payment`) with its
own fields (`customerId`, `purchaseToken`, `originalTransactionId`). The core `subscription`
table is provider-agnostic with a `paymentProvider` discriminator. Adding a provider means
adding a table, never widening an existing one with more nullable columns.

Prevents: a query that filters on the wrong provider's fields silently matching rows from a
different provider because they share one table with a pile of nullable, provider-specific
columns.

## Rule 10: Version external product/price IDs

External IDs carry the version for grandfathering: Apple `plus_2025_12_monthly` (underscores),
Google `plus-2025-12-monthly` (hyphens only), Stripe an auto-generated price id with the plan
version in `metadata.planId`. A price change creates new products/prices; old ones stay active
for whoever already bought them. Store `providerPriceId` on `subscription_price` for reverse
lookup.

Prevents: raising the price for new subscribers silently changing what existing subscribers
are charged on renewal, because everyone points at the same mutable product ID.

## Rule 11: Look up provider price IDs in the database — never parse the string

Webhooks map an external price ID to an internal plan via
`lookupPriceInfo(channel, providerPriceId)` — a database lookup by `channel + providerPriceId`
— never by parsing the ID string for embedded plan info. The lookup must not filter on
`isActive`: deprecated plans still need their webhooks processed for subscribers still on them.

Prevents: a webhook handler that regex-parses `plus_2025_12_monthly` breaking silently the
first time a plan name contains an underscore the parser didn't anticipate, or dropping events
for a deprecated plan because the lookup excluded inactive rows.

## Rule 12: Defer downgrades with a pendingPlanId

A downgrade never changes `planId` immediately — the user already paid for the higher tier.
Store the target in `pendingPlanId`; apply it when `periodEnd` is reached (on renewal). Clear
`pendingPlanId` on cancel, upgrade, reactivation, expiry, or if the user reverses the
downgrade. The UI shows the pending change and its effective date.

Prevents: a user who downgrades on day 2 of a 30-day paid period immediately losing the
features they already paid for that period.

## Rule 13: Keep billing periods continuous on revert

When an expired subscription reverts to free, set `billingAnchor` to the *previous*
`periodEnd`, never to `new Date()`. New signups anchor on current time (a fresh cycle);
upgrades and downgrades keep the existing anchor.

Prevents: a quota-reset date that silently drifts forward every time a subscription lapses and
reverts, or a gap/overlap in billing periods that lets a user double up on quota.

## Rule 14: Self-heal on demand — never depend on cron

An on-demand check reverts an expired subscription the moment it's queried (e.g., in the API
resolver), not on a schedule. Both webhooks and the on-demand check handle expiry — defense in
depth. On-demand reverts log with `source: 'on_demand_check'` like any other change (Rule 6).

Prevents: a missed or delayed webhook leaving a subscription stuck in a stale paid state
indefinitely, with no cron and no read-path check to ever catch it.

## Rule 15: Return effective values from the API — never raw DB rows

The API computes and returns effective plan, effective status, computed quotas, and reset
dates — never a raw `planId`/`status`/`periodEnd` for an expired plan. The frontend has zero
subscription business logic: it displays what the API returns and never derives flags like
`isPremium` from a plan name.

Prevents: two frontend surfaces (web and mobile) each re-implementing "is this user still
premium" slightly differently, disagreeing at the exact moment a subscription expires.

## Rule 16: Webhook is the single source of truth for side effects

When a mutation and a webhook could both produce the same side effect (a history row, a status
change), the webhook owns all database side effects — the mutation only triggers the provider
action (call the Stripe API) and returns. Exception: one-time fulfillment reachable by both a
webhook and a synchronous client confirm (checkout success-redirect, credit grant) — converge
both on one idempotent core keyed on a shared business id (the checkout session id,
UNIQUE-constrained). See `principles/webhook.md` Rule 18.

Prevents: a `subscription_history` row written twice — once by the mutation, once by the
webhook that fires moments later for the same change — or a race where both write conflicting
state.

## Rule 17: Use the provider's timestamps — never `new Date()` for billing

`billingAnchor` and `periodEnd` come from the provider's own event data: Stripe
`subscription.current_period_start/end`, Apple `transaction.purchaseDate/expiresDate`, Google
`purchase.startTimeMillis/expiryTimeMillis`. Update `billingAnchor` on every renewal to the new
period start.

Prevents: a webhook retried hours or days after the actual event (Stripe retries failed
webhooks for up to three days) stamping the period with the retry time instead of when billing
actually happened, shifting every subsequent renewal date.

## Rule 18: Payment tables are ledgers, not upsert targets

One row per transaction, inserted with `ON CONFLICT DO NOTHING` — never an UPSERT that
overwrites a previous payment row. Each payment is an immutable financial record, keyed on a
transaction-level id (Stripe `invoiceId`, Apple `transactionId`, Google `orderId`).
Subscription-level ids (`sub_xxx`, `purchaseToken`) are stored for reference, never as the
unique key.

Prevents: a retried or duplicate webhook UPSERTing over an existing payment row and silently
losing the record of an earlier charge — a real problem the moment finance needs to reconcile
against Stripe.

## Rule 19: Detect cancellation from both fields

Check both `cancel_at_period_end` (boolean) and `cancel_at` (timestamp). A direct cancel sets
the boolean; a cancel bundled with a scheduled change (e.g., downgrade-then-cancel) sets
`cancel_at` instead. Reactivation is `status === 'canceled'` with neither field set. On
downgrade with `proration_behavior: 'always_invoice'`, don't touch `planId` until the renewal
invoice fires with `billing_reason: 'subscription_cycle'` (Rule 12).

Prevents: a handler that only checks `cancel_at_period_end` missing every cancellation that
was scheduled alongside a plan change — the subscription silently renews when the user thought
they'd canceled.

## Rule 20: Retain compliance records through account deletion

Payment and subscription records survive account deletion (7+ years, tax/legal). Use a
two-transaction hybrid delete: TX1 attempts a hard delete; on FK `restrict_violation` (Postgres
23001), TX2 soft-deletes and masks PII (`"Deleted #abc123"`, `"abc123@deleted.local"`). Payment
tables use `onDelete: 'restrict'`. Never catch-and-continue inside one Postgres transaction —
it's already aborted after the error.

Prevents: a naive `ON DELETE CASCADE` wiping a user's payment history the moment they delete
their account, destroying records the business is legally required to keep.

## Rule 21: Route by Stripe metadata, not a mapping table

For multi-app platforms, use `price.metadata.app` for routing — no separate app-mapping table
to keep in sync. Every webhook event traces back to a price object (it carries both `id` and
`metadata`). Use the event's `previous_attributes` for change detection; in
`subscription.updated`, always sync to Stripe's current values rather than diffing against
your own prior state.

Prevents: a hand-rolled mapping table drifting out of sync with what's actually configured in
Stripe, routing an event to the wrong app.

## Rule 22: Link purchases to internal accounts at purchase time

Every purchase carries an internal account identifier so the webhook can attribute the event:
Stripe `metadata.familyId` in the checkout session, Apple `appAccountToken` (UUID) at purchase,
Google `obfuscatedAccountIdAndroid` with a JSON payload. Without this, a webhook has a payment
event and no reliable way to say whose account it belongs to.

Prevents: a purchase completing on the provider's side with no way to attribute it to a user —
support has to manually match a Stripe invoice to an account by email and hope it's unique.

## Not this

- A cron job resetting usage counters or flipping `status` to `expired`.
- Editing a plan's quota in place instead of publishing a new versioned plan id.
- `new Date()` anywhere near `billingAnchor` or `periodEnd`.
- UPSERT on a payment table keyed by subscription id instead of transaction id.
- Frontend computing `isPremium` from a plan name instead of reading it from the API.
- Both a mutation and its webhook writing the same `subscription_history` row.
