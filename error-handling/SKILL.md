---
name: error-handling
description: Principles for detecting, representing, propagating, and handling errors so that failures are visible, safe, and easy to diagnose, in any language. Covers failing fast, classifying errors, designing error types, where to catch and where to propagate, adding context, translating errors at system boundaries, user-facing messages, logging errors, cleaning up resources, partial failures, and asynchronous errors. Use this skill whenever writing code that can fail, including input handling, I/O, database or network calls, parsing, external services, background jobs, and API or UI responses, and whenever writing try/catch blocks, error types, or error responses, even if the task does not mention errors.
---

# Error Handling

Every piece of software fails sometimes: input is wrong, networks drop, disks fill, code has bugs. Good error handling doesn't prevent failure. It makes failure **visible**, **contained**, and **understandable**: the system stays in a consistent state, users get a useful response, and developers can find the cause quickly.

Read the project's `AGENTS.md` first. Use the project's error mechanism (exceptions, result types, or error values), its error types, its response format, and its logger. Where the project's conventions differ from this skill, follow the project.

## Before you start

For any operation you write, ask:

1. **What can fail here?** Invalid input, missing data, a broken rule, an unavailable dependency, a bug.
2. **Who can do something about each failure?** The user, the caller, a retry, an operator, or nobody.
3. **What state is left if this fails halfway?** Is anything partially written, open, or locked?
4. **What will the user see, and what will the developer see?** Both should be useful, and they are usually different.

## Core rules

### 1. Fail fast, at the boundary

Detect problems as early as possible, as close to their source as possible.

- Parse and validate external input when it enters the system: requests, files, messages, configuration, responses from other services. Reject invalid input immediately with a clear reason.
- Check configuration at startup. A missing setting should stop the application from starting, not crash it at 2 a.m. on the first request that needs it.
- Check assumptions explicitly. When code relies on something being true, assert it, so a violation fails loudly where it happens rather than corrupting data further along.

### 2. Know which kind of error you have

Different errors need different handling. Classify every failure:

| Kind | Example | Handling |
|---|---|---|
| **Invalid input** | Missing field, malformed email | Reject with a specific, actionable message. Not logged as an error. |
| **Business rule violation** | Insufficient balance, order already shipped | Return a specific, expected error the caller can act on. Not logged as an error. |
| **Not found / not allowed** | Record doesn't exist, user lacks permission | Return a specific error. Don't reveal more than the user may know. |
| **Transient infrastructure failure** | Timeout, connection reset, service temporarily unavailable | May be retried, if the operation is safe to repeat. Log if it persists. |
| **Permanent infrastructure failure** | Wrong credentials, missing table | Fail, log with full detail, alert. Retrying won't help. |
| **Bug / broken invariant** | Unexpected null, impossible state | Fail loudly. Log with full detail. Don't try to continue. |

The first three are **expected outcomes** of normal operation and part of the function's contract. The last three are **failures** of the system itself.

For more depth, read `references/error-types.md`.

### 3. Never swallow errors

An error that disappears silently is the hardest kind of bug to find.

- No empty catch blocks.
- Don't catch an error, log it, and carry on as if nothing happened, unless continuing is genuinely correct, and then explain why in a comment.
- Don't return a default value (empty list, zero, `null`) to hide a failure. An empty list means "there are none", not "the database was down".
- Don't catch broad error types (everything, all exceptions) except at the top-level boundary, where every error must be handled somehow.

### 4. Handle errors where something can be done

Catch an error only where you can do something meaningful with it: recover, retry, use a sensible fallback, add context, or translate it for the layer above. Otherwise, let it propagate.

- Most functions should not catch errors at all. They let errors pass up to the level that knows what to do.
- Business logic decides how to respond to business errors. Entry points decide how to present errors to the outside world.
- Every application needs a top-level handler that catches anything unhandled, logs it, and returns a safe, generic response. Errors should never escape to the user as raw crashes or stack traces.

For more depth, read `references/propagation.md`.

### 5. Add context, keep the cause

When an error passes up through layers, each layer that catches it should add what it knows and keep the original.

```
Poor:   raise Error("Failed")
Poor:   raise Error("Could not load invoice")              # original cause lost
Better: raise InvoiceLoadError("Could not load invoice 4821 for account 77", cause=originalError)
```

Include the identifiers and parameters needed to reproduce the problem. Use the language's mechanism for chaining causes, so the full trail is preserved.

### 6. Use specific error types, consistently

- Represent each distinct failure with its own type or code (`EmailAlreadyRegistered`, `PaymentDeclined`), not a generic error with a message string. Callers must be able to tell errors apart without parsing text.
- Use one error mechanism consistently, as the project does: exceptions, result types, or error values. Don't mix them for the same kind of failure.
- Make expected outcomes visible in function signatures where the language allows it.

### 7. Translate errors at the system's edge

Inside the system, errors carry business meaning. At the edge, translate them into what the outside world understands: HTTP status codes and response bodies, UI messages, CLI exit codes, message-queue acknowledgements.

- Do the translation in one place (error-handling middleware, a response mapper, a UI error boundary), not in every handler.
- Never expose internal details to users: stack traces, SQL, file paths, internal hostnames, library error text, or whether a particular account exists, where that is sensitive.
- Return a consistent error format everywhere. Include a reference ID that matches the logs, so support can find the details.

### 8. Write messages for their audience

- **For users:** what went wrong in plain language, and what they can do about it. "That email is already registered. Try signing in instead." Never blame the user; never expose internals.
- **For developers:** what failed, where, with which inputs, and why. Precise and full of context.

