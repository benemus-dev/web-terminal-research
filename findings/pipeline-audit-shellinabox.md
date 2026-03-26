# Pipeline Audit: shellinabox

Source: `/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/`
Primary files: `vt100.jspp`, `shell_in_a_box.jspp`

---

## 1. DOM Mutations at Idle (Cursor Blink)

### Mechanism
shellinabox uses `setInterval` at 500ms for cursor animation. The handler is `animateCursor()`, set up at line 3200-3210:

```js
this.cursorInterval = setInterval(function(vt100) {
    return function() {
        vt100.animateCursor();
        vt100.checkComposedKeys();
    }
}(this), 500);
```

Each blink cycle changes the cursor's **className**:
```js
// line 3216-3221
if (this.blinkingCursor) {
    this.cursor.className = this.cursor.className == 'bright'
                            ? 'dim' : 'bright';
} else {
    this.cursor.className = 'bright';
}
```

Additionally, `checkComposedKeys()` is called on every blink tick -- this checks for pasted text in the hidden `<input>` element. This means the interval handler does more than just cursor animation.

### DOM mutations per blink cycle
- **1** `className` write on cursor element (toggle between `'bright'` and `'dim'`)
- **0** style property changes
- **0** node additions/removals
- Plus `checkComposedKeys()` may read `input.value` (not a DOM layout read, just a property read)

### Nodes touched: 1

### What pipeline stages fire?
- **Style Recalc**: YES. `className` change invalidates style for the cursor element. CSS rules `.bright` and `.dim` control the cursor appearance (opacity/visibility via CSS classes in the theme stylesheets).
- **Layout**: NO. The cursor is a `<pre id="cursor">` positioned absolutely via `style.left` and `style.top`. The className toggle changes visual properties (colors/opacity), not geometry.
- **Paint**: YES. Visual appearance change requires repaint of cursor region.
- **Composite**: YES. Compositor must update with the repainted cursor.

### Blink timing
500ms interval = 2 className changes/sec (1 Hz blink rate).

Unlike hterm, there is no "pause on input" mechanism. The cursor blinks continuously at 2 Hz regardless of user activity.

---

## 2. Forced Synchronous Layout

### Sites in vt100.jspp

| Line | Property | Preceded by DOM write? | Context |
|------|----------|----------------------|---------|
| 469 | `cursor.clientWidth` | Yes (style.cssText='') | **FORCED REFLOW** -- resets cursor CSS then reads clientWidth |
| 470 | `lineheight.clientHeight` | Same function | **FORCED REFLOW** -- reads after style reset |
| 516-523 | `box.offsetLeft/Top/Width/Height`, `key.offsetWidth/Height` | No | Keyboard layout positioning (8 reads in one function) |
| 974 | `cursor.clientWidth` | No | Initialization |
| 975 | `lineheight.clientHeight` | No | Initialization |
| 978 | `ieProbe.offsetTop` | No | IE detection |
| 991-995 | `container.offsetLeft/Top`, `parent.offsetLeft/Top` loop | No | Position calculation |
| 1000-1003 | `window.innerWidth`, `document.documentElement.clientWidth`, `document.body.clientWidth`, `container.offsetWidth` | No | Width calculation |
| 1115 | `getComputedStyle(elem, null)[style]` | Possibly | Style utility function |
| 1134 | `line.clientHeight` | Yes (line was just created) | **FORCED REFLOW** -- checks if newly created line has height |
| 1169 | `newCursor.clientHeight` | Yes (cursor just replaced) | **FORCED REFLOW** -- checks replacement cursor |
| 1193 | `cursorWidth > 0` check | No | Dimension guards |
| 1203-1206 | `container.clientHeight`, `window.innerHeight`, `document.documentElement.clientHeight`, `document.body.clientHeight` | No | Height calculation |
| 1265 | `scrollable.scrollTop` write | No | Scroll position set |
| 1282-1284 | `reconnectBtn.clientWidth/Height` | No | Button positioning |
| 1299-1302 | `curSizeBox.clientWidth/Height` | No | Size box positioning |
| 1352-1356 | `container.offsetLeft/Top` + parent loop | No | Event position calculation |
| 1373 | `scrollable.scrollTop` read | No | Mouse position to row |
| 1598-1604 | `cursor.clientWidth`, `console.offsetWidth` | Possibly | **POTENTIAL REFLOW** in `updateWidth()` |
| 1622 | `container.clientHeight` | No | `updateHeight()` |
| 1626-1628 | `window.innerHeight`, `document.documentElement.clientHeight`, `document.body.clientHeight` | No | `updateHeight()` |
| 1636 | `console.offsetHeight` | No | `updateNumScrollbackLines()` |
| 1880-1881 | `span.offsetTop`, `span.offsetParent.offsetTop` | Yes (text just written) | **FORCED REFLOW** in `putString()` |
| 1886-1897 | `span.offsetLeft`, `span.offsetWidth`, `span.nextSibling.offsetLeft` | Yes (text just written) | **FORCED REFLOW** in `putString()` |
| 1912-1913 | `space.offsetWidth`, `console.offsetLeft` | Yes (text content just set) | **FORCED REFLOW** in `putString()` |
| 1920 | `console.offsetTop` | Yes | **FORCED REFLOW** in `putString()` |
| 2078-2080 | `scrollable.scrollTop`, `scrollable.clientHeight` | No | `scrollBack()` |
| 2084-2086 | `scrollable.scrollTop`, `scrollable.clientHeight` | No | `scrollFore()` |
| 2239 | `scrollable.scrollTop` | No | Scroll position read |
| 2346 | `scrollable.scrollTop` write | No | Scroll position set |
| 2451-2452 | `container.offsetWidth/Height` | No | Keyboard overlay sizing |
| 2461-2467 | `kbd.offsetWidth/Height`, `container.offsetWidth/Height` | Yes (style.width/height just set) | **FORCED REFLOW** in keyboard overlay |
| 2476-2479 | `container.offsetWidth/Height`, `kbd.offsetWidth/Height` | Yes | **FORCED REFLOW** continuation |
| 2623-2624 | `container.offsetWidth/Height` | No | Menu sizing |
| 2629-2636 | `container.offsetWidth/Height`, `popup.clientWidth/Height` | No | Popup positioning |
| 2717 | `scrollable.scrollTop` write | No | After line feed |
| 2820 | `scrollable.scrollTop` write | No | After delete |

