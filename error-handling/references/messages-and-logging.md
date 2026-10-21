# Error Messages and Logging

The same error needs two very different descriptions: one for the person using the system, and one for the person fixing it.

## Contents

1. Messages for users
2. Messages for developers
3. Consistent API error responses
4. Logging errors
5. What never to include

## 1. Messages for users

A good user-facing message says:

1. **What happened**, in plain language.
2. **Why**, if it helps and is safe to say.
3. **What to do next.**

| Poor | Better |
|---|---|
| "Error 500" | "Something went wrong on our side. Please try again. Reference: 8f3a2c." |
| "Invalid input" | "Enter a phone number with 10 to 13 digits." |
| "Constraint violation: users_email_key" | "That email is already registered. Try signing in instead." |
| "You entered an invalid date!" | "Choose a delivery date at least two days from today." |

Guidelines:

- Be specific about what to fix, and attach validation messages to the field they concern.
- Use a calm, neutral tone. Don't blame the user.
- Don't expose internal terms, codes, or technical detail.
- Keep wording consistent across the product.
- For security-sensitive flows, such as sign-in and password reset, give the same message whether or not the account exists ("If an account exists for this email, we've sent a reset link.").

## 2. Messages for developers

Developer-facing messages, in error objects and logs, should be precise and full of context:

```
Poor:   "Failed"
Poor:   "Something went wrong in the payment service"
Better: "Charge declined by payment provider for order 9921 (amount 4,500 KES, provider code card_declined)"
```

Include: the operation, the relevant identifiers, the key parameters, and the cause. Avoid vague words like "something", "failed", or "error occurred" on their own.

## 3. Consistent API error responses

Every error from an API should share one format. RFC 9457 ("Problem Details for HTTP APIs") is a widely used standard; follow the project's format if it has one.

```
{
  "type": "https://example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "The account balance is lower than the transfer amount.",
  "code": "INSUFFICIENT_FUNDS",
  "requestId": "8f3a2c91"
}
```

For validation errors, list each problem with the field it concerns:

```
{
  "title": "Validation failed",
  "status": 422,
  "code": "VALIDATION_FAILED",
  "errors": [
    { "field": "email", "message": "Enter a valid email address." },
    { "field": "amount", "message": "Enter an amount greater than zero." }
  ],
  "requestId": "8f3a2c91"
}
```

Rules:

- A stable machine-readable `code` that clients can rely on, separate from the human-readable message.
- A request or correlation ID that matches the server logs.
- The same structure for every error, from every endpoint.

## 4. Logging errors

**Log once.** Log an error where it is finally handled, usually the top-level handler. If every layer logs and rethrows, one failure produces many log entries and obscures the real story.

**Log at the right level:**

| Level | Use for |
|---|---|
| Error | Failures that need attention: bugs, permanent infrastructure failures, persistent transient failures |
| Warning | Something unusual the system recovered from: a retry that succeeded, a fallback used |
| Info | Normal significant events, including expected outcomes if worth recording |
| Debug | Detail useful when investigating; usually off in production |

Invalid input and business rule violations are normal operation. Logging them as errors buries real problems and causes false alerts.

**Log with structure and context.** Use structured logging (key-value or JSON) as the project does, and include:

- The error type, message, and full cause chain with stack trace.
- The request or correlation ID.
- The operation and relevant identifiers (order ID, account ID).
- Enough information to reproduce the problem.

## 5. What never to include

In user messages, API responses, **or** logs:

- Passwords, tokens, API keys, session IDs, or other secrets.
- Full payment card numbers, security codes, or bank credentials.
- Unnecessary personal data. Log identifiers instead of names, addresses, or contents where possible.

In user messages and API responses only:

- Stack traces, SQL statements, file paths, internal hostnames, or library and framework error text.
- Whether a specific user, email, or resource exists, where that is sensitive.
- Internal system names or version numbers.