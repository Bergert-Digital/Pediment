# Image/Content Slider Block — Design

**Date:** 2026-06-19
**Status:** Approved (pending spec review)

## Goal

Add an image/content slider to the Pediment block selection: a one-slide-at-a-time
carousel where each slide pairs an image with a colored content panel (heading +
copy, etc.). Manual navigation only (prev/next arrows + dot nav). Styled in
Pediment's own design language (navy panel, rounded corners, theme tokens) rather
than the yellow-panel reference screenshot that prompted the work.

## Block structure

Two blocks, following the established parent/child pattern (`testimonial-grid` +
`testimonial`):

### `pediment/slider` (parent)

- `category: pediment`, `supports: { html: false, align: [ "wide", "full" ] }`.
- Holds `pediment/slide` children via InnerBlocks (`allowedBlocks` restricted to
  `pediment/slide`, default template = 2 slides, `templateLock: false`).
- Owns the slider chrome (track, prev/next arrows, dot nav) and the frontend
  interactivity.
- Slider-level attributes (apply to every slide, for visual coherence):
  - `mediaPosition`: `"left"` (default) | `"right"` — flips image/panel side.
  - `panelColor`: string — panel background color value, default the navy
    `primary` preset. Chosen via a theme-palette-bound color picker (custom
    colors allowed).
- `viewScriptModule: file:./view.js` (WordPress Interactivity API).

### `pediment/slide` (child)

- `parent: [ "pediment/slider" ]`, `supports: { html: false, reusable: false }`.
- Attributes: `mediaId` (number, default 0), `altOverride` (string, default "").
- **InnerBlocks** for the panel content (heading / paragraph / button / list…),
  with a starter template of a heading + paragraph. Mirrors how `media-text`
  models its text column.
- The slide is purely image + content. Layout direction and panel color come from
  the parent via cascading CSS (modifier class + CSS var on the parent wrapper) —
  no block context wiring needed.

## Registration

Blocks auto-register from `build/blocks/<name>/` via
`wp_register_block_metadata_collection` in `inc/register-blocks.php`. Adding
`src/blocks/slider/` and `src/blocks/slide/` and running the existing build
(`npm run build`) is sufficient — no PHP edits.

## Rendering

### `slide/render.php`

```
<div class="starter-slide" data-wp-context='{"index":0}'
     data-wp-class--is-active="state.isActive"
     data-wp-bind--aria-hidden="!state.isActive"
     role="group" aria-roledescription="slide">
  <figure class="starter-slide__media">
    {image via wp_get_attachment_image($mediaId,'large',…) OR the media-text SVG placeholder}
  </figure>
  <div class="starter-slide__panel">
    {InnerBlocks content ($content)}
  </div>
</div>
```

Image + placeholder markup reuses `media-text/render.php`'s approach (alt override,
`large` size, inline SVG fallback so the layout stays meaningful before an image is
set). The `index` in `data-wp-context` is a placeholder `0`; the real per-slide
index is assigned client-side by DOM order in `view.ts` `init` (see Interactivity).

### `slider/render.php`

```
<section class="starter-slider is-media-{left|right}"
         style="--slide-panel-bg:{color}; --slide-panel-fg:{light|dark token}"
         data-wp-interactive="pediment/slider"
         data-wp-context='{"active":0,"count":N}'
         role="group" aria-roledescription="carousel">
  <div class="starter-slider__track">{$content = rendered slides}</div>
  <button class="starter-slider__arrow starter-slider__arrow--prev"
          aria-label="Vorherige Folie" data-wp-on--click="actions.prev">‹</button>
  <button class="starter-slider__arrow starter-slider__arrow--next"
          aria-label="Nächste Folie" data-wp-on--click="actions.next">›</button>
  <div class="starter-slider__dots" role="tablist" aria-label="Folien">
    {N dot <button>s, each with data-wp-context index, aria-label "Folie i von N"}
  </div>
  <p class="screen-reader-text" aria-live="polite" data-wp-text="state.liveLabel"></p>
</section>
```

- Slide count `N` = `count( $block->parsed_block['innerBlocks'] )`.
- Dots generated server-side (one `<button>` per slide). Each dot carries its
  index via `data-wp-context='{"index":i}'`, binds active/`aria-selected` state,
  and `data-wp-on--click="actions.goTo"`.

**Slide indexing — decision:** indices are assigned client-side by DOM order in the
`view.ts` `init` callback. Neither `slide/render.php` nor the dots need PHP to know
their position; `init` walks the slider's `.starter-slide` children in order and
seeds each slide's context `index` (and verifies the dot order matches). This avoids
any cross-block / request-scoped PHP state and keeps `slide/render.php` a pure,
context-free template. DOM order is the source of truth, which is exactly what the
visual order is.

### Panel color & text contrast

`panelColor` resolves to a CSS color. `render.php` computes its relative luminance
and sets `--slide-panel-fg` to a light token (on dark panels) or a dark token (on
light panels), so body text stays readable on any chosen color with no separate
text-color control. Muted body text derived via `color-mix` on `--slide-panel-fg`,
matching the `cta` block's treatment. Default panel = navy `primary` with light
text.

