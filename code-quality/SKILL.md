---
name: code-quality
description: Line-by-line standards for writing code that is clear, readable, and easy to change, in any language. Covers naming, functions, conditionals, variables, comments, consistency with the existing codebase, keeping changes focused, recognising code smells, and refactoring safely. Use this skill whenever writing, editing, or refactoring any code, cleaning up or improving existing code, naming things, or reviewing code for readability, even for small changes or when the task does not mention quality.
---

# Code Quality

Code is read many more times than it is written, by teammates, by future you, and by other agents. Quality code lets a reader understand what it does and change it safely, quickly, and without surprises. This skill covers code at the level of names, functions, and lines. For how to structure modules and layers, apply the `software-design` skill.

Read the project's `AGENTS.md` first. Follow its formatter, linter, naming conventions, and style. Where the project's conventions differ from this skill, follow the project.

## Before you start

1. **Look at the surrounding code.** How are similar things named, structured, and tested here? Match it.
2. **Know the project's tools.** Find the format, lint, and test commands in `AGENTS.md`. You will run them before finishing.
3. **Define the scope.** What exactly needs to change? Keep everything else untouched.

## Core rules

### 1. Match the codebase

Consistency matters more than personal preference. A codebase that follows one style everywhere is easier to read than one with several good styles mixed together.

- Use the conventions already present: naming style, file layout, error patterns, library choices.
- Let the project's formatter decide formatting. Don't hand-format against it.
- Don't introduce a new library, pattern, or style for something the codebase already has a way of doing.

### 2. Name things precisely

Names are the most important documentation in code.

- Name for meaning: `unpaidInvoices`, not `list2` or `data`.
- Functions are verbs that say what they do: `calculateTax`, `sendReminder`. Booleans read as questions: `isActive`, `hasAccess`, `canRetry`.
- Use the business's terms, consistently. One word per concept across the codebase.
- Name length should match scope: short names for tiny scopes (`i` in a short loop), descriptive names for wide ones.
- Avoid vague words that hide meaning: `manager`, `handler`, `processor`, `helper`, `util`, `data`, `info`, `temp`, unless the codebase uses them as established terms.
- If you struggle to name something, it may be doing too many things. Reconsider its design.

For more depth, read `references/naming.md`.

### 3. Write functions that do one thing clearly

- A function should do one coherent task at one level of detail. Don't mix high-level steps with low-level details in the same body.
- Keep parameters few. Beyond three, group related ones into an object with named fields.
- Avoid boolean flag parameters that switch behaviour (`render(true)`). Write two functions, or pass a named option.
- A function either answers a question or performs an action, not both. A function named `getUser` should not also create one.
- Make side effects visible. A function called `validateOrder` must not save or send anything.

Length alone is not a problem. A clear function that reads top to bottom is better than the same logic scattered across many tiny ones.

For more depth, read `references/functions.md`.

### 4. Keep control flow flat and obvious

- Use early returns (guard clauses) for invalid cases, so the main path isn't buried in nesting.
- Keep nesting shallow, ideally no more than two or three levels.
- Give complex conditions a name: `isEligibleForRefund = order.isPaid and withinRefundWindow(order)`.
- Prefer positive conditions: `if isValid` over `if not isInvalid`.
- Replace magic numbers and strings with named constants: `MAX_LOGIN_ATTEMPTS`, not `5`.

### 5. Keep variables close and few

- Declare variables near their first use, in the smallest scope possible.
- Prefer values that don't change once set. Reassigning a variable for several purposes makes code hard to follow.
- Don't reuse a variable for a different meaning. Create a new, well-named one.

### 6. Prefer clarity to cleverness

- Write the straightforward version. Dense one-liners, tricks, and unusual language features cost every future reader time.
- Be explicit. Avoid hidden behaviour, implicit type conversion, and action at a distance.
- Optimise for performance only where measurement shows a real need, then explain the optimisation in a comment.

### 7. Comment the why, not the what

- Good code explains *what* it does through names and structure. Comments explain *why*: the reason behind a non-obvious decision, a business rule, a workaround, a trade-off, or a danger.
- Document the public interface of modules: what it does, its inputs and outputs, and how it fails.
- Delete comments that repeat the code, and never leave commented-out code. Version control remembers it.
- Keep comments true. Update or remove any comment your change makes inaccurate.
- A `TODO` must say what is needed and why, and should reference a ticket where the project uses them.

