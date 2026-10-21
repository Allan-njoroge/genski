---
name: frontend-components
description: Principles for building user interface components that are focused, reusable, predictable, and fast, in any component-based framework (React, Vue, Svelte, Angular, Solid, and others). Covers component responsibilities and boundaries, props and component APIs, composition, where state should live, derived state, server data fetching and caching, loading, error, and empty states, side effects and cleanup, forms, rendering performance, styling with design tokens, error boundaries, frontend security, and testing components from the user's point of view. Use this skill whenever creating, changing, splitting, or reviewing a UI component, page, view, form, hook, composable, or store; fetching data in the frontend; managing UI state; or fixing slow, flickering, or buggy interface behaviour, even if the task only says "build this screen".
---

# Frontend Components

A user interface is built from components: small pieces that each render part of the screen and respond to the user. Well-designed components are easy to understand, reuse, test, and change. Poorly designed ones become tangled: giant components that do everything, state duplicated in several places that drifts out of sync, data fetched in ways that flicker or show stale results, and screens that slow down as they grow.

The principles here apply to any component framework. Examples use neutral pseudocode; apply them through the project's framework idioms (hooks, composables, signals, services, stores).

Read the project's `AGENTS.md` first. Use the project's framework, component library or design system, state management and data-fetching tools, styling approach, folder structure, and conventions. Where the project's conventions differ from this skill, follow the project. Apply `frontend-accessibility` alongside this skill for every component, and the core skills (`software-design`, `code-quality`, `error-handling`, `testing`, `security`) as everywhere else.

## Before you start

1. **Does this already exist?** Check the project's design system and shared components before building a new button, input, modal, or table.
2. **What is this component's one job?** If the answer needs "and", it may be several components.
3. **What state does it need, and where should that state live?** Local, shared with siblings, in the URL, from the server, or global (rule 4).
4. **What does the user see while waiting, on failure, and when there's nothing to show?** Every data-driven view needs all three.
5. **How will a keyboard and screen-reader user use it?** Apply `frontend-accessibility` from the start.

## Core rules

### 1. Give each component one clear job

- A component should do one thing: render a product card, show a list of orders, manage a checkout form. Split components that mix unrelated concerns.
- Separate **what is shown** from **where data comes from** where it helps: a presentational component receives data and callbacks through props and renders them; a container (or page, or hook) fetches data and handles state. Presentational components are easy to reuse, preview, and test.
- Don't split for the sake of it. A 150-line component that reads clearly as one unit is better than ten tiny components that must be read together (see `software-design`, deep modules).
- Organise by feature: `orders/OrderList`, `orders/OrderCard`, `orders/useOrders`. Keep truly shared, generic components in a shared folder or the design system.

### 2. Design small, clear component APIs

- Keep props few and meaningful. Many props usually mean a component is doing too much.
- Name props for what they mean to the caller: `isDisabled`, `onSubmit`, `variant="danger"`, not internal details.
- Prefer a single descriptive prop over several booleans that conflict: `variant="primary" | "secondary" | "danger"` rather than `isPrimary`, `isSecondary`, `isDanger`.
- Name event props after what happened, `onSelect`, `onChange`, `onClose`, and let the parent decide what to do.
- Provide sensible defaults, so common uses need few props.
- Use the framework's type system or prop validation to make the API explicit.

For more depth, read `references/component-design.md`.

### 3. Compose instead of configuring

- Build flexible components by accepting children or slots, rather than adding a prop for every variation: a `Card` that accepts any content beats a `Card` with `title`, `subtitle`, `image`, `footerText`, `showFooter`, and twenty more.
- Combine small components into larger ones, rather than building one component with many modes.
- Avoid passing props through many layers that don't use them (prop drilling). Use composition first, then context or a store if needed.

### 4. Put state in the right place

Choose the narrowest home that works:

