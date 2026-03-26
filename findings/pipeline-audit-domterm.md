# Pipeline Audit: DomTerm

Source: `/c/dev/web-terminal-research/DomTerm/hlib/`

---

## 1. DOM Mutations at Idle (Cursor Blink)

### Mechanism
DomTerm uses **two separate blink systems**:

#### A. Caret blink (cursor) -- CSS Animation
The primary cursor blink uses a pure **CSS `@keyframes` animation** defined in `domterm-default.css:196`:
```css
@keyframes blinking-caret {
    0% { }
    30% { border-right: inherit; margin-right: inherit;
          background-color: inherit; color: inherit; text-decoration: inherit }
    100% {}
}
```
Applied via:
```css
div.domterm-active span[caret="blinking-block"][std="caret"] {
    animation: blinking-caret 1.5s steps(1) 0s var(--caret-blink-count);
}
```
Where `--caret-blink-count` defaults to 20 (finite iteration count -- blink stops after 30 seconds).

**DOM mutations per blink cycle: 0**. CSS animations are handled entirely by Chrome's compositor. No JavaScript runs, no DOM mutations, no `setAttribute` calls. This is the optimal approach.

#### B. Text blink (for blinking text attribute) -- classList toggle via setTimeout
`terminal.js:594-627` -- `startBlinkTimer()`:
```js
this.topNode.classList.remove('blinking-hide');
// ... setTimeout ...
this.topNode.classList.add('blinking-hide');
```
This toggles `classList` on the top-level container to hide/show all text with the `blink` decoration attribute. Default timing: 700ms show / 300ms hide = 1 Hz.

**Only active when the terminal has text with the blink attribute**. The function checks for `.term-style[text-decoration~="blink"]` elements and stops if none exist. At normal idle (no blinking text), this does not run.

### DOM mutations per blink cycle (cursor)
- **0** -- pure CSS animation, no JS involvement

### DOM mutations per blink cycle (text blink, when active)
- **1** classList toggle on `topNode`
- Affects all descendants with blink attribute via CSS cascade

### Nodes touched at idle: 0 (cursor blink is CSS-only)

### What pipeline stages fire from cursor blink?
- **Style Recalc**: YES, but handled by compositor animation tick, not main thread style recalc
- **Layout**: NO. CSS animation changes `background-color`, `color`, `border-right` -- none affect geometry (except `margin-right: inherit` in the hide phase, which could trigger layout if the inherited value differs)
- **Paint**: YES. Color/background changes require repaint.
- **Composite**: YES. But since CSS animations can be compositor-driven, this may not require main thread involvement at all.

---

## 2. Forced Synchronous Layout

### Sites in terminal.js and related files

| File | Line | Property | Preceded by DOM write? | Context |
|------|------|----------|----------------------|---------|
| `terminal.js` | 1037-1038 | `_vspacer.getBoundingClientRect().top`, `buffers.scrollTop` | No | `_dataHeight()` |
| `terminal.js` | 1497 | `lineStart.offsetHeight` | No | scroll calculation |
| `terminal.js` | 1499 | `lineStart.offsetTop` | No | scroll calculation |
| `terminal.js` | 1512 | `_vspacer.offsetTop` | No | spacer height check |
| `terminal.js` | 1730 | `getComputedStyle(cur.parentNode)['white-space']` | Possibly | **Potential forced reflow** -- depends on prior DOM writes in same call |
| `terminal.js` | 3578 | `topNode.offsetLeft` | No | initialization only |
| `terminal.js` | 3838 | `buffers.offsetWidth` | No | popup positioning |
| `terminal.js` | 3847 | `getComputedStyle(popup)['border-color']` | Yes (style writes) | **FORCED REFLOW** in popup code |
| `terminal.js` | 3892 | `topNode.clientWidth` | No | margin calculation |
| `terminal.js` | 3940 | `topNode.offsetHeight`, `widget.offsetHeight` | No | widget positioning |
| `terminal.js` | 4185 | `buffers.getBoundingClientRect()` | No | `measureWindow()` |
| `terminal.js` | 4192 | `ruler.getBoundingClientRect()` | No | character measurement |
| `terminal.js` | 4195 | `_wrapDummy.getBoundingClientRect().width` | No | margin measurement |
| `terminal.js` | 4214 | `topNode.offsetHeight`, `topNode.clientHeight` | No | available height |
| `terminal.js` | 4218 | `initial.getBoundingClientRect().width` | No | available width |
| `terminal.js` | 4233 | Multiple offset/client reads | No | debug logging |
| `terminal.js` | 4250 | `getComputedStyle(zoomTop)['zoom']` | No | zoom calculation |
| `terminal.js` | 4515 | `initial.getBoundingClientRect().right` | No | mouse handling |
| `terminal.js` | 4529 | `buffers.scrollTop`, `_vspacer.offsetTop` | No | scroll check |
| `terminal.js` | 6240-6241 | `root.getBoundingClientRect()`, `table.getBoundingClientRect()` | No | floating table header |
| `terminal.js` | 6868 | `last.getBoundingClientRect().bottom`, `buffers.scrollTop` | No | `_scrollNeeded()` |
| `terminal.js` | 6876 | `last.getBoundingClientRect().bottom` | No | `_scrolledAtBottom()` |
| `terminal.js` | 6881, 6888 | `buffers.scrollTop` read + write | No | `_scrollIfNeeded()` |
| `terminal.js` | 6893-6908 | Multiple `getBoundingClientRect()` | Yes (style writes) | **FORCED REFLOW** in `adjustFocusCaretStyle()` -- writes `.style.top`, `.style.left` etc. and then the function is called again on next frame potentially |
| `domterm-layout.js` | 68-69 | `body.offsetWidth`, `body.offsetHeight`, `element.offsetLeft`, `element.offsetTop` | No | layout sizing |
| `domterm-layout.js` | 209 | `getComputedStyle(document.body)['zoom']` | No | zoom calculation |
| `domterm-layout.js` | 214-215 | `sizeElement.offsetWidth`, `sizeElement.offsetHeight` | No | size calculation |
| `domterm-overlays.js` | 95 | `parent.clientWidth` | No | overlay sizing |
| `domterm-overlays.js` | 226 | `getComputedStyle(header)['zoom']` | No | drag handling |
| `domterm-overlays.js` | 294-300 | `widget.offsetHeight`, `topNode.offsetHeight` | No | info display position |
| `domterm-utils.js` | 1078, 1145, 1200, 1212, 1226 | `getBoundingClientRect()` | No | position utilities |
| `domterm-parser.js` | 1845, 1876 | `getComputedStyle()` | Possibly | color/style queries during parsing |

