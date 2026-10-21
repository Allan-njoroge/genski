# Testing Accessibility

No single method finds all accessibility problems. Automated tools are fast but catch only part of the issues; manual testing with a keyboard and screen reader catches most of the rest; testing with disabled users finds problems nobody else will.

## Contents

1. Layers of testing
2. Automated testing
3. Keyboard testing
4. Screen reader testing
5. Zoom, reflow, and spacing
6. Visual checks
7. A quick manual check for every change
8. Fixing reported issues
9. Testing with real users

## 1. Layers of testing

| Layer | Catches | When |
|---|---|---|
| Linting (accessibility lint rules for the framework) | Missing alt, invalid ARIA, clickable non-interactive elements | While writing code, in CI |
| Automated checks in tests (such as axe) | Missing names, contrast failures, invalid ARIA, duplicate IDs, missing labels | Component and end-to-end tests, CI |
| Role-based queries in component tests | Missing roles and names on controls | Every component test (see `frontend-components`) |
| Manual keyboard testing | Focus order, traps, invisible focus, unreachable controls | Every new or changed interactive feature |
| Screen reader testing | Unclear names, missing announcements, confusing structure | New components, key flows, significant changes |
| Zoom, reflow, and spacing checks | Layouts that break, overlap, or hide content | New layouts and pages |
| Testing with disabled users | Real-world usability problems | Key journeys, periodically |

## 2. Automated testing

- Enable the framework's accessibility lint rules (for example, a JSX accessibility plugin for React) and fix their warnings rather than silencing them.
- Run an automated checker (such as axe-core) in component tests and end-to-end tests of key pages, failing the build on serious violations.
- Run checks against each important state, not just the initial render: open dialogs, expanded menus, error states, and loaded data.
- Remember the limits: automated tools can't judge whether alt text is meaningful, whether focus order makes sense, whether announcements are helpful, or whether a custom widget works with a keyboard. Passing automated checks is a minimum, not proof of accessibility.

## 3. Keyboard testing

Put the mouse aside and complete the task:

- [ ] Tab reaches every interactive element, and nothing non-interactive.
- [ ] The order is logical and matches the visual layout.
- [ ] Focus is always clearly visible, and never hidden behind sticky elements.
- [ ] Every control works with the expected keys (Enter, Space, arrows, Escape).
- [ ] No keyboard traps; you can always move on.
- [ ] Dialogs move focus in, keep it inside, close with Escape, and return focus.
- [ ] Menus and popovers open, navigate, and close with the keyboard.
- [ ] Focus goes somewhere sensible after deleting items, submitting forms, and changing views.
- [ ] The skip link appears on first Tab and works.
- [ ] Hover-only content (tooltips, dropdowns) is also available on focus.

## 4. Screen reader testing

Test with at least one screen reader and browser combination that users commonly use:

| Platform | Screen reader | Browser commonly paired |
|---|---|---|
| Windows | NVDA (free) or JAWS | Chrome, Firefox, Edge |
| macOS | VoiceOver (built in) | Safari |
| iOS | VoiceOver (built in) | Safari |
| Android | TalkBack (built in) | Chrome |

Check:

- [ ] Page title is announced and describes the page.
- [ ] Headings list (using the screen reader's headings navigation) shows a logical outline.
- [ ] Landmarks list shows the main regions, each distinctly named where needed.
- [ ] Every control announces a clear name, the right role, and its state.
- [ ] Images announce useful alt text, or are skipped if decorative.
- [ ] Form fields announce their label, hints, required state, and errors.
- [ ] Dynamic changes (results loaded, item saved, errors) are announced.
- [ ] Dialogs announce their title when opened.
- [ ] View changes in single-page apps are noticed.
- [ ] Tables announce headers when moving between cells.

Learn the basic commands for the screen reader you use. Screen readers behave differently; problems found in one should be checked in another when the cause is unclear.

## 5. Zoom, reflow, and spacing

- [ ] Browser zoom to 200%: all content and functions available, nothing overlapping or cut off.
- [ ] Viewport at 320 CSS pixels wide (or 400% zoom): single column, no horizontal scrolling for regular content.
- [ ] Increased text spacing (use a text-spacing bookmarklet or browser extension): no overlapping or clipped text.
- [ ] Larger default font size in browser settings: text scales.
- [ ] Portrait and landscape both work on mobile.

## 6. Visual checks

- [ ] Text contrast at least 4.5:1 (3:1 for large text), checked with a contrast tool, in every state and theme.
- [ ] Interface elements and focus indicators at least 3:1.
- [ ] Greyscale view: no information lost.
- [ ] Forced colours or high contrast mode: borders, focus, icons, and states still visible.
- [ ] Reduced-motion setting on: non-essential animation reduced or removed.
- [ ] Targets at least 24 × 24 pixels (or adequately spaced).

## 7. A quick manual check for every change

When time is short, for every new or changed interface, at minimum:

1. Run the automated checker on the changed screen in its key states.
2. Complete the main task with only the keyboard.
3. Zoom to 200% and check nothing breaks.
4. Check the contrast of any new colours.

Then do screen reader testing for new components and key flows.

## 8. Fixing reported issues

- **Fix the cause, not the symptom.** A missing name on a clickable `div` is best fixed by making it a `button`, not by adding ARIA attributes.
- **Fix it in the shared component** when the problem comes from one, so every use benefits.
- **Add a test** that would have caught it: a role-based query, an automated check of that state, or a keyboard interaction test.
- **Re-test with the same method** that found the problem (keyboard, screen reader, zoom).
- **Never suppress** a lint rule or automated check failure without fixing the issue or documenting why it's a false positive.

## 9. Testing with real users

Testing with disabled people who use assistive technology daily finds problems no checklist will: confusing wording, unexpected workflows, and issues specific to how people really use their tools. Recommend it for key journeys and major launches, where the project can arrange it.