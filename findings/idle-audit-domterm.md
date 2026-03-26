# DomTerm Idle Activity Audit

**Project:** DomTerm (terminal emulator, WebSocket-based)
**Date:** 2026-03-26
**Scope:** hlib/*.js (frontend), lws-term/*.c (backend), all .css files
**Idle score:** MINOR_ACTIVITY

## Summary Counts

| Category                | Total | Runs during idle? |
|-------------------------|-------|-------------------|
| setInterval             | 1     | Conditionally     |
| setTimeout (recursive)  | 2     | YES (text blink)  |
| requestAnimationFrame   | 2+5   | No                |
| CSS @keyframes          | 2     | YES (cursor blink)|
| CSS transitions         | ~18   | No (hover/drag)   |
| Network timers          | 3     | Conditionally     |
| MutationObserver        | 1     | Event-driven only |
| ResizeObserver          | 2     | Event-driven only |
| IntersectionObserver    | 1     | Event-driven only |
| C backend timers        | 0     | N/A               |

---

## 1. setInterval

### 1a. GoldenLayout popout checkReady
- **File:** hlib/goldenlayout.js:1 (minified)
- **Code:** `this._checkReadyInterval=setInterval((()=>this.checkReady()),10)`
- **Interval:** 10ms
- **Purpose:** Polls whether a popped-out window has finished initializing.
- **Cleared on idle?** YES -- cleared via `clearInterval(this._checkReadyInterval)` once the popout window's `__glInstance.isInitialised` becomes true.
- **Runs during idle?** Only if a popout window is opening. Not active during normal terminal use.
- **Disableable?** N/A, self-terminating.

**No other setInterval calls found in hlib/*.js.**

---

## 2. setTimeout (recursive / self-rescheduling)

### 2a. Text blink timer (SGCR blink attribute)
- **File:** hlib/terminal.js:604-624
- **Function:** `startBlinkTimer()` / inner `flip()` function
- **Interval:** Alternates between `_blinkShowTime` (default 700ms) and `_blinkHideTime` (default 300ms). Total cycle: 1000ms.
- **Purpose:** Toggles CSS class `blinking-hide` on `topNode` to show/hide text elements with the SGR "blink" attribute.
- **Runs during idle?** YES -- but ONLY if any terminal content has the SGR blink text-decoration. The flip function checks `buffers.querySelector('.term-style[text-decoration~="blink"]')` and stops (`_blinkEnabled = 0`) if no blinking elements exist.
- **Disableable?** Yes. Set `_blinkHideTime` to 0 via `setBlinkRate("0")` to disable. Also stops automatically when no blink-decorated text is present.

### 2b. Remote input confirm timer
- **File:** hlib/domterm.js:725-739
- **Function:** `remote_input_timer`
- **Interval:** `_remote_input_interval`, default 10000ms (10s) for remote sessions, 0 (disabled) for local.
- **Purpose:** Sends a "RECEIVED" acknowledgment to the backend after input inactivity. Ensures flow control stays in sync.
- **Runs during idle?** Conditionally. Fires once after last input, then stops (not truly recursive -- re-armed only on each `processInputBytes` call). During pure idle with no typing, it fires once after last keystroke then goes dormant.
- **Disableable?** Set `remote-input-interval` option to 0.

---

## 3. requestAnimationFrame

### 3a. Display update (terminal.js)
- **File:** hlib/terminal.js:6678-6682 (requestUpdateDisplay), hlib/terminal.js:433-446 (_updateDisplay)
- **Code:** `this._updateTimer = requestAnimationFrame(this._updateDisplay)`
- **Purpose:** Coalesces terminal DOM updates (line breaking, scrolling, caret positioning) into a single frame.
- **Continuous loop?** NO. One-shot. Requested when output arrives or display changes. The `_deferUpdate` mechanism can cause one re-schedule (up to `updateDeferLimit` = 200ms) to batch rapid updates, but it always terminates. Sets `_updateTimer = null` when done.
- **Runs during idle?** No. Only triggered by incoming data or user actions.
- **Disableable?** N/A, demand-driven.

### 3b. GoldenLayout (5 occurrences, all one-shot)
- **File:** hlib/goldenlayout.js:1 (minified, 5 RAF calls)
- **Purposes:**
  1. `propagateEventToLayoutManager` -- deferred event propagation, one-shot per event
  2. `updateTabSizes` -- recalculate tab widths after layout change, one-shot
  3. `updateSize` after splitter drag -- one-shot
  4. Remove drag element cleanup -- one-shot
  5. `_removeItem` during layout close -- one-shot
- **Continuous loop?** NO, all are one-shot.
- **Runs during idle?** No. All triggered by user layout actions (dragging, resizing, closing tabs).

---

## 4. CSS @keyframes Animations

### 4a. blinking-caret
- **File:** hlib/domterm-default.css:196-201
- **Keyframes:** 0% (visible) -> 30% (inherit/transparent) -> 100% (visible)
- **Applied to:** `span[caret="blinking-block"][std="caret"]`, etc. (lines 183-191)
- **Duration:** 1.5s per cycle, `steps(1)` timing
- **Iteration count:** `var(--caret-blink-count)`, default = **20** (line 148)
- **Runs during idle?** YES, but only 20 blinks (30 seconds total) then STOPS. Only active on `div.domterm-active` (focused terminal). Not infinite by default.
- **Disableable?** Yes. Use non-blinking caret style (block/underline/bar without "blinking-" prefix). Or set `--caret-blink-count: 0`.
- **Note:** The documentation (DomTerm.texi:4686-4688) shows users CAN set `--caret-blink-count: infinite` for perpetual blinking, but the default is 20.

### 4b. blinking-caret-lineend
- **File:** hlib/domterm-default.css:202-207
- **Same as 4a** but with `background-color: var(--input-line-color)` instead of `inherit`. Same iteration count (20), same behavior.

**No other @keyframes found in any .css file. No `animation: ... infinite` declarations found.**

---

## 5. Network Timers

### 5a. Remote input confirm timer
- **File:** hlib/domterm.js:739, hlib/terminal.js:6706-6714
- **Interval:** 10000ms (remote sessions), 0 (local sessions)
- **Purpose:** After user sends input, schedules a `_confirmReceived()` call to acknowledge received output bytes. Flow-control mechanism.
- **Runs during idle?** Fires once after last input, then dormant. Not periodic.
- **Disableable?** `remote-input-interval` setting.

### 5b. Remote output timeout
- **File:** hlib/domterm.js:747-759, hlib/terminal.js:6130-6135
- **Interval:** Default `2 * remote-output-interval` = 20000ms (20s). Set to 0 for local sessions.
- **Purpose:** Watchdog -- if no output received within timeout, calls `showConnectFailure(-1)`. Re-armed on every message.
- **Runs during idle?** YES for remote sessions. A single pending setTimeout is always active waiting for next output. Fires only on genuine timeout (connection loss). For local sessions, disabled (0).
- **Disableable?** Set `remote-output-timeout` to -1 or use local session.

### 5c. WebSocket reconnect
- **File:** hlib/domterm.js:791-800
- **Delays:** [0, 200, 400, 400]ms, max 3 retries then stops.
- **Purpose:** Reconnects after unexpected WebSocket close.
- **Runs during idle?** No. Only triggered by connection drop.
- **Disableable?** N/A, error recovery only.

---

## 6. Observers

### 6a. ResizeObserver (terminal)
- **File:** hlib/terminal.js:3751-3754
- **Code:** `new ResizeObserver(entries => { this.resizeHandler(); })`
- **Observed:** `this.contentElement`
- **Purpose:** Detects terminal size changes for reflow.
- **Runs during idle?** Event-driven only. No polling. Zero cost when size is static.

### 6b. ResizeObserver (GoldenLayout)
- **File:** hlib/domterm-layout.js:625-627
- **Code:** `new ResizeObserver(entries => { DomTermLayout.manager.updateSize(); })`
- **Observed:** Layout top element.
- **Purpose:** Propagates window resize to layout manager.
- **Runs during idle?** Event-driven only.

### 6c. ResizeObserver (GoldenLayout internal)
- **File:** hlib/goldenlayout.js:1 (minified)
- **Code:** `new ResizeObserver(()=>this.handleContainerResize())`
- **Purpose:** Internal layout container resize detection.
- **Runs during idle?** Event-driven only.

### 6d. MutationObserver (mini-buffer)
- **File:** hlib/terminal.js:4012-4015
- **Code:** `new MutationObserver(options.mutationCallback)`
- **Observed:** Mini-buffer element (childList, characterData, subtree).
- **Purpose:** Watches for changes to the mini-buffer content (used for command-line mini-buffer).
- **Runs during idle?** Event-driven only. No mutations = no callbacks.

### 6e. IntersectionObserver (table headers)
- **File:** hlib/terminal.js:6296-6308
- **Code:** `new IntersectionObserver(tableIntersectionCallback, options)`
- **Root:** `dt.buffers`, thresholds [0.01, 0.99].
- **Purpose:** Sticky table headers -- detects when table thead scrolls out of view to float it.
- **Runs during idle?** Event-driven only. Active only while scrolling tables.

---

## 7. C Backend Timers

### No timer code found in lws-term/*.c

The `lws-term/` directory contains only:
- `junzip.c` -- ZIP file reading (no timers)
- `whereami.c` -- executable path detection (no timers)
- `server.h` -- declares `lws_callback` functions and structs but no timer setup

The actual server C code (with lws timer setup, pty I/O callbacks, etc.) is NOT present in this repository snapshot. Only headers and utility files are included. The libwebsockets event loop itself has built-in idle/service timeouts, but the DomTerm-specific timer configuration cannot be audited from these files alone.

---

## 8. Other One-Shot Timeouts (not recursive, no idle impact)

For completeness, these setTimeout calls exist but are purely one-shot, event-triggered, and do NOT run during idle:

| File | Line | Purpose | Interval |
|------|------|---------|----------|
| domterm-client.js | 526 | Debounce window resize | 100ms |
| domterm-parser.js | 1723 | Deferred reverse-video update | 50ms |
| domterm-parser.js | 2183 | afterOutputHook timeout | configurable |
| domterm-layout.js | 265 | Deferred popout window open | 0ms |
| terminal.js | 1137 | Deferred layout close | 0ms |
| terminal.js | 3780 | Clear notification overlay | configurable |
| terminal.js | 3868-3878 | Hover popup enter/leave delays | configurable |
| terminal.js | 6701 | Delete pending echo timeout | 400ms default |
| terminal.js | 11014 | Password hide delay | configurable |

---

## Idle Score: MINOR_ACTIVITY

### What runs during true idle (cursor blinking in focused terminal, no typing, no output):

1. **CSS cursor blink animation** -- 1.5s cycle, but LIMITED to 20 iterations (30 seconds), then stops. GPU-composited via CSS, minimal CPU cost while running.
2. **Text blink timer** (if SGR blink text exists) -- 1s cycle setTimeout chain. Unlikely in normal terminal use. Self-terminates when no blink elements present.
3. **Remote output watchdog** (remote sessions only) -- single pending setTimeout, 20s. Fires only on genuine timeout.

### What does NOT run during idle:
- No setInterval is active during idle
- No requestAnimationFrame loop
- No continuous CSS animation (cursor stops after 20 blinks)
- No WebSocket keepalive/ping from JS side
- All observers are event-driven (zero CPU when nothing changes)

### Comparison notes:
DomTerm is notably well-behaved for idle. The cursor blink uses CSS animation with a finite iteration count (20) rather than an infinite JS timer or infinite CSS animation. This is better than most terminals. The text-blink feature (SGR attribute) does use a JS setTimeout chain, but it auto-stops when no blinking content exists. The only perpetual idle activity in remote sessions is a single watchdog timeout that fires once then shows an error -- not a polling loop.