For more depth, read `references/comments.md`.

### 8. Keep changes focused

- Change only what the task requires. Don't reformat, rename, or restructure unrelated code in the same change.
- If you notice a separate problem, mention it rather than fixing it silently, or fix it in a separate, clearly described change.
- Leave the code you touch a little cleaner than you found it, within the scope of your change (the "Boy Scout Rule").
- Remove anything you added temporarily: debug prints, test data, unused imports, unused variables, and dead code.

### 9. Refactor in small, safe steps

- Refactoring changes structure without changing behaviour. Keep the two separate: don't change behaviour and structure in the same step.
- Make sure tests cover the behaviour before restructuring. If they don't, add them first.
- Take small steps and run tests after each one.
- When a change is hard to make, first refactor so the change becomes easy, then make the easy change.

For a catalogue of problems and how to fix them, read `references/code-smells.md`. For safe techniques, read `references/refactoring.md`.

### 10. Finish properly

Before considering the work done:

1. Run the formatter, linter, and tests from `AGENTS.md`. Fix what they report; don't silence warnings without reason.
2. Read your full change as a reviewer would. Is every name clear? Is anything left over from experiments?
3. Check that the change does exactly what was asked, and no more.

## Example

**Poor:**

```
function proc(d, f):
    r = []
    for x in d:
        if x.s == 2:
            if f:
                if x.t > 1000:
                    r.append(x.t * 0.95)
                else:
                    r.append(x.t)
            else:
                r.append(x.t)
    return r
```

A reader can't tell what `d`, `f`, `s == 2`, `1000`, or `0.95` mean, and the logic is buried three levels deep.

**Better:**

```
SHIPPED = OrderStatus.Shipped
LARGE_ORDER_THRESHOLD = 1000
LARGE_ORDER_DISCOUNT = 0.05

function shippedOrderTotals(orders, applyLargeOrderDiscount):
    shippedOrders = orders.filter(order => order.status == SHIPPED)
    return shippedOrders.map(order => totalAfterDiscount(order, applyLargeOrderDiscount))

function totalAfterDiscount(order, applyLargeOrderDiscount):
    if not applyLargeOrderDiscount or order.total <= LARGE_ORDER_THRESHOLD:
        return order.total
    return order.total * (1 - LARGE_ORDER_DISCOUNT)
```

Same behaviour, but the names explain the purpose, the rules are visible, and the main path is flat. A next step, if the discount is used elsewhere, would be passing a named option or separating the two cases, so the boolean parameter disappears.

## Common mistakes

- Vague names such as `data`, `result`, `item`, `handle`, `process`, `temp`.
- Deep nesting where guard clauses would flatten the code.
- Magic numbers and strings without named constants.
- Comments that restate the code, or that no longer match it.
- Commented-out code and leftover debug output.
- Boolean parameters that silently change what a function does.
- Functions whose names hide their side effects.
- Reformatting or renaming unrelated code inside a feature change.
- Introducing a new style or library when the codebase already has one.
- Splitting readable code into many tiny functions that must be read together.
- Disabling a linter warning instead of fixing its cause.

## Reference files

| File | Read when |
|---|---|
| `references/naming.md` | Naming modules, types, functions, variables, or booleans, or renaming |
| `references/functions.md` | Designing function signatures, parameters, return values, or side effects |
| `references/comments.md` | Writing comments, documentation, or TODOs |
| `references/code-smells.md` | Reviewing or improving existing code, or something feels hard to change |
| `references/refactoring.md` | Restructuring code without changing behaviour |

## Sources

Robert C. Martin, *Clean Code*; Martin Fowler, *Refactoring* (2nd edition); Steve McConnell, *Code Complete* (2nd edition); Dustin Boswell and Trevor Foucher, *The Art of Readable Code*; Andrew Hunt and David Thomas, *The Pragmatic Programmer*; John Ousterhout, *A Philosophy of Software Design*; Kent Beck, *Tidy First?*; Google Engineering Practices.