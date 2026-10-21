# Queries and Indexes

Queries that are fast with test data can be painfully slow in production. This file covers writing efficient queries, using indexes, avoiding the N+1 problem, and working with ORMs.

## Contents

1. Query habits
2. The N+1 problem
3. How indexes help
4. Choosing indexes
5. Reading query plans
6. Pagination queries
7. Working with ORMs
8. Bulk operations
9. Reporting and heavy queries

## 1. Query habits

- **Parameterise every value.** Never concatenate input into SQL.
- **Select only what you need.** `SELECT id, status, total FROM orders` rather than `SELECT *`, especially when tables have large text or JSON columns.
- **Let the database do the work.** Filtering, sorting, counting, grouping, and joining are what databases are optimised for. Loading thousands of rows to filter in application code wastes memory, network, and time.
- **Limit results.** Every query that could return many rows has a limit.
- **Check existence with `EXISTS`** or `LIMIT 1`, not by counting all matching rows.
- **Keep functions off indexed columns in filters.** `WHERE LOWER(email) = $1` can't use a plain index on `email`. Store normalised values, or create an index on the expression.
- **Be careful with leading wildcards.** `LIKE '%term'` can't use a normal index. Use full-text search for real search needs.

## 2. The N+1 problem

The N+1 problem is running one query to get a list, then one more query for each item.

```
Poor (1 + N queries):
    orders = query("SELECT ... FROM orders WHERE customer_id = $1", customerId)    # 1 query
    for order in orders:
        order.items = query("SELECT ... FROM order_items WHERE order_id = $1", order.id)   # N queries

Better (2 queries):
    orders = query("SELECT ... FROM orders WHERE customer_id = $1", customerId)
    items = query("SELECT ... FROM order_items WHERE order_id = ANY($1)", orders.map(o => o.id))
    group items by order_id and attach

Or (1 query):
    a JOIN, if the result size stays reasonable
```

With 50 orders, the poor version makes 51 database round trips; the better version makes 2.

Where it hides:

- **ORM lazy loading:** accessing `order.items` inside a loop triggers a query each time. Use the ORM's eager-loading feature.
- **Serialisers and response mappers** that load related data for each item.
- **GraphQL resolvers** for nested fields. Use a batching tool such as a data loader.
- **Loops calling service methods** that each query the database.

Spot it by logging queries in development and watching for repeated similar queries, or using the ORM's query count tools in tests.

## 3. How indexes help

Without an index, finding rows means scanning the whole table. An index is a sorted structure that lets the database jump straight to matching rows, like the index of a book.

Indexes speed up:

- `WHERE` filters on indexed columns.
- `JOIN` conditions.
- `ORDER BY` on indexed columns, avoiding a separate sort.
- Uniqueness checks.

Indexes cost:

- Slower inserts, updates, and deletes, since each index must be updated.
- Storage space.
- Maintenance overhead.

## 4. Choosing indexes

- **Index foreign keys** used in joins or lookups (`order_items.order_id`). Some databases don't do this automatically.
- **Index columns used in frequent filters** on large tables.
- **Composite indexes** for multi-column filters. Column order matters: put equality filters first, then range or sort columns. An index on `(customer_id, created_at)` serves:
  - `WHERE customer_id = $1`
  - `WHERE customer_id = $1 ORDER BY created_at DESC`
  - `WHERE customer_id = $1 AND created_at > $2`
  
  but not `WHERE created_at > $1` alone.
- **Covering indexes** include all columns a query needs, so the table itself isn't read.
- **Partial indexes** index only some rows, such as `WHERE status = 'pending'`, keeping the index small when queries only ever look at that subset.
- **Expression indexes** support filters on computed values, such as `LOWER(email)`.
- **Low-selectivity columns** (such as a boolean where most rows have the same value) rarely benefit from an index alone; combine them in a composite or partial index.
- **Remove unused indexes.** Databases can report index usage.

Base indexes on the queries the code actually runs, and check with query plans.

## 5. Reading query plans

`EXPLAIN` shows how the database will execute a query; `EXPLAIN ANALYZE` (PostgreSQL) actually runs it and shows real timings. Use them for queries on large tables or performance-sensitive paths.

Look for:

- **Sequential or full table scans** on large tables, where an index scan was expected.
- **Large differences between estimated and actual row counts,** which suggest outdated statistics.
- **Sorts of many rows** that an index could avoid.
- **Nested loops over large row counts.**

Run plans against realistic data volumes. A query plan on a ten-row development table tells you little about production.

## 6. Pagination queries

- **Offset pagination** (`LIMIT 20 OFFSET 100000`) gets slower as the offset grows, because the database must read and discard all earlier rows.
- **Keyset (cursor) pagination** stays fast at any depth:

```
SELECT id, created_at, ... FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 21
```

with an index on `(customer_id, created_at, id)`. See `backend-api-design` for the API side.

## 7. Working with ORMs

ORMs save time and prevent injection by default, but hide what queries run.

- **Know what SQL your ORM code produces.** Enable query logging in development for new data-access code.
- **Use eager loading** to avoid N+1 problems.
- **Select specific fields** for large or performance-sensitive queries.
- **Use raw SQL when it's clearer or faster,** for complex reports or bulk operations, always with parameters.
- **Keep ORM usage inside the data-access layer.** Business logic should receive plain domain objects, not ORM entities that can trigger lazy queries anywhere (see `software-design`).
- **Watch for ORM features that load whole tables,** such as calling `.all()` and filtering in code.

## 8. Bulk operations

- Insert many rows in one statement or with the database's bulk loading feature, not one statement per row.
- Update many rows with a single set-based `UPDATE`, not by loading and saving each.
- For very large operations (millions of rows), work in batches of a few thousand, committing between batches, so locks are short and progress isn't lost on failure.

## 9. Reporting and heavy queries

- Heavy reporting and analytics queries can slow down the main application. Run them against a read replica, a separate reporting database, or precomputed summary tables where the project supports it.
- Schedule expensive aggregations as background jobs and store the results, rather than computing them on every request.
- Set statement timeouts, so a runaway query is stopped rather than degrading everything.