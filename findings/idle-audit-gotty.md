# GoTTY Idle Activity Audit

**Project:** GoTTY (github.com/yudai/gotty)
**Path:** /c/dev/web-terminal-research/gotty-readonly
**Date:** 2026-03-26
**Method:** Static code analysis only (JS source in js/src/, bundle at js/dist/gotty-bundle.js, Go backend)

## Summary

| Category | Count | Idle Impact |
|---|---|---|
| setInterval | 3 (1 app-level, 2 xterm.js internal) | HIGH - cursor blink 600ms + ping 30s always running |
| setTimeout recursive | 1 (reconnect, conditional) | NONE when connected |
| requestAnimationFrame | 2 patterns (xterm.js internal) | LOW - on-demand only, not looping |
| CSS animations | 1 (hterm blink-node in bundle) | MINOR - only if hterm mode + blink content |
| WebSocket ping | 1 client-side setInterval | MEDIUM - 30s ping even when idle |
| Observers | 0 (no MutationObserver/ResizeObserver/IntersectionObserver) | NONE |
| Go backend timers | 1 (timeout timer, not periodic) | NONE - event-driven |

**Idle Score: MINOR_ACTIVITY**

## Detailed Findings

### 1. setInterval

#### 1a. WebSocket ping timer (GoTTY app code)
- **File:** `js/src/webtty.ts` line 97
- **Code:** `pingTimer = setInterval(() => { connection.send(msgPing) }, 30 * 1000);`
- **Interval:** 30,000ms (30 seconds)
- **Cleared:** YES, on connection close (`clearInterval(pingTimer)` at line 127)
- **Idle impact:** MEDIUM. Sends a WebSocket message every 30s even when terminal is completely idle. This keeps the connection alive but generates network traffic and wakes the CPU.

#### 1b. xterm.js cursor blink timer (in bundle)
- **File:** `js/dist/gotty-bundle.js` (xterm.js internal)
- **Code:** `this.cursorBlinkInterval=setInterval(function(){t.element.classList.toggle("xterm-cursor-blink-on")},600)`
- **Interval:** 600ms
- **Cleared:** YES, via `clearCursorBlinkingInterval()` on blur. Restarted on focus via `restartCursorBlinking()`.
- **Idle impact:** HIGH when focused. Toggles a CSS class every 600ms, causing a DOM repaint. Stops when terminal loses focus. This is the primary source of idle display activity.

#### 1c. xterm.js drag-scroll timer (in bundle)
- **File:** `js/dist/gotty-bundle.js` (xterm.js SelectionManager)
- **Code:** `this._dragScrollIntervalTimer=setInterval(function(){return e._dragScroll()},50)`
- **Interval:** 50ms
- **Cleared:** YES, on mouseup via `_removeMouseDownListeners()` which calls `clearInterval(this._dragScrollIntervalTimer)`
- **Idle impact:** NONE at idle. Only active during mouse-drag text selection operations.

### 2. setTimeout (recursive / self-rescheduling)

#### 2a. Reconnect timer
- **File:** `js/src/webtty.ts` line 131
- **Code:** `reconnectTimeout = setTimeout(() => { connection = this.connectionFactory.create(); ... setup(); }, this.reconnect * 1000);`
- **Condition:** Only fires when `this.reconnect > 0` AND connection has closed.
- **Idle impact:** NONE during normal connected operation. Only triggers after disconnect.

#### 2b. Message overlay dismissal (xterm.ts)
- **File:** `js/src/xterm.ts` line 58
- **Code:** `this.messageTimer = setTimeout(() => { this.elem.removeChild(this.message); }, timeout);`
- **One-shot, not recursive.** Default timeout is 2000ms.
- **Idle impact:** NONE. Fires once when a message is shown (e.g., on resize).

### 3. requestAnimationFrame

#### 3a. xterm.js Renderer refresh loop (in bundle)
- **Code:** `this._refreshAnimationFrame=window.requestAnimationFrame(this._refreshLoop.bind(this))`
- **Behavior:** On-demand. Only scheduled when `queueRefresh()` is called (i.e., when terminal output arrives). The RAF fires, processes queued row refreshes, then STOPS. It is NOT a continuous loop.
- **Idle impact:** NONE at idle. Only active when new terminal output arrives.

