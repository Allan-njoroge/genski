# Functions

Functions are the basic unit of behaviour. A well-designed function can be understood from its name and signature and trusted to do exactly what they say.

## Contents

1. One task, one level of detail
2. Parameters
3. Return values
4. Side effects
5. Guard clauses and control flow
6. Size

## 1. One task, one level of detail

A function should do one coherent task. A good test: can you describe it in one sentence without "and"?

Keep each function at one level of detail. High-level steps and low-level details don't mix well:

```
Mixed (avoid):
    publishArticle(article):
        validate(article)
        html = article.body.replace("\n\n", "</p><p>").replace("**", "<b>")   # low-level detail
        save(article, html)
        notifySubscribers(article)

Consistent (prefer):
    publishArticle(article):
        validate(article)
        html = renderToHtml(article.body)
        save(article, html)
        notifySubscribers(article)
```

The second reads as a list of steps. Details live one level down.

## 2. Parameters

- **Fewer is better.** Zero to two parameters is ideal; three is acceptable; more suggests the function does too much or the parameters belong together.
- **Group related values.** `createBooking(guest, room, checkIn, checkOut, adults, children)` becomes `createBooking(guest, room, stay)`, where `stay` holds the dates and party size.
- **Avoid boolean flags.** `exportReport(report, true)` hides meaning at the call site. Use two functions (`exportReportAsPdf`, `exportReportAsCsv`) or a named option (`exportReport(report, format: Pdf)`).
- **Avoid output parameters.** Return results instead of modifying arguments.
- **Order consistently.** Put the main subject first, and keep similar functions in the same order.
- **Accept the narrowest type needed.** A function that needs an email address should take an email, not an entire user.

## 3. Return values

- Return one kind of thing. A function that returns a user, `null`, `false`, or an error string depending on the case is hard to use correctly.
- Make absence explicit. Use the language's option or nullable types, or a clear "not found" result, consistently.
- Return errors the way the project does (exceptions, result types, error values). Apply the `error-handling` skill.
- Prefer returning new values over modifying inputs.

## 4. Side effects

A side effect is anything a function does beyond returning a value: writing to a database, sending a message, changing shared state, writing to a file, logging.

- **Names must reveal side effects.** `checkPassword` must not also start a session. If it does, name it `authenticateAndStartSession`, or split it.
- **Command-query separation:** a function either changes something (a command) or answers a question (a query), not both. `getOrCreateUser` is an acceptable, explicitly named exception.
- **Keep side effects at the edges.** Do decision-making in pure functions and perform side effects separately (see the `software-design` skill, "Functional core, imperative shell").
- **Avoid hidden shared state.** Functions that read or change global or module-level variables are hard to test and reason about. Pass what is needed in.

## 5. Guard clauses and control flow

Handle invalid and special cases first and return early. The main logic then sits at the lowest nesting level.

```
Nested (avoid):
    applyCoupon(order, coupon):
        if coupon is not null:
            if not coupon.isExpired():
                if order.total >= coupon.minimumSpend:
                    order.applyDiscount(coupon.discount)
                    return Applied
                else:
                    return BelowMinimum
            else:
                return Expired
        else:
            return NoCoupon

Guard clauses (prefer):
    applyCoupon(order, coupon):
        if coupon is null: return NoCoupon
        if coupon.isExpired(): return Expired
        if order.total < coupon.minimumSpend: return BelowMinimum

        order.applyDiscount(coupon.discount)
        return Applied
```

Other control-flow guidelines:

- Replace long chains of conditions on a type or kind with a lookup table, or with polymorphism when the variation is real and growing.
- Avoid deeply nested loops. Extract the inner loop into a named function or use collection operations.
- Use the language's collection operations (filter, map, reduce) where they make intent clearer, and plain loops where they read better.

## 6. Size

There is no ideal number of lines. Judge by these instead:

- Can the function be understood in one reading?
- Does it do one task?
- Does it stay at one level of detail?

A 40-line function that reads top to bottom as one clear process is fine. A 10-line function that mixes three concerns is not. Extract a helper when it has a meaningful name and is understandable alone; don't extract fragments only to reduce line count.