For more depth, read `references/messages-and-logging.md`.

### 9. Log once, with context, at the right level

- Log an error once, where it is finally handled, usually the top-level handler. Logging at every layer it passes through creates noisy, duplicated logs.
- Include context: what operation, which identifiers, the request or correlation ID, and the full cause chain.
- Use log levels honestly. Invalid input and business rule violations are not errors; they are expected. Reserve the error level for failures someone needs to look at.
- Never log secrets, passwords, tokens, full payment details, or unnecessary personal data.

### 10. Leave things clean and consistent

When something fails partway through, the system must not be left broken.

- Release resources on every path: close files, connections, and handles, and release locks. Use the language's guaranteed-cleanup construct (`finally`, `using`, `with`, `defer`, and similar).
- Make operations that must succeed or fail together atomic, using a transaction or an equivalent.
- When several external steps can't share a transaction, design for partial failure: make steps safe to repeat, and undo or compensate completed steps when a later one fails.

For more depth, read `references/resources-and-consistency.md`.

### 11. Retry only what is safe to retry

- Retry only transient failures, only for operations that are safe to repeat, with a limit and an increasing delay between attempts.
- Never retry invalid input, business rule violations, or bugs. They will fail the same way every time.
- On the backend, apply the `backend-reliability` skill for timeouts, retries, and circuit breakers.

### 12. Handle asynchronous errors explicitly

- Every asynchronous operation (promise, future, task, callback, background job, message handler) must have its errors handled or propagated. Unhandled asynchronous errors are often silently lost or crash the process.
- Background jobs and message consumers have no user to report to: log failures with full context, and route repeatedly failing work somewhere it can be inspected, such as a dead-letter queue.
- When running operations in parallel, decide what happens if one fails: cancel the others, or collect all results and report every failure.

### 13. Crash rather than corrupt

If the program reaches a state that should be impossible, stop that operation (or the process, for unrecoverable states) rather than continuing with data that may be wrong. A crash is visible and recoverable; silently corrupted data may not be.

### 14. Test the failure paths

Errors are part of a function's behaviour. Test them like any other behaviour: invalid input is rejected, business rules produce the right error, dependency failures are handled, cleanup happens, and responses don't leak internal details. Apply the `testing` skill.

## Example

**Poor:**

```
function transferFunds(fromId, toId, amount):
    try:
        from = accounts.find(fromId)
        to = accounts.find(toId)
        from.balance = from.balance - amount
        accounts.save(from)
        to.balance = to.balance + amount
        accounts.save(to)
        return true
    catch error:
        print(error)
        return false
```

Problems: the caller can't tell "insufficient funds" from "database down" from "account not found". If the second save fails, money disappears. A negative balance is allowed. The error is printed and lost. Nothing is in a transaction.

**Better:**

```
function transferFunds(fromId, toId, amount: Money):
    if amount <= 0: raise InvalidAmount(amount)
    if fromId == toId: raise SameAccountTransfer(fromId)

    transaction:
        from = accounts.find(fromId) or raise AccountNotFound(fromId)
        to = accounts.find(toId) or raise AccountNotFound(toId)
        if from.balance < amount:
            raise InsufficientFunds(fromId, available=from.balance, requested=amount)

        accounts.save(from.withdraw(amount))
        accounts.save(to.deposit(amount))
    # If anything inside fails, the transaction rolls back; no money is lost.
    # Infrastructure errors propagate to the top-level handler, which logs them once.

# At the edge (in one central place):
InvalidAmount, SameAccountTransfer  -> 422, "Enter an amount greater than zero." / ...
AccountNotFound                     -> 404, "Account not found."
InsufficientFunds                   -> 422, "Insufficient funds for this transfer."
anything else                       -> 500, "Something went wrong. Reference: <request ID>." (logged with full detail)
```

## Common mistakes

- Empty catch blocks, or catching, printing, and continuing.
- Returning `null`, `false`, an empty list, or a default value to hide a failure.
- Catching all errors in the middle of the code, where nothing useful can be done.
- Losing the original error when wrapping or rethrowing.
- Generic errors with only a message string, forcing callers to parse text.
- Logging the same error at every layer it passes through.
- Logging invalid input or expected business outcomes at error level.
- Returning stack traces, SQL, or library error messages to users.
- Leaving connections, files, or locks open when an error occurs.
- Multi-step changes without a transaction, leaving partial updates behind.
- Retrying errors that can never succeed, or retrying non-repeatable operations.
- Unhandled promise rejections or background job errors that vanish.
- Only testing the success path.

## Reference files

| File | Read when |
|---|---|
| `references/error-types.md` | Designing error types, choosing between exceptions and result types, or classifying failures |
| `references/propagation.md` | Deciding where to catch, how to wrap, or how to translate errors at boundaries |
| `references/messages-and-logging.md` | Writing error messages, error responses, or logging errors |
| `references/resources-and-consistency.md` | Handling cleanup, transactions, partial failure, or asynchronous and parallel errors |

## Sources

Andrew Hunt and David Thomas, *The Pragmatic Programmer* ("Dead programs tell no lies", "Crash early"); Michael Nygard, *Release It!*; John Ousterhout, *A Philosophy of Software Design* ("Define errors out of existence"); Robert C. Martin, *Clean Code*; Joe Armstrong, Erlang's "let it crash" philosophy; RFC 9457, *Problem Details for HTTP APIs*; OWASP Error Handling Cheat Sheet and OWASP Logging Cheat Sheet.