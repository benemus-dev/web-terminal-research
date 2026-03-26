# Idle Activity Audit: claude-code-remote

Audited: 2026-03-26
Source: `/c/dev/web-terminal-research/claude-code-remote/scripts/`
Files: `voice-wrapper.py`, `start-remote-cli.sh`, `stop-remote-cli.sh`, `tmux-attach.sh`, `remote-cli.plist`

## Summary

| Category | Count | Idle impact |
|----------|-------|-------------|
| setInterval | 0 | none |
| setTimeout (recursive) | 0 | none |
| requestAnimationFrame | 0 | none |
| CSS @keyframes | 0 | none |
| Network timers (wrapper) | 0 | none |
| visibilitychange handler | 1 | on-wake only |
| Bash sleep loops | 1 | watchdog, sleeps until crash |
| ttyd inherited timers | 3 | continuous |

**Total idle sources: 5** (1 wrapper-level, 1 bash-level, 3 ttyd-inherited)
**Idle score: 3/10** -- Low idle overhead from the wrapper itself. The dominant idle cost comes from ttyd/xterm.js internals (cursor blink, WebSocket ping) which are inherited, not authored.

---

## Detailed Findings

### 1. setInterval

**None found.** The inline JS in `voice-wrapper.py` contains no `setInterval` calls.

### 2. setTimeout (recursive / self-rescheduling)

**None that run during idle.** Two `setTimeout` calls exist but are one-shot and user-triggered only:

| # | File:Line | Code | Interval | Runs during idle? | Disableable? |
|---|-----------|------|----------|--------------------|--------------|
| 1 | `voice-wrapper.py:291` | `setTimeout(() => sendText('claude'), 1500)` | 1500ms | No -- fires once after "New" button press | N/A |
| 2 | `voice-wrapper.py:297` | `setTimeout(() => sendText('claude --resume'), 1500)` | 1500ms | No -- fires once after "Resume" button press | N/A |
| 3 | `voice-wrapper.py:379` | `setTimeout(() => { input.placeholder = origPlaceholder; }, 3000)` | 3000ms | No -- fires once after upload failure | N/A |

None of these re-schedule themselves. Zero idle impact.

### 3. requestAnimationFrame

**None found.** No `requestAnimationFrame` in the inline JS.

### 4. CSS @keyframes

**None found.** The inline CSS in `voice-wrapper.py` contains no `@keyframes` rules, no `animation` properties, no `transition` properties on continuously-visible elements. The only pseudo-class with visual effect is `:active` on buttons (fires only on touch).

### 5. Network timers (wrapper page)

**None found.** The FastAPI wrapper (`voice-wrapper.py`) has no polling, no keepalive timers, no periodic fetch calls. All network requests are user-initiated:
- `/send` -- on button press or Enter key
- `/key` -- on quick-key button press
- `/copy` -- on Copy button press
- `/upload` -- on file selection

Uvicorn's HTTP server uses standard TCP keepalive (OS-level, not application timers). No WebSocket connection exists between the wrapper page and FastAPI -- all communication is plain HTTP POST/GET.

### 6. visibilitychange handler

| # | File:Line | Code | Interval | Runs during idle? | Disableable? |
|---|-----------|------|----------|--------------------|--------------|
| 1 | `voice-wrapper.py:390-393` | `document.addEventListener('visibilitychange', ...)` reloads iframe via `terminal.src = terminal.src` | Event-driven (on tab focus) | **No** -- only fires on visibility transition from hidden to visible | Yes -- remove the event listener |

Purpose: Auto-reconnect ttyd iframe when phone wakes from sleep.
Impact: Causes a full iframe reload (new WebSocket connection to ttyd) each time the tab becomes visible. Does NOT poll periodically. However, if a user rapidly switches tabs, this triggers repeated iframe reloads. Not an idle concern -- it is a wake-from-idle concern.

### 7. Bash script timers

| # | File:Line | Code | Interval | Runs during idle? | Disableable? |
|---|-----------|------|----------|--------------------|--------------|
| 1 | `start-remote-cli.sh:70-91` | Watchdog loop: `wait $TTYD_PID` then `sleep 5` then restart ttyd | 5000ms sleep, but only after ttyd crashes | **No** -- `wait` blocks until ttyd exits; sleep only runs between crash and restart | Yes -- remove the while loop |
| 2 | `start-remote-cli.sh:21` | `sleep 1` after `pkill -f ttyd` | 1000ms, once at startup | No -- runs once during startup, not a loop | N/A |

The watchdog (lines 70-91) is NOT a polling loop. It calls `wait $TTYD_PID` which blocks the shell with zero CPU until the ttyd process exits. The `sleep 5` only executes between a crash and the restart attempt. During normal idle operation, this loop consumes zero resources.

### 8. ttyd inherited idle timers

ttyd is launched at `start-remote-cli.sh:31-42` with explicit xterm.js configuration. The following idle timers are inherited from ttyd and xterm.js:

| # | Source | Timer | Interval | Runs during idle? | Disableable? |
|---|--------|-------|----------|--------------------|--------------|
| 1 | ttyd WebSocket | Ping/pong keepalive | ~5000ms (5s, ttyd default) | **YES** -- continuous binary WebSocket ping frames from server to client | Yes -- ttyd `--ping-interval 0` flag (but connection may drop) |
| 2 | xterm.js | Cursor blink timer | ~600ms (xterm.js default) | **YES** -- continuous DOM toggle on cursor element | Yes -- configured ON at line 37: `-t cursorBlink=true`. Change to `cursorBlink=false` |
| 3 | xterm.js | Selection/link hover debounce | varies | No -- event-driven only | N/A |

**Note on cursor blink:** `start-remote-cli.sh:37` explicitly enables cursor blink with `-t cursorBlink=true`. This creates a ~600ms setInterval inside the xterm.js terminal that runs continuously while the page is open, toggling CSS opacity on the cursor element. This is the single most significant idle timer in the entire stack, because it triggers a DOM repaint every 600ms even when nothing else is happening.

**Note on WebSocket ping:** ttyd sends a binary ping frame every ~5 seconds over the WebSocket connection to detect disconnected clients. The browser responds with a pong. This is network I/O during idle but does not cause any DOM/rendering activity.

---

## Idle Reduction Recommendations

| Priority | Change | Eliminates |
|----------|--------|------------|
| 1 | Change `-t cursorBlink=true` to `-t cursorBlink=false` in `start-remote-cli.sh` | ~1.67 repaints/sec continuous |
| 2 | Add `--ping-interval 0` to ttyd launch (or increase to 30s) | 5s WebSocket ping/pong (but risks stale connections) |
| 3 | (Optional) Add `cursorStyle=underline` or `cursorStyle=bar` if blink is disabled | Reduces cursor visual weight without animation |

The wrapper page itself (voice-wrapper.py) is remarkably clean -- zero polling, zero animation, zero recurring timers. All idle cost is from the ttyd/xterm.js iframe.
