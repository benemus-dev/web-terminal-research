# xterm.js DOM Renderer: Hidden Chrome Redraw Trigger Audit

Audited: `src/browser/renderer/dom/`, `src/browser/renderer/shared/`, `css/xterm.css`,
`src/browser/scrollable/`, `src/browser/Viewport.ts`, `src/browser/decorations/`

Date: 2026-03-26

---

## 1. CSS Transitions

**TWO transitions found**, both on the custom scrollbar:

```css
/* xterm.css line 250 */
.xterm .xterm-scrollable-element > .xterm-visible {
    transition: opacity 100ms linear;
}

/* xterm.css line 258-259 */
.xterm .xterm-scrollable-element > .xterm-invisible.xterm-fade {
    transition: opacity 800ms linear;
}
```

**Verdict: ACTIVE REDRAW TRIGGER.** The scrollbar fades out with an 800ms opacity
transition after every mouse-leave or scroll-stop (500ms hide timeout in
`scrollableElement.ts:536`, then the 800ms CSS fade). During this 800ms window,
Chrome's compositor is running continuous frames to interpolate opacity. This fires
after EVERY scroll interaction.

The `xterm-visible` class also has `transition: opacity 100ms linear` for the
reveal direction -- another 100ms of compositor work on every scroll-start.

**No transitions on cursor, rows, or selection.**

---

## 2. will-change

**None found anywhere.** No `will-change` in CSS or inline styles.

This is actually a problem for the scrollbar slider, which uses `transform: translate3d`
as a layer hint instead (see section 3). `will-change: transform` would be more modern
but functionally equivalent.

---

## 3. Compositor Layers

### Forced compositor layers:

1. **Scrollbar slider** -- `abstractScrollbar.ts:107-108`:
   ```ts
   this.slider.setLayerHinting(true);   // sets transform: translate3d(0px, 0px, 0px)
   this.slider.setContain('strict');
   ```
   This forces the scrollbar slider thumb onto its own compositor layer via the
   translate3d hack. The `contain: strict` is excellent -- it prevents layout/paint
   from propagating. **This layer exists permanently even when scrollbar is hidden
   (opacity: 0).**

2. **Scrollbar container** -- `abstractScrollbar.ts:75`: `position: absolute` on the
   scrollbar DOM node. Combined with z-index: 11 (from CSS), this likely promotes
   to its own layer.

### position: absolute elements in xterm.css (potential layer promotion):

| Element | Position | z-index | Layer likely? |
|---------|----------|---------|---------------|
| `.xterm-helpers` | absolute | 5 | Yes (z-index + absolute) |
| `.xterm-helper-textarea` | absolute | -5 | Yes but off-screen (-9999em) |
| `.composition-view` | absolute | 1 | Only when `display: block` |
| `.xterm-viewport` | absolute | none | Maybe (covers full area) |
| `.xterm-screen canvas` | absolute | none | Yes if canvas exists (WebGL mode) |
| `.xterm-accessibility` | absolute | 10 | Yes |
| `.xterm-decoration` | absolute | 6-7 | Per decoration |
| `.xterm-decoration-overview-ruler` | absolute | 8 | Yes |
| `.live-region` | absolute | none | Off-screen |
| `.xterm-scrollable-element .xterm-shadow` | absolute | none | Yes when visible |

### DOM renderer injected layers (from DomRenderer.ts `_injectCss`):

3. **Selection container** -- `position: absolute; z-index: 1` (line 284).
   Each selection div inside is also `position: absolute`. These are separate
   compositor layers when a selection exists.

4. **Selection divs** -- `position: absolute` (lines 291, 295). Created dynamically
   per selection row.

5. **Decoration-top spans** -- `position: relative; z-index: 2` (line 213 of xterm.css).
   Inline within rows, creates stacking context.

### Cursor element:

The cursor is NOT a separate DOM element. It is a CSS class (`xterm-cursor`) applied to
a `<span>` within the row div. It does NOT get its own compositor layer -- it is part of
the row's paint. This is efficient.

### Selection overlay:

The selection IS a separate compositor layer. The `xterm-selection` container is
`position: absolute; z-index: 1` overlaid on top of the row container. Selection
divs inside are also `position: absolute`. When selection exists, this is a separate
layer the compositor must composite on every frame.

