# Pipeline Audit: hterm

Source: `/c/dev/web-terminal-research/libapps-mirror/hterm/js/`

---

## 1. DOM Mutations at Idle (Cursor Blink)

### Mechanism
hterm uses `setTimeout` to blink the cursor. The handler is `onCursorBlink_()` in `hterm_terminal.js:4142`.

Each blink cycle toggles a **single attribute** on the cursor `<div>`:
```js
this.cursorNode_.setAttribute('visible', 'true');  // or 'false'
```

The cursor node is a `<div>` created at line 1876:
```js
this.cursorNode_ = this.document_.createElement('div');
```

### DOM mutations per blink cycle
- **1** `setAttribute('visible', ...)` call -- toggles between `'true'` and `'false'`
- **0** classList changes
- **0** style property changes
- **0** node additions/removals

### Nodes touched: 1

### What pipeline stages fire?
- **Style Recalc**: YES. `setAttribute` on a node that has CSS selectors matching `[visible]` forces Chrome to recalculate styles for that element and its subtree. hterm's CSS uses attribute selectors like `div[cursor][visible="false"]` to control opacity.
- **Layout**: NO. The cursor is absolutely positioned with fixed dimensions. Toggling `visible` (which maps to `opacity` via CSS) does not change geometry.
- **Paint**: YES. The opacity change requires repainting the cursor's layer.
- **Composite**: YES. The compositor must composite the updated cursor layer.

### Blink timing
Default: `cursorBlinkCycle_ = [100, 100]` -- 100ms on, 100ms off = 5 Hz blink rate (10 attribute sets/sec).

A "pause blink" mechanism exists (`pauseCursorBlink_`): on user input, blinking pauses for 500ms, showing cursor solid. This temporarily reduces idle mutations to zero.

---

## 2. Forced Synchronous Layout

### Sites in production code (excluding tests)

| File | Line | Property | Preceded by DOM write? | Context |
|------|------|----------|----------------------|---------|
| `hterm_scrollport.js` | 987 | `screen_.getBoundingClientRect()` | No | `getScreenSize()` -- geometry query |
| `hterm_scrollport.js` | 1022 | `screen_.getBoundingClientRect().width` | No | `getScrollbarX()` |
| `hterm_scrollport.js` | 1233 | `node.getBoundingClientRect().height` | No | `syncRowNodesDimensions_()` -- loop over top fold nodes |
| `hterm_scrollport.js` | 1242 | `screen_.offsetLeft` | Yes (style write at 1238) | **FORCED REFLOW** in `syncRowNodesDimensions_()` -- writes `rowNodes_.style.width` then reads `screen_.offsetLeft` |
| `hterm_scrollport.js` | 1244 | `screen_.offsetTop` | Yes (style write at 1238-1240) | **FORCED REFLOW** -- same function, writes multiple styles then reads offsetTop |
| `hterm_scrollport.js` | 1253-1254 | `screen_.getBoundingClientRect().width`, `screen_.clientWidth` | No | `syncScrollbarWidth_()` |
| `hterm_scrollport.js` | 1706-1708 | `scrollArea_.getBoundingClientRect().height`, `screen_.getBoundingClientRect().height` | No | `getScrollMax_()` |
| `hterm_scrollport.js` | 1735, 1739, 1770 | `screen_.scrollTop` | No | scroll position read/write |
| `hterm_scrollport.js` | 1795 | `getScreenSize()` (calls getBoundingClientRect) | No | `onScroll_()` handler |
| `hterm_scrollport.js` | 1841, 1851, 1853 | `screen_.scrollTop` | No | `onScrollWheel_()` |
| `hterm_scrollport.js` | 1888 | `screen_.getBoundingClientRect()` | No | touch handler |
| `hterm_terminal.js` | 3566 | `img.clientHeight` | No | image sizing |
| `hterm_contextmenu.js` | 123, 128 | `body.getBoundingClientRect()`, `element_.getBoundingClientRect()` | No | context menu positioning |
| `hterm_notifications.js` | 91 | `container_.getBoundingClientRect()` | No | notification positioning |

