# Shell In A Box - Static Code Analysis Findings

## 1. Tech Stack

**Backend:** Pure C, custom HTTP server implementation (no external web framework). Built with autotools (`configure.ac`, `Makefile.am`). The C code includes:
- `libhttp/` — Custom HTTP server with SSL support (optional OpenSSL)
- `shellinabox/` — Application logic: session management, PTY launching, privilege handling
- `logging/` — Logging subsystem

**Frontend:** Custom JavaScript preprocessed through a C preprocessor (`.jspp` files compiled with `#define` macros). Two main JS files:
- `vt100.jspp` (~4584 lines) — Full VT100 terminal emulator
- `shell_in_a_box.jspp` (~350 lines) — AJAX communication layer, extends VT100

The JS is compiled and embedded into C header files (e.g., `shell_in_a_box.h`) which are `#include`d directly into `shellinaboxd.c` — the frontend is baked into the binary at compile time.

## 2. Rendering Pipeline: Pure DOM

**Zero canvas usage.** Confirmed: no `canvas`, no `getContext`, no `2d`/`webgl` anywhere in the codebase.

The rendering is **100% DOM-based** using `<pre>` and `<span>` elements:

1. Terminal grid is a `<pre id="console">` element containing nested `<pre>` lines
2. Each line is a `<pre>` (or `<div>` in some modes) containing `<span>` elements
3. Each `<span>` represents a run of characters with the same attributes (color, bold, etc.)
4. Text content is set via `document.createTextNode()` and `appendChild()` — NOT innerHTML for character data (see vt100.jspp line 1498-1501: explicitly sets innerHTML to '' then uses createTextNode)
5. Attributes (colors, bold, underline, blink) are applied via CSS classes on spans
6. Cursor is a separate `<pre id="cursor">` element positioned absolutely

The DOM structure (from `vt100.jspp` line 891-925):
```
<div id="scrollable">
  <pre id="console">
    <pre>  <!-- line 1 -->
      <span>text</span><span>more text</span>
    </pre>
    ...
  </pre>
  <pre id="cursor">&nbsp;</pre>
</div>
```

## 3. Update/Flush Mechanism: AJAX Long-Polling

**No WebSocket. No requestAnimationFrame. No setInterval for rendering.**

Transport drives rendering directly:
1. Client sends HTTP POST with session ID, terminal dimensions
2. Server holds the connection until output is available (long-poll)
3. Response arrives as JSON: `{data: "terminal output", session: "id"}`
4. Client calls `this.vt100(response.data)` which synchronously parses VT sequences and updates DOM
5. Client immediately sends next POST request (`this.sendRequest(request)` on line 216)
6. On timeout/error, retries after 1 second (`setTimeout(..., 1000)` on line 220)

**Key input** is sent via separate POST requests (`sendKeys()` method, line 231). Keys are hex-encoded and sent with session ID. A `keysInFlight` flag prevents concurrent key requests — pending keys are buffered and sent when the previous request completes.

The POST request has a 30-second timeout (`request.timeout = 30000`, line 174) to prevent proxy thread hijacking.

## 4. Transport: AJAX Long-Polling (HTTP POST)

**No WebSocket at all.** Uses `XMLHttpRequest` exclusively. Even includes an IE ActiveX fallback for XMLHttpRequest (line 74-81).

**Protocol:**
- Output polling: POST to `url?` with body `width=W&height=H&session=SID` (or `&rooturl=URL` for first request)
- Key input: Separate POST to `url?` with body `width=W&height=H&session=SID&keys=HEXENCODED`
- Response format: JSON `{data: "...", session: "..."}`
- Session lifecycle: Server assigns session ID on first request, client includes it in subsequent requests

**Latency characteristics:** Long-polling means output arrives as soon as the server has data (no polling interval). But there's inherent HTTP overhead per round-trip, and keys go through separate POST requests with serialization (`keysInFlight` flag).

## 5. PTY

Uses **`openpty()`** directly from C (not forkpty, despite a reference to it in error messages). In `launcher.c`:
- Line 800: `openpty(pty, &slave, NULL, NULL, NULL)` — creates PTY pair
- Line 1497: `ioctl(pty, TIOCSWINSZ, &win)` — resize via ioctl
- Conditional compilation for `pty.h`, `libutil.h`, `util.h` depending on platform
- Handles utmp/utmpx entries for session tracking
- Privilege separation: launcher runs privileged, drops privileges after PTY setup