### Forced reflow sites: ~3-4
- `adjustFocusCaretStyle()` -- writes style properties to caret mark/line nodes, then reads BoundingClientRect
- `getComputedStyle` in popup border-color check (line 3847)
- Potentially in `_breakVisibleLines` + subsequent offset reads

### Idle impact
`adjustFocusCaretStyle()` runs inside `_updateDisplay()`, which is triggered by `requestAnimationFrame`. It fires on scroll events and after output. At true idle (no output, no scroll), it does NOT run. The RAF callback is only scheduled on demand (`requestUpdateDisplay()` at line 6678-6682).

**Forced reflows at idle: 0** (assuming no text blink or active output).

---

## 3. The Render Path: DOM Mutations per Row

### Row structure
DomTerm uses a persistent DOM tree. Rows are `<pre>` (or `<div class="domterm-pre">`) elements created by `_createPreNode()` (line 3379). Text spans within rows are created by `_createSpanNode()` (line 3408).

Line breaks are `<span>` elements with class `"hard"` or `"soft"`, created by `_createLineNode()` (line 3416).

### Key difference from hterm
DomTerm does NOT use a virtual viewport. **ALL rows remain in the DOM at all times.** The scrolling is handled by native CSS `overflow: scroll` on the `buffers` div. This means the DOM can grow unbounded.

### Rendering one row (80 chars, 4 attribute runs of 20 chars each)

The parsing happens in `domterm-parser.js`, which builds DOM nodes incrementally:

1. For each attribute change: `_createSpanNode()` = 1 `createElement('span')` + 1 `classList.add` or `className` set
2. Text insertion: `createTextNode(str)` + `insertBefore(text, current)` or `appendChild`
3. Style application via `term-style` class spans with inline CSS

**Estimated DOM mutations per row:**
- `createElement`: 4-5 (1 per attribute run + line node)
- `createTextNode`: 4-5
- `appendChild`/`insertBefore`: 8-10 (parent + text child for each span)
- `className`/`classList` changes: 4-5
- `style` property sets: 0-8 (colors/decorations)
- **Total DOM mutations: ~20-33**

### Batching behavior
DomTerm uses `requestAnimationFrame` for display updates (`_updateDisplay` at line 433). The RAF callback does:
1. `_restoreInputLine()` -- DOM writes
2. `_breakVisibleLines()` -- DOM reads and writes (line wrapping)
3. `_checkSpacer()` -- DOM writes
4. `_scrollIfNeeded()` -- DOM read (`getBoundingClientRect`) + DOM write (`scrollTop`)
5. `adjustFocusCaretStyle()` -- DOM reads (`getBoundingClientRect`) + DOM writes (style)

This sequence **does interleave reads and writes**, which can cause forced reflows within a single frame. Specifically, `_breakVisibleLines()` likely reads layout to determine wrap points, then modifies the DOM, then `_scrollIfNeeded()` reads layout again.

---

## 4. Scroll Handling

### Scroll event handler
`terminal.js:410-417`:
```js
this.buffers.addEventListener('scroll', (e) => {
    dt.requestUpdateDisplay();
    const adjust = dt.buffers._adjustFloatingTableHead;
    if (adjust) adjust();
}, false);
```

