# Forms and Dynamic Content

Forms are where accessibility failures most often stop people completing a task. Dynamic content, which changes without a page load, is where screen-reader users most often miss information. This file covers both.

## Contents

1. Labels
2. Instructions and hints
3. Grouping related fields
4. Required fields
5. Error messages
6. Error summaries
7. Autocomplete and input help
8. Accessible authentication
9. Live regions
10. Notifications and toasts
11. Loading and busy states
12. Single-page application navigation

## 1. Labels

- Every input has a visible label, connected programmatically:

```html
<label for="phone">Phone number</label>
<input id="phone" type="tel" autocomplete="tel">
```

- Placeholders are not labels: they disappear when typing, often have poor contrast, and aren't reliably announced.
- Keep labels close to their fields and visible at all times. "Floating" labels that shrink into the field must stay readable and meet contrast.
- Labels for checkboxes and radio buttons should be clickable (connected with `for` or by wrapping), which also enlarges the target.

## 2. Instructions and hints

- Give format requirements before the user types: "Date of birth, for example 27 3 1990", "Password must be at least 12 characters".
- Connect hints to fields with `aria-describedby`, so they're announced with the field:

```html
<label for="mpesa">M-Pesa number</label>
<p id="mpesa-hint">The number registered for M-Pesa, starting with 07 or 01</p>
<input id="mpesa" type="tel" aria-describedby="mpesa-hint">
```

- Keep instructions short and in plain language.

## 3. Grouping related fields

- Group related controls, especially radio buttons and checkboxes, in a `fieldset` with a `legend` that states the question:

```html
<fieldset>
  <legend>Delivery method</legend>
  <input type="radio" id="pickup" name="delivery" value="pickup"><label for="pickup">Pick up</label>
  <input type="radio" id="courier" name="delivery" value="courier"><label for="courier">Courier</label>
</fieldset>
```

- Group multi-part inputs (such as a date in three fields) the same way.

## 4. Required fields

- Indicate required (or optional) fields in text, not only with colour or an unexplained asterisk. If using an asterisk, explain it at the start of the form.
- Use the native `required` attribute, or `aria-required="true"` for custom controls.
- Consider marking only optional fields "(optional)" when most fields are required.

## 5. Error messages

When a field has an error:

- Show the error as text near the field, visible and connected with `aria-describedby`.
- Set `aria-invalid="true"` on the field.
- Use more than colour: include text and, optionally, an icon.
- Describe how to fix it: "Enter a phone number with 10 digits, like 0712345678", not "Invalid".
- Remove the error (and `aria-invalid`) promptly once corrected.

```html
<label for="email">Email</label>
<input id="email" type="email" aria-invalid="true" aria-describedby="email-error">
<p id="email-error" class="error"><span aria-hidden="true">⚠</span> Enter an email address in the correct format, like name@example.com</p>
```

Validate at sensible times (see `frontend-components`, forms): not on every keystroke while the user is still typing.

## 6. Error summaries

For forms with several fields, on submit with errors:

- Show a summary at the top of the form listing each error, each linking to its field.
- Move focus to the summary (give it `tabindex="-1"` and a heading), so screen-reader users hear that there are problems.
- Also show each error next to its field.
- Update the page title to begin with "Error:" where the project follows that convention, so it's announced.

## 7. Autocomplete and input help

- Add `autocomplete` attributes to fields collecting the user's own information (`name`, `email`, `tel`, `street-address`, `postal-code`, `cc-number`, `username`, `current-password`, `new-password`, `one-time-code`). This lets browsers fill them and helps people with memory or motor difficulties.
- Use the correct `type` (`email`, `tel`, `number`, `url`) or `inputmode`, so mobile devices show the right keyboard.
- Don't disable paste.
- Don't ask users to re-enter information they've already provided in the same process, unless it's essential (such as confirming a password, and even then, consider a "show password" option instead).

## 8. Accessible authentication

WCAG 2.2 requires that signing in doesn't depend on a cognitive test (such as remembering a password, solving a puzzle, or transcribing characters) unless an alternative or assistance is available.

- Allow password managers and paste in all credential fields.
- Allow copying one-time codes, and use `autocomplete="one-time-code"`.
- Offer alternatives to CAPTCHAs that require recognising images or text, or use less intrusive bot protection.
- Passkeys, email links, and similar methods are accessible alternatives.

## 9. Live regions

Live regions tell screen readers to announce content when it changes.

| Use | Markup | Behaviour |
|---|---|---|
| Status messages: "Saved", "5 results", "Added to basket" | `role="status"` (or `aria-live="polite"`) | Announced when the user is idle |
| Urgent errors that need immediate attention | `role="alert"` (or `aria-live="assertive"`) | Interrupts the current announcement |
| Logs, chat, feeds | `role="log"` | New entries announced politely |

Guidelines:

- **The live region element must already be in the page** before you change its content. Inserting a new element that is itself a live region often isn't announced.
- Update the text content of the existing region to trigger an announcement.
- Keep messages short and meaningful.
- Use `alert` sparingly. Interrupting the user for non-urgent information is disruptive.
- Don't announce changes the user caused and can already perceive, such as typing in a field.
- Status messages must be announced without moving focus (WCAG requirement for status messages).

## 10. Notifications and toasts

Temporary notifications are a common accessibility problem.

- Announce them with a live region.
- Keep them visible long enough to read, and pause dismissal on hover and focus; or don't auto-dismiss at all for important messages.
- Make them dismissible with a labelled close button.
- Never put essential information or the only copy of an error in a disappearing notification.
- Don't put interactive controls that need action (such as "Undo") only in a toast that disappears quickly; give enough time and make it reachable by keyboard.

## 11. Loading and busy states

- When loading content in response to user action, announce significant states: "Loading results", then "12 results loaded", through a status region.
- Mark regions being updated with `aria-busy="true"` while loading, and remove it when done.
- Spinners need an accessible name ("Loading"), or should be accompanied by announced text.
- Skeleton placeholders should be hidden from assistive technology, with loading announced separately.

## 12. Single-page application navigation

When a single-page application changes view without a full page load, screen readers may not notice anything happened.

On each route change:

1. **Update the page `title`** to describe the new view.
2. **Move focus** to the new content's main heading or main container (with `tabindex="-1"`), or **announce** the new page title through a status region.
3. **Restore scroll position** appropriately: top of the page for new views, previous position for back navigation.

Many routers and frameworks provide hooks or settings for this; use them consistently.