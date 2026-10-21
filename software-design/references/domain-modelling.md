# Domain Modelling

Domain modelling is shaping code around the concepts and rules of the business. Good models make correct code easy to write and incorrect code hard to write.

## Contents

1. Use the language of the business
2. Make illegal states unrepresentable
3. Parse, don't validate
4. Functional core, imperative shell
5. Immutability
6. Where business rules live

## 1. Use the language of the business

Name types, functions, and modules after the concepts the business uses: `Invoice`, `refundOrder`, `SubscriptionRenewal`, not `DataObject`, `processItem`, `Handler2`.

- When the business has a precise term, use it consistently everywhere: code, database, API, documentation.
- If the business uses one word for two different things, give them separate types in code.
- Code that reads like a description of the business is easier to check against requirements.

## 2. Make illegal states unrepresentable

Design types so invalid data can't exist in the first place.

**Use specific types instead of loose ones:**

```
Avoid:   status: string                     # "pending", "Pending", "paid ", "shiped" all possible
Prefer:  status: Pending | Paid | Shipped   # only valid values exist
```

**Replace combinations of flags with explicit states:**

```
Avoid:
    isVerified: boolean
    isSuspended: boolean
    verifiedAt: date or null
    # Is "suspended but not verified" valid? Is "verified with no date" valid?

Prefer:
    AccountState =
        Unverified
      | Active(verifiedAt)
      | Suspended(verifiedAt, reason)
```

**Give important values their own types:** an `EmailAddress` that can only be created from a valid email, a `Money` that carries its currency, an `OrderId` that can't be confused with a `UserId`.

The language's type system should catch the mistake before the code runs. Where the language has weak types, use constructors or factory functions that refuse invalid values.

## 3. Parse, don't validate

Validation checks data and returns yes or no, leaving the data as it was. The rest of the code can't tell whether it was checked, so checks get repeated or forgotten.

Parsing checks data and converts it into a trusted type. Once parsed, the type itself proves the data is valid.

```
Validate (avoid):
    if isValidEmail(input.email): register(input.email)   # register receives a plain string

Parse (prefer):
    email = EmailAddress.parse(input.email)   # fails here if invalid
    register(email)                           # register only accepts EmailAddress
```

Rules:

- Parse all external input once, at the boundary: request bodies, query parameters, environment variables, file contents, responses from other services.
- Inside the system, pass the parsed types, never the raw input.
- Use the project's schema or validation library to do the parsing where one exists.

## 4. Functional core, imperative shell

Separate deciding from doing.

- The **core** contains business decisions as pure functions: given inputs, return outputs, with no database calls, network calls, clock reads, or random numbers.
- The **shell** gathers inputs (loads data, reads the time), calls the core, and carries out the results (saves data, sends messages).

```
# Core: pure, easy to test with plain values
calculateRefund(order, today) -> Refund or RefundNotAllowed

# Shell: does the input and output
refundOrder(orderId):
    order = orderRepository.find(orderId)
    result = calculateRefund(order, clock.today())
    if result is Refund:
        payments.refund(result)
        orderRepository.save(order.markRefunded())
    return result
```

The core holds the logic most likely to be wrong and most important to get right, and it can be tested exhaustively with no setup. Pass time, randomness, and configuration in as values so the core stays pure.

## 5. Immutability

Prefer data that doesn't change after creation. To "modify" it, create a new value.

- Shared mutable data is a common source of bugs: one part of the code changes data another part is still using.
- Immutable values are safe to pass anywhere, cache, and compare.
- Mutate only where it's clearly needed for performance or the language's idioms, and keep the mutation local.

## 6. Where business rules live

Put business rules in the business logic layer, close to the data they govern.

- A rule about an order ("an order can't be cancelled after shipping") belongs with the order model or order service, not in a controller, a UI component, or a database trigger.
- One rule, one place. If the same rule is checked in the UI, the API, and a background job, implement it once in business logic and call it from each.
- Database constraints (unique, not null, foreign keys) are a safety net that backs up business rules, not a replacement for them.