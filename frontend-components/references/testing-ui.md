# Testing User Interfaces

Interface tests should give confidence that users can actually do what they need to. The most useful principle, from Testing Library: the more your tests resemble the way your software is used, the more confidence they can give you. Apply the core `testing` skill as well; this file covers what's specific to interfaces.

## Contents

1. What to test at each level
2. Querying like a user
3. Interactions
4. Asynchronous behaviour
5. Mocking the network
6. Testing states
7. What not to test
8. End-to-end tests
9. Visual and accessibility checks

## 1. What to test at each level

| Level | Use for | Typical tools |
|---|---|---|
| **Unit** | Pure logic: formatting, calculations, validation, reducers, state machines | The project's unit test runner |
| **Component / integration** | A component or feature rendered with its children, interacting with it as a user, with the network faked | Testing Library (or the framework's equivalent) with the project's runner |
| **End-to-end** | Critical journeys through the real application in a real browser | A browser automation tool such as Playwright or Cypress |

Most interface tests should be component or integration tests: realistic enough to give confidence, fast enough to run constantly.

## 2. Querying like a user

Find elements the way users and assistive technology do, in this order of preference:

1. **By role and accessible name:** `getByRole("button", { name: "Place order" })`, `getByRole("textbox", { name: "Email" })`.
2. **By label:** `getByLabelText("Email")`.
3. **By visible text:** `getByText("No orders yet")`.
4. **By test ID** (`data-testid`), only when there's no accessible way to identify the element.

Avoid querying by CSS class, element structure, or component internals. Those break when styling or structure changes, even though users see no difference.

Querying by role and label has a bonus: if the test can't find a button by its role and name, a screen-reader user probably can't either.

## 3. Interactions

- Simulate real user interactions with the testing library's user-event utilities (typing, clicking, tabbing, selecting), rather than calling handlers directly or firing low-level events.
- Test keyboard interaction for interactive components: tabbing to elements, activating with Enter and Space, closing with Escape.
- Assert on the outcome the user sees: new text, a changed value, a message, navigation.

```
test "shows an error when the email is invalid":
    render(SignUpForm())
    user.type(getByRole("textbox", { name: "Email" }), "not-an-email")
    user.click(getByRole("button", { name: "Create account" }))
    expect(getByText("Enter a valid email address")).toBeVisible()
```

## 4. Asynchronous behaviour

- Wait for what the user would wait for: use the library's "find" or "wait for" utilities to wait until expected content appears.
- Never use fixed delays (sleeps) to wait.
- Make sure every test waits for pending updates to finish, so state updates don't leak into the next test.
- Use fake timers for debounce, throttle, polling, and timeouts.

## 5. Mocking the network

- Fake responses at the network level, using a request-mocking tool (such as Mock Service Worker), rather than mocking the API client module or the data-fetching library. The component, data layer, and API client then all run as in production.
- Define realistic responses, including errors, empty results, and slow responses.
- Reuse the same mock handlers for tests, local development, and component previews where the project does.
- Never call real backends or third-party services from component tests.

## 6. Testing states

For each data-driven view, test:

- **Loading:** a loading indicator or skeleton appears.
- **Success:** the data is shown correctly.
- **Empty:** the empty state message and action appear.
- **Error:** an understandable error and a retry option appear; retrying works.
- **Permissions:** actions the user can't perform are hidden or disabled, where relevant (remembering the server must enforce them too).

For forms, test: validation messages, successful submission, server errors shown next to fields, prevention of double submission, and that input is kept after a failure.

## 7. What not to test

- Internal component state or private functions.
- That a specific child component was rendered (test what it displays instead).
- CSS class names or exact markup structure.
- The framework or third-party libraries themselves.
- Large snapshot tests of rendered output; they break on every change, and reviewers approve updates without reading them. Small, focused snapshots of stable output can be acceptable.

## 8. End-to-end tests

- Reserve them for a handful of critical journeys: sign up, sign in, the core purchase or creation flow.
- Use role- and label-based selectors here too.
- Seed test data through APIs or fixtures, not through the interface, to keep tests fast and focused.
- Make them independent: each test creates what it needs and doesn't rely on other tests.
- Avoid fixed waits; use the tool's automatic waiting.
- Run them against an environment with controlled data, never production.

## 9. Visual and accessibility checks

- **Automated accessibility checks** (such as axe) in component and end-to-end tests catch many common problems: missing labels, poor contrast, invalid ARIA. They catch only part of the problems, so manual checks are still needed (see `frontend-accessibility`).
- **Visual regression tests** (comparing screenshots) can catch unintended visual changes in design system components, where the project uses them. Keep them focused to avoid noise.
- **Component previews** (such as a component workshop) that show each state of a component are useful documentation and can double as test fixtures.