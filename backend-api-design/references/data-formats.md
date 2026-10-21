# Data Formats

Consistent data formats let clients handle every endpoint the same way. Inconsistency, such as dates in three formats, forces special cases in every client.

## Contents

1. General conventions
2. Field types
3. Response structure
4. Requests
5. Partial updates and null
6. Large payloads and files
7. Representing relationships

## 1. General conventions

- **JSON** for request and response bodies, unless the project uses something else.
- **One field naming style** across the entire API, matching the project (`camelCase` or `snake_case`).
- **Consistent names for the same concept** in every endpoint: if it's `createdAt` in one, it's `createdAt` everywhere.
- **Descriptive names:** `dueDate`, not `dd`; `isArchived`, not `flag`.
- **Booleans named as questions:** `isActive`, `hasAttachments`.

## 2. Field types

| Kind of value | Format | Example |
|---|---|---|
| Timestamp | ISO 8601 / RFC 3339 in UTC | `"2026-10-21T14:30:00Z"` |
| Date without time | ISO 8601 date | `"2026-10-21"` |
| Duration | ISO 8601 duration or explicit unit in the name | `"PT30M"` or `"timeoutSeconds": 1800` |
| Money | Integer in smallest unit plus currency code, or decimal string plus currency | `{ "amount": 250000, "currency": "KES" }` |
| Identifier | String | `"ord_9fK2"`, `"42"` |
| Enumeration | Lowercase descriptive string | `"pending"`, `"shipped"` |
| Country, currency, language | ISO codes | `"KE"`, `"KES"`, `"sw"` |
| Phone number | E.164 format | `"+254712345678"` |
| Large integers | String, if they may exceed JavaScript's safe integer range | `"9007199254740993"` |

Why:

- **No floating-point money:** floats can't represent many decimal amounts exactly, causing rounding errors.
- **String IDs:** let you change ID formats later, and avoid precision problems with large numbers.
- **String enumerations:** are readable, self-documenting, and don't break when values are reordered. Document that new values may be added, so clients handle unknown ones gracefully.
- **UTC timestamps:** avoid time-zone ambiguity. Convert to local time in the client.
- **Units in names** where a number has a unit: `sizeBytes`, `durationMs`, `weightGrams`.

## 3. Response structure

Choose one structure and use it everywhere. Common patterns:

**Single resource:** return the resource object, optionally wrapped:

```
{ "id": "ord_9", "status": "shipped", ... }
or
{ "data": { "id": "ord_9", ... } }
```

**Collection:** always wrap, so pagination information and future metadata have a place:

```
{
  "data": [ { ... }, { ... } ],
  "nextCursor": "eyJpZCI6Im9yZF84In0"
}
```

Never return a bare array at the top level for a collection. It leaves nowhere to add pagination later without breaking clients.

**Errors:** use the error format from the `error-handling` skill (RFC 9457 Problem Details or the project's format), with `Content-Type: application/problem+json` where RFC 9457 is used.

Guidelines:

- Include the resource's `id` in every representation.
- Return the full updated resource after create and update operations, so clients don't need another request.
- Build responses with explicit mapping functions. Never serialise database records or internal objects directly; that leaks internal fields and couples the API to the database schema.
- Be consistent about absent values: either omit fields with no value or include them as `null`, project-wide.

## 4. Requests

- Validate every request against a schema (see `SKILL.md`, rule 7).
- Accept only the fields the client may set. Server-controlled fields (`id`, `createdAt`, `status` where transitions are controlled, `ownerId`) are never taken from the request.
- Use the same field names and formats in requests as in responses.
- Keep request bodies flat where possible; nest only for genuine sub-objects such as an address.

## 5. Partial updates and null

With `PATCH` using JSON Merge Patch semantics:

| Sent | Meaning |
|---|---|
| Field absent | Leave unchanged |
| Field with a value | Set to that value |
| Field set to `null` | Clear the value (if the field is nullable) |

```
PATCH /customers/cus_1
{ "phone": "+254700000000", "middleName": null }
→ phone updated, middleName cleared, everything else unchanged
```

Make sure the code distinguishes "absent" from "null". Many validation libraries treat them the same by default.

## 6. Large payloads and files

- Set maximum sizes for request bodies, and reject larger ones with `413`.
- For file uploads, prefer direct-to-storage uploads with signed URLs from object storage, so large files don't pass through the API servers. Otherwise, use `multipart/form-data`. Apply the `security` skill's upload rules.
- For large downloads or exports, generate the file asynchronously and return a link (see `reliability-patterns.md`).
- Enable response compression for large JSON responses.

## 7. Representing relationships

- **Reference by ID** by default: `{ "customerId": "cus_1" }`.
- **Embed** related data when clients almost always need it together, and it's small.
- **Optional expansion:** allow clients to request related data with a parameter such as `?expand=customer`, if the project uses this pattern. Allow-list which relations can be expanded, and apply authorisation to the expanded data too.
- Avoid deeply nested responses that load large object graphs; they're slow and hard to evolve.