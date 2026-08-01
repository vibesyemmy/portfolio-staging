# Kolo Launch Banner — Design Spec

**Date:** 2026-08-01
**Status:** Approved
**Target:** Announce the launch of [Kolo](https://usekolo.app/) (a color studio: palettes → contrast-safe tokens → code) at the top of every page of the portfolio.

## Decisions (confirmed with the user)

- **Behavior:** dismissible, stays pinned while scrolling.
- **Scope:** every page.
- **Style:** bright orange strip (`--accent`) with dark ink on top. Because the fill
  is the accent colour, everything on the strip inverts: the `NEW` chip becomes a
  dark pill with white text, and the call to action carries a standing underline
  rather than a colour change (nothing on the strip can be tinted orange any more).

## Placement and the nav collision

The banner is the first child of `<body>`, rendered in `src/layouts/Layout.astro`. Every page routes through that layout — `index`, `about`, `blog/index`, `blog/[slug]`, `heatmap-logo`, and `work/[slug]` (via `ProjectLayout` → `Layout`) — so a single insertion covers the whole site.

It uses **`position: sticky; top: 0`**. Sticky is load-bearing: the element stays in normal flow (reserving its own space, pushing page content down without body-padding hacks) while remaining pinned during scroll. Lenis runs native scrolling (no transform on a wrapper — see `src/scripts/smooth-scroll.ts`), so sticky is safe.

The nav is `position: fixed; top: 1rem; z-index: 100`, so it ignores flow and would overlap the banner. The banner's height is therefore published as a CSS custom property that the nav consumes:

```css
:root { --banner-h: 44px; }
.nav-wrap { top: calc(1rem + var(--banner-h, 0px)); }
```

The `var(--banner-h, 0px)` fallback means the nav is correct even if the variable is never defined.

Two other places must respect the same offset so in-page anchors don't land beneath the chrome:

- `html { scroll-padding-top }` in `src/styles/global.css` — add `--banner-h`.
- `NAV_OFFSET` in `src/scripts/smooth-scroll.ts` — currently a hardcoded `-96`; it must grow by the live banner height (read from the computed custom property at scroll time, so it stays correct after dismissal).

## Dismissal

The `×` button writes `localStorage.setItem('kolo-banner', 'dismissed')` and stamps `document.documentElement.dataset.koloBanner = 'dismissed'`. CSS does the rest:

```css
html[data-kolo-banner="dismissed"] { --banner-h: 0px; }
```

The banner hides and the nav returns to `top: 1rem` — driven by one variable, with no JS layout math.

**Anti-flash requirement:** the dismissed state is read by an inline `<head>` script that runs *before first paint*, reusing the pattern already present in `Layout.astro` (`document.documentElement.classList.add('js')`). Returning visitors must never see the banner paint and then disappear. This is the same class of bug as the ContentGate FOUC fix — styles and state that arrive after paint cause a visible flash.

## Content and style

Dark strip (`--dark-bg`, `#0a0a0a`) with white text and an orange accent for the call to action. This keeps the frosted-white nav pill readable floating above it and matches the site's existing dark-section language.

Copy (desktop):

> `NEW` · I launched **Kolo** — turn one palette into contrast-safe tokens and code. **Try it free →**

The strip is a single link to `https://usekolo.app/`, opening in a new tab with `rel="noopener noreferrer"`. The `×` is a separate button, not nested inside the link.

On narrow screens the long clause is hidden via CSS so the line never wraps (which would make `--banner-h` a lie and misalign the nav). The short form reads:

> **Kolo** — the color studio →

## Accessibility

- The banner is a `<aside>` labelled `aria-label="Announcement"`.
- The dismiss button has `aria-label="Dismiss announcement"` and is a real `<button type="button">`.
- The outbound link's accessible name states the destination; the `→` is decorative (`aria-hidden`).
- Focus-visible styling is inherited from the global `:focus-visible` rule.
- Any entrance transition is gated behind `prefers-reduced-motion: no-preference`.

## Files

- **Create** `src/components/Banner.astro` — markup, scoped styles, dismiss script. One responsibility: render and dismiss the announcement.
- **Modify** `src/layouts/Layout.astro` — import and render `<Banner />` as the first body child; add the pre-paint inline script.
- **Modify** `src/components/Nav.astro` — `top: calc(1rem + var(--banner-h, 0px))`.
- **Modify** `src/styles/global.css` — define `--banner-h`, the dismissed override, and the `scroll-padding-top` adjustment.
- **Modify** `src/scripts/smooth-scroll.ts` — make `NAV_OFFSET` account for the live banner height.

No new dependencies.

## Testing

No unit tests: this is presentational markup plus roughly ten lines of DOM wiring, consistent with how `ContentGate.astro` and the bowling module were handled in this repo (pure functions get vitest; DOM/visual work is verified in the browser).

Verification, all in-browser against the dev server:

1. Banner renders at the top of the home page; the nav pill sits below it, not overlapping.
2. Banner persists at the top while scrolling.
3. It appears on a case study page (`/work/...`), a blog post, and `/about`.
4. Clicking `×` hides it and the nav slides up to `top: 1rem`.
5. After a reload the banner stays hidden **with no flash** of it appearing first.
6. Clearing `localStorage` brings it back.
7. At a 375px-wide viewport the text does not wrap and the nav clears it.
8. An in-page anchor (nav "Work") scrolls to a position not hidden under the banner + nav.
9. `npm run build` succeeds.

## Non-goals

- No analytics or click tracking on the banner.
- No scheduled auto-expiry date; the banner is removed by editing the code when the launch moment passes.
- No animation beyond a subtle entrance.
- No per-page opt-out.
