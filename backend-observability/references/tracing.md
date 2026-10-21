# Distributed Tracing

A trace follows one request through every service, queue, and database call it touches, showing how long each step took and where errors occurred. Traces answer the question metrics and logs struggle with: where did the time go, and which part failed?

## Contents

1. Concepts
2. OpenTelemetry
3. Automatic and manual instrumentation
4. Spans worth adding
5. Attributes
6. Context propagation
7. Asynchronous work
8. Errors in traces
9. Sampling
10. Linking traces, logs, and metrics

## 1. Concepts

- A **trace** represents one end-to-end operation, such as a user request, identified by a trace ID.
- A trace is made of **spans**: timed operations such as "handle HTTP request", "query orders table", "call payment provider". Each span has a start, a duration, a status, attributes, and a parent.
- Spans form a tree, showing what called what:

```
POST /orders                                   480 ms
├── validate request                             2 ms
├── SELECT products WHERE id IN (...)           12 ms
├── POST payment-provider/charges              410 ms   ← most of the time
└── INSERT orders                                9 ms
```

- **Context propagation** passes the trace ID and parent span ID between services, so their spans join the same trace.

## 2. OpenTelemetry

OpenTelemetry is the vendor-neutral standard for traces, metrics, and logs, supported by most monitoring platforms and languages. Prefer it unless the project uses something else.

- Use the official SDK and instrumentation packages for the project's language and frameworks.
- Configure the exporter and destination through environment variables or configuration, not code.
- Follow the OpenTelemetry **semantic conventions** for span names and attributes (HTTP, database, messaging, and so on), so tools understand them.

## 3. Automatic and manual instrumentation

**Automatic instrumentation** covers most of what matters with no code changes: incoming HTTP requests, outgoing HTTP calls, database queries, cache calls, and message publishing and consuming. Enable it first.

**Manual instrumentation** adds spans for things automatic instrumentation can't see: significant business operations, expensive computations, and calls through libraries without instrumentation.

Don't duplicate spans that automatic instrumentation already creates.

## 4. Spans worth adding

Add a manual span when the operation:

- Takes meaningful time, or could become slow (a complex calculation, file processing, a batch).
- Is a significant business step whose duration or failure you'd want to see (`payment.charge`, `invoice.generate`).
- Calls something not automatically instrumented.

Don't add spans for:

- Trivial functions that take microseconds.
- Every iteration of a loop over many items. Use one span for the batch, with the count as an attribute.

Name spans after the operation, with low-cardinality names: `invoice.generate`, not `generate invoice inv_4821`. Put identifiers in attributes.

## 5. Attributes

Attributes add searchable detail to spans.

- **Useful:** identifiers (`order.id`, `customer.id`, `tenant.id`), sizes and counts (`items.count`, `payload.size_bytes`), outcomes (`payment.outcome`), and relevant configuration (`retry.attempt`, `feature_flag.variant`).
- **Unlike metric labels, high-cardinality values such as IDs are fine** in span attributes; that's what makes traces searchable.
- **Never include** secrets, tokens, passwords, payment details, or unnecessary personal data. The same rules as logs apply (see `logging.md`).
- Follow semantic conventions for standard attributes, and a consistent naming scheme for custom ones.

## 6. Context propagation

Traces only connect across services if context is passed along.

- Use the **W3C Trace Context** headers (`traceparent`, `tracestate`), the standard that OpenTelemetry uses by default.
- Instrumented HTTP clients and servers propagate context automatically. Check that custom clients, proxies, and gateways forward the headers.
- Keep the active span context available throughout the request, using the SDK's context mechanism, so spans created deep in the code attach to the right parent.

## 7. Asynchronous work

Context is easily lost when work crosses a queue or is scheduled for later.

- **When enqueuing** a job or publishing a message, inject the trace context into the message's metadata or headers.
- **When consuming,** extract it and start the processing span as a child of, or linked to, the producing span. Links are often better for asynchronous work, since processing may happen much later or in batches.
- **For in-process asynchronous work** (promises, tasks, thread pools), make sure the language's context propagation is enabled, or pass the context explicitly.
- **For outbox publishing,** store the trace context with the outbox row (see `backend-data`).

## 8. Errors in traces

- When an operation fails, record the exception on the span and set the span's status to error.
- Add the error type or code as an attribute.
- Don't mark spans as errors for expected outcomes, such as a `404` from a lookup that legitimately finds nothing, or a validation failure, unless the project treats them so.

## 9. Sampling

Recording every trace in a busy system is expensive. Sampling keeps a representative subset.

- **Head sampling** decides at the start of a trace (for example, keep 10% of traces). Simple and cheap, but can miss rare errors.
- **Tail sampling** decides after the trace completes, so it can keep all errors and slow traces while sampling routine ones. Requires a collector that supports it.
- Make sampling decisions consistently across services (the sampling decision propagates in the trace context), so traces aren't left with missing parts.
- Configure sampling rates through configuration, so they can change without deployments.

## 10. Linking traces, logs, and metrics

The signals are most powerful together:

- **Logs include the trace ID and span ID,** so from a log line you can open the full trace, and from a trace you can find its logs. Configure the logger to add them automatically.
- **Metrics can carry exemplars** (references to example trace IDs) where supported, so a spike on a latency chart leads straight to a slow trace.
- **Use consistent identifiers** (service names, route templates, attribute names) across all three, so you can move between them easily.