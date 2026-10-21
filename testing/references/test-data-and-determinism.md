# Test Data and Determinism

Reliable tests produce the same result every time. This file covers creating test data, and controlling the sources of non-determinism that make tests flaky.

## Contents

1. Test data builders
2. Isolating data between tests
3. Time
4. Randomness and generated values
5. Waiting and asynchronous code
6. Concurrency
7. External systems
8. Fixing a flaky test
9. Keeping the suite fast

## 1. Test data builders

Create test data with builders or factory functions that fill in valid defaults, so each test specifies only the values it cares about.

```
anOrder(overrides):
    return Order(
        id = newTestId(),
        customer = aCustomer(),
        status = Paid,
        total = Money(1000, "KES"),
        createdAt = FIXED_TEST_TIME,
        ...overrides
    )

test "...":
    order = anOrder(status=Shipped)     # only the relevant value is visible
```

Guidelines:

- Defaults must produce valid objects. A test should only see invalid data when it asks for it.
- Make the values that matter to the test explicit in the test, even if they equal the default.
- Use clearly fake data: `test-user@example.com`, not real-looking personal details. Never use real customer data or production secrets.
- Keep builders alongside the tests, following the project's conventions.

## 2. Isolating data between tests

Tests that share data interfere with each other, producing failures that depend on run order.

- Each test creates its own data and doesn't rely on data from other tests.
- For database tests, isolate by one of: running each test in a transaction that is rolled back; clearing tables between tests; or using unique identifiers so tests never collide.
- Avoid seed data that tests modify. Read-only reference data, such as a list of currencies, is fine.
- Tests must pass when run alone, in any order, and in parallel.

## 3. Time

Code that reads the current time produces tests that pass today and fail tomorrow, or fail near midnight, at month end, or in a different time zone.

- Pass a clock (or the current time) into code instead of reading it directly, so tests can supply a fixed time.
- Or use the framework's fake timers to control time.
- Use fixed, explicit times in tests: `2026-03-15T10:00:00Z`.
- Test time-sensitive boundaries explicitly: just before and after expiry, month and year ends, leap days, and time zone changes where relevant.
- Be explicit about time zones. Store and compare in UTC unless the behaviour is specifically about local time.

## 4. Randomness and generated values

- Inject the random source, or use a fixed seed in tests.
- For generated IDs, either inject the generator or assert on the presence and format of an ID rather than an exact value.
- When a test uses random inputs (as in property-based testing), the library should report the seed or failing input so failures can be reproduced.

## 5. Waiting and asynchronous code

Fixed sleeps are the most common cause of flaky and slow tests.

```
Poor:   startJob(); sleep(2 seconds); expect(job.isDone())
Better: startJob(); waitUntil(() => job.isDone(), timeout=5 seconds)
```

- Await asynchronous operations directly where possible.
- When polling is unavoidable, wait for the specific condition, with a timeout, and fail with a clear message if it times out.
- In unit tests, avoid real asynchrony where you can, by making the asynchronous boundary replaceable.

## 6. Concurrency

- Tests running in parallel must not share mutable state: global variables, singletons, files, ports, or database rows.
- Use unique names for resources each test creates, such as temporary directories and queue names.
- To test concurrent behaviour (such as two requests racing to book the same slot), make the race explicit and repeatable, for example by controlling when each step proceeds, rather than hoping timing triggers it.

## 7. External systems

- Automated tests must not call real third-party services. They're slow, can fail for reasons unrelated to your code, may cost money, and may have side effects.
- Use fakes in unit tests and sandboxes or local stand-ins in integration tests (see `test-doubles.md`).
- Infrastructure you control (your database, your cache, your queue) should run as real local instances, typically in containers, rather than shared environments other people use.

## 8. Fixing a flaky test

1. Don't ignore it or simply re-run until it passes. A flaky test may be revealing a real race condition.
2. Reproduce it: run it repeatedly, alone and with the full suite, in parallel and in sequence.
3. Look for the usual causes: time, randomness, shared data, test order, fixed sleeps, real network calls, unordered results.
4. Fix the cause. If it can't be fixed immediately, quarantine it visibly with a tracked issue, and tell the person you're working with. Never silently delete or skip it.

## 9. Keeping the suite fast

A slow suite gets run less, so problems are found later.

- Keep most tests at the unit level.
- Share expensive setup, such as starting a database container, across a test run, while keeping each test's data isolated.
- Run tests in parallel where the framework supports it, which requires the isolation described above.
- Find and fix the slowest tests periodically. They often contain sleeps or unnecessary real dependencies.