# shellinabox Idle Activity Audit

**Project:** Shell In A Box (shellinabox)
**Path:** /c/dev/web-terminal-research/shellinabox-readonly
**Date:** 2026-03-26
**Method:** Static code analysis only (JS in shellinabox/*.jspp, CSS in shellinabox/*.css, C backend in libhttp/ and shellinabox/)

## Summary

| Category | Count | Idle Impact |
|---|---|---|
| setInterval | 1 (cursor blink, 500ms, NEVER cleared) | HIGH - runs forever once started |
| setTimeout recursive | 1 (AJAX long-poll loop) | HIGH - continuous HTTP POST cycle |
| setTimeout one-shot | 4 (init, resize indicator, paste, flash) | NONE at idle |
| requestAnimationFrame | 0 | NONE |
| CSS animations | 0 (no @keyframes anywhere) | NONE |
| AJAX polling | 1 (the long-poll IS the data transport) | HIGH - always pending or re-firing |
| Observers | 0 (pre-Observer API codebase) | NONE |
| C backend timers | 0 periodic (poll()-based event loop with idle timeout) | NONE - blocks on poll() |

**Idle Score: BUSY**

## Detailed Findings

### 1. setInterval

#### 1a. Cursor blink + composed key check (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 3201
- **Code:**
  ```
  this.cursorInterval = setInterval(
    function(vt100) {
      return function() {
        vt100.animateCursor();
        vt100.checkComposedKeys();
      }
    }(this), 500);
  ```
- **Interval:** 500ms
- **Cleared:** NEVER. There is no `clearInterval(this.cursorInterval)` anywhere in the codebase. The interval is guarded by `if (!this.cursorInterval)` to prevent duplicates, but once started it runs for the lifetime of the page.
- **What it does:**
  1. Toggles cursor between `bright` and `dim` CSS classes (or stays `inactive` if blurred)
  2. Calls `checkComposedKeys()` which reads `this.input.value` to detect composed/pasted text
- **Idle impact:** HIGH. This fires every 500ms forever, even when:
  - The terminal is idle with no output
  - The user has switched to another tab
  - The session has closed
  - The cursor is inactive/blurred (still fires, just sets `inactive` class)
- **Display impact:** When focused, toggles cursor opacity between full and 0.2 every 500ms via CSS class change. This is a DOM modification that triggers repaint. When blurred, still fires but the class change is a no-op (already `inactive`). The `checkComposedKeys()` call also reads DOM state every 500ms unnecessarily.

### 2. setTimeout (recursive / self-rescheduling)

#### 2a. AJAX long-poll loop (shell_in_a_box.jspp)
- **File:** `shellinabox/shell_in_a_box.jspp` lines 169-228
- **Mechanism:** This is the core data transport. There is NO WebSocket; all communication is HTTP POST long-polling.
- **Flow:**
  1. `sendRequest()` sends POST to server with session ID
  2. Server holds the request (long-poll, up to `AJAX_TIMEOUT=45s`)
  3. `onReadyStateChange()` fires when response arrives
  4. If status 200 and session alive: immediately calls `this.sendRequest(request)` (line 216) -- recursive
  5. If status 0 (timeout/connection error): `setTimeout(..., 1000)` then retries (line 220-224) -- 1s delay
  6. If session closed: stops
- **Request timeout:** 30,000ms (`request.timeout = 30000` at line 174)
- **Idle impact:** HIGH. Even when the terminal is completely idle:
  - Client sends POST
  - Server holds it for up to 45s waiting for PTY output
  - Server responds (empty or with timeout)
  - Client immediately sends another POST
  - Cycle repeats forever
  - This means there is ALWAYS an HTTP request in flight or being immediately re-sent
- **Network overhead at idle:** One HTTP POST round-trip every ~45 seconds (server AJAX_TIMEOUT) plus the POST headers on each cycle. Less frequent than WebSocket ping but heavier per-request (full HTTP headers).

#### 2b. Error retry (shell_in_a_box.jspp)
- **File:** `shellinabox/shell_in_a_box.jspp` line 220-224
- **Code:** `setTimeout(function(shellInABox) { return function() { shellInABox.sendRequest(); }; }(this), 1000);`
- **Fires only on:** HTTP status 0 (connection failure)
- **Delay:** 1000ms before retry
- **Idle impact:** NONE under normal operation. Only during connection problems.

### 3. setTimeout (one-shot, non-recursive)

#### 3a. Initial bootstrap (shell_in_a_box.jspp)
- **File:** `shellinabox/shell_in_a_box.jspp` line 119
- **Code:** `setTimeout(function(shellInABox) { return function() { shellInABox.messageInit(); shellInABox.sendRequest(); }; }(this), 1);`
- **Delay:** 1ms. Fires once on construction to kick off the first request.
- **Idle impact:** NONE. One-shot at startup.

#### 3b. Size indicator delay (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 1012
- **Code:** `setTimeout(function(vt100) { return function() { vt100.indicateSize = true; }; }(this), 100);`
- **One-shot, 100ms.** Enables size indicator display after initial render.

#### 3c. Resize size display (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 1315
- **Code:** `this.curSizeTimeout = setTimeout(function(vt100) { ... vt100.curSizeTimeout = null; }, timeout);`
- **Cleared:** YES, via `clearTimeout(this.curSizeTimeout)` at line 1305 before setting new one.
- **One-shot on resize events.**

#### 3d. Paste double-paste prevention (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 1445
- **One-shot timeout to prevent double-paste on Chrome/Linux.**

#### 3e. Visual bell flash (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 3238
- **Code:** `setTimeout(function(vt100) { ... vt100.refreshInvertedState(); }, 100);`
- **One-shot, 100ms.** Reverts screen inversion after visual bell.

#### 3f. Print window timeout (vt100.jspp)
- **File:** `shellinabox/vt100.jspp` line 3609
- **One-shot for print functionality.**

### 4. requestAnimationFrame

**None.** This is a pre-RAF era codebase. All rendering is synchronous DOM manipulation.

### 5. CSS Animations

**None.** Zero `@keyframes` rules in any of the CSS files:
- `styles.css` -- structural layout only
- `white-on-black.css` -- color definitions only
- `black-on-white.css` -- color definitions only
- `color.css` -- 256-color palette
- `monochrome.css` -- inherit-based monochrome
- `print-styles.css` -- print layout only

Cursor blink is entirely JS-driven via the 500ms setInterval toggling CSS classes (`bright`, `dim`, `inactive`). The `dim` class uses `opacity: 0.2` -- this is a static property, not animated.

### 6. AJAX Polling (the long-poll transport)

This is the most significant idle activity source. See section 2a for full details.

**Architecture:** shellinabox uses HTTP long-polling as its sole transport. There is no WebSocket support. The cycle is:

1. Client POSTs with session ID and terminal dimensions
2. Server holds the connection, blocking on `poll()` watching the PTY fd
3. When PTY has output OR timeout (45s), server responds with JSON `{data: "...", session: "..."}`
4. Client processes response and immediately POSTs again
5. Repeat forever

**At idle:** The server's poll() blocks for up to 45s, then times out, sends an empty response, and the client re-sends. So the steady-state idle behavior is one HTTP round-trip every ~45 seconds.

### 7. Observers

**None.** This codebase predates the Observer APIs. Window resize is handled via `window.addEventListener('resize', ...)` (vt100.jspp line 1017). No MutationObserver, ResizeObserver, or IntersectionObserver.

### 8. C Backend Timers

#### 8a. Server event loop (libhttp/server.c)
- **File:** `libhttp/server.c` line 535, `serverLoop()`
- **Mechanism:** `poll()` system call with a computed timeout
- **Timeout calculation:** Derived from the earliest connection timeout. Default connection timeout is `CONNECTION_TIMEOUT = 600s` (10 minutes). AJAX session timeout is `AJAX_TIMEOUT = 45s`.
- **Behavior:** The main loop calls `poll()` which blocks the process until either:
  - A file descriptor has an event (new connection, PTY output, client data)
  - The timeout expires (to handle connection timeouts)
- **Idle impact:** NONE in terms of CPU. `poll()` is a kernel-level block. The process consumes zero CPU while waiting. When idle, it wakes approximately every 45s to handle the AJAX long-poll timeout, sends the response, and blocks again on the next poll().

#### 8b. No periodic timers
- No `setitimer()`, `alarm()`, `SIGALRM`, `timer_create()`, or `timerfd` anywhere in the C code.
- The server is purely event-driven via `poll()`.

## Idle Behavior Summary

When the terminal is connected and the user is doing nothing:

1. **Cursor blink fires every 500ms** (setInterval, NEVER cleared) -- DOM class toggle + checkComposedKeys() input read
2. **AJAX long-poll cycles every ~45s** (HTTP POST, server holds then responds, client immediately re-sends)
3. **C backend blocks on poll()** -- zero CPU until AJAX timeout

When the terminal tab is not visible:
1. **Cursor blink STILL fires every 500ms** -- the interval is never cleared, even on blur. The `animateCursor(true)` sets class to `inactive` but the timer keeps firing.
2. **AJAX long-poll continues** -- no visibility-based throttling
3. Browser may throttle the 500ms timer to 1000ms in background tabs (browser behavior, not shellinabox code)

When the session is closed:
1. **Cursor blink STILL fires every 500ms** -- never cleared
2. **AJAX long-poll stops** -- `sessionClosed()` does not re-send

**Primary idle concerns for display strain:**
- The 500ms cursor blink is never stopped. It is the most aggressive idle timer in any project audited so far.
- The `checkComposedKeys()` piggy-backed on the blink timer reads DOM state every 500ms, adding unnecessary work even when no input is occurring.
- The AJAX long-poll is architecturally unavoidable (it IS the transport) but at ~45s intervals it is low-frequency.
