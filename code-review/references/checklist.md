# Review Checklist

Detailed questions for each area of a review. Not every question applies to every change. Use the ones relevant to what the change touches, and spend the most time on the riskiest parts.

## Contents

1. Intent and scope
2. Correctness
3. Error handling
4. Security
5. Design and architecture
6. Data and persistence
7. Tests
8. Readability
9. Performance
10. Operations
11. Interfaces and compatibility
12. Documentation

## 1. Intent and scope

- [ ] The change's purpose is clear from its description.
- [ ] The change does what it claims, and nothing unrelated.
- [ ] The change is small enough to review properly, or is split sensibly.
- [ ] The approach is reasonable compared with simpler alternatives.

## 2. Correctness

- [ ] Logic is correct for typical inputs.
- [ ] Boundaries are handled: zero, one, empty, maximum, first and last items, just above and below limits.
- [ ] Absent values (null, missing, not found) are handled where they can occur.
- [ ] Conditions are correct: no inverted checks, off-by-one errors, or wrong operators.
- [ ] Loops terminate, and iterate over the right range.
- [ ] Types and units are consistent (money in the smallest unit, times in UTC, and so on).
- [ ] Concurrent use is safe: no race conditions on shared state, no lost updates, no double processing.
- [ ] Operations that must be atomic are in a transaction or equivalent.
- [ ] Retried or repeated operations are safe (idempotent).
- [ ] Time-dependent logic handles time zones, month ends, and daylight-saving changes where relevant.

## 3. Error handling

- [ ] No empty catch blocks or silently ignored errors.
- [ ] Failures aren't hidden behind default values.
- [ ] Errors are caught only where something useful can be done.
- [ ] Original causes are preserved when errors are wrapped.
- [ ] Specific error types are used, consistent with the project.
- [ ] User-facing errors are helpful and leak no internal details.
- [ ] Resources (files, connections, locks) are released on every path.
- [ ] Asynchronous errors are handled; nothing is fire-and-forget without error handling.

## 4. Security

- [ ] All external input is validated on the server, with size limits.
- [ ] Queries use parameters; no string-built SQL, commands, or HTML.
- [ ] Every new operation checks authentication, and authorisation for the specific object.
- [ ] Identity and roles come from the verified session, never the request.
- [ ] Responses include only the necessary fields.
- [ ] Only permitted fields can be written (no mass assignment).
- [ ] No secrets in code, configuration, tests, or logs.
- [ ] No sensitive data in logs, errors, or analytics.
- [ ] File paths, uploads, and outgoing URLs from users are handled safely.
- [ ] New dependencies are necessary, legitimate, maintained, and pinned.
- [ ] No security control was disabled or loosened.
- [ ] Rate limits exist on sensitive or expensive endpoints.

## 5. Design and architecture

- [ ] Code lives in the right module and layer.
- [ ] Module boundaries are respected; other modules are used through their public interfaces.
- [ ] No module reads or writes another module's data directly.
- [ ] Dependencies point the right way (business logic doesn't depend on frameworks or infrastructure).
- [ ] No unnecessary abstractions, interfaces with a single implementation, or speculative generality.
- [ ] No premature extraction of shared code.
- [ ] New code follows existing patterns rather than introducing competing ones.
- [ ] Invalid states are prevented by types where practical.
- [ ] Architecture checks pass, if the project has them.
- [ ] Nothing contradicts a recorded architecture decision.

## 6. Data and persistence

- [ ] Schema changes are in migrations, versioned and reviewable.
- [ ] Migrations work with existing data, and with the old code still running during deployment.
- [ ] Destructive changes (dropping columns or tables) are separated from the code changes that stop using them.
- [ ] New queries are supported by appropriate indexes.
- [ ] No N+1 query patterns (a query per item in a loop).
- [ ] List queries are paginated or bounded.
- [ ] Constraints (unique, not null, foreign keys) back up business rules.

## 7. Tests

- [ ] New and changed behaviour has tests.
- [ ] Failure paths and edge cases are tested, not only the success path.
- [ ] Tests check behaviour through public interfaces, not implementation details.
- [ ] Tests would fail if the code were broken.
- [ ] Tests are at the right level (unit, integration, end-to-end).
- [ ] Tests are deterministic: no real clock, randomness, sleeps, or external services.
- [ ] Test names describe the behaviour.
- [ ] Bug fixes include a regression test.
- [ ] No tests were deleted, skipped, or weakened without good reason.
- [ ] The test suite passes (verified by running it, where possible).

## 8. Readability

- [ ] Names are precise and consistent with the codebase.
- [ ] Functions do one thing at one level of detail.
- [ ] Control flow is flat and easy to follow.
- [ ] No magic numbers or strings.
- [ ] Comments explain *why* where the reason isn't obvious, and none are out of date.
- [ ] No commented-out code, debug output, or unused code.
- [ ] Consistent with the project's conventions.

## 9. Performance

Only where it matters: frequently run code, large data, or user-facing latency.

- [ ] No obviously expensive work inside loops (queries, network calls, repeated parsing).
- [ ] Large datasets are streamed or paginated rather than loaded whole.
- [ ] Caching, if added, has a clear invalidation strategy.
- [ ] Claims of performance improvement are supported by measurement.

## 10. Operations

- [ ] Important events and failures are logged, with context, without sensitive data.
- [ ] New configuration is documented, validated at startup, and has safe defaults.
- [ ] External calls have timeouts.
- [ ] The change can be deployed safely, and rolled back if needed.
- [ ] Feature flags, if used, have a plan for removal.

## 11. Interfaces and compatibility

- [ ] Public API changes are backward compatible, or versioned.
- [ ] Changes to events, messages, or shared data formats won't break consumers.
- [ ] Error responses follow the project's standard format.
- [ ] Changes to exported functions or types are reflected in all callers.

## 12. Documentation

- [ ] Public interfaces are documented.
- [ ] README, API documentation, or `AGENTS.md` are updated if behaviour, setup, or commands changed.
- [ ] Significant decisions are recorded where the project keeps decision records.