## 6. ANSI/VT Processing: Client-Side vt100.jspp

**Full VT100 emulator in JavaScript** (~4584 lines). `vt100.jspp` implements:

- Complete VT100/ANSI escape sequence parser with state machine (states: ESnormal, ESesc, ESsquare, ESgetpars, ESgotpars, ESdeviceattr, ESfunckey, EShash, ESsetG0-G3, ESbang, ESpercent, ESignore, ESnonstd, ESpalette, EStitle, ESss2, ESss3, ESVTEtitle)
- Character attribute handling: reverse, underline, dim, bright, blink, default fg/bg (bitmask-based: `ATTR_DEFAULT=0x60F0`)
- Character set support: Latin1Map, VT100GraphicsMap, CodePage437Map, DirectToFontMap
- Mouse tracking (MOUSE_DOWN, MOUSE_UP, MOUSE_CLICK events)
- URL linkification (configurable via `linkifyURLs` variable)
- Scrollback buffer (default 2000 lines, `maxScrollbackLines`)
- Alternate screen buffer (console + alt_console)
- Copy/paste support via hidden input element

The server sends raw terminal output — all parsing happens client-side.

## 7. Font Rendering: CSS Monospace Stack

From `styles.css` line 48:
```css
font-family: "DejaVu Sans Mono", "Everson Mono", FreeMono, "Andale Mono", Consolas, monospace;
```

Applied to `#console`, `#alt_console`, `#cursor`, `#lineheight`, and `.hidden pre`.

Since rendering is pure DOM (`<pre>` and `<span>` elements with text nodes), the browser applies **full native font rendering** including:
- Subpixel antialiasing (ClearType on Windows, FreeType on Linux)
- Font hinting
- Kerning (though monospace fonts have uniform advance widths)

Character dimensions are measured at initialization: `this.cursorWidth = this.cursor.clientWidth` and `this.cursorHeight = this.lineheight.clientHeight` (lines 974-975).

## 8. Eye Strain Assessment: LOW

**Rating: LOW strain potential**

- **Pure DOM rendering:** Text is native browser text in `<span>` elements. Gets full subpixel AA/ClearType/font hinting. This is the best possible text rendering quality on any platform.
- **No canvas:** Zero bitmap text rendering. Every character is a real DOM text node.
- **Cursor blink:** `setInterval` at 500ms (line 3201), toggling CSS class between `bright` and `dim`. The `dim` class uses `opacity: 0.2` (line 71-73). Configurable via `blinkingCursor` property — can be disabled. Standard blink rate, not aggressive.
- **No animation loops:** No RAF, no render loops. DOM updates happen synchronously when data arrives from long-poll. The browser's own compositor handles repainting.
- **Scrollback:** 2000 lines default. Older lines get className `'scrollback'`. DOM nodes are created/removed as needed — no virtual scrolling, but the scrollback limit prevents unbounded DOM growth.
- **Long-poll latency:** The AJAX approach adds slight latency vs WebSocket, but this actually helps reduce rapid re-render storms — HTTP round-trip naturally throttles update frequency.

**Key advantage over canvas-based terminals:** Native text rendering with full OS font stack. This is the single biggest factor for sustained reading comfort.

**Minor concerns:** Using `eval()` to parse JSON responses (line 200) is a security issue but not a strain factor. The `.innerHTML` usage for structural setup is fine since character content uses `createTextNode`.

## 9. Key Files

1. **`/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/vt100.jspp`** — Complete VT100 terminal emulator (4584 lines). DOM rendering, escape sequence parsing, cursor management, scrollback, mouse handling.

2. **`/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/shell_in_a_box.jspp`** — AJAX communication layer. Extends VT100 with XMLHttpRequest long-polling, key input, session management.

3. **`/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/launcher.c`** — PTY creation (openpty), privilege management, process launching (1894 lines).

4. **`/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/styles.css`** — Terminal CSS including font stack, cursor styles, scrollable container layout.

5. **`/c/dev/web-terminal-research/shellinabox-readonly/shellinabox/shellinaboxd.c`** — Main daemon entry point, embeds all resources (JS, CSS, HTML compiled into C headers).
