---
name: backend-api-design
description: Principles for designing HTTP APIs that are predictable, consistent, safe to call, and able to evolve without breaking clients, in any backend language or framework. Covers resource and URL design, HTTP methods and status codes, request and response formats, error responses, validation, pagination, filtering and sorting, idempotency, concurrency control, long-running operations, webhooks, versioning and backward compatibility, API security, caching, and documentation with OpenAPI. Use this skill whenever creating, changing, or reviewing an API endpoint, route, controller, request or response schema, webhook, or API contract, and whenever deciding URL structure, status codes, payload shape, or how an API should change, even if the task only says "add an endpoint".
---

# Backend API Design

An API is a promise to everyone who calls it. Once clients depend on it, every detail (URL, field name, status code, error format, ordering) becomes hard to change. Good API design makes that promise clear, consistent, and safe to keep: clients can guess how an endpoint behaves before reading its documentation, retries don't cause duplicates, and the API can grow for years without breaking anyone.

This skill focuses on HTTP APIs in the REST style, the most common case. The principles of consistency, idempotency, safe evolution, and clear errors apply to other styles too.

Read the project's `AGENTS.md` first. Follow the project's existing API conventions: URL style, field naming, response envelope, error format, pagination style, and authentication. **Consistency with the existing API matters more than any rule here.** Where the project's conventions differ from this skill, follow the project. Apply the `security` and `error-handling` skills alongside this one.

## Before you start

1. **Who calls this API?** A web or mobile app, other internal services, or external developers? External and mobile clients make breaking changes much costlier.
2. **What are the existing conventions?** Look at several existing endpoints and the API specification, if one exists. Match them exactly.
3. **What resource is this?** Identify the noun the endpoint is about, and how it relates to existing resources.
4. **Is this a change to an existing endpoint?** If so, will it break any current client? See rule 12.

## Core rules

### 1. Design from the client's point of view

- Design the API around what clients need to do, not around database tables or internal structure. An endpoint should expose a clear business concept.
- Write or update the contract (such as an OpenAPI specification) as part of the change, so the API's shape is explicit and reviewable.
- Hide implementation details: internal IDs, table names, and internal states shouldn't leak into the API.

### 2. Model resources as nouns

- Use plural nouns for collections and IDs for items: `/orders`, `/orders/{orderId}`.
- Nest only for real ownership, and no more than one level: `/orders/{orderId}/items`. Prefer `/items?orderId=...` or top-level resources over deep nesting.
- Use lowercase, hyphen-separated path segments: `/payment-methods`.
- Don't put verbs in URLs for standard operations. The HTTP method is the verb: `POST /orders`, not `POST /createOrder`.
- For actions that don't map to create, read, update, or delete, use a clearly named action resource: `POST /orders/{orderId}/cancellation` or `POST /orders/{orderId}/cancel`, following the project's style.

### 3. Use HTTP methods for their meaning

| Method | Use for | Safe | Idempotent |
|---|---|---|---|
| `GET` | Reading | Yes | Yes |
| `POST` | Creating, or performing an action | No | No (make it so with idempotency keys) |
| `PUT` | Replacing a resource completely | No | Yes |
| `PATCH` | Partially updating a resource | No | Not necessarily |
| `DELETE` | Removing a resource | No | Yes |

- **`GET` must never change data.** Crawlers, caches, prefetching, and retries all assume it's safe.
- **`PUT` and `DELETE` must be idempotent:** repeating them has the same result as doing them once. Deleting something already deleted returns success or 404 consistently, not an error that breaks retries.

### 4. Return accurate status codes

Use the code that matches what happened. The most common:

| Code | When |
|---|---|
| `200 OK` | Successful read or update, with a response body |
| `201 Created` | Resource created; include it in the body and a `Location` header |
| `202 Accepted` | Request accepted for processing later |
| `204 No Content` | Success with no body, often for `DELETE` |
| `400 Bad Request` | Malformed request |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Authenticated but not permitted |
| `404 Not Found` | Resource doesn't exist, or the caller mustn't know it exists |
| `409 Conflict` | Conflicts with current state (duplicate, version mismatch) |
| `422 Unprocessable Content` | Valid format, but fails validation or a business rule |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server failure |
| `503 Service Unavailable` | Temporarily unavailable |

