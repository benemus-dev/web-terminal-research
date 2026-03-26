# DomTerm - Static Code Analysis Findings

**Repository:** https://github.com/PerBothner/DomTerm
**Version analyzed:** 3.2.0
**Analysis date:** 2026-03-26

---

## 1. Tech Stack

### Languages
- **JavaScript** — client-side terminal emulator (~15k LOC across hlib/)
- **C/C++** — backend server (lws-term/, ~10 files), native PTY (native/pty/)
- **Java** — optional backend (websocketterm/, domterm-jar-manifest)
- Optional Qt/C++ frontend (qtdomterm/)
- Optional Electron frontend (electron/)
- Optional Wry/Rust frontend (dt-wry/)

### Build System
- **Autotools** (configure.ac + Makefile.am) — primary build system
- No CMakeLists.txt. No webpack/bundler for JS (raw ES modules).

### Dependencies
- **libwebsockets (lws)** — C WebSocket server (lws-term/server.cc)
- **json-c** — JSON parsing in C backend
- **libmagic** — MIME type detection (optional)
- **@xterm/xterm ^5.5.0** — optional alternative renderer (package.json)
- **GoldenLayout** — tab/panel layout manager (hlib/goldenlayout.js)

---

## 2. Rendering Pipeline

### Classification: **DOM_ONLY** (primary mode)

DomTerm has TWO rendering modes, but its primary/native mode is pure DOM:

1. **DomTerm native mode (DOM_ONLY):** Terminal output is structured as `<div class="domterm-pre">` elements containing `<span>` elements for styled text. No canvas or WebGL for text rendering.

2. **xterm.js mode (optional):** Via `xterminal.js`, can use xterm.js with DOM or WebGL addon. This is an alternative mode, not the default. (`Terminal.defaultXtRendererType = 'dom'` in terminal.js:1053)

### Canvas usage
Canvas is used ONLY for **Sixel graphics** (inline images) in `domterm-parser.js:1561-1608`. A `<canvas>` element is created per Sixel image, using `getContext('2d')` to draw pixel data. This is NOT used for text rendering.

### DOM structure
```
div.domterm
  div.dt-buffers          (overflow-y: scroll — scrollback container)
    div.dt-buffer          (the active buffer)
      div.domterm-pre      (one per logical line)
        span.term-style    (styled text segments — colors, bold, etc.)
        span[std="caret"]  (cursor element, contentEditable=true)
```

Key evidence:
- `terminal.js:3384`: `n.setAttribute("class", "domterm-pre")` — lines are div elements
- `domterm-core.css:55-66`: "We use `<div class="domterm-pre">` instead of `<pre>` for regular terminal lines"
- Colors applied via CSS classes on `<span>` elements with `term-style` class

### Input handling
- **contentEditable** on the caret span: `caretNode.contentEditable = true` (terminal.js:4060)
- **keydown** listener on topNode (terminal.js:4066)
- **keypress** listener on topNode (terminal.js:4070)
- **input** event listener on topNode (terminal.js:4072)
- **compositionStart/End** for IME support (terminal.js:4080+)
- Uses browserkeymap.js for key name resolution

---

## 3. Update/Flush Mechanism

### requestAnimationFrame-based display updates

```js
// terminal.js:6678-6682
Terminal.prototype.requestUpdateDisplay = function() {
    if (this._updateTimer)
        this._deferUpdate = true;
    else
        this._updateTimer = requestAnimationFrame(this._updateDisplay);
}
```

- Uses RAF with a **defer limit of 200ms** (`this.updateDeferLimit = 200`, terminal.js:432)
- When already in an update cycle, sets `_deferUpdate = true` and schedules another RAF
- The `_updateDisplay` callback does: `_restoreInputLine()`, `_breakVisibleLines()`, `_checkSpacer()`, `_scrollIfNeeded()`

### Data flow
1. WebSocket `onmessage` receives ArrayBuffer or string
2. `DomTerm._handleOutputData()` called (domterm.js:753)
3. Data fed to `terminal.processInputCharacters()` which calls `parser.parseBytes()`
4. Parser directly manipulates DOM during parse (inserting text nodes, creating spans)
5. After parse, `requestUpdateDisplay()` schedules RAF for layout fixup

The parser does **synchronous DOM manipulation** — text and elements are inserted immediately during `parseBytes()`. The RAF is only for post-parse layout work (line breaking, scroll adjustment).

---

## 4. Transport

### WebSocket

- Client creates `new WebSocket(wspath, wsprotocol)` (domterm.js:716)
- Binary mode: `wsocket.binaryType = "arraybuffer"` (domterm.js:722)
- Server: **libwebsockets** C library (lws-term/server.cc)
  - Uses `lws_protocols`, `lws_context_creation_info` structs
  - Supports permessage-deflate and deflate-frame extensions
- Flow control: client sends RECEIVED confirmations back to server (`_confirmReceived()`)
- Reconnection: `setTimeout(reconnect, reconnectDelay)` on close (domterm.js:795)

