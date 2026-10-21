# Forms and Feedback

Forms are where users give the system information, and feedback is how the system tells them what happened. Both are where frustration happens most. This file covers building forms and the states around data and actions.

## Contents

1. Form structure
2. Validation
3. Submission
4. Loading states
5. Empty states
6. Error states
7. Success and notifications
8. Optimistic updates
9. Destructive actions

## 1. Form structure

- Use the project's form library or patterns consistently.
- Use real form elements: a `form` with a submit button, so pressing Enter submits and browsers and assistive technology behave correctly.
- Every field has a **visible label** connected to its input. Placeholders are not labels; they disappear on typing.
- Use the right input types (`email`, `tel`, `number`, `date`, `password`) and `autocomplete` attributes, so mobile keyboards and password managers work.
- Group related fields with headings or fieldsets; mark optional or required fields consistently.
- Ask only for what's needed. Every field adds effort and risk of abandonment.
- Keep user input when anything goes wrong: never clear a form because submission failed.

Apply `frontend-accessibility` to every form.

## 2. Validation

- **Validate on the client for speed and helpfulness, on the server for correctness and security.** Client-side validation can always be bypassed (see `security`).
- **Share validation rules** between client and server where the project supports it (such as a shared schema), so they don't diverge.
- **Timing:** validate a field when the user leaves it (on blur) or on submit, not on every keystroke while they're still typing. After a field has shown an error, re-validate as they fix it, so the error clears promptly.
- **Messages:** specific and helpful, next to the field: "Enter a phone number with 10 to 13 digits", not "Invalid input".
- **On submit with errors:** show all errors, move focus to the first invalid field or an error summary, and announce the errors to screen readers.
- **Server errors for specific fields** (such as "email already registered") should appear next to the relevant field, just like client-side errors.

## 3. Submission

- **Prevent duplicate submissions:** disable the submit button or ignore repeated submits while a request is in flight, and show that it's working.
- **Make the request idempotent** where duplicates would be harmful (see `backend-api-design`).
- **Show the outcome:** success (and what happens next), or failure (with the reason and a way to retry).
- **Handle navigation away** from unsaved changes where losing input would be costly: warn the user, or save a draft.
- **Long forms:** consider saving progress, or splitting into steps with clear progress indication.

## 4. Loading states

| Situation | Approach |
|---|---|
| Initial page or section content | Skeleton placeholders shaped like the final content, to avoid layout shift |
| Very short waits (under about a second) | Often nothing, or a subtle indicator after a short delay, to avoid flicker |
| An action the user triggered (save, submit) | Indicator on the button or near the action; disable repeat actions |
| Background refresh of data already shown | Keep showing current data; indicate refresh subtly, if at all |
| Long operations | Progress indicator with explanation; allow continuing other work if possible |

- Don't replace content already on screen with a spinner during a refresh; it's jarring.
- Reserve space for content that's loading, so the page doesn't jump when it arrives.
- Make loading states accessible: announce significant loading and completion to screen readers where appropriate.

## 5. Empty states

An empty list or page should never look broken.

- Explain why it's empty: "No orders yet", "No results for 'solar lamp'".
- Offer a next step: "Browse products", "Clear filters", "Create your first invoice".
- Distinguish "nothing exists yet" from "nothing matches your filters".

## 6. Error states

- **Explain in plain language** what happened, without technical details (see `error-handling`).
- **Offer a way forward:** retry, go back, change input, or contact support with a reference ID.
- **Scope the error:** if one section fails, show the error there and keep the rest of the page working.
- **Distinguish kinds of error** where it helps the user: offline, not found, not permitted, server problem.
- **Never fail silently,** and never leave a permanent spinner.
- **Report unexpected errors** to the project's error tracking.

## 7. Success and notifications

- Confirm successful actions in proportion to their importance: a subtle change for a toggle, a clear message for a submitted order.
- Show confirmation where the user is looking, or move them to the result (for example, the new item's page).
- **Notifications that disappear** ("toasts") must stay long enough to read, be announced to screen readers, and never be the only way important information or errors are communicated.
- Don't interrupt users with modal confirmations for routine successes.

## 8. Optimistic updates

Optimistic updates show the expected result immediately, before the server confirms.

**Good for:** frequent, low-risk, likely-to-succeed actions: liking, starring, reordering, toggling a setting, marking as read.

**Not for:** payments, orders, deletions that can't be undone, or anything where a false success could mislead the user.

How to do it safely:

1. Save the current state.
2. Apply the change to the screen.
3. Send the request.
4. On success, reconcile with the server's response.
5. On failure, restore the saved state and tell the user clearly.

## 9. Destructive actions

For actions that delete data or can't easily be undone:

- Make them visually distinct (for example, a danger style) and harder to trigger accidentally.
- **Prefer undo** over confirmation dialogs where possible ("Message deleted. Undo"): it's faster for users and safer, because people click through confirmations by habit.
- For truly irreversible, high-impact actions, require explicit confirmation that states the consequence: "Delete this project and its 42 invoices? This can't be undone."
- Never put a destructive action as the default button in a dialog.