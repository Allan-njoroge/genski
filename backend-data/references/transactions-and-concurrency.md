# Transactions and Concurrency

Most data bugs that appear "randomly" in production are concurrency bugs: two operations interleaving in a way nobody tested. This file covers transactions, isolation, and techniques for keeping concurrent changes correct.

## Contents

1. What transactions guarantee
2. Where transaction boundaries belong
3. Keeping transactions short
4. Isolation levels
5. Common concurrency problems
6. Techniques for correct concurrent updates
7. Deadlocks
8. Testing concurrency

## 1. What transactions guarantee

A transaction groups operations so that:

- **Atomicity:** all of them happen, or none do.
- **Consistency:** constraints hold before and after.
- **Isolation:** concurrent transactions don't see each other's partial work (to a degree set by the isolation level).
- **Durability:** once committed, the change survives crashes.

Use a transaction whenever several changes must succeed or fail together: creating an order with its lines, moving money between accounts, changing a status and recording the change in a history table.

## 2. Where transaction boundaries belong

- The **use case** (a service method such as `placeOrder`) knows what must succeed or fail together, so the transaction usually starts and ends there.
- Repository methods take part in the current transaction rather than each starting their own. Otherwise, two repository calls in one use case run in separate transactions and lose atomicity.
- Use the project's mechanism for this: a unit-of-work object, a transaction passed to repositories, or framework-managed transactions.
- One transaction per use case is a good default. Nested transactions and savepoints are rarely needed.

## 3. Keeping transactions short

While a transaction is open, it may hold locks that block other work, and it ties up a database connection.

- **No network calls inside transactions:** no payment provider, email service, HTTP API, or message broker. If the call is slow, everything waiting on those locks is slow. If it succeeds but the transaction rolls back, the two systems disagree.
- **No waiting for users** inside a transaction.
- **Prepare before, act after:** do validation, calculations, and external reads before opening the transaction; perform external side effects after it commits, or via the outbox (see `consistency-and-messaging.md`).
- **Keep large batch work out of single transactions:** process in batches with a commit per batch.

## 4. Isolation levels

Isolation levels control what concurrent transactions can see of each other. Stronger levels prevent more anomalies but can reduce throughput and cause more transaction retries.

| Level | Prevents | Still allows |
|---|---|---|
| Read Uncommitted | Almost nothing | Dirty reads (seeing uncommitted changes) |
| Read Committed | Dirty reads | Non-repeatable reads, lost updates (in read-modify-write code), write skew |
| Repeatable Read / Snapshot | Dirty and non-repeatable reads; most lost updates in some databases | Write skew |
| Serializable | All of the above: transactions behave as if run one at a time | Nothing, but some transactions fail and must be retried |

Defaults differ: PostgreSQL defaults to Read Committed; MySQL with InnoDB defaults to Repeatable Read. Exact behaviour also differs between databases, so check the documentation for the one the project uses.

Most applications use the default level and protect specific critical operations with the techniques in section 6. Use Serializable for critical operations where correctness is subtle, and retry the transaction when the database reports a serialisation failure.

## 5. Common concurrency problems

**Lost update:** two transactions read the same value, both modify it in code, and both write. The first write is lost.

```
T1 reads stock = 5      T2 reads stock = 5
T1 writes stock = 4     T2 writes stock = 4      # two sold, stock shows one sold
```

**Check-then-act race:** two transactions check a condition, both see it as true, and both act.

```
T1: is email taken? no      T2: is email taken? no
T1: insert user             T2: insert user        # duplicate account
```

**Write skew:** two transactions read overlapping data, make decisions based on it, and write different rows, together breaking a rule neither broke alone. Example: a rule that at least one doctor must be on call; two doctors each check that the other is on call, and both go off call.

**Double processing:** two workers pick up the same job or message and both process it.

## 6. Techniques for correct concurrent updates

**Atomic updates.** Let the database do the read-modify-write in one statement:

```
UPDATE products SET stock = stock - $2 WHERE id = $1 AND stock >= $2
-- check the affected row count: 0 means insufficient stock
```

This is the simplest and most efficient fix for counters, balances, and stock.

**Unique constraints.** Replace "check, then insert" with "insert, and handle the constraint violation":

```
try: insert user with email
catch unique violation on email: raise EmailAlreadyRegistered
```

Keep a friendly pre-check if useful for user experience, but rely on the constraint for correctness.

**Upserts.** Use the database's insert-or-update (`INSERT ... ON CONFLICT` in PostgreSQL, `INSERT ... ON DUPLICATE KEY UPDATE` in MySQL) for "create if missing, otherwise update".

**Optimistic locking.** For records edited by people or long processes, add a `version` column:

```
UPDATE documents SET body = $1, version = version + 1
WHERE id = $2 AND version = $3
-- 0 rows: someone else changed it; report a conflict
```

Best when conflicts are rare. Nothing is locked while the user is editing.

**Pessimistic locking.** Lock rows for the duration of a short transaction:

```
SELECT ... FROM seats WHERE id = $1 FOR UPDATE
-- other transactions trying to lock this row wait until this one commits
```

Best for short, high-contention sections, such as allocating limited inventory. Keep these transactions very short.

**Claiming work safely.** For job queues in a database, lock and skip rows other workers already hold:

```
SELECT id FROM jobs WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED
```

**Serializable isolation with retries,** for complex rules spanning many rows, such as write skew cases.

| Situation | Usual technique |
|---|---|
| Counters, balances, stock | Atomic update with a condition |
| Preventing duplicates | Unique constraint |
| Create or update | Upsert |
| User-edited records | Optimistic locking |
| Short, contended allocation | `SELECT ... FOR UPDATE` |
| Job and queue processing | `FOR UPDATE SKIP LOCKED` or the queue's own mechanism |
| Complex multi-row rules | Serializable isolation with retry, or an explicit lock on a parent row |

## 7. Deadlocks

A deadlock happens when two transactions each hold a lock the other needs. The database detects it and aborts one of them.

- **Lock rows in a consistent order** across the codebase, such as by ascending ID.
- **Keep transactions short,** reducing the window for conflicts.
- **Retry** the aborted transaction a limited number of times, if the operation is safe to repeat.

## 8. Testing concurrency

- Concurrency bugs rarely appear in normal tests. For critical operations (payments, stock, bookings), write tests that run the operation concurrently against a real database and check the invariant afterwards: stock never negative, no duplicate bookings, balances add up.
- Make tests deterministic where possible by controlling the order of steps (see `testing`).