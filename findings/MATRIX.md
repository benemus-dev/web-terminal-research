# Web Terminal Rendering Matrix

## Comparison Table

| Project | Version | Renderer | Update Mechanism | Transport | PTY | VT Parser | Strain | Notes |
|---------|---------|----------|-----------------|-----------|-----|-----------|--------|-------|
| **xterm.js DOM** | 6.0.0 | DOM_ONLY | RAF_BATCHED (RenderDebouncer) | n/a (library) | n/a | Client (built-in) | Low | Default in v6. `<div>`/`<span>` tree, CSS cursor blink |
| **xterm.js WebGL** | 6.0.0 | WEBGL (WebGL2) | RAF_BATCHED (RenderDebouncer) | n/a (library) | n/a | Client (built-in) | Medium | Addon. Texture atlas via Canvas 2D fillText, JS cursor blink |
| **ttyd** | 1.7.7 | XTERM_WEBGL (default) | PUSH_THEN_RAF + flow control | WebSocket (libwebsockets, binary) | C forkpty / ConPTY | Client (xterm.js 5.5) | Low | Fallback: WebGL -> Canvas -> DOM. C99 backend |
| **WeTTY** | 2.7.0 | XTERM_DEFAULT (DOM) | Delegated to xterm.js RAF | Socket.IO v4 (WebSocket) | node-pty (SSH/login) | Client (xterm.js 5.2) | Medium | DOM renderer despite `rendererType:'canvas'` config (no addon loaded) |
| **DomTerm** | 3.2.0 | DOM_ONLY | RAF + 200ms defer | WebSocket (libwebsockets, binary) | C openpty+fork | Client (domterm-parser.js) | Low | `<div.domterm-pre>`/`<span>`, CSS cursor blink, native scrollbar |
| **hterm** | 1.92.1 | DOM_ONLY | setTimeout (coalesced) | n/a (library) | n/a | Client (hterm_vt.js) | Low | `<x-row>`/`<span>`, virtual viewport, canvas for measurement only |
| **claude-code-webui** | 0.1.56 | DOM_ONLY (React) | REACT_RECONCILER + NDJSON stream | HTTP fetch streaming (NDJSON) | n/a (SDK) | None (chat UI) | Low | React 19, Hono backend, Claude Code SDK `query()` |
| **claudecodeui** | 1.26.3 | HYBRID | PUSH_THEN_RAF (terminal) / REACT_RECONCILER (chat) | WebSocket (ws) | node-pty (shell tab) | Client (xterm.js 5.5, shell tab) | Low-Medium | Dual mode: xterm.js WebGL terminal + React chat UI |
| **CUI** | 0.6.3 | DOM_ONLY (React) | REACT_RECONCILER + SSE stream | SSE via fetch ReadableStream | n/a (CLI spawn) | None (chat UI) | Low | React 18, spawns `claude` CLI, parses JSONL |
| **claude-code-remote** | n/a | XTERM_CANVAS (via ttyd) | ttyd internal (PUSH_THEN_RAF) | REST (FastAPI) + ttyd WebSocket | tmux + ttyd | Client (xterm.js via ttyd) | Medium | Bash/Python scripts, ttyd in iframe, tmux send-keys |
| **GoTTY** | 1.0.1 | CANVAS_2D (xterm.js v3/v4) | PUSH_THEN_RAF | WebSocket (gorilla/websocket) | Go kr/pty | Client (xterm.js) | Medium | Archived. Base64-encoded output, pre-modules Go |
| **shellinabox** | 2.20 | DOM_ONLY | PUSH_IMMEDIATE (sync on poll) | AJAX long-poll (XMLHttpRequest) | C openpty | Client (vt100.jspp) | Low | Unmaintained. `<pre>`/`<span>`, custom VT100 parser |

## Renderer Classification Key

