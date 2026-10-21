# Code Smells

A code smell is a sign in the code that something may be wrong with its design. A smell is not a bug and not always a problem; it is a reason to look more closely. Most of these come from Martin Fowler and Kent Beck's catalogue in *Refactoring*.

Use this file when reviewing code, when improving existing code, or when a change feels harder than it should be. For each smell: what it looks like, why it matters, and the usual fix.

## Contents

1. Bloaters: code that has grown too large
2. Change preventers: code that makes changes spread
3. Couplers: code that is too entangled
4. Dispensables: code that shouldn't exist
5. Misused language features
6. Choosing what to fix

## 1. Bloaters

| Smell | Looks like | Usual fix |
|---|---|---|
| Long function | Hard to understand in one reading; mixes several tasks or levels of detail | Extract meaningful steps into named functions |
| Large class or module | Many unrelated fields and methods; changes for many reasons | Split by responsibility |
| Long parameter list | Four or more parameters, especially of the same type | Group related parameters into an object; pass the object that already has the data |
| Data clumps | The same few values always travel together (`street, city, postcode`) | Create a type for them (`Address`) |
| Primitive obsession | Business concepts as plain strings and numbers (money as a float, email as a string) | Create small types for the concepts (`Money`, `EmailAddress`) |

## 2. Change preventers

| Smell | Looks like | Usual fix |
|---|---|---|
| Divergent change | One module changes for many unrelated reasons | Split it so each part has one reason to change |
| Shotgun surgery | One logical change requires small edits in many places | Move the scattered pieces into one module |
| Repeated switches | The same `switch` or `if` chain on a type appears in several places | Replace with polymorphism or a single lookup table |

## 3. Couplers

| Smell | Looks like | Usual fix |
|---|---|---|
| Feature envy | A function uses another object's data more than its own | Move the function to the object whose data it uses |
| Inappropriate intimacy | Modules reach into each other's internals | Go through public interfaces; move code to where it belongs |
| Message chains | `a.getB().getC().getD().doThing()` | Ask the first object for what you need; hide the chain |
| Middle man | A class that only forwards calls to another | Remove it and call the target directly |
| Global mutable state | Shared variables changed from many places | Pass state explicitly; contain changes in one owner |

## 4. Dispensables

| Smell | Looks like | Usual fix |
|---|---|---|
| Duplicated code | The same logic in several places | If it's the same rule appearing a third time, extract it; if the copies represent different rules, leave them |
| Dead code | Unused functions, variables, parameters, branches, imports | Delete it |
| Speculative generality | Abstractions, hooks, or parameters "for later" that nothing uses | Remove them; add them when actually needed |
| Lazy class | A class or module that does too little to justify existing | Merge it into its caller or neighbour |
| Comments covering for code | Comments explaining confusing code | Rename and restructure so the comment becomes unnecessary |
| Data class | A type with only fields, while the logic about it lives elsewhere | Move the related behaviour onto the type, where it suits the design |

## 5. Misused language features

| Smell | Looks like | Usual fix |
|---|---|---|
| Magic numbers and strings | `if status == 3`, `timeout = 86400` | Named constants or enumerations |
| Boolean flag arguments | `save(user, true, false)` | Separate functions or named options |
| Deep nesting | Logic four or more levels deep | Guard clauses, extracted functions |
| Temporary field | A field that's only set in certain situations | Move it into a separate object for that situation |
| Refused bequest | A subclass ignores or overrides most of what it inherits | Replace inheritance with composition |
| Swallowed errors | Empty catch blocks; errors logged and ignored | Handle, propagate, or explicitly document why it's safe (see `error-handling`) |
| Mutable shared data | An object changed by several parts of the code | Make it immutable; return new values |

## 6. Choosing what to fix

Not every smell needs fixing, and not now.

- **Fix smells in the code you're changing** when the fix makes your change easier or clearer.
- **Mention smells elsewhere** rather than fixing them inside an unrelated change.
- **Prioritise smells in code that changes often.** A smell in stable, rarely touched code costs little.
- **Weigh the fix against its risk.** If there are no tests, a refactor may cause more harm than the smell. Add tests first, or leave it and flag it.