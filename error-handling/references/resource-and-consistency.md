# Resources, Consistency, and Asynchronous Errors

When an operation fails partway through, it must not leave resources open or data half-changed. This file covers cleanup, atomic changes, partial failure across systems, and errors in asynchronous and parallel code.

## Contents

1. Releasing resources
2. Atomic changes
3. Partial failure across systems
4. Idempotency
5. Asynchronous errors
6. Parallel operations
7. Background work

## 1. Releasing resources

Anything acquired must be released on every path, including failures: files, database connections, network sockets, locks, temporary files, subscriptions.

Use the language's guaranteed-cleanup construct rather than manual cleanup at the end of a function:

| Language | Construct |
|---|---|
| Python | `with` statements, `try/finally` |
| JavaScript / TypeScript | `try/finally`; `using` where supported |
| Java | `try-with-resources` |
| C# | `using` |
| Go | `defer` |
| Rust | Ownership and `Drop` (automatic) |

```
Poor:
    file = open(path)
    data = parse(file.read())     # if this fails, the file is never closed
    file.close()

Better:
    with open(path) as file:      # closed on every path
        data = parse(file.read())
```

Cleanup code must not hide the original error. If cleanup itself can fail, report that failure separately and keep the original as the primary error.

## 2. Atomic changes

When several changes must succeed or fail together, make them atomic.

- Within one database, use a transaction. Keep it as short as possible and don't make network calls to other systems inside it.
- Group changes so that the system is consistent at every point another process could observe it.
- Prefer designs where a single write captures a whole change, such as writing one record that represents the operation.

## 3. Partial failure across systems

When an operation spans systems that can't share a transaction (your database plus a payment provider, an email service, or a message queue), some steps can succeed while others fail.

Strategies:

- **Order steps to minimise damage.** Do the step that is hardest to undo last, or do the step that is easiest to check first.
- **Record intent before acting.** Save a "pending" record, perform the external step, then mark it complete. If the process crashes, the pending record shows what needs checking.
- **Compensate.** If a later step fails, undo earlier ones with explicit reverse actions (refund the charge, release the reservation).
- **Use the outbox pattern** for "save data and publish an event": write the event to an outbox table in the same transaction as the data, and publish it from there separately. On the backend, see the `backend-data` skill.
- **Reconcile.** For critical flows such as payments, run periodic checks that compare your records with the external system and repair differences.

## 4. Idempotency

An operation is idempotent if doing it twice has the same effect as doing it once. Idempotency makes failures much easier to handle, because uncertain operations can simply be retried.

- Prefer "set" over "add": `setStatus(Paid)` is safe to repeat; `incrementBalance(100)` is not.
- For operations that create things or move money, accept an idempotency key and ignore repeats of the same key.
- Message handlers and background jobs must be idempotent, because messages can be delivered more than once.

## 5. Asynchronous errors

Errors in asynchronous code are easily lost.

- Await or otherwise observe every asynchronous operation. A promise, future, or task that is started and never awaited can fail silently.
- Don't start "fire and forget" work without attaching an error handler that logs failures.
- Ensure the application has a process-wide handler for unhandled asynchronous errors that logs them with full detail.
- Callback-style APIs must check the error argument every time.

## 6. Parallel operations

When running several operations at once, decide what one failure means:

- **All required:** if one fails, cancel or ignore the others and fail the whole operation.
- **Best effort:** let all finish, collect successes and failures, and report each failure clearly.

Use the language's construct that matches the decision (for example, `Promise.all` versus `Promise.allSettled` in JavaScript). Never let failures from some operations disappear because others succeeded.

## 7. Background work

Background jobs, scheduled tasks, and message consumers have no user waiting for an answer, so failures must be visible in other ways.

- Wrap each job in a handler that logs failures with the job's identifiers and inputs.
- Retry transient failures a limited number of times with increasing delays.
- Move work that keeps failing to a dead-letter queue or failed-jobs table where it can be inspected and replayed, rather than retrying forever or dropping it.
- Alert when failures accumulate.