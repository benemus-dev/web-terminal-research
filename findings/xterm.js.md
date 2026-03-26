# xterm.js - Rendering Pipeline Analysis

**Version:** 6.0.0
**Repository:** https://github.com/xtermjs/xterm.js
**License:** MIT

---

## 1. Tech Stack

### Package Info
- **Name:** `@xterm/xterm`
- **Version:** 6.0.0
- **Runtime dependencies:** None (zero production deps)
- **Build tools:** TypeScript (`tsgo`), esbuild (bundling), webpack (packaging)
- **Test:** mocha (unit), Playwright (integration across Chromium/Firefox/WebKit)
- **Dev dependencies:** ~30 packages (eslint, esbuild, webpack, ts-loader, playwright, etc.)

### File Counts
| Type | Count |
|------|-------|
| `.ts` | 331 |
| `.js` | 26 |
| `.html` | 4 |
| `.css` | 2 |

### Project Structure
- `src/browser/` - Browser-specific rendering, input, services
- `src/common/` - Shared logic (buffer, parser, options)
- `src/headless/` - Headless terminal (no rendering)
- `addons/` - 13 addon packages (webgl, fit, search, image, ligatures, web-fonts, etc.)

---

## 2. Rendering Pipeline

### Classification: **HYBRID** (DOM default, WebGL optional addon)

### Architecture
xterm.js uses a **pluggable renderer** behind the `IRenderer` interface. The core ships with a **DOM renderer** as default/fallback. A **WebGL2 renderer** is available as an optional addon (`@xterm/addon-webgl`). There is no Canvas 2D renderer in current v6 (the old canvas renderer was removed; canvas is only used internally for font measurement and texture atlas rasterization).

### DOM Renderer (Default) - `DomRenderer.ts`
- Creates a `<div class="xterm-rows">` container
- Each terminal row is a `<div>` element
- Characters within a row are `<span>` elements with text set via `textContent`
- Adjacent cells with identical attributes are **merged into a single span** (optimization)
- Colors applied via CSS classes (`xterm-fg-N`, `xterm-bg-N`) for palette colors, inline `style` for RGB/truecolor
- All text styling (bold, italic, underline, strikethrough, dim) uses CSS classes
- Selection rendered as separate absolute-positioned `<div>` overlays

### WebGL Renderer (Addon) - `WebglRenderer.ts`
- Creates an `HTMLCanvasElement` with `webgl2` context (`antialias: false, depth: false`)
- Requires WebGL2 (no WebGL1 fallback)
- Uses a **texture atlas** (`TextureAtlas.ts`): glyphs rasterized to a Canvas 2D temp surface via `fillText()`, then uploaded to WebGL textures
- Two main GPU renderers:
  - `RectangleRenderer` - draws cell backgrounds and cursor
  - `GlyphRenderer` - draws text glyphs from the texture atlas
- `RenderModel` tracks per-cell state (fg, bg, ext attributes) for dirty detection
- Link underlines rendered via a separate Canvas 2D `BaseRenderLayer`
- Falls back to DOM renderer on WebGL context loss (after 3s timeout)

### No Canvas 2D Renderer
The old `CanvasRenderer` was removed. Canvas 2D is only used for:
- Font measurement (`CharSizeService` uses `OffscreenCanvas` + `measureText`)
- Width caching (`WidthCache` uses canvas for character width measurement)
- WebGL texture atlas glyph rasterization (internal to addon-webgl)

---

## 3. Update/Flush Mechanism

### Classification: **RAF_BATCHED** with dirty-row tracking

### Flow
1. **Trigger:** Any content change calls `RenderService.refreshRows(start, end)`
2. **Debounce:** `RenderDebouncer.refresh()` coalesces row ranges and schedules a single `requestAnimationFrame`
3. **RAF callback:** `_innerRefresh()` fires, calling `_renderRows(start, end)` on the active renderer
4. **Render:** Only the dirty row range `[start, end]` is re-rendered

### Key Details
- Row ranges are accumulated: `_rowStart = min(existing, new)`, `_rowEnd = max(existing, new)`
- If a RAF is already pending, new refresh calls just update the range (no second RAF)
- `_needsFullRefresh` flag defers rendering while terminal is not visible (IntersectionObserver)
- Synchronized output mode (DEC 2026): buffers all row updates, flushes on mode exit or 1000ms timeout
- Synchronous render path exists (`sync: true`) for resize to avoid flicker