**Total permanent compositor layers at idle (DOM renderer, no selection, no decorations):**
- Terminal root (`.xterm`, relative)
- Viewport (`.xterm-viewport`, absolute)
- Scrollable element container (relative)
- Screen element (`.xterm-screen`, relative)
- Row container (`xterm-rows`, static -- no forced layer)
- Selection container (absolute, z-index 1 -- empty but layer exists)
- Helpers container (absolute, z-index 5)
- Textarea (absolute, off-screen)
- Scrollbar container (absolute, z-index 11)
- Scrollbar slider (absolute + translate3d -- forced GPU layer)
- 2x style elements (not layers)

**Minimum ~8 compositor layers at idle.** Chrome can verify with Layers panel.

---

## 4. CSS Containment

**NOT used on rows or viewport.** The only `contain` usage is on the scrollbar slider:

```ts
// abstractScrollbar.ts:108
this.slider.setContain('strict');
```

**This is a significant missed optimization.** Adding containment would help:

- `contain: content` on the row container (`xterm-rows`) would prevent row changes
  from triggering layout recalc on the entire terminal.
- `contain: strict` on individual row divs would isolate each row's layout/paint
  entirely.
- `contain: layout style` on the viewport would prevent scroll-related layout from
  propagating upward.

Without containment, ANY row innerHTML change triggers layout recalc that can
propagate to the terminal root and potentially to ancestors.

---

## 5. Viewport DOM Structure

The terminal creates this layer hierarchy (DOM renderer path):

```
.xterm (position: relative)
  .xterm-viewport (position: absolute, overflow-y: scroll)
    .xterm-scrollable-element (position: relative)
      .xterm-screen (position: relative)
        .xterm-helpers (position: absolute, z-index: 5)
          textarea (position: absolute, off-screen)
        .xterm-rows (no positioning -- static)
          div (row 0)
          div (row 1)
          ...
          div (row N)
        .xterm-selection (position: absolute, z-index: 1)
          [dynamic selection divs]
        .xterm-decoration-container (no positioning in CSS -- inherits)
          [dynamic decoration divs, position: absolute, z-index: 6-7]
        style (theme CSS)
        style (dimensions CSS)
        style (scrollbar slider colors)
      .xterm-scrollbar [horizontal] (position: absolute)
        .xterm-slider (position: absolute, translate3d, contain: strict)
      .xterm-scrollbar [vertical] (position: absolute)
        .xterm-slider (position: absolute, translate3d, contain: strict)
      [shadow elements if useShadows=true]
  .xterm-accessibility (position: absolute, z-index: 10)
```

**Total container layers: 6-8 depending on state.** Each is a potential compositor
layer boundary in Chrome. The decoration container and selection container exist
permanently even when empty.

---

## 6. Scrollbar Implementation

**Custom scrollbar, NOT native `overflow: auto`.** The implementation is a VS Code-derived
custom scrollbar system (`src/browser/scrollable/`).

Key characteristics:
- `SmoothScrollableElement` wraps the screen element
- Scrollbar DOM is JS-created and positioned via `position: absolute`
- Slider uses `transform: translate3d(0,0,0)` for GPU layer promotion
- Visibility controlled by CSS class toggling (`xterm-visible`/`xterm-invisible`)
- Fade-out uses CSS `transition: opacity 800ms linear`
- **Smooth scroll animation uses `requestAnimationFrame` loop** (`scrollable.ts:316,348`)

**Compositor activity pattern:**
1. User scrolls -> scrollbar reveals (100ms opacity transition = compositor frames)
2. Scroll stops -> 500ms timeout -> scrollbar hides with 800ms fade
3. Total: up to 1400ms of compositor activity per scroll gesture
4. During smooth scrolling: continuous rAF loop until animation completes

The smooth scroll `Scrollable` class (`scrollable.ts:316`) chains rAF calls:
```ts
this._smoothScrolling.animationFrameDisposable = this._scheduleAtNextAnimationFrame(() => {
    // ... update scroll position, may schedule another frame
});
```

This is expected behavior during active scrolling but worth noting.

---

## 7. CSS Custom Properties

**Used only in the scrollbar CSS**, and only with fallbacks:

