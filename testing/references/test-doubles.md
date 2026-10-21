# Test Doubles

A test double stands in for a real dependency during a test. Used well, doubles make tests fast and deterministic. Overused, they produce tests that pass while the real system is broken.

## Contents

1. Kinds of test doubles
2. When to use a double
3. Prefer fakes and real objects
4. Don't mock what you don't own
5. State verification and interaction verification
6. Signs of mocking problems

## 1. Kinds of test doubles

These terms come from Gerard Meszaros, *xUnit Test Patterns*. Many tools call everything a "mock", but the distinctions help in choosing well.

| Kind | What it does | Example |
|---|---|---|
| **Dummy** | Fills a parameter; never actually used | A placeholder logger passed to a constructor |
| **Stub** | Returns preset answers | An exchange-rate source that always returns 130 |
| **Fake** | A simple working implementation | An in-memory repository backed by a map |
| **Spy** | Records how it was used, for checking afterwards | An email sender that stores sent messages in a list |
| **Mock** | Pre-programmed with expected calls; fails if they don't happen | A mock that expects `charge(500)` exactly once |

## 2. When to use a double

Replace a dependency when it is:

- **Slow:** network calls, large computations.
- **External:** third-party APIs, payment providers, email and SMS services.
- **Non-deterministic:** the current time, random numbers, generated IDs.
- **Has side effects you can't allow in tests:** sending real emails, charging cards.
- **Hard to put into a specific state:** simulating a provider outage or a rare error.

Keep it real when it is:

- Your own in-process code: domain objects, calculations, small services.
- Value objects and simple data structures.
- The thing you're actually trying to test.

## 3. Prefer fakes and real objects

Fakes behave like the real thing for the purposes of a test, so tests using them check real outcomes rather than scripted calls.

```
Mock-heavy (fragile):
    repo = mock(UserRepository)
    when(repo.findByEmail("a@x.com")).thenReturn(null)
    service.register("a@x.com", "pw")
    verify(repo).save(argThat(u => u.email == "a@x.com"))

Fake (robust):
    repo = InMemoryUserRepository()
    service.register("a@x.com", "pw")
    expect(repo.findByEmail("a@x.com")).toBeDefined()
```

The mock version breaks if the service starts using `existsByEmail` instead of `findByEmail`, even though behaviour is unchanged. The fake version only cares about the outcome.

Guidelines for fakes:

- Keep them simple and shared across tests, so each test doesn't reinvent them.
- Make them faithful in the ways that matter, such as enforcing uniqueness if the real one does.
- Check that a fake and the real implementation behave alike, for example by running the same set of contract tests against both.

## 4. Don't mock what you don't own

Avoid mocking third-party libraries and SDKs directly. Their behaviour is complex, they change between versions, and a mock reflects only your assumptions about them.

Instead:

1. Wrap the library in a small adapter you own, with an interface in your own terms: `PaymentGateway.charge(order)`, not the provider's full SDK.
2. In unit tests, replace your adapter with a fake or stub.
3. In integration tests, test your adapter against the real library: the provider's sandbox, or a faithful local stand-in.

This keeps unit tests simple and ensures the real integration is tested somewhere.

## 5. State verification and interaction verification

**State verification** checks the result: what was returned, what the data looks like afterwards. Prefer it.

**Interaction verification** checks that a call happened. Use it only when the call *is* the observable behaviour, typically an outgoing effect to an external system:

- An email or notification was sent.
- A payment was requested.
- An event was published.

Even then, a spy or fake that records what happened (`emails.sent`) usually reads better than strict mock expectations. Avoid verifying calls to your own internal collaborators, such as "the service called the repository", which ties tests to implementation.

## 6. Signs of mocking problems

- **A test mocks more than two or three things.** The code under test probably has too many dependencies; consider redesigning it.
- **Mocks return mocks.** The code reaches through objects (see `software-design`, Law of Demeter).
- **Tests break on harmless refactors.** They're verifying interactions instead of outcomes.
- **The test re-implements the code.** If the mock setup mirrors the function line by line, it tests nothing.
- **Everything passes, but production fails.** The doubles don't match reality. Add integration tests at the boundaries.