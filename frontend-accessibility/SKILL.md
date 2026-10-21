---
name: frontend-accessibility
description: Principles for building web interfaces that everyone can use, including people who navigate by keyboard, use screen readers or voice control, zoom or magnify, have low vision or colour blindness, or are sensitive to motion, following WCAG 2.2 level AA in any framework. Covers semantic HTML, accessible names and labels, keyboard access, focus management, headings and landmarks, ARIA, forms and error messages, images and media, colour contrast, text resizing and reflow, touch target sizes, motion, announcing dynamic content, accessible widget patterns (dialogs, menus, tabs, comboboxes), and testing with tools, keyboards, and screen readers. Use this skill for every UI component, page, form, or interactive element, and whenever reviewing or fixing interface code, even if the task does not mention accessibility.
---

# Frontend Accessibility

Accessibility means people can use what you build regardless of how they perceive or interact with it: with a screen reader, only a keyboard, voice control, a magnifier, high zoom on a small phone, colour blindness, a tremor, or sensitivity to motion. Many people have a disability, and everyone benefits from accessible design at some point: in bright sunlight, with a broken trackpad, holding a baby in one arm, or simply getting older.

Accessibility is far cheaper to build in than to add later. Most of it comes from doing the basics correctly: using the right HTML elements, labelling things, supporting the keyboard, and checking contrast. It's also a legal requirement in many places.

The target is **WCAG 2.2 Level AA** (Web Content Accessibility Guidelines), the standard most laws and policies refer to, unless the project specifies otherwise.

Read the project's `AGENTS.md` first. Use the project's design system, which may already provide accessible components; prefer them to building new ones. Where the project's conventions differ from this skill in ways that reduce accessibility, follow the project but point out the issue. Apply alongside `frontend-components` for every component.

## Before you start

1. **Is there a native HTML element for this?** A button, link, input, select, checkbox, dialog, or details element already has keyboard support and meaning built in.
2. **Can it be done with the keyboard alone?** Every action, in a sensible order, with visible focus.
3. **What will a screen reader announce?** Every control needs a name, a role, and its current state.
4. **Does it rely only on sight, colour, or a mouse?** Provide an alternative for each.
5. **What changes dynamically?** Content that appears, updates, or moves focus needs to be announced or managed.

## Core rules

### 1. Use semantic HTML first

The right element gives you behaviour, keyboard support, and meaning for free.

- `button` for actions; `a` with `href` for navigation. Never a clickable `div` or `span`.
- Real form controls: `input`, `select`, `textarea`, `input type="checkbox"`, `input type="radio"`.
- Headings (`h1` to `h6`) in a logical outline, not chosen for their size.
- Lists (`ul`, `ol`) for lists, `table` with headers for tabular data.
- Landmarks for page regions: `header`, `nav`, `main`, `aside`, `footer`, and `form` or `section` with labels where useful.
- Native `dialog` and `details`/`summary` where they fit.

If you find yourself adding `role="button"`, `tabindex`, and key handlers to a `div`, use a `button` instead.

### 2. Give everything an accessible name

Every interactive element and meaningful image must have a name that assistive technology can announce.

- Buttons and links: visible text that describes the action or destination: "Download invoice", not "Click here" or "Read more" on its own.
- Icon-only buttons: an accessible label, such as visually hidden text or `aria-label`: a trash icon button is named "Delete order".
- Form fields: a visible `label` connected to the input.
- Images: meaningful `alt` text describing the content or purpose; `alt=""` for purely decorative images, so they're skipped.
- The name should match the visible label, so voice control users can say what they see.

### 3. Make everything work with a keyboard

- Every interactive element can be reached with Tab and operated with the expected keys: Enter for links and buttons, Space for buttons and checkboxes, arrow keys within composite widgets such as menus, tabs, and radio groups, Escape to close dialogs and menus.
- Focus order follows the visual and reading order. Don't use positive `tabindex` values to rearrange it.
- No keyboard traps: users can always move focus away, except in modal dialogs, where focus is deliberately contained until the dialog closes.
- Provide a "skip to main content" link at the start of pages with repeated navigation.
- Custom widgets follow the established keyboard patterns from the WAI-ARIA Authoring Practices Guide (see `references/widget-patterns.md`).

### 4. Keep focus visible and managed

- **Never remove focus outlines** without providing an equally visible replacement. Use `:focus-visible` to show focus styles for keyboard users without affecting mouse clicks.
- Make focus indicators clearly visible against every background they appear on.
- Make sure focused elements aren't hidden behind sticky headers, footers, or banners.
- **Manage focus deliberately** when the interface changes:
  - Opening a dialog moves focus into it; closing it returns focus to the element that opened it.
  - After navigating to a new view in a single-page application, move focus to the new content (often its main heading) or announce the change.
  - When content the user was focused on is removed, move focus somewhere sensible, never let it fall to the top of the page.

For more depth, read `references/keyboard-and-focus.md`.

### 5. Structure pages clearly