| Kind of state | Where it belongs |
|---|---|
| Used by one component (an open dropdown, an input's text while typing) | Local component state |
| Shared by a few nearby components | Lifted to their closest common parent |
| Should survive refresh or be shareable (filters, search, tab, page) | The URL |
| Data from the server | A server-state or data-fetching layer with caching (rule 6) |
| Truly global, client-only (theme, signed-in user, feature flags) | Context or a global store |
| Form input | The form, or the project's form library |

- **Keep one source of truth.** Never store the same data in two places that must be kept in sync.
- **Derive, don't duplicate.** Compute values from existing state during rendering (a filtered list, a total, a "can submit" flag) instead of storing them separately and updating them by hand.
- **Avoid global stores for server data.** Copying server data into a global store means writing your own caching, invalidation, and loading logic; data-fetching libraries do this better.
- **Model states explicitly:** `status = "idle" | "loading" | "success" | "error"`, not `isLoading`, `isError`, `isSuccess` booleans that can contradict each other (see `software-design`).

For more depth, read `references/state-and-data.md`.

### 5. Keep rendering pure and side effects contained

- Rendering should compute what to show from props and state, with no side effects: no fetching, no subscriptions, no changing variables outside the component, no writing to storage.
- Put side effects in the framework's effect or lifecycle mechanisms, or better, in event handlers when they're caused by a user action. Many effects are unnecessary: if something can be calculated during rendering or done in an event handler, do it there.
- **Clean up** every subscription, timer, listener, and request when the component is removed or its inputs change.
- Never mutate props or shared state directly; update through the owning component or store.

### 6. Fetch server data through a proper data layer

- Use the project's data-fetching approach (such as a query library, the framework's loaders, or a shared API client with caching), rather than ad hoc requests in every component.
- Handle **every state** of every request: loading, success, empty, and error (rule 7).
- **Prevent race conditions:** when inputs change quickly (typing in a search), make sure an older, slower response can't overwrite a newer one. Cancel or ignore outdated requests. Good data libraries handle this.
- Cache and deduplicate requests, so the same data isn't fetched repeatedly by several components.
- After changing data (a mutation), update or invalidate the affected cached data, so the screen reflects the change.
- Keep API calls in a dedicated client or service layer, not scattered through components.

### 7. Design every state, not just the happy one

Every view that depends on data or user action must handle:

- **Loading:** show progress without layout jumps. Prefer skeletons or placeholders that match the final layout for content; spinners for short actions.
- **Empty:** explain that there's nothing yet, and what the user can do ("No orders yet. Browse products").
- **Error:** explain what went wrong in plain language, and offer a way forward (retry, go back, contact support). Never show raw error messages or leave a blank screen (see `error-handling`).
- **Partial:** when some data loads and some fails, show what's available.
- **Pending actions:** disable or indicate in-progress buttons, and prevent duplicate submissions.

For forms, feedback, and optimistic updates, read `references/forms-and-feedback.md`.

### 8. Contain failures with error boundaries

Use the framework's error boundary (or equivalent) so a failure in one part of the screen shows a fallback for that part, rather than a blank page. Place boundaries around independent sections (a sidebar widget, a chart, a page's main content), report caught errors to the project's error tracking, and offer a way to retry or navigate away.

### 9. Build for performance, and measure before optimising

- **Measure first,** with the browser's performance tools and the framework's profiler. Optimise what's actually slow.
- Give list items **stable, unique keys** from the data (an ID), never array indexes when items can be reordered, inserted, or removed.
- Avoid unnecessary re-rendering of large parts of the tree: keep state close to where it's used, and use the framework's memoisation tools where measurement shows a benefit, not everywhere.
- **Virtualise very long lists**, rendering only visible rows.
- **Split code** by route or large feature, so users download only what they need.
- **Optimise images:** appropriate sizes and formats, lazy loading below the fold, and explicit dimensions to prevent layout shifts.
- Watch **Core Web Vitals**: loading (Largest Contentful Paint), responsiveness (Interaction to Next Paint), and visual stability (Cumulative Layout Shift).

For more depth, read `references/performance.md`.

### 10. Style consistently with the design system

- Use the project's design system, component library, and design tokens (colours, spacing, typography, radii, shadows) rather than hard-coded values.
- Follow the project's styling approach consistently; don't introduce a second one.
- Build responsive layouts with relative units and flexible layouts that work from small phones to large screens.
- Support light and dark themes through tokens, where the project does.
- Keep styles scoped to their component, so changes don't leak elsewhere.

### 11. Keep the frontend secure

Everything in the frontend is visible to and changeable by the user. Apply the `security` skill, and in particular:

- **Never put secrets in frontend code** or public environment variables. API keys for paid or privileged services belong on the server.
- **Never rely on the frontend for authorisation.** Hiding a button is a convenience; the server must enforce every permission.
- **Never insert untrusted content as raw HTML.** Use the framework's escaping; sanitise rich text with a maintained sanitiser when unavoidable.
- **Validate user-supplied URLs** before using them in links or redirects.
- Store tokens as the project's authentication approach requires, preferring `HttpOnly` cookies over storage that any script can read.

### 12. Test components the way users use them

