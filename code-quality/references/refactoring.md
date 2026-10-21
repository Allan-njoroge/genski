# Refactoring Safely

Refactoring is changing the structure of code without changing its behaviour. Done well, it is low-risk and continuous. Done badly, it introduces bugs and makes changes hard to review.

## Contents

1. Rules for safe refactoring
2. When to refactor
3. Common refactorings
4. Refactoring code without tests
5. Presenting refactoring changes

## 1. Rules for safe refactoring

1. **Separate structure from behaviour.** A refactoring step changes structure only. A feature or fix step changes behaviour only. Never both at once; when something breaks, you need to know which caused it.
2. **Have tests first.** Tests that cover the behaviour you're restructuring tell you immediately if you changed it. If they don't exist, write them before refactoring.
3. **Take small steps.** Rename, run tests. Extract a function, run tests. Move it, run tests. Small steps are easy to verify and easy to undo.
4. **Use tools where possible.** Automated rename, extract, and move operations from the editor or language server are safer than manual edits.
5. **Keep it working.** The code should build and the tests pass after every step.

## 2. When to refactor

- **Before a change:** "Make the change easy, then make the easy change" (Kent Beck). If the code resists the feature you need to add, restructure it first, then add the feature.
- **During understanding:** renaming things to reflect what you've learned helps the next reader.
- **After a change:** clean up what your change made untidy.
- **During review:** when a reviewer finds the code hard to follow.

Avoid large, standalone "rewrite everything" refactors. They're risky, hard to review, and often stall. Improve code gradually, in the areas you're working in.

## 3. Common refactorings

| Refactoring | Use when |
|---|---|
| Rename | A name is unclear or misleading |
| Extract function | A block of code has a clear purpose that can be named |
| Inline function | A function's body is as clear as its name, or a function is a pointless middle layer |
| Extract variable | A complex expression would be clearer with a name |
| Introduce parameter object | Several parameters always travel together |
| Replace magic value with constant | A literal number or string carries hidden meaning |
| Replace nested conditional with guard clauses | Nesting buries the main path |
| Decompose conditional | A complex condition would be clearer as a named function |
| Move function | A function uses another module's data more than its own |
| Split module | A module changes for several unrelated reasons |
| Replace conditional with polymorphism | The same type-based switch appears in several places |
| Replace inheritance with composition | A subclass doesn't fit its parent |
| Remove dead code | Code is unused |

Each is a small, well-understood transformation. Larger restructurings are sequences of these steps.

## 4. Refactoring code without tests

Legacy code often has no tests, and changing it without them is risky.

1. **Add characterisation tests:** tests that record what the code currently does, even if the behaviour seems wrong. They protect against accidental change.
2. **Make only the minimal, safest structural changes** needed to get the code under test, preferably using automated tools.
3. **Then refactor** with the tests as a safety net.
4. If adding tests isn't feasible in the current task, limit changes to what the task requires and flag the risk to the person you're working with.

## 5. Presenting refactoring changes

- Keep refactoring in its own commit or pull request, separate from behaviour changes, so reviewers can check it quickly.
- Describe it clearly: "Refactor: extract pricing logic from OrderService into PricingCalculator. No behaviour change."
- If the refactoring is large, break it into a sequence of small, reviewable changes.
- When working on a task, only refactor beyond its scope if the person you're working with agrees.