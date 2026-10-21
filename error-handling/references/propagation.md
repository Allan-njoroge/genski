# Propagating and Translating Errors

This file explains where errors should be caught, how they should travel through the layers of a system, and how to translate them at its edge.

## Contents

1. Where to catch
2. The journey of an error through the layers
3. Wrapping with context
4. Translating at boundaries
5. The top-level handler
6. Errors from external services

## 1. Where to catch

Catch an error only if, at that point, you can do one of these:

- **Recover:** use a cached value, a sensible fallback, or an alternative path, where that is genuinely correct.
- **Retry:** the failure is transient and the operation is safe to repeat.
- **Add context:** wrap it with information the caller doesn't have, then rethrow.
- **Translate:** convert it into the vocabulary of this layer, then rethrow.
- **Handle it finally:** at the system's edge, turn it into a response and log it.

If none apply, don't catch it. Let it propagate. Most functions in a well-designed system contain no error-catching code.

## 2. The journey of an error through the layers

**Data access layer.** Converts low-level, library-specific errors into meaningful ones where the meaning is clear: a unique-constraint violation on the email column becomes `EmailAlreadyRegistered`; "no rows" becomes not-found or absent. Other infrastructure errors propagate with context added. Data access never decides what the user sees.

**Business logic layer.** Raises business errors when rules are broken. Lets infrastructure errors pass through, unless it can genuinely recover. Contains no HTTP status codes, UI messages, or logging of errors it rethrows.

**Entry point / boundary layer.** Translates every error into the outside world's terms, in one central place. Logs failures once.

```
Database driver:  UniqueViolation(constraint="users_email_key")
       │  data access translates (meaning is clear)
       ▼
Repository:       EmailAlreadyRegistered(email)
       │  business logic lets it propagate (nothing to add)
       ▼
Service:          EmailAlreadyRegistered(email)
       │  boundary translates (one central mapping)
       ▼
HTTP response:    409 { code: "EMAIL_ALREADY_REGISTERED", message: "That email is already registered." }
```

## 3. Wrapping with context

When adding context, keep the original error attached as the cause, using the language's chaining mechanism (`cause`, `from`, `%w`, inner exception, and so on).

```
Poor:   catch error: raise ReportError("Report failed")                 # cause lost
Poor:   catch error: raise error                                         # nothing added, so why catch?
Better: catch error: raise ReportError("Generating monthly report for account 77, March 2026", cause=error)
```

Good context answers: what was being attempted, on which data, with which key parameters. Don't add context that repeats what the cause already says.

Don't wrap at every layer by reflex. Wrap where it adds information a developer would need to diagnose the problem.

## 4. Translating at boundaries

Every boundary between the system and the outside world needs a single, central translation:

| Boundary | Translate into |
|---|---|
| HTTP API | Status code and a consistent error body (see `messages-and-logging.md`) |
| User interface | A message for the user, and a recovery option where possible |
| Command-line tool | A clear message on standard error and a meaningful non-zero exit code |
| Message consumer | Acknowledge, retry later, or send to a dead-letter queue |
| Calls to another internal module | That module's public error types |

A typical HTTP mapping, by error category:

| Category | Status |
|---|---|
| Malformed request | 400 Bad Request |
| Not authenticated | 401 Unauthorized |
| Authenticated but not permitted | 403 Forbidden (or 404 if the resource's existence must not be revealed) |
| Not found | 404 Not Found |
| Conflict with current state | 409 Conflict |
| Validation or business rule failed | 422 Unprocessable Content (or 400, per project convention) |
| Rate limit exceeded | 429 Too Many Requests |
| Unexpected failure | 500 Internal Server Error |
| Dependency unavailable | 502 / 503 / 504 as appropriate |

Follow the project's existing mapping where one exists.

## 5. The top-level handler

Every entry point needs a final safety net that catches anything not handled elsewhere. It should:

1. Log the error once, at error level, with full cause chain, request or correlation ID, and relevant context.
2. Return a safe, generic response with a reference ID. Never internal details.
3. Keep the process healthy where possible. If the error indicates the process itself is in a broken state, let it shut down and be restarted by its supervisor.

Make sure the same safety net covers background jobs, scheduled tasks, message consumers, and unhandled asynchronous errors, not just web requests.

## 6. Errors from external services

Calls to other systems fail in more ways than local code:

- **Set a timeout on every call.** Without one, a slow dependency can hang your system.
- **Classify the response:** client errors (your request was wrong; don't retry), server errors and timeouts (possibly transient), and unexpected formats (treat as failure).
- **Translate into your own types** at the integration layer. The rest of the system should never see a third-party library's error classes.
- **Don't pass a dependency's error messages to your users.** They may contain internal details and won't make sense to them.
- **Decide on fallback behaviour explicitly:** fail the request, degrade gracefully, or use cached data. On the backend, apply the `backend-reliability` skill.