# WeTTY - Static Code Analysis Findings

**Repository:** https://github.com/butlerx/wetty
**Version analyzed:** 2.7.0
**Analysis date:** 2026-03-26

---

## 1. Tech Stack

### package.json

**Dependencies (key):**
- `@xterm/xterm` ^5.2.0 — terminal emulator
- `@xterm/addon-fit` ^0.10.0 — auto-fit terminal to container
- `@xterm/addon-image` ^0.8.0 — inline image support (sixel/iTerm2)
- `@xterm/addon-web-links` ^0.11.0 — clickable URLs
- `express` ^4.17.1 — HTTP server
- `socket.io` ^4.5.1 / `socket.io-client` ^4.5.1 — WebSocket transport
- `node-pty` ^0.10.0 — pseudo-terminal
- `compression`, `helmet`, `winston`, `prom-client` — middleware/logging/metrics
- `sass` ^1.54.4 — SCSS compilation
- `toastify-js` — toast notifications

**DevDependencies (key):**
- `typescript` ^5.1.3
- `esbuild` ^0.21.5 — bundler (no webpack/vite)
- `mocha` + `chai` + `sinon` — test framework
- `eslint` + `prettier` — linting

**Scripts:**
- `build` — `node build.js` (esbuild-based)
- `dev` — concurrent build watch + nodemon
- `start` — `NODE_ENV=production node .`
- `test` — mocha
- `lint` / `lint:fix` — eslint

### Language Breakdown

| Type | Count |
|------|-------|
| TypeScript (.ts) | 45 |
| JavaScript (.js) | 5 (xterm config files) |
| SCSS (.scss) | 5 |

**Framework:** Express.js (HTTP server) + Socket.IO (real-time transport)

---

## 2. Rendering Pipeline

### xterm.js Version

`@xterm/xterm` ^5.2.0 (the `@xterm/` scoped packages — post-rename from `xterm`).

### Addons Loaded

In `src/client/wetty/term.ts`, the `Term` constructor loads exactly 3 addons:

```typescript
this.loadAddon(new FitAddon());       // @xterm/addon-fit
this.loadAddon(new WebLinksAddon());  // @xterm/addon-web-links
this.loadAddon(new ImageAddon());     // @xterm/addon-image
```

**No WebGL or Canvas addon is loaded.** There is no import of `@xterm/addon-webgl` or `@xterm/addon-canvas` anywhere in the source.

### rendererType Configuration

The defaults config (`src/assets/xterm_config/xterm_defaults.js`) sets:
```javascript
rendererType: 'canvas'
```

The advanced options UI (`xterm_advanced_options.js`) exposes a dropdown with choices `['canvas', 'dom']`. Note: in xterm.js v5, the `rendererType` option is **deprecated** — the default renderer is the DOM renderer, and `CanvasAddon`/`WebglAddon` must be explicitly loaded as addons to use those renderers. Since neither addon is loaded, **the actual renderer is the DOM renderer regardless of this config value.**

### Renderer Classification

**DOM renderer** — xterm.js default. Text rendered as `<span>` elements inside the terminal container. No canvas, no WebGL.

---

## 3. Update/Flush Mechanism

### RAF / Dirty Flags / Debounce

- **No explicit `requestAnimationFrame` usage** in WeTTY source code. All rendering is delegated to xterm.js internals.
- **One `debounce` call** in `term.ts` (line 100) — used only for a workaround to simulate backspace after CTRL+key on mobile, not for rendering.
- **`term.refresh(0, this.rows - 1)`** is called on resize (`resizeTerm()`), triggering a full xterm.js repaint.
- **Flow control** acts as an implicit update throttle: the server-side `tinybuffer` accumulates PTY output for up to 2ms or 512KB before emitting a Socket.IO message. This batches small writes.

### Classification

**Delegated to xterm.js** — WeTTY itself has no rendering loop. Data arrives via Socket.IO `data` events and is passed directly to `term.write()`. xterm.js internally batches writes and uses RAF for rendering.

---

## 4. Transport

### WebSocket

**Socket.IO v4** over WebSocket (with HTTP long-polling fallback).

- Server: `new Server(server, { path: '${path}/socket.io', pingInterval: 3000, pingTimeout: 7000 })`
- Client: `io(window.location.origin, { path: '${trim(socketBase)}/socket.io' })`

Events:
- `input` — client-to-server keystrokes
- `data` — server-to-client terminal output
- `resize` — bidirectional terminal dimensions
- `commit` — flow control acknowledgment
- `login` / `logout` / `disconnect` — session lifecycle

