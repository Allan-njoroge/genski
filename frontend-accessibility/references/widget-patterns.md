# Accessible Widget Patterns

Custom interactive widgets are where accessibility most often breaks. The WAI-ARIA Authoring Practices Guide (APG) defines the expected roles, states, and keyboard behaviour for common widgets. This file summarises the most common ones.

**First, prefer native elements and the design system's accessible components.** Building custom widgets correctly is hard; well-maintained, accessible component libraries save effort and mistakes. Check the APG for full details when building any pattern from scratch.

## Contents

1. Disclosure (show and hide)
2. Accordion
3. Modal dialog
4. Non-modal popovers
5. Menu button and actions menu
6. Navigation menus
7. Tabs
8. Combobox and autocomplete
9. Listbox and custom select
10. Tooltip
11. Toasts and alerts
12. Switch and toggle button
13. Sortable and interactive tables
14. Carousel
15. Pagination and breadcrumbs

## 1. Disclosure (show and hide)

- A `button` that toggles a section, with `aria-expanded="true|false"`, optionally `aria-controls` pointing to the section.
- Or use native `<details>` and `<summary>`.
- Keys: Enter and Space toggle. Focus stays on the button.

## 2. Accordion

- A set of disclosures, each with a heading containing a button: `<h3><button aria-expanded="false" aria-controls="panel-1">Shipping</button></h3>`.
- Panels can have `role="region"` with `aria-labelledby` pointing to their button, when there are only a few.
- Keys: Enter and Space toggle. Tab moves between buttons. Optional: Up and Down arrows move between headers; Home and End to first and last.

## 3. Modal dialog

- Use the native `<dialog>` element with `showModal()`, which provides much of the behaviour, or a well-tested library.
- **Name:** `aria-labelledby` pointing to the dialog's heading.
- **On open:** move focus into the dialog: to the first focusable element, or to the heading or dialog itself (with `tabindex="-1"`) if the content should be read first, or to the least destructive action in confirmation dialogs.
- **While open:** focus stays inside the dialog (Tab and Shift+Tab cycle within it); content behind it is inert and hidden from assistive technology (`showModal()` does this; otherwise use `inert` on the background).
- **Close:** with Escape, and with a visible, labelled close button.
- **On close:** return focus to the element that opened the dialog.
- Don't open dialogs automatically on page load unless essential.
- Prevent the page behind from scrolling while open.

## 4. Non-modal popovers

For content shown next to a control without blocking the rest of the page (date pickers, filter panels, information popovers):

- Triggered by a button with `aria-expanded` (and `aria-haspopup` where appropriate).
- Escape closes and returns focus to the trigger.
- Clicking outside closes, where appropriate.
- Focus moves into the popover if it contains interactive content.
- The native `popover` attribute provides some of this behaviour where supported.

## 5. Menu button and actions menu

For a button that opens a list of actions (such as "More actions" with Edit, Duplicate, Delete):

- Button: `aria-haspopup="menu"` (or `true`), `aria-expanded`, and an accessible name.
- Menu: `role="menu"`, items `role="menuitem"` (or `menuitemcheckbox`, `menuitemradio`).
- Keys: Enter, Space, or Down arrow on the button opens the menu and focuses the first item; Up arrow may focus the last item. Up and Down arrows move between items; Home and End to first and last; typing a character moves to the next item starting with it; Enter activates; Escape closes and returns focus to the button. Tab closes the menu.
- Use roving focus within the menu.

**Important:** `role="menu"` is for application-style action menus. Don't use it for website navigation; screen-reader users expect menus to behave like desktop application menus.

## 6. Navigation menus

For site navigation, including dropdown navigation:

- A `<nav>` landmark with a list of links.
- Dropdown sections are disclosures: a `button` with `aria-expanded` that shows a nested list of links.
- Mark the current page with `aria-current="page"`.
- Don't use `role="menu"` or `menuitem`; ordinary links and buttons with Tab navigation are expected.
- Make sure dropdowns open on click or Enter, not only on hover, and close with Escape.

## 7. Tabs

