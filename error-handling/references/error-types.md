# Error Types and Classification

This file explains how to classify failures, how to design error types, and how to choose between error mechanisms.

## Contents

1. Expected outcomes and failures
2. Designing error types
3. Exceptions, result types, and error values
4. Defining errors out of existence
5. Absence: "not found" and null

## 1. Expected outcomes and failures

Every error falls into one of two broad groups, and mixing them up causes most error-handling problems.

**Expected outcomes** are part of normal operation. The function's contract includes them, and callers are expected to handle them.

- Invalid input
- Business rule violations (insufficient funds, slot already booked)
- Not found, not permitted, conflict

**Failures** mean something is wrong with the system or its environment.

- Infrastructure problems: network, database, disk, external services. These may be **transient** (likely to succeed if tried again) or **permanent** (will keep failing until someone fixes something).
- Bugs: broken assumptions, impossible states, programming mistakes.

How the groups differ in practice:

| | Expected outcomes | Failures |
|---|---|---|
| Who handles it | The caller, often the user | The top-level handler, operators, developers |
| Logged as error? | No (at most informational) | Yes, with full detail |
| Shown to the user | A specific, helpful message | A generic message with a reference ID |
| Retry? | No | Only if transient and safe to repeat |
| Alert? | No | Yes, if persistent or severe |

## 2. Designing error types

**Make errors distinguishable by type or code, not by message text.**

```
Avoid:
    raise Error("insufficient funds")
    # Callers must check: if error.message contains "insufficient"...

Prefer:
    raise InsufficientFunds(accountId, available, requested)
    # Callers check the type or a stable code.
```

**Carry structured data.** Include the values a caller or developer needs: identifiers, limits, the conflicting value. Don't make anyone parse a message to get them.

**Give each error a stable code** if errors cross a boundary (an API, a queue, a log search). Codes such as `INSUFFICIENT_FUNDS` stay stable while messages are reworded.

**Keep a small, shallow hierarchy.** A handful of base categories lets boundary code handle whole groups at once:

```
AppError
├── ValidationError         → 400 / 422
├── NotFoundError           → 404
├── PermissionError         → 403
├── ConflictError           → 409
├── BusinessRuleError       → 422
└── (anything else)         → 500
```

Specific errors (`InsufficientFunds`, `EmailAlreadyRegistered`) extend the right category. The edge of the system maps categories to responses, so adding a new specific error needs no new mapping.

**Place error types with their module.** `InsufficientFunds` belongs to the payments or accounts module, as part of its public interface. Only the generic categories are shared.

## 3. Exceptions, result types, and error values

Languages offer different mechanisms. Follow the project's choice and be consistent.

| Mechanism | Common in | Strengths | Watch out for |
|---|---|---|---|
| Exceptions | Java, C#, Python, JavaScript, Ruby | Failures propagate automatically; clean happy path | Invisible in signatures; easy to forget or over-catch |
| Result / Either types | Rust, Kotlin, Swift, F#, functional TypeScript | Expected outcomes are explicit in the signature; compiler forces handling | More verbose; must be propagated by hand or with language support |
| Error return values | Go, C | Explicit and simple | Easy to ignore; repetitive |

A widely used combination, where the language allows:

- Use **result types or explicit return values for expected outcomes**, so callers can see and must handle them.
- Use **exceptions (or panics) for failures and bugs**, which most code can't handle and should let propagate to the top level.

Whatever the mechanism, never ignore a returned error, and never use exceptions for ordinary control flow such as ending a loop.

## 4. Defining errors out of existence

Sometimes the best error handling is to design the error away (John Ousterhout, *A Philosophy of Software Design*).

- **Make the operation's meaning cover the edge case.** "Delete this file" can mean "make sure this file doesn't exist", so deleting an already-missing file succeeds instead of failing.
- **Accept a wider input range sensibly.** A substring function can clamp out-of-range indices instead of raising errors, if that matches what callers want.
- **Use types that make the error impossible** (see `software-design`, "Make illegal states unrepresentable"). An `EmailAddress` type means no function downstream ever needs an "invalid email" error.

Only do this when the new meaning is genuinely what callers want. Hiding real problems is swallowing errors under another name.

## 5. Absence: "not found" and null

"Nothing found" is often an expected outcome, not an error. Decide per function, and make the choice clear:

- A lookup that may legitimately find nothing (`findUserByEmail`) returns an explicit "absent" value: the language's optional or nullable type.
- A lookup where absence means the caller made a mistake (`getUserById` with an ID that should exist) raises a not-found error.
- Never use `null` to mean both "not found" and "something went wrong".
- Handle absent values at the point they're returned. Passing possibly-null values deep into the code leads to crashes far from the cause.