### Visibility Optimization
- Uses `IntersectionObserver` to detect terminal visibility
- When hidden: sets `_isPaused = true`, accumulates `_needsFullRefresh`
- When visible again: flushes pending resize tasks and does a full refresh

---

## 4. Deep Dive

### Renderer Interface (`IRenderer` in `src/browser/renderer/shared/Types.ts`)
```typescript
interface IRenderer extends IDisposable {
  readonly dimensions: IRenderDimensions;
  readonly onRequestRedraw: IEvent<IRequestRedrawEvent>;
  renderRows(start: number, end: number): void;
  handleDevicePixelRatioChange(): void;
  handleResize(cols: number, rows: number): void;
  handleSelectionChanged(...): void;
  handleCursorMove(): void;
  clear(): void;
  clearTextureAtlas?(): void;
}
```

### RenderService (`src/browser/services/RenderService.ts`)
- Owns the `_renderer: MutableDisposable<IRenderer>` (swappable at runtime)
- Owns the `RenderDebouncer` (RAF gate)
- Listens for buffer resize, option changes, decoration changes, DPR changes
- Supports `setRenderer()` to hot-swap between DOM and WebGL

### DOM Renderer Details
- `DomRenderer.renderRows()`: iterates rows, calls `DomRendererRowFactory.createRow()` which returns an array of `HTMLSpanElement`
- Row elements updated via `rowElement.replaceChildren(...spans)` (full row replacement per dirty row)
- Character merging: consecutive cells with same fg/bg/ext/spacing are merged into one `<span>` with concatenated text
- Width correction: per-character `letter-spacing` CSS applied based on `WidthCache` measurements to prevent glyph stacking errors

### WebGL Texture Atlas (`addons/addon-webgl/src/TextureAtlas.ts`)
- Glyphs rasterized to a temporary `HTMLCanvasElement` using `CanvasRenderingContext2D.fillText()`
- Rasterized glyph images uploaded to WebGL textures via `texImage2D`/`texSubImage2D`
- Atlas uses multiple pages (each up to 4096x4096 px), starts at 512x512 and grows
- Glyph cache keyed by `(code, bg, fg, ext)` 4-key map
- Custom box-drawing/powerline glyphs rendered procedurally (`CustomGlyphRasterizer`)
- Warm-up: pre-rasterizes common ASCII glyphs on idle via `IdleTaskQueue`

### Dirty Tracking in WebGL
- `RenderModel` stores per-cell attribute data in flat arrays
- `renderRows()` compares current buffer state against model, only updates changed cells
- Changed cells get their glyph looked up in the atlas and drawn via `GlyphRenderer`

---

## 5. Font Rendering

### Default Font
- `fontFamily: 'monospace'` (default in `OptionsService`)
- `fontSize: 15` (px)
- `fontWeight: 'normal'`, `fontWeightBold: 'bold'`
- `lineHeight: 1.0`, `letterSpacing: 0`

### Font Measurement
Two strategies with automatic fallback:

1. **TextMetricsMeasureStrategy** (preferred): Uses `OffscreenCanvas` + `measureText()` with `fontBoundingBoxAscent`/`fontBoundingBoxDescent`
2. **DomMeasureStrategy** (fallback): Creates a hidden `<span>` with 32 repeated 'W' characters, measures via `offsetWidth`/`offsetHeight`

### Font Smoothing / Antialiasing
- **No explicit font smoothing settings** anywhere in the codebase (no `-webkit-font-smoothing`, no `textRendering`, no `imageSmoothingEnabled`)
- DOM renderer: relies on browser-native text rendering (subpixel AA by default)
- WebGL renderer: glyph rasterization uses Canvas 2D `fillText()` which gets browser's default AA, then the rasterized bitmap is uploaded to a texture - so the AA is "baked in" at rasterization time as grayscale alpha

### Width Cache (`WidthCache.ts`)
- Measures character widths using canvas `measureText()` (32x repeated chars for precision)
- Flat `Float32Array` cache for codepoints 0-255 (~4x faster than Map)
- Holey `Map<string, number>` cache for everything else
- Separate caches for regular/bold/italic/bold+italic variants

---

## 6. Strain Assessment

### Overall: **Low** (DOM default), **Medium** (WebGL addon)