```css
background-color: var(--vscode-scrollbarSliderBackground, rgba(100, 100, 100, 0.4));
box-shadow: var(--vscode-scrollbar-shadow, #000) 0 6px 6px -6px inset;
```

These are VS Code integration points. The variables are on the scrollbar elements only,
not on `:root`. When used standalone (not in VS Code), the fallback values apply and
the CSS variables have no effect.

**No CSS custom properties on the terminal element, rows, or viewport.**

The scrollbar slider colors ARE injected via a `<style>` element per-terminal in
`Viewport.ts:87-97`, using direct color values (not CSS variables). Changes to theme
trigger `textContent` replacement on this style element, which causes a full style
recalc on the scrollbar subtree.

**Verdict: Not a concern for idle redraw.** CSS variables are scoped to scrollbar only.

---

## 8. Theme Injection

Theme colors are injected via a **`<style>` element** (`_themeStyleElement`) appended to
the screen element. `DomRenderer.ts:176-311` builds a massive CSS string covering:

- Foreground/background colors
- All 256 ANSI colors (as `.xterm-fg-N` / `.xterm-bg-N` classes)
- Cursor colors and animations
- Selection colors
- Dim colors

The entire CSS string is replaced on theme change via:
```ts
this._themeStyleElement.textContent = styles;
```

**This triggers a full style recalc on the entire terminal subtree.** For a terminal
with 24 rows x 80 columns, this means potentially thousands of spans get their computed
style invalidated.

However, theme changes are infrequent (user-initiated). **Not an idle concern.**

A separate dimensions `<style>` element handles the `inline-block` / height rules for
spans. Same mechanism, same full-recalc implications on resize.

---

## 9. Character Width / letter-spacing

**Two levels of letter-spacing:**

1. **Default spacing** on the row container (`DomRenderer.ts:324`):
   ```ts
   this._rowContainer.style.letterSpacing = `${spacing}px`;
   ```
   Set once during init and on font/DPR changes. NOT per-render.

2. **Per-span override** when a character's measured width differs from default
   (`DomRendererRowFactory.ts:477-478`):
   ```ts
   if (spacing !== this.defaultSpacing) {
       charElement.style.letterSpacing = `${spacing}px`;
   }
   ```
   Applied during `createRow()` only. Characters like `W` that match the default
   spacing get no inline style.

**Verdict: No idle layout thrashing.** Letter-spacing is set during render passes only
(`renderRows`), not continuously. The `WidthCache` uses OffscreenCanvas / HTMLCanvasElement
for measurement, which does NOT trigger DOM layout. The cache (`WidthCache.ts`) stores
measured widths in a Float32Array (fast path for ASCII) and a Map (rest).

The `_setDefaultSpacing` method does read from `WidthCache.get('W', ...)` and write to
`style.letterSpacing`, but only on init, font change, or DPR change.

---

## 10. IntersectionObserver

**Used in `RenderService.ts:125-133`:**

```ts
const observer = new w.IntersectionObserver(
    e => this._handleIntersectionChange(e[e.length - 1]),
    { threshold: 0 }
);
observer.observe(screenElement);
```

**Does this cause compositor ticks?** No. Chrome's IntersectionObserver is implemented
at the compositor level but does NOT cause continuous compositor ticks. It only fires
callbacks when the intersection ratio crosses the specified thresholds. With
`threshold: 0`, it fires only on visibility/invisibility transitions.

The handler pauses/resumes rendering when the terminal goes off-screen. This is
purely beneficial -- it PREVENTS unnecessary rendering when the terminal is not visible.

---

## BONUS FINDINGS

### 11. Cursor Blink Animation -- THE BIGGEST IDLE TRIGGER

**Three `@keyframes` animations** are defined for cursor blink (DomRenderer.ts:221-252):

```css
@keyframes blink_underline_N { 50% { border-bottom-style: hidden; } }
@keyframes blink_bar_N { 50% { box-shadow: none; } }
@keyframes blink_block_N { 0% { background-color: ...; color: ...; } 50% { ... } }
```

Applied as `animation: ... 1s step-end infinite;` on the cursor span.

**This causes continuous compositor activity** because:
- CSS animations, even `step-end`, keep the compositor active for the entire duration
- Chrome cannot optimize away a running animation even if the visual output is static
  between steps
