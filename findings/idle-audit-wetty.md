# WeTTY Idle Activity Audit

**Version:** 2.7.0
**Source:** `/c/dev/web-terminal-research/wetty/src/`
**Date:** 2026-03-26
**Method:** Static code analysis only (no runtime)

## Summary

| Category | Count | Idle impact |
|---|---|---|
| setInterval | 0 | None |
| setTimeout (recursive) | 0 | None |
| requestAnimationFrame | 0 | None |
| CSS @keyframes infinite | 0 | None |
| Network timers (Socket.IO ping) | 1 | YES - runs always |
| MutationObserver | 1 (FontAwesome dom.watch) | YES - runs always |
| ResizeObserver | 0 | None |
| IntersectionObserver | 0 | None |
| xterm.js inherited timers | 1 (cursor blink, OFF by default) | Configurable |

**Idle score: MINOR_ACTIVITY**

WeTTY's own code is remarkably clean. The two always-on idle sources are both from dependencies: Socket.IO ping/pong and FontAwesome's DOM MutationObserver. The xterm.js cursor blink is off by default.

---

## Detailed Findings

### 1. setInterval

**None found.** Zero occurrences in `src/`.

---

### 2. setTimeout (recursive / self-rescheduling)

**None found.** There is one `setTimeout` in the codebase but it is NOT recursive:

| Location | Purpose | Recursive? |
|---|---|---|
| `src/server/flowcontrol.ts:26` | tinybuffer: coalesces PTY output into batched WebSocket messages. Fires once per data burst, then clears. | NO - one-shot, only created when data arrives, cleared on send |

The tinybuffer timeout is purely data-driven. When no PTY output arrives, no timer is scheduled. Zero idle activity.

---

### 3. requestAnimationFrame

**None found.** Zero occurrences in `src/`.

Note: xterm.js v5.2 with the **canvas renderer** uses rAF internally for rendering. However, WeTTY's default config sets `rendererType: 'canvas'` and xterm.js canvas renderer only calls rAF when there are pending writes/redraws -- it does NOT run a continuous rAF loop when idle. See xterm.js internals section below.

---

### 4. CSS animations (@keyframes infinite)

**None found.** Zero `@keyframes`, zero `animation`, zero `animation-iteration-count` in any SCSS or CSS file under `src/`.

Files checked:
- `src/assets/scss/styles.scss` -- body/html reset only
- `src/assets/scss/overlay.scss` -- static positioning
- `src/assets/scss/terminal.scss` -- static flex layout
- `src/assets/scss/options.scss` -- static grid layout
- `src/assets/scss/variables.scss` -- color vars only
- `src/assets/xterm_config/style.css` -- static form styles

---

### 5. Network Timers (Socket.IO ping/pong)

**ONE always-on timer found.**

| Location | Setting | Value |
|---|---|---|
| `src/server/socketServer/socket.ts:40` | `pingInterval` | **3000 ms** |
| `src/server/socketServer/socket.ts:41` | `pingTimeout` | **7000 ms** |

**How it works:** The Socket.IO server sends a ping frame every 3 seconds. The client responds with pong. If no pong arrives within 7 seconds, the connection is considered dead.

