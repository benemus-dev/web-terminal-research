# Chrome Rendering Pipeline Audit: xterm.js

Source: xterm.js repository at `/c/dev/web-terminal-research/xterm.js/`
Covers: `src/browser/` and `addons/` (excluding test/spec/demo/node_modules)
Date: 2026-03-26

---

## 1. DOM Mutations

### 1a. DOM mutations at IDLE (cursor blink, no terminal output)

**DOM renderer (default fallback):**
Cursor blink is handled entirely by CSS animation (`@keyframes`). The DomRenderer injects CSS like:
```
animation: blink_block_1 1s step-end infinite;
```
This means **zero DOM mutations per cursor blink cycle**. Chrome's compositor handles the CSS animation without going through Style Recalc > Layout > Paint. The main thread is not involved.

The only DOM mutation related to cursor idle is the **idle timeout** (5 minutes): after `CURSOR_BLINK_IDLE_TIMEOUT = 300000ms`, `CursorBlinkStateManager._stopBlinkingDueToIdle()` adds `classList.add('xterm-cursor-blink-idle')` which sets `animation: none !important`. This is 1 classList mutation per 5-minute idle period.

- File: `src/browser/renderer/dom/DomRenderer.ts` lines 651-706
- File: `src/browser/renderer/shared/Constants.ts` line 12

**WebGL renderer (addon-webgl):**
Cursor blink uses `setInterval(600ms)` + `requestAnimationFrame` to toggle `isCursorVisible` and call the render callback. Each blink cycle:
- 1 RAF callback
- 0 DOM mutations (it re-renders via WebGL `gl.drawArrays`, not DOM)
- But the RAF callback itself triggers a Chrome rendering pipeline run (Composite at minimum)

Blink rate: 600ms interval = ~1.67 blinks/sec = **~1.67 RAF callbacks/sec at idle** (WebGL path)

- File: `addons/addon-webgl/src/CursorBlinkStateManager.ts` lines 114-131

**TextBlinkStateManager (SGR blink attribute):**
Only active when viewport contains cells with the blink attribute (`\e[5m`). Uses `setInterval(blinkIntervalDuration)` and fires a full viewport redraw callback. Inactive by default (no blinking text = no interval).

- File: `src/browser/renderer/shared/TextBlinkStateManager.ts`

### 1b. DOM mutations during ACTIVE rendering (terminal output)

Per `renderRows(start, end)` call in DomRenderer:

For each row `y` in `[start, end]`:
1. `rowElement.replaceChildren(...)` -- 1 call, replaces all children at once
2. Inside `DomRendererRowFactory.createRow()`, for each row of 80 characters:
   - `document.createElement('span')` -- varies by cell merging (typically 1-5 spans for a uniform row, up to 80 for fully heterogeneous)
   - `charElement.textContent = text` -- 1 per span (deferred until merge boundary)
   - `charElement.className = classes.join(' ')` -- 1 per span (only if classes exist)
   - `charElement.style.letterSpacing` -- 1 per span (only if spacing differs from default)
   - `charElement.setAttribute('style', ...)` via `_addStyle()` -- for RGB colors, underline colors, minimum contrast
   - `charElement.style.textDecoration` -- only during link hover

**Estimated DOM mutations per row update (typical 80-char uniform row):**
- 1 `replaceChildren`
- ~2-5 `createElement` calls
- ~2-5 `textContent` sets
- ~2-5 `className` sets
- 0-2 `style` attribute sets (RGB colors)
- Total: **~7-17 DOM mutations per row**

**Estimated DOM mutations per row update (worst case, every cell different):**
- 1 `replaceChildren`
- ~80 `createElement` calls
- ~80 `textContent` sets
- ~80 `className` sets
- 0-80 `style` sets
- Total: **~241-321 DOM mutations per row**

### 1c. Full inventory of DOM mutation sites in src/browser/

