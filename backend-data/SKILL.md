---
name: backend-data
description: Principles for storing, querying, and changing data safely and efficiently in backend systems, with any database, ORM, or language. Covers data modelling and schema design, constraints, queries and indexes, the N+1 problem, transactions and isolation, locking and concurrent updates, database migrations without downtime, backfills, keeping data and messages consistent with the outbox pattern, idempotent consumers, caching, connection management, and data lifecycle. Use this skill whenever designing tables or collections, writing queries, repositories, or ORM code, adding indexes, writing or reviewing migrations, using transactions, publishing or consuming events, adding a cache, or investigating slow queries, even if the task only says "save this" or "add a column".
---

# Backend Data

Data outlives code. Code can be rewritten in a week; a bad data model, a lost update, or a corrupted table can affect a business for years. This skill is about keeping data **correct** (it means what it should, and stays consistent), **safe** (changes never lose or corrupt it), and **fast** (queries stay efficient as data grows).

Most guidance here applies to relational databases (such as PostgreSQL and MySQL), the most common case. The principles of constraints, transactions, safe migrations, and consistency apply to other databases too, with the specific mechanisms differing.

Read the project's `AGENTS.md` first. Use the project's database, ORM or query builder, migration tool, naming conventions, and data-access patterns. Where the project's conventions differ from this skill, follow the project. Apply the `software-design` skill for where data-access code belongs, and the `security` skill for query safety.

## Before you start

1. **Which module owns this data?** Only that module's data-access code reads or writes it (see `software-design`).
2. **What must always be true about this data?** Uniqueness, required values, valid ranges, relationships. These become constraints.
3. **How will it be queried?** Which filters, sorts, and joins, at what volume? This decides indexes.
4. **What happens with two at once?** If two requests or jobs change this data simultaneously, what could go wrong?
5. **Does this change existing data or schema?** If so, it needs a migration that is safe while the current code is still running.

## Core rules

### 1. Model the business, and let the database enforce its rules

- Design tables around business concepts, using the business's names (see `software-design`, domain modelling).
- **Use constraints** to enforce what must always be true: `NOT NULL` for required fields, `UNIQUE` for unique values, foreign keys for relationships, `CHECK` for valid ranges and states. Application checks give good error messages; constraints guarantee correctness when application code has a bug, a race condition, or is bypassed.
- **Use precise types:** proper timestamp types with time zones, exact decimal or integer types for money, booleans for flags, enumerations or check constraints for fixed sets of values. Never store numbers or dates as text.
- **Normalise by default:** store each fact once. Denormalise deliberately, for measured performance needs, and keep copies in sync explicitly.
- Add `created_at` and `updated_at` timestamps to most tables.

For more depth, read `references/modelling.md`.

### 2. Write queries that stay safe and fast

- **Always use parameterised queries** or the ORM's binding. Never build SQL from strings (see `security`).
- **Select only the columns you need,** especially for wide tables or large fields.
- **Never load unbounded results.** Paginate or limit every query that can return many rows.
- **Filter, sort, and aggregate in the database,** not by loading rows into application memory.
- **Avoid the N+1 problem:** never run a query inside a loop over query results. Load related data with a join or a single batched query (`WHERE id IN (...)`), or use the ORM's eager loading.

### 3. Index for real queries

- Add indexes that support the queries the code actually runs: columns used in `WHERE`, `JOIN`, and `ORDER BY`.
- Index every foreign key column that is used for lookups or joins.
- For queries filtering on several columns, use a composite index with columns in the right order (equality filters first, then range or sort columns).
- Don't add indexes speculatively: each one slows writes and uses storage.
- Check the query plan (`EXPLAIN`) for queries on large tables or performance-sensitive paths.

For queries, indexes, and the N+1 problem in depth, read `references/queries-and-indexes.md`.

### 4. Use transactions for changes that belong together

- Wrap operations that must succeed or fail together in a single transaction.
- **Keep transactions short.** Never make network calls (to payment providers, email services, other APIs) or wait for users inside a transaction. Long transactions hold locks and block other work.
- Decide transaction boundaries in the business logic or use-case layer, where the unit of work is known, not deep inside individual repository methods.
- A transaction covers one database. For changes that span the database and another system, use the outbox pattern (rule 7).

### 5. Prevent lost updates and race conditions

When two operations read and then write the same data, one can silently overwrite the other.

- **Prefer atomic updates in the database:** `UPDATE accounts SET balance = balance - 100 WHERE id = $1 AND balance >= 100`, rather than reading the balance, subtracting in code, and writing it back.
- **Use unique constraints** to prevent duplicates, rather than "check if exists, then insert", which fails under concurrency. Handle the constraint violation as a business error.
- **Use optimistic locking** (a version column checked on update) for records edited by people or long processes.
- **Use row locks** (`SELECT ... FOR UPDATE`) for short, high-contention critical sections, such as allocating limited stock.
- Know the isolation level your database uses by default, and what it does and doesn't protect against.

For transactions, isolation, and locking in depth, read `references/transactions-and-concurrency.md`.

### 6. Change schemas without downtime or data loss

- **Every schema change goes through a versioned migration,** committed with the code and reviewed. Never change a shared database by hand.
- **Migrations must work while the current code is still running,** because deployments are not instant. Use the **expand and contract** pattern: add new structures, migrate code and data, then remove old structures in a later release.
- **Separate destructive changes** (dropping columns or tables, renaming, narrowing types) from the code changes that stop using them.
- **Backfill large tables in batches,** outside the schema migration, not in one huge statement that locks the table.
- **Beware of locking operations** on large tables, such as adding indexes or columns with defaults in some databases. Use the database's non-blocking options where available.
- Never edit a migration that has already run in a shared environment. Write a new one.

