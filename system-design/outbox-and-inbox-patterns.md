# The Outbox and Inbox Patterns

> "Two systems, one truth, no transaction between them."
>
> *The outbox makes sure the message is never lost; the inbox makes sure it is never applied twice. You need both, because neither is sufficient alone.*

## The problem: the dual write

Almost every service eventually has to do two things at once:

```
1. Change its own database   (save the order)
2. Tell the outside world    (publish OrderPlaced)
```

These live in **two different systems** — your database and your broker — and there is no transaction spanning them. So whatever order you choose, a crash in the gap breaks something:

| Order | Crash in the gap | Result |
| --- | --- | --- |
| **Publish, then commit** | DB transaction rolls back after the message went out | **Phantom event** — the world believes in an order that doesn't exist. Inventory reserved, email sent, nothing to ship |
| **Commit, then publish** | Process dies before publishing | **Lost event** — the order exists but nothing downstream ever hears. Silent, and usually discovered by a customer |

Both failure modes are *rare* and *silent*, which is the worst combination: they don't show up in testing, and when they do happen in production the data is already inconsistent.

**Why not just use a distributed transaction (2PC)?** It technically solves this, but it's blocking (a coordinator failure leaves locks held), it's slow, and most modern brokers — Kafka, SQS, RabbitMQ in typical setups — don't support XA with your database in a way anyone wants to operate. The whole industry moved away from it, which is why these two patterns exist.

**Real-life analogy.** A shop clerk must record a sale in the ledger and hand the order slip to the courier. If they give the slip to the courier first and then discover the card was declined (**publish then roll back**), a parcel goes out for a sale that never happened. If they record the sale and then drop dead before reaching the courier (**commit then crash**), the customer is charged and nothing ships. The fix is that the clerk never talks to the courier at all: they write the sale **and** a note in an out-tray, both in the same ledger, in one stroke of the pen (**one database transaction**). A separate runner empties the out-tray whenever they pass (**the relay**). The runner may occasionally deliver the same slip twice, so the courier keeps a list of slip numbers already accepted (**the inbox**) and bins repeats.

---

## The outbox pattern (producer side)

**Write the event into your own database, in the same transaction as the business change.** Publishing becomes someone else's problem, later.

```sql
BEGIN;
  INSERT INTO orders (id, customer_id, total) VALUES (...);
  INSERT INTO outbox (id, aggregate_id, type, payload, created_at)
       VALUES (..., 'OrderPlaced', '{...}', now());
COMMIT;
```

Both rows commit or neither does — because it's **one transaction in one database**. The dual write is gone; there is now only a single write.

A separate **relay** then reads unpublished rows and pushes them to the broker:

```
[ Service ] ──one txn──► [ orders table ]
                         [ outbox table ] ──► [ relay ] ──► [ broker ] ──► consumers
```

**"Relay" here means a forwarder** — the component that *carries* messages from the outbox to the broker without originating or consuming them. It's borrowed from the relay race: your service runs its leg and drops the baton in the out-tray; the relay picks it up and runs the next leg. (Same sense as a radio relay station or a mail relay — receive, then pass on unchanged.)

It's a **role, not a technology**. Concretely it can be:

- a `@Scheduled` method inside the same service (the usual starting point),
- a separate small deployment dedicated to draining the table, or
- an off-the-shelf CDC connector like Debezium.

The property that makes it work is that it runs **outside the original transaction, asynchronously**. The service commits and returns to the caller immediately; whether the message has reached the broker yet is not its concern. That decoupling is exactly what removes the dual write — and also exactly why delivery can only be at-least-once.

### Two ways to run the relay

| | **Polling publisher** | **CDC / log tailing** |
| --- | --- | --- |
| How | `SELECT * FROM outbox WHERE published_at IS NULL` on a timer | Read the database's replication log directly — Debezium on the Postgres WAL or MySQL binlog ([write-ahead-log](write-ahead-log.md)) |
| Latency | Polling interval (typically 100 ms–1 s) | Near-real-time |
| Load | Constant queries even when idle | None on the table |
| Ordering | Whatever your query orders by | Exactly the commit order |
| Complexity | Trivial — a scheduled job | A connector to deploy, configure and operate |
| Deletes | You mark and clean up rows | The log already has them |

**Start with polling.** It's a `@Scheduled` method, it's easy to reason about, and its latency is fine for most systems. Move to CDC when the polling load or the latency actually hurts.

### Why the outbox is only half the answer

The relay publishes a row, then marks it published. If it crashes **between** those two steps, the row is still unpublished — so on restart it publishes again.