- Container: `role="tablist"` with an accessible name.
- Each tab: `role="tab"`, `aria-selected="true|false"`, `aria-controls` pointing to its panel.
- Each panel: `role="tabpanel"`, `aria-labelledby` pointing to its tab, `tabindex="0"` if it has no focusable content.
- Keys: Tab moves into the tab list (to the selected tab) and then into the panel. Left and Right arrows (or Up and Down for vertical tabs) move between tabs; Home and End to first and last. Tabs may activate on focus (automatic) or on Enter and Space (manual); use manual activation if loading a panel is slow.
- Use roving focus.

If "tabs" actually navigate to different URLs, they're navigation links: use a `nav` with links and `aria-current`, not the tabs pattern.

## 8. Combobox and autocomplete

One of the most complex patterns; prefer a well-tested library component.

- Input: `role="combobox"`, `aria-expanded`, `aria-controls` pointing to the list, `aria-autocomplete` (`list` or `both`), and `aria-activedescendant` pointing to the highlighted option.
- List: `role="listbox"`, options `role="option"` with `aria-selected`.
- Keys: typing filters; Down arrow opens the list or moves to the next option; Up arrow moves back; Enter selects; Escape closes the list (and may clear the input); focus stays in the input throughout.
- Announce the number of results with a status region: "8 results available".
- Make the input's label visible and connected.

Consider whether a native `<select>` or a simple input with a `datalist` would meet the need.

## 9. Listbox and custom select

- Prefer the native `<select>`. It's accessible, works well on mobile, and needs no custom code. Recent browsers increasingly allow styling it.
- If a custom select is required: a button with `aria-haspopup="listbox"` and `aria-expanded` opening a `role="listbox"` with `role="option"` items; Up and Down arrows move; Enter selects and closes; Escape closes; type-ahead moves to matching options.

## 10. Tooltip

- Tooltips are brief text that describes a control, shown on hover and on keyboard focus.
- Element: `role="tooltip"`, connected to its trigger with `aria-describedby`.
- Shown on both hover and focus; hidden with Escape; stays visible while the pointer is over the tooltip itself.
- Never put essential information or interactive content (links, buttons) in a tooltip. Use a popover or visible text instead.
- Touch users can't hover, so tooltips must not be the only way to get information.
- Icon-only buttons still need an accessible name; a tooltip alone doesn't reliably provide it.

## 11. Toasts and alerts

- Use a pre-existing live region: `role="status"` for routine messages, `role="alert"` for urgent ones (see `forms-and-dynamic-content.md`).
- Don't move focus to the toast.
- Allow enough time to read, pause on hover and focus, and provide a labelled close button.
- Don't rely on toasts for errors that require action; show those in context.

## 12. Switch and toggle button

- **Toggle button:** a `button` with `aria-pressed="true|false"`, for things like "Mute" or "Bold". The label stays the same; the state changes.
- **Switch:** `role="switch"` with `aria-checked="true|false"`, or a native checkbox styled as a switch, for on/off settings that take effect immediately.
- Keys: Space toggles (and Enter for buttons).
- Don't change the label to describe the next action and also set a pressed state; that announces contradictory information.

## 13. Sortable and interactive tables

- Use a real `<table>` with header cells.
- Sortable column headers contain a `button`; the header has `aria-sort="ascending|descending"` on the currently sorted column (and none on others).
- Announce sorting changes through a status region where helpful.
- Row actions: use buttons with specific names ("Edit order #42").
- Selectable rows: checkboxes with labels that identify the row.
- For spreadsheet-like editing with arrow-key navigation between cells, use the APG grid pattern (`role="grid"`), which is complex. Only use it when that interaction is genuinely needed.
- Wide tables can scroll horizontally within a focusable, labelled container.

## 14. Carousel

Carousels are frequently inaccessible and often ineffective. Consider simpler alternatives first, such as a static list or grid.

If needed:

- Provide visible, labelled previous and next buttons, and slide pickers if useful.
- Don't auto-rotate; if it must, provide a pause button, and stop rotation on hover and keyboard focus.
- Each slide is a group with a label: "Slide 2 of 5".
- Hide non-visible slides from assistive technology and the tab order.
- Respect reduced-motion preferences.

## 15. Pagination and breadcrumbs

**Pagination:** a `nav` labelled "Pagination", a list of links, the current page marked with `aria-current="page"`, and descriptive names for previous and next ("Previous page", "Next page"). Page number links can be named "Page 3".

**Breadcrumbs:** a `nav` labelled "Breadcrumb", an ordered list of links, with the current page marked `aria-current="page"`. Hide visual separators from assistive technology.