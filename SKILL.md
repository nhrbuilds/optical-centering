---
name: optical-centering
description: Use when building or reviewing UI rows that mix text with icons, badges, or uppercase labels, when a user reports text sitting off-center or icons hanging low against labels, or before shipping any chip, menu item, button, table header strip, or badge row. Covers measuring true glyph-ink positions in the browser and correcting vertical optical imbalance between text and geometric elements with verifiable CSS.
license: MIT
version: 1.0.0
---

# Optical centering

CSS centers boxes. Readers judge ink. Those two centers rarely coincide, and the gap is where the classic complaints come from: the icon hangs low while the text rides high, or an uppercase label floats above the middle of its strip.

## Why it happens

A text line box is taller than its glyphs. The font's ascent and descent define the box, and the descent is empty space for glyphs like `g`, `y`, and `p`. When a flex container centers a row with `align-items: center`, it centers these boxes:

- Text with descenders: ink reaches the bottom of the box, so the ink center sits at or below the box center. The icon next to it reads as sitting high.
- Text without descenders (uppercase, digits, most short labels): the box still reserves the same empty descent, so the ink center sits above the box center. The icon next to it reads as hanging low, exactly as the user describes it: icon down, text up.
- Icons and geometric elements have no ink asymmetry. Their rect center is their visual center.

Both failures come from the same mechanism, so one fixed nudge cannot serve every row. A menu containing "Settings" and "Clear All Items" can measure opposite offsets on adjacent items.

`line-height` changes the box height, not the ink position, and `align-items: center` keeps centering the same lopsided pair. That is why the naive fixes do not work.

## Measure before touching anything

Never ship a nudge taken from squinting at a screenshot. Three probes, in rising order of cost and truth. Run the cheapest that answers your question, record the before value, apply one fix, record the after value.

### 1. Box delta (fast, blind to ink)

Compares the icon's rect center with the text's line-box center. It cannot see ink, so a zero here means nothing, but a large value is a real layout bug.

```js
// For each element holding an svg and a text node, positive means the
// icon sits lower than the text box.
const sr = svg.getBoundingClientRect();
const range = document.createRange();
range.selectNodeContents(textNode);
const tr = range.getBoundingClientRect();
const delta = sr.top + sr.height / 2 - (tr.top + tr.height / 2);
```

### 2. Ink delta (sees ink, trusts font metrics)

Canvas `measureText` reports where the glyphs actually sit. Place the baseline with the WebKit half-leading model, then compare the icon center with the ink center.

```js
const ctx = document.createElement("canvas").getContext("2d");
const cs = getComputedStyle(el);
ctx.font = `${cs.fontStyle} ${cs.fontWeight} ${cs.fontSize} ${cs.fontFamily}`;
const m = ctx.measureText(text);
const baseline = tr.top + (tr.height - m.fontBoundingBoxAscent - m.fontBoundingBoxDescent) / 2
  + m.fontBoundingBoxAscent;
const inkCenter = baseline - m.actualBoundingBoxAscent
  + (m.actualBoundingBoxAscent + m.actualBoundingBoxDescent) / 2;
const delta = svgCenterY - inkCenter;
```

`fontBoundingBox` values differ between browsers and may not match the layout engine's baseline placement, so treat this probe as a signal, not a verdict.

### 3. Pixel ground truth (authoritative)

Render the element at 3x to 4x device scale (`browser.newContext({ deviceScaleFactor: 4 })` in Playwright), screenshot just that element, and find the ink rows per column group: background = the most common pixel color, foreground = any pixel past a threshold from it, group columns into runs (icon run, text run) and take each run's vertical center. Antialiasing fights the threshold, so verify the run split visually before trusting the numbers. This is the arbiter when probes 1 and 2 disagree.

### The action threshold

Deltas under 0.5 CSS px are invisible at 1x and churn if you chase them. Fix measured deltas of 0.5px and above. One fix per row, then remeasure.

## Fixes, in order of preference

### line-height: 1 on single-line uppercase or numeric labels

The strip, badge, and table-header case. With `line-height: 1` the centered box hugs the em square instead of the inflated line box, which removes most of the ride-high. This is a fix you can apply site-wide without per-row measurement.

Know its limit: on fonts with compact metrics (ascent plus descent near 1em), the glyphs sit at the top of the em box and the caps keep riding high even at `line-height: 1`, because the ink offset inside the box is mathematically independent of line-height. The centered box moves with it. Measure after applying; if the ink still rides high, use one of the corrections below.

### A padding differential for padded strips

When the label and its background live on the same element (table header strips, badges), correct with padding, not transforms. Moving the transform moves the background too. Measured rule for the common 11px uppercase strip: caps occupy roughly the top 7px of an 11px box, so a 4px differential between top and bottom padding centers the ink while keeping the strip height:

