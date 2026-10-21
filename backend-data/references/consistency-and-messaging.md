# Consistency and Messaging

As soon as a change involves more than one database transaction (sending an event, updating another module, calling another service), it's possible for part of it to succeed and part to fail. This file covers patterns that keep data and messages consistent.

## Contents

1. The dual-write problem
2. The transactional outbox
3. Idempotent consumers
4. Ordering
5. Eventual consistency
6. Coordinating multi-step processes
7. Events between modules in one application

## 1. The dual-write problem

Code that writes to the database and then publishes a message (or the reverse) can fail between the two steps:

```
save order to database      ✓
publish OrderPlaced event   ✗ (broker unavailable, or the process crashes)
→ the order exists, but no other system knows about it
```

```
publish OrderPlaced event   ✓
save order to database      ✗
→ other systems react to an order that doesn't exist
```

Publishing inside the database transaction doesn't fix it: the message can be sent and then the transaction can roll back. Retrying doesn't fully fix it either. The outbox pattern does.

## 2. The transactional outbox

1. In the **same transaction** as the business change, insert a row describing the event into an `outbox` table.
2. A separate **publisher process** reads unpublished outbox rows, sends them to the message broker or handlers, and marks them as published.
3. If publishing fails, the publisher retries later. The event is never lost, because it was committed with the data.

```
transaction:
    insert order
    insert into outbox (id, type, payload, created_at) values (..., 'OrderPlaced', {...}, now())

publisher (runs continuously or on a schedule):
    rows = select from outbox where published_at is null order by created_at limit 100 for update skip locked
    for row in rows:
        publish(row)
        mark row published
```

Guidelines:

- Give each event a unique ID, so consumers can detect duplicates.
- Delivery is **at least once:** if the publisher crashes after sending but before marking a row as published, the event is sent again. Consumers must handle duplicates.
- Clean up old published rows periodically.
- Some platforms read the database's change log instead of polling a table (change data capture). The principle is the same.

The same pattern applies to other side effects that should follow a committed change, such as sending emails: record the intent in the transaction, and perform it afterwards from a reliable job.

## 3. Idempotent consumers

Any message, event, webhook, or job can be delivered more than once. Consumers must make repeated processing harmless.

**Technique 1: record processed message IDs.**

```
transaction:
    if processed_messages contains message.id: return     # already handled
    apply the effect
    insert message.id into processed_messages
```

The check, the effect, and the record are in one transaction, so a crash can't leave the effect applied but unrecorded. A unique constraint on the message ID guards against two workers processing it at once.

**Technique 2: design naturally idempotent effects.**

- "Set status to paid" is idempotent; "add 100 to balance" is not.
- Upserts keyed by a natural identifier are idempotent.
- Check the current state before acting: if the order is already marked shipped, do nothing.

**Also:**

- Retry transient failures with increasing delays, a limited number of times.
- Move messages that keep failing to a dead-letter queue or failed-jobs table for inspection (see `error-handling`).
- Log the message ID with every processing attempt.

## 4. Ordering

Messages can arrive out of order: retries, parallel consumers, and multiple partitions all reorder them.

- Don't assume order unless the infrastructure guarantees it for the relevant key, and you've configured it to.
- Include a version number or timestamp in events, so consumers can ignore stale updates: if the stored version is newer than the event's, skip it.
- Prefer events that carry enough information to be processed independently.

## 5. Eventual consistency

When modules or services communicate through events, their data is updated at slightly different times. For a short period, one may show an order as placed while another doesn't know about it yet. This is **eventual consistency**: given time and no new changes, all parts agree.

- Decide which data needs immediate consistency (it must be in one transaction, in one module) and which can be eventually consistent.
- Design the user experience for the delay: show "processing" states, refresh after a moment, or confirm actions without assuming every downstream effect is complete.
- Monitor the delay (how far behind consumers are), so problems are noticed.

## 6. Coordinating multi-step processes

Some business processes span several modules or services that can't share a transaction, such as reserving stock, charging a payment, and arranging shipping.

**Sagas** coordinate such processes as a series of local transactions, each with a **compensating action** that undoes it if a later step fails:

| Step | Action | Compensation |
|---|---|---|
| 1 | Reserve stock | Release stock |
| 2 | Charge payment | Refund payment |
| 3 | Create shipment | Cancel shipment |

If step 3 fails, the saga runs the compensations for steps 2 and 1.

- **Orchestration:** one coordinator tells each participant what to do and tracks progress. Easier to follow.
- **Choreography:** each participant reacts to events from the others. Less central control, but the overall flow is harder to see.

Store the process's state durably, so it can resume after a crash. Sagas add significant complexity; first check whether the steps can live in one module with one transaction.

## 7. Events between modules in one application

In a modular monolith (see `software-design`), modules may share one database while still communicating through events.

- **In-process events within the same transaction** (handlers run before commit) keep everything consistent, but couple the modules' failures together: a failing handler rolls back the original change.
- **Events after commit through an outbox** keep modules independent and make it easier to split them into services later, at the cost of eventual consistency.

Choose deliberately per case, follow the project's existing mechanism, and record significant choices (see `decisions`).