# Metrics

Metrics are numbers measured over time: request rates, error counts, latencies, queue lengths. They're cheap to store and query, ideal for dashboards and alerts, and show trends that individual log lines can't.

## Contents

1. What to measure
2. Metric types
3. Measuring latency
4. Naming
5. Labels and cardinality
6. Business metrics
7. Where to instrument

## 1. What to measure

**The four golden signals** (Google SRE), for every user-facing service:

| Signal | Meaning |
|---|---|
| **Latency** | How long requests take, separately for successes and failures |
| **Traffic** | How much demand: requests per second, jobs per minute |
| **Errors** | Rate of failed requests, including wrong results where detectable |
| **Saturation** | How full the service is: pools, queues, CPU, memory |

**RED** (for request-driven services): **R**ate, **E**rrors, **D**uration, per operation.

**USE** (for resources): **U**tilisation, **S**aturation, **E**rrors, per resource such as a connection pool, worker pool, disk, or CPU.

A practical baseline for a backend service:

| Area | Metrics |
|---|---|
| Incoming requests | Count by route, method, status class; duration histogram by route |
| Each dependency | Call count by outcome; duration histogram; circuit breaker state |
| Database | Query duration; connection pool in use, idle, and waiting |
| Background jobs | Processed, failed, and retried counts by job type; duration; queue length; age of oldest waiting job |
| Reliability mechanisms | Retries, timeouts, fallbacks used, requests shed, rate limits hit |
| Runtime | CPU, memory, garbage collection, event loop or thread pool saturation (usually automatic) |
| Business | Key outcomes (see section 6) |

Many frameworks and libraries provide much of this automatically. Enable and use that before adding custom metrics.

## 2. Metric types

| Type | Use for | Example |
|---|---|---|
| **Counter** | Things that only increase; queried as a rate | Requests handled, errors, payments succeeded |
| **Gauge** | Values that go up and down | Queue length, connections in use, items in cache |
| **Histogram** | Distributions, especially latency and sizes | Request duration, payload size |
| **Summary** | Precomputed percentiles on one instance (some systems) | Prefer histograms, which can be combined across instances |

Common mistakes: using a gauge for something that should be a counter (losing increments between readings), or a counter for something that goes down.

## 3. Measuring latency

- **Use histograms,** and look at percentiles: p50 (typical), p95 and p99 (the slow experience many users still have).
- **Never rely on averages alone.** An average of 100 ms can hide 5% of requests taking 3 seconds.
- **Separate successes from failures.** Fast failures can make latency look better while the service is broken.
- **Choose histogram buckets** that cover the expected range with detail around your targets (for example, around a 300 ms objective), or use the library's defaults if they fit.
- **Measure at the right point:** at the server boundary for the service's own latency, and around each dependency call to see where time goes.

## 4. Naming

Follow the project's conventions and the metrics system's guidelines. Common practice (Prometheus and OpenTelemetry styles):

- Lowercase, with a consistent separator, and a prefix for the domain: `http_server_request_duration_seconds`, `orders_placed_total`, `job_queue_length`.
- **Units in the name,** using base units: `_seconds`, `_bytes`, not milliseconds or kilobytes.
- **Counters end in `_total`** in Prometheus-style systems.
- **Name the thing measured,** not the label values: `payments_total{outcome="failed"}`, not `failed_payments_total` alongside `succeeded_payments_total`.
- Use OpenTelemetry semantic convention names for standard measurements (HTTP, database, messaging) where the project uses OpenTelemetry.

## 5. Labels and cardinality

Labels (also called attributes or tags) split a metric into series: `http_requests_total{route="/orders", method="GET", status="200"}`. Each unique combination of label values is a separate series that the metrics system must store.

**Cardinality** is the number of unique series. High cardinality makes metrics slow and expensive, and can bring down the metrics system.

| Good labels (bounded) | Dangerous labels (unbounded) |
|---|---|
| Route template: `/orders/{id}` | Raw path: `/orders/ord_9fK2` |
| HTTP method | User ID, order ID, session ID |
| Status code or class: `2xx`, `5xx` | Email address, IP address |
| Dependency name | Full URL or query string |
| Job type | Error message text |
| Outcome: `succeeded`, `failed`, `declined` | Timestamps |
| Region, environment, version | Free-form input |

Rules:

- Every label must have a small, known set of possible values.
- Use route templates, never raw paths.
- Map error details to a small set of categories or codes.
- Put high-cardinality details (specific IDs) in logs and traces, which are designed for them, not in metrics.

## 6. Business metrics

Technical metrics can look healthy while the business is broken: a payment provider might return success while silently failing, or a bug might stop orders being created without causing errors.

Measure the outcomes that matter to the business:

- Sign-ups, sign-ins, orders placed, payments succeeded and failed by reason, messages delivered, subscriptions renewed.
- Funnel steps for critical journeys: started checkout, submitted payment, completed order.

A sudden drop in a business metric is often the earliest and most reliable sign of a problem. Alert on unexpected drops in critical flows.

## 7. Where to instrument

- **Middleware** for request metrics, applied once to all routes.
- **Client wrappers** for dependency metrics, so every call is measured consistently (usually provided by instrumentation libraries).
- **Job framework hooks** for background job metrics.
- **Business logic** for business metrics, at the point the outcome is certain (after the transaction commits, for example).

Avoid scattering ad hoc metric calls through the code. Define metrics in one place per module and reuse them.