| Mutation type | File | Line(s) | When |
|---|---|---|---|
| `replaceChildren` | DomRenderer.ts | 516, 541, 615 | Active render |
| `replaceChildren` | DomRenderer.ts | 383 | Selection change |
| `textContent =` | DomRendererRowFactory.ts | 230, 474, 487 | Active render |
| `createElement` | DomRendererRowFactory.ts | 185, 232 | Active render |
| `createElement` | DomRenderer.ts | 78, 83, 158, 178, 337, 472 | Init/resize |
| `appendChild` | DomRenderer.ts | 97-98, 159, 179, 338, 436 | Init/resize/selection |
| `removeChild` | DomRenderer.ts | 344 | Resize (shrink) |
| `classList.add` | DomRenderer.ts | 370, 702 | Focus/idle timeout |
| `classList.remove` | DomRenderer.ts | 364, 670, 682 | Blur/blink restart |
| `style.*` | DomRenderer.ts | 80, 150-154, 171-173, 324 | Init/resize |
| `textContent =` | DomRenderer.ts | 169, 310 | Resize/theme change |
| `textContent =` | AccessibilityManager.ts | 154, 161, 188, 191, 209 | Active render (a11y) |
| `setAttribute` | AccessibilityManager.ts | 194-195 | Active render (a11y) |
| `style.transform` | AccessibilityManager.ts | 429, 436 | Active render (a11y) |
| `getBoundingClientRect` | AccessibilityManager.ts | 430 | Active render (a11y) -- **FORCED REFLOW** |
| `textContent =` | CompositionHelper.ts | 82, 94 | IME composition |
| `style.*` | CompositionHelper.ts | various | IME composition |
| `textContent =` | Viewport.ts | 87 | Theme change |
| `style.backgroundColor` | Viewport.ts | 77-78 | Theme change |
| `style.*` | fastDomNode.ts | 29,38,47,56,65,74,95,104,115 | Scrollbar updates |

### 1d. Addon DOM mutations

| Mutation type | File | When |
|---|---|---|
| `appendChild` / `removeChild` | WebglRenderer.ts:150,160 | Init/dispose |
| `style.width/height` | WebglRenderer.ts:200-205 | Resize |
| `classList.add` | BaseRenderLayer.ts:43 | Init |
| `style.*` | BaseRenderLayer.ts:44,112-113 | Init/resize |
| `classList.add` | DecorationManager.ts:98-105 | Search result decoration |
| `style.*` | ImageRenderer.ts:341-359 | Image addon init |
| `style.fontFeatureSettings` | LigaturesAddon.ts:42,51 | Ligatures init/dispose |

---

## 2. Forced Synchronous Layout (Layout Thrashing)

### 2a. `offsetWidth` / `offsetHeight` reads

| Location | File:Line | Context |
|---|---|---|
| `this._measureElement.offsetWidth` | CharSizeService.ts:98 | Font measurement |
| `this._measureElement.offsetHeight` | CharSizeService.ts:98 | Font measurement |

**When:** Only on `measure()` calls -- triggered by font option changes (`fontFamily`, `fontSize`) and when terminal becomes visible after being hidden. NOT called at idle. NOT called per frame.

**Forced reflow?** YES. Lines 94-95 set `style.fontFamily` and `style.fontSize` immediately before reading `offsetWidth`/`offsetHeight` on line 98. This is a classic write-then-read forced reflow. But it only happens on font changes, not during normal rendering.

However: `TextMetricsMeasureStrategy` (the preferred path) uses `OffscreenCanvas.measureText()` which does NOT trigger layout. The DOM fallback (`DomMeasureStrategy`) is only used when `OffscreenCanvas` + `fontBoundingBoxAscent` are not available.

### 2b. `getBoundingClientRect()` reads

| Location | File:Line | Context |
|---|---|---|
| `element.getBoundingClientRect()` | AccessibilityManager.ts:430 | Row width alignment for a11y |
| `screenElement.getBoundingClientRect()` | Clipboard.ts:69 | Clipboard paste positioning |
| `this._compositionView.getBoundingClientRect()` | CompositionHelper.ts:271 | IME overlay positioning |
| `element.getBoundingClientRect()` | Mouse.ts:7 | Mouse coordinate translation |
| `domNode.getBoundingClientRect()` | Dom.ts:79 | Generic DOM position helper |
| `iframeElement.getBoundingClientRect()` | mouseEvent.ts:88 | Scrollbar iframe handling |

**AccessibilityManager forced reflow (the worst one):**
In `_alignRowWidth()` (line 428-437), it does:
1. `element.style.transform = ''` -- DOM WRITE
2. `element.getBoundingClientRect().width` -- DOM READ (forced reflow!)
3. `element.style.transform = \`scaleX(...)\`` -- DOM WRITE

