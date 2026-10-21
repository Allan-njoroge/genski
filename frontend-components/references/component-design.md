# Component Design

This file covers how to decide what a component is, how to shape its API, and how components fit together.

## Contents

1. Kinds of components
2. Finding component boundaries
3. Props and APIs
4. Composition patterns
5. Controlled and uncontrolled components
6. Reusable versus feature components
7. Folder structure
8. Hooks, composables, and shared logic

## 1. Kinds of components

Most interfaces have a few layers. Names vary by project; the roles are what matter.

| Kind | Responsibility | Example |
|---|---|---|
| **Primitives / design system** | Generic, styled building blocks with no business knowledge | `Button`, `TextField`, `Modal`, `Table` |
| **Presentational / feature components** | Render a business concept from props; no data fetching | `OrderCard`, `PriceTag`, `InvoiceSummary` |
| **Containers / connected components** | Fetch data, manage state, connect presentational components | `OrderListContainer`, or a hook used by a page |
| **Pages / routes / views** | Compose a full screen for a URL; handle URL state and layout | `OrdersPage` |
| **Layouts** | Shared page structure: navigation, header, sidebar | `AppLayout` |

Not every feature needs every layer. A small page can fetch and render directly. Introduce the separation when it makes components easier to reuse, preview, or test.

## 2. Finding component boundaries

Split a component when:

- Part of it is reused elsewhere, or clearly could be.
- Part of it changes for a different reason, such as data loading versus layout.
- It's hard to name without "and" ("OrderListAndFiltersAndPagination").
- A section has its own state that the rest doesn't care about.
- It has grown hard to read as one unit.

Keep code together when:

- The pieces only make sense together and share a lot of state.
- Splitting would create components that just pass props along.
- The result would be many tiny components a reader must jump between.

A useful starting point (from "Thinking in React"): sketch the screen, draw boxes around each part with a single responsibility, and name each box. Those boxes are candidate components.

## 3. Props and APIs

**Make the common case easy:**

```
Button("Save")                                   # sensible defaults
Button("Delete", variant="danger", onClick=...)  # explicit when it matters
```

**Prefer one meaningful prop over conflicting booleans:**

```
Avoid:   Alert(isSuccess, isWarning, isError)          # what if two are true?
Prefer:  Alert(tone="success" | "warning" | "error")
```

**Name events after what happened:** `onSelect(item)`, `onChange(value)`, `onClose()`. Let the parent decide what to do; the component reports, it doesn't decide.

**Pass the data the component needs, not more:** `Avatar(name, imageUrl)` rather than `Avatar(user)` with the entire user object, when it only needs two fields. This makes the component reusable and its dependencies clear. (For domain components such as `OrderCard(order)`, passing the domain object is fine.)

**Avoid props that mirror internal implementation:** `showSpinnerInsteadOfIcon` exposes how the component works; `isLoading` describes what's true.

**Forward standard attributes** to the underlying element where appropriate (such as `id`, `aria-*`, `className`), so reusable components remain flexible and accessible.

**Type the API** with the project's type system or prop validation, and document non-obvious props.

## 4. Composition patterns

**Children and slots:** let callers provide content instead of configuring it through props.

```
Avoid:
    Card(title="Order #42", subtitle="Shipped", imageUrl=..., footerText="View", footerLink=..., showFooter=true)

Prefer:
    Card:
        CardHeader: heading("Order #42"), Badge("Shipped")
        CardBody: ...
        CardFooter: Link("View", to=...)
```

**Compound components:** a set of components that work together and share state implicitly, such as `Tabs`, `TabList`, `Tab`, and `TabPanel`. They give callers flexibility in arrangement while the parent manages coordination.

**Render props, scoped slots, or callbacks for rendering:** let the caller decide how part of the component renders, such as how each row in a generic list appears.

**Avoiding prop drilling:**

1. First, try composition: pass components rather than data through intermediate layers.
2. If many distant components need the same value, use context, dependency injection, or a store.
3. Don't reach for global state just to avoid passing a prop through two levels.

## 5. Controlled and uncontrolled components

- **Controlled:** the parent owns the value and passes it in, with a change callback. The parent always knows the current value, and can validate, transform, or reset it.
- **Uncontrolled:** the component keeps its own value internally; the parent reads it when needed (for example, on submit).

Guidelines:

- Reusable inputs should support controlled use, and ideally both.
- Don't switch a component between controlled and uncontrolled during its life.
- Don't copy a prop into local state and then try to keep them in sync. Either the parent controls the value, or the component owns it with the prop as an initial value only (and named so: `defaultValue`, `initialValue`).

## 6. Reusable versus feature components

- **Don't make a component generic before it's needed twice.** Build it for the feature first. When a second use appears, extract what's genuinely shared (see `software-design`, rule of three).
- **Reusable components know nothing about the business.** A `DataTable` doesn't know about orders; an `OrdersTable` uses `DataTable` and knows about orders.
- **Prefer the design system's components** to building new ones. If a design system component is missing something, extend it there, or discuss with whoever owns it, rather than building a parallel version.

## 7. Folder structure

Follow the project's structure. A common, scalable approach groups by feature:

```
src/
├── features/
│   ├── orders/
│   │   ├── components/      OrderList, OrderCard, OrderStatusBadge
│   │   ├── api/             orders API client functions
│   │   ├── hooks/           useOrders, useCancelOrder
│   │   ├── pages/           OrdersPage, OrderDetailPage
│   │   └── index            public exports for other features
│   └── checkout/
├── shared/
│   ├── components/          generic UI not in the design system
│   ├── hooks/
│   └── lib/                 API client setup, formatting, utilities
└── app/                     routing, providers, layout
```

The module boundary rules from `software-design` apply: features import other features only through their public exports.

## 8. Hooks, composables, and shared logic

Extract reusable stateful logic into the framework's mechanism (hooks, composables, services, stores) rather than duplicating it across components.

- Name them for what they provide: `useOrders`, `useDebouncedValue`, `useMediaQuery`.
- Keep them focused; a hook that fetches, transforms, and manages unrelated UI state is as hard to work with as a giant component.
- Put pure logic (formatting, calculations, validation) in plain functions, not hooks. They're simpler to test and reuse.