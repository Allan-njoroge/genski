---
name: backend-reliability
description: Principles for building backend services that keep working when the things around them fail, in any language or framework. Covers designing for failure, timeouts and deadlines, retries with backoff and jitter, circuit breakers, bulkheads and isolation, fallbacks and graceful degradation, overload protection, rate limiting, load shedding and backpressure, health checks, startup and graceful shutdown, safe deployments, and testing failure. Use this skill whenever writing code that calls a database, cache, queue, or another service or external API; adding retries, timeouts, or fallbacks; building background workers; configuring health checks, startup, or shutdown; handling high load; or investigating outages, slowness, or cascading failures, even if the task only says "call this API".
---

# Backend Reliability

Every backend depends on things that fail: networks drop packets, databases slow down, third-party APIs go offline, servers restart, traffic spikes without warning. A reliable service doesn't assume these won't happen. It expects them, limits the damage when they do, and recovers on its own.

The most dangerous failures aren't crashes. They're **slowness that spreads**: one slow dependency ties up threads and connections, requests pile up, memory fills, and a problem in one small service brings down everything that calls it. Most rules in this skill exist to stop that spread.

Read the project's `AGENTS.md` first. Use the project's HTTP clients, resilience libraries, configuration, and conventions for timeouts and retries. Where the project's conventions differ from this skill, follow the project. Apply the `error-handling` skill for classifying failures, and the `backend-data` skill for idempotency and consistency.

## Before you start

For any call to something outside the process (database, cache, queue, another service, a third-party API):

1. **What if it's slow?** How long will we wait, and what happens to the caller meanwhile?
2. **What if it's down?** Do we fail, retry, or fall back to something?
3. **Is it safe to repeat?** If we retry, could it cause a duplicate charge, email, or record?
4. **What if it's overwhelmed?** Will our retries make things worse?
5. **How important is it?** Is it essential to this request, or could the request succeed without it?

## Core rules

### 1. Assume everything outside the process can fail

Treat these as facts, not edge cases (from the "Fallacies of Distributed Computing"): the network is unreliable, latency is never zero, bandwidth is limited, and services you depend on will be slow or unavailable at some point. Design every remote call with that in mind.

### 2. Set a timeout on every remote call

A call without a timeout can wait forever, holding a thread, a connection, and memory the whole time.

- Set a **connection timeout** and a **request or read timeout** on every HTTP client, database driver, cache client, and queue client. Many libraries default to no timeout or a very long one.
- Choose timeouts from how long the operation normally takes plus a margin, not arbitrary large numbers.
- Make the **total time** for a request, including all its calls and retries, fit within the time its own caller will wait. If the client gives up after 10 seconds, retrying a downstream call for 30 seconds is wasted work.
- Pass deadlines along when calling other services, where the project supports it, so downstream work stops when nobody is waiting for the result.

### 3. Retry carefully

Retries recover from brief, transient failures. Done carelessly, they turn a small problem into an outage.

- **Retry only transient failures:** timeouts, connection errors, `429`, `502`, `503`, `504`. Never retry validation errors, authentication failures, business rule violations, or `4xx` responses other than `408` and `429` (see `error-handling`).
- **Retry only operations that are safe to repeat:** idempotent operations, or those protected with idempotency keys (see `backend-api-design`).
- **Limit attempts:** usually 2 or 3 retries.
- **Use exponential backoff with jitter:** wait longer after each failure, with randomness added, so many clients don't retry at the same moment.
- **Respect `Retry-After`** when a service provides it.
- **Retry at one layer only.** If the client, the service, and the gateway each retry three times, one failure becomes 27 requests.

For more depth, read `references/timeouts-and-retries.md`.

### 4. Stop calling what's broken

When a dependency is failing, continuing to call it wastes resources, slows every request, and prevents it from recovering.

- Use a **circuit breaker** for important remote dependencies: after repeated failures, it stops calls for a short period and fails immediately, then lets a few test calls through to check whether the dependency has recovered.
- Use the project's resilience library, rather than writing a circuit breaker from scratch.

### 5. Isolate failures

One failing dependency should not take down unrelated features.

- Give important dependencies **separate resource pools or concurrency limits** (bulkheads), so a slow payment provider can't consume every connection and thread the service has.
- Keep slow, non-essential work out of the request path. Move it to background jobs.
- Separate critical traffic (such as checkout) from less critical traffic (such as reports) where the system's scale justifies it.

For circuit breakers, bulkheads, and fallbacks, read `references/isolation-and-degradation.md`.

### 6. Degrade gracefully

Decide for each dependency whether the request can still succeed without it.

- **Essential dependencies:** if they fail, fail the request clearly and quickly.
- **Non-essential dependencies:** fall back. Show the page without recommendations, use the last known value from a cache, skip optional enrichment, or queue the work for later.
- Fallbacks must be simple and reliable themselves. A complicated fallback that also fails just adds another failure.
- Make degraded behaviour visible in logs and metrics, so it's noticed.

### 7. Protect the service from overload

A service receiving more work than it can handle should reject the excess quickly, not slow down for everyone until it collapses.

- **Rate-limit** clients, so one client can't consume all capacity.
- **Limit concurrency and queue sizes.** Unbounded queues and thread pools turn overload into memory exhaustion and unbearable latency.
- **Shed load** when overloaded: reject some requests quickly with `503` or `429`, rather than accepting everything and timing out on all of it.
- **Apply backpressure** in pipelines and workers: slow down producers when consumers fall behind.

