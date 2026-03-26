# xterm.js Idle Activity Audit

## Summary
- Total setInterval calls: 3
- Total recursive setTimeout: 1
- Total RAF calls: 5 (continuous: 0, one-shot: 5)
- Total CSS infinite animations: 3
- Total Observers: 2 (1 IntersectionObserver, 1 ResizeObserver)

## Idle Activity Score: MINOR_ACTIVITY
Cursor blink is the only continuous idle activity (either CSS animation in DomRenderer or setInterval in WebGL addon). It auto-stops after 5 minutes of idle. Text blink (SGR attribute 5) also runs via setInterval but only when blinking text is visible in the viewport. Everything else is on-demand.

---

## Detailed Findings

### setInterval

| # | File:Line | Purpose | Interval (ms) | Cleared on idle? | Disableable? |
|---|-----------|---------|---------------|-----------------|-------------|
| 1 | `src/browser/renderer/shared/TextBlinkStateManager.ts:74` | Text blink (SGR attribute 5). Toggles `_blinkOn` flag and calls `_renderCallback()` each tick. | Configurable via `blinkIntervalDuration` option (default: **0 = disabled**) | YES -- cleared when no blinking text in viewport (`setNeedsBlinkInViewport(false)`) AND when viewport not visible (`setViewportVisible(false)`) | YES -- set `blinkIntervalDuration: 0` to disable entirely |
| 2 | `addons/addon-webgl/src/CursorBlinkStateManager.ts:114` | WebGL cursor blink. Toggles cursor visibility and triggers RAF render callback. | 600ms (hardcoded `BLINK_INTERVAL`) | YES -- stops after 5 min idle (`CURSOR_BLINK_IDLE_TIMEOUT = 300000`). Also paused on blur. | YES -- set `cursorBlink: false` |
| 3 | `src/browser/services/SelectionService.ts:503` | Drag-scroll during mouse selection. Scrolls viewport while mouse is held outside terminal bounds. | 50ms (`DRAG_SCROLL_INTERVAL`) | YES -- cleared on mouseup (`_mouseUpListener` calls `clearInterval`) | N/A -- only active during mouse drag, never idle |

### IntervalTimer utility (generic wrapper)

| # | File:Line | Purpose | Notes |
|---|-----------|---------|-------|
| 4 | `src/common/Async.ts:90` | `IntervalTimer.cancelAndSet()` -- generic wrapper around `setInterval`. Used by `WindowIntervalTimer` in `src/browser/Dom.ts:167`. | Not a direct source of idle activity; used on-demand by other code. No persistent callers found during idle. |

### Recursive setTimeout

| # | File:Line | Purpose | Effective frequency | Cleared on idle? | Disableable? |
|---|-----------|---------|-------------------|-----------------|-------------|
| 1 | `addons/addon-webgl/src/CursorBlinkStateManager.ts:94-132` | Cursor blink restart: `_restartInterval()` uses a `setTimeout` that, on completion, calls `_restartInterval()` again if `_animationTimeRestarted` was set during the timeout. This is a re-scheduling pattern but only triggers during active typing/interaction, not during true idle. After restart, it falls into the steady-state `setInterval` (finding #2 above). | ~600ms during interaction bursts | YES -- same idle timeout as setInterval (5 min) | YES -- `cursorBlink: false` |

### requestAnimationFrame

All RAF usage in xterm.js is **one-shot** (fire once, complete). None create continuous RAF loops.

| # | File:Line | Purpose | Continuous? | Dirty check? | Runs when hidden? |
|---|-----------|---------|------------|-------------|------------------|
| 1 | `src/browser/RenderDebouncer.ts:35,53` | Debounces terminal row rendering. Schedules a single RAF when rows need refresh, then clears `_animationFrame`. | No -- one-shot | Yes -- only fires when `_rowStart`/`_rowEnd` are set (dirty range) | No -- `RenderService` pauses when `IntersectionObserver` reports not visible |
| 2 | `src/browser/Dom.ts:161` | `scheduleAtNextAnimationFrame()` -- queues work items for next frame. Single RAF per frame, runs queued items, stops. | No -- one-shot | Yes -- only runs if items queued | No -- callers are event-driven |
| 3 | `src/browser/decorations/OverviewRulerRenderer.ts:208` | Renders overview ruler decorations. Single RAF with guard (`if _animationFrame !== undefined return`). | No -- one-shot | Yes -- guard prevents re-entry | N/A |
| 4 | `src/browser/services/SelectionService.ts:282` | Refreshes selection rendering. Single RAF with guard. | No -- one-shot | Yes -- guard | N/A |
| 5 | `addons/addon-webgl/src/CursorBlinkStateManager.ts:76,108,127` | Renders cursor after blink state change. Single RAF per blink tick (driven by setInterval #2). | No -- one-shot (but called periodically by setInterval) | N/A -- always renders when called | Paused on blur |

### CSS Animations

All CSS animations are injected dynamically in `src/browser/renderer/dom/DomRenderer.ts` (lines 217-257). These are the **DomRenderer cursor blink** animations.

| # | File:Line | Keyframes name | Duration | Iteration | Runs during idle? | Disableable? |
|---|-----------|---------------|----------|-----------|------------------|-------------|
| 1 | `src/browser/renderer/dom/DomRenderer.ts:221` | `blink_underline_{id}` | 1s step-end | **infinite** | YES -- but stopped after 5 min idle by adding class `xterm-cursor-blink-idle` which sets `animation: none !important` (line 255-257). Also only active when terminal is focused and cursor style is underline+blink. | YES -- `cursorBlink: false` |
| 2 | `src/browser/renderer/dom/DomRenderer.ts:227` | `blink_bar_{id}` | 1s step-end | **infinite** | Same as above, for bar cursor | YES -- `cursorBlink: false` |
| 3 | `src/browser/renderer/dom/DomRenderer.ts:233` | `blink_block_{id}` | 1s step-end | **infinite** | Same as above, for block cursor | YES -- `cursorBlink: false` |

The idle timeout is implemented in `DomRenderer.ts:651-706` (`CursorBlinkStateManager` class) which adds the CSS class `xterm-cursor-blink-idle` after `CURSOR_BLINK_IDLE_TIMEOUT` (5 minutes, from `src/browser/renderer/shared/Constants.ts:12`).

**No CSS animations exist in `css/xterm.css`** -- that file contains no `@keyframes` rules.

### Scrollbar fade animation (CSS transition, not animation)

| # | File:Line | What | Notes |
|---|-----------|------|-------|
| 1 | `css/xterm.css:258-260` | `.xterm-invisible.xterm-fade { transition: opacity 800ms linear; }` | This is a CSS **transition**, not an animation. It only fires once when the scrollbar hides. Does not loop. |

### Observers

| # | File:Line | Type | Purpose | Triggers redraw? | Runs during idle? |
|---|-----------|------|---------|-----------------|------------------|
| 1 | `src/browser/services/RenderService.ts:129` | `IntersectionObserver` | Pauses/resumes rendering when terminal scrolls in/out of viewport. Threshold 0 (any visibility change). | Yes -- triggers full refresh on becoming visible. But only fires on visibility change, not continuously. | No -- only fires on visibility state transitions |
| 2 | `addons/addon-webgl/src/DevicePixelObserver.ts:13` | `ResizeObserver` | Monitors canvas element for device-pixel-accurate sizing (prevents blurry rendering). Observes `device-pixel-content-box`. | Yes -- calls callback with new dimensions, which triggers canvas resize + re-render. | No -- only fires when element actually resizes |

### Network Timers

**None found.** The `addon-attach` addon connects to a WebSocket but implements no heartbeat, keep-alive, or ping/pong timers. It simply listens for `message`, `open`, and `close` events.

### Other Continuous Activity

| # | File:Line | What | Notes |
|---|-----------|------|-------|
| 1 | `src/common/TaskQueue.ts:117` | `PriorityTaskQueue` uses `setTimeout` with 16ms deadline to process batched tasks | Not idle activity -- only runs when tasks are enqueued. Queue clears itself when empty. |
| 2 | `src/browser/TimeBasedDebouncer.ts:59` | Trailing refresh `setTimeout` for accessibility live region updates | Not idle -- only fires when terminal content changes, with 1s debounce. One-shot. |
| 3 | `addons/addon-webgl/src/WebglRenderer.ts:126` | `setTimeout` for WebGL context loss recovery (3000ms) | Only fires on context loss event. Not idle activity. |
| 4 | `src/browser/services/RenderService.ts:354` | `SynchronizedOutputHandler` timeout (1000ms) | Safety timeout for DEC mode 2026. Only active during synchronized output sequences. One-shot. |

---

## Analysis: What Happens During True Idle

When the terminal has no input, no new data, and is just sitting there:

### If `cursorBlink: true` (non-default):
1. **DomRenderer path**: One CSS animation runs continuously at 1s cycle (step-end, so 2 repaints/sec). After 5 minutes of no keyboard input, the animation is disabled via CSS class override.
2. **WebGL addon path**: One `setInterval` at 600ms toggles cursor visibility and triggers one-shot RAF. After 5 minutes of no keyboard input, the interval is cleared.

### If `cursorBlink: false` (default):
**Nothing runs.** Zero timers, zero intervals, zero RAF loops. The terminal is completely quiescent.

### If blinking text (SGR 5) is present in viewport:
- `TextBlinkStateManager` setInterval runs at `blinkIntervalDuration` ms (default 0 = disabled).
- If enabled and blinking text scrolls out of viewport, interval is cleared.
- If viewport becomes hidden (IntersectionObserver), interval is cleared.

### Display-strain relevance:
The cursor blink is the primary idle activity source. At default settings (`cursorBlink: false`), xterm.js produces zero idle GPU/CPU activity. The 5-minute idle timeout for cursor blink is a good practice that eliminates long-term idle overhead even when blink is enabled.
