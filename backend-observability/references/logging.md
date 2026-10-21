# Logging

Logs are the detailed record of what happened. Good logs let you reconstruct a specific request, job, or failure after the fact. This file covers writing them well.

## Contents

1. Structured logging
2. Event names and fields
3. Standard fields
4. Levels in practice
5. Correlation IDs
6. What to log at each point
7. Sensitive data
8. Volume and cost
9. Audit logs
10. Where logs go

## 1. Structured logging

Structured logs are records with named fields, usually emitted as JSON, rather than lines of prose.

```
Unstructured:
    2026-10-21 14:30:02 INFO Order ord_9 placed by customer cus_1 for 2500 KES

Structured:
    {"timestamp":"2026-10-21T14:30:02.341Z","level":"info","event":"order.placed",
     "orderId":"ord_9","customerId":"cus_1","amount":250000,"currency":"KES",
     "requestId":"8f3a2c91","traceId":"4bf92f3577b34da6a3ce929d0e0e4736","service":"orders"}
```

Structured logs can be searched, filtered, and aggregated reliably: "all `payment.failed` events for provider X in the last hour", without fragile text searches.

- Use the project's structured logging library and configuration.
- Pass values as fields, not by interpolating them into the message.
- In local development, a human-friendly format is fine; the logger should switch formats by configuration, not by code changes.

## 2. Event names and fields

- **Event names** are stable, dot-separated or similar identifiers describing what happened, in the past tense: `order.placed`, `payment.declined`, `job.failed`, `cache.miss`. They're easier to search than sentences, and don't change when wording is improved.
- If the project uses a human-readable `message` instead, keep it constant for a given event and put variable data in fields.
- **Field names** are consistent across the codebase. Agree on one name per concept (`orderId`, `userId`, `durationMs`) and use it everywhere. Follow the OpenTelemetry semantic conventions or the project's own list where one exists.
- **Include units** in names for numbers: `durationMs`, `sizeBytes`.
- **Identifiers over descriptions:** `customerId`, not the customer's name.

## 3. Standard fields

Configure the logger to add these to every record automatically, rather than adding them by hand:

| Field | Purpose |
|---|---|
| `timestamp` | When, in UTC, with milliseconds |
| `level` | Severity |
| `event` or `message` | What happened |
| `service` | Which service emitted it |
| `environment` | Production, staging, and so on |
| `version` | Application version or commit, to correlate with deployments |
| `requestId` / `correlationId` | Links all records for one request |
| `traceId`, `spanId` | Links logs to traces |
| `userId` or `tenantId` | Who it concerns, where appropriate and permitted |

Use the logger's context mechanism (sometimes called a child logger, log context, or mapped diagnostic context) so request-specific fields are attached automatically for the duration of the request or job.

## 4. Levels in practice

| Situation | Level |
|---|---|
| Unhandled exception reaching the top-level handler | Error |
| Payment provider returns `500` after all retries | Error |
| Job moved to the dead-letter queue | Error |
| Request succeeded after one retry | Warning (or debug, if routine) |
| Fallback used because a dependency was unavailable | Warning |
| Circuit breaker opened | Warning |
| Deprecated endpoint called | Warning |
| User failed validation | Info or debug, not error |
| Wrong password entered | Info (security event), not error |
| Order placed | Info |
| Service started, with configuration summary | Info |
| Cache hit or miss for each request | Debug |
| Request and response details for diagnosis | Debug, with sensitive fields removed |

A good test for **error**: would someone reasonably want to investigate this occurrence? If it happens hundreds of times a day as part of normal operation, it isn't an error.

## 5. Correlation IDs

A correlation ID ties together everything that happened because of one request.

1. **At the edge:** accept an incoming ID header (such as `X-Request-Id`, or the trace context's `traceparent`) if present and well-formed; otherwise generate one.
2. **Throughout the request:** attach it to the logger's context, so every log line includes it.
3. **On outgoing calls:** send it to downstream services in a header.
4. **Into queued work:** include it in job and message metadata, and restore it into the logger's context when the job runs.
5. **In responses:** return it in a header and in error responses (see `error-handling`), so users and support can quote it.

With tracing in place, the trace ID can serve as the correlation ID. Use whichever the project does, consistently.

## 6. What to log at each point

**Request boundary** (usually middleware, once per request): method, route template (not the raw URL with IDs and query strings), status, duration, correlation ID, user or client identifier.

**Business operations:** one info event per significant outcome, with identifiers. Not one per internal step.

**Calls to dependencies:** usually covered by metrics and traces; log failures, retries, and fallbacks rather than every successful call.

**Background jobs:** start (debug or info), completion with duration and counts (info), failure with context (error after final attempt, warning for retried attempts).

**Startup and shutdown:** version, effective configuration without secrets, and lifecycle events.

**Security events:** see `security`.

## 7. Sensitive data

Never log:

- Passwords, password hashes, tokens, API keys, session IDs, cookies, authorisation headers.
- Full card numbers, security codes, bank credentials, one-time codes.
- Private keys and signing secrets.

Avoid logging, or mask:

- Personal data: names, email addresses, phone numbers, addresses, national IDs, dates of birth, location, message contents. Log identifiers instead.
- Full request and response bodies, query strings, and headers, which often contain the above.

Protective measures:

- Configure the logger to redact known sensitive field names (`password`, `token`, `authorization`, `cardNumber`) automatically, as a safety net, not a substitute for care.
- Log selected fields from objects, not whole objects. Logging an entire user or request object is how sensitive data usually leaks.
- Restrict who can read production logs, and set retention periods.

## 8. Volume and cost

Logs cost money to store and search, and excessive volume hides important records.

- Don't log inside tight loops or for every item in a large batch; log a summary.
- Keep debug logging off in production by default; make it possible to enable temporarily, ideally for a specific request, user, or component.
- Sample very high-volume, low-value events if needed.
- Avoid logging large payloads.
- Watch for repeated identical errors flooding the logs; fix the cause, and consider rate-limited logging for known noisy conditions.

## 9. Audit logs

Audit logs are a distinct kind of record: who did what to which data, kept for accountability, security investigations, or compliance.

- Record actor, action, target, time, source, and outcome.
- Store them separately from operational logs, with stricter access controls and longer retention, where the project requires it.
- Protect them from modification.
- Don't rely on ordinary application logs, which may be sampled, rotated, or deleted, for audit purposes.

## 10. Where logs go

- Write logs to standard output (or the project's configured destination), and let the platform collect and ship them (Twelve-Factor App). Don't manage log files and rotation in application code unless the project requires it.
- Logging must never block or crash the application: use asynchronous or buffered logging where the library supports it, and handle logging failures quietly.