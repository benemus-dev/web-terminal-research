# hterm Idle Activity Audit

**Source:** `libapps-mirror/hterm/js/` (production files only, tests excluded)
**Date:** 2026-03-26

## Summary

| Category | Count | Runs During Idle? |
|---|---|---|
| setInterval | 1 | NO (mouse-drag only) |
| setTimeout (recursive / self-rescheduling) | 1 | YES -- cursor blink |
| setTimeout (one-shot, event-driven) | ~12 | NO |
| requestAnimationFrame | 0 | -- |
| CSS @keyframes animation | 1 | YES -- ANSI blink text |
| MutationObserver / ResizeObserver | 0 | -- |
| Network timers | 0 | -- |

**Idle Score: MINOR_ACTIVITY**

Two sources of continuous idle activity: cursor blink (JS setTimeout chain) and ANSI blink text (CSS animation). Both are disableable via preferences and both default to reasonable states (cursor blink OFF by default, text blink ON by default but only activates when ANSI SGR 5 text exists on screen).

---

## Detailed Findings

### 1. setInterval -- 1 occurrence

**hterm_scrollport.js:363** -- Auto-scroll during mouse drag selection
```
setInterval(this.autoScroll_.bind(this), 200)
```
- **Purpose:** Accelerating scroll when mouse is dragged above/below terminal during text selection.
- **Interval:** 200ms
- **Runs during idle?** NO. Only active while mouse button is held and cursor is outside terminal rows. Cleared on mouseup (`stopAutoScroll_` at line 372 calls `clearInterval`).
- **Disableable?** Implicitly -- only runs during active mouse drag.

### 2. setTimeout (recursive / self-rescheduling) -- 1 pattern (CURSOR BLINK)

**hterm_terminal.js:4142-4158** -- `onCursorBlink_()`
```js
hterm.Terminal.prototype.onCursorBlink_ = function() {
  if (!this.options_.cursorBlink) {
    delete this.timeouts_.cursorBlink;
    return;
  }
  if (/* cursor not focused, not visible, or blink paused */) {
    this.cursorNode_.setAttribute('visible', 'true');
    this.timeouts_.cursorBlink = setTimeout(this.myOnCursorBlink_,
                                            this.cursorBlinkCycle_[0]);
  } else {
    this.cursorNode_.setAttribute('visible', 'false');
    this.timeouts_.cursorBlink = setTimeout(this.myOnCursorBlink_,
                                            this.cursorBlinkCycle_[1]);
  }
};
```
- **Purpose:** Cursor blink on/off toggle via recursive setTimeout chain.
- **Interval:** `cursorBlinkCycle_` -- default pref is `[1000, 500]` (1000ms visible, 500ms hidden). Initial code value is `[100, 100]` but overwritten by prefs on load.
- **Runs during idle?** YES, continuously, whenever cursor blink is enabled and terminal has focus.
- **Pauses on input:** YES -- `pauseCursorBlink_()` (line 3188) sets `cursorBlinkPause_=true` for 500ms after each keystroke/cursor move. During pause, the blink timer still runs but keeps cursor visible.
- **Stops when unfocused:** YES -- `onCursorBlink_` checks `this.cursorNode_.getAttribute('focus') == 'false'` and keeps cursor visible (but timer still runs).
- **Disableable?** YES.
  - Preference `'cursor-blink'` (default: **false** -- blink is OFF by default).
  - `setCursorBlink(false)` clears the timeout chain immediately.
  - ANSI escape sequences can toggle it at runtime (DECSET/DECRST mode 12).

### 3. setTimeout (one-shot, event-driven) -- NOT idle activity

These are all event-triggered, fire once, and do not re-schedule:

| File:Line | Purpose | Delay | Idle? |
|---|---|---|---|
| hterm_terminal.js:736 | Ready callback dispatch | 0 (microtask) | NO |
| hterm_terminal.js:1778 | Focus change on mousedown | 0 | NO |
| hterm_terminal.js:1927 | Focus on cursor mousedown | 0 | NO |
| hterm_terminal.js:2817 | `scheduleRedraw_()` | 0 | NO |
| hterm_terminal.js:2837 | `scheduleScrollDown_()` | 10ms | NO |
| hterm_terminal.js:2963 | Bell visual flash removal | 200ms | NO |
| hterm_terminal.js:2972 | Bell audio squelch | 500ms | NO |
| hterm_terminal.js:3201 | `pauseCursorBlink_` resume | 500ms | NO (input-triggered) |
| hterm_terminal.js:3312 | `scheduleSyncCursorPosition_` | 0 | NO |
| hterm_terminal.js:3388 | Copy overlay show | 200ms | NO |
| hterm_terminal.js:3847 | Mouse hide debounce | 1000ms | NO |
| hterm_terminal.js:3923 | URL open debounce | 500ms | NO |
| hterm_terminal.js:4090 | Copy to clipboard | 0 | NO |
| hterm_scrollport.js:737 | A11y button display delay | 500ms | NO (init only) |
| hterm_scrollport.js:1102 | `scheduleInvalidate` | 0 | NO |
| hterm_scrollport.js:1284 | `scheduleRedraw` | 0 | NO |
| hterm_scrollport.js:2114 | Paste handler | 0 | NO |
| hterm_notifications.js:107-109 | Overlay auto-hide + fadeout | configurable | NO (triggered) |
| hterm_find_bar.js:323 | Find batch processing | 0 | NO |
| hterm_find_bar.js:394 | Find results sync | 0 | NO |
| hterm_find_bar.js:555 | Find redraw | 0 | NO |
| hterm_find_bar.js:784 | Find notify changes | 0 | NO |
| hterm_pubsub.js:84,105 | Event dispatch chain | 0 | NO |
| hterm_accessibility_reader.js:244 | Live region announce | 50ms | NO (output-triggered) |

All one-shot. None re-schedule themselves. None run during true idle.

### 4. requestAnimationFrame -- 0 occurrences

No RAF usage anywhere in hterm/js/. All rendering is done via setTimeout(0) microtask scheduling and CSS.

### 5. CSS @keyframes -- 1 animation (ANSI BLINK TEXT)

**hterm_terminal.js:1858-1868** (inline CSS injected into shadow DOM)
```css
@keyframes blink {
  from { opacity: 1.0; }
  to { opacity: 0.0; }
}
.blink-node {
  animation-name: blink;
  animation-duration: var(--hterm-blink-node-duration);  /* default: 0.7s */
  animation-iteration-count: infinite;
  animation-timing-function: ease-in-out;
  animation-direction: alternate;
}
```
- **Purpose:** ANSI SGR attribute 5 (blink text). Applied as CSS class `blink-node` to spans with the blink attribute.
- **Interval:** 0.7s per half-cycle (1.4s full cycle), CSS-driven, runs at compositor frame rate (~60fps GPU-side).
- **Runs during idle?** YES, but ONLY if blink-attributed text is currently visible on screen. If no text has the SGR 5 blink attribute, no `.blink-node` elements exist and the animation has zero cost.
- **Disableable?** YES.
  - Preference `'enable-blink'` (default: **true**).
  - `setTextBlink(false)` sets `--hterm-blink-node-duration: 0` which effectively freezes the animation (0s duration = stuck at initial frame).
  - Even when enabled, it's a no-op unless an application actually sends SGR 5 escape sequences.

### 6. Network timers -- 0 occurrences

hterm is a pure terminal emulator library. No WebSocket, fetch, XHR, or polling of any kind.

### 7. MutationObserver / ResizeObserver / IntersectionObserver -- 0 in production

All `MutationObserver` usage (8 instances) is in `hterm_accessibility_reader_tests.js` only. Zero observer usage in production code.

The preference system uses `addObservers` (hterm_terminal.js:295) but this is hterm's own pubsub pattern for preference change callbacks, not a DOM Observer API.

### 8. ANSI Blink (SGR 5) Implementation Detail

hterm implements ANSI blink entirely via CSS animation, NOT via setInterval or setTimeout. The `blink-node` class is applied to text spans. The animation runs in the browser's compositor thread. This is the most efficient possible implementation -- no JS execution during idle, only GPU compositing of opacity changes on affected elements.

---

## Idle Profile

**When terminal is completely idle (no typing, no output, no blink text):**

| Cursor blink OFF (default) | Cursor blink ON |
|---|---|
| ZERO JS timers running | 1 setTimeout chain: ~0.67 fires/sec |
| ZERO CSS animations | ZERO CSS animations (unless SGR 5 text present) |
| ZERO observers | ZERO observers |
| **CALM** | **MINOR_ACTIVITY** |

**Worst case (cursor blink ON + SGR 5 blink text on screen):**
- 1 JS setTimeout chain (~0.67 fires/sec, alternating DOM attribute)
- 1 CSS animation per blink-text span (GPU compositor, no JS)
- Rating: **MINOR_ACTIVITY**

## Disabling All Idle Activity

```js
term.prefs_.set('cursor-blink', false);   // stops setTimeout chain
term.prefs_.set('enable-blink', false);   // freezes CSS blink animation
```

Both take effect immediately. After this, hterm has ZERO running timers or animations during idle.
