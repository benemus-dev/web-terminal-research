# ttyd - Static Code Analysis Findings

**Repository:** https://github.com/tsl0922/ttyd
**Version analyzed:** 1.7.7
**Analysis date:** 2026-03-26

---

## 1. Tech Stack

### Backend (C)
- **Language:** C99
- **Build system:** CMake (minimum 3.12.0)
- **Core dependencies:**
  - **libwebsockets** (>= 3.2.0) - WebSocket server
  - **json-c** - JSON parsing for protocol messages and config
  - **libuv** - Event loop, async I/O, process management
  - **zlib** - Compression (permessage-deflate WS extension)
  - **OpenSSL** (optional) - TLS support
  - On Windows: shell32, ws2_32, dbghelp, getopt
- **Backend C files:**
  - `src/server.c` - Main entry, CLI parsing, libwebsockets context setup
  - `src/protocol.c` - WebSocket protocol handler (tty protocol callbacks)
  - `src/pty.c` - PTY spawning (forkpty on Unix, ConPTY on Windows)
  - `src/http.c` - HTTP serving (static files, token endpoint)
  - `src/utils.c` - Utility functions

### Frontend (TypeScript/Preact)
- **Framework:** Preact 10.x (lightweight React alternative)
- **Language:** TypeScript 5.3
- **Build tools:** Webpack 5, gulp (for inlining built assets into C binary)
- **Package manager:** Yarn 3.6.3
- **xterm.js version:** `@xterm/xterm` ^5.5.0 (v5 series, `@xterm/` scoped packages)
- **Key frontend dependencies:**
  - `@xterm/xterm` ^5.5.0 - Terminal emulator
  - `@xterm/addon-webgl` ^0.18.0 - WebGL renderer
  - `@xterm/addon-canvas` ^0.7.0 - Canvas renderer
  - `@xterm/addon-fit` ^0.10.0 - Auto-resize
  - `@xterm/addon-image` ^0.8.0 - Sixel image support
  - `@xterm/addon-clipboard` ^0.1.0
  - `@xterm/addon-web-links` ^0.11.0
  - `@xterm/addon-unicode11` ^0.8.0
  - `zmodem.js` ^0.1.10 - ZMODEM file transfer
  - `trzsz` ^1.1.5 - trzsz file transfer
- **Frontend source files:**
  - `html/src/components/app.tsx` - Root component, default config
  - `html/src/components/terminal/index.tsx` - Terminal container component
  - `html/src/components/terminal/xterm/index.ts` - Core xterm wrapper (rendering, WS, flow control)
  - `html/src/components/terminal/xterm/addons/overlay.ts` - Resize/status overlay
  - `html/src/components/terminal/xterm/addons/zmodem.ts` - ZMODEM/trzsz addon
  - `html/src/index.tsx` - Entry point
  - `html/src/style/index.scss` - Minimal CSS

---

## 2. Rendering Pipeline

### Renderer Selection

**Default:** `rendererType: 'webgl'` (hardcoded in `html/src/components/app.tsx` line 13)

ttyd ships all three xterm.js renderers and selects at runtime via `setRendererType()` in `html/src/components/terminal/xterm/index.ts`:

- **`'webgl'`** (default): Loads `@xterm/addon-webgl`. On WebGL context loss, disposes and does NOT auto-fallback (just disposes). On load failure, falls back to canvas.
- **`'canvas'`**: Loads `@xterm/addon-canvas`. Disposes WebGL first if loaded. On failure, falls back to DOM.
- **`'dom'`**: Disposes both WebGL and canvas addons, uses xterm.js v5's built-in DOM renderer.

The renderer can be changed at runtime via:
1. Server-side `--client-option rendererType=canvas` CLI flag
2. URL query parameter `?rendererType=canvas`
3. Server preferences sent over WebSocket (`SET_PREFERENCES` command)

URL query params override server preferences (see `parseOptsFromUrlQuery`).

### Fallback Chain
```
WebGL (default) --[load failure]--> Canvas --[load failure]--> DOM
```

### Classification: **XTERM_WEBGL** (default)

The WebGL addon uses GPU-accelerated glyph atlas rendering via WebGL2. Glyphs are rasterized to a texture atlas on first use, then rendered as textured quads. This is the highest-performance xterm.js renderer.