### Forced reflow sites: 2

Both are in `syncRowNodesDimensions_()` -- the function writes `rowNodes_.style.width`, `rowNodes_.style.height`, then reads `screen_.offsetLeft` and `screen_.offsetTop`. This is a **write-then-read** pattern that forces synchronous layout.

### Idle impact
These reflow sites only fire during **resize** or **redraw**, not during idle. The cursor blink path does not touch any layout-triggering properties. At idle, forced reflow count is **0**.

---

## 3. The Render Path: DOM Mutations per Row

### Row structure
Each row is an `<x-row>` custom element. Text content is stored as either:
- **Text nodes** (for default-styled text) -- `createTextNode()` in `hterm_text_attributes.js:204`
- **`<span>` elements** (for styled text) -- `createElement('span')` in `hterm_text_attributes.js:209`

### Rendering one row (80 chars, mixed attributes -- say 4 attribute runs of 20 chars each)

For overwrite mode (the common case via `hterm.Screen.overwriteString`):

1. **Node splitting**: If the cursor is in the middle of an existing text node, it gets split via `splitNode_()`:
   - 1 `cloneNode(false)`
   - 2 `textContent` writes
   - 1 `insertBefore`
   - Conditionally 1 `remove()`

2. **Container creation** per attribute run: `createContainer()`:
   - Default text: 1 `createTextNode()`
   - Styled text: 1 `createElement('span')` + N `style.X = ...` sets (color, backgroundColor, fontWeight, etc.) + optional classList additions

3. **Insertion**: 1 `insertBefore` or implicit via `textContent` write per container

4. **Cleanup**: removal of overwritten nodes via `removeChild`

**Estimated DOM mutations for 80 chars, 4 attribute runs:**
- `createElement`/`createTextNode`: 4 (one per attribute run)
- `appendChild`/`insertBefore`: 4-6
- `style` property sets: 4-12 (depending on attributes: color, bg, bold, underline, etc.)
- `textContent` writes: 4-8
- `removeChild` calls: 0-4 (removing overwritten content)
- `className`/`classList` changes: 0-4
- **Total DOM mutations: ~16-38**

### Batching behavior
hterm does **NOT** interleave reads and writes during row rendering. The render path is purely writes (DOM mutations), then any layout reads happen later in `syncRowNodesDimensions_()` / `syncCursorPosition_()`. This is a good pattern -- it avoids forced reflows during the render pass.

The ScrollPort uses a **virtual viewport** -- only visible rows (~25 for a typical terminal) are in the DOM. Row nodes are cached (`previousRowNodeCache_`, `currentRowNodeCache_`) and reused across redraws.

---

## 4. Scroll Handling

### onScroll handler
`hterm_scrollport.js:1794` -- `onScroll_(e)`:
1. Calls `getScreenSize()` which reads `screen_.getBoundingClientRect()` -- **layout read**
2. Compares with cached `lastScreenWidth_` / `lastScreenHeight_`
3. If dimensions changed (resize during scroll), calls `resize()` and returns
4. Otherwise calls `redraw_()` directly

### redraw_() scroll path
`redraw_()` at line 1315:
1. `syncScrollHeight()` -- writes `scrollArea_.style.height`
2. `getTopRowIndex()` -- reads `screen_.scrollTop` (after style write = **potential forced reflow**)
3. `drawVisibleRows_()` -- inserts/removes row nodes
4. `syncRowNodesDimensions_()` -- writes styles, reads offsets (**forced reflow**)

### scrollTop writes
- `scrollRowToTop()`: 1 `scrollTop` write
- `onScrollWheel_()`: 1 `scrollTop` write
- `scrollBack/Forward`: 1 `scrollTop` write each

