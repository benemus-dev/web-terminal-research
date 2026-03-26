# Idle Activity Audit Summary

What runs when the screen is sitting there doing nothing — no user input, no new data.

## Idle Score Table

| Project | Score | Cursor Blink Default | Pixel Repaints at Idle | Network at Idle | Worst Offender |
|---------|-------|---------------------|----------------------|-----------------|----------------|
| **claude-code-webui** | CALM | n/a (chat UI) | 0 | 0 | Nothing. Truly zero idle activity |
| **hterm** | CALM | OFF | 0 | 0 | Nothing at defaults. Cursor blink is opt-in setTimeout chain |
| **DomTerm** | MINOR | ON (stops after 30s) | CSS cursor blink x20 cycles then stops | 0 | Cursor blink, but self-terminating |
| **CUI** | MINOR | n/a (chat UI) | 0 | 30s SSE heartbeat (server-only) | Shimmer on "ongoing" tasks |
| **ttyd** | MINOR | OFF | 0 | 5s WebSocket ping | WebSocket keepalive only |
| **WeTTY** | MINOR | OFF | 0 | 3s Socket.IO ping | Socket.IO ping (8x more aggressive than default) |
| **xterm.js** | MINOR | OFF | 0 at defaults | 0 | Cursor blink auto-stops after 5min if enabled |
| **GoTTY** | MINOR | ON (stops on blur) | 600ms cursor blink (focused only) | 30s WebSocket ping | Cursor blink, but stops on blur |
| **claude-code-remote** | MINOR | ON (via ttyd flag) | 600ms cursor blink (ttyd) | 5s ttyd WebSocket ping | Inherited from ttyd; wrapper itself is clean |
| **claudecodeui** | MODERATE | ON | 600ms WebGL cursor redraw + CSS pulse | 5min version check | 3.2 timer events/sec: sidebar poll + cursor + CSS pulse |
| **shellinabox** | BUSY | ON (never stops) | 500ms setInterval FOREVER | Continuous AJAX long-poll | 500ms cursor blink + key check runs forever, even after session close |

## Detailed Breakdown

### What keeps the screen busy when idle

| Activity | claude-code-webui | hterm | DomTerm | CUI | ttyd | WeTTY | xterm.js | GoTTY | remote | claudecodeui | shellinabox |
|----------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| setInterval (any) | 0 | 0 | 0 | 2 (server) | 0 | 0 | 3* | 1 | 0 | 3 | 1 |
| Recursive setTimeout | 0 | 1* | 1* | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Continuous RAF loop | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| One-shot RAF | 0 | 0 | 1 | 0 | 0 | 0 | 5 | 0 | 0 | 0 | 0 |
| CSS infinite animation | 0 | 1* | 0* | 0* | 0 | 0 | 3* | 0 | 0 | 3 | 0 |
| Network keepalive | 0 | 0 | 0 | 30s | 5s | 3s | 0 | 30s | 5s | 5min | continuous |
| Observers | 0 | 0 | 4 | 2 | 0 | 1 | 2 | 0 | 0 | 3 | 0 |

`*` = exists but disabled by default, gated by state, or self-terminating

### Key Design Patterns (Best to Worst)

**Tier 1: True Zero Idle (CALM)**
- claude-code-webui: No terminal emulator = no cursor blink, no canvas. React only renders on state change.
- hterm: Cursor blink OFF by default. SGR blink via CSS (compositor, not JS). Zero RAF.

**Tier 2: Self-Terminating (MINOR)**
- DomTerm: Cursor blink stops after 20 cycles (30 seconds). Best cursor blink design of any terminal.
- xterm.js: Cursor blink auto-stops after 5 minutes (`CURSOR_BLINK_IDLE_TIMEOUT = 300000ms`). All RAF one-shot.
- GoTTY: Cursor blink stops on blur (unfocused tabs are silent).

**Tier 3: Always-On but Minimal (MINOR)**
- ttyd: Only 5s WebSocket ping. Cursor blink OFF by default.
- WeTTY: Only 3s Socket.IO ping. Cursor blink OFF by default.
- CUI: Only 30s SSE heartbeat (server-side, no DOM).

**Tier 4: Multiple Idle Timers (MODERATE)**
- claudecodeui: ~3.2 events/sec. Sidebar localStorage poll (1Hz, redundant), cursor blink (1.67Hz), CSS pulse on sidebar items. Respects `prefers-reduced-motion`.

**Tier 5: Never Stops (BUSY)**
- shellinabox: 500ms setInterval runs forever. Cursor blink + `checkComposedKeys()` DOM read every 500ms, even after session close, even when blurred. AJAX long-poll adds continuous HTTP POST cycle.

## Recommendations for a Strain-Free Terminal

Based on this audit, a truly calm terminal display should:

1. **Cursor blink OFF by default** (hterm, ttyd, WeTTY, xterm.js all do this right)
2. **If cursor blink is enabled, auto-stop it** -- DomTerm's 30-second limit or xterm.js's 5-minute timeout
3. **CSS `@keyframes` for cursor blink, not JS setInterval** -- compositor-driven, zero main thread cost (DomTerm, xterm.js DOM renderer do this)
4. **All RAF must be one-shot** (demand-driven, not continuous loops). xterm.js gets this right.
5. **No polling** -- use events (ResizeObserver, IntersectionObserver, `storage` event) instead of setInterval
6. **Respect `prefers-reduced-motion`** -- claudecodeui does this; most others don't
7. **Clear all timers on blur/hidden** -- GoTTY does this for cursor; shellinabox fails completely
8. **Network keepalive is acceptable** (5-30s ping has zero visual impact) but should not trigger DOM work
