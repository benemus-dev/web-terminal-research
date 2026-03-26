# Freeze Script Analysis: claude.ai Idle Rendering Behavior

What a Tampermonkey script that intercepts RAF and injects CSS animation/transition
blanket overrides tells us about claude.ai's rendering behavior, and what "truly calm"
would actually require.

---

## 1. What RAF Callbacks claude.ai Is Likely Running at Idle

The freeze script catches RAF callbacks that are registered even when the user is not
interacting. claude.ai is a Next.js/React application. These are the probable RAF sources,
ranked by likelihood:

### High Confidence (React/Next.js Framework)

| Source | Why It Runs at Idle | Evidence Pattern |
|--------|-------------------|-----------------|
| **React Concurrent Mode scheduler** | React 18's `scheduler` package uses `MessageChannel` for task scheduling, but falls back to RAF for animation-priority updates. Even when no state changes occur, if the scheduler has pending work from a prior render cycle (e.g., a low-priority hydration pass), it fires RAF to check. | RAF callback containing `performWorkUntilDeadline` or `workLoop` |
| **Framer Motion / animation library** | claude.ai uses Framer Motion for UI transitions (message appearance, sidebar, etc.). Framer Motion maintains a global animation loop via `sync` (formerly `popmotion`). Even when no animation is active, the library's `frameloop` keeps a RAF registered to detect new animations. It self-cancels after ~1 frame of inactivity in recent versions, but older versions hold the RAF open. | RAF callback referencing `frameloop`, `sync`, or `motionValue` |
| **Resize/layout measurement hooks** | Common React pattern: `useLayoutEffect` + RAF to measure element dimensions after render. Libraries like `@radix-ui` (used for claude.ai's dialog, dropdown, popover components) schedule measurement RAFs to position floating elements. Even at idle, if a popover or tooltip component is mounted (but hidden), its positioning logic may hold a RAF. | RAF containing `getBoundingClientRect`, `offsetWidth`, or `getComputedStyle` |

### Medium Confidence (Library Ecosystem)

| Source | Why It Runs at Idle | Evidence Pattern |
|--------|-------------------|-----------------|
| **Smooth scroll / scroll position tracking** | Next.js router and libraries like `scroll-into-view-if-needed` use RAF for scroll interpolation. If a conversation was scrolled to bottom and a `scrollTo` animation didn't complete cleanly, the RAF loop may persist. | RAF containing `scrollTop`, `scrollTo`, `scrollBy` |
| **Intersection Observer polyfill / supplement** | Some React lazy-loading libraries (for code blocks, images in conversation) supplement native IntersectionObserver with RAF-based polling for edge cases (iframes, transformed containers). | RAF containing intersection/viewport checks |
| **Syntax highlighting (Prism / Shiki)** | claude.ai renders code blocks with syntax highlighting. Some highlighters use RAF to defer tokenization of large blocks, leaving a pending RAF callback even after rendering completes. | RAF containing tokenization or highlighting logic |
| **Virtualized list (conversation messages)** | If claude.ai virtualizes long conversations (likely, given potentially thousands of messages), the virtualizer (react-virtuoso, react-window, or custom) maintains a RAF for scroll-position-based rendering decisions. | RAF containing `overscan`, `visibleRange`, `startIndex` |
| **Toast/notification timeout animations** | Sonner or react-hot-toast (common in Next.js apps) use RAF for progress bar animation on toasts. Even after dismissal, cleanup may not cancel the RAF. | RAF with timer/progress calculations |

### Lower Confidence (But Possible)

| Source | Why It Runs at Idle | Evidence Pattern |
|--------|-------------------|-----------------|
| **Analytics/telemetry frame counting** | Some telemetry SDKs (Sentry, Datadog RUM) use RAF loops to measure Long Animation Frames, frame drops, and CLS. These run continuously by design. | RAF containing performance measurement, `performance.now()` deltas |
| **WebSocket reconnection UI** | claude.ai maintains a WebSocket/SSE connection. Some reconnection libraries animate a "reconnecting" indicator via RAF even when the connection is healthy (pre-rendering the reconnection animation). | RAF with connection state checks |
| **Cursor presence indicators** | If claude.ai shows "typing" indicators for the AI response, the animation driver may keep RAF alive between messages. | RAF with cursor/typing state |

### What Is NOT Likely at Idle

- **Monaco editor**: claude.ai does not embed a code editor for user input.
- **Canvas rendering**: claude.ai uses DOM-based rendering, not canvas.
- **WebGL**: No 3D or WebGL content.
- **Service worker RAF**: Service workers don't have RAF.

---

## 2. What CSS Animations claude.ai Likely Has at Idle

The freeze script injects `animation: none !important; animation-play-state: paused !important;
transition: none !important;` on all elements. The fact that this "makes it a bit better"
confirms active CSS animations at idle.

### Confirmed Likely Sources

| Animation | CSS Pattern | Idle Impact |
|-----------|------------|-------------|
| **Streaming cursor / thinking indicator** | `@keyframes pulse` or `@keyframes blink` on a `<span>` or `<div>` that persists after response completes. Common bug: the element stays in DOM with `opacity: 0` but animation still running. | Compositor frames every cycle (~60fps during blink, or 1-2fps for slow pulse) |
| **Input textarea caret** | Not CSS-animated (browser-native), but claude.ai may have a custom caret overlay with CSS `@keyframes`. The ProseMirror or Tiptap editor used for rich input sometimes implements custom carets. | ~1.5fps compositor frames (500ms blink cycle = 2 state changes/sec) |
| **Skeleton loading placeholders** | `@keyframes shimmer` on skeleton screens. If a skeleton component is rendered but hidden (`display: none` does NOT stop `@keyframes` unless the element is removed from DOM; `visibility: hidden` also does NOT stop it), it still produces compositor frames. | 60fps compositor frames while shimmer is active |
| **Button hover/focus ring transitions** | `transition: background-color 0.2s, box-shadow 0.2s` on buttons and interactive elements. These are declarations and are INERT at idle -- they only activate during state changes. The freeze script's `transition: none` is overly aggressive here; it eliminates transition declarations that would only fire on interaction. | Zero idle impact (transitions are not animations) |
| **Sidebar notification dot** | `@keyframes pulse` on a badge or notification indicator. If a conversation has unread messages, the dot may pulse continuously. | ~2fps compositor frames |
| **Message appearance animation** | `@keyframes fadeInUp` or Framer Motion `initial/animate` on new messages. After animation completes, this is inert. But if Framer Motion's exit animation cleanup fails, the animation may loop. | Should be zero at idle; nonzero = library bug |
| **Scroll-to-bottom button** | `@keyframes bounce` or `transition: opacity` on the "scroll to bottom" FAB that appears when scrolled up. If always mounted with animation, causes continuous compositor frames. | ~1-2fps if using slow animation cycle |
| **Focus ring on input** | `@keyframes` pulse on the focused input border. Some design systems animate the focus ring with a subtle pulse rather than a static outline. | ~1-2fps compositor frames |

### What the Blanket CSS Override Actually Does

```css
*, *::before, *::after {
  animation-play-state: paused !important;
  transition: none !important;
  animation: none !important;
}
```

This eliminates:
- All `@keyframes` animations (both active and would-be-active)
- All CSS transitions (both active and declarative)
- All `::before`/`::after` pseudo-element animations (loading spinners, decorative pulses)

Side effects:
- **Breaks intentional hover feedback** (no transition on button hover = instant snap)
- **Breaks Framer Motion** (Framer Motion uses inline `transition` styles set via JS; the
  `!important` override blocks them, causing visual glitches or stuck states)
- **Breaks streaming response animation** (if the typing effect uses CSS transitions for
  opacity on new tokens)

---

## 3. What the Freeze Script CANNOT Catch

Even with RAF intercepted and all CSS animations paused, Chrome still redraws for these
reasons. This is why the user reports "it makes it a bit better" but not silent.

### Browser-Internal Timers (Not Accessible to JavaScript)

| Source | Mechanism | Frequency | Can Userscript Fix? |
|--------|-----------|-----------|-------------------|
| **`<textarea>` / `<input>` caret blink** | Chromium's `FrameCaret` class uses an internal `TaskRunnerTimer` tied to `BlinkPlatformImpl::CurrentTimeTicks`. This is a C++ timer in Blink, not a JS timer or CSS animation. It fires every 500ms to toggle caret visibility, causing a paint invalidation on the caret's layer. | ~2 state changes/sec = ~2 paint events/sec | **Partially**: `caret-color: transparent` hides the caret visually but the timer STILL fires and STILL causes paint invalidation (Chromium does not check if caret is transparent before invalidating). Only removing focus from the element stops the timer. |
| **`contenteditable` caret** | Same `FrameCaret` mechanism as `<textarea>`. ProseMirror/Tiptap (which claude.ai likely uses for the input editor) uses `contenteditable`, so this applies directly. | Same as above | Same as above |
| **Selection highlight repaint** | If text is selected (even outside the active element), the selection highlight can cause repaints when the selection changes or the browser re-validates it. | Irregular | Not practically fixable |

### Compositor-Level Activity

| Source | Mechanism | Can Userscript Fix? |
|--------|-----------|-------------------|
| **Scrollbar overlay fade** | On Windows 11 with overlay scrollbars enabled, the scrollbar fades in/out with a compositor-driven animation when the mouse approaches the scroll region. This is a native compositor animation, not CSS. | No. OS-level behavior. `scrollbar-width: none` in CSS removes the scrollbar entirely. |
| **Composited layer damage from neighboring tabs/windows** | Chrome's viz display compositor aggregates surfaces from all browser UI components. In rare cases, damage from the browser chrome (tab strip, omnibox) can propagate to page compositor. | No. Process-level. |
| **GPU texture cache eviction** | If GPU memory pressure causes texture eviction, Chrome re-rasterizes affected layers on next frame. This causes a redraw even with zero content changes. | No. Hardware-level. |

### DOM/Platform Features

| Source | Mechanism | Can Userscript Fix? |
|--------|-----------|-------------------|
| **WebSocket/SSE `message` events** | claude.ai maintains a connection for conversation updates. Each message event triggers JS execution, which may cause React state updates and DOM mutations even if the message is a heartbeat/keepalive. | Could override `WebSocket.prototype` but would break the app. |
| **`MutationObserver` / `ResizeObserver` / `IntersectionObserver` callbacks** | React and its ecosystem register observers that fire callbacks which may schedule renders. `ResizeObserver` fires when browser layout changes (e.g., scrollbar appearance), and this can cascade into React renders. | Could disconnect all observers, but would break layout. |
| **`setTimeout` / `setInterval` DOM mutations** | Any timer that modifies DOM causes rendering work. The freeze script only intercepts RAF, not `setTimeout`/`setInterval`. If claude.ai has timers that update DOM (e.g., relative time stamps "2 minutes ago" -> "3 minutes ago"), these cause redraws. | Could intercept `setTimeout`/`setInterval` but this breaks the app (breaks React, breaks connection keepalive, breaks everything). |
| **Chrome extension content scripts** | Extensions injecting DOM, running observers, or registering timers in the page's context cause redraws attributed to the page. Each extension badge animation also causes compositor frames. | Disable extensions. Or use `--disable-extensions` flag. |
| **`<img>` decode completion** | If images in the conversation are lazily loaded and their decode completes while idle, the decoded bitmap replaces the placeholder, causing a paint. | Not practically fixable (and desirable behavior). |

### Things That Are NOT Causing Redraws (Common Misconceptions)

- **WebSocket connection itself**: An open WebSocket with no messages causes zero rendering work.
- **React virtual DOM diffing with no DOM output**: Zero compositor cost (JS CPU only).
- **Event listener registration**: Passive listeners cause zero rendering overhead.
- **CSS `transition` declarations** (not active transitions): Zero idle cost.
- **`will-change` without animation**: Layer promotion but zero continuous frames.
- **Static `position: fixed` elements**: Zero idle compositor cost.

---

## 4. What a "Truly Calm" Page Would Need

Design spec for a web page that causes ZERO Chrome rendering pipeline activity at idle.

### Mandatory Requirements

| Requirement | Reason | Implementation |
|-------------|--------|---------------|
| **No `<input>`, `<textarea>`, or `contenteditable`** | Browser-internal caret timer causes paint invalidation every 500ms. Cannot be stopped by JS or CSS. | Use a `<div>` with `keydown`/`keyup` event listeners and custom caret rendering (but then you need to NOT use CSS animation for the caret either). |
| **No CSS `@keyframes` with `infinite` iteration** | Compositor produces frames for every animation cycle, even if the element is invisible. | Use `animation-iteration-count` with finite values, or remove animated elements from DOM when not visible. |
| **No active CSS transitions** | Transitions cause compositor frames during interpolation. | Declarations are fine; just ensure no property is actively transitioning at idle. |
| **No RAF loop callbacks** | Any RAF that re-registers itself runs JS every frame. If the callback touches DOM, it causes rendering work. Even no-op RAF has JS execution cost. | All RAF must be one-shot (demand-driven). |
| **No `setTimeout`/`setInterval` that touch DOM** | Timer-driven DOM mutations cause full rendering pipeline execution. | Timers for non-visual work (network, data) are fine. Timers that update DOM are not. |
| **No `<video>` or `<canvas>` with active content** | Hardware-decoded video and canvas animation loops produce continuous compositor frames. | Pause video. Stop canvas animation. |
| **No composited `<iframe>` with active content** | Each iframe is a separate renderer process, but its surface is composited into the parent page. Active content in iframe = damage in parent compositor. | Ensure iframe content is also calm. |

### Strongly Recommended

| Requirement | Reason | Implementation |
|-------------|--------|---------------|
| **`scrollbar-width: none`** (or classic non-overlay scrollbars) | Windows 11 overlay scrollbar fade animation is compositor-driven and not controllable by CSS. | Apply to all scrollable containers. Use custom scrollbar UI if scroll position indication is needed. Or use `overflow: hidden` on static content. |
| **Minimal compositor layers** | Each layer is a surface in the viz compositor. More layers = more damage-checking work per potential frame. | Avoid `will-change`, `transform: translateZ(0)`, `backface-visibility: hidden` unless actively animating. Use DevTools Layers panel to audit. |
| **`contain: strict` on all independent sections** | Scopes style/layout/paint invalidation. If one section gets a forced redraw (e.g., from extension injection), the invalidation doesn't cascade. | Apply to all top-level layout containers. |
| **`content-visibility: auto` on off-screen content** | Eliminates rendering work for content outside viewport. | Apply to conversation message list items, scrollback, any long list. |
| **Disconnect all `ResizeObserver`s when not needed** | `ResizeObserver` fires on layout changes including scrollbar appearance/disappearance, window resize, and zoom. Callbacks often trigger state updates that cause renders. | Connect on mount, disconnect when component is stable. |
| **`pointer-events: none` on non-interactive elements** | Reduces hit-testing work during mouse movement. Hit testing is compositor-thread work that can trigger main-thread callbacks. | Apply to decorative elements, backgrounds, static text containers. |
| **No `MutationObserver` on large subtrees** | Mutation observers on `document.body` with `subtree: true` fire for every DOM change anywhere, including Chrome extension injections. | Scope observers to minimal subtrees. |

### Nice to Have

| Requirement | Reason | Implementation |
|-------------|--------|---------------|
| **`prefers-reduced-motion` media query** | OS-level signal that the user wants less motion. Respect it by disabling all animations. | `@media (prefers-reduced-motion: reduce) { *, *::before, *::after { animation: none; transition: none; } }` |
| **Tab `visibilitychange` handler** | Stop all visual work when tab is hidden. Chrome already throttles background tabs, but not to zero. | `document.addEventListener('visibilitychange', () => { if (document.hidden) stopEverything(); })` |
| **No custom fonts loading** | Font swap causes re-layout and re-paint of all text using that font. If a font loads asynchronously after idle, it causes a redraw burst. | Preload critical fonts. Use `font-display: optional` (skip font if not loaded by first paint). |
| **Static `favicon`** | Animated favicons (GIF, canvas-drawn) cause compositor frames in the tab strip, which can propagate. | Use static PNG favicon. |

### The Theoretical Minimum

A page with absolutely ZERO idle rendering activity would look like:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    * { animation: none; transition: none; }
    html { overflow: hidden; scrollbar-width: none; }
  </style>
</head>
<body>
  <div>Static text content only. No inputs. No images loading. No scripts.</div>
</body>
</html>
```

Chrome's compositor would produce zero frames for this page. The `cc::Scheduler` would
have `needs_begin_frame = false`, and the `viz::DisplayScheduler` would stop issuing
`BeginFrame` signals entirely. GPU memory would hold a single rasterized texture. CPU
usage would be effectively zero (only OS-level thread scheduling overhead).

**The gap between this and a usable interactive application is the entire problem.**

---

## 5. Practical Improvements for the Freeze Script

Prioritized by impact and feasibility, ordered from "add this now" to "technically possible
but may break things."

### Tier 1: Safe, High Impact

| Improvement | What It Does | How to Implement |
|-------------|-------------|-----------------|
| **Intercept `setInterval` and `setTimeout`** | Catch timer-driven DOM mutations (relative timestamps, polling loops, analytics beacons). Don't block all timers -- only those whose callbacks touch DOM. | Override `window.setInterval` and `window.setTimeout`. When frozen, store callbacks. On unfreeze, replay with adjusted delays. Problem: cannot know if a callback touches DOM without executing it. Alternative: intercept and delay all timers over a threshold (e.g., block intervals < 5s, allow longer ones). |
| **`caret-color: transparent` on all inputs** | Hides the visual caret. The browser timer still fires but the paint invalidation produces identical pixels (transparent caret on = transparent caret off), so Chrome's damage detection MAY skip the redraw. In practice, Chromium still invalidates the caret rect, but the paint output is the same so rasterization can be skipped. Net effect: reduced (not zero) CPU from caret blink. | `document.querySelectorAll('input, textarea, [contenteditable]').forEach(el => el.style.caretColor = 'transparent')` |
| **Force `scrollbar-width: none`** | Eliminates overlay scrollbar fade animations on Windows 11. These are compositor-driven and not caught by RAF/CSS interception. | Inject CSS: `:root, * { scrollbar-width: none !important; }` |
| **Disconnect `ResizeObserver` and `IntersectionObserver`** | Prevents observer callbacks from triggering React re-renders at idle. These fire on subtle layout changes (zoom, scrollbar, window resize) and often cascade into state updates. | Override `ResizeObserver.prototype.observe`, `IntersectionObserver.prototype.observe`. When frozen, call `disconnect()` on all active observers. Store references for reconnection on unfreeze. |

### Tier 2: Moderate Impact, Some Risk

| Improvement | What It Does | Risk |
|-------------|-------------|------|
| **Blur the active element** | `document.activeElement.blur()` stops the browser caret timer entirely. This is the ONLY way to truly stop `FrameCaret`'s internal timer. | Loses keyboard focus. User must click back into input to type. Could add a keyboard shortcut (e.g., Escape to freeze + blur, Enter to unfreeze + refocus). |
| **Override `Element.prototype.scrollIntoView`** | Prevents scroll animations triggered by React component lifecycle (auto-scroll to bottom on new messages, scroll to focused element). | May break navigation within conversation. Only override when frozen. |
| **Intercept `MutationObserver` callbacks** | Prevent mutation observers (used by React DevTools, analytics, accessibility tools, and the app itself) from triggering cascading renders. | Breaks React internals if React uses MutationObserver (it generally doesn't, but third-party libraries do). |
| **Force `will-change: auto` on all elements** | De-promotes compositor layers. Fewer layers = less compositor overhead. | May cause visual glitches or performance regression when unfreezing (elements need re-promotion). |

### Tier 3: Aggressive, High Risk

| Improvement | What It Does | Risk |
|-------------|-------------|------|
| **Replace `<textarea>` with a capturing `<div>`** | Eliminates browser caret timer entirely while maintaining keyboard input. | Loses all native input behavior: autocomplete, spell check, IME composition, selection, paste formatting, accessibility. Enormous implementation effort for minimal gain over Tier 2's blur approach. |
| **Inject `<meta name="viewport">` to disable zoom** | Prevents pinch-zoom from triggering compositor layer reconfiguration. | Desktop browsers mostly ignore this. Not useful on claude.ai. |
| **Override `performance.now()`** | Some RAF-like patterns use `performance.now()` deltas to decide whether to re-render. Freezing the clock would prevent time-based render triggers. | Breaks everything that depends on time: animations, performance monitoring, WebSocket ping timing. |
| **Force `document.hidden = true`** | Tricks the page into thinking the tab is hidden, triggering background-tab throttling behavior. Many apps disable visual work when hidden. | Cannot override getter on `document.hidden` reliably (it's a native property). Would break if app checks `document.visibilityState`. |
| **`pointer-events: none` on everything** | Eliminates all hit testing. Compositor thread does zero work on mouse movement. | Page becomes unclickable. Must be toggled off before interaction. |

### Recommended Script Architecture

A more comprehensive freeze script would have three modes:

**Mode 1: Light Freeze (current + Tier 1)**
- Intercept RAF (current behavior)
- Pause CSS animations (current behavior)
- Hide carets (`caret-color: transparent`)
- Remove scrollbars (`scrollbar-width: none`)
- Disconnect observers

**Mode 2: Deep Freeze (+ Tier 2)**
- Everything in Light Freeze
- Blur active element (stop caret timer)
- Block `scrollIntoView`
- Set `pointer-events: none` on non-interactive elements
- Intercept short-interval timers (< 2s)

**Mode 3: Full Stop (+ Tier 3)**
- Everything in Deep Freeze
- `pointer-events: none` on everything
- Force `will-change: auto` on all elements
- Intercept ALL timers
- Block all observer callbacks

Unfreeze must restore all overridden behaviors in reverse order.

---

## 6. The Fundamental Problem

claude.ai is a React 18 application with Framer Motion, Radix UI, and a rich text editor
(likely Tiptap/ProseMirror). This stack is architecturally incapable of zero-idle-activity
because:

1. **React's reconciler is event-driven but not compositor-aware.** React does not know or
   care whether its renders cause compositor frames. It optimizes for correct UI, not for
   compositor silence.

2. **Framer Motion maintains a global animation frame loop.** Even when no animations are
   active, the frame loop infrastructure has overhead (though recent versions minimize this).

3. **ProseMirror/Tiptap uses `contenteditable`.** This means browser-internal caret
   blink that no JavaScript can stop (only blur can).

4. **The WebSocket connection triggers periodic React state updates.** Even heartbeat/ping
   messages may update connection state in React, causing reconciliation cycles.

5. **Modern CSS design systems assume transitions are free.** They are free when not
   transitioning, but the sheer number of `transition` declarations means any state change
   triggers multiple simultaneous transitions across many elements.

A web application built for true visual calm would need to be built from the ground up with
that as an architectural constraint -- not bolted on after the fact. The freeze script is
fighting against the grain of the entire web platform.

### What Would "Truly Calm claude.ai" Look Like?

- **No `contenteditable`**: Custom input handling with no browser caret.
- **No Framer Motion**: CSS transitions only, applied surgically, not globally.
- **No blanket `transition` declarations**: Explicit `transition: none` on all elements,
  with transitions enabled only on specific user interactions (hover, click) and immediately
  removed after completion.
- **`content-visibility: auto`** on all conversation messages outside viewport.
- **`contain: strict`** on conversation list, sidebar, and input area.
- **Static rendering by default**: No RAF, no intervals, no observers. All rendering
  triggered exclusively by user input events (keydown, click, scroll) with immediate
  quiescence after the event is processed.
- **Server-Sent Events with no UI update on heartbeat**: Connection keepalive handled
  in a Web Worker with zero main-thread involvement.

This would produce a page that, between user interactions, has literally zero CPU usage
and zero compositor frames. The screen would be as still as a static document.

---

## What I Haven't Checked / What Could Be Wrong

1. **claude.ai's actual framework versions**: The analysis assumes React 18 concurrent mode,
   Framer Motion v10+, and ProseMirror/Tiptap. The actual versions may differ, changing the
   RAF behavior (e.g., React 17 doesn't use concurrent mode scheduler at all).

2. **Chrome version differences**: Compositor behavior varies between Chrome versions.
   The caret timer internals changed in Chrome 120+ with the `FrameCaret` refactor.
   `scrollbar-width` support landed in Chrome 121.

3. **Whether the streaming response cursor is CSS or JS**: If it's JS-driven (via
   `setInterval` toggling a class), the CSS freeze catches it. If it's purely CSS
   `@keyframes`, the CSS freeze catches it. But if it's a custom `<canvas>` cursor or
   SVG animation, neither interception works.

4. **Extension interference**: The Tampermonkey script itself is a content script that
   may cause minor overhead (MutationObserver on script injection, periodic Tampermonkey
   API calls).

5. **Whether `animation: none !important` actually stops all animation types**: CSS
   Houdini `@property` registered custom properties with `inherits: true` may have
   animation behavior not covered by the blanket override. Web Animations API
   (`element.animate()`) is not stopped by CSS `animation: none` -- it requires
   `Animation.pause()` on each instance.

6. **Web Animations API**: If claude.ai or Framer Motion uses `element.animate()` (the
   Web Animations API) instead of CSS `@keyframes`, the CSS override does nothing. Framer
   Motion v11+ uses WAAPI by default. This would be a significant gap in the freeze script.
   Fix: `const origAnimate = Element.prototype.animate; Element.prototype.animate = function(...args) { const anim = origAnimate.apply(this, args); if (frozen) anim.pause(); return anim; };`
