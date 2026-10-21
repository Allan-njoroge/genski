---
name: testing
description: Principles for writing automated tests that give real confidence, run fast, stay reliable, and survive refactoring, in any language or framework. Covers choosing the right test level (unit, integration, end-to-end), testing behaviour instead of implementation, test structure and naming, what cases to cover, test doubles and mocking, test data, avoiding flaky tests, regression tests for bugs, test-driven development, and coverage. Use this skill whenever writing, fixing, or reviewing tests, adding a feature or fixing a bug that needs tests, deciding what or how to test, or when tests fail, are slow, or are flaky, even if the task does not explicitly ask for tests.
---

# Testing

Tests exist to give confidence that the software works and to make change safe. A good test suite catches real problems, runs fast enough to run constantly, fails only when something is actually wrong, and keeps passing when code is restructured without changing behaviour. Tests that fail for no reason, break on every refactor, or pass while the software is broken are worse than useless: they cost time and teach people to ignore failures.

Read the project's `AGENTS.md` first. Use the project's test framework, file locations, naming style, helpers, and commands. Where the project's conventions differ from this skill, follow the project.

## Before you start

1. **What behaviour needs confidence?** State it in one sentence from the user's or caller's point of view: "A user with insufficient funds can't complete a transfer."
2. **How does this project test?** Look at existing tests for similar code. Match their structure, helpers, and style.
3. **Which level fits?** Pure logic → unit test. Code talking to a database, queue, or another module → integration test. A critical user journey → end-to-end test.
4. **How do I run them?** Find the test commands in `AGENTS.md`. You will run the relevant tests, then the full suite, before finishing.

## Core rules

### 1. Test behaviour, not implementation

Test what the code does, through its public interface, not how it does it internally.

- Assert on outputs, returned values, observable state, and effects visible to callers.
- Don't assert on private methods, internal call order, or intermediate variables.
- A good test keeps passing when the code is refactored without changing behaviour. If a harmless refactor breaks tests, the tests are coupled to the implementation.

### 2. Choose the right level

| Level | Tests | Speed | Use for |
|---|---|---|---|
| **Unit** | One unit of behaviour in isolation from slow or external dependencies | Milliseconds | Business rules, calculations, parsing, edge cases |
| **Integration** | Code together with real dependencies: database, queue, file system, other modules | Seconds | Data access, queries, module wiring, API endpoints |
| **End-to-end** | The whole system as a user experiences it | Slowest | A few critical journeys: sign up, checkout, payment |

Have many fast unit tests, a solid layer of integration tests, and few end-to-end tests. Test each behaviour at the lowest level that can give real confidence in it. Pure logic needs no database; a SQL query can't be trusted without one.

For more depth, read `references/test-levels.md`.

### 3. Keep tests FIRST

- **Fast:** slow tests get run less often, so problems are found later.
- **Independent:** each test sets up what it needs and can run alone, in any order, in parallel.
- **Repeatable:** the same result every time, on every machine. No dependence on the clock, network, random values, or leftover data.
- **Self-validating:** the test passes or fails on its own. No reading output to decide.
- **Timely:** written with the code, not weeks later.

### 4. One behaviour per test, clearly structured

Structure every test in three visible parts:

```
test "transfer is rejected when balance is insufficient":
    # Arrange
    account = anAccount(balance=100)

    # Act
    result = transfer(from=account, to=anotherAccount(), amount=150)

    # Assert
    expect(result).toBe(InsufficientFunds)
    expect(account.balance).toBe(100)
```

- Each test checks one behaviour. Several assertions are fine if they describe that one behaviour.
- No logic in tests: no loops, conditionals, or calculations that could themselves be wrong. Use the expected value directly.
- Prefer clear, repeated setup over clever shared setup. Each test should be understandable on its own, without scrolling to find hidden fixtures.

### 5. Name tests as statements of behaviour

A test name should say what behaviour is expected, so a failure explains itself.

```
Poor:   test1, testTransfer, transfer works
Better: "rejects transfer when balance is insufficient"
        "applies 5% discount to orders over 1,000"
        "returns 404 when the order does not exist"
```

### 6. Cover what matters

For each behaviour, consider:

- **The main success path.**
- **Boundaries:** zero, one, empty, maximum, just below and just above limits.
- **Invalid input:** missing, malformed, wrong type, out of range.
- **Business rule violations:** every rule that can reject an operation.
- **Failure of dependencies:** what happens when the database, network, or external service fails.
- **Security-relevant cases:** a user accessing another user's data, missing or invalid credentials.

You don't need every combination. Group inputs that behave the same way and test one from each group, plus the boundaries between groups.

For more depth, read `references/test-design.md`.

### 7. Use test doubles sparingly and at the edges

A test double (mock, stub, fake, spy) replaces a real dependency in a test.

- Replace things that are slow, external, non-deterministic, or have side effects: third-party APIs, email and payment services, the clock, random number generators.
- Prefer real objects for your own in-process code. Prefer fakes (simple working implementations, such as an in-memory repository) over mocks that script exact calls.
- Don't mock what you don't own. Wrap third-party libraries in your own adapter, mock the adapter in unit tests, and test the adapter against the real thing in integration tests.
- Too many mocks is a design signal: the code under test may have too many dependencies.
- Verify interactions (that something was called) only when the interaction *is* the behaviour, such as "an email was sent".

For more depth, read `references/test-doubles.md`.

### 8. Make test data explicit and minimal