## Styling (`style.scss` per block, `starter-*` BEM)

The slider is a single rounded, shadowed card; the image and panel are full-bleed
halves clipped to that rounding (image full-bleed cover, square inner seam meeting
the panel, outer corners following the card).

- `.starter-slider`: un-clipped outer wrapper — positioning context for the
  overlaid arrows and the dots row below the card. No rounding/overflow itself.
- `.starter-slider__track`: the **card** — slides stacked in the same grid cell;
  only `.starter-slide.is-active` is visible (fade via `opacity`/`visibility`).
  Carries the **`--r-panel` rounding + `box-shadow: var(--wp--preset--shadow--medium)`**
  and `overflow: hidden` so the full-bleed image and panel clip to the rounded
  corners.
- `.starter-slide`: `display: grid; grid-template-columns: 1fr 1fr;
  align-items: stretch` (both halves full height). `.is-media-left` /
  `.is-media-right` on the parent control image `order`. Panel uses
  `var(--slide-panel-bg)` / `var(--slide-panel-fg)`.
- `.starter-slide__media`: full-bleed — `margin: 0`, fills its half.
- `.starter-slide__img`: `width: 100%; height: 100%; object-fit: cover` with a
  fixed `aspect-ratio` (e.g. `4 / 3`) on the media column, so every slide is the
  **same height** and the carousel doesn't jump when slides swap. **No** per-image
  border-radius or shadow (the rounding/shadow live on the slider shell). The
  `media-text` SVG placeholder fills the same area when no image is set.
- `.starter-slide__panel`: padded (`--r-panel`-scale padding like `cta`), content
  vertically centered.
- Responsive: collapse to single column under 781px (image first), matching
  `media-text`; the fixed media aspect ratio keeps the stacked image height sane.
- Arrows: pill/circle buttons using theme tokens; `:focus-visible` outline.
  Absolutely positioned on `.starter-slider` (the un-clipped wrapper), vertically
  centered, overlaid near the card's left/right edges — so they're never clipped by
  the card's `overflow: hidden`.
- Dots: small round buttons centered in a row below the card; active dot filled
  with accent.
- **No color literals** — only theme presets / CSS vars (satisfies `lint:colors`).

## Interactivity (`slider/view.ts`, WordPress Interactivity API)

Same stack as `mega-menu`'s `view.ts`.

- Store `pediment/slider`. Parent context `{ active, count }`; slides/dots carry
  their own `{ index }`.
- Actions:
  - `next()` — `active = (active + 1) % count` (wrap-around).
  - `prev()` — `active = (active - 1 + count) % count`.
  - `goTo()` — set `active` to the clicked dot's `index`.
- State (derived):
  - `isActive` — slide's `index === parent.active`.
  - `isCurrentDot` — dot's `index === parent.active` (drives active class +
    `aria-selected`/`aria-current`).
  - `liveLabel` — `"Folie {active+1} von {count}"` for the polite live region.
- Keyboard: ArrowLeft / ArrowRight move when focus is within the slider (handled in
  an `init` callback document/root listener scoped to the slider root). Arrows and
  dots are real `<button>`s, so Tab + Enter/Space work natively.
- No autoplay (per decision) — no timers, no `prefers-reduced-motion` handling
  required.

## Editor (`edit.tsx` per block)

### Slider edit

- `useInnerBlocksProps` allowing only `pediment/slide`, default template of 2
  slides.
- InspectorControls:
  - Image-side toggle (`ToggleGroupControl` left/right) → `mediaPosition`.
  - Panel color: `PanelColorSettings` (or `ColorGradientSettingsDropdown`) bound to
    the theme palette → `panelColor`.
- Canvas: slides render **stacked and all visible** (no JS carousel in the editor)
  so authors can edit every slide. Arrows/dots shown as non-interactive display
  hints, with a small "Folie N" label per slide.

### Slide edit

- Image: `MediaPlaceholder` + `MediaReplaceFlow` (mirrors `media-text/edit.tsx`),
  writing `mediaId` / `altOverride`.
- `InnerBlocks` for the panel with a starter template (heading + paragraph).

## Testing

- **Playwright e2e** (`tests/e2e/`, new `slider.spec.ts`, matching existing specs):
  - Insert slider, add/remove slides.
  - Front end: next/prev advance and wrap around; clicking a dot jumps to that
    slide; only one slide visible at a time.
  - Keyboard ArrowLeft/ArrowRight navigate.
  - `mediaPosition` toggle flips layout; `panelColor` applies to panel background.
- **Edit/render parity** if applicable (`edit-render-parity.spec.ts` conventions).
- Lints/format green: `lint:blocks`, `lint:colors`, `lint:js`, `phpcs`.

## Out of scope (YAGNI)

- Autoplay / timers.
- Multiple slides visible at once / peek / variable slides-per-view.
- Per-slide color or per-slide layout overrides.
- Thumbnail navigation, fraction counters, progress bars.
- Touch/swipe gestures (can be a follow-up; arrows + dots + keyboard cover the
  baseline). Documented here so it's a conscious deferral.
