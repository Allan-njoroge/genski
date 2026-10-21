# HTTP Semantics

HTTP has well-defined meanings for methods, status codes, and headers (RFC 9110). Clients, proxies, caches, browsers, and monitoring tools all rely on them. Using them correctly makes an API predictable for free.

## Contents

1. URL design
2. Methods in detail
3. Actions that aren't create, read, update, or delete
4. Status codes in detail
5. Useful headers
6. Choosing between 401, 403, and 404

## 1. URL design

| Guideline | Example |
|---|---|
| Plural nouns for collections | `/customers`, `/invoices` |
| ID for a single item | `/customers/{customerId}` |
| Sub-collection for genuine ownership, one level deep | `/invoices/{invoiceId}/lines` |
| Lowercase, hyphen-separated segments | `/payment-methods` |
| Query parameters for filtering, sorting, and paging | `/invoices?status=unpaid&sort=-dueDate` |
| No verbs for standard operations | `DELETE /invoices/{id}`, not `POST /deleteInvoice` |
| No file extensions | `/reports/{id}`, using `Accept` for format |
| No trailing slashes, consistently | `/customers`, not `/customers/` |

Avoid deep nesting such as `/customers/{c}/orders/{o}/items/{i}/notes`. It couples URLs to one view of relationships and becomes awkward when resources are accessed in other ways. Prefer `/order-items/{id}/notes` or query filters.

The current user's resources can use a `me` alias where the project does so: `/me/orders`. The identity still comes from the token, never from the URL.

## 2. Methods in detail

**`GET`** reads. It must be **safe** (no side effects that matter to the client) and **idempotent**. It has no request body. Clients, crawlers, and caches may call it at any time.

**`POST`** creates a resource in a collection (`POST /orders`) or performs an action. It is neither safe nor idempotent by default. For create operations, return `201 Created` with the resource and a `Location` header pointing to it.

**`PUT`** replaces a resource entirely with the representation sent. Fields omitted are cleared or reset to defaults. It is idempotent. It can create a resource at a client-chosen ID where that makes sense. Many APIs use `PATCH` instead, because full replacement is error-prone for clients.

**`PATCH`** partially updates a resource: only the fields sent are changed. Use JSON Merge Patch semantics (RFC 7396) for simple cases: send the fields to change, and `null` to clear a field. Use JSON Patch (RFC 6902) only if the project already does. Follow the project's convention.

**`DELETE`** removes a resource. It is idempotent: deleting something already deleted should not fail in a way that breaks retries. Return `204 No Content` (or `200` with a body). Whether a repeated delete returns `204` or `404`, be consistent across the API.

**`HEAD`** and **`OPTIONS`** are usually handled by the framework. `OPTIONS` is used for CORS preflight requests.

## 3. Actions that aren't create, read, update, or delete

Some operations are naturally actions: cancel, approve, refund, resend, publish. Options, in order of preference:

1. **Model the action's result as a resource:** `POST /orders/{id}/refunds` creates a refund. This is clean when the result is something with its own identity and history.
2. **Model it as a state change:** `PATCH /orders/{id}` with `{ "status": "cancelled" }`, if the change is simple and has no extra parameters or side effects. Validate allowed transitions on the server.
3. **Use an action endpoint:** `POST /orders/{id}/cancel`, when the action has side effects beyond a field change. Some style guides write it as `POST /orders/{id}:cancel`. Follow the project's style and use it consistently.

Action endpoints should be `POST`, return the updated resource or the created result, and support idempotency keys where repeating would be harmful.

## 4. Status codes in detail

**Success**

| Code | Use |
|---|---|
| `200 OK` | General success with a body |
| `201 Created` | A resource was created; include `Location` |
| `202 Accepted` | Accepted for asynchronous processing; return a way to check status |
| `204 No Content` | Success without a body |

**Redirection** (rare in APIs)

| Code | Use |
|---|---|
| `301` / `308` | Resource has permanently moved |
| `304 Not Modified` | The cached version is still current (conditional `GET`) |

**Client errors**

| Code | Use |
|---|---|
| `400 Bad Request` | Malformed syntax: invalid JSON, wrong types, missing required structure |
| `401 Unauthorized` | Not authenticated: missing, invalid, or expired credentials |
| `403 Forbidden` | Authenticated, but not allowed to do this |
| `404 Not Found` | The resource doesn't exist, or the caller mustn't learn that it does |
| `405 Method Not Allowed` | The method isn't supported on this URL |
| `409 Conflict` | Conflicts with current state: duplicate, invalid state transition, concurrent edit |
| `410 Gone` | Permanently removed, such as a retired API version |
| `412 Precondition Failed` | An `If-Match` or similar condition failed |
| `413 Content Too Large` | Request body too large |
| `415 Unsupported Media Type` | Wrong `Content-Type` |
| `422 Unprocessable Content` | Well-formed, but fails validation or business rules |
| `429 Too Many Requests` | Rate limit exceeded; include `Retry-After` |

Some projects use `400` for all validation errors and reserve `422` for business rule violations, or use `400` for both. Follow the project's convention consistently.

**Server errors**

| Code | Use |
|---|---|
| `500 Internal Server Error` | Unexpected failure |
| `502 Bad Gateway` | An upstream service returned an invalid response |
| `503 Service Unavailable` | Temporarily overloaded or in maintenance; include `Retry-After` if known |
| `504 Gateway Timeout` | An upstream service didn't respond in time |

## 5. Useful headers

| Header | Purpose |
|---|---|
| `Content-Type` | Format of the body, usually `application/json` (`application/problem+json` for RFC 9457 errors) |
| `Accept` | Format the client wants |
| `Authorization` | Credentials, such as `Bearer <token>` |
| `Location` | URL of a newly created resource, or of a status resource for `202` |
| `ETag` | Version identifier of a resource, for caching and concurrency control |
| `If-Match` | Proceed only if the resource still has this `ETag` (prevents lost updates) |
| `If-None-Match` | Return `304` if unchanged (caching) |
| `Cache-Control` | Caching rules |
| `Idempotency-Key` | Client-generated key that makes a `POST` safe to retry |
| `Retry-After` | When to retry after `429` or `503` |
| `X-Request-Id` (or similar) | Correlation ID for tracing a request through logs |
| `RateLimit-*` | Remaining quota and reset time, where the project provides them |
| `Deprecation`, `Sunset` | Signal that an endpoint is deprecated and when it will be removed |

## 6. Choosing between 401, 403, and 404

- **No valid credentials:** `401`.
- **Valid credentials, but the caller lacks a permission they could know about** (for example, a non-admin calling an admin endpoint): `403`.
- **Valid credentials, but the resource belongs to someone else:** usually `404`, so the API doesn't reveal that the resource exists. This prevents attackers from discovering valid IDs.

Whatever the project chooses, apply it consistently.