This is a textbook forced reflow pattern. It fires during `_renderRows()` which is called on every render for every visible row. However, the AccessibilityManager has its own `RenderDebouncer` so it batches to one RAF. For N rows visible, this causes **N forced reflows per render frame** in the a11y layer.

**Mouse.ts:** `getBoundingClientRect()` on every mouse event (click, move during selection). Not at idle.

### 2c. `getComputedStyle()` reads

| Location | File:Line | Context |
|---|---|---|
| `window.getComputedStyle(element)` | Mouse.ts:8 | Mouse coordinate calculation |
| `getComputedStyle(el)` | FitAddon.ts:32-33 | Terminal fitting (resize) |

**When:** Mouse.ts fires on mouse events only. FitAddon fires only when `fit()` or `proposeDimensions()` is called (user-initiated resize). Neither fires at idle.

### 2d. `scrollTop`/`scrollHeight` reads

The scrollable system (`src/browser/scrollable/`) uses its own internal `Scrollable` class that tracks scroll position mathematically (not via DOM). The `verticalScrollbar.ts` line 79 does `target.scrollTop = scrollPosition` which is a WRITE (setting scroll position on a hidden element). No DOM scroll property reads that would force layout.

---

## 3. requestAnimationFrame Callback Work

### 3a. RAF sites

| Location | File:Line | What it does |
|---|---|---|
| `RenderDebouncer._innerRefresh()` | RenderDebouncer.ts:35,53 | Calls `_renderCallback(start, end)` which calls `DomRenderer.renderRows()` or `WebglRenderer.renderRows()` |
| `SelectionService._refresh()` | SelectionService.ts:282 | Refreshes selection rendering |
| `OverviewRulerRenderer` | OverviewRulerRenderer.ts:208 | Redraws overview ruler canvas |
| `scheduleAtNextAnimationFrame` | Dom.ts:161 | Generic RAF queue (used by Viewport smooth scroll) |
| `CursorBlinkStateManager` (WebGL) | CursorBlinkStateManager.ts:76,108,127 | Cursor blink render callback |

### 3b. Does the RAF callback cause forced reflows?

**RenderDebouncer -> DomRenderer.renderRows():** Does only DOM writes (replaceChildren, textContent, className, style). Does NOT read layout properties after writing. **No forced reflow.**

**RenderDebouncer -> WebglRenderer.renderRows():** Does zero DOM mutations. All rendering via WebGL calls. **No forced reflow.**

**SelectionService._refresh():** Calls `handleSelectionChanged` which does `replaceChildren()` then creates new selection div elements with style sets. Does NOT read layout. **No forced reflow.**

**AccessibilityManager (via its own RenderDebouncer):** As noted above, `_alignRowWidth` does write-read-write per row. **YES, forced reflow, N times per render.**

---

## 4. Virtual DOM / Diffing

xterm.js is vanilla TypeScript with no virtual DOM, no React, no framework reconciliation.

However, the **WebGL renderer's `RenderModel`** (`addons/addon-webgl/src/RenderModel.ts`) acts as a diff model. It stores the previously-rendered state of every cell and only sends changed cells to the GPU. This is conceptually similar to virtual DOM diffing but operates on a flat typed array, not a DOM tree. It does not touch the DOM at all.

The **DOM renderer** does NOT diff. It calls `rowElement.replaceChildren(...)` with a freshly-built list of spans for every dirty row. Every render of a row tears down and rebuilds its entire DOM subtree.

---

## 5. Scroll Event Handlers

**No `addEventListener('scroll', ...)` or `onscroll` handlers found anywhere in src/ or addons/.**

