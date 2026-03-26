# hterm - Static Code Analysis Findings

**Repo:** libapps-mirror (Chromium libapps monorepo)
**Path:** `/c/dev/web-terminal-research/libapps-mirror/hterm/`
**Version:** 1.92.1
**Source:** https://chromium.googlesource.com/apps/libapps

---

## 1. Tech Stack

- **Language:** JavaScript (ES modules, `"type": "module"`)
- **Monorepo structure:** hterm is one package within the libapps monorepo alongside:
  - `libdot/` - shared utility library (storage, i18n, etc.)
  - `nassh/` - Secure Shell Chrome extension (uses hterm)
  - `terminal/` - ChromeOS terminal app (NOTE: this uses **xterm.js** with WebGL addon, NOT hterm for rendering)
  - `ssh_client/`, `wassh/`, `wasi-js-bindings/` - SSH/WASM infrastructure
- **Build tools:** Rollup (bundling), Terser (minification), ESLint, Mocha+Chai (testing)
- **Dependencies:** Only `punycode` (runtime). Dev: rollup, terser, eslint, mocha
- **Output:** `dist/js/hterm_all.js` (single bundle)

**Key observation:** The `terminal/` app (ChromeOS Terminal) has migrated to xterm.js with WebglAddon/CanvasAddon (`terminal_emulator.js:463-471`). hterm itself remains a pure DOM-based terminal library.

---

## 2. Rendering Pipeline

**Classification: DOM_ONLY**

hterm uses a pure DOM rendering approach. Canvas is used solely for font measurement, never for text rendering.

### Canvas usage (measurement only)
In `hterm_scrollport.js:1134-1148`, a `<canvas>` element is created as a "ruler" to measure character dimensions via `measureText()`. This is a standard technique to get pixel-accurate character cell sizes. No `fillText`, `drawText`, or any canvas painting is performed anywhere in the codebase.

### DOM rendering architecture

**Custom HTML elements:**
- `<x-screen>` - The root container element (`hterm_scrollport.js:617`). Made contenteditable for IME support.
- `<x-row>` - Each terminal row is an `<x-row>` element. CSS sets `display: block; height: var(--hterm-charsize-height)`.
- Text spans within rows use `<span>` elements with inline styles for attributes (color, bold, italic, underline, etc.)

**Text attribute containers** (`hterm_text_attributes.js:200-274`):
- Default text (no attributes): bare `TextNode` (via `document.createTextNode`)
- Styled text: `<span>` with inline CSS for color, font-weight, font-style, text-decoration, etc.
- Wide characters (CJK): `<span class="wc-node">`
- Blink text: `<span class="blink-node">`

**Virtual viewport** (`hterm_scrollport.js:34-50`):
The ScrollPort implements a virtual viewport -- only visible rows exist in the DOM. If the scrollback has 100,000 rows but only 25 are visible, only ~25 `<x-row>` DOM nodes exist. Rows are fetched from the RowProvider on demand during redraw.

**Redraw flow:**
1. `scheduleRedraw()` -> `setTimeout(() => redraw_())` (coalesced)
2. `redraw_()` calls `drawVisibleRows_(topRowIndex, bottomRowIndex)`
3. `drawVisibleRows_` reuses cached row nodes where possible, removes/inserts nodes as needed
4. Selection-aware: rows containing active selection are never touched to avoid disrupting it

### Architecture layers
- `hterm.Terminal` - orchestrator; holds two `hterm.Screen` instances (primary + alternate)
- `hterm.Screen` - manages an array of row Elements (`rowsArray`), cursor position, text insertion
- `hterm.ScrollPort` - virtual viewport; owns the `<x-screen>` DOM, handles scroll, renders only visible rows
- `hterm.VT` - escape sequence parser, calls Terminal methods
- `hterm.TextAttributes` - creates styled `<span>` containers matching current SGR attributes

---

## 3. Update/Flush Mechanism

**Classification: setTimeout (NOT requestAnimationFrame)**

hterm does NOT use `requestAnimationFrame`. All redraws are scheduled via `setTimeout`:

```
// hterm_scrollport.js:1279-1288
hterm.ScrollPort.prototype.scheduleRedraw = function() {
  if (this.timeouts_.redraw) return;  // coalesce
  this.timeouts_.redraw = setTimeout(() => {
    delete this.timeouts_.redraw;
    this.redraw_();
  });
};
```

The Terminal has its own `scheduleRedraw_()` with the same pattern (`hterm_terminal.js:2812-2821`). Multiple calls are coalesced -- only one redraw fires per event loop turn.

Scroll-to-bottom is also `setTimeout` based, with a 10ms delay (`hterm_terminal.js:2837`).

No RAF means redraws are not synchronized with vsync. However, since hterm only mutates DOM nodes (not canvas), the browser's own compositor handles actual painting at vsync.

---

## 4. Transport

hterm is a **terminal emulator library only** -- it has no built-in transport. It exposes `hterm.Terminal.IO` for I/O:

- `terminal.interpret(str)` - feed VT data into the terminal
- `io.onVTKeystroke` / `io.sendString` - callbacks for user input

