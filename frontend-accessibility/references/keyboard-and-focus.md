# Keyboard and Focus

Many people navigate entirely by keyboard or by tools that act like a keyboard: switch devices, voice control, and screen readers. If something can't be done with a keyboard, these users can't do it at all.

## Contents

1. Expected keys
2. Focus order
3. Making elements focusable
4. Visible focus
5. Focus not obscured
6. Managing focus during changes
7. Skip links
8. Roving focus in composite widgets
9. Keyboard shortcuts

## 1. Expected keys

| Key | Expected behaviour |
|---|---|
| Tab / Shift+Tab | Move to the next / previous focusable element |
| Enter | Activate links and buttons; submit forms |
| Space | Activate buttons; toggle checkboxes; select options |
| Arrow keys | Move within composite widgets: radio groups, menus, tabs, listboxes, sliders, grids |
| Escape | Close dialogs, menus, popovers, and tooltips |
| Home / End | Move to the first / last item in lists, menus, and similar widgets |

Native elements provide these behaviours automatically. Custom widgets must implement them, following the WAI-ARIA Authoring Practices Guide.

## 2. Focus order

- Focus moves in an order that matches the visual layout and reading order: generally top to bottom, left to right (for left-to-right languages).
- Keep the DOM order matching the visual order. CSS that rearranges content visually (`order`, absolute positioning, grid placement) can make focus jump around confusingly.
- **Never use positive `tabindex` values** (`tabindex="1"` and higher). They override the natural order and create confusing sequences across the whole page.

## 3. Making elements focusable

| `tabindex` value | Effect | Use |
|---|---|---|
| (none, on native interactive elements) | Focusable in natural order | Buttons, links, form controls: the default |
| `0` | Adds a non-interactive element to the natural tab order | Custom widgets with appropriate roles and keyboard handling |
| `-1` | Focusable by script only, not by Tab | Elements you move focus to programmatically: dialog containers, headings after navigation, items in roving focus |
| Positive | Overrides order | Never |

Don't make non-interactive content (paragraphs, headings, images) focusable for no reason; it adds useless tab stops. Screen-reader users have other ways to read content.

Disabled controls: the native `disabled` attribute removes them from the tab order, so users may not discover them. Where users need to find a disabled control and understand why it's disabled, consider `aria-disabled="true"` with an explanation instead, and block activation in code.

## 4. Visible focus

- Every focusable element has a clearly visible focus indicator when focused by keyboard.
- **Never use `outline: none` (or `outline: 0`) without an equally visible replacement.**
- Use `:focus-visible` to show focus rings for keyboard users without showing them on every mouse click.
- Make indicators strong: a solid outline of at least 2 pixels, with good contrast (at least 3:1) against both the element and its surroundings. An offset helps it stand out.
- Check focus visibility on every background colour, in every theme, and on every component state.

```css
:focus-visible {
  outline: 3px solid var(--color-focus);
  outline-offset: 2px;
}
```

## 5. Focus not obscured

When an element receives focus, it must not be completely hidden by other content, such as sticky headers, cookie banners, chat widgets, or footers (WCAG 2.2).

- Use `scroll-padding-top` (and bottom) equal to the height of sticky elements, so focused items scroll into a visible position.
- Make sure non-modal overlays don't cover focused content, or can be dismissed.

## 6. Managing focus during changes

When the interface changes, decide where focus should go. Focus left on a removed element falls back to the top of the page, disorienting keyboard and screen-reader users.

| Change | Where focus goes |
|---|---|
| Modal dialog opens | Into the dialog: its first focusable element, or its heading or container (with `tabindex="-1"`) when the content should be read first |
| Modal dialog closes | Back to the element that opened it (or a logical alternative if that element no longer exists) |
| Navigation to a new view in a single-page app | The new view's main heading (with `tabindex="-1"`), or the main container; or announce the new page title |
| Item deleted from a list | The next item, the previous item, or the list's heading |
| Inline edit saved or cancelled | Back to the control that started editing |
| Form submitted with errors | The error summary, or the first invalid field |
| Content expanded (accordion, disclosure) | Stays on the toggle button |
| Menu or popover closed with Escape | Back to the button that opened it |

Don't move focus unexpectedly in response to ordinary input (for example, jumping to the next field when one is filled), unless the user has been told it will happen.

## 7. Skip links

Pages with repeated navigation before the main content should start with a skip link:

```html
<a class="skip-link" href="#main">Skip to main content</a>
…
<main id="main" tabindex="-1">…</main>
```

- It's the first focusable element on the page.
- It may be visually hidden until focused, but must become visible when focused.
- Its target receives focus when activated.

Proper landmarks and headings also help screen-reader users skip content, but skip links serve keyboard users who don't use screen readers.

## 8. Roving focus in composite widgets

In widgets made of many items (tabs, toolbars, menus, radio groups, listboxes, grids), Tab moves into and out of the widget as a single stop, and arrow keys move between items. This avoids forcing users to tab through every item.

Implement with **roving `tabindex`**: the active item has `tabindex="0"`, all others `tabindex="-1"`; arrow keys move `tabindex="0"` and focus to the next item. Alternatively, use `aria-activedescendant` on the container, where the pattern calls for it.

Follow the specific pattern in the WAI-ARIA Authoring Practices Guide for each widget (see `widget-patterns.md`).

## 9. Keyboard shortcuts

- If you add single-character keyboard shortcuts (a letter or number without modifier keys), users must be able to turn them off or remap them, or they must only work when a specific component has focus. Otherwise they conflict with voice control and screen-reader commands.
- Don't override standard browser or screen-reader shortcuts.
- Document available shortcuts.