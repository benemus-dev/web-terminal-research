# Chrome Rendering Pipeline Audit: ttyd

## Architecture Summary
- **Framework:** Preact (minimal virtual DOM)
- **Terminal:** xterm.js with WebGL addon (default), Canvas fallback, DOM fallback
- **Transport:** Raw WebSocket (binary ArrayBuffer)
- **Components:** App -> Terminal -> Xterm class (vanilla JS, not Preact component)
- **Source:** `/c/dev/web-terminal-research/ttyd/html/src/`

---

## 1. Forced Synchronous Layout

### Hits Found

| Location | Property | Context | Severity |
|----------|----------|---------|----------|
| `overlay.ts:53` | `getBoundingClientRect()` | Centering overlay after DOM append | MEDIUM |
| `overlay.ts:54` | `getBoundingClientRect()` | Reading overlay size after append | MEDIUM |

### Analysis

**overlay.ts:49-57 -- Forced reflow pattern:**
```
overlayNode.parentNode || terminal.element.appendChild(overlayNode);  // DOM WRITE
const divSize = terminal.element.getBoundingClientRect();              // LAYOUT READ (forced reflow)
const overlaySize = overlayNode.getBoundingClientRect();               // LAYOUT READ
overlayNode.style.top = ...                                           // DOM WRITE
overlayNode.style.left = ...                                          // DOM WRITE
```
This is a classic write-then-read forced reflow. The `appendChild` invalidates layout, then `getBoundingClientRect` forces synchronous layout computation. However, this only runs on:
- Resize overlay (user drags window edge)
- Copy overlay (user selects text)
- Connection status overlay (disconnect/reconnect)

**Not in a loop.** Not idle. Only triggered by user action or connection events.

### FitAddon (xterm.js internal)
`fitAddon.fit()` is called on:
- `xterm/index.ts:169` -- Initial open (once)
- `xterm/index.ts:202` -- `window.resize` event
- `xterm/index.ts:465` -- Font preference change

FitAddon internally reads `clientWidth`/`clientHeight` of the terminal container and writes terminal dimensions. This is a single read-write cycle per resize, not thrashing.

---

## 2. React Reconciliation

**Not applicable.** ttyd uses Preact with only 2 components:
- `App` -- static, never re-renders after mount
- `Terminal` -- only re-renders on modal show/hide (file upload dialog)

The Xterm class is pure vanilla JS. No virtual DOM overhead during terminal operation.

---

## 3. DOM Mutations During Idle

### Cursor Blink
Handled entirely by xterm.js internally. With WebGL renderer (ttyd default), cursor blink is a GPU shader operation -- no DOM mutations. With Canvas renderer, it redraws to canvas -- no DOM mutations. Only the DOM renderer (fallback) would mutate DOM for cursor blink.

**ttyd default config: WebGL renderer = 0 DOM mutations for cursor blink.**

### Status Indicators
No connection status polling. No heartbeat DOM writes. The overlay is only shown on connect/disconnect events, not periodically.

### Other Idle Activity
- No `setInterval` found in ttyd source
- No polling loops
- No periodic DOM writes

**Idle DOM mutations: 0/sec (WebGL/Canvas), ~2/sec cursor blink (DOM renderer only)**

---

## 4. Scroll Event Handling

No `onScroll` handlers in ttyd source. xterm.js handles scrolling internally via its own viewport. No `scrollIntoView` calls in ttyd code.

---

## 5. Resize Handling

**Flow:** `window.resize` -> `fitAddon.fit()` -> sends resize command to server

FitAddon.fit() internally:
1. Reads container `clientWidth`/`clientHeight` (layout read)
2. Computes new cols/rows
3. Calls `terminal.resize(cols, rows)` which triggers xterm.js internal re-render

This is a single layout read followed by writes. **No interleaved read-write thrashing.**

The resize also triggers `terminal.onResize` callback which calls `overlayAddon.showOverlay()` -- that's the forced reflow pattern in overlay.ts (see section 1). So resize does 1 forced reflow for the overlay centering.

**Total layout reads per resize: 3** (1 from FitAddon + 2 from overlay getBoundingClientRect)

---

## 6. WebSocket Message Processing

**Each WS message goes directly to xterm.js `terminal.write()`.**

```
onSocketData -> switch(cmd) -> this.writeFunc(data) -> terminal.write(data)
```

xterm.js `terminal.write()` uses its own internal write buffer and batches rendering. It does NOT render on every write call -- it coalesces writes and renders on the next animation frame internally.

**Flow control:** ttyd implements backpressure via PAUSE/RESUME commands when pending writes exceed `highWater` (10). The flow control callback fires when xterm finishes rendering a chunk:
```
terminal.write(data, () => { this.pending--; if (pending < lowWater) RESUME; });
```

**No immediate DOM mutations per WS message.** xterm.js batches internally.

---

## 7. Estimated Pipeline Runs/Second

### Idle State (cursor blinking, no output)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 0 | WebGL renderer: no DOM changes |
| Layout | 0 | No layout-triggering operations |
| Paint | 0 | WebGL renders to canvas, no paint |
| Composite | ~2 | Cursor blink via WebGL canvas composited by DWM |

### With DOM Renderer (fallback)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~2 | Cursor blink toggles visibility |
| Layout | 0 | Cursor blink is visibility/opacity only |
| Paint | ~2 | Repaint cursor area |
| Composite | ~2 | Composited with page |

### During Terminal Output (e.g., `ls -R`)

xterm.js batches writes to RAF. With WebGL renderer, all rendering is GPU -- Chrome's layout/paint pipeline is not involved. Only compositing of the WebGL canvas element.

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 0 | WebGL: no DOM changes |
| Layout | 0 | No layout changes |
| Paint | 0 | WebGL: no paint |
| Composite | 60 | Canvas recomposited every frame during output |

---

## Key Findings

1. **ttyd is extremely clean from a pipeline perspective.** The WebGL renderer means almost zero Chrome rendering pipeline involvement during normal operation.
2. **One forced reflow exists** in the overlay addon (resize/copy overlay), but it only fires on user action, not idle.
3. **No idle DOM mutations** with WebGL or Canvas renderer.
4. **WebSocket messages are batched** by xterm.js internal write buffer -- not rendered per-message.
5. **Preact's virtual DOM is essentially dormant** during terminal operation -- the Terminal component never re-renders.
6. **FitAddon resize is well-behaved** -- single read-write cycle, not thrashing.
