# Pagination and Querying

Every list endpoint needs pagination, and most need filtering and sorting. Designing them well from the start avoids breaking changes and performance problems later.

## Contents

1. Why paginate from the start
2. Cursor pagination
3. Offset pagination
4. Page size limits
5. Stable ordering
6. Filtering
7. Sorting
8. Search
9. Field selection
10. Totals

## 1. Why paginate from the start

A list endpoint that returns everything works fine with ten records and fails with ten million: slow responses, memory exhaustion, and database load. Adding pagination later changes the response shape and behaviour, which breaks clients. Paginate every collection from the first version.

## 2. Cursor pagination

The server returns an opaque cursor marking where the page ended. The client passes it back to get the next page.

```
GET /orders?limit=20
→ { "data": [ ... ], "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTEwLTIwIiwiaWQiOiJvcmRfOCJ9" }

GET /orders?limit=20&cursor=eyJjcmVhdGVkQXQiOiIyMDI2LTEwLTIwIiwiaWQiOiJvcmRfOCJ9
→ { "data": [ ... ], "nextCursor": null }       # null means no more pages
```

**How it works internally:** the cursor encodes the sort values of the last item (for example, its creation time and ID). The next query fetches items after that position:

```
WHERE (created_at, id) < ($lastCreatedAt, $lastId)
ORDER BY created_at DESC, id DESC
LIMIT 21          # fetch one extra to know whether another page exists
```

**Advantages:**

- Fast at any depth, using an index, unlike large offsets.
- Stable when items are added or removed: no skipped or repeated items.

**Guidelines:**

- Make cursors opaque to clients (for example, base64-encoded). Clients must not construct or parse them, so you can change the format.
- Validate cursors, and return `400` for invalid ones.
- Keep the filters and sort the same between pages; encode them in the cursor or reject mismatches.
- Index the columns used for ordering.

## 3. Offset pagination

```
GET /orders?limit=20&offset=40
or
GET /orders?page=3&pageSize=20
```

**Acceptable when:** data is small, changes rarely, or clients genuinely need to jump to page numbers, such as admin tables.

**Limitations:**

- Large offsets are slow, because the database must skip all earlier rows.
- When items are inserted or deleted between requests, items are skipped or repeated.

Use cursor pagination for large or frequently changing collections and public APIs. If the project already uses offsets, stay consistent unless there's a performance problem.

## 4. Page size limits

- Default page size: a sensible number such as 20 or 50.
- Maximum page size: enforce one, such as 100. Requests above it are capped at the maximum or rejected with a validation error, following the project's convention.
- Never let a client remove the limit.

## 5. Stable ordering

Always sort by something unique, or add a unique tiebreaker. Sorting only by `createdAt` is unstable when two items share a timestamp, causing items to appear on two pages or none. Sort by `createdAt` then `id`.

## 6. Filtering

Use query parameters named after fields:

```
GET /orders?status=shipped
GET /orders?status=shipped,delivered            # multiple values
GET /orders?createdAfter=2026-10-01T00:00:00Z&createdBefore=2026-11-01T00:00:00Z
GET /orders?customerId=cus_1
```

Guidelines:

- **Allow-list filterable fields.** Never pass client parameters directly into queries.
- Validate filter values with the same rules as the field itself.
- Use clear names for ranges (`createdAfter`/`createdBefore`, or `minTotal`/`maxTotal`), following the project's style.
- **Authorisation still applies:** filters narrow the caller's own accessible data. A `customerId` filter must not let a user see another customer's orders.
- Make sure common filters are supported by indexes.

## 7. Sorting

```
GET /orders?sort=createdAt           # ascending
GET /orders?sort=-createdAt          # descending
GET /orders?sort=-total,createdAt    # multiple fields
```

- **Allow-list sortable fields**, mapping each to a column through a fixed table. Sort fields can't be parameterised in SQL, so passing them through directly is an injection risk.
- Document the default sort.
- Always append a unique tiebreaker internally.

## 8. Search

- For simple matching, a `q` or `search` parameter is common: `GET /products?q=solar+lamp`.
- Simple `LIKE '%term%'` queries don't scale to large tables. For real search needs, use the database's full-text search or a dedicated search engine.
- Limit search term length, and escape wildcard characters in the input when using pattern matching.

## 9. Field selection

Some APIs let clients request only certain fields: `GET /orders?fields=id,status,total`. Use only if the project does, and allow-list the fields. It reduces payload size but adds complexity; often a well-designed default response is enough.

## 10. Totals

Total counts (`"total": 48213`) are convenient but can be expensive on large tables, since counting requires scanning matching rows.

- Omit totals by default for large collections, or make them optional (`?includeTotal=true`).
- Consider approximate counts where exact numbers aren't needed.
- Cursor pagination with a `nextCursor` is enough for "load more" and infinite scrolling, which need no total.