That's unavoidable: marking-then-publishing loses messages instead, which is worse. So the outbox gives you **at-least-once delivery**, and duplicates are a normal, expected part of its operation — not a bug to fix on the producer side.

---

## The inbox pattern (consumer side)

Since duplicates are guaranteed, the consumer must make **re-processing harmless**. The inbox pattern does that by recording what it has already handled — in the same transaction as the work itself:

```sql
BEGIN;
  INSERT INTO inbox (message_id) VALUES ('msg-abc');   -- PK violation if seen before
  UPDATE inventory SET reserved = reserved + 1 WHERE sku = 'X';
COMMIT;
```

If the message arrives twice, the second `INSERT` violates the primary key, the transaction rolls back, and **the side effect doesn't happen twice.** The uniqueness constraint is doing the deduplication — no extra locking, no distributed coordination.

The atomicity is the whole point: recording the ID and doing the work must commit together. Record-then-work loses messages on a crash between them; work-then-record double-applies.

See [idempotency](idempotency.md) — the inbox is one concrete mechanism for it, and where the operation is *naturally* idempotent (setting a status to a fixed value, an upsert by key) you may not need an inbox table at all.

---

## Together: effectively-once without distributed transactions

```
OUTBOX  →  guarantees the event is never LOST          (at-least-once out)
INBOX   →  guarantees the event is never APPLIED TWICE (dedup on the way in)
─────────────────────────────────────────────────────────────────
         =  effectively-once processing
```

Note what that is **not**: exactly-once *delivery*, which is impossible across a network. The message may well cross the wire several times. What you get is **exactly one effect**, which is the property anyone actually wanted.

Each pattern is insufficient alone: outbox without inbox means duplicate charges and double-decremented inventory; inbox without outbox means deduplicating messages that sometimes never arrive.

---

## Implementation notes

### The outbox table

```sql
CREATE TABLE outbox (
    id            UUID PRIMARY KEY,
    aggregate_id  VARCHAR(64) NOT NULL,   -- becomes the partition key: per-aggregate ordering
    type          VARCHAR(128) NOT NULL,
    payload       JSONB NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at  TIMESTAMPTZ              -- NULL = not yet published
);

CREATE INDEX ON outbox (created_at) WHERE published_at IS NULL;   -- partial: stays tiny
```

The partial index matters — it covers only the unpublished backlog, so the query stays fast no matter how many rows the table has accumulated.

### Writing and relaying, in Spring

```kotlin
@Service
class OrderService(
    private val orders: OrderRepository,
    private val outbox: OutboxRepository,
) {
    /** The whole point: the order and its event commit together, or not at all. */
    @Transactional
    fun place(order: Order): Order {
        val saved = orders.save(order)
        outbox.save(OutboxMessage(aggregateId = saved.id, type = "OrderPlaced", payload = saved.toEvent()))
        return saved
    }
}

@Component
class OutboxRelay(private val outbox: OutboxRepository, private val broker: MessagePublisher) {

    @Scheduled(fixedDelay = 500)
    @Transactional
    fun publishPending() {
        // SKIP LOCKED lets several instances drain the table concurrently
        // without two of them grabbing the same row.
        outbox.findUnpublishedForUpdateSkipLocked(limit = 100).forEach { message ->
            broker.publish(topic = message.type, key = message.aggregateId, payload = message.payload)
            message.markPublished()   // dirty checking flushes at commit
        }
    }
}
```

`SKIP LOCKED` is the detail that makes a multi-instance relay safe. Without it, either you run exactly one relay instance (a single point of failure) or two instances publish the same row concurrently.

### Consuming with an inbox

```kotlin
@Component
class InventoryConsumer(private val inbox: InboxRepository, private val inventory: InventoryService) {

    @Transactional
    fun onOrderPlaced(messageId: String, event: OrderPlaced) {
        // Unique constraint on messageId does the deduplication for us.
        if (!inbox.recordIfNew(messageId)) return    // already processed — silently skip

        inventory.reserve(sku = event.sku, quantity = event.quantity)
    }
}
```

### Housekeeping

- **Clean up the outbox.** Delete or archive published rows on a schedule (say, older than 7 days). An unbounded outbox table is the most common operational complaint about this pattern.
- **Prune the inbox** too, but only beyond your broker's maximum retry/redelivery window — delete an ID too early and a late redelivery gets reprocessed.
- **Keep payloads small.** The outbox is not a document store; if the event is large, publish a reference and let consumers fetch.

### Ordering

Rows commit in transaction order, but a relay publishing in parallel can emit them out of order. If order matters — and it usually only matters *per entity* — use the `aggregate_id` as the broker partition key, so all events for one order land on one partition in order. Global ordering across all entities is far more expensive and almost never actually required.