### Forced reflow sites: ~8-10

The most concerning one is in `putString()` (lines 1880-1920), which is the **core text rendering function**. After writing text content to spans, it immediately reads `offsetTop`, `offsetLeft`, `offsetWidth`, `offsetParent.offsetTop` to position the cursor. This forces a synchronous layout on EVERY text write operation.

### Idle impact
The `animateCursor()` function at line 3212-3223 only writes `className`. It does NOT read any layout properties. So at idle, forced reflows are **0/sec**.

However, `showCursor()` (called when making cursor visible) calls `putString()` which DOES force reflow. If cursor visibility toggles during blink, this would be a problem. Let's check: `animateCursor()` only changes `className`, NOT `visibility`. The `showCursor`/`hideCursor` functions change `style.visibility`, and are NOT called during blink. So at idle, **forced reflows = 0**.

---

## 3. The Render Path: DOM Mutations per Row

### Row structure
shellinabox uses a flat DOM structure:
- Each visible row is a `<div>` inside `<pre id="console">`
- Blank rows are `<pre>` elements containing `'\n'`
- Each attribute run within a row is a `<span>` with `className` (color class) and `style.cssText` (additional styling)

### Rendering one row (80 chars, 4 attribute runs of 20 chars) via `putString()`

The function at line 1701 does:

1. **Line creation** (if missing): `document.createElement('div')` + `style.height` set + `innerHTML = '<span></span>'` or `insertBefore`/`appendChild`
2. **Span traversal**: walks existing spans to find the write position
3. **Text insertion**: For each attribute run:
   - If same class/style as current span: modify `textContent` via `setTextContent()`
   - If different: split span, create new span, set `className`, set `style.cssText`, set text
4. **Span merging**: merges adjacent spans with matching className and style.cssText
5. **Whitespace pruning**: removes trailing whitespace spans
6. **Cursor positioning**: reads `offsetTop`, `offsetLeft`, `offsetWidth` to position cursor absolutely

**Estimated DOM mutations per row (fresh write, 80 chars, 4 attribute runs):**
- `createElement` (`div` + `span`s): 4-5
- `appendChild`/`insertBefore`: 4-5
- `textContent`/`setTextContent` writes: 4-8
- `className` sets: 4
- `style.cssText` sets: 4
- `style.height` set: 1 (on the `<div>`)
- `style.left`/`style.top` sets on cursor: 2
- `removeChild` (span merging/pruning): 0-4
- **Total DOM mutations: ~23-32**

### Read/write interleaving
**YES -- severe.** `putString()` interleaves DOM writes (text content, span creation) with DOM reads (offset properties for cursor positioning) **within the same function call**. Lines 1880-1920 write text, then immediately read `span.offsetTop`, `span.offsetLeft`, `span.offsetWidth`, `span.offsetParent.offsetTop` to calculate pixel positions for the cursor.

This means **every single character output forces a synchronous layout**. This is the single worst pattern in all three terminals.

### Row approach
shellinabox keeps ALL rows in the DOM (no virtual viewport). Rows are added/removed only via scrollback pruning. The `<pre id="console">` can contain thousands of child nodes.

---

## 4. Scroll Handling