Typical consumers:
- **nassh/** (Secure Shell) - provides SSH transport via NaCl/WASM
- **ChromeOS terminal** - connects to local shell (crostini/Linux)
- Any web app can embed hterm and provide its own transport (WebSocket, etc.)

---

## 5. ANSI/VT Processing

**Client-side parser in `hterm_vt.js` (3,480 lines)**

The VT parser is a state machine in `hterm.VT`:

- `parseUnknown_()` - scans for printable text vs control characters using a regex (`cc1Pattern_`)
- `parseCSI_()` - Control Sequence Introducer (SGR, cursor movement, etc.)
- `parseESC_()` - basic escape sequences
- `dispatch(type, code, parseState)` - dispatches to handler tables (`CC1`, `CSI`, `ESC`, `OSC`, `DCS`)
- Supports xterm compatibility, 8-bit control characters, OSC sequences (title, clipboard, hyperlinks)

Character maps handled in `hterm_vt_character_map.js` (DEC special graphics, etc.)

Entry point: `hterm.Terminal.prototype.interpret(str)` -> `this.vt.interpret(str)`.

---

## 6. Font Rendering

**CSS-based font rendering (browser native subpixel AA)**

Font configuration (`hterm_preference_manager.js:473-494`):
- Default font-family: `"DejaVu Sans Mono", "Noto Sans Mono", "Everson Mono", FreeMono, Menlo, Terminal, monospace`
- Default font-size: 15px
- Font smoothing: `antialiased` (CSS `-webkit-font-smoothing`)

**Character cell sizing** (`hterm_scrollport.js:1134-1148`):
- Uses canvas `measureText('X')` for width, `measureText('X\u2588')` with `actualBoundingBoxAscent/Descent` for height
- `lineHeightPaddingSize` adds configurable padding
- Cell dimensions stored as CSS custom properties (`--hterm-charsize-width`, `--hterm-charsize-height`)
- All `<x-row>` elements sized via these CSS variables

**Font change flow:**
`syncCharacterSize()` -> re-measures with canvas -> updates CSS variables -> `resize()` -> `scheduleRedraw()`

---

## 7. Strain Assessment

### Text rendering: LOW STRAIN
hterm uses pure DOM text rendering. The browser handles all text rasterization with its native font renderer, which uses **subpixel antialiasing** (ClearType on Windows, subpixel AA on macOS/Linux). This produces the highest-quality text rendering available in a browser.

The `font-smoothing: antialiased` default forces grayscale AA on macOS, but on Windows ClearType is still used (the CSS property is a webkit extension primarily affecting macOS).

### Cursor blink: setTimeout-based, configurable
- Blink cycle: configurable via `cursor-blink-cycle` pref, default `[100, 100]` ms (100ms on, 100ms off -- very fast)
- Implementation: `onCursorBlink_()` toggles a `visible` attribute on the cursor DOM node via `setTimeout` (`hterm_terminal.js:4142-4157`)
- **Blink pause:** When user is typing or moving cursor, blinking pauses for 500ms then resumes (`pauseCursorBlink_`, line 3188)
- Cursor is a DIV element (`this.cursorNode_`), styled via CSS

### Scrollback: Virtual viewport, efficient
- Only visible rows in DOM (~25-50 nodes typically)
- Scrollback rows stored as detached `<x-row>` elements in `scrollbackRows_[]` array
- No DOM overhead for offscreen rows -- they exist only as JavaScript objects
- Scrollbar position drives which rows get rendered

### Overall strain factors:
- **Text quality:** Best possible (browser native subpixel AA) -- LOW
- **Cursor blink:** Default 100ms cycle is aggressive (10Hz toggle); configurable or disableable -- LOW to MEDIUM depending on config
- **Reflow/repaint:** DOM mutations trigger browser layout; for bulk output this could cause intermediate paints, but coalesced setTimeout helps -- LOW
- **No vsync sync:** setTimeout-based redraws are not frame-aligned, but browser compositor handles final paint -- NEGLIGIBLE impact

---

## 8. Key Files

1. **`hterm/js/hterm_terminal.js`** (4,241 lines) - Main terminal class. Orchestrates Screen, ScrollPort, VT. Cursor management, interpret(), all terminal operations.

2. **`hterm/js/hterm_scrollport.js`** (2,200 lines) - Virtual viewport renderer. DOM management, `<x-screen>`/`<x-row>` creation, redraw logic, character measurement, scroll handling.

3. **`hterm/js/hterm_vt.js`** (3,480 lines) - VT/ANSI escape sequence parser. State machine, CSI/OSC/DCS dispatch, xterm compatibility.

4. **`hterm/js/hterm_screen.js`** (1,095 lines) - Screen buffer. Row array management, cursor positioning, text insert/overwrite operations on DOM nodes.

5. **`hterm/js/hterm_text_attributes.js`** - SGR attribute tracking. Creates styled `<span>` containers for colored/bold/italic/underline text.

---

## Notable Design Decisions

- **Pure DOM, no canvas text rendering** -- relies entirely on browser's native text rasterizer. This is the simplest approach and produces the best text quality, but limits rendering speed for very high throughput scenarios.
- **Virtual viewport** -- only visible rows are in the DOM, so scrollback size has minimal performance impact.
- **setTimeout, not RAF** -- redraws are not vsync-synchronized, but since hterm only mutates DOM (no canvas/WebGL), the browser compositor handles frame timing naturally.
- **The ChromeOS Terminal app (`terminal/`) has migrated to xterm.js** with WebGL rendering (`WebglAddon` primary, `CanvasAddon` fallback). This suggests Google found hterm's DOM rendering insufficient for their performance needs, even though hterm remains available as a library.