### SSH vs Direct PTY

**Both.** WeTTY spawns a PTY via `node-pty` and runs either:

1. **SSH client** (`ssh -t <host>`) — the default mode, connecting to a remote SSH server. Uses the system `ssh` binary (not the `ssh2` JS library). Password auth optionally via `sshpass`.
2. **Local login** (`login -h <address>`) — when running as root and host is localhost.

The command is determined in `src/server/command.ts`:
- If not `forcessh` and host is localhost and running as root: uses `login`
- Otherwise: builds SSH command args via `src/server/command/ssh.ts`

No `ssh2` library, no `child_process.spawn` directly — always goes through `node-pty`.

---

## 5. PTY

### Library

**node-pty** ^0.10.0

### Configuration

From `src/server/shared/xterm.ts`:
```typescript
const xterm: IPtyForkOptions = {
  name: 'xterm-256color',
  cols: 80,
  rows: 30,
  cwd: process.cwd(),
  env: { ...process.env },
};
```

### Default Shell

Not a shell directly — spawns `/usr/bin/env` with either `ssh` or `login` as arguments. The user's shell is determined by the remote SSH host or the `login` command.

---

## 6. ANSI/VT Processing

**Client-side.** The server sends raw PTY output bytes to the client over Socket.IO. The xterm.js library on the client side parses all ANSI escape sequences, VT control codes, and renders them. The server performs no terminal emulation — it is a transparent pipe between PTY and WebSocket.

The PTY terminal type is set to `xterm-256color`, so the remote application generates xterm-compatible escape sequences that xterm.js understands natively.

---

## 7. Font Rendering

From `src/assets/xterm_config/xterm_defaults.js`:

```javascript
fontFamily: 'courier-new, courier, monospace',
fontSize: 15,
fontWeight: 'normal',
fontWeightBold: 'bold',
lineHeight: 1.0,
letterSpacing: 0,
```

No custom web fonts loaded. Falls back to system monospace fonts. The CSS (`terminal.scss`) contains no font declarations — all font configuration is handled through xterm.js options.

With the DOM renderer, fonts are rendered by the browser's standard text layout engine (no canvas `measureText` or WebGL glyph atlas involved).

---

## 8. Strain Assessment

**Medium**

Reasoning:

**Positive factors (reduce strain):**
- DOM renderer means browser-native text rendering with subpixel antialiasing (ClearType on Windows, LCD font smoothing on macOS). This is the best possible text quality.
- No canvas/WebGL rendering avoids the glyph atlas quality issues that plague those renderers.
- Flow control prevents overwhelming the client with data, providing backpressure.
- Server-side `tinybuffer` batches small writes (2ms / 512KB threshold), reducing per-message overhead.
- Default `lineHeight: 1.0` and `letterSpacing: 0` avoid spacing artifacts.

**Negative factors (increase strain):**
- DOM renderer is the **slowest** xterm.js renderer. Heavy terminal output (e.g., scrolling logs) causes significant layout thrashing as the browser must reflow/repaint hundreds of span elements. This can cause visible jank and dropped frames during fast output.
- Socket.IO adds overhead vs raw WebSocket — protocol framing, JSON encoding of events, ping/pong every 3s.
- Default font is `courier-new` — a dated font with poor legibility compared to modern coding fonts (Cascadia Code, JetBrains Mono, etc.).
- `cursorBlink: false` is fine, but `scrollback: 1000` is relatively low — heavy scrollback usage could cause reflows.
- No WebGL addon means no GPU acceleration for rendering at all.

**Net assessment:** Acceptable for typical SSH sessions. DOM renderer gives excellent text quality but will cause visible stutter during high-throughput output (cat large-file, build logs). The flow control mitigates but does not eliminate this. For extended use, the default font choice could contribute to fatigue.

---

## 9. Key Files

| File | Role |
|------|------|
| `src/client/wetty/term.ts` | Terminal class — creates xterm.js instance, loads addons (FitAddon, WebLinksAddon, ImageAddon), handles resize |
| `src/client/wetty.ts` | Client entry point — connects Socket.IO, wires data flow between socket and terminal, flow control |
| `src/server/spawn.ts` | Server PTY — spawns node-pty process, wires PTY data to socket via tinybuffer, server-side flow control |
| `src/server/flowcontrol.ts` | Server flow control — tinybuffer (2ms/512KB batching), high/low watermark PTY pause/resume |
| `src/assets/xterm_config/xterm_defaults.js` | Default xterm.js options — font, renderer, theme, scrollback |