---

## 3. Update/Flush Mechanism

### Data Flow: WebSocket Message to Screen

1. **WebSocket `message` event** fires (`onSocketData`)
2. First byte parsed as command type (`'0'` = OUTPUT)
3. For OUTPUT: data passed to `writeFunc(data)` which calls `writeData()`
4. `writeData()` calls `terminal.write(data)` or `terminal.write(data, callback)`
5. xterm.js internally batches writes and renders on next animation frame (RAF)

### Flow Control (Backpressure)

ttyd implements explicit flow control between WebSocket and PTY:

- **`written` counter** tracks bytes written to terminal since last pause check
- **`limit`**: 100,000 bytes - threshold to engage flow control
- **`highWater`**: 10 pending writes - sends PAUSE command to server
- **`lowWater`**: 4 pending writes - sends RESUME command to server

When `written > limit`:
- `terminal.write(data, callback)` is used (with completion callback)
- Each callback decrements `pending` counter
- If `pending > highWater`: sends PAUSE to server (server calls `uv_read_stop`)
- If `pending < lowWater`: sends RESUME to server (server calls `uv_read_start`)

When `written <= limit`:
- `terminal.write(data)` is used (no callback, no flow control overhead)

### Server-Side Buffering

The server (`protocol.c`) reads PTY output in `read_cb`, immediately stops reading (`uv_read_stop`), buffers one chunk in `pss->pty_buf`, and signals `lws_callback_on_writable`. On `LWS_CALLBACK_SERVER_WRITEABLE`, it sends the buffer and calls `pty_resume` to read the next chunk. This is a **one-chunk-at-a-time** design - no server-side output queue.

### Classification: **PUSH_THEN_RAF**

Data is pushed immediately into xterm.js's write buffer on each WebSocket message. xterm.js internally coalesces pending writes and renders on the next requestAnimationFrame. The flow control mechanism throttles the data source (PTY) rather than the rendering.

---

## 4. Transport Layer

### Server-Side
- **Library:** libwebsockets (C library, >= 3.2.0)
- **Protocol:** Binary WebSocket frames (`LWS_WRITE_BINARY`)
- **Subprotocol:** `"tty"` (registered as lws protocol)
- **Extensions:** permessage-deflate, deflate-frame (WebSocket compression)
- **Ping/keepalive:** 5-second interval (configurable via `--ping-interval`)
- **Port:** Default 7681

### Client-Side
- **API:** Native browser `WebSocket` (no wrapper library)
- **Binary mode:** `socket.binaryType = 'arraybuffer'`
- **Subprotocol:** `['tty']`
- **Reconnect:** Auto-reconnect on abnormal close (code != 1000), with token refresh

### Protocol Format
- First byte = command type (ASCII character)
- Server-to-client: `'0'` OUTPUT, `'1'` SET_WINDOW_TITLE, `'2'` SET_PREFERENCES
- Client-to-server: `'0'` INPUT, `'1'` RESIZE_TERMINAL, `'2'` PAUSE, `'3'` RESUME
- Initial handshake: client sends JSON with `{AuthToken, columns, rows}`
- All binary framing, text encoded as UTF-8 after command byte

---

## 5. PTY / Process Spawning

### Unix (Linux/macOS/BSD)
- **Function:** `forkpty()` (from `<pty.h>`, `<util.h>`, or `<libutil.h>` depending on platform)
- Creates a pseudo-terminal pair and forks in one call
- Child: `setsid()`, optional `chdir()`, `putenv()` for custom env vars, then `execvp()`
- Parent: sets master fd to non-blocking, wraps with two `uv_pipe_t` (in/out) via `fd_duplicate()`/`uv_pipe_open()`
- Wait thread: `uv_thread_create()` running `waitpid()` loop, signals exit via `uv_async_send()`

### Windows
- **Function:** `CreatePseudoConsole()` (ConPTY API, Windows 10 1809+)
- Dynamically loaded from kernel32.dll at startup (`conpty_init()`)
- Creates named pipes (`\\.\pipe\ttyd-term-in-PID-N` / `out`), connects via `uv_pipe_connect()`
- Process spawned with `CreateProcessW()` + `EXTENDED_STARTUPINFO_PRESENT` + `PROC_THREAD_ATTRIBUTE_PSEUDOCONSOLE`
- Exit detection: `RegisterWaitForSingleObject()` on process handle, signals via `uv_async_send()`