---

## How to decide

### Do I need the outbox pattern here?

| Question to ask | → Yes, use an outbox | Real-life example | → No, publish directly | Real-life example |
|---|---|---|---|---|
| Does a lost event cause real damage? | Yes | An order placed but never sent to fulfilment | No, it's advisory | A "user viewed page" analytics ping |
| Does a phantom event cause real damage? | Yes | Inventory reserved for a rolled-back order | No | A cache-warming hint |
| Is the publish inside a DB transaction that can still roll back? | Yes | Anything in a `@Transactional` service method | No — the write already committed independently | A batch job publishing a summary afterwards |
| Can the consumer tolerate silent gaps? | No | Payments, inventory, ledgers | Yes | Metrics, logs, recommendations |

### Polling relay or CDC?

| Question to ask | → Polling | Real-life example | → CDC (Debezium) | Real-life example |
|---|---|---|---|---|
| How much end-to-end latency is acceptable? | Hundreds of ms is fine | An order confirmation email | Must be near-real-time | A trading or pricing feed |
| What's the event volume? | Moderate | A few hundred events/minute | Very high, or polling load is showing on the DB | Tens of thousands/second |
| Do you want to operate a connector? | No — a `@Scheduled` job is it | A small team | Yes, and you may already run one | An existing CDC pipeline feeding a warehouse |
| Is strict commit ordering required? | Approximate is fine | Per-aggregate ordering via partition key | Yes | Rebuilding a downstream replica |

### Do I need an inbox table?

| Question to ask | → Yes, inbox table | Real-life example | → No, natural idempotency is enough | Real-life example |
|---|---|---|---|---|
| Is the operation an increment or an append? | Yes — it accumulates | `reserved = reserved + 1`, appending a ledger entry | No | `status = SHIPPED` (setting a fixed value) |
| Does it have an external side effect? | Yes | Charging a card, sending an email | No | An internal upsert |
| Can you dedupe on a natural key instead? | No | A generic event stream | Yes — a unique constraint does it | `INSERT ... ON CONFLICT DO NOTHING` keyed by order ID |

---

## Common mistakes

| Mistake | Consequence |
| --- | --- |
| Publishing inside the transaction, before commit | Phantom events for transactions that roll back |
| Publishing after commit, in the same method | Lost events when the process dies in the gap |
| Outbox without a consumer-side inbox | Duplicates applied — double charges, double decrements |
| Recording the inbox ID in a *separate* transaction from the work | Reintroduces the exact dual-write problem you were fixing |
| Never cleaning up the outbox table | Unbounded growth; the relay query slows down over time |
| Pruning the inbox faster than the broker's redelivery window | A late redelivery is reprocessed as if new |
| Multi-instance relay without `SKIP LOCKED` (or a lock) | The same row published by several instances simultaneously |
| Expecting exactly-once *delivery* | Impossible across a network. Aim for exactly-once *effect* |
| Assuming global ordering | The relay parallelises; use the aggregate ID as partition key for per-entity order |
| Large payloads in the outbox | Table bloat and slow replication; publish a reference instead |

## Key takeaways

1. **The problem is the dual write** — a DB change and a message publish with no transaction between them. Both orderings fail, silently.
2. **The outbox turns two writes into one** by putting the event in your own database, in the same transaction.
3. **A relay publishes it afterwards** — polling to start with, CDC when latency or load demands it.
4. **The outbox guarantees at-least-once, so duplicates are expected**, not a defect.
5. **The inbox makes duplicates harmless** by recording processed IDs in the same transaction as the work.
6. **Together they give exactly-once *effect*, not exactly-once delivery** — which is the property that was actually wanted.
7. **Neither works alone**: outbox without inbox double-applies; inbox without outbox deduplicates messages that may never arrive.
8. **Order per entity, not globally** — partition by aggregate ID.

## See also

- [idempotency](idempotency.md) — the inbox is one mechanism for it; where an operation is naturally idempotent you may not need the table.
- [saga-pattern-compensating-transactions](saga-pattern-compensating-transactions.md) — how multi-service workflows stay correct once each step publishes its own events this way.
- [message-queue-vs-pubsub](message-queue-vs-pubsub.md) — what the relay is publishing into, and why at-least-once is the normal delivery guarantee.
- [write-ahead-log](write-ahead-log.md) — the log a CDC relay tails, and why "read the log" gives you replication and event publishing from the same mechanism.
- [async-transaction-confirmation](async-transaction-confirmation.md) — the user-facing side of committing now and completing the rest shortly afterwards.