- The animation runs INFINITELY (`infinite`)

**CRITICAL MITIGATION EXISTS:** The `CursorBlinkStateManager` (DomRenderer.ts:651-706)
stops the animation after 5 minutes of idle:

```ts
CURSOR_BLINK_IDLE_TIMEOUT = 5 * 60 * 1000  // Constants.ts:12
```

It adds `xterm-cursor-blink-idle` class which applies `animation: none !important;`.

**However, 5 minutes is a long time.** For the first 5 minutes after any keypress or
mouse click, the cursor blink animation drives continuous compositor frames. Every
`mousedown` event restarts the 5-minute timer (`DomRenderer.ts:104`).

### 12. Text Blink (SGR Blink attribute)

`TextBlinkStateManager.ts` runs a `setInterval` that toggles visibility of blink-attributed
text. This only activates when blinking cells exist in the viewport AND the viewport is
visible.

When active, it calls `_renderCallback()` on every interval tick, which fires
`onRequestRedraw` and triggers a full viewport re-render. This is a JS-driven render
cycle, much heavier than the CSS cursor blink.

**Idle impact: None unless terminal content has SGR blink attribute.** The manager
correctly disables the interval when no blinking cells are in viewport.

### 13. Row Rendering Strategy -- replaceChildren()

Every `renderRows()` call does `rowElement.replaceChildren(...spans)` for each dirty row.
This is a full DOM replacement, not a diff. For the DOM renderer this means:

- Old child nodes are detached (GC pressure)
- New span elements are created fresh every render
- Chrome must re-layout and re-paint the entire row

This is expected for the DOM renderer (it's the fallback, not the fast path), but it
means any render trigger causes significant compositor work.

### 14. Overview Ruler Canvas

`OverviewRulerRenderer.ts` creates a canvas element with class `xterm-decoration-overview-ruler`
positioned absolutely with z-index 8. It uses `requestAnimationFrame` for rendering but
only when decorations change. Not an idle concern.

### 15. Transform on Accessibility Tree

```css
.xterm .xterm-accessibility-tree > div {
    transform-origin: left;
}
```

`transform-origin` alone does not create a compositor layer or trigger redraws. It only
takes effect when a `transform` is applied. However, if the accessibility tree applies
transforms for scaling (to match cell dimensions), each row div in the accessibility
tree becomes a compositor layer.

---

## Summary: Idle Compositor Activity Sources

| Source | Duration | Severity | Can disable? |
|--------|----------|----------|-------------|
| Cursor blink CSS animation | 5 min after last input | HIGH | Set `cursorBlink: false` or reduce idle timeout |
| Scrollbar fade-out transition | 800ms after each scroll | LOW | Would need CSS override |
| Scrollbar fade-in transition | 100ms on scroll start | LOW | Would need CSS override |
| Selection overlay layer | Permanent when selected | NEGLIGIBLE | Clear selection |
| Empty selection container | Permanent | NEGLIGIBLE | Cannot remove |
| Scrollbar slider GPU layer | Permanent | NEGLIGIBLE | translate3d is cheap |
| Text blink interval | Continuous if blink cells | MEDIUM | No blink content = no cost |

### Key Missing Optimizations

1. **No `contain` on rows or viewport** -- every row change can trigger ancestor layout
2. **No `content-visibility: auto`** -- rows outside viewport still participate in layout
3. **Cursor blink idle timeout is too long** (5 minutes) for a screen-sharing scenario
4. **Scrollbar opacity transitions** could use `will-change: opacity` to avoid
   promoting the entire scrollbar subtree
5. **Empty selection/decoration containers** exist permanently as compositor layers

### Recommendations for Minimal Idle Compositor Activity

1. Set `cursorBlink: false` or patch `CURSOR_BLINK_IDLE_TIMEOUT` to ~3 seconds
2. Add `contain: content` to `.xterm-rows` via CSS override
3. Add `contain: strict` to row divs via CSS override
4. Consider `content-visibility: auto` on row divs for large terminals
5. Override scrollbar transition to `transition: none` if fade is not needed
6. The custom scrollbar's permanent GPU layer (translate3d) is fine -- the compositor
   knows it is static and skips it