---

## 5. PTY

### C-level PTY via openpty/fork

Located in `lws-term/protocol.cc`:
- Uses `openpty(&master, &slave, NULL, NULL, NULL)` (line 708)
- Forks child process, calls `login_tty(slave)` (line 807)
- Fallback PTY implementation in `native/pty/pty_fork.c`

The backend (lws-term) is a C/C++ server that:
1. Listens for WebSocket connections (via libwebsockets)
2. Spawns shell processes with PTY
3. Bridges PTY I/O to WebSocket messages

---

## 6. ANSI/VT Processing

### Client-side JavaScript parser

**File:** `hlib/domterm-parser.js` (2657 lines) — `DTParser` class

- Full VT100/xterm escape sequence parser in JS
- State machine: `controlSequenceState` tracks parse position (INITIAL_STATE, etc.)
- Processes bytes directly: `parseBytes(bytes, beginIndex, endIndex)`
- Handles CSI, OSC, DCS sequences with parameter parsing
- `_flagChars` tracks prefix characters and intermediates
- DEC private mode support (`saved_DEC_private_mode_flags`)
- Sixel graphics decoding in `sixel-decode.js`

The server does NOT parse escape sequences — raw PTY output is sent over WebSocket to the client, which does all VT processing.

---

## 7. Font Rendering

### CSS fonts only — no canvas text rendering

```css
/* domterm-default.css:72-73 */
div.domterm { --monospace-family: Cascadia Code, Terminal, Menlo, monospace, DejaVu Sans Mono; }
div.domterm-pre, span.docutils.literal { font-family: var(--monospace-family) }
```

- Font family defined via CSS custom property `--monospace-family`
- Wide character handling via CSS: `span.dt-cluster.w1` and `span.dt-cluster.w2` classes with explicit widths (domterm-core.css:34-42)
- All text rendered by the browser's native text layout engine

---

## 8. Strain Assessment

### Cursor blink: CSS animation (good)

```css
/* domterm-default.css:186, 196-201 */
animation: blinking-caret 1.5s steps(1) 0s var(--caret-blink-count);

@keyframes blinking-caret {
    0% { }
    30% { border-right: inherit; margin-right: inherit;
          background-color: inherit; color: inherit; text-decoration: inherit }
    100% {}
}
```

- Pure CSS `@keyframes` animation — no JS interval for cursor blink
- 1.5s cycle, `steps(1)` for sharp on/off transition
- `--caret-blink-count` CSS variable controls how many times it blinks before stopping
- Multiple caret styles supported: blinking-block, blinking-bar, blinking-underline

### Text blink (ANSI blink attribute): JS setTimeout

- `terminal.js:617-624`: `setTimeout(flip, this._blinkShowTime)` / `setTimeout(flip, hideTime)`
- Toggle via `topNode.classList.add/remove('blinking-hide')`
- This is for ANSI SGR blink attribute, NOT cursor — rarely used in practice

### Scrollback: CSS overflow

```css
/* domterm-core.css */
div.dt-buffers { overflow-y: scroll; width: 100%; height: 100%; }
div.dt-buffer { overflow-x: hidden; overflow-y: hidden }
```

- Native browser scrollbar on `dt-buffers` container
- No virtual scrolling or manual scroll management
- `_scrollIfNeeded()` called from `_updateDisplay` RAF callback

### Colors: CSS properties

- CSS custom properties: `--background-color`, `--foreground-color`, `--caret-color`, etc.
- Light/dark themes via `div.domterm[reverse-video]` selector
- ANSI 16 colors + 256 colors + truecolor via inline styles on `<span>` elements
- Selection colors: `--selection-foreground-color`, `--selection-background-color`

### Strain summary

DomTerm's DOM-only approach is **low strain** for static content:
- No canvas redraws, no GPU shader overhead
- Browser handles all text rasterization and composition
- CSS-driven cursor blink (no JS timer jitter)
- Native scroll (smooth, hardware-accelerated by browser)

Potential strain concerns:
- Synchronous DOM manipulation during parse — large output bursts create many DOM nodes
- No virtual scrolling — very long scrollback buffers accumulate DOM nodes
- Each styled character range is a separate `<span>` — colorful output (e.g., `ls --color`) creates high DOM node counts

---

## 9. Key Files

| File | Purpose |
|------|---------|
| `hlib/terminal.js` (11k lines) | Core terminal class: DOM structure, input handling, display updates, caret management |
| `hlib/domterm-parser.js` (2.6k lines) | VT/ANSI escape sequence parser — all terminal protocol processing |
| `hlib/domterm.js` (930 lines) | WebSocket connection, data flow, session management |
| `hlib/domterm-core.css` | Core layout: domterm-pre structure, buffer overflow, caret styling |
| `lws-term/server.cc` | C/C++ backend: libwebsockets server, PTY management, HTTP serving |