- Test behaviour through what the user sees and does: find elements by their role, label, or text; click, type, and submit; assert on what appears. Avoid testing internal state, implementation details, or CSS class names.
- Querying by accessible role and label also checks that components are accessible.
- Fake the network at the boundary (with the project's request-mocking tool) rather than mocking internal modules.
- Cover loading, empty, error, and success states, and key interactions.
- Use end-to-end tests only for critical user journeys (see `testing`).

For more depth, read `references/testing-ui.md`.

## Example

**Poor:**

```
component OrdersPage():
    orders = state([])
    filtered = state([])
    loading = state(false)
    error = state(false)
    search = state("")

    effect(on search change):
        loading.set(true)
        fetch("/api/orders?q=" + search).then(r => {
            orders.set(r.data)
            filtered.set(r.data.filter(o => o.status != "cancelled"))   # duplicated state
            loading.set(false)
        })                                                           # errors ignored; no race handling

    render:
        input(value=search, onChange=search.set)
        if loading: spinner()
        for o, i in filtered:
            div(key=i, onClick=() => navigate("/orders/" + o.id),    # index key; clickable div
                html=o.notes)                                         # raw HTML from data
                ...150 more lines of markup for each order...
```

Problems: one component fetches, filters, and renders everything; `filtered` duplicates `orders` and can drift; errors leave the page stuck; fast typing lets an old response overwrite a newer one; no empty state; index keys; a clickable `div` that keyboard users can't reach; and untrusted notes rendered as HTML.

**Better:**

```
# Data: one place, with caching, cancellation, and every state handled
hook useOrders(search):
    return query(key=["orders", search], fetch=signal => api.orders.list({ search }, signal))

# Page: wires data, URL state, and presentation together
component OrdersPage():
    search = urlParam("q", default="")                   # shareable, survives refresh
    result = useOrders(search)
    visibleOrders = derived(result.data?.filter(o => o.status != "cancelled") ?? [])

    render:
        SearchInput(label="Search orders", value=search, onChange=setUrlParam("q"))
        match result.status:
            "loading" → OrderListSkeleton()
            "error"   → ErrorMessage("We couldn't load your orders.", onRetry=result.retry)
            "success" → if visibleOrders.isEmpty:
                            EmptyState("No orders yet.", action=Link("Browse products", "/products"))
                        else:
                            OrderList(orders=visibleOrders)

# Presentation: receives data, renders it, no fetching
component OrderList(orders):
    render: list(for order in orders: OrderCard(key=order.id, order=order))

component OrderCard(order):
    render:
        article:
            Link(to="/orders/" + order.id): heading(order.number)    # a real link: keyboard and screen-reader friendly
            OrderStatusBadge(status=order.status)
            text(order.notes)                                        # escaped by default
```

## Common mistakes

- Very large components that fetch, transform, and render everything.
- Building a new button, modal, or input when the design system already has one.
- Storing derived values in state, or the same data in two places.
- Copying server data into a global store instead of using a data-fetching layer.
- Many contradictory boolean flags instead of one status value.
- Effects for things that could be computed during rendering or done in event handlers.
- Missing cleanup for subscriptions, timers, and listeners.
- Ignoring loading, empty, or error states; blank screens on failure.
- Race conditions where old responses overwrite newer ones.
- Array indexes as keys for dynamic lists.
- Memoising everything without measuring, or nothing where it matters.
- Hard-coded colours and spacing instead of design tokens.
- Clickable `div`s instead of buttons and links.
- Secrets in frontend code; permission checks only in the frontend; raw HTML from data.
- Tests that check internal state or class names instead of what users see.

## Reference files

| File | Read when |
|---|---|
| `references/component-design.md` | Designing component APIs, splitting or combining components, composition, or controlled and uncontrolled inputs |
| `references/state-and-data.md` | Deciding where state lives, using stores, URL state, fetching and caching server data, or mutations |
| `references/forms-and-feedback.md` | Building forms, validation, submission, loading, empty, and error states, notifications, or optimistic updates |
| `references/performance.md` | Investigating slow rendering or loading, long lists, bundle size, images, or Core Web Vitals |
| `references/testing-ui.md` | Writing component, interaction, or end-to-end tests for the interface |

## Sources

React documentation ("Thinking in React", "You Might Not Need an Effect", "Choosing the State Structure"); Vue, Svelte, and Angular documentation on component design and state; Dan Abramov, "Presentational and Container Components" (and his later note on hooks); Kent C. Dodds, Testing Library guiding principles and "Application State Management"; Brad Frost, *Atomic Design*; TanStack Query documentation (server state vs client state); web.dev guidance on Core Web Vitals; MDN Web Docs; OWASP Cross-Site Scripting Prevention Cheat Sheet.