```css
/* was: padding: 6px 8px;  ink gaps measured 6px above, 10px below */
.vars-table-head {
  padding: 8px 8px 4px;
}
```

Derive the differential for your own measure: `pad_top - pad_bottom = line_height - cap_height`. On a spacing scale, round to the nearest step pair that preserves the difference.

### A 1px label nudge

When measured ink sits low against icons in a chip or menu, lift the label's own span:

```css
.filter-chip span {
  display: inline-flex;
  align-items: center;
  line-height: 1;
  transform: translateY(-1px);
}
```

Nudge the element you can target. Bare text inside a flex row is an anonymous box that CSS cannot select, which is why the label should live in a span.

### An icon nudge via the translate property

When the measured icon sits below the ink, move the icon up:

```css
.menu-item svg {
  translate: 0 -1px;
}
```

Use the `translate` property, not a second `transform` declaration. `transform` clobbers: an icon rotated by `transform: rotate(90deg)` loses the rotation the moment a rule adds `transform: translateY(-1px)`. The individual transform properties compose with `transform` instead of replacing it.

Direction matters and is not symmetric. Lifting text and lifting the icon produce the same relative change, but text-only siblings of the row may already be correct. Measure, then move exactly one side.

### Align icons to the cap band, not the full ink

For mixed-case labels, the reader judges the icon against the cap-height band from cap top to baseline, not against ink that includes a `g` descender. Aligning an icon to the full-ink center of "Settings" makes it read low. When in doubt, target the icon center midway between cap top and baseline.

### Baseline-align wordmark and descriptor pairs

An app name and a surface label beside it are different font sizes on one line. Centering each box puts their baselines a pixel apart, which reads as the smaller text floating up. The typographic contract for this pair is a shared baseline:

```css
.title-container { display: flex; align-items: center; }
.header-subtitle { translate: 0 1px; } /* measured drop onto the wordmark baseline */
```

Measure the two cap bands: the fix lands when both baselines agree within half a pixel. Icons in the same row center against the cap band, not the baseline.

### Digits inherit the uppercase bias

Badge counts and counters are lining figures: no descenders, same top-of-box ink placement as uppercase. Any centered digit badge needs the same correction as an uppercase label. Prefer an em-relative nudge so it tracks the font size:

```css
.folder-count { translate: 0 0.15em; }
```

### Inputs and their placeholders

A text input centers its content by the same box math, so placeholder and typed text ride high inside the field by about a pixel. Nudge the input element itself when it has no visible border or background of its own, and give the field's leading icon the same correction so icon and text move together:

```css
.search-field > svg { translate: 0 1px; }
.search-field input { translate: 0 1px; }
```

Verify with the field's typed state too, not only the placeholder.

### Same-slot absolute positioning for badge and actions

The recurring list-row case: a count badge at rest and action buttons on hover, sharing one spot at the row's end. Reserve no flow space for the hidden element, or the badge sits visibly left of the edge, waiting for buttons that are not there yet:

```css
.row { position: relative; }
.row .count,
.row .actions {
  position: absolute;
  inset-inline-end: 8px;
  top: 0;
  bottom: 0;
  display: inline-flex;
  align-items: center;
}
.row .actions { opacity: 0; pointer-events: none; }
.row:hover .actions { opacity: 1; pointer-events: auto; }
.row:hover .count { opacity: 0; }
```

Crossfade between the two on hover. The buttons' solid background covers the fading count, so no extra choreography is needed. Elements that only render conditionally, such as a drag-over pill, take the slot with `margin-left: auto`, and the absolute pair yields to them.

## Anti-patterns

- Declaring centering from box math or a spec sheet. The font's real metrics in the real engine decide, and they vary. Measure pixels.
- Shipping a nudge without a measured before and after.
- Moving the icon and the text at once. Pick one side from the measurement.
- Stacking `transform` declarations and silently discarding rotations.
- Chasing deltas under 0.5px.
- Trusting one probe. Box delta, ink delta, and pixels answer different questions.
- Assuming a harness capture proves the fix. Verify in the running app the user sees.
- `overflow: hidden` on a fixed-height well that must clip its rows. `hidden` makes the box a scroll container, its flex children lose their content-based minimum size, and rows vanish. Use `overflow: clip`, which forbids scrolling and keeps the minimum size.

## Verify

1. Rerun the probe that found the problem. Require the delta within plus or minus 0.25px.
2. Capture the fixed element at 4x device scale and read the crop. Pixels beat prose.
3. If a real build of the app is running, capture the real window and measure there before claiming the fix. A test harness shares the engine but not always the fonts, window scale, or theme of the user's session, and a number measured only in the harness can be wrong where it matters. Locate the window, verify its content is actually on screen before clicking or measuring, and re-capture after the fix lands through hot reload.
4. Look at it once at 1x in both themes. A correction that only works on one background color is not done.
