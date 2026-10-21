# Test Design

This file covers how to decide what to test, how to structure and name tests, and how to work test-first.

## Contents

1. Choosing test cases
2. Structure: Arrange, Act, Assert
3. Naming
4. Assertions
5. Readability over reuse
6. Test-driven development
7. Regression tests for bugs
8. Property-based testing
9. Tests for legacy code

## 1. Choosing test cases

Exhaustive testing is impossible. Choose cases systematically.

**Equivalence partitioning:** divide inputs into groups that the code should treat the same way, and test one value from each group.

For a function that grants free shipping on orders of 5,000 or more:

| Group | Example value |
|---|---|
| Below threshold | 1,200 |
| At or above threshold | 7,500 |
| Invalid (negative) | -50 |

**Boundary values:** bugs cluster at the edges of groups. Test exactly at, just below, and just above each boundary: 4,999, 5,000, and 5,001. Also consider zero, empty collections, a single item, maximum sizes, and the first and last items.

**Checklist for any behaviour:**

- The main success case
- Each boundary
- Each kind of invalid input
- Each business rule that can reject the operation
- Each dependency failure that the code handles
- Permission and ownership cases, where relevant
- Repeated calls, where idempotency matters

## 2. Structure: Arrange, Act, Assert

```
test "...":
    # Arrange: set up only what this behaviour needs
    # Act: perform exactly one action
    # Assert: check the outcome
```

Guidelines:

- **One act per test.** If a test acts, asserts, acts again, and asserts again, it's testing two behaviours. Split it.
- **Keep arrange short.** A long setup suggests the code under test has too many dependencies, or that a test data builder would help.
- **Blank lines or comments between the three sections** make them easy to see. Some teams use Given / When / Then; follow the project.

## 3. Naming

A test name should describe the behaviour and its condition, so a failure report reads as a sentence about what broke.

Common patterns, depending on the project's style:

- `"rejects transfer when balance is insufficient"`
- `"transfer_withInsufficientBalance_isRejected"`
- Nested descriptions: `describe("transfer") → describe("when balance is insufficient") → it("is rejected")`

Avoid names that describe the implementation (`"calls repository.save"`) or nothing at all (`"test1"`, `"works"`).

## 4. Assertions

- **Assert on outcomes the caller cares about:** return values, resulting state, responses, emitted events.
- **Be specific.** `expect(total).toBe(4750)` beats `expect(total).toBeGreaterThan(0)`.
- **Assert on the error type or code,** not on the exact wording of a message, unless the wording is the behaviour.
- **Don't recompute the expected value** with the same logic as the code. Write the expected value directly; if the logic is wrong in both places, the test passes anyway.
- **Make failures informative.** Use the framework's descriptive matchers so a failure shows expected and actual values.

## 5. Readability over reuse

In production code, removing duplication is usually good. In tests, clarity matters more. Tests should be **DAMP** ("Descriptive And Meaningful Phrases") rather than strictly DRY.

- A reader should understand a test without opening other files. Keep the important values visible in the test itself.
- Shared helpers are good for *how* to build things (`anOrder(...)`), not for hiding *what* the test sets up.
- Avoid large shared fixtures and setup blocks that many tests depend on. Changing them for one test breaks others, and readers can't see what each test relies on.

## 6. Test-driven development

Test-driven development (TDD), from Kent Beck, works in short cycles:

1. **Red:** write a small test for the next piece of behaviour. Run it and see it fail.
2. **Green:** write the simplest code that makes it pass.
3. **Refactor:** clean up the code and the test, keeping all tests passing.

Benefits:

- Every behaviour has a test, by construction.
- Seeing the test fail first proves it can detect the problem.
- Code designed to be tested tends to have clearer interfaces and fewer hidden dependencies.

TDD fits best when the expected behaviour is clear: business rules, calculations, parsing, and bug fixes. For exploratory work, where the shape of the solution is still unclear, it's reasonable to explore first, then write tests before the code is considered finished.

## 7. Regression tests for bugs

Every bug fix should come with a test that:

1. Reproduces the reported problem, at the lowest level that can show it.
2. Fails before the fix, for the reason described in the bug.
3. Passes after the fix.

Name it after the behaviour, not the ticket number alone: `"does not double-charge when payment confirmation is received twice"`. A ticket reference in a comment is helpful.

## 8. Property-based testing

Instead of specific examples, property-based tests state a rule that must hold for all inputs and let a library generate many inputs to try to break it.

Useful properties:

- **Round trips:** `parse(format(x)) == x`
- **Invariants:** a sorted list has the same items as the original, in order
- **Equivalence:** an optimised function gives the same result as a simple one
- **Idempotency:** applying an operation twice equals applying it once

Libraries exist for most languages (Hypothesis for Python, fast-check for JavaScript and TypeScript, jqwik for Java, and others). Use them where the project already does, or suggest them for parsing, serialisation, and calculations with wide input ranges.

## 9. Tests for legacy code

When code has no tests and needs to change (Michael Feathers, *Working Effectively with Legacy Code*):

1. **Write characterisation tests** that capture what the code currently does, even where the behaviour seems odd. They protect against unintended change.
2. **Find seams:** places where a dependency can be substituted without editing much code, such as a constructor parameter or a function argument.
3. **Make the smallest safe change** needed to get the code under test.
4. **Then change the behaviour**, with tests guarding everything else.