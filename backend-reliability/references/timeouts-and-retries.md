# Timeouts and Retries

Timeouts stop a service from waiting forever. Retries recover from brief failures. Together, configured well, they absorb most everyday failures. Configured badly, they cause outages.

## Contents

1. Kinds of timeouts
2. Choosing timeout values
3. Deadlines and time budgets
4. What to retry
5. Backoff and jitter
6. Retry amplification
7. Retry budgets
8. Retries and idempotency
9. Configuration summary

## 1. Kinds of timeouts

Set each that applies, on every client:

| Timeout | Limits |
|---|---|
| **Connection timeout** | How long to wait to establish a connection. Usually short (often around a second or less within a data centre). |
| **Request or read timeout** | How long to wait for a response once connected. |
| **Total or overall timeout** | The whole operation, including retries. |
| **Pool acquisition timeout** | How long to wait for a free connection from a pool. |
| **Query or statement timeout** | How long the database may spend on a single query. |
| **Idle timeout** | How long an unused connection stays open. |
| **Server request timeout** | How long the service itself works on an incoming request. |
| **Job timeout** | How long a background job may run. |

Check library defaults. Many HTTP clients and database drivers have no timeout, or one measured in minutes.

## 2. Choosing timeout values

- **Base them on observed latency.** Look at how long the call normally takes at a high percentile (for example, the 99th percentile), and add a margin. A call that normally takes 50 ms with a 99th percentile of 200 ms might get a 500 ms timeout.
- **Different operations need different timeouts.** A quick lookup and a report generation shouldn't share one value.
- **Shorter is usually better** for calls in the request path: failing fast frees resources and allows a fallback.
- **Make them configurable,** so they can be tuned without code changes.
- **Revisit them** when latency patterns change.

## 3. Deadlines and time budgets

A request has a time budget set by whoever is waiting for it. Every call made while serving it must fit inside that budget.

```
Client waits 5 s
└── Service A (budget 5 s)
    ├── Database query: timeout 1 s
    └── Service B call: timeout 2 s, up to 1 retry
        → worst case: 1 s + 2 s + backoff + 2 s ≈ 5.5 s  ✗ exceeds the client's 5 s
```

Guidelines:

- Add up the worst case: each call's timeout multiplied by its attempts, plus backoff delays. It must be less than the caller's timeout.
- **Propagate deadlines** where the project supports it: pass the remaining time to downstream services (for example, in a header or through the RPC framework's deadline mechanism), so they stop working when the caller has already given up.
- Check the remaining time before retrying. If there isn't enough time for another attempt to succeed, don't retry.

## 4. What to retry

| Retry | Don't retry |
|---|---|
| Connection refused or reset | `400 Bad Request`, `422` validation errors |
| Timeouts (for idempotent operations) | `401` and `403` authentication and permission errors |
| `408 Request Timeout` | `404 Not Found` (usually) |
| `429 Too Many Requests` (after `Retry-After`) | `409 Conflict` (unless the conflict is known to be temporary) |
| `502`, `503`, `504` | Business rule violations |
| Database deadlocks and serialisation failures (retry the whole transaction) | Bugs and unexpected errors |
| Temporary DNS failures | Errors the provider documents as permanent |

`500 Internal Server Error` is ambiguous: it may be a bug that will fail again. Retry it at most once, if at all, and only for idempotent operations.

## 5. Backoff and jitter

Retrying immediately, or at fixed intervals, causes many clients to retry at the same moment, creating spikes that keep a recovering service overwhelmed.

**Exponential backoff** increases the wait after each failure:

```
attempt 1 fails → wait ~100 ms
attempt 2 fails → wait ~200 ms
attempt 3 fails → wait ~400 ms
(capped at a maximum, such as a few seconds)
```

**Jitter** adds randomness, spreading retries out. A widely used approach ("full jitter"):

```
delay = random(0, min(maxDelay, baseDelay × 2^attempt))
```

Always use jitter with backoff. Most resilience libraries support both.

When a response includes `Retry-After`, wait at least that long.

## 6. Retry amplification

Retries at several layers multiply:

```
Client: 3 attempts
└── Gateway: 3 attempts each
    └── Service A: 3 attempts each
        → one failing call to Service B becomes up to 27 calls
```

When a service is already struggling, amplified retries make recovery impossible.

- **Retry at one layer**, usually the one closest to the failing dependency, or the one with the most context about whether retrying is safe.
- Other layers should not retry the same failure, or should retry only when the lower layer clearly hasn't.
- Signal "don't retry" to callers when a request has already been retried and failed (for example, by returning a non-retryable status).

## 7. Retry budgets

A retry budget limits retries to a fraction of total traffic (for example, retries may add at most 10% more requests). When a dependency is broadly failing, most requests then fail fast instead of tripling the load.

Circuit breakers (see `isolation-and-degradation.md`) achieve a similar effect. Use whichever the project's tools support.

## 8. Retries and idempotency

A timeout doesn't mean the operation failed. It may have succeeded, with only the response lost. Retrying a non-idempotent operation after a timeout can duplicate it.

- **Safe to retry:** reads, and writes designed to be idempotent (set a value, upsert by key, delete).
- **Needs protection:** creating records, charging payments, sending messages. Use idempotency keys (see `backend-api-design`), so the server recognises and ignores repeats.
- **Can't be made safe:** don't retry automatically. Fail, record the uncertain state, and reconcile later (see `error-handling`).

## 9. Configuration summary

A reasonable starting point for a call in the request path, to adjust from real measurements:

| Setting | Starting point |
|---|---|
| Connection timeout | About 1 s or less within a data centre; somewhat longer for external APIs |
| Request timeout | A margin above the observed 99th-percentile latency |
| Retries | 1–3, only for transient errors on safe operations |
| Backoff | Exponential, starting around 100 ms, capped at a few seconds |
| Jitter | Always |
| Total time | Less than the caller's own timeout |

Follow the project's existing values and shared client configuration where they exist.