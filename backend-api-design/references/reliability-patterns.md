# Reliability Patterns for APIs

Clients retry, networks drop responses, users double-click, and two people edit the same record. These patterns keep the API correct when that happens.

## Contents

1. Idempotency keys
2. Optimistic concurrency control
3. Long-running operations
4. Bulk operations
5. Webhooks
6. Rate limiting responses
7. Timeouts

## 1. Idempotency keys

A `POST` that creates an order or charges a card is dangerous to retry: if the response was lost, the client can't tell whether it succeeded, and a retry may create a duplicate. An idempotency key solves this.

**How it works:**

1. The client generates a unique key (such as a UUID) for each logical operation and sends it in an `Idempotency-Key` header.
2. The server stores the key with the result of the first request.
3. If a request arrives with a key already seen, the server returns the stored result instead of performing the operation again.

```
POST /payments
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
{ "orderId": "ord_9", "amount": { "amount": 250000, "currency": "KES" } }
→ 201 { "id": "pay_1", "status": "succeeded" }

(same request retried after a timeout)
→ 201 { "id": "pay_1", "status": "succeeded" }     # same result, no second charge
```

**Implementation guidelines:**

- Store keys per client or user, so keys from different clients can't collide.
- Store the key and the operation in the same transaction where possible, so a crash can't leave the operation done but the key unrecorded.
- Handle concurrent requests with the same key: lock on the key, and return `409` to the second request while the first is still in progress.
- If the same key arrives with a *different* request body, reject it with `422` or `409`. It's a client bug.
- Keep keys for a reasonable period (commonly 24 hours or more).
- Require idempotency keys for operations with financial or external side effects: payments, transfers, orders, sending messages.

## 2. Optimistic concurrency control

When two clients read a resource, both change it, and both save, the second save silently overwrites the first (a "lost update"). Optimistic concurrency control detects this.

**With ETags:**

```
GET /documents/doc_1
→ 200  ETag: "v7"   { ... }

PATCH /documents/doc_1
If-Match: "v7"
{ "title": "New title" }
→ 200  ETag: "v8"          # succeeded; version advanced
→ 412 Precondition Failed  # someone else changed it since v7
```

**With a version field:** include `"version": 7` in the resource, require it in updates, and return `409` on mismatch.

**Server side:** the update must check and increment the version atomically:

```
UPDATE documents SET title = $1, version = version + 1
WHERE id = $2 AND version = $3
-- zero rows updated means the version didn't match
```

Use it for resources that several people or processes may edit: documents, settings, inventory, profiles.

## 3. Long-running operations

Requests that take more than a few seconds (exports, report generation, large imports, video processing) shouldn't hold a connection open.

```
POST /exports
{ "type": "orders", "from": "2026-01-01", "to": "2026-09-30" }
→ 202 Accepted
  Location: /exports/exp_4
  { "id": "exp_4", "status": "pending" }

GET /exports/exp_4
→ 200 { "id": "exp_4", "status": "processing", "progress": 0.4 }

GET /exports/exp_4
→ 200 { "id": "exp_4", "status": "completed", "downloadUrl": "https://...", "expiresAt": "..." }
```

Guidelines:

- Return `202` with a status resource immediately, then do the work in a background job.
- Status values: `pending`, `processing`, `completed`, `failed`, plus `cancelled` if cancellation is supported.
- On failure, include an error in the standard format in the status resource.
- Optionally notify completion with a webhook.
- Make result links expire, and apply authorisation to the status resource and the result.

## 4. Bulk operations

When clients need to create or update many items:

- Provide a bulk endpoint rather than forcing hundreds of single requests: `POST /contacts/batch`.
- Limit the number of items per request.
- Decide and document whether the operation is **all-or-nothing** (one invalid item fails the whole batch) or **partial** (each item succeeds or fails independently).
- For partial success, return a result for each item, matched by position or by a client-provided reference:

```
→ 200
{
  "results": [
    { "ref": "a", "status": "created", "id": "con_1" },
    { "ref": "b", "status": "failed", "error": { "code": "INVALID_EMAIL", "message": "..." } }
  ]
}
```

- For very large batches, use the long-running operation pattern.

## 5. Webhooks

Webhooks are requests your system sends to clients when something happens.

**Sending webhooks:**

- **Sign every request** with an HMAC of the body using a per-client secret, and include a timestamp, so receivers can verify authenticity and reject replays.
- **Retry with increasing delays** on failure or timeout, for a limited period, then mark the delivery failed and let the client see and replay it.
- **Include an event ID,** so receivers can detect duplicates. Delivery is "at least once", so duplicates will happen.
- **Keep payloads small and stable:** an event type, an ID, a timestamp, and the relevant data or a link to fetch it.
- **Version the payload format** like any API.
- **Apply `security` rules for outgoing requests:** validate client-registered URLs, block internal addresses, and use short timeouts.

**Receiving webhooks:**

- Verify the signature before trusting the content, using constant-time comparison.
- Reject old timestamps to prevent replays.
- Respond quickly (with `2xx`) and process the event in a background job.
- Handle duplicates and out-of-order delivery: process idempotently, keyed by event ID.

## 6. Rate limiting responses

When a client exceeds a limit:

- Return `429 Too Many Requests` in the standard error format.
- Include `Retry-After` with the number of seconds to wait.
- Where the project supports them, include headers showing the limit, remaining requests, and reset time, so well-behaved clients can slow down before hitting the limit.

## 7. Timeouts

- Set a server-side timeout on every request, so stuck requests don't consume resources indefinitely.
- Set timeouts on every call the API makes to databases and other services (see `backend-reliability`).
- If an operation can't finish within a reasonable request time, use the long-running operation pattern rather than raising the timeout.