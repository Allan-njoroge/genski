# Frontend Performance

Users notice slowness immediately, especially on mid-range phones and slower networks. Many users access the web on such devices, so performance work matters most where development machines least reveal it. This file covers measuring and improving loading, rendering, and responsiveness.

## Contents

1. Measure first
2. Core Web Vitals
3. Loading less
4. Images and media
5. Rendering efficiently
6. Long lists
7. Responsiveness
8. Network and data
9. Checklist

## 1. Measure first

- Optimise only what measurement shows is slow. Guessing leads to complex code that doesn't help.
- Use the browser's performance and network tools, the framework's profiler, and audits such as Lighthouse.
- **Test on realistic conditions:** throttle the CPU and network to approximate a mid-range phone on a mobile connection. Fast development laptops hide most problems.
- Collect real-user measurements where the project supports it, since lab tests don't capture every device and network.

## 2. Core Web Vitals

Google's Core Web Vitals measure the user experience of loading, responsiveness, and stability. Commonly used "good" thresholds, measured at the 75th percentile of page loads:

| Metric | Measures | Good |
|---|---|---|
| **Largest Contentful Paint (LCP)** | How long until the main content appears | 2.5 s or less |
| **Interaction to Next Paint (INP)** | How quickly the page responds to interactions | 200 ms or less |
| **Cumulative Layout Shift (CLS)** | How much the layout jumps unexpectedly | 0.1 or less |

Common causes and fixes:

- **Slow LCP:** large unoptimised images, render-blocking scripts and styles, slow server responses, content that waits on client-side data fetching. Fix with optimised images, preloading the main image, server rendering or earlier data fetching, and less blocking JavaScript.
- **Poor INP:** long-running JavaScript on the main thread, expensive re-renders on interaction, large event handlers. Fix by breaking up long tasks, reducing re-render scope, and deferring non-urgent work.
- **High CLS:** images and embeds without dimensions, content inserted above existing content, web fonts swapping late, banners pushing content down. Fix by reserving space, setting dimensions, and avoiding inserting content above what the user is viewing.

## 3. Loading less

- **Split code** by route, and lazy-load large features that aren't needed immediately (rich editors, charts, maps).
- **Check bundle size** with the build tool's analyser. Look for large dependencies and duplicate packages.
- **Choose dependencies carefully:** a large library for a small feature costs every user on every visit. Prefer smaller alternatives or the platform's built-in features.
- **Import only what you use,** so unused code can be removed by the build.
- **Defer non-critical scripts:** analytics, chat widgets, and other third-party scripts should not block the main content.
- **Use caching:** static assets with content hashes in file names can be cached long-term.

## 4. Images and media

- Serve images at the size they're displayed, with responsive images (`srcset`, `sizes`) for different screens.
- Use modern formats (such as WebP or AVIF) with fallbacks where needed.
- **Always set width and height** (or aspect ratio) to reserve space and prevent layout shift.
- **Lazy-load images below the fold;** never lazy-load the main image at the top of the page.
- Compress images, and avoid using images for text.
- Use the framework's image component where available; it often handles much of this.

## 5. Rendering efficiently

- **Keep state close to where it's used,** so a change re-renders only the part of the tree that depends on it.
- **Split large components** so unrelated parts don't re-render together.
- **Avoid creating new objects and functions on every render** when they're passed to memoised children, if the framework requires stable references for memoisation to work.
- **Memoise selectively:** use the framework's memoisation for expensive calculations and for components that re-render often with unchanged props, where profiling shows a benefit. Memoising everything adds complexity and overhead.
- **Use stable keys** for lists, from the data's IDs, so the framework can update items instead of recreating them.
- **Avoid layout thrashing:** don't alternate between reading layout values (such as element sizes) and writing styles in loops.
- **Prefer CSS** for animations and transitions, animating `transform` and `opacity` where possible, which browsers can do efficiently.

## 6. Long lists

- **Paginate or load more** for large collections, rather than loading everything (see `backend-api-design`).
- **Virtualise** lists or tables with hundreds or thousands of rows: render only the visible rows plus a small buffer. Use a maintained virtualisation library.
- Keep each row component light.
- Make sure virtualised lists remain accessible: keyboard navigation and screen readers must still work.

## 7. Responsiveness

The interface must react to input quickly, even when work is happening.

- **Break up long tasks:** split heavy JavaScript work into smaller pieces, or move it to a web worker.
- **Debounce or throttle** handlers for frequent events: typing in search, scrolling, resizing.
- **Give immediate feedback** to interactions (pressed state, spinner, optimistic update) even if the full result takes longer.
- **Defer non-urgent updates** using the framework's tools for marking updates as low priority, where available.

## 8. Network and data

- **Avoid request waterfalls:** a component that loads, then renders a child that starts its own request, then another, multiplies latency. Fetch data in parallel, earlier (at the route level), or in one request.
- **Cache and deduplicate** with the data layer (see `state-and-data.md`).
- **Prefetch** likely next pages or data when the user shows intent (hovering a link), where the framework supports it.
- **Request only what's needed:** use pagination and field selection where available.
- **Handle slow and offline networks** gracefully, with clear states and retries.

## 9. Checklist

- [ ] Tested with throttled CPU and network.
- [ ] Core Web Vitals within targets on key pages.
- [ ] Routes and large features are code-split.
- [ ] No unexpectedly large dependencies in the bundle.
- [ ] Images sized, compressed, in modern formats, with dimensions, lazy-loaded below the fold.
- [ ] No layout shifts as content loads.
- [ ] Long lists paginated or virtualised.
- [ ] No request waterfalls on key pages.
- [ ] Expensive work doesn't block interactions.