- Each page has a unique, descriptive `title` that changes with the view, including in single-page applications.
- One `h1` describing the page's main content, and headings that don't skip levels for visual effect.
- Landmarks for the main regions, so screen-reader users can jump between them.
- The page's language is set (`lang` on the `html` element), and passages in another language are marked, so screen readers pronounce them correctly.

### 6. Use ARIA only when HTML can't do the job

ARIA (Accessible Rich Internet Applications) attributes add roles, names, and states for assistive technology. They change only what's announced, never behaviour.

- **No ARIA is better than bad ARIA.** Incorrect roles and states actively mislead screen-reader users.
- Use native elements instead of ARIA roles wherever possible.
- When building custom widgets, use the roles, states, and properties the WAI-ARIA Authoring Practices Guide specifies, and implement all the keyboard behaviour the role promises.
- Keep states in sync with reality: `aria-expanded`, `aria-selected`, `aria-checked`, `aria-pressed`, `aria-current`, `aria-invalid`.
- Never put `aria-hidden="true"` on focusable elements or on content users need.

For more depth, read `references/semantics-and-aria.md`.

### 7. Make forms understandable and forgiving

- Every field has a visible, connected label. Placeholders don't replace labels.
- Instructions and format requirements appear before or with the field, not only after an error.
- Required fields are indicated in text or with an accessible indicator, not by colour alone.
- Errors are described in text, next to the field, connected to it (for example, with `aria-describedby`), and the field marked invalid. On submit, move focus to the first error or an error summary.
- Use `autocomplete` attributes for personal data (name, email, phone, address), so browsers and password managers can fill them.
- Don't make people re-enter information they've already given in the same process, and don't block pasting into fields, including passwords.
- Authentication must not rely on solving puzzles or remembering information alone; support password managers and copy-paste, and offer alternatives to cognitive tests.

### 8. Announce dynamic changes

When content changes without a page load, screen-reader users must learn about it.

- Use **live regions** (`aria-live="polite"`, or `role="status"`) for status messages: "Saved", "3 results found", "Item added to basket". Use `role="alert"` (assertive) only for urgent messages such as errors that need immediate attention.
- The live region must exist on the page before content is inserted into it.
- Don't announce every small change; announce what the user needs to know.
- Notifications that disappear must stay long enough to read and must not be the only way important information is given.

For forms and dynamic content, read `references/forms-and-dynamic-content.md`.

### 9. Don't rely on sight or colour alone

- **Text contrast:** at least 4.5:1 against its background for normal text, 3:1 for large text (about 24 px, or 19 px bold).
- **Non-text contrast:** at least 3:1 for interface components' boundaries and states (input borders, focus indicators, icons that convey meaning, chart lines).
- **Never use colour as the only signal:** errors need text or an icon as well as red; links in text need an underline or other non-colour cue; charts need labels or patterns as well as colours.
- Check contrast in every theme (light, dark, high contrast) and every state (hover, focus, disabled, error).

### 10. Support zoom, reflow, and different devices

- Text can be resized to 200% without loss of content or function.
- Content reflows to a single column at 320 CSS pixels wide (equivalent to 400% zoom on a desktop) without horizontal scrolling, except for content that genuinely needs two dimensions, such as maps and data tables.
- Use relative units for text, and don't set fixed heights on text containers.
- Don't disable zooming in the viewport settings.
- Users can override line height, letter spacing, and word spacing without breaking the layout.
- Content works in both portrait and landscape orientation.

### 11. Make targets easy to hit

- Interactive targets are at least 24 by 24 CSS pixels, or spaced so that a 24-pixel circle around each doesn't overlap another (WCAG 2.2 minimum). Aim for about 44 by 44 for primary controls on touch screens.
- Functions that use complex gestures (pinch, multi-finger swipes, dragging) also have a simple alternative, such as buttons; dragging always has a single-pointer alternative.
- Actions trigger on release (click), not on press, so users can cancel by moving away.

### 12. Respect motion and time

- Respect the user's reduced-motion setting (`prefers-reduced-motion`): reduce or remove non-essential animation, parallax, and auto-playing motion.
- Nothing flashes more than three times per second.
- Moving, auto-updating, or auto-playing content that lasts more than five seconds can be paused, stopped, or hidden.
- Time limits can be turned off, adjusted, or extended, unless they're essential (such as an auction).
- Media doesn't play sound automatically, or offers an immediate way to stop it.

For colour, text, motion, and targets, read `references/visual-and-motion.md`.

### 13. Provide alternatives for images and media

- Informative images have `alt` text conveying the same information. Complex images (charts, diagrams) have a fuller description nearby or a data table.
- Decorative images have empty `alt=""` or are added through CSS.
- Images of text are avoided; use real text.
- Videos have captions; audio has transcripts; videos with important visual information have audio description or a text alternative.
- Icons that convey meaning have text alternatives; icons beside visible text are hidden from assistive technology to avoid repetition.

### 14. Test with tools, keyboards, and screen readers

Automated tools catch only part of accessibility problems. Use all of these:

