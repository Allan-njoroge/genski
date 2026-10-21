# Test Levels

Different kinds of tests give different kinds of confidence at different costs. This file explains each level, what belongs at it, and how to balance them.

## Contents

1. The balance of levels
2. Unit tests
3. Integration tests
4. API and contract tests
5. End-to-end tests
6. Choosing the level for a behaviour

## 1. The balance of levels

The test pyramid (Mike Cohn) describes a healthy balance:

```
        /\         End-to-end: few, slow, broad
       /  \
      /----\       Integration: a solid layer
     /      \
    /--------\     Unit: many, fast, focused
```

The reasoning: lower-level tests are faster, more reliable, and pinpoint failures precisely. Higher-level tests catch problems in how parts fit together, but are slower, more fragile, and harder to diagnose.

Some teams, particularly in frontend and API-heavy work, put more weight on integration tests (sometimes drawn as a "testing trophy"). That's fine. The principle that holds everywhere: **test each behaviour at the lowest level that gives real confidence in it**, and keep the slowest tests for what only they can check.

## 2. Unit tests

**What they test:** one unit of behaviour, in isolation from slow and external dependencies. A "unit" is a behaviour, not necessarily a single function or class; it may involve several in-process objects working together.

**Good candidates:**

- Business rules and calculations
- Validation and parsing
- State transitions
- Edge cases and boundary values
- Pure functions in the functional core

**Characteristics:** run in milliseconds, no network, no database, no file system, no real clock.

**Watch out for:** isolating so aggressively that every collaborator is mocked. Then the test checks wiring, not behaviour. Use real in-process objects where practical.

## 3. Integration tests

**What they test:** your code working together with real infrastructure or with other modules.

**Good candidates:**

- Repositories and queries against a real database of the same type as production
- Database migrations
- Message publishing and consuming against a real broker
- Adapters for external services, against a sandbox or a faithful local stand-in
- Several modules working together through their public interfaces

**How to do them well:**

- Use the same database engine as production, typically in a container started for the test run. An in-memory substitute of a different engine can hide real differences in queries, types, and constraints.
- Isolate each test's data: wrap each test in a rolled-back transaction, truncate tables between tests, or use unique identifiers.
- Keep them focused. An integration test of a repository checks the repository, not the whole application.

## 4. API and contract tests

**API tests** exercise the system through its HTTP (or other) interface, with real routing, middleware, validation, and serialisation, usually against a test database. They are often the most valuable backend tests, because they check what clients actually experience.

Cover for each endpoint:

- Success responses: status, body shape, headers that matter.
- Validation errors and their format.
- Authentication and authorisation: missing credentials, wrong user, insufficient permission.
- Not-found and conflict cases.
- That internal details don't leak in error responses.

**Contract tests** check that two separately deployed systems agree on the shape of their communication. The consumer defines what it expects; the provider verifies it meets those expectations. Use them when services owned by different teams or repositories talk to each other, to catch breaking changes before deployment.

## 5. End-to-end tests

**What they test:** the complete system as a user experiences it: user interface, backend, database, and integrations together.

**Use them for:** a small number of critical journeys whose failure would be serious, such as signing up, signing in, paying, and the core action the product exists for.

**Keep them few, because they are:**

- Slow to run.
- Prone to flakiness from timing, networks, and environment.
- Hard to diagnose: a failure says something is broken, not where.

**Don't use them** to test edge cases, validation rules, or business logic variations. Those belong in unit and integration tests.

## 6. Choosing the level for a behaviour

| Behaviour | Level |
|---|---|
| Discount calculation rules | Unit |
| Parsing and validating a request body | Unit |
| A state machine's allowed transitions | Unit |
| A repository query returns the right rows | Integration (real database) |
| A migration runs and preserves data | Integration (real database) |
| An endpoint returns 403 for another user's order | API test |
| Error responses follow the standard format | API test |
| A service and its consumer agree on message shape | Contract test |
| A new user can sign up and complete a first purchase | End-to-end |

When unsure, ask: what is the cheapest test that would fail if this behaviour broke?