### DOM Renderer (Default) - LOW STRAIN

| Factor | Assessment | Details |
|--------|-----------|---------|
| **Text rendering** | Good | Browser-native `<span>` text with subpixel antialiasing |
| **Cursor blink** | Good | CSS `@keyframes` animation (`step-end`), no JS redraw loop. Stops after 5min idle. |
| **Scrolling** | Good | Custom `SmoothScrollableElement` with `overflow-y: scroll`, but manages its own scrollbar DOM (VS Code-derived). Not raw `overflow:auto`. |
| **Colors** | Good | CSS classes for palette colors (`xterm-fg-N`), inline `style` for RGB truecolor |
| **Row updates** | Acceptable | Full `replaceChildren()` per dirty row (not per-character DOM diffing), but only dirty rows are touched |

**Cursor blink detail (DOM):** Uses CSS `@keyframes` with `animation: blink_block_N 1s step-end infinite`. A `CursorBlinkStateManager` adds/removes the `xterm-cursor-blink-idle` class via JS `setTimeout` (5min idle timeout) to stop blinking after inactivity. The blink animation itself is pure CSS, zero JS redraws.

### WebGL Renderer (Addon) - MEDIUM STRAIN

| Factor | Assessment | Details |
|--------|-----------|---------|
| **Text rendering** | Medium | Glyphs pre-rasterized via Canvas 2D `fillText()`, then displayed as WebGL textured quads. Gets grayscale AA at rasterization time, no subpixel AA on display. |
| **Cursor blink** | Poor | JS `setInterval(600ms)` toggles `isCursorVisible`, triggers RAF-based redraw of the entire viewport. GPU redraw every 600ms while blinking. |
| **Scrolling** | Good | Same scrollbar infrastructure as DOM renderer |
| **Colors** | Good | Cell backgrounds drawn as colored rectangles via `RectangleRenderer` shader |
| **Text blink** | Medium | `TextBlinkStateManager` uses `setInterval(blinkIntervalDuration)` + full render callback. Only active when blinking cells exist in viewport. |

**Cursor blink detail (WebGL):** `CursorBlinkStateManager` uses `setInterval` at 600ms, toggling `isCursorVisible` and calling `_renderCallback()` which triggers a full `requestAnimationFrame`-gated viewport redraw. Has idle timeout (5min) that pauses the interval. This is worse than the DOM renderer's CSS-only approach.

### Text Blink (Both Renderers)
- `TextBlinkStateManager` uses `setInterval(blinkIntervalDuration)` when cells with blink attribute exist
- Default `blinkIntervalDuration: 0` (disabled) - so no strain unless explicitly enabled
- When active: triggers full viewport redraw on each interval tick

---

## 7. Key Files

| File | Role |
|------|------|
| `src/browser/services/RenderService.ts` | Central render orchestrator - owns renderer, debouncer, RAF scheduling, visibility management |
| `src/browser/RenderDebouncer.ts` | RAF-gated row range coalescing (the core "one RAF per frame" mechanism) |
| `src/browser/renderer/dom/DomRenderer.ts` | Default DOM renderer - creates div/span tree, CSS cursor blink, theme injection |
| `src/browser/renderer/dom/DomRendererRowFactory.ts` | Builds span elements per row with attribute merging, color resolution, cursor styling |
| `addons/addon-webgl/src/WebglRenderer.ts` | WebGL2 renderer - texture atlas, glyph/rectangle shaders, JS cursor blink |
| `addons/addon-webgl/src/TextureAtlas.ts` | Glyph rasterization via Canvas 2D fillText, multi-page texture management |
| `src/browser/renderer/shared/Types.ts` | `IRenderer` interface definition |

---

## Summary

xterm.js v6 uses a **pluggable renderer architecture**. The **default DOM renderer** is low-strain: browser-native text in `<span>` elements with subpixel AA, CSS `@keyframes` cursor blink, and CSS-class-based coloring. The **WebGL addon** trades subpixel AA for GPU-accelerated rendering: glyphs are pre-rasterized via Canvas 2D `fillText()` into a texture atlas, then rendered as textured quads. Its cursor blink is JS-interval-driven with full viewport redraws. Both renderers share a RAF-gated dirty-row batching system via `RenderDebouncer` that coalesces updates into a single animation frame.