Never return `200` with an error in the body. Clients, proxies, and monitoring rely on the status code.

For the full guide to methods and status codes, read `references/http-semantics.md`.

### 5. Keep request and response formats consistent

- Use one field naming style everywhere (commonly `camelCase` or `snake_case`), matching the project.
- **Dates and times:** ISO 8601 strings in UTC, such as `2026-10-21T14:30:00Z`.
- **Money:** an integer amount in the smallest currency unit plus a currency code, `{ "amount": 250000, "currency": "KES" }`, or a decimal string. Never a floating-point number.
- **IDs:** strings, even if they're numbers internally, so formats can change later.
- **Enumerations:** descriptive strings such as `"shipped"`, never numeric codes.
- **Return only what clients need.** Build response objects explicitly, never by returning database records directly.

For more depth, read `references/data-formats.md`.

### 6. Return consistent, useful errors

- Use one error format for every endpoint, such as RFC 9457 Problem Details, or the project's own.
- Include a stable machine-readable `code`, a human-readable message, a request ID, and, for validation errors, the field each problem concerns.
- Never expose stack traces, SQL, internal paths, or library messages.
- Apply the `error-handling` skill for the full approach.

### 7. Validate every request at the boundary

- Validate the body, path parameters, query parameters, and relevant headers against a schema, using the project's validation library.
- Reject unknown or read-only fields rather than silently accepting them, so clients can't set fields like `role`, `status`, or `ownerId`.
- Enforce size limits on bodies, strings, and arrays.
- Return all validation problems at once, not one per request.

### 8. Paginate every collection

- Never return an unbounded list. Every collection endpoint is paginated from the start, because adding pagination later is a breaking change.
- Prefer **cursor-based pagination** for large or frequently changing data; offset pagination is acceptable for small, stable data or where the project already uses it.
- Set a default page size and a maximum, and enforce the maximum.
- Guarantee a stable order (for example, by creation time then ID), so pages don't skip or repeat items.

For pagination, filtering, sorting, and field selection, read `references/pagination-and-querying.md`.

### 9. Make retries safe

Networks fail, and clients retry. Design so a retry never causes harm.

- `GET`, `PUT`, and `DELETE` are naturally idempotent if implemented correctly.
- For `POST` requests that create resources, move money, or send messages, **accept an `Idempotency-Key` header**. Store the result for each key and return the same result for repeated requests with that key.
- Detect concurrent conflicting updates with optimistic concurrency: a version number or `ETag`, checked with `If-Match`, returning `409` or `412` on mismatch.

For idempotency, concurrency, long-running operations, bulk operations, and webhooks, read `references/reliability-patterns.md`.

### 10. Handle slow work asynchronously

If an operation takes more than a few seconds, accept it with `202 Accepted` and return a status resource the client can check: `GET /exports/{exportId}`. Don't keep clients waiting on long requests.

### 11. Secure every endpoint

Apply the `security` skill. In particular, for APIs:

- Require authentication by default; mark public endpoints explicitly.
- Check object-level authorisation on every endpoint that takes an ID, preferably by scoping queries to the caller.
- Rate-limit, with stricter limits on authentication and expensive endpoints.
- Never put secrets, tokens, or sensitive personal data in URLs; they end up in logs and browser history. Use headers or the body.

### 12. Evolve without breaking clients

Assume every field and behaviour you publish is relied on by someone.

**Safe (non-breaking) changes:** adding a new endpoint, adding an optional request field, adding a response field, adding a new enumeration value where clients were told to expect new values.

**Breaking changes:** removing or renaming a field, endpoint, or parameter; changing a field's type or format; making an optional field required; changing status codes, error codes, or default behaviour; tightening validation on existing fields.

- Make only additive changes to an existing version.
- When a breaking change is truly needed, introduce it in a new version, keep the old one working during a published deprecation period, and signal deprecation to clients.
- Treat breaking changes as decisions for the person you're working with (see the `decisions` skill).