| Code | Meaning |
|------|---------|
| DOM_ONLY | All text via `<div>`, `<span>`, `<pre>` text nodes. Browser-native text pipeline |
| CANVAS_2D | Uses `getContext('2d')`, `fillText()` for character rendering |
| WEBGL | Uses `getContext('webgl')` or `'webgl2'` with texture atlas |
| XTERM_DEFAULT | Defers to xterm.js default renderer (DOM in v5+/v6) |
| XTERM_WEBGL | Explicitly loads `@xterm/addon-webgl` |
| XTERM_CANVAS | Explicitly loads `@xterm/addon-canvas` or uses pre-v5 xterm.js canvas default |
| HYBRID | Mix of renderers for different UI modes |
| REACT_RECONCILER | React state update -> virtual DOM diff -> DOM patch (chat UIs) |

## Update Mechanism Key

| Code | Meaning |
|------|---------|
| RAF_BATCHED | Uses requestAnimationFrame, queues dirty rows, renders once per frame |
| PUSH_THEN_RAF | Data pushed immediately into write buffer on message, flushed via RAF |
| PUSH_IMMEDIATE | Renders synchronously on each incoming message/poll response |
| REACT_RECONCILER | React/framework state update batching -> DOM patch |
| setTimeout | Coalesced via setTimeout (not RAF-aligned, but browser compositor handles vsync) |

## xterm.js Version Boundary (Critical for Strain)

| xterm.js Version | Default Renderer | Canvas Renderer | WebGL Renderer |
|-------------------|-----------------|-----------------|----------------|
| v3/v4 | CANVAS_2D (built-in) | Built-in default | Addon (optional) |
| v5 | DOM (built-in) | Addon (`@xterm/addon-canvas`) | Addon (`@xterm/addon-webgl`) |
| v6 | DOM (built-in) | **Removed** | Addon (`@xterm/addon-webgl`) |

**This is the single most important version boundary for the strain question.** Projects using xterm.js v5+ without explicitly loading a canvas/WebGL addon get DOM rendering (low strain). Projects using v3/v4 or loading the WebGL addon get canvas-based rendering (medium strain).

| Project | xterm.js Version | Actual Renderer |
|---------|-----------------|-----------------|
| ttyd | ^5.5 | WebGL (addon loaded) |
| WeTTY | ^5.2 | DOM (no addon loaded) |
| claudecodeui | 5.5 | WebGL (addon loaded) |
| claude-code-remote | via ttyd | Canvas/WebGL (ttyd default) |
| GoTTY | v3/v4 era | Canvas (built-in default) |

---

## Strain Analysis

### Strain Risk Scale

- **Low strain risk** - Pure DOM, browser-native text pipeline, subpixel antialiasing, no canvas repaint loops
- **Medium strain risk** - Canvas 2D (grayscale AA, no subpixel) or WebGL with texture atlas (custom font rasterization)
- **High strain risk** - Aggressive redraws, no dirty tracking, custom font rasterization without caching

### Low Strain Projects

| Project | Cursor Blink | Scrollback | Colors | Notes |
|---------|-------------|------------|--------|-------|
| **xterm.js DOM** | CSS `@keyframes step-end` (good) | Custom scroll element with `overflow-y: scroll` (acceptable) | CSS classes `xterm-fg-N` for palette, inline `style` for RGB (good) | 5min idle timeout pauses blink |
| **DomTerm** | CSS `@keyframes steps(1)` 1.5s cycle (good) | CSS `overflow-y: scroll` on container (good) | CSS custom properties + inline styles on spans (good) | No virtual scrolling = DOM growth risk |
| **hterm** | JS `setTimeout` toggle (acceptable, configurable) | Virtual viewport: only visible rows in DOM (excellent) | Inline styles on `<span>` (good) | Default 100ms blink cycle is fast; disableable |
| **claude-code-webui** | No cursor (chat UI) | React list with `scrollIntoView` (good) | TailwindCSS classes (good) | Streaming append-only, minimal DOM churn |
| **CUI** | No cursor (chat UI) | React list (good) | CSS variables + Tailwind (good) | 13px font may cause strain for extended use |
| **shellinabox** | JS `setInterval(500ms)` toggling CSS class (acceptable) | CSS with 2000-line limit (good) | CSS classes on spans (good) | Native DOM text = best text quality |
| **ttyd** | Delegated to xterm.js WebGL (JS interval, medium) | xterm.js scrollbar (good) | xterm.js WebGL rectangle shader (good) | Rated Low overall due to GPU acceleration + flow control |

