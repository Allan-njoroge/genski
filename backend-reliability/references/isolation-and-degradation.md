# Isolation and Graceful Degradation

These patterns keep a failure in one place from spreading, and let a service keep doing useful work when some of its dependencies can't.

## Contents

1. Cascading failures
2. Circuit breakers
3. Bulkheads
4. Classifying dependencies
5. Fallback strategies
6. Asynchronous work as isolation
7. Feature switches for degradation

## 1. Cascading failures

A typical cascade:

1. A dependency becomes slow, not down, just slow.
2. Callers wait for it, holding threads, connections, and memory.
3. Requests pile up; the calling service runs out of resources.
4. The calling service becomes slow or unresponsive to its own callers.
5. The problem spreads upstream until large parts of the system fail.

The defences: timeouts (bound the wait), circuit breakers (stop waiting on something known to be broken), bulkheads (limit how much of the service one dependency can consume), and fallbacks (keep serving without it).

## 2. Circuit breakers

A circuit breaker wraps calls to a dependency and tracks failures (from Michael Nygard, *Release It!*).

```
       failures exceed threshold
CLOSED ─────────────────────────► OPEN
  ▲                                 │
  │ test calls succeed              │ wait period elapses
  │                                 ▼
  └────────────────────────── HALF-OPEN
        (test calls fail → back to OPEN)
```

- **Closed:** calls pass through normally. Failures are counted.
- **Open:** calls fail immediately without contacting the dependency. Callers use a fallback or report the failure at once.
- **Half-open:** after a wait, a few test calls are allowed. Success closes the circuit; failure opens it again.

Benefits: callers fail fast instead of waiting for timeouts; the struggling dependency gets breathing room to recover; and failures are visible as a clear state change.

Guidelines:

- Use one circuit breaker per dependency (or per distinct endpoint, if they fail independently).
- Count only failures that indicate the dependency is unhealthy: timeouts, connection errors, and `5xx` responses. Don't count `4xx` responses caused by bad requests.
- Base thresholds on failure rate over a window with a minimum number of calls, not a single failure.
- Log and record metrics on every state change.
- Use a well-tested library rather than implementing the state machine yourself.

## 3. Bulkheads

On a ship, bulkheads divide the hull into compartments so one leak doesn't sink the whole vessel. In software, they divide resources so one failing dependency can't consume everything.

Ways to apply them:

- **Separate connection pools** per downstream service.
- **Concurrency limits** per dependency: at most N calls in flight to the recommendation service at once. Extra calls fail immediately or use the fallback.
- **Separate worker pools or queues** for different kinds of background jobs, so a flood of slow report jobs can't delay password-reset emails.
- **Separate deployments** for critical and non-critical workloads, at larger scale.

Size each compartment so the service still has capacity for everything else when one compartment is completely full.

## 4. Classifying dependencies

For each dependency of a request, decide in advance:

| Category | When it fails | Example |
|---|---|---|
| **Essential** | The request fails, quickly and clearly | The database holding the order being placed; the payment provider during payment |
| **Degradable** | The request succeeds with reduced functionality | Recommendations, reviews, personalisation, analytics |
| **Deferrable** | The work is queued and done later | Sending a confirmation email, updating a search index, notifying another system |

Most dependencies are less essential than they first appear. Making them degradable or deferrable is one of the most effective reliability improvements available.

## 5. Fallback strategies

| Strategy | Example | Watch out for |
|---|---|---|
| **Cached or last known value** | Show yesterday's exchange rate with its timestamp | Must be acceptable to show stale data; make it visible where it matters |
| **Default or generic value** | Popular products instead of personalised recommendations | Must be clearly safe |
| **Omit the feature** | Hide the reviews section | The interface must handle its absence |
| **Queue for later** | Accept the request, send the email when the service recovers | Needs a reliable queue and idempotent processing |
| **Alternative provider** | Send SMS through a secondary provider | Adds complexity; test it regularly, or it won't work when needed |
| **Fail clearly** | "Payments are temporarily unavailable. Your basket is saved." | The right choice for essential dependencies |

Rules for fallbacks:

- **Keep them simple.** A fallback should be much more reliable than what it replaces.
- **Never fall back silently in ways that mislead,** such as showing a zero balance because the balance service is down. Showing "balance unavailable" is honest; showing zero is wrong.
- **Never fall back to less security.** If the authorisation service is down, deny access; don't allow it.
- **Record when fallbacks are used,** in metrics and logs, so degradation is noticed.
- **Test fallbacks.** An untested fallback often fails exactly when it's needed.

## 6. Asynchronous work as isolation

Moving work out of the request path is one of the strongest isolation techniques:

- The user's request completes as soon as the essential work is done.
- Slow or failing downstream systems affect only the queue, not users.
- Work is retried automatically when the dependency recovers.

Use it for anything the user doesn't need to wait for: emails, notifications, webhooks, search indexing, analytics, report generation, and calls to slow third-party services. Combine it with the outbox pattern (see `backend-data`) so queued work is never lost.

## 7. Feature switches for degradation

For larger systems, configuration switches (sometimes called kill switches) let operators turn off non-essential features during an incident, freeing capacity for essential ones. They should:

- Take effect without a deployment.
- Default to "on" (feature available) and be clearly named.
- Be tested, so turning one off actually works.