For the full procedure, read `references/migrations.md`.

### 7. Keep data and messages consistent

When a change must both save data and notify another system (publish an event, send to a queue), doing them separately risks one succeeding while the other fails.

- **Use the transactional outbox pattern:** in the same transaction as the data change, insert the event into an outbox table. A separate process reads the outbox and publishes the events, retrying until they succeed.
- **Make every consumer idempotent:** messages will sometimes arrive more than once. Record processed message IDs, or design the effect so repeating it does no harm.
- **Expect eventual consistency** between modules and services, and design the user experience for it.

For outbox, idempotent consumers, and eventual consistency, read `references/consistency-and-messaging.md`.

### 8. Cache deliberately

- Add a cache only for a measured performance need. Caches add complexity and are a common source of stale data bugs.
- Decide the invalidation strategy before adding a cache: how long data may be stale, and what clears it when the source changes.
- Never cache data in a way that could serve one user's data to another: include the user or tenant in cache keys where data is user-specific.

For caching patterns, read `references/caching.md`.

### 9. Manage connections and resources

- Use a connection pool, configured with sensible limits. Each application instance's pool size multiplied by the number of instances must stay within the database's connection limit.
- Always release connections, including on errors (see `error-handling`).
- Set timeouts on queries, so a slow query can't hold resources indefinitely.
- Use bulk inserts and updates for large numbers of rows, rather than one statement per row.

### 10. Plan for the data's lifecycle

- **Deletion:** decide per table whether deletion is permanent (hard delete) or a flag (soft delete). Soft deletes must be filtered from every query and complicate unique constraints; use them only when there's a real need, such as recovery or audit.
- **Retention:** keep data only as long as needed, especially personal data, and remove it when required by the project's policies or regulations.
- **Audit:** for sensitive or financial data, record who changed what and when.
- **Backups:** know that backups exist and are tested by restoring them. A backup that has never been restored is a hope, not a backup.
- **Large tables:** plan for growth, through archiving old data or partitioning, before a table becomes a problem.

## Example

**Poor:**

```
function placeOrder(userId, items):
    for item in items:
        product = db.query("SELECT * FROM products WHERE id = " + item.productId)   # injection, N+1
        if product.stock < item.quantity: raise OutOfStock
    order = db.insert("orders", { userId, status: "placed" })
    for item in items:
        product = db.query("SELECT * FROM products WHERE id = " + item.productId)
        db.update("products", item.productId, { stock: product.stock - item.quantity })   # lost update
        db.insert("order_items", { orderId: order.id, ...item })
    messageBus.publish("OrderPlaced", order)                                             # may be lost
```

Problems: SQL built from strings, a query per item (twice), stock checked and then updated separately (two simultaneous orders can both pass the check and oversell), no transaction (a failure halfway leaves a partial order), and the event can be lost if publishing fails, or published even if the database changes fail.

**Better:**

```
function placeOrder(userId, items):
    transaction:
        productIds = items.map(i => i.productId)
        products = productRepository.findByIds(productIds)             # one parameterised query

        for item in items:
            reserved = productRepository.decrementStock(item.productId, item.quantity)
            # UPDATE products SET stock = stock - $2 WHERE id = $1 AND stock >= $2
            if not reserved: raise OutOfStock(item.productId)            # atomic: no overselling

        order = orderRepository.create(userId, items)                   # order and items, bulk insert
        outbox.add(OrderPlaced(order.id, userId))                       # same transaction as the order

    # A separate outbox publisher sends OrderPlaced reliably, retrying until it succeeds.
    return order
```

## Common mistakes

- No constraints: relying only on application code to keep data valid.
- Money stored as floating-point numbers; dates stored as text.
- Queries inside loops (N+1).
- `SELECT *` on wide tables, and unbounded queries without limits.
- Missing indexes on foreign keys and common filters; or many speculative indexes.
- "Check then insert" or "read, modify in code, write" under concurrency.
- Network calls inside transactions; very long transactions.
- Saving data and publishing an event as two separate steps.
- Message consumers that aren't idempotent.
- Renaming or dropping columns in the same release as the code change.
- Backfilling millions of rows in one statement inside a migration.
- Editing migrations that have already run.
- Adding a cache without an invalidation plan, or with keys that ignore the user or tenant.
- Soft deletes that some queries forget to filter.

## Reference files

| File | Read when |
|---|---|
| `references/modelling.md` | Designing tables, columns, types, keys, constraints, relationships, or deletion strategy |
| `references/queries-and-indexes.md` | Writing or optimising queries, adding indexes, fixing N+1 problems, or using an ORM |
| `references/transactions-and-concurrency.md` | Using transactions, choosing isolation levels, locking, or preventing race conditions |
| `references/migrations.md` | Writing or reviewing migrations, renaming or removing columns, or backfilling data |
| `references/consistency-and-messaging.md` | Publishing events, consuming messages, the outbox pattern, or consistency across modules |
| `references/caching.md` | Adding, using, or debugging a cache |

## Sources

Martin Kleppmann, *Designing Data-Intensive Applications*; Martin Fowler, *Patterns of Enterprise Application Architecture* (Unit of Work, Repository, Optimistic Offline Lock); Pramod Sadalage and Scott Ambler, *Refactoring Databases*; Markus Winand, *SQL Performance Explained* and use-the-index-luke.com; Chris Richardson, *Microservices Patterns* and microservices.io (Transactional Outbox, Idempotent Consumer); Bill Karwin, *SQL Antipatterns*; the PostgreSQL documentation on transaction isolation and explicit locking.