### Medium Strain Projects

| Project | Primary Strain Factor | Mitigation |
|---------|----------------------|------------|
| **xterm.js WebGL** | Grayscale AA from canvas `fillText` rasterization. JS `setInterval(600ms)` cursor blink triggers full viewport GPU redraw | Texture atlas caches glyphs. 5min idle timeout. RAF batching prevents redundant frames |
| **WeTTY** | DOM renderer is slowest for high-throughput output (layout thrashing). Dated default font (Courier New) | Flow control (tinybuffer 2ms/512KB batching). Subpixel AA compensates for font choice |
| **claudecodeui** | Terminal tab uses WebGL addon (same as xterm.js WebGL above) | Chat tab is pure DOM. Dark theme. Resize debounce |
| **claude-code-remote** | Canvas-based xterm.js via ttyd in iframe. Iframe reload on tab wake causes flash | Dark theme. ttyd flow control. 14px Menlo is readable |
| **GoTTY** | Canvas 2D text rendering (no subpixel AA). Pre-v5 xterm.js | xterm.js internal RAF. Base64 overhead slightly throttles updates |

### High Strain Projects

None of the investigated projects reach high strain risk. All use some form of update batching (RAF, setTimeout, or React reconciler) and none implement aggressive custom repaint loops without dirty tracking.

### Strain Ranking (Best to Worst for Extended Reading)

1. **DomTerm** - Pure DOM, CSS cursor blink, native scrollbar, Cascadia Code default font
2. **hterm** - Pure DOM, virtual viewport (excellent scrollback perf), configurable blink
3. **shellinabox** - Pure DOM, DejaVu Sans Mono, simple architecture, no framework overhead
4. **xterm.js DOM** - Pure DOM, CSS cursor blink, RAF batching, but custom scrollbar
5. **claude-code-webui** - React DOM, no terminal artifacts, smooth streaming
6. **CUI** - React DOM, but 13px font is small for extended use
7. **WeTTY** - DOM renderer (good text), but Courier New font and DOM perf under load
8. **ttyd** - WebGL default is fast but grayscale AA; DOM fallback available via URL param
9. **claudecodeui** - WebGL terminal tab; chat tab is fine
10. **GoTTY** - Canvas 2D, no subpixel AA, archived
11. **claude-code-remote** - Canvas via ttyd in iframe, reload flicker

### Key Takeaways for Display-Strain-Free Rendering

1. **DOM text rendering is definitively better for reading comfort** than canvas/WebGL. Browser-native text gets subpixel antialiasing (ClearType on Windows, LCD smoothing on macOS), font hinting, and kerning. Canvas `fillText()` produces only grayscale antialiasing.

2. **CSS cursor blink >> JS interval cursor blink.** CSS `@keyframes` with `step-end` requires zero JS execution and zero repaints beyond the cursor element. JS `setInterval` triggers callback execution and (in WebGL) full viewport GPU redraws every 600ms.

3. **Virtual scrolling matters.** hterm's virtual viewport (only visible rows in DOM) is the gold standard. DomTerm and shellinabox lack this and accumulate DOM nodes for scrollback, which degrades performance over long sessions.

4. **xterm.js v5/v6 DOM default was a strain-positive change.** The v4->v5 migration from canvas-default to DOM-default directly improves reading comfort for all consumers that don't explicitly opt into WebGL.

5. **Font choice matters more than renderer for eye strain.** WeTTY with DOM rendering but Courier New is worse than ttyd with WebGL but Consolas/Menlo. The best projects (DomTerm, hterm) ship with modern font stacks (Cascadia Code, DejaVu Sans Mono).

6. **Flow control prevents render flooding.** ttyd's PAUSE/RESUME backpressure and WeTTY's tinybuffer both prevent the terminal from being overwhelmed during high-throughput output (cat large-file, build logs). This is critical for preventing frame drops and visual jank.

7. **Chat UIs are inherently low-strain** because they render static message blocks (append-only) rather than character-by-character terminal updates. React reconciler batching prevents rapid DOM mutations during streaming.