- Each test creates its own data, showing only the values that matter to that test.
- Use builders or factories with sensible defaults, so tests specify only what's relevant: `anOrder(status=Shipped)`.
- Never depend on data left by another test, or on shared data that tests modify.
- Use obviously fake values for personal data, and never real customer data or production secrets.

### 9. Eliminate flakiness

A flaky test passes and fails without any code change. It destroys trust in the whole suite.

- Control time: inject a clock or use the framework's fake timers. Never depend on the current date.
- Control randomness: use fixed seeds or inject the random source.
- Never use fixed sleeps to wait for something. Wait for the actual condition, with a timeout.
- Don't call real external services in automated tests.
- Isolate data: unique identifiers, transactions rolled back after each test, or a fresh database per test run.
- Don't depend on ordering where none is guaranteed, such as unordered query results or map iteration.

For more depth, read `references/test-data-and-determinism.md`.

### 10. Reproduce every bug with a failing test first

When fixing a bug:

1. Write a test that reproduces the bug, and see it fail for the right reason.
2. Fix the code.
3. See the test pass.

The test proves the fix works and prevents the bug from returning.

### 11. Consider test-driven development

Writing the test before the code (red, green, refactor) clarifies what the code should do, produces testable designs, and guarantees tests exist. It works especially well for business logic and bug fixes. Use it where the project or the person you're working with prefers it, and wherever the expected behaviour is clear in advance.

### 12. Treat coverage as a signal, not a target

- Coverage shows what code no test runs. Uncovered code is untested code; look at what it is.
- High coverage doesn't mean good tests. Tests can run every line while asserting nothing useful.
- Don't write tests only to raise a percentage. Write tests for behaviour that matters, and coverage follows.

### 13. Never cheat a failing test

When a test fails, find out why. Either the code is wrong, or the test's expectation is out of date because the requirement genuinely changed.

- Never change an assertion just to match what the code currently does, unless you have confirmed the new behaviour is correct.
- Never delete, skip, or disable a test to make a change pass. If a test seems wrong, explain why to the person you're working with and let them decide.
- Never add special cases to production code that exist only to satisfy a test.

### 14. Run the tests

- Run the tests you wrote or changed and confirm they pass for the right reason. A new test should be seen to fail when the behaviour is broken.
- Run the full relevant suite before finishing, to catch anything else your change affected.
- Report honestly: which tests were run, which passed, and anything that couldn't be run.

## Example

**Poor:**

```
test "order service":
    repo = mock(OrderRepository)
    when(repo.find(1)).thenReturn(order)
    service = OrderService(repo, mock(Payments), mock(Clock))
    service.cancel(1)
    verify(repo).find(1)
    verify(repo).save(any())
    expect(service.lastCancelledId).toBe(1)
```

The name says nothing. The test checks internal calls and a private detail, not behaviour. It doesn't show what state the order was in or what "cancelled" should mean. A harmless refactor, such as loading the order differently, breaks it.

**Better:**

```
test "cancelling an unshipped order refunds the customer and marks it cancelled":
    orders = InMemoryOrderRepository()
    payments = FakePayments()
    order = orders.add(anOrder(status=Paid, total=Money(2500, "KES")))
    service = OrderService(orders, payments)

    service.cancel(order.id)

    expect(orders.get(order.id).status).toBe(Cancelled)
    expect(payments.refunds).toContain(Refund(order.id, Money(2500, "KES")))

test "cancelling a shipped order is rejected and nothing is refunded":
    orders = InMemoryOrderRepository()
    payments = FakePayments()
    order = orders.add(anOrder(status=Shipped))
    service = OrderService(orders, payments)

    result = service.cancel(order.id)

    expect(result).toBe(CannotCancelShippedOrder)
    expect(orders.get(order.id).status).toBe(Shipped)
    expect(payments.refunds).toBeEmpty()
```

Each test names a behaviour, sets up only what matters, uses simple fakes, and checks outcomes a user would care about.

## Common mistakes

- Testing private methods or internal call sequences instead of behaviour.
- Mocking everything, so the test only proves the mocks were configured.
- Mocking third-party libraries directly instead of your own adapter.
- Unit-testing database queries with a mocked database.
- Only testing the success path.
- Vague names like `test1` or `itWorks`.
- Tests that depend on each other, on run order, or on shared data.
- Using the real clock, real randomness, fixed sleeps, or real external services.
- Logic and loops inside tests.
- Changing assertions to match broken behaviour, or skipping failing tests.
- Writing tests only to increase coverage.
- Claiming tests pass without running them.

## Reference files

| File | Read when |
|---|---|
| `references/test-levels.md` | Deciding between unit, integration, and end-to-end tests, or testing APIs and data access |
| `references/test-design.md` | Deciding which cases to test, naming and structuring tests, or practising TDD |
| `references/test-doubles.md` | Using mocks, stubs, fakes, or spies, or isolating dependencies |
| `references/test-data-and-determinism.md` | Creating test data, fixing flaky or slow tests, or handling time, randomness, and concurrency |

## Sources

Kent Beck, *Test-Driven Development: By Example*; Mike Cohn, *Succeeding with Agile* (the test pyramid); Gerard Meszaros, *xUnit Test Patterns*; Steve Freeman and Nat Pryce, *Growing Object-Oriented Software, Guided by Tests*; Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns*; Michael Feathers, *Working Effectively with Legacy Code*; Martin Fowler, "Mocks Aren't Stubs" and "The Practical Test Pyramid"; Google, *Software Engineering at Google* (testing chapters); Tim Ottinger and Jeff Langr (FIRST).