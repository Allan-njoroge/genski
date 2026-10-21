---
name: backend-observability
description: Principles for making backend systems explain their own behaviour in production, with any language, framework, or monitoring tools. Covers structured logging and log levels, what to log and what never to log, correlation and request IDs, metrics (RED, USE, and the golden signals), labels and cardinality, latency histograms, business metrics, distributed tracing with OpenTelemetry, context propagation, instrumenting dependencies and background jobs, alerting on symptoms and SLOs, dashboards, runbooks, sampling, and cost. Use this skill whenever adding or changing logging, metrics, tracing, monitoring, alerts, or dashboards; building a new service, endpoint, background job, or integration that needs to be observable; or investigating a production problem, even if the task only says "add some logs".
---

# Backend Observability

When something goes wrong in production, you can't attach a debugger or add print statements. You can only see what the system was already telling you. Observability is the ability to understand what a system is doing, and why, from the signals it emits: **logs** (what happened), **metrics** (how much and how often), and **traces** (where time went across services).

A well-instrumented system lets you answer new questions about production, ones you didn't anticipate, quickly and without deploying new code. A poorly instrumented one leaves you guessing during an outage, with either no information or a flood of noise.

Read the project's `AGENTS.md` first. Use the project's logger, metrics library, tracing setup, naming conventions, and monitoring platform. Where the project's conventions differ from this skill, follow the project. Apply the `error-handling` skill for logging errors, and the `security` skill for keeping sensitive data out of telemetry.

## Before you start

1. **What will someone need to know when this breaks?** Picture debugging this code at 2 a.m. from production signals alone.
2. **How will we know it's healthy?** Which measures show it's working: request rates, error rates, latency, queue lengths, business outcomes?
3. **What does the project already provide?** Many frameworks and libraries instrument requests, database calls, and HTTP clients automatically. Don't duplicate it.
4. **What must never appear?** Secrets, tokens, passwords, payment details, unnecessary personal data.

## Core rules

### 1. Log structured events, not prose

- Use the project's structured logger, emitting machine-readable records (usually JSON) with consistent fields, not free-form strings built by concatenation.
- Log a stable **event name** plus **fields**: `log.info("order.placed", orderId=..., customerId=..., totalAmount=...)`, not `log.info("Order " + id + " was placed by " + customer)`.
- Use consistent field names across the codebase (`orderId` everywhere, never `order_id` in one place and `oid` in another).
- Never use `print` or console output in place of the logger.

### 2. Use log levels honestly

| Level | Use for |
|---|---|
| **Error** | A failure that needs attention: an unexpected exception, a permanent dependency failure, data that couldn't be processed |
| **Warning** | Something unusual the system handled: a retry that succeeded, a fallback used, a deprecated feature called |
| **Info** | Significant normal events: a request completed, an order placed, a job finished, a service started |
| **Debug** | Detail for investigation, usually disabled in production |

- Expected outcomes (invalid input, a user entering a wrong password, a record not found) are not errors. Logging them as errors buries real problems and triggers false alerts.
- Log an error **once**, where it is finally handled, with full context (see `error-handling`).

### 3. Make every request traceable

- Give every incoming request a **correlation ID** (or use the trace ID), accept it from upstream callers when present, include it in every log line, return it in the response, and pass it to every downstream call and queued job.
- Make sure background jobs and message consumers carry the ID of the request or event that created them.
- With this, one ID finds every log line, across every service, for one user's request.

### 4. Log the right things

**Log:**

- Each request's outcome: method, route, status, duration, correlation ID (often done automatically by middleware).
- Significant business events: orders placed, payments completed, accounts created.
- Decisions that explain behaviour: a fallback used, a circuit breaker opening, a retry, a feature flag evaluated to a non-default value.
- Failures, with context and cause.
- Security events: sign-ins, failed sign-ins, permission denials, administrative actions (see `security`).
- Startup configuration (without secrets), and shutdown.

**Don't log:**

- Secrets, passwords, tokens, API keys, session IDs, full card numbers, one-time codes.
- Full request or response bodies by default, which often contain the above.
- Unnecessary personal data. Log identifiers, not names, phone numbers, addresses, or message contents.
- Every step of normal processing at info level. That's noise; use debug.
- The same error at every layer.

For more depth, read `references/logging.md`.

### 5. Measure with metrics

Logs describe individual events; metrics describe the system's overall behaviour cheaply over time. Instrument:

- **Every service (RED):** the **R**ate of requests, the **E**rrors, and the **D**uration (latency distribution) of requests, per route or operation.
- **Every resource (USE):** the **U**tilisation, **S**aturation (work waiting), and **E**rrors of resources such as connection pools, worker pools, queues, CPU, and memory.
- **Every dependency:** call rate, errors, and latency for each database, cache, and external service.
- **Background work:** jobs processed, failed, retried; queue length; age of the oldest waiting item.
- **Business outcomes:** orders placed, payments succeeded and failed, sign-ups. These often reveal problems technical metrics miss.

Rules:

- **Measure latency with histograms,** and look at percentiles (p50, p95, p99), never only the average. Averages hide the slow requests users notice.
- **Keep label values bounded:** labels like `route`, `method`, `status_class`, and `dependency` are fine; labels like `userId`, `orderId`, full URLs, or error messages create a separate series for every value and can overwhelm the metrics system.
- Follow the project's metric naming conventions, including units in names (`_seconds`, `_bytes`).

For more depth, read `references/metrics.md`.

### 6. Trace requests across boundaries