For versioning strategies and deprecation, read `references/evolution-and-versioning.md`.

### 13. Avoid chatty and slow endpoints

- Design endpoints so common client tasks need few requests.
- Avoid N+1 queries when building responses: load related data in batches (see `backend-data`).
- Use caching headers (`Cache-Control`, `ETag`) for data that can be cached, and compression for large responses.

### 14. Document and test the contract

- Keep the API specification in sync with the code. Prefer generating one from the other, as the project does.
- Include an example request and response for each endpoint, and document every error code it can return.
- Write API tests for each endpoint: success, validation errors, authentication, authorisation (including another user's resources), not-found, and conflicts. Apply the `testing` skill.

## Example

**Poor:**

```
POST /getUserOrders          body: { "userId": 42 }
→ 200 { "success": false, "error": "user not found" }

POST /order/update           body: { "id": 9, "status": 3, "total": 99.99, "role": "admin" }
→ 200 [ ...every column of the orders table... ]
```

Verbs in URLs, the user ID taken from the body (anyone can read anyone's orders), an error returned with `200`, numeric status codes, money as a float, unknown fields like `role` accepted, internal columns returned, and no pagination.

**Better:**

```
GET /orders?status=shipped&limit=20&cursor=eyJpZCI6...
Authorization: Bearer <token>                    # user identity comes from the token
→ 200
{
  "data": [
    { "id": "ord_9", "status": "shipped", "total": { "amount": 9999, "currency": "KES" },
      "createdAt": "2026-10-20T09:15:00Z" }
  ],
  "nextCursor": "eyJpZCI6Im9yZF84In0"
}

PATCH /orders/ord_9
If-Match: "v3"
{ "shippingAddress": { ... } }                   # only fields the user may change
→ 200 { ...updated order... }
→ 412 if the order changed since version v3
→ 422 { "code": "VALIDATION_FAILED", "errors": [...], "requestId": "8f3a2c" }

POST /orders/ord_9/cancel
Idempotency-Key: 5f1c9e2a-...
→ 200 { ...cancelled order... }                  # same response if retried with the same key
→ 409 { "code": "ORDER_ALREADY_SHIPPED", ... }
```

## Common mistakes

- Verbs in URLs, or inconsistent naming between endpoints.
- `GET` requests that change data.
- Returning `200` for errors, or using generic `400` and `500` for everything.
- Taking the user's identity from the request body or query instead of the verified token.
- Missing object-level authorisation checks.
- Returning database records directly, including internal or sensitive fields.
- Accepting unknown fields, allowing clients to set protected values.
- Unpaginated list endpoints.
- Money as floating-point numbers; dates in local time or inconsistent formats.
- Numeric codes for statuses and enumerations.
- `POST` endpoints for payments or orders with no idempotency protection.
- Renaming or removing fields in an existing version.
- Error formats that differ between endpoints.
- An API specification that doesn't match the code.

## Reference files

| File | Read when |
|---|---|
| `references/http-semantics.md` | Choosing methods, status codes, or headers, or designing URLs and actions |
| `references/data-formats.md` | Designing request or response bodies, field types, envelopes, or partial updates |
| `references/pagination-and-querying.md` | Building list endpoints, pagination, filtering, sorting, search, or field selection |
| `references/reliability-patterns.md` | Handling retries, idempotency keys, concurrent updates, long-running work, bulk operations, or webhooks |
| `references/evolution-and-versioning.md` | Changing an existing API, versioning, or deprecating endpoints or fields |

## Sources

Roy Fielding, "Architectural Styles and the Design of Network-based Software Architectures" (REST); RFC 9110, *HTTP Semantics*; RFC 9457, *Problem Details for HTTP APIs*; RFC 5789 (PATCH); RFC 7396, *JSON Merge Patch*; RFC 8594 (the `Sunset` header) and RFC 9745 (the `Deprecation` header); the OpenAPI Specification; IETF draft, *The Idempotency-Key HTTP Header Field*; Google API Design Guide and AIPs; Microsoft REST API Guidelines; Zalando RESTful API Guidelines; Stripe API documentation (idempotency and versioning practices); OWASP API Security Top 10.