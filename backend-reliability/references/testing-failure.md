    # Testing Failure

Reliability mechanisms are code, and code that is never exercised often doesn't work. Timeouts, retries, circuit breakers, and fallbacks only run when something goes wrong, which is exactly when there's no time to discover they're broken.

## Contents

1. Testing failure paths in code
2. Testing against real failure conditions
3. Load testing
4. Failure injection
5. Reliability targets
6. Learning from incidents

## 1. Testing failure paths in code

For each remote dependency, write tests (see `testing`) that simulate:

- **Errors:** the dependency returns `500`, `503`, or a connection error.
- **Timeouts:** the dependency doesn't respond within the timeout. Use the test framework's fake timers or a fake that delays, rather than real waiting.
- **Slow responses:** the response arrives just under and just over the timeout.
- **Malformed responses:** unexpected formats or missing fields.
- **Rate limiting:** `429` with `Retry-After`.

And check that:

- Timeouts are actually applied.
- Retries happen only for retryable errors, the right number of times, with backoff.
- Non-retryable errors are not retried.
- Circuit breakers open after repeated failures and fail fast while open.
- Fallbacks produce the expected degraded result, and are recorded in metrics or logs.
- Essential dependency failures produce a clear error, not a hang.
- Idempotency holds when an operation is retried.

## 2. Testing against real failure conditions

Unit tests with fakes check the logic. Integration tests check that the real clients are configured correctly:

- Run a dependency stand-in (or a proxy that injects faults) that can delay, drop, or reject connections, and confirm the real client's timeouts and retries behave as configured.
- Stop the database or cache in an integration environment and confirm the service reports not-ready, recovers when it returns, and doesn't crash-loop.
- Send a termination signal during a request or job and confirm graceful shutdown completes the work or returns it safely.

## 3. Load testing

Load tests answer: how much traffic can the service handle, and how does it behave beyond that?

- **Test realistic scenarios:** a representative mix of requests, realistic data volumes, and realistic dependency latency.
- **Find the limit:** increase load until latency or errors become unacceptable. Know this number before production traffic finds it for you.
- **Test beyond the limit:** confirm the service sheds load and recovers, rather than collapsing.
- **Test sustained load:** some problems (memory leaks, growing queues, connection exhaustion) appear only after hours.
- **Test in an environment like production,** and never against production systems or real third-party services without explicit agreement.

Use the project's load-testing tool, and run load tests before launches and expected traffic spikes.

## 4. Failure injection

Failure injection (sometimes called chaos engineering) deliberately introduces failures in controlled conditions to confirm the system handles them: terminating instances, adding latency to a dependency, or making a dependency unavailable.

- Start in test environments. Move to production only with mature monitoring, clear procedures, limited scope, and the team's agreement.
- Have a hypothesis: "If the recommendation service is down, product pages still load within 1 second without recommendations."
- Be able to stop the experiment immediately.
- Fix what you learn.

This is an advanced practice. Suggest it where it fits; don't introduce it unilaterally.

## 5. Reliability targets

To know whether a service is reliable enough, define it in measurable terms (from Google's Site Reliability Engineering practices):

- **Service level indicator (SLI):** a measure of user experience, such as the percentage of requests that succeed, or that complete within 300 ms.
- **Service level objective (SLO):** a target for that measure over a period, such as "99.9% of checkout requests succeed each month".
- **Error budget:** the allowed failure implied by the target (0.1% of requests in the example). When the budget is being used up quickly, reliability work takes priority over new features.

Choose targets from what users actually need, not 100%. Every additional "nine" of reliability costs significantly more. Targets also tell you which dependencies and features deserve the most investment (see `backend-observability` for measuring them).

## 6. Learning from incidents

When something fails in production:

- Restore service first; investigate afterwards.
- Write a short, blameless review: what happened, how it was detected, what the impact was, why the defences didn't prevent it, and what will change.
- Turn the findings into concrete improvements: a missing timeout, an absent alert, a fallback that didn't work, a test for the failure.
- Look for the same weakness elsewhere in the system.