Each `scrollTop` write triggers the native `scroll` event, which triggers `onScroll_()`, which triggers `redraw_()`. So one scroll action = 1 `scrollTop` write + 1 full redraw cycle.

### Layout thrashing during scroll
YES, mild. `redraw_()` writes `scrollArea_.style.height` then reads `screen_.scrollTop` in `getTopRowIndex()`. Then `syncRowNodesDimensions_()` writes multiple style properties then reads `offsetLeft`/`offsetTop`. This is 2 forced reflows per scroll event.

---

## 5. contentEditable Side Effects

### Setup
`hterm_scrollport.js:618`:
```js
this.screen_.setAttribute('contenteditable', 'true');
this.screen_.setAttribute('spellcheck', 'false');
this.screen_.setAttribute('autocomplete', 'off');
this.screen_.setAttribute('autocorrect', 'off');
this.screen_.setAttribute('autocapitalize', 'none');
```

The x-screen element is contentEditable, but with spellcheck, autocomplete, autocorrect, and autocapitalize all explicitly disabled.

### Chrome side effects
- **Spellcheck**: Disabled via `spellcheck="false"`. No forced layout from spellcheck.
- **Autocorrect/Autocomplete**: Disabled. No UI activation.
- **`input` events**: Chrome fires `input` events on contentEditable elements when the DOM changes programmatically from user-initiated context (e.g., paste). hterm's own programmatic DOM changes (from terminal output) do NOT trigger `input` events because they're not user-initiated.
- **`compositionstart`/`compositionend`**: Only fire during actual IME composition (CJK input). Do NOT fire at idle.
- **Caret rendering**: hterm sets `caret-color: transparent` (line 644 in `decorate`) -- Chrome's native caret is invisible. This means Chrome still maintains its internal caret position tracking but does not paint a native caret.
- **Selection tracking**: Chrome tracks the selection on contentEditable elements. `syncSelectionCaret()` is called to collapse selection to cursor position. This can trigger style recalc but does NOT trigger layout.

### Idle impact from contentEditable
Minimal. With spellcheck off and caret transparent, the contentEditable attribute does not cause additional idle pipeline activity. The only ongoing cost is Chrome's internal bookkeeping of the editable state.

---

## 6. Idle Pipeline Runs per Second

### With cursor blink enabled (default: 100ms/100ms cycle)

**Style Recalc**: ~10/sec
- Each cursor blink fires `setAttribute('visible', ...)` which invalidates styles
- 5 Hz blink = 10 attribute changes/sec (on + off)

**Layout**: ~0/sec
- Cursor attribute toggle does not change geometry
- No forced reflows in the blink path

**Paint**: ~10/sec
- Each style change that affects visual appearance (opacity/visibility) triggers a paint
- Scoped to the cursor's paint region only (small area)

**Composite**: ~10/sec
- Each paint triggers compositor update
- If the cursor is on its own compositor layer (likely due to being an absolutely positioned div), this is a cheap composite-only operation

### With cursor blink disabled

**Style Recalc**: 0/sec
**Layout**: 0/sec
**Paint**: 0/sec
**Composite**: 0/sec

hterm is completely quiescent at idle with blink disabled. No timers, no observers, no polling.

---

## Summary

| Metric | Value |
|--------|-------|
| Cursor blink mechanism | `setAttribute('visible')` via setTimeout |
| DOM mutations per blink | 1 |
| Forced reflow sites (total) | 2 (both in `syncRowNodesDimensions_`) |
| Forced reflows at idle | 0 |
| DOM mutations per row (80ch, 4 attr runs) | ~16-38 |
| Read/write interleaving in render | No (writes-only, then reads) |
| Scroll forced reflows per event | 2 |
| contentEditable | Yes, with spellcheck/autocorrect OFF |
| Idle Style Recalc/sec (blink on) | ~10 |
| Idle Layout/sec (blink on) | 0 |
| Idle Paint/sec (blink on) | ~10 |
| Idle Composite/sec (blink on) | ~10 |
