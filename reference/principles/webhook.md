# webhook

Governs any handler that receives a request from a system this app does not control — a
payment processor's event, a chat platform's callback, a delivery receipt, a partner API's
push notification. The invariants below hold regardless of which provider is on the other end.

## Rule 1: Atomic, ultra-thin ingestion

The handler does four things and nothing else: verify the signature, check for a duplicate,
store the raw payload, and hand off to async processing — all inside one transaction. No
parsing, no business logic, no calls to other systems. If the handoff (queue push, job
enqueue) fails, the whole transaction rolls back so the sender's retry redoes the work. A
handler that parses and acts on the payload inline turns a network hiccup into a half-applied
event, and turns the handler's response-timeout budget into the ceiling on how much processing
a single delivery can survive.

DO:
```
tx:
  verify_signature(req) or reject
  if exists(event_id): return 200  # duplicate, no-op
  record = insert(raw_payload, event_id, status="pending")
  enqueue(record.id)
```
DON'T:
```
verify_signature(req)
data = parse(req.body)
updateSubscription(data)   # business logic inline, no transaction, no dedupe
```

## Rule 2: Hand off an ID, not the payload

The async message or job carries only the stored record's ID, never the payload itself. The
consumer reads the payload back from the DB — the durable copy — instead of trusting the
in-flight message as its source. This keeps the message small and stable across schema
changes, and means replaying a record later (debugging, recovery) goes through the exact same
code path as the first delivery.

## Rule 3: Verify the signature before anything else, every time

Signature verification is the first line the handler executes, using the sender's own
verification method, over HTTPS only, before any parsing or DB access. Anything that fails
verification is rejected immediately. If the provider signs a timestamp into the payload,
reject anything outside a bounded tolerance window (e.g. 5 minutes) before doing anything else,
so a captured signed payload has a limited replay lifetime independent of dedup. A failed
verification is still recorded for audit —
under a synthetic ID, with the raw payload and error — but that recording must never affect
the response, and the record must never carry a retryable status. Verifying after peeking at
the payload to route it is the same bug as not verifying at all: a forged request gets treated
as trusted for however long that peek takes.

## Rule 4: Dedupe on the sender's event ID, inside the transaction, and return success for duplicates

Every delivery carries an ID from the sender. Check it against a unique constraint before
recording anything new, in the same transaction as the insert — never as a separate read
beforehand. When the ID already exists, return the success status the sender expects (not an
error) so it stops retrying, and log the duplicate for visibility. A dedupe check outside the
transaction is a race two concurrent deliveries can both pass; an error response on a duplicate
just buys another unnecessary retry.

## Rule 5: The payload is a notification, not a data source

Business logic never reads amounts, statuses, or other authoritative fields off the payload. It
re-fetches current state from the source system and acts on that. Payloads can arrive late, out
of order, or with fields missing relative to what's true right now; treating one as ground truth
applies a stale snapshot over the present. If the re-fetch fails, the event fails and retries
later — it never falls back to trusting the payload.

DO:
```
current = fetchFromSource(record.subjectId)
target = transform(current)
```
DON'T:
```
target = transform(record.payload)  # payload may be stale or incomplete
```

## Rule 6: State-based idempotency

Compare current stored state against the target state derived from the re-fetch; act only on
the difference, and do nothing when they already match. This is what makes duplicate and
out-of-order deliveries harmless without needing any delivery-order guarantee from the sender —
two deliveries of the same underlying change converge on one write, and a late-arriving stale
event becomes a no-op once state has already moved past it.

## Rule 7: Lock before compare-and-write

Read the row to compare against with a row lock held inside the same transaction as the write
(e.g. `SELECT ... FOR UPDATE`), never as a plain read taken before the transaction opens. Two
concurrent handlers — a redelivered event and a same-effect direct call, or two redeliveries of
the same event — can otherwise both read the old state and both write, producing a lost update
or a duplicated side effect.

## Rule 8: One idempotent core for every path that can produce the same effect

When an effect can be reached by more than one path — an async notification and a synchronous
confirmation the caller makes directly, a periodic reconciliation sweep, a manual retry — every
path calls the same function to resolve and apply it. The only things allowed to differ between
callers are how trust is established (signature check vs. session/ownership check) and where
the event's identity and timestamp come from (the sender's vs. a synthetic one for the
synchronous path). Idempotency must key on a stable identifier every path actually has — the
business record's own ID, enforced with a unique constraint — never on the sender's event ID,
which only the async path has. Whichever path lands first wins; the other collides on the same
key and becomes a no-op. Copy-pasting the resolve/apply logic into a second handler instead of
sharing it is how the two paths quietly drift and end up double-applying the effect.

## Rule 9: Keep the transaction to DB work only

All the state writes belonging to one processed event — the state update, its history record,
and the event's status flip to done — commit in a single transaction. Any outbound network call
(fetching current state, calling another service) happens before or after that transaction,
never inside it. Holding a transaction open across a network call turns a slow or hung remote
call into a held row lock and a stalled table.

## Not this

- Parsing and acting on the payload inline in the handler — no async hop, no transaction.
- Trusting payload fields (amount, status, currency) instead of re-fetching from the source.
- Dedupe check as a plain `SELECT` before the transaction instead of inside it.
- Returning a 4xx/5xx for a duplicate delivery instead of the success the sender expects.
- A second, copy-pasted implementation of the same fulfillment logic for a synchronous path.
- Holding a DB transaction open across an outbound API call.