- Use **OpenTelemetry** (or the project's tracing library) for distributed tracing. Prefer automatic instrumentation for frameworks, HTTP clients, and database drivers.
- **Propagate trace context** on every outgoing call and into every queued message and job, so traces connect across services and asynchronous work.
- Add spans for significant internal operations that automatic instrumentation doesn't cover, with useful attributes (identifiers, sizes, outcomes), never sensitive data.
- Record errors on spans, and mark the span as failed.
- Include the trace ID in logs, so logs and traces link to each other.

For more depth, read `references/tracing.md`.

### 7. Instrument reliability mechanisms

Make resilience visible: log and count retries, timeouts, circuit breaker state changes, fallbacks used, load shed, and rate limits hit (see `backend-reliability`). A system silently running on fallbacks looks healthy until the fallbacks fail too.

### 8. Alert on what users experience

- **Alert on symptoms,** not causes: high error rates, high latency, failing critical business flows, and queues falling behind. CPU at 80% isn't a problem unless users are affected.
- **Base alerts on reliability targets (SLOs)** where defined, such as alerting when the error budget is being used up too quickly.
- **Every alert must be actionable:** someone must need to do something, now. Alerts that are routinely ignored train people to ignore all alerts.
- **Every alert links to a runbook:** what it means, how to check, and what to do.
- Send urgent alerts to someone who can respond; send non-urgent issues to a ticket or dashboard.

For alerts, dashboards, and runbooks, read `references/alerting-and-dashboards.md`.

### 9. Keep telemetry safe and affordable

- Treat logs, metrics, and traces as potentially readable by many people and tools: apply the same data protection as anywhere else (see `security`).
- Control volume: use appropriate levels, sample high-volume traces and debug logs, avoid logging inside tight loops, and set retention periods.
- Make sure logging failures never crash the application or block requests.

### 10. Instrument as you build

Add logs, metrics, and tracing as part of writing a feature, not after the first incident. A new endpoint, job, or integration isn't finished until you could tell from production signals whether it's working.

## Instrumentation checklist

For a new endpoint, job, or integration:

- [ ] Requests or runs are counted, with errors and durations (automatic middleware may cover this).
- [ ] The correlation or trace ID flows in, through, and out, including to queued work.
- [ ] Significant business events are logged at info level with identifiers.
- [ ] Failures are logged once, with context and cause, at the right level.
- [ ] Calls to dependencies have metrics and spans (automatic instrumentation may cover this).
- [ ] Retries, fallbacks, and circuit breaker changes are recorded.
- [ ] No secrets or unnecessary personal data in logs, labels, or span attributes.
- [ ] Metric labels have bounded values.
- [ ] If it's critical, an alert exists on its error rate or latency, with a runbook.

## Example

**Poor:**

```
function processPayment(order, card):
    print("Processing payment for " + order.customer.name + " card " + card.number)
    try:
        result = paymentProvider.charge(card, order.total)
        print("done")
    catch error:
        log.error("Error: " + error)
        log.error("Payment failed")
        raise error
```

Unstructured text, the customer's name and full card number in the logs, no identifiers or correlation ID, no duration, the error logged twice and then again by whatever catches it, and no metric to show how often payments fail.

**Better:**

```
function processPayment(order, paymentToken):
    with tracer.span("payment.charge", attributes={ orderId: order.id, amount: order.total.amount, currency: order.total.currency }):
        timer = metrics.paymentDuration.startTimer(provider="mpesa")
        try:
            result = paymentProvider.charge(paymentToken, order.total)
            metrics.payments.increment(provider="mpesa", outcome="succeeded")
            log.info("payment.succeeded", orderId=order.id, paymentId=result.id)
            return result
        catch PaymentDeclined as declined:
            metrics.payments.increment(provider="mpesa", outcome="declined")
            log.info("payment.declined", orderId=order.id, reason=declined.code)    # expected outcome, not an error
            raise
        catch error:
            metrics.payments.increment(provider="mpesa", outcome="failed")
            raise PaymentFailed(order.id, cause=error)    # logged once by the top-level handler
        finally:
            timer.stop()

# The correlation ID and trace ID are added to every log line automatically by the logger's context.
```

## Common mistakes

- Unstructured log messages built by string concatenation.
- `print` or console output instead of the logger.
- Logging secrets, tokens, card numbers, or personal data; logging full request bodies.
- Logging expected outcomes (validation failures, not-found) as errors.
- Logging the same error at every layer.
- No correlation ID, or one that isn't passed to downstream calls and jobs.
- Metric labels with unbounded values such as user IDs or URLs.
- Reporting average latency instead of percentiles.
- No metrics for background jobs, queues, or business outcomes.
- Trace context lost at queue or service boundaries.
- Alerts on causes (CPU, memory) rather than user-facing symptoms; alerts nobody acts on.
- Alerts without runbooks.
- Adding instrumentation only after an incident.

## Reference files

| File | Read when |
|---|---|
| `references/logging.md` | Writing log statements, choosing levels and fields, correlation IDs, or audit logs |
| `references/metrics.md` | Adding or naming metrics, choosing metric types and labels, or measuring latency |
| `references/tracing.md` | Setting up tracing, adding spans, or propagating context across services and queues |
| `references/alerting-and-dashboards.md` | Creating alerts, dashboards, or runbooks, or defining what "healthy" means |

## Sources

Charity Majors, Liz Fong-Jones, and George Miranda, *Observability Engineering*; Betsy Beyer and others (eds.), *Site Reliability Engineering* (Google), especially "Monitoring Distributed Systems" (the four golden signals); *The Site Reliability Workbook* (Google), "Alerting on SLOs"; Tom Wilkie, the RED method; Brendan Gregg, the USE method; the OpenTelemetry specification and semantic conventions; the W3C Trace Context recommendation; Prometheus documentation on naming and instrumentation; Rob Ewaschuk, "My Philosophy on Alerting"; OWASP Logging Cheat Sheet; The Twelve-Factor App (logs as event streams).