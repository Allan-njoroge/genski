# Visual Design and Motion

People see in different ways: low vision, colour blindness, sensitivity to light, zoomed-in screens, small phones in bright sunlight. This file covers colour, contrast, typography, layout, targets, motion, and media.

## Contents

1. Colour contrast
2. Not relying on colour
3. Typography
4. Zoom, reflow, and text spacing
5. Orientation and responsive layout
6. Target size and pointer input
7. Motion and animation
8. Flashing
9. Time limits and moving content
10. Images and media
11. Themes and high contrast

## 1. Colour contrast

WCAG 2.2 AA minimum contrast ratios:

| Content | Minimum ratio |
|---|---|
| Normal text | 4.5 : 1 |
| Large text (at least 24 px regular, or about 18.7 px bold) | 3 : 1 |
| Interface components: input borders, focus indicators, toggle states, meaningful icons | 3 : 1 against adjacent colours |
| Meaningful graphics: chart lines, map markers, diagram elements | 3 : 1 |
| Disabled controls, pure decoration, logos | No requirement |

Guidelines:

- Check contrast with a contrast checker for every new colour pairing, including text on images and gradients (check the worst point).
- **Common failures:** light grey text, placeholder text, text on brand colours, white text on light buttons, thin icons, subtle input borders.
- Define colours as design tokens with contrast already checked, so components can't accidentally use failing combinations.
- Check every state: default, hover, focus, active, selected, error, and every theme.

## 2. Not relying on colour

Colour must never be the only way information is conveyed.

| Situation | Add |
|---|---|
| Error fields shown in red | Error text, and an icon |
| Status shown by a coloured dot | A text label ("Cancelled", "Paid") |
| Links in a paragraph | An underline (or another non-colour cue with 3:1 contrast against surrounding text) |
| Chart series in different colours | Direct labels, patterns, or distinct shapes |
| Required fields in a colour | Text or a symbol with explanation |
| Selected items highlighted by colour | A checkmark, border, or other indicator |

A quick test: view the page in greyscale. Is everything still understandable?

## 3. Typography

- Use a readable base font size (commonly at least 16 px for body text).
- Set font sizes in relative units (`rem`, `em`), so they scale with the user's settings.
- Comfortable line height (around 1.5 for body text) and line length (around 45 to 80 characters).
- Avoid justified text, which creates uneven gaps.
- Avoid long passages in all capitals or italics.
- Avoid images of text; use real text, which can be resized, translated, and read aloud.

## 4. Zoom, reflow, and text spacing

- **Resize text:** content and functionality work when text is enlarged to 200%.
- **Reflow:** at 320 CSS pixels wide (equivalent to 400% zoom on a 1280-pixel screen), content fits in one column without horizontal scrolling. Exceptions: content that needs two dimensions, such as data tables, maps, and diagrams, which may scroll within their own container.
- **Text spacing:** layouts must not break, overlap, or cut off text when users increase line height (to 1.5), paragraph spacing (to 2× font size), letter spacing (to 0.12×), and word spacing (to 0.16×).
- **Don't disable zoom:** never set `maximum-scale=1` or `user-scalable=no` in the viewport tag.
- **Avoid fixed heights** on containers with text, and test with long content and translations.

## 5. Orientation and responsive layout

- Content works in both portrait and landscape, unless a specific orientation is essential.
- Layouts adapt to small screens without hiding essential content or functionality.
- Content that appears on hover or focus (tooltips, popovers) must be dismissible with Escape, hoverable (the user can move the pointer onto it without it disappearing), and persistent until dismissed or no longer relevant.

## 6. Target size and pointer input

- **Minimum target size (WCAG 2.2 AA):** at least 24 × 24 CSS pixels, or with enough spacing that a 24-pixel circle centred on each target doesn't overlap another target. Exceptions include links within sentences.
- **Recommended:** about 44 × 44 pixels for primary controls on touch devices.
- Enlarge small icons' clickable areas with padding rather than enlarging the icon.
- **Gestures:** anything done with multi-finger or path-based gestures (pinch, swipe patterns) also works with a single tap or click, such as zoom buttons alongside pinch.
- **Dragging (WCAG 2.2):** any drag-and-drop function also has a single-pointer alternative, such as "Move up" and "Move down" buttons, or a select menu.
- **Pointer cancellation:** trigger actions on release (the `click` event), not on press, so users can slide off to cancel.
- **Motion actuation:** functions triggered by shaking or tilting the device have a button alternative and can be turned off.

## 7. Motion and animation

- **Respect `prefers-reduced-motion`:** when users have asked for less motion, remove or reduce non-essential animation: parallax, large movements, zooms, auto-scrolling, and animated transitions between views. Simple fades are usually fine.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

(Adjust to the project's approach; some animations convey meaning and should be replaced, not just removed.)

- Keep animations short and purposeful.
- Don't animate in response to scrolling in ways that can't be turned off.

## 8. Flashing

Nothing on the page flashes more than three times in any one-second period. Flashing content can trigger seizures in people with photosensitive epilepsy. This applies to videos, animations, and transitions. Avoid large, high-contrast flashing areas, especially red.

## 9. Time limits and moving content

- **Time limits** (such as session timeouts) can be turned off, adjusted, or extended. Warn before expiry and allow at least 20 seconds to extend with a simple action. Exceptions: real-time events and cases where the limit is essential.
- **Moving, blinking, scrolling, or auto-updating content** that starts automatically and lasts more than five seconds can be paused, stopped, or hidden. This includes carousels, tickers, and animated backgrounds.
- **Audio that plays automatically** for more than three seconds can be paused or stopped, or its volume controlled independently.

## 10. Images and media

**Alt text:**

| Image | Alt text |
|---|---|
| Informative (a photo of a product) | Describe what matters in context: "Solar lamp with USB charging port" |
| Functional (an image inside a link or button) | Describe the action or destination: "Home", "Search" |
| Decorative (background textures, flourishes) | Empty: `alt=""` |
| Complex (charts, diagrams, maps) | A short alt summary, plus a full description or data table nearby |
| Image of text | The same text (but avoid images of text) |

- Don't start with "Image of" or "Picture of"; screen readers announce it's an image.
- Keep alt text concise; put long descriptions in visible text where others can benefit too.
- Inline SVG icons: decorative ones get `aria-hidden="true"` and `focusable="false"`; meaningful standalone ones get `role="img"` and a name.

**Media:**

- Pre-recorded video with speech has accurate captions (not just automatic captions without review).
- Audio-only content has a transcript.
- Video that conveys important visual information has audio description or a text alternative.
- Live video with speech has live captions, where WCAG AA applies.
- Media players have keyboard-accessible, labelled controls.

## 11. Themes and high contrast

- Check contrast in every theme (light and dark) the project supports.
- Respect the user's colour scheme preference where the project supports themes.
- Test with forced colours or high contrast modes (such as Windows Contrast Themes): make sure focus indicators, borders, icons, and states remain visible. Use real borders and outlines rather than relying only on background colours and shadows, which may be removed in these modes.