For more depth, read `references/load-and-overload.md`.

### 8. Make background work reliable

- Jobs and message handlers must be **idempotent**, because they will sometimes run twice (see `backend-data`).
- Retry failed jobs with backoff and a limit, then move them to a **dead-letter queue** or failed-jobs table, rather than retrying forever or dropping them.
- Set a timeout on each job, so a stuck job doesn't block a worker indefinitely.
- Make long jobs resumable, saving progress so a restart doesn't start from scratch.

### 9. Start up, report health, and shut down properly

- **Validate configuration at startup** and fail immediately if something required is missing.
- Provide **health checks**: a **liveness** check (is the process working at all?) and a **readiness** check (can it serve traffic right now?). Liveness checks must not depend on external services, or a database outage will cause every instance to be restarted in a loop.
- **Shut down gracefully:** on a termination signal, stop accepting new work, finish in-flight requests and jobs within a time limit, close connections, then exit.

For more depth, read `references/lifecycle-and-deployment.md`.

### 10. Deploy safely

- Make every deployment **reversible**: you can roll back quickly to the previous version. This requires backward-compatible database migrations (see `backend-data`).
- Roll out gradually where the platform supports it, and watch error rates and latency as the new version takes traffic.
- Use **feature flags** to separate deploying code from enabling behaviour, for risky changes.

### 11. Test how the system fails

- Write tests for failure paths: dependency timeouts, errors, slow responses, and circuit breakers opening (see `testing`).
- For critical services, test under realistic load before it arrives in production.
- Where the team's maturity allows, deliberately inject failures in controlled conditions to confirm that timeouts, retries, and fallbacks behave as designed.

For more depth, read `references/testing-failure.md`.

## Example

**Poor:**

```
function getProductPage(productId):
    product = http.get(catalogService + "/products/" + productId)           # no timeout
    reviews = retryForever(() => http.get(reviewService + "/reviews?product=" + productId))
    recommendations = http.get(recommendationService + "/for/" + productId)  # no timeout
    return render(product, reviews, recommendations)
```

If the recommendation service hangs, every product page hangs with it, holding connections until the server runs out. If the review service is down, `retryForever` hammers it, prevents recovery, and never returns. A non-essential feature brings down the whole page.

**Better:**

```
catalog = HttpClient(timeout=800ms, retries=2, backoff=exponentialWithJitter, circuitBreaker=on)
reviews = HttpClient(timeout=500ms, retries=1, circuitBreaker=on)
recommendations = HttpClient(timeout=300ms, retries=0, circuitBreaker=on, concurrencyLimit=50)

function getProductPage(productId):
    product = catalog.get("/products/" + productId)        # essential: if it fails, the request fails clearly

    reviewsResult = reviews.get("/reviews", productId=productId)
        fallback: cachedReviews(productId) or EmptyReviews  # degrade: older or no reviews
    recs = recommendations.get("/for/" + productId)
        fallback: popularProducts()                          # degrade: generic suggestions

    if reviewsResult.isFallback or recs.isFallback:
        metrics.increment("product_page.degraded")

    return render(product, reviewsResult, recs)
```

Every call is bounded, retries are limited and spread out, a failing dependency is skipped quickly, and only the essential dependency can fail the page.

## Common mistakes

- Remote calls without timeouts, or with very long default timeouts.
- Retrying every error, including validation and authentication failures.
- Retrying non-idempotent operations such as payments without idempotency keys.
- Retries without backoff or jitter, or with unlimited attempts.
- Retries at several layers that multiply each other.
- Total retry time longer than the caller is willing to wait.
- One slow dependency able to consume all connections or threads.
- Treating every dependency as essential, so any failure breaks the whole request.
- Unbounded queues, thread pools, or in-memory buffers.
- Liveness checks that call the database, causing restart loops during outages.
- Exiting immediately on shutdown, dropping in-flight requests and jobs.
- Jobs that retry forever, or vanish when they fail.
- Deployments that can't be rolled back because of incompatible migrations.
- Only testing the success path.

## Reference files

| File | Read when |
|---|---|
| `references/timeouts-and-retries.md` | Setting timeouts, deadlines, or retry policies, or choosing backoff |
| `references/isolation-and-degradation.md` | Adding circuit breakers, bulkheads, fallbacks, or deciding what a request can do without |
| `references/load-and-overload.md` | Handling high traffic, rate limiting, load shedding, backpressure, queues, or capacity |
| `references/lifecycle-and-deployment.md` | Configuring health checks, startup, graceful shutdown, deployments, rollbacks, or feature flags |
| `references/testing-failure.md` | Testing timeouts, failures, and load, or defining reliability targets |

## Sources

Michael Nygard, *Release It!* (2nd edition); Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall Richard Murphy (eds.), *Site Reliability Engineering* (Google); *The Site Reliability Workbook* (Google); Peter Deutsch and others, "Fallacies of Distributed Computing"; Marc Brooker, "Exponential Backoff and Jitter" (AWS Architecture Blog); the Amazon Builders' Library ("Timeouts, retries, and backoff with jitter", "Using load shedding to avoid overload", "Avoiding insurmountable queue backlogs"); Microsoft Azure Architecture Center, cloud design patterns (Circuit Breaker, Bulkhead, Retry); Kubernetes documentation on liveness, readiness, and startup probes; The Twelve-Factor App (disposability).