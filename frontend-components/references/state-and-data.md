# State and Data

Most frontend bugs are state bugs: values out of sync, stale data, contradictory flags, or updates that overwrite each other. This file covers where state belongs and how to handle data from the server.

## Contents

1. Kinds of state
2. Choosing where state lives
3. Derived state
4. Modelling state explicitly
5. URL state
6. Global stores
7. Server state
8. Mutations and keeping data fresh
9. Race conditions
10. Persisting state in the browser

## 1. Kinds of state

| Kind | Examples | Characteristics |
|---|---|---|
| **UI state** | Menu open, selected tab, hover, input text while typing | Local, short-lived |
| **Form state** | Field values, touched fields, validation errors | Local to the form until submitted |
| **URL state** | Search query, filters, sort, page, selected item | Shareable, survives refresh, works with back and forward |
| **Server state** | Orders, products, the user's profile | Owned by the server; the frontend holds a cached copy that can become stale |
| **Global client state** | Theme, locale, feature flags, signed-in user summary | Needed widely; changes rarely |

Server state and client state need different tools. Server state needs caching, refetching, invalidation, and deduplication; client state needs simple, predictable updates.

## 2. Choosing where state lives

Ask, in order:

1. **Can it be computed from other state or props?** Then don't store it; derive it.
2. **Does only one component use it?** Keep it local.
3. **Do a few related components use it?** Lift it to their nearest common parent.
4. **Should it be in the URL?** If a user would expect to bookmark, share, or refresh and keep it, yes.
5. **Is it server data?** Use the data-fetching layer.
6. **Is it needed across distant parts of the app, and client-only?** Use context or a global store.

Keep state as close as possible to where it's used. State placed too high causes unnecessary re-rendering and makes components harder to understand.

## 3. Derived state

If a value can be calculated from existing state, calculate it instead of storing it.

```
Avoid:
    items = state([...])
    total = state(0)
    on items change: total.set(sum(items.price))     # can drift if an update is missed

Prefer:
    items = state([...])
    total = derived(sum(items.price))                # always correct
```

Common derived values: filtered and sorted lists, totals and counts, "is the form valid?", "has anything changed?", display formats.

Only store a derived value (memoised or cached) if computing it is measurably expensive.

## 4. Modelling state explicitly

Replace sets of booleans with a single status, so impossible combinations can't occur:

```
Avoid:   isLoading, isError, isSuccess, data, error     # isLoading and isError both true?

Prefer:  state =
           { status: "idle" }
         | { status: "loading" }
         | { status: "success", data }
         | { status: "error", error }
```

For complex interactions (multi-step flows, wizards, drag and drop, media players), model the component as a state machine: a fixed set of states and allowed transitions. Libraries exist for this, or a reducer with explicit states works well.

## 5. URL state

Put state in the URL when the user would expect it to persist or be shareable:

- Search queries, filters, sort order, page number.
- Selected tab or view mode on a page.
- The item shown in a detail panel.

Benefits: refresh keeps the view, links can be shared, the back button works, and analytics see meaningful URLs.

Guidelines:

- Validate and parse URL parameters like any input; they can be anything.
- Use the router's tools for reading and updating parameters.
- Don't put sensitive data in URLs.
- Replace rather than push history entries for rapid changes such as typing, so the back button isn't flooded.

## 6. Global stores

Global stores (and context) are useful for truly global client state, but are often overused.

- **Use them for:** theme, locale, signed-in user summary, feature flags, cross-cutting UI such as a notification queue.
- **Avoid them for:** server data (use the data layer), state used by one feature (keep it in the feature), and form state.
- **Keep stores small and focused,** split by domain rather than one store for everything.
- **Update through defined actions,** never by mutating store state directly from components.
- **Select only what a component needs,** so it re-renders only when that part changes.

## 7. Server state

Server data in the frontend is a cache of something owned by the server. Treat it that way:

- **Use a data-fetching library** or the framework's data loading (such as a query library or route loaders) that provides caching, deduplication of identical requests, background refetching, loading and error states, and cancellation.
- **Identify data with stable keys** that include every parameter: `["orders", { status, page }]`.
- **Decide how fresh data must be:** some data can be cached for minutes; some should be refetched whenever the user returns to the screen.
- **Keep API calls in a client layer** (`api/orders`), which handles base URLs, authentication headers, error parsing, and types. Components call this layer, not raw `fetch` with hand-built URLs.
- **Validate response data** at the boundary, where the project does, so unexpected shapes fail clearly rather than causing errors deep in components.

## 8. Mutations and keeping data fresh

After creating, updating, or deleting data:

- **Invalidate or update** the cached data affected by the change, so lists and details reflect it. For example, after creating an order, invalidate the orders list.
- **Show progress** while the mutation runs, and **prevent duplicate submissions.**
- **Handle failure** with a clear message and a way to retry, keeping the user's input.
- **Optimistic updates** (showing the change before the server confirms) make interfaces feel fast, but must roll back cleanly on failure. Use them for low-risk, likely-to-succeed actions such as toggling a favourite, not for payments or irreversible actions (see `forms-and-feedback.md`).
- Make server-side mutations idempotent where retries are possible (see `backend-api-design`).

## 9. Race conditions

When requests can overlap, responses can arrive out of order:

```
User types "sh"  → request A (slow)
User types "sho" → request B (fast)
B arrives → shows results for "sho"
A arrives → overwrites with results for "sh"   ✗
```

Prevent this by:

- Using a data library that tracks the latest request per key (most do).
- Cancelling the previous request when inputs change (with an abort signal).
- Ignoring responses that don't match the current inputs.
- Debouncing rapid inputs such as search typing, to reduce requests in the first place.

Also guard against updating state after a component has been removed, by cancelling requests in cleanup.

## 10. Persisting state in the browser

- Persist only what improves the experience: preferences, drafts, recently viewed items.
- Never store secrets or sensitive personal data in local storage; any script on the page can read it (see `security`).
- Version persisted data, and handle missing, old, or corrupted values gracefully, since it can outlive many releases.
- Wrap storage access in error handling; storage can be full, disabled, or unavailable in private browsing.