- **Automated checks** (such as axe) in development and in component and end-to-end tests.
- **Keyboard testing:** unplug the mouse and complete every task with Tab, Shift+Tab, Enter, Space, arrows, and Escape. Check focus is always visible and logical.
- **Screen reader testing** for new components and key flows, with at least one common combination (such as NVDA or JAWS with a browser on Windows, VoiceOver on macOS or iOS, or TalkBack on Android).
- **Zoom and reflow:** 200% text size and a 320-pixel-wide viewport.
- **Contrast checks** for new colours and states.
- Query elements by role and label in component tests (see `frontend-components`), which also verifies names and roles.

For more depth, read `references/testing-accessibility.md`.

### 15. Never fake or remove accessibility to make something work

- Don't add `role`, `tabindex`, or `aria-*` attributes to silence a linter without implementing the behaviour they promise.
- Don't remove focus outlines, labels, or alt text for visual reasons; find a design that keeps them.
- Don't suppress accessibility linter or test failures without fixing the cause. If a fix isn't possible within the task, tell the person you're working with.

## Before finishing

- [ ] Native elements used for buttons, links, and form controls.
- [ ] Every interactive element and meaningful image has an accessible name.
- [ ] Everything works with the keyboard alone, in a logical order, with visible focus.
- [ ] Focus is managed for dialogs, view changes, and removed content.
- [ ] Headings, landmarks, page title, and language are correct.
- [ ] Form fields have labels; errors are in text, connected to fields, and announced.
- [ ] Status messages and dynamic updates are announced.
- [ ] Contrast meets 4.5:1 for text and 3:1 for interface elements; colour is never the only signal.
- [ ] Works at 200% text size and at 320 pixels wide.
- [ ] Targets are at least 24 by 24 pixels; gestures and dragging have simple alternatives.
- [ ] Reduced-motion preference respected; nothing flashes.
- [ ] Automated checks pass, and keyboard testing done.

## Example

**Poor:**

```
<div class="card" onclick="openOrder(42)">
  <img src="box.png">
  <span class="title">Order #42</span>
  <span style="color:red">●</span>
</div>
<div class="btn" onclick="deleteOrder(42)"><svg>…trash icon…</svg></div>
<input placeholder="Search orders">
<style> *:focus { outline: none; } </style>
```

Keyboard users can't reach or activate the card or the delete control. Screen readers announce nothing meaningful: the image has no alt text, the delete control has no name or role, and the red dot's meaning (cancelled?) is conveyed only by colour. The search box has no label once the user starts typing. Focus outlines are removed for everyone.

**Better:**

```
<article class="card">
  <h3><a href="/orders/42">Order #42</a></h3>
  <img src="box.png" alt="">                          <!-- decorative: the heading says what it is -->
  <p class="status status--cancelled">
    <svg aria-hidden="true">…icon…</svg> Cancelled     <!-- text, not only colour -->
  </p>
  <button type="button" onclick="deleteOrder(42)" aria-label="Delete order #42">
    <svg aria-hidden="true" focusable="false">…trash icon…</svg>
  </button>
</article>

<label for="order-search">Search orders</label>
<input id="order-search" type="search">

<style>
  :focus-visible { outline: 3px solid var(--focus-color); outline-offset: 2px; }
</style>
```

## Common mistakes

- Clickable `div`s and `span`s instead of buttons and links.
- Icon-only buttons with no accessible name.
- Images without `alt`, or decorative images with descriptive `alt` that adds noise.
- Placeholders used instead of labels.
- Focus outlines removed with no replacement.
- Dialogs that don't move focus in, trap it, close on Escape, or return focus.
- Headings chosen for size rather than structure; missing page titles in single-page apps.
- Colour as the only way to show errors, status, or links.
- Low-contrast text, especially placeholder and disabled text, or light grey on white.
- Status updates and errors that screen readers never announce.
- ARIA roles added without the matching keyboard behaviour; `aria-hidden` on focusable content.
- Tiny touch targets; drag-only interactions.
- Disabled zoom; layouts that break at 200% text or on narrow screens.
- Relying on automated checks alone.

## Reference files

| File | Read when |
|---|---|
| `references/semantics-and-aria.md` | Choosing elements, naming controls, using landmarks and headings, or using ARIA |
| `references/keyboard-and-focus.md` | Implementing keyboard support, focus styles, focus management, or skip links |
| `references/forms-and-dynamic-content.md` | Building forms, error handling, live regions, notifications, or single-page navigation |
| `references/visual-and-motion.md` | Working on colour, contrast, typography, zoom, layout, target sizes, animation, or media |
| `references/widget-patterns.md` | Building dialogs, menus, tabs, accordions, comboboxes, tooltips, toasts, tables, or carousels |
| `references/testing-accessibility.md` | Testing or auditing accessibility, or fixing reported accessibility problems |

## Sources

W3C, *Web Content Accessibility Guidelines (WCAG) 2.2* and *Understanding WCAG 2.2*; W3C, *WAI-ARIA 1.2*; W3C WAI, *ARIA Authoring Practices Guide (APG)*; W3C, "Using ARIA" (including the first rule of ARIA use); WebAIM articles and the WebAIM Million report; MDN Web Docs accessibility guides; GOV.UK Design System and GOV.UK accessibility guidance; Deque, axe-core documentation; Inclusive Components (Heydon Pickering).