### What happens on scroll
1. `requestUpdateDisplay()` schedules RAF
2. RAF fires `_updateDisplay()` which:
   - Reads `_vspacer.getBoundingClientRect()` (layout read)
   - Reads `buffers.scrollTop` (layout read)
   - Writes `buffers.scrollTop` (if auto-scroll needed)
   - Reads multiple `getBoundingClientRect()` for caret positioning (layout reads)
   - Writes multiple `.style.top`, `.style.left` properties (style writes)

### scrollTop writes
- `_scrollIfNeeded()`: 1 `scrollTop` write
- `scrollToCaret()`: 1 `scrollTop` write
- These are guarded by conditions, so typically 0-1 per scroll event

### Layout thrashing during scroll
YES. `adjustFocusCaretStyle()` (line 6892-6910) reads `getBoundingClientRect()` on selection and topNode, then writes style properties to caret elements. If this triggers additional reflow queries downstream, it creates a read-write-read cycle.

However, since this is all within a single RAF callback, it's at most 1-2 forced reflows per frame, not per-element thrashing.

---

## 5. contentEditable Side Effects

### Setup
`terminal.js:3570`:
```js
topNode.contentEditable = true;
```

But the buffers div is explicitly set to non-editable:
```js
this.buffers.contentEditable = false;  // line 408
```

And spellcheck is disabled:
```js
topNode.spellcheck = false;  // line 387
// later: topNode.spellcheck = false;  // line 4049
```

The caretNode is separately made contentEditable:
```js
caretNode.contentEditable = true;  // line 4060
```

### Chrome side effects
- **Spellcheck**: Disabled at topNode level. However, `topNode.contentEditable = true` with `spellcheck = false` still causes Chrome to maintain internal editing state. Since buffers is `contentEditable = false`, the terminal output area is not editable, reducing side effects.
- **`input` events**: May fire on the editable regions (topNode, caretNode, miniBuffer). DomTerm handles these for IME support.
- **`compositionstart`/`compositionend`**: Will fire during IME input on the contentEditable nodes. Not at idle.
- **Native caret**: `caret-color: transparent` is set in CSS (`domterm-core.css:89, 101`), so Chrome's native caret is invisible.
- **Selection change listener**: DomTerm registers `selectionchangeListener` (line 3622) which fires on any selection change in the document. This can cause additional processing but only when the user interacts.

### Idle impact from contentEditable
Low. With spellcheck off and caret transparent, the main cost is Chrome's internal editable state tracking. The `buffers.contentEditable = false` means the terminal content area is not treated as editable, which significantly reduces Chrome's editing overhead.

However, `topNode.contentEditable = true` means the top-level container IS editable. Chrome will maintain undo history and other editing state for this container. This is a non-zero but generally small idle cost.

---

## 6. Idle Pipeline Runs per Second

### With cursor blink (CSS animation, default 1.5s cycle, 20 iterations)

**Style Recalc**: ~0/sec (main thread)
- CSS animations are compositor-driven in Chrome
- The `steps(1)` timing function means the animation has discrete jumps, not continuous interpolation
- Chrome handles this entirely on the compositor thread

**Layout**: 0/sec
- No layout-triggering properties change during CSS animation blink
- Exception: `margin-right: inherit` in the blink keyframe COULD trigger layout if the inherited margin differs from current, but in practice the caret span has no meaningful margin context

**Paint**: ~0.67/sec (compositor)
- The 1.5s animation cycle means ~0.67 paint operations per second
- These are compositor-driven paints, not main-thread paints
- Scoped to the caret element only

**Composite**: ~0.67/sec
- Compositor tick for the animation frame
- Very cheap -- no main thread involvement

### After caret blink stops (30 seconds -- 20 iterations x 1.5s)

**Style Recalc**: 0/sec
**Layout**: 0/sec
**Paint**: 0/sec
**Composite**: 0/sec

DomTerm becomes completely quiescent. No timers, no RAF, no observers. The RAF is only scheduled on demand.

### With text blink active (rare -- requires blink text attribute)

**Style Recalc**: 1/sec (classList toggle on topNode)
**Layout**: 0-1/sec (classList change could trigger layout if CSS rules change geometry)
**Paint**: 1/sec
**Composite**: 1/sec

---

## Summary

| Metric | Value |
|--------|-------|
| Cursor blink mechanism | CSS `@keyframes` animation (compositor-driven) |
| DOM mutations per blink | 0 |
| Forced reflow sites (total) | ~3-4 |
| Forced reflows at idle | 0 |
| DOM mutations per row (80ch, 4 attr runs) | ~20-33 |
| Read/write interleaving in render | Yes (`_updateDisplay` interleaves) |
| Virtual viewport | NO -- all rows in DOM |
| Scroll forced reflows per event | 1-2 (in RAF callback) |
| contentEditable | topNode=true, buffers=false, spellcheck=false |
| Idle Style Recalc/sec | ~0 (compositor-driven) |
| Idle Layout/sec | 0 |
| Idle Paint/sec | ~0.67 (compositor, first 30s only) |
| Idle Composite/sec | ~0.67 (first 30s only, then 0) |