### Scroll mechanism
shellinabox does NOT register an `onscroll` event handler. There is no `addEventListener('scroll', ...)` in the codebase.

Instead, scrolling is done imperatively:
- `scrollBack()` / `scrollFore()` (lines 2077-2091): read `scrollable.scrollTop` and `scrollable.clientHeight`, then write `scrollable.scrollTop`
- After adding/removing lines: `scrollable.scrollTop` is set directly (lines 1265, 2717, 2820)

### scrollTop writes per scroll action
- `scrollBack()`: 1 read of `scrollTop` + 1 read of `clientHeight` + 1 write of `scrollTop` = 3 accesses
- `scrollFore()`: same pattern = 3 accesses

### Layout thrashing during scroll
Moderate. `scrollBack()`/`scrollFore()` read `clientHeight` then write `scrollTop`. Since `clientHeight` is a layout-triggering property and `scrollTop` write changes scroll position, this is a read-before-write pattern (not thrashing). However, if anything wrote to the DOM before calling these functions, the `clientHeight` read would force reflow.

The lack of an onscroll handler means no additional processing happens in response to native scroll events. This is actually good -- no reflow chains triggered by scroll.

---

## 5. contentEditable Side Effects

### Setup
shellinabox does NOT use contentEditable on any visible terminal element. The terminal output is rendered in `<pre>` and `<div>` elements that are not editable.

Input is captured via a hidden `<input>` element:
```html
<input type="text" id="input" autocorrect="off" autocapitalize="off" />
```
(Line 919 in `vt100.jspp`)

### Chrome side effects
- **No contentEditable**: No editable state overhead on the terminal content
- **Hidden input**: `autocorrect="off"` and `autocapitalize="off"` are set, but `spellcheck` is NOT explicitly disabled. Chrome may run spellcheck on the hidden input's content, but since the input is hidden and periodically cleared, this has negligible impact.
- **No `input` events on terminal**: Since the terminal content is not contentEditable, Chrome does not fire input events on it.
- **No composition events on terminal**: IME events fire on the hidden `<input>`, not on terminal content.

### Idle impact
None. No contentEditable overhead at all on the visible terminal.

---

## 6. Idle Pipeline Runs per Second

### With cursor blink enabled (default: 500ms setInterval)

**Style Recalc**: 2/sec
- `className` toggle on cursor element fires every 500ms
- Each className change invalidates styles for the cursor element

**Layout**: 0/sec
- className toggle changes visual classes (`bright`/`dim`), not geometry
- Cursor is absolutely positioned, so className changes don't trigger layout

**Paint**: 2/sec
- Each visual appearance change triggers a repaint
- Scoped to the cursor element's bounding box

**Composite**: 2/sec
- Each paint triggers compositor update

### Additional idle cost: `checkComposedKeys()`
The setInterval also calls `checkComposedKeys()` every 500ms. This reads `input.value` on the hidden `<input>` element. This is a DOM property read but NOT a layout-triggering property. No additional pipeline cost.

### With cursor blink disabled

**Style Recalc**: 0/sec (the interval sets className to 'bright' and stops toggling, but the interval still runs)
**Layout**: 0/sec
**Paint**: 0/sec (className stays 'bright', no change)
**Composite**: 0/sec

Wait -- the setInterval still fires even with blink disabled (line 3219-3221: it just always sets 'bright'). This means:
- The JS callback runs 2/sec
- It writes `className = 'bright'` even if it's already 'bright'
- If Chrome detects no actual change (same className), no style recalc is triggered
- But the JS execution cost is still present (negligible)

So with blink disabled: effectively 0 pipeline runs/sec (Chrome optimizes away no-op className writes), but the setInterval timer still fires.

### Truly idle (cursor not visible / inactive)

When the cursor className is set to `'inactive'`:
```js
this.cursor.className = 'inactive';
```
The interval still runs but `animateCursor` is a no-op when `cursor.className == 'inactive'` and `inactive != undefined` (line 3212).

---

## Summary

| Metric | Value |
|--------|-------|
| Cursor blink mechanism | `className` toggle via `setInterval(500)` |
| DOM mutations per blink | 1 |
| Forced reflow sites (total) | ~8-10 |
| Forced reflows at idle | 0 |
| DOM mutations per row (80ch, 4 attr runs) | ~23-32 |
| Read/write interleaving in render | YES -- severe (offset reads in putString after writes) |
| Virtual viewport | NO -- all rows in DOM |
| Scroll handler | None (imperative scrollTop writes only) |
| Scroll forced reflows per action | 0-1 |
| contentEditable | NO (uses hidden `<input>` instead) |
| Idle Style Recalc/sec (blink on) | 2 |
| Idle Layout/sec (blink on) | 0 |
| Idle Paint/sec (blink on) | 2 |
| Idle Composite/sec (blink on) | 2 |
| Always-running timer | Yes (setInterval never cleared while connected) |