#### 3b. xterm.js SelectionManager refresh (in bundle)
- **Code:** `this._refreshAnimationFrame=window.requestAnimationFrame(function(){return t._refresh()})`
- **Behavior:** On-demand, triggered by selection changes. Single frame, not looping.
- **Idle impact:** NONE at idle.

### 4. CSS Animations

#### 4a. hterm blink-node animation (in bundle, from hterm/libapps)
- **Code in bundle:** `.blink-node { animation-name: blink; animation-duration: var(--hterm-blink-node-duration); animation-iteration-count: infinite; ... }`
- **Condition:** Only relevant when using hterm mode (`gotty_term == "hterm"`) AND the terminal receives blink-attribute text (ANSI blink escape sequence).
- **Idle impact:** MINOR. If hterm mode is used AND blink text is present, this CSS animation runs continuously via the compositor. In the default xterm mode, irrelevant.

#### 4b. GoTTY's own CSS
- **Files:** `resources/index.css`, `resources/xterm_customize.css`
- **Content:** No @keyframes, no animation properties. Only static styles and a CSS `transition: opacity 180ms ease-in` on `.xterm-overlay` (message overlay). Transitions only fire on state change, not continuously.

### 5. WebSocket Ping/Pong

#### Client side (JS)
- GoTTY sends application-level pings via `setInterval` every 30s (see 1a above).
- Message format: sends `msgPing = '2'` over the WebSocket text channel.
- Server responds with `msgPong = '2'`.
- This is NOT the WebSocket protocol-level ping/pong; it is application-layer keepalive over the data channel.

#### Server side (Go / gorilla/websocket)
- gorilla/websocket has a default `SetPingHandler` that auto-responds to protocol-level pings with a pong. But GoTTY's server code does NOT configure any periodic protocol-level pings. The server does NOT call `SetPingHandler` or `WriteControl(PingMessage, ...)` anywhere in server/*.go.
- The default gorilla handler responds to pings from the browser, but browsers do not send WebSocket protocol pings.
- **Net result:** Only the client-side 30s application ping is active. No server-initiated pings.

### 6. Observers

- **MutationObserver:** Not used anywhere in GoTTY source or bundle.
- **ResizeObserver:** Not used. Window resize is handled via `window.addEventListener("resize", ...)` in `js/src/xterm.ts` line 34.
- **IntersectionObserver:** Not used.
- The bundle contains hterm's `PreferenceManager` observer pattern (custom event observers, not DOM Observers). These are callback-based, not polling.

### 7. Go Backend Timers

#### 7a. Connection timeout counter
- **File:** `server/handler_atomic.go`
- **Type:** `time.Timer` (one-shot), NOT `time.Ticker` (periodic).
- **Behavior:** If a server `Timeout` is configured (default: 0 = disabled), a timer fires once after all connections drop to zero for the configured duration. This shuts down the server.
- **Idle impact:** NONE. Not periodic. When timeout is 0 (default), the timer is drained immediately and never fires.

#### 7b. WebTTY Run loop
- **File:** `webtty/webtty.go`
- **Behavior:** Two goroutines block on `Read()` calls (one for the slave/PTY, one for the master/WebSocket). These are blocking I/O, NOT polling. No `time.Ticker`, no `time.After` loops.
- **Idle impact:** NONE. Goroutines sleep on blocking reads when idle.

## Idle Behavior Summary

When the terminal is connected and the user is doing nothing:

1. **Cursor blinks every 600ms** (xterm.js setInterval, toggles CSS class) -- causes DOM repaint
2. **Ping sent every 30s** (application-level WebSocket message) -- minimal network traffic
3. Everything else is dormant (blocking I/O on backend, no RAF loops, no polling)

When the terminal window is blurred/unfocused:
1. Cursor blink interval is CLEARED (xterm.js calls `clearCursorBlinkingInterval` on blur)
2. Ping continues every 30s
3. Effectively near-zero display activity

**Primary idle concern for display strain:** The 600ms cursor blink is the only continuous visual update. It is cleared on blur, which is correct behavior.
