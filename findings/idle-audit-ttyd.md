# ttyd Idle Activity Audit

**Audited version:** ttyd source at `/c/dev/web-terminal-research/ttyd`
**Date:** 2026-03-26
**Scope:** All files in `html/src/` (frontend) and `src/` (C backend), excluding node_modules

## Summary

| Category | Count | Idle Impact |
|---|---|---|
| setInterval | 0 | None |
| Recursive setTimeout | 0 | None |
| requestAnimationFrame | 0 | None |
| CSS @keyframes / infinite animation | 0 | None |
| Network timers (ping/pong) | 1 | YES - runs during idle |
| MutationObserver / ResizeObserver / IntersectionObserver | 0 | None |
| C backend timers (uv_timer, lws_set_timer) | 0 | None |
| xterm.js cursor blink (inherited) | 1 | YES - runs during idle (library default) |

**Idle Activity Score: MINOR_ACTIVITY**

Two sources of idle activity: WebSocket ping/pong every 5 seconds (server-driven, protocol-level), and xterm.js cursor blink animation (library default, not set by ttyd). ttyd's own code adds zero periodic timers or animation loops.

---

## Detailed Findings

### 1. setInterval

**None found.** Zero occurrences in `html/src/`.

---

### 2. setTimeout (recursive / re-scheduling)

**None that re-schedule during idle.**

| File | Line | Purpose | Interval | Idle? | Disableable? |
|---|---|---|---|---|---|
| `html/src/components/terminal/xterm/addons/overlay.ts` | 62-71 | Fade out overlay notification after display timeout | 1500ms then 200ms (one-shot pair) | NO - only fires after user action triggers overlay (resize, copy, reconnect) | N/A |

The overlay setTimeout is strictly event-driven: it fires once to fade out a notification, then removes the DOM node. Not recursive, not idle.

---

### 3. requestAnimationFrame

**None found.** Zero occurrences in `html/src/`. ttyd delegates all rendering to xterm.js (which uses rAF internally for its canvas/webgl renderer, but that is in the library, not ttyd code).

Note: xterm.js's canvas/webgl addon does run a rAF render loop internally. This is covered in the xterm.js audit, not here. ttyd defaults to `rendererType: 'webgl'` (`html/src/components/app.tsx:13`).

---

### 4. CSS Animations

**None found.** Zero `@keyframes` or `animation` properties in:
- `html/src/style/index.scss` (19 lines, layout only)
- `html/src/components/modal/modal.scss` (82 lines, static styles only)

The `-webkit-transition: opacity 180ms ease-in` in `overlay.ts:19` is a CSS transition (not animation), and only triggers on overlay show/hide events.

---

### 5. Network Timers (WebSocket ping/pong)

| File | Line | Purpose | Interval | Idle? | Disableable? |
|---|---|---|---|---|---|
| `src/server.c` | 48 | `secs_since_valid_ping = 5` in lws retry policy | 5000ms | **YES** | Yes: `--ping-interval` / `-P` CLI flag |
| `src/server.c` | 49 | `secs_since_valid_hangup = 10` (disconnect if no pong in 10s) | 10000ms | **YES** | Yes: derived from ping-interval + 7 |
| `src/server.c` | 542 | `ws_ping_pong_interval = 5` (older lws <4.0 path) | 5000ms | **YES** | No (hardcoded for lws <4.0) |

**Mechanism:** libwebsockets sends WebSocket protocol-level PING frames every 5 seconds. The client browser responds with PONG automatically (browser handles this at protocol level, no JS code needed). If no valid PONG received within 10 seconds, connection is terminated.

**JS client side:** No JS-level ping/heartbeat timer. The WebSocket PONG response is handled by the browser's WebSocket implementation automatically.

**Impact:** One 5-second interval server-to-client PING frame (~2 bytes payload). Minimal CPU, but non-zero network and lws event loop wakeup.

**Disableable:** Set `--ping-interval 0` to disable (lws >=4.0 only). For lws <4.0, hardcoded at 5s.

---

### 6. Observers (MutationObserver / ResizeObserver / IntersectionObserver)

**None found.** Zero occurrences in `html/src/`.

ttyd uses a `window.resize` event listener (`xterm/index.ts:202`) which calls `fitAddon.fit()`. This is event-driven (only fires on actual window resize), not a polling observer.

---

### 7. C Backend Timers

**No uv_timer usage.** No `lws_set_timer` calls. No periodic callbacks.

The only libuv primitives used:
- `uv_signal_t` for SIGINT/SIGTERM handling (`server.c:617-621`) - signal, not timer
- `uv_async_t` in `pty_process_` struct (`pty.h:46`) - async notification from PTY thread, not periodic
- `uv_pipe_t` for PTY I/O (`pty.h:47-48`) - data-driven, not periodic
- `uv_loop_t` as main event loop (`server.h:85`) - runs lws service

The C backend is fully event-driven: PTY read callbacks fire only when the child process produces output. No polling.

---

### 8. Inherited from xterm.js (not ttyd code, but relevant)

| Source | Mechanism | Interval | Idle? | Disableable? |
|---|---|---|---|---|
| xterm.js library | Cursor blink (setInterval ~600ms) | ~600ms | **YES** | Yes: `terminal.options.cursorBlink = false` or via ttyd `-t cursorBlink=false` |
| xterm.js webgl/canvas addon | rAF render loop | 16ms | Depends on xterm.js dirty-flag logic | Switch to `rendererType: 'dom'` via `-t rendererType=dom` |

ttyd does NOT set `cursorBlink` in its default `termOptions` (`app.tsx:23-48`). xterm.js defaults `cursorBlink` to `false`, so **cursor blink is OFF by default** unless the server operator passes `-t cursorBlink=true`.

ttyd defaults to WebGL renderer (`app.tsx:13: rendererType: 'webgl'`). The xterm.js webgl addon runs a rAF loop, but with dirty-flag checking (see xterm.js audit for details).

---

## Architecture Notes

ttyd's own code is remarkably clean for idle:

1. **No polling** - All I/O is event-driven (libuv + libwebsockets)
2. **No intervals** - Zero setInterval in ttyd code
3. **No animation loops** - Zero rAF in ttyd code
4. **No CSS animations** - Static styles only
5. **Data flow is push-based** - PTY output -> uv callback -> lws_callback_on_writable -> WS message -> terminal.write()

The only idle activity comes from:
- WebSocket ping/pong (5s, server-driven, configurable)
- Whatever xterm.js does internally (cursor blink if enabled, renderer loop)

---

## Idle Activity Score: MINOR_ACTIVITY

**Rationale:** ttyd itself is nearly silent at idle. The 5-second WS ping is standard keepalive and barely registers. The main question for display-strain research is what xterm.js does with its renderer internally (covered in separate xterm.js audit). ttyd's code adds no additional idle overhead beyond what the terminal library requires.
