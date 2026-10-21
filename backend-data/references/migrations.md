# Migrations

A migration changes the database schema or existing data. Because databases are shared, long-lived, and full of real data, migrations carry more risk than almost any other change. This file covers writing them so they never cause downtime or data loss.

## Contents

1. Rules for every migration
2. Why migrations must work with old code
3. Expand and contract
4. Common changes, done safely
5. Locking and large tables
6. Backfilling data
7. Reversibility
8. Checklist

## 1. Rules for every migration

- **Use the project's migration tool.** Every schema change is a versioned migration file committed with the code.
- **Never change a shared database by hand.** Development, staging, and production must be reproducible from migrations.
- **Never edit a migration that has run** in any shared environment. Fix mistakes with a new migration.
- **One logical change per migration,** with a descriptive name.
- **Keep schema changes and large data changes separate.** Schema migrations should be fast; large data backfills run separately, in batches.
- **Test migrations** against a copy of realistic data where possible, and run them in continuous integration.

## 2. Why migrations must work with old code

During a deployment, there's a period when the migration has run but old application instances are still serving traffic, or the new code is running but some instances still have the old version. If the database schema breaks the old code, users see errors.

So every migration must be compatible with both:

- the code currently running, and
- the code being deployed.

This rules out renaming or dropping something the current code still uses, in the same release.

## 3. Expand and contract

The expand and contract pattern (also called parallel change) makes breaking schema changes safe by splitting them across several releases:

1. **Expand:** add the new structure alongside the old. Nothing is removed. Old code keeps working.
2. **Migrate:** deploy code that writes to both old and new (or to the new, with the old kept in sync), backfill existing data, then switch reads to the new structure.
3. **Contract:** once no running code uses the old structure, remove it in a later release.

Each step is a separate deployment. It takes longer, but each step is safe and reversible.

## 4. Common changes, done safely

**Add a nullable column, or a column with a simple default:** usually safe in one step. Check whether the database rewrites the table for defaults; modern PostgreSQL versions don't for constant defaults, but some databases and versions do.

**Add a required (`NOT NULL`) column:**

1. Add it as nullable (or with a default).
2. Deploy code that always writes it.
3. Backfill existing rows in batches.
4. Add the `NOT NULL` constraint. On large tables, use the database's non-blocking method where available (in PostgreSQL, add a `CHECK (column IS NOT NULL) NOT VALID` constraint, then validate it separately).

**Rename a column:**

1. Add the new column.
2. Deploy code that writes to both columns and reads from the old one.
3. Backfill the new column.
4. Deploy code that reads from the new column (still writing both).
5. Deploy code that stops writing the old column.
6. Drop the old column.

Never use a direct `RENAME` while old code is running, unless the project accepts downtime.

**Remove a column:**

1. Deploy code that no longer reads or writes it (and, for ORMs, no longer maps it).
2. In a later release, drop the column.

**Change a column's type:** treat it like a rename: add a new column of the new type, dual-write, backfill, switch reads, then drop the old.

**Add an index on a large table:** use the non-blocking option (`CREATE INDEX CONCURRENTLY` in PostgreSQL; online DDL in MySQL). These usually can't run inside a transaction, so check how the migration tool handles them.

**Add a foreign key or check constraint on a large table:** in PostgreSQL, add it as `NOT VALID` first (only new rows are checked), then `VALIDATE CONSTRAINT` separately, which doesn't block writes.

**Drop a table:** make sure no code uses it, then drop it in a later release. Consider renaming or archiving it first, so recovery is possible.

## 5. Locking and large tables

Some schema changes lock a table while they run, blocking reads or writes. On a small table, the lock is brief. On a table with millions of rows, it can mean minutes of downtime.

- Know which operations lock or rewrite tables in the project's database and version. Check the documentation.
- Set a **lock timeout** for migrations, so a migration waiting for a lock fails quickly instead of queueing behind long-running queries and blocking everything behind it.
- Run risky migrations during low-traffic periods, if the project's process requires it.
- For very large or complex changes, consider the database's online schema change tools.

## 6. Backfilling data

Backfilling fills new columns or transforms existing data.

- **Run backfills separately from schema migrations,** as a script or background job.
- **Work in batches** (for example, 1,000 to 10,000 rows), committing between batches, so each transaction is short.
- **Make it resumable and idempotent:** it should be safe to stop and restart, processing only rows not yet done.
- **Throttle** if needed, so the backfill doesn't overload the database.
- **Handle unusual existing data:** nulls, duplicates, and legacy formats that new constraints won't accept.
- **Verify afterwards:** count rows that still need processing, and check that the data meets the new rules.

## 7. Reversibility

- Write a reverse (down) migration where the tool supports it and the change is reversible, and test that it works.
- Some changes can't be truly reversed: dropping a column loses its data. For these, make sure the data is no longer needed, or back it up first.
- Rolling back application code is usually easier than rolling back a schema. Expand and contract keeps code rollbacks safe, because the schema supports both versions.

## 8. Checklist

- [ ] The migration works with both the currently running code and the new code.
- [ ] No column or table still used by running code is renamed or dropped.
- [ ] Destructive changes are in a separate, later release.
- [ ] Large tables use non-blocking operations, and a lock timeout is set.
- [ ] Large data changes are batched and run separately.
- [ ] Existing data that might violate new constraints has been handled.
- [ ] New queries are supported by indexes.
- [ ] A way back exists: a down migration, or a recovery plan.
- [ ] The migration has been run against realistic data.