The scrollable system is entirely custom (ported from VS Code's `scrollableElement`). Scroll input comes from:
- `wheel` events on the scrollable element container
- Pointer drag on the scrollbar slider
- Programmatic `setScrollPosition()` calls

The `_handleScroll` in `Viewport.ts:204` fires from the internal `Scrollable`'s `onScroll` event (not a DOM scroll event). It does simple math (division, rounding) and fires `_onRequestScrollLines`. **No layout reads, no DOM writes.**

---

## 6. Resize Handling

### 6a. DomRenderer.handleResize()

1. `_refreshRowElements()` -- adds/removes row divs (createElement + appendChild/removeChild)
2. `_updateDimensions()` -- sets `style.width`, `style.height`, `style.lineHeight`, `style.overflow` on every row element (4 style writes x N rows), plus style writes on screenElement and selectionContainer
3. `handleSelectionChanged()` -- rebuilds selection overlays

**Forced reflow during resize?** No. All operations are DOM writes. No layout property reads interleaved with writes.

### 6b. WebglRenderer.handleResize()

1. Sets `canvas.width`, `canvas.height` (canvas dimension attributes)
2. Sets `canvas.style.width`, `canvas.style.height` (CSS)
3. Sets `screenElement.style.width`, `screenElement.style.height`
4. WebGL viewport/uniform updates
5. Triggers synchronous full redraw

**Forced reflow?** No. All writes, no reads.

### 6c. FitAddon.proposeDimensions()

Calls `getComputedStyle()` on parent element and terminal element. This forces layout if there are pending style changes. However, `fit()` is only called by the application (user-initiated), not by xterm.js internally.

### 6d. CharSizeService.measure()

Called on font changes and when terminal becomes visible. Uses `offsetWidth`/`offsetHeight` after setting font styles. **Forced reflow**, but only on font change events (rare).

---

## 7. MutationObserver Callbacks

**No MutationObserver usage found anywhere in src/ or addons/.** Zero occurrences.

---

## 8. Font Loading / Measurement

### 8a. TextMetricsMeasureStrategy (preferred)

Uses `OffscreenCanvas` + `ctx.measureText('W')`. **Zero layout impact.** No DOM involvement at all.

- File: `src/browser/services/CharSizeService.ts` lines 104-127

### 8b. DomMeasureStrategy (fallback)

Creates a hidden `<span>` with 32 'W' characters, reads `offsetWidth` and `offsetHeight`.

**When called:**
- On construction (once)
- On `fontFamily` or `fontSize` option change
- When terminal becomes visible (IntersectionObserver fires)

**NOT called periodically. NOT called at idle.**

### 8c. WidthCache (DomRenderer only)

`src/browser/renderer/dom/WidthCache.ts` measures individual character widths using a canvas `measureText()` API (NOT DOM layout). Results are cached. Called during `createRow()` for each character to determine `letter-spacing`. **No layout impact.**

---

## 9. The Render Path: DOM Mutations Per Frame

### 9a. DomRenderer: 1 row of 80 identical characters (best case)

Row factory merges all 80 chars into 1 span:
- 1 `createElement('span')`
- 1 `textContent = "AAAA...A"` (80 chars)
- 1 `className = ""`  (no classes for plain text)
- 1 `replaceChildren(span)`
- **Total: 4 DOM mutations**

### 9b. DomRenderer: 1 row of 80 characters, alternating colors (worst realistic)

Each color change forces a new span. With 2 colors alternating:
- ~40 `createElement('span')`
- ~40 `textContent` sets
- ~40 `className` sets
- ~40 `setAttribute('style', ...)` for RGB colors
- 1 `replaceChildren(...40 spans)`
- **Total: ~161 DOM mutations**

### 9c. DomRenderer: Full screen refresh (24 rows x 80 cols, uniform)

- 24 x 4 = **~96 DOM mutations**

### 9d. DomRenderer: Full screen refresh (24 rows, heterogeneous)

- 24 x ~161 = **~3,864 DOM mutations**

### 9e. WebGL renderer: any content

- **0 DOM mutations** per frame (all rendering via WebGL draw calls)
- Canvas composite happens at browser level

---

## 10. Batch vs Interleaved DOM Writes

### DomRenderer: BATCHED (no interleaving)

The render path is:
1. `DomRendererRowFactory.createRow()` builds an array of `HTMLSpanElement[]` entirely in memory (creates elements, sets textContent, sets className, sets style attributes)
2. These elements are NOT attached to the document during construction
3. `rowElement.replaceChildren(...elements)` attaches them all at once in a single DOM operation

**This is correct batching.** The browser only needs to run Style Recalc + Layout + Paint once after `replaceChildren`, not per-element.

### AccessibilityManager: INTERLEAVED (forced reflow per row)

For each row in `_renderRows()`:
1. `element.textContent = lineData` -- WRITE
2. `element.setAttribute('aria-posinset', ...)` -- WRITE
3. `element.setAttribute('aria-setsize', ...)` -- WRITE
4. `_alignRowWidth(element)`:
   - `element.style.transform = ''` -- WRITE
   - `element.getBoundingClientRect().width` -- READ (forces layout!)
   - `element.style.transform = \`scaleX(...)\`` -- WRITE
5. Next row: repeat

This is **N forced reflows per render frame** where N = number of visible rows. Each `getBoundingClientRect()` forces Chrome to flush all pending style/layout changes synchronously.

### FastDomNode (scrollbar): GUARDED

`FastDomNode` caches the last-set value and skips the DOM write if unchanged. This prevents redundant style mutations but does not batch across multiple nodes.

---

## Summary: Rendering Pipeline Runs

### At IDLE (focused terminal, cursor blinking, no output)

**DOM renderer:**
- Cursor blink: CSS animation (compositor-only, zero main thread involvement)
- No timers, no RAF callbacks, no DOM mutations
- **Estimated pipeline runs: 0/sec on main thread** (compositor handles CSS animation)
- After 5 minutes idle: 1 classList mutation, then truly zero

**WebGL renderer:**
- Cursor blink: setInterval(600ms) -> RAF -> WebGL render
- ~1.67 RAF callbacks/sec
- 0 DOM mutations, but RAF triggers Composite
- **Estimated pipeline runs: ~1.67/sec** (Composite only, no Layout/Paint)
- After 5 minutes idle: cursor stops blinking, drops to **0/sec**

### At ACTIVE (receiving terminal output)

**DOM renderer:**
- RenderDebouncer coalesces to 1 RAF per frame
- DOM mutations: ~4-161 per row, ~96-3864 per full screen
- AccessibilityManager adds N forced reflows per render
- **Estimated pipeline runs: 1/frame** (but each frame can be expensive due to DOM rebuild)

**WebGL renderer:**
- RenderDebouncer coalesces to 1 RAF per frame
- 0 DOM mutations (WebGL draw calls only)
- AccessibilityManager still adds N forced reflows if enabled
- **Estimated pipeline runs: 1/frame** (lightweight -- just Composite)

### On RESIZE

- DomRenderer: ~4N+10 DOM mutations (N = rows), 0 forced reflows
- WebGL: ~6 DOM mutations (canvas + screen element sizing)
- FitAddon: 2 `getComputedStyle` calls (forced layout if pending changes)
- CharSizeService: 1 forced reflow (offsetWidth/offsetHeight after style write)

---

## The Critical Path: What's Most Expensive

### 1. AccessibilityManager forced reflows (active rendering)
**Impact: HIGH** -- N forced synchronous layouts per render frame (N = visible rows, typically 24-50). Each `getBoundingClientRect()` in `_alignRowWidth()` forces Chrome to compute layout for the entire document. This is the single most expensive operation in xterm.js's rendering pipeline.
- File: `src/browser/AccessibilityManager.ts` lines 428-437

### 2. DomRenderer full row rebuild (active rendering)
**Impact: MEDIUM-HIGH** -- Every dirty row has its entire DOM subtree torn down and rebuilt via `replaceChildren()`. For a full-screen refresh with heterogeneous content, this is ~3,864 DOM mutations in a single frame. Chrome must then Style Recalc + Layout + Paint all of it.
- File: `src/browser/renderer/dom/DomRenderer.ts` line 541

### 3. CharSizeService DomMeasureStrategy forced reflow (font change)
**Impact: LOW** -- Only on font changes. Single forced reflow. The preferred `TextMetricsMeasureStrategy` avoids this entirely.
- File: `src/browser/services/CharSizeService.ts` line 98

### 4. WebGL cursor blink RAF (idle)
**Impact: LOW** -- 1.67 RAF/sec at idle, but each only triggers a WebGL Composite (no Layout, no Paint, no DOM). Stops after 5 minutes.

### For this project (building a terminal): Key takeaways

1. **CSS animations for cursor blink = zero idle cost** (DOM renderer's approach is ideal)
2. **WebGL renderer = zero DOM mutations during active rendering** (ideal for performance)
3. **AccessibilityManager is the hidden performance killer** -- disable or defer `_alignRowWidth` if a11y is not needed
4. **No virtual DOM diffing means full row rebuild cost** in DOM renderer -- the WebGL renderer's `RenderModel` diff approach is far superior
5. **No scroll event handlers on DOM** -- the custom scrollable avoids Chrome's scroll-related forced layouts entirely
6. **No MutationObservers** -- no surprise layout flushes from observer callbacks
7. **RenderDebouncer correctly coalesces** to max 1 render per animation frame
