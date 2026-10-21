# Load and Overload

Every service has a limit. What matters is how it behaves when that limit is reached: a well-designed service rejects excess work quickly and keeps serving what it can; a poorly designed one slows down for everyone until it collapses.

## Contents

1. How overload causes collapse
2. Rate limiting
3. Concurrency limits
4. Bounded queues
5. Load shedding
6. Backpressure
7. Prioritisation
8. Scaling
9. Expensive requests

## 1. How overload causes collapse

When requests arrive faster than they can be handled:

1. Requests wait in queues; latency rises.
2. Clients time out and retry, adding more load.
3. The service keeps working on requests whose clients have already given up, wasting capacity.
4. Memory fills with queued work; garbage collection or swapping slows everything further.
5. Throughput drops even though demand is unchanged, and the service may crash.

The aim is to stay in control: accept what can be handled in time, and reject the rest fast.

## 2. Rate limiting

Rate limiting caps how many requests a client can make in a period, protecting the service from any one client, whether a bug, an abusive user, or a sudden burst.

- **Limit by identity:** per user, per API key, or per tenant for authenticated traffic; per source address for unauthenticated traffic.
- **Use stricter limits** on sensitive and expensive endpoints: sign-in, password reset, search, exports.
- **Common algorithms:** token bucket (allows short bursts up to a capacity, refilled at a steady rate), sliding window, and fixed window. Use the project's rate-limiting library or gateway.
- **Use a shared store** (such as Redis) when there are several instances, so limits apply across all of them.
- **Respond with `429 Too Many Requests`** and `Retry-After` (see `backend-api-design`).
- **Decide what happens if the rate limiter's store is down:** usually allow traffic (fail open) for general limits, but consider denying (fail closed) for security-critical limits such as sign-in attempts.

## 3. Concurrency limits

Rate limits control arrivals per second; concurrency limits control how much work is in progress at once. Both are useful.

- Limit concurrent requests per instance to what it can actually handle well.
- Limit concurrent calls to each dependency (see bulkheads in `isolation-and-degradation.md`).
- Limit concurrent jobs per worker.

When the limit is reached, reject or queue (in a bounded queue) rather than starting more work.

## 4. Bounded queues

Every queue, buffer, and pool must have a maximum size.

- An unbounded in-memory queue turns overload into memory exhaustion and a crash.
- A very long queue means work waits so long that the result is useless by the time it's processed.
- When a queue is full, reject new work with a clear signal, so callers can back off or try elsewhere.
- For message brokers and job queues, monitor queue length and age of the oldest item, and alert when they grow.

Prefer processing newer work over very old work when old work has lost its value: a user who left the page ten minutes ago doesn't need their request processed now.

## 5. Load shedding

Load shedding deliberately rejects some requests when the service is overloaded, so the rest can be served well.

- Detect overload from signals such as concurrent requests, queue length, latency, or CPU.
- Reject excess requests early and cheaply, before doing expensive work, with `503 Service Unavailable` (and `Retry-After` if appropriate).
- Drop requests whose deadlines have already passed, rather than working on them.
- Shed the least important traffic first (see section 7).

Rejecting 10% of requests quickly is far better than serving 100% of requests too slowly to be useful.

## 6. Backpressure

In pipelines (producers feeding consumers, streams, job queues), backpressure means slowing the producer when the consumer can't keep up, rather than letting work pile up.

- Use bounded queues between stages, so a full queue makes the producer wait or receive a rejection.
- Use pull-based consumption where possible: consumers fetch work when they're ready, instead of having work pushed at them.
- For streams, use the platform's flow-control mechanisms.
- Propagate the signal upstream, so the original source slows down, not just the middle of the pipeline.

## 7. Prioritisation

When capacity is short, not all requests are equally important.

- Identify critical traffic: completing purchases, signing in, health checks, and requests from paying customers may matter more than browsing, reports, or background synchronisation.
- Shed or delay low-priority traffic first.
- Use separate queues or worker pools for different priorities, so low-priority floods can't block high-priority work.

## 8. Scaling

- **Stateless services scale horizontally:** add instances to handle more load. Keep session and other shared state in a database or cache, not in instance memory (see the Twelve-Factor App).
- **Autoscaling** helps with gradual growth, but reacts in minutes, not seconds. Rate limits and load shedding are still needed for sudden spikes.
- **Scaling the application doesn't scale the database.** More instances mean more database connections and queries. Check that connection pools and the database can handle the total.
- **Know your limits:** measure capacity with load tests (see `testing-failure.md`), so you know when to scale before users find out.

## 9. Expensive requests

Some requests cost far more than others and can overload a service on their own.

- Limit request sizes, page sizes, query complexity, and date ranges.
- Move expensive operations (exports, large reports, bulk imports) to background jobs with their own limits.
- Apply stricter rate limits to expensive endpoints.
- Cache expensive results where appropriate (see `backend-data`).