# Chrome Rendering Pipeline Audit: WeTTY

## Architecture Summary
- **Framework:** None (vanilla JS/TS, no React/Preact)
- **Terminal:** xterm.js (no explicit WebGL/Canvas addon -- uses default DOM renderer)
- **Transport:** Socket.IO (WebSocket with HTTP long-polling fallback)
- **Components:** Vanilla DOM, no component tree
- **Source:** `/c/dev/web-terminal-research/wetty/src/`

---

## 1. Forced Synchronous Layout

### Hits Found

**None in WeTTY's own source code.** No calls to `offsetWidth`, `offsetHeight`, `getBoundingClientRect`, `getComputedStyle`, `scrollIntoView`, `scrollTop`, `scrollHeight`, `clientWidth`, or `clientHeight` in the WeTTY client source.

### FitAddon (xterm.js internal)
`fitAddon.fit()` is called on:
- `term.ts:30` -- `resizeTerm()` method, called on:
  - `wetty.ts:35` -- Initial connect
  - `wetty.ts:65` -- `login` event
  - `wetty.ts:33` / `term.ts:223` -- `window.resize` / `window.onresize`

FitAddon internally reads `clientWidth`/`clientHeight` and writes terminal dimensions. Single read-write cycle, not thrashing.

**Note:** WeTTY has TWO resize handlers registered:
1. `wetty.ts:33`: `window.addEventListener('resize', onResize(term))`
2. `term.ts:223`: `window.onresize = function onResize() { term.resizeTerm(); }`

The second overwrites any previous `window.onresize`. Both call `term.resizeTerm()`. On a resize event, `resizeTerm()` fires twice (once from addEventListener, once from onresize assignment). However, `resizeTerm()` also calls `this.refresh(0, this.rows - 1)` before fit, which forces a full terminal re-render.

**Double resize handling is a bug** -- causes 2x layout reads per resize event.

---

## 2. React Reconciliation

**Not applicable.** WeTTY uses no framework. Pure vanilla DOM manipulation.

---

## 3. DOM Mutations During Idle

### Cursor Blink
WeTTY's default xterm config (`xterm_defaults.js:10`): **`cursorBlink: false`**

With cursor blink disabled, there are no periodic DOM mutations at idle.

If the user enables cursor blink via config, xterm.js DOM renderer would toggle cursor element visibility ~2/sec, causing:
- 2 style recalcs/sec
- 2 paints/sec

### Status Indicators
No connection status polling. No heartbeat DOM writes. The disconnect overlay (`disconnect.ts`) is a simple `style.display` toggle on disconnect -- not periodic.

### FontAwesome `dom.watch()`
`wetty.ts:19`: `dom.watch()` -- This installs a MutationObserver on the document that watches for `<i>` elements with FontAwesome classes and replaces them with SVG. After initial page load, this observer fires on every DOM mutation but typically does nothing (no new FA icons being added). **Minor overhead: MutationObserver callback runs on every DOM change but exits early.**

### Other Idle Activity
- `term.ts:100`: `_.debounce(() => simulateBackspace(), 100)` -- only fires on CTRL key sequences, not idle
- No `setInterval` in client code
- No polling loops

**Idle DOM mutations: 0/sec (cursorBlink=false default)**

---

## 4. Scroll Event Handling

No `onScroll` handlers in WeTTY source. xterm.js handles scrolling internally. No `scrollIntoView` calls.

---

## 5. Resize Handling

**Flow:** `window.resize` -> `resizeTerm()` -> `this.refresh(0, rows-1)` -> `fitAddon.fit()` -> `socket.emit('resize')`

`term.ts:28-31`:
```
resizeTerm(): void {
    this.refresh(0, this.rows - 1);     // Force full terminal re-render
    if (this.shouldFitTerm) this.fitAddon.fit();  // Read container size, compute new dims
    this.socket.emit('resize', { cols: this.cols, rows: this.rows });
}
```

The `this.refresh(0, this.rows - 1)` call forces xterm.js to re-render ALL rows before the fit. With DOM renderer, this means replacing/updating all row DOM elements, then FitAddon reads container dimensions (forced reflow since DOM was just mutated).

**This is a forced reflow on every resize:** DOM write (refresh) followed by layout read (fit).

**Bug: double invocation** (two resize handlers). So per resize event: 2 full refreshes + 2 fit() calls = 2 forced reflows.

**Total layout reads per resize: ~4** (2 invocations x 2 reads from FitAddon internals)

---

## 6. WebSocket Message Processing

**Each Socket.IO `data` event triggers `term.write()` directly:**

```
socket.on('data', (data: string) => {
    const remainingData = fileDownloader.buffer(data);  // Check for file download markers
    // ... flow control ...
    term.write(remainingData);
});
```

Socket.IO delivers messages individually (not coalesced at transport level like raw WS). Each `data` event calls `term.write()`. xterm.js internally batches write calls and renders on the next animation frame.

**No immediate DOM mutations per Socket.IO message.** xterm.js batches internally.

However, Socket.IO adds overhead compared to raw WebSocket:
- JSON message framing
- Event name parsing
- Acknowledgment handling

---

## 7. Estimated Pipeline Runs/Second

### Idle State (cursorBlink=false, no output)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 0 | No DOM changes |
| Layout | 0 | No layout-triggering operations |
| Paint | 0 | Nothing to repaint |
| Composite | 0 | No compositing needed |

WeTTY with default config is **completely silent** at idle -- 0 pipeline runs.

### Idle State (cursorBlink=true, user-enabled)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~2 | Cursor element visibility toggle |
| Layout | 0 | Visibility change doesn't trigger layout |
| Paint | ~2 | Repaint cursor region |
| Composite | ~2 | Recomposite affected layer |

### During Terminal Output

WeTTY uses DOM renderer by default (no WebGL/Canvas addon loaded). Each xterm.js render frame mutates DOM row elements.

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 60 | DOM renderer: span style changes per frame |
| Layout | 0-60 | Usually 0 (text replacement), up to 60 if rows added/removed |
| Paint | 60 | Repaint changed character cells |
| Composite | 60 | Recomposite terminal layer |

---

## Key Findings

1. **WeTTY is very clean at idle** (cursorBlink=false default) -- literally 0 pipeline activity.
2. **DOM renderer is the main weakness.** No WebGL or Canvas addon means all terminal rendering goes through Chrome's full rendering pipeline (Style -> Layout -> Paint -> Composite) during output.
3. **Double resize handler bug** causes redundant work on window resize -- `resizeTerm()` is called twice, each doing a full refresh + fit.
4. **`term.refresh(0, rows-1)` before `fit()`** is a forced reflow pattern -- writes all row DOM then reads container dimensions.
5. **FontAwesome `dom.watch()`** adds a persistent MutationObserver that fires on every DOM mutation, though it exits early when no FA icons need processing.
6. **Socket.IO overhead** vs raw WebSocket is minimal for rendering, but adds per-message parsing cost.
7. **No framework overhead** -- vanilla JS means no reconciliation, no virtual DOM diffing.