### Default Shell
- No default shell hardcoded - the command is **required** as a CLI argument: `ttyd [options] <command> [args...]`
- Terminal type defaults to `xterm-256color` (set via TERM env var)

---

## 6. ANSI / VT Processing

**Client-side only.** xterm.js handles all ANSI/VT parsing and rendering.

The server is a transparent byte pipe: PTY output bytes are forwarded as-is over WebSocket (prefixed with command byte `'0'`), and client input bytes are written as-is to the PTY. No server-side terminal emulation, no escape sequence processing, no screen buffer.

The server sets `TERM=xterm-256color` for the child process, so applications generate xterm-compatible escape sequences.

---

## 7. Font Rendering

### Default Font Configuration (from `app.tsx`)
```
fontSize: 13
fontFamily: 'Consolas,Liberation Mono,Menlo,Courier,monospace'
```

### Font Smoothing
- No explicit `-webkit-font-smoothing` or `font-smooth` CSS properties set
- No subpixel rendering configuration
- Relies on browser/OS defaults for font antialiasing

### CSS
- Minimal: `html/src/style/index.scss` sets only `height: 100%`, `margin: 0`, `overflow: hidden`
- Terminal container gets `padding: 5px`
- No custom font-face declarations

### WebGL Renderer Font Handling
When using the WebGL renderer (default), xterm.js renders glyphs to a texture atlas using an offscreen canvas with `CanvasRenderingContext2D.fillText()`, then uploads to a WebGL texture. This means font rendering quality is determined by the browser's canvas text rendering, not CSS.

---

## 8. Strain Assessment

### Rating: **Low**

**Reasoning:**
- **WebGL renderer by default** - the highest-performance xterm.js renderer. Uses GPU-accelerated texture atlas rendering, avoiding DOM reflows and CSS layout recalculations. Consistent glyph positioning from atlas reuse.
- **Terminal text is inherently static** - unlike video, terminal content changes infrequently and in discrete character-cell updates. No subpixel motion, no temporal compression artifacts.
- **Flow control prevents render flooding** - the backpressure mechanism (PAUSE/RESUME at 100KB threshold) prevents the terminal from being overwhelmed with data, which could cause frame drops or rendering lag.
- **No post-processing or filtering** - raw glyph rendering from atlas, no sharpening/smoothing passes that could introduce artifacts.
- **Potential concern:** The WebGL glyph atlas uses nearest-neighbor or linear texture filtering. At non-integer DPI scaling, glyph edges may appear slightly blurred. This is a minor concern compared to video codec artifacts.
- **Fallback concern:** If WebGL context is lost or unavailable, the canvas fallback is still GPU-composited. DOM fallback would be the lowest quality but is unlikely to be reached.

---

## 9. Key Files for Rendering Logic

1. **`/c/dev/web-terminal-research/ttyd/html/src/components/terminal/xterm/index.ts`** - Core rendering orchestration: renderer selection (`setRendererType`), WebSocket data flow to `terminal.write()`, flow control (PAUSE/RESUME backpressure), addon loading (WebGL, Canvas, Fit)

2. **`/c/dev/web-terminal-research/ttyd/html/src/components/app.tsx`** - Default configuration: `rendererType: 'webgl'`, font settings (`fontSize: 13`, `fontFamily`), theme colors, flow control limits (`100000/10/4`)

3. **`/c/dev/web-terminal-research/ttyd/src/protocol.c`** - Server-side data pipeline: PTY output to WebSocket delivery, one-chunk-at-a-time buffering, initial preferences/title transmission, PAUSE/RESUME handling

4. **`/c/dev/web-terminal-research/ttyd/src/pty.c`** - PTY management: `forkpty()` (Unix) / `CreatePseudoConsole` (Windows), `uv_pipe_t` I/O, read callbacks with `uv_read_stop`/`uv_read_start` for flow control

5. **`/c/dev/web-terminal-research/ttyd/html/src/components/terminal/xterm/addons/overlay.ts`** - Status overlay (resize dimensions, connection state) - DOM element positioned over terminal, CSS opacity transitions
