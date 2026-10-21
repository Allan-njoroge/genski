# Data Modelling

A good data model makes correct data easy to store and incorrect data impossible. This file covers designing tables, choosing types and keys, and using constraints.

## Contents

1. Start from the business
2. Naming
3. Types
4. Primary keys
5. Constraints
6. Relationships
7. Normalisation and denormalisation
8. Common columns
9. Deletion strategies
10. Flexible data (JSON columns)
11. Document databases

## 1. Start from the business

- Identify the business concepts (customer, order, invoice, payment) and the rules about them before designing tables.
- One table per concept, named after it.
- Model states and transitions explicitly: an order's `status` with a fixed set of values, not several loose boolean flags (see `software-design`, "Make illegal states unrepresentable").
- Model history where the business needs it, for example a `price_changes` table rather than overwriting a price that past orders depended on.
- Store values that were true at the time of an event alongside the event. An order line should store the price paid, not just a reference to the product's current price.

## 2. Naming

Follow the project's conventions. Common defaults:

- Lowercase `snake_case` for tables and columns.
- Consistent plural (`orders`) or singular (`order`) table names, project-wide.
- Foreign keys named after the referenced table: `customer_id`.
- Booleans as questions: `is_active`, `has_paid`.
- Timestamps ending in `_at`: `created_at`, `shipped_at`. Dates ending in `_on` or `_date`.
- Avoid reserved words (`user`, `order`, `group` in some databases) or quote them consistently.

## 3. Types

| Data | Use | Avoid |
|---|---|---|
| Money | Integer in the smallest unit, or exact decimal (`NUMERIC`/`DECIMAL`), with a currency column | `FLOAT`, `DOUBLE`, text |
| Timestamps | Timestamp with time zone (`timestamptz` in PostgreSQL), stored in UTC | Text, local time without zone, integers unless the project uses them |
| Dates | `DATE` | Text |
| Fixed set of values | Enumeration type, or text with a `CHECK` constraint, or a lookup table | Free text without constraints; magic numbers |
| Flags | `BOOLEAN` | `0`/`1` integers or `'Y'`/`'N'` text, unless the database lacks booleans |
| Identifiers | The project's ID type (integer, UUID, or other) | Mixing types for the same concept |
| Short text with a known limit | Text with a length constraint | Unbounded text where a limit is a business rule |
| Structured but variable data | JSON column (see section 10) | Serialised blobs that can't be queried or validated |

Use `NOT NULL` unless a value is genuinely optional, and decide what absence means.

## 4. Primary keys

- Every table has a primary key.
- **Surrogate keys** (generated IDs) are the usual choice. Natural keys (email, national ID) change, and changing a primary key is painful.
- **Auto-incrementing integers** are compact and fast, but reveal volume and ordering if exposed, and are hard to generate outside the database.
- **UUIDs** can be generated anywhere and don't reveal information. Time-ordered UUIDs (such as UUIDv7) index much better than random ones (UUIDv4) on large tables.
- Whatever the project uses, be consistent. Exposing IDs in APIs is fine, but authorisation must never rely on IDs being unguessable.

## 5. Constraints

Constraints are the database's guarantee that data obeys rules, even when application code has bugs or race conditions.

| Constraint | Enforces | Example |
|---|---|---|
| `NOT NULL` | A value must be present | `email NOT NULL` |
| `UNIQUE` | No duplicates | `UNIQUE (email)`, `UNIQUE (order_id, product_id)` |
| `FOREIGN KEY` | Referenced row exists | `customer_id REFERENCES customers(id)` |
| `CHECK` | Values meet a condition | `CHECK (quantity > 0)`, `CHECK (status IN ('pending','paid','shipped'))` |
| Exclusion (some databases) | No overlapping ranges | No two bookings for the same room at overlapping times |

Guidelines:

- Validate in the application too, for good error messages. Treat constraints as the final safety net.
- Translate constraint violations into meaningful business errors in the data-access layer (a unique violation on `email` becomes `EmailAlreadyRegistered`; see `error-handling`).
- Use **partial unique indexes** for conditional uniqueness, such as "one active subscription per customer": `UNIQUE (customer_id) WHERE status = 'active'`.
- Name constraints explicitly where the project does, so violations can be recognised reliably.

## 6. Relationships

- **One-to-many:** a foreign key on the "many" side (`orders.customer_id`).
- **Many-to-many:** a join table with two foreign keys and a unique constraint on the pair (`order_tags (order_id, tag_id)`). Give the join table its own columns if the relationship has data, such as `quantity`.
- **One-to-one:** a foreign key with a unique constraint, or merge into one table if they're always used together.
- **Choose deletion behaviour deliberately:** `ON DELETE RESTRICT` (prevent deleting a parent with children) is the safe default. Use `CASCADE` only when children have no meaning without the parent, such as an order's lines.
- **Foreign keys across module boundaries** couple modules at the database level. Within a module, use them freely. Across modules, follow the project's approach: some keep cross-module foreign keys for integrity; others store only the ID and rely on the owning module's interface.

## 7. Normalisation and denormalisation

**Normalise by default:** store each fact in one place. A customer's address lives in one row, referenced elsewhere by ID. This prevents data getting out of sync.

**Denormalise deliberately** when a measured performance need justifies it:

- Store a computed total on an order, rather than summing lines on every read.
- Copy a customer's name onto an invoice, because it must reflect the name at the time.
- Maintain a summary or read-optimised table for reporting.

When denormalising, decide how the copy stays correct: updated in the same transaction, recomputed by a job, or intentionally frozen at a point in time.

## 8. Common columns

- `id`: primary key.
- `created_at`: when the row was created, set by the database or application, never by the client.
- `updated_at`: when it was last changed. Make sure every update path sets it.
- `version`: for optimistic locking, where records are edited concurrently.
- Tenant or owner ID: on every row of tenant-scoped data, in multi-tenant systems (see `security`).
- `created_by` / `updated_by`: where accountability matters.

## 9. Deletion strategies

| Strategy | How | Use when | Costs |
|---|---|---|---|
| Hard delete | Row is removed | Data has no further value, or must be erased (such as privacy requests) | Gone permanently; foreign keys must allow it |
| Soft delete | `deleted_at` timestamp set | Recovery, audit, or references must be kept | Every query must filter it; unique constraints need partial indexes; data accumulates |
| Archive | Moved to an archive table or storage | Old data rarely needed but must be kept | Two places to query if needed |
| Status | `status = 'cancelled'`, `'closed'` | The "deletion" is really a business state | None; often the most honest model |

Soft deletes are often overused. Ask whether the business actually has a state such as "cancelled" or "archived", and model that instead. If soft deletes are used, filter them centrally (in the repository or ORM default scope), not in each query.

## 10. Flexible data (JSON columns)

JSON columns are useful for data whose structure genuinely varies or that the system stores but doesn't query much, such as provider responses, user preferences, or event payloads.

- Don't use JSON to avoid designing a schema. Data you filter, join, or enforce rules on belongs in proper columns with constraints.
- Validate JSON content in the application against a schema.
- Index specific JSON fields if you must query them, using the database's JSON indexing support.

## 11. Document databases

In document databases (such as MongoDB), the same principles apply with different mechanisms:

- Design documents around how data is read: embed data that is always read together and owned by the parent; reference data that is shared, large, or grows without bound.
- Enforce schemas with the database's validation features or in the application, since there are no tables to enforce them.
- Use unique indexes for uniqueness.
- Use the database's atomic update operators to avoid lost updates, and multi-document transactions where available and needed.