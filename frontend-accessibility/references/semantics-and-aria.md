# Semantics and ARIA

Assistive technologies understand a page through its semantics: what each element is (its role), what it's called (its name), and what condition it's in (its state). Native HTML provides most of this automatically. ARIA fills the gaps when it can't.

## Contents

1. Name, role, and state
2. Choosing the right element
3. Accessible names
4. Headings
5. Landmarks
6. Lists and tables
7. Rules for using ARIA
8. Common ARIA attributes
9. Hiding content correctly

## 1. Name, role, and state

For every interactive element, a screen reader announces roughly: **name**, **role**, **state**. For example: "Email, edit text, required, invalid" or "Show filters, button, collapsed".

If any of the three is missing or wrong, users can't understand or operate the control.

## 2. Choosing the right element

| Need | Use | Not |
|---|---|---|
| Perform an action | `<button type="button">` | `<div onclick>`, `<a href="#">` |
| Submit a form | `<button type="submit">` | A `div` with a submit handler |
| Go to another page or location | `<a href="...">` | `<button>` that changes location, `<span onclick>` |
| Single-line text input | `<input type="text|email|tel|search|...">` | `contenteditable` div |
| Choose one of a few options | Radio buttons in a `fieldset` with a `legend` | Custom clickable cards without radio semantics |
| Choose one of many options | `<select>`, or a well-built combobox | A custom dropdown without keyboard support |
| On/off setting | `<input type="checkbox">`, or a switch built on a checkbox or button with `aria-pressed`/`role="switch"` | A styled `div` |
| Show and hide a section | `<details>` and `<summary>`, or a button with `aria-expanded` | A clickable heading with no button |
| Modal dialog | `<dialog>` opened with `showModal()`, or a library dialog following the APG | A positioned `div` |
| Data in rows and columns | `<table>` with `<th>` | A grid of `div`s |

**Buttons versus links:** if it goes somewhere (changes the URL), it's a link. If it does something on the current page, it's a button. Users and assistive technology expect different behaviour from each.

## 3. Accessible names

Ways to give an element a name, in order of preference:

1. **Visible content:** the text inside a button or link: `<button>Save changes</button>`.
2. **A connected label:** `<label for="email">Email</label><input id="email">`, or wrapping the input in the label.
3. **`aria-labelledby`:** pointing to visible text elsewhere on the page: `<section aria-labelledby="billing-heading">`.
4. **`aria-label`:** a text label not shown on screen, for icon-only controls: `<button aria-label="Close dialog">`.
5. **Visually hidden text:** text inside the element hidden with a "visually hidden" CSS class, readable by screen readers.

Guidelines:

- **Names should be concise and specific.** "Delete order #42" is better than "Delete" when there are many delete buttons; "Edit shipping address" is better than "Edit".
- **Make visible labels and accessible names match,** with the visible text at the start of the name, so voice-control users can say what they see.
- **Avoid vague link text** like "click here" or "read more". If design requires short visible text, extend the accessible name: visible "Read more", accessible "Read more about solar lamps".
- **Don't include the role in the name:** "Close", not "Close button". The role is announced separately.
- **`title` attributes** are not reliable names and don't appear for keyboard or touch users.

## 4. Headings

Screen-reader users frequently navigate by headings to understand and skim a page.

- One `h1` per page or view, describing its main content.
- Use levels to reflect structure: `h2` for main sections, `h3` for subsections within them. Don't skip levels to get a particular size; style headings with CSS instead.
- Don't use heading elements for text that isn't a heading, such as large decorative text.
- Make heading text descriptive enough to understand out of context.

## 5. Landmarks

Landmarks let users jump between major regions of the page.

| Element | Landmark |
|---|---|
| `<header>` (not inside another sectioning element) | Banner |
| `<nav>` | Navigation |
| `<main>` (one per page) | Main |
| `<aside>` | Complementary |
| `<footer>` (not inside another sectioning element) | Content information |
| `<form>` with an accessible name | Form |
| `<section>` with an accessible name | Region |
| `<search>`, or `role="search"` | Search |

- When there are several landmarks of the same type (for example, two `nav` elements), give each a distinct name: `<nav aria-label="Main">`, `<nav aria-label="Breadcrumb">`.
- Put all content inside a landmark.

## 6. Lists and tables

- Use `ul` or `ol` for lists of items, including navigation links and groups of cards. Screen readers announce the number of items, which helps users orient.
- Some CSS (such as `list-style: none`) can cause certain browsers to drop list semantics; where that matters, add `role="list"` to restore it.
- Use `table` only for tabular data, never for layout.
- Data tables need header cells (`<th>`) with `scope="col"` or `scope="row"`, and a caption or accessible name describing the table.
- For complex tables with multi-level headers, consider simplifying the table; otherwise associate cells with headers explicitly.
- Sortable column headers contain a button, and indicate the current sort with `aria-sort`.

## 7. Rules for using ARIA

Based on W3C's "Using ARIA":

1. **If a native HTML element or attribute provides the semantics and behaviour you need, use it instead of ARIA.**
2. **Don't change native semantics** unless you really must: don't put `role="button"` on a heading; put a button inside the heading.
3. **All interactive ARIA controls must be usable with the keyboard.** A `role="button"` must respond to Enter and Space and be focusable.
4. **Don't use `role="presentation"` or `aria-hidden="true"` on focusable elements.**
5. **All interactive elements must have an accessible name.**

ARIA changes only what assistive technology announces. It adds no behaviour, no focusability, and no keyboard handling. Those must be implemented separately.

## 8. Common ARIA attributes

| Attribute | Use |
|---|---|
| `aria-label`, `aria-labelledby` | Provide a name |
| `aria-describedby` | Connect additional description, such as hints or error messages |
| `aria-expanded` | Whether a controlled section is expanded (on the button that toggles it) |
| `aria-controls` | Which element a control affects |
| `aria-pressed` | Toggle button state |
| `aria-checked` | State of custom checkboxes, switches, and menu items |
| `aria-selected` | Selected item in tabs, listboxes, and grids |
| `aria-current` | Current item in a set: `page` for navigation, `step` for progress |
| `aria-invalid` | Field has a validation error |
| `aria-required` | Field is required (the native `required` attribute is usually better) |
| `aria-disabled` | Disabled but still focusable and discoverable (unlike the native `disabled` attribute) |
| `aria-live`, `role="status"`, `role="alert"` | Announce dynamic content |
| `aria-busy` | Region is updating |
| `aria-modal` | Element is a modal dialog |
| `aria-haspopup` | Control opens a menu, listbox, dialog, or similar |
| `aria-hidden` | Hide from assistive technology |

Keep every state attribute in sync with the actual state, every time it changes.

## 9. Hiding content correctly

| Goal | Technique |
|---|---|
| Hide from everyone | `display: none`, the `hidden` attribute, or `visibility: hidden` |
| Hide visually but keep for screen readers | A "visually hidden" CSS class (clip-based), never `display: none` |
| Show visually but hide from screen readers | `aria-hidden="true"` (only for decorative or duplicated content, never focusable content) |
| Make everything outside a modal inert | The native `dialog` with `showModal()`, or the `inert` attribute on background content |

A common mistake: hiding content visually with `opacity: 0` or by moving it off-screen while it remains focusable, so keyboard users tab into invisible elements.