- **Runs during idle?** YES -- unconditionally, as long as the WebSocket connection is open.
- **Disableable?** Yes, by changing server config. Setting `pingInterval` higher (e.g., 25000 which is Socket.IO's default) reduces idle traffic.
- **Display impact:** The ping/pong frames are tiny WebSocket control frames. They cause no DOM changes and no repaints. Network-only activity.
- **Client-side config:** The client (`src/client/wetty/socket.ts`) uses default Socket.IO client settings (no custom ping config). The server dictates the interval.

**Note:** Socket.IO's default `pingInterval` is 25000ms. WeTTY sets it to 3000ms (8x more frequent than default). This is aggressive for a terminal that may sit idle.

---

### 6. MutationObserver / ResizeObserver / IntersectionObserver

**No direct usage in WeTTY source code.**

However, there is one **indirect** MutationObserver via a dependency:

| Location | Source | Purpose |
|---|---|---|
| `src/client/wetty.ts:19` | `dom.watch()` (FontAwesome) | Watches the entire DOM for `<i>` tags with FA classes, auto-replaces them with SVGs |

**FontAwesome `dom.watch()` details:**
- Installs a `MutationObserver` on `document.body` with `{ childList: true, subtree: true }`.
- Fires on ANY DOM mutation anywhere in the document.
- During idle, the terminal has no DOM mutations, so the observer callback does not fire.
- **Runs during idle?** The observer is registered but does not fire unless DOM changes occur. Effectively dormant during idle.
- **Disableable?** Yes. Replace `dom.watch()` with `dom.i2svg()` (one-shot conversion) since WeTTY only uses two icons (cogs, keyboard) that are present at load time.

---

### 7. xterm.js Inherited Timers (from node_modules)

WeTTY uses `@xterm/xterm` v5.2.0 with three addons: `addon-fit`, `addon-image`, `addon-web-links`.

Default config from `src/assets/xterm_config/xterm_defaults.js`:

| Setting | Value | Idle impact |
|---|---|---|
| `cursorBlink` | **false** | No timer. If set to `true`, xterm.js runs a ~600ms setInterval to toggle cursor visibility. |
| `rendererType` | `'canvas'` | Canvas renderer uses rAF but only on dirty (data-driven). No continuous loop when idle. |
| `bellStyle` | `'none'` | No audio timers. |
| `scrollback` | `1000` | No timer impact. |

**xterm.js idle behavior with these defaults:**
- **Cursor blink OFF:** No blink timer runs. The cursor is a static block. Zero repaints from cursor.
- **Canvas renderer:** Maintains a rAF-based render loop, but xterm.js v5+ is smart -- it only schedules rAF when `_needsRefresh` is true. When idle (no data, no user input), no rAF runs.
- **Selection / link hover:** These are event-driven only.
- **FitAddon:** Only runs on explicit `fit()` calls (window resize). No polling.
- **ImageAddon:** Data-driven only, no timers.
- **WebLinksAddon:** Hover-driven only, no timers.

**Risk if user changes config:** If a user enables `cursorBlink: true` through the settings UI (exposed at `src/assets/xterm_config/xterm_general_options.js:87`), a ~600ms blink timer will start and cause continuous small repaints. This is the single most likely source of idle display activity in a WeTTY deployment.

---

## Additional Observations

### `lodash.debounce` usage
- `src/client/wetty/term.ts:100-102` -- A debounced backspace simulation (100ms) is created on every CTRL+key press. This is user-input-driven only, never fires during idle.

### window.onresize handler
- `src/client/wetty/term.ts:223` -- Calls `term.resizeTerm()` which triggers `term.refresh(0, rows-1)`. Event-driven only, zero idle cost.

### Server-side metrics (prom-client)
- `src/server/metrics.ts` and `src/server/socketServer/metrics.ts` -- Prometheus counters/histograms. No periodic collection timers in WeTTY's code. prom-client's `register.metrics()` is on-demand (HTTP scrape). However, `prom-client` does install a default `collectDefaultMetrics()` interval internally if enabled -- WeTTY does NOT call `collectDefaultMetrics()`, so no server-side polling timer exists.

### gc-stats dependency
- Listed in `package.json` but hooks into Node.js GC events (callback-driven), not a timer.

---

## Idle Activity Summary

When WeTTY is idle (connected, cursor visible, no typing, no output):

| Source | Type | Interval | Pixels affected | Disableable? |
|---|---|---|---|---|
| Socket.IO ping/pong | Network | 3000 ms | 0 (no DOM) | Yes (increase pingInterval) |
| FontAwesome MutationObserver | DOM observer | on-mutation (dormant when idle) | 0 | Yes (use dom.i2svg() instead) |
| xterm.js cursor blink | Timer + repaint | OFF by default | 0 | Already off |

**Total idle repaints: 0**
**Total idle network: 1 ping/pong exchange every 3 seconds**

## Recommendations for Minimal Idle Activity

1. **Increase Socket.IO pingInterval** from 3000ms to 25000ms (Socket.IO default) or higher for LAN use where connection drops are rare. File: `src/server/socketServer/socket.ts:40`.

2. **Replace `dom.watch()` with `dom.i2svg()`** in `src/client/wetty.ts:19`. The two FA icons (cogs, keyboard) are in the initial HTML and don't need continuous DOM watching.

3. **Keep `cursorBlink: false`** as default (already the case). Document that enabling it adds a continuous repaint timer.
