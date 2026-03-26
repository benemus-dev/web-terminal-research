# GoTTY - Static Code Analysis Findings

## 1. Tech Stack

**Backend:** Go (no go.mod — uses Godeps + vendor directory for dependency management, pre-modules era).
Key dependencies (from vendor/):
- `github.com/gorilla/websocket` — WebSocket server
- `github.com/kr/pty` — PTY management
- `github.com/codegangsta/cli` — CLI framework
- `github.com/NYTimes/gziphandler` — gzip compression for HTTP
- `github.com/elazarl/go-bindata-assetfs` — embedded static assets

**Frontend:** TypeScript compiled via webpack (`js/webpack.config.js`, entry: `./src/main.ts`), bundled into `js/dist/gotty-bundle.js`. Uses **xterm.js** (NOT hterm) as the terminal emulator library. The bundle includes xterm.js with addons: `attach` (WebSocket attachment), `fit` (auto-sizing), `fullscreen`, and `terminado`.

The `Term` server option defaults to `"xterm"` but also supports `"hterm"` — see `server/options.go` line 32: `Term: "xterm"`.

## 2. Rendering Pipeline

**xterm.js** renders using a **Canvas-based renderer**. The bundled xterm.js version uses `<canvas>` elements for text rendering, drawing character glyphs onto a 2D canvas context. This is xterm.js's standard approach — it creates a canvas layer for the terminal grid and renders text glyphs as bitmap draws.

The pipeline:
1. WebSocket message arrives (base64-encoded PTY output)
2. Message decoded and written to xterm.js Terminal instance
3. xterm.js parses VT/ANSI sequences internally
4. xterm.js renders to canvas (character grid draw operations)

No custom DOM rendering — all rendering is delegated to xterm.js.

## 3. Update/Flush Mechanism

The bundle does NOT use explicit `requestAnimationFrame` or `setTimeout`/`setInterval` for rendering. xterm.js internally handles its own render scheduling (uses RAF internally within its renderer). GoTTY's role is purely data delivery — it writes incoming WebSocket data to the xterm.js Terminal instance, and xterm.js handles when to actually paint.

Data flow is event-driven: WebSocket `onmessage` -> decode -> `terminal.write()`.

## 4. Transport: WebSocket

**Protocol:** `gorilla/websocket` on the server side, native browser WebSocket on the client.

**Custom protocol ("webtty"):** Single-byte message type prefix, defined in `webtty/message_types.go`:
- Client->Server: `'1'` Input, `'2'` Ping, `'3'` ResizeTerminal
- Server->Client: `'1'` Output (base64-encoded), `'2'` Pong, `'3'` SetWindowTitle, `'4'` SetPreferences, `'5'` SetReconnect

**Connection flow** (`server/handlers.go`):
1. HTTP upgrade to WebSocket
2. Client sends JSON auth message (token + arguments)
3. Server creates PTY slave via factory
4. `webtty.Run()` starts two goroutines: one reads PTY->WebSocket, one reads WebSocket->PTY
5. PTY output is base64-encoded before sending (`webtty.go` line 137)

**WebSocket wrapper** (`server/ws_wrapper.go`): Adapts gorilla/websocket to io.Reader/Writer interface using `TextMessage` type.

## 5. PTY

Uses **`github.com/kr/pty`** (vendored). In `backend/localcommand/local_command.go`:
- `pty.Start(cmd)` — creates PTY and starts the command
- Read/Write directly on the PTY file descriptor
- Resize via `TIOCSWINSZ` ioctl (raw syscall, `SYS_IOCTL`)
- Close sends `SIGINT`, then `SIGKILL` after timeout (default 10s)

Buffer size: 1024 bytes per read (`webtty.go` line 71).

## 6. ANSI/VT Processing

**Client-side only.** The server treats PTY output as raw bytes — base64-encodes and forwards verbatim. All VT100/ANSI escape sequence parsing is handled by xterm.js on the client. The server has zero knowledge of terminal content.

## 7. Font Rendering

Delegated entirely to xterm.js configuration. The server can push `HtermPrefernces` (sic) via the `SetPreferences` message type, which includes `FontFamily`, `FontSize`, and `FontSmoothing` settings. However, these are hterm-style preference names — when using xterm.js (the default), font configuration is handled by xterm.js's own options.

xterm.js uses canvas-based text rendering: it measures monospace glyphs and draws them as bitmap characters on a `<canvas>` element. Subpixel rendering depends on the browser's canvas text rendering implementation.

## 8. Eye Strain Assessment: MEDIUM

**Rating: MEDIUM strain potential**

- **Canvas rendering** (xterm.js): Text is drawn as bitmap glyphs on canvas, not as native DOM text. This means no subpixel font rendering (ClearType/FreeType) — canvas text is typically grayscale-antialiased only. This is measurably worse for readability than native DOM text.
- **No explicit frame rate control:** xterm.js uses internal RAF-based rendering, which is generally well-behaved.
- **Cursor blink:** Handled by xterm.js internally (configurable via preferences).
- **Update mechanism:** Event-driven WebSocket data -> immediate write to terminal. No artificial batching delays, but also no jitter buffering. High-throughput output could cause rapid re-renders.
- **Base64 encoding overhead:** Every PTY read (up to 1024 bytes) is base64-encoded, adding ~33% bandwidth overhead. Not a strain factor directly but affects latency.

**Key strain factor:** Canvas-based text rendering lacks native font hinting and subpixel AA. For prolonged terminal use, this is the primary strain source compared to DOM-rendered alternatives.

## 9. Key Files

1. **`/c/dev/web-terminal-research/gotty-readonly/webtty/webtty.go`** — Core bidirectional bridge between WebSocket (master) and PTY (slave). Contains the main Run() loop, message encoding, and protocol handling.

2. **`/c/dev/web-terminal-research/gotty-readonly/server/handlers.go`** — WebSocket upgrade, authentication, and session lifecycle. Creates the webtty instance per connection.

3. **`/c/dev/web-terminal-research/gotty-readonly/backend/localcommand/local_command.go`** — PTY creation via kr/pty, read/write/resize/close operations.

4. **`/c/dev/web-terminal-research/gotty-readonly/webtty/message_types.go`** — Protocol message type definitions (the "webtty" protocol).

5. **`/c/dev/web-terminal-research/gotty-readonly/js/dist/gotty-bundle.js`** — Compiled frontend bundle containing xterm.js + WebSocket client + addons.
