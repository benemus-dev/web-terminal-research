# Chrome/Chromium Rendering Pipeline: Idle Redraw Analysis

What causes screen redraws when JavaScript is completely idle? This document covers every
layer of the pipeline, from CSS property changes down to compositor scheduling decisions.

---

## 1. Rendering Pipeline Stages

Chrome's pixel pipeline has five stages. Every visual change flows through some subset of them.

```
JavaScript/CSS change  -->  Style  -->  Layout  -->  Paint  -->  Composite
```

### Stage Details

| Stage | What it does | When it runs |
|-------|-------------|-------------|
| **Style** | Matches CSS selectors to DOM nodes, computes final styles | Any DOM mutation, class change, pseudo-class change (`:hover`), media query match change |
| **Layout** | Calculates geometry: width, height, position of every element | Any change to `width`, `height`, `top`, `left`, `margin`, `padding`, `display`, `position`, `font-size`, `float`, `overflow`, or reading `offsetWidth`/`getBoundingClientRect` (forces synchronous layout) |
| **Paint** | Records draw commands (text, colors, borders, shadows, images) into display lists | Any change to `background-color`, `color`, `border-color`, `box-shadow`, `visibility`, `border-radius`, `outline`, `background-image` |
| **Composite** | Composites separately-rasterized layers into the final frame on the GPU | Any change to `transform`, `opacity`, `filter`, `backdrop-filter`, `will-change` (on its own layer) |

### Three Execution Paths

1. **Full pipeline** (JS > Style > Layout > Paint > Composite): Triggered by geometry changes (`width`, `height`, `top`, `left`).
2. **Skip Layout** (JS > Style > Paint > Composite): Triggered by paint-only properties (`color`, `background-color`, `box-shadow`).
3. **Compositor-only** (JS > Style > Composite): Triggered by `transform` and `opacity` on promoted layers. The cheapest path.

### Which Stages Run When JS Is Idle?

When JavaScript is completely idle and no DOM mutations occur:

- **Style**: Does NOT run. Style recalc only fires when something invalidates computed styles.
- **Layout**: Does NOT run. Layout only fires when geometry is invalidated.
- **Paint**: Does NOT run. Paint records are cached until content changes.
- **Composite**: **CAN still run**. The compositor thread operates independently and will produce frames if:
  - A CSS animation or transition is active (even if visually imperceptible)
  - Scrolling is in progress (compositor-thread scroll)
  - A composited layer is being animated by the compositor thread
  - Damage is detected in any compositor layer

The compositor is the only stage that can run without any main-thread involvement.

---

## 2. Compositor-Driven Redraws Without JS

### Does the Compositor Run at 60fps Continuously?

**No. Chrome's compositor is demand-driven, not continuous.**

The `cc::Scheduler` decides whether to produce a frame each vsync based on:
- Whether `SetNeedsRedraw` has been called (visual damage exists)
- Whether `SetNeedsCommit` has been called (main-thread content changed)
- Whether compositor animations are active
- Whether scrolling is in progress

If nothing has changed, the compositor does NOT produce a new frame. The `viz::DisplayScheduler`
only issues `BeginFrame` signals when there is pending damage to composite.

**However**, several CSS features silently keep the compositor producing frames:

### CSS Properties That Force Continuous Compositing

| CSS Feature | Continuous Frames? | Why |
|-------------|-------------------|-----|
| `@keyframes` animation with `infinite` | **YES** | Compositor ticks every frame for the duration. Even `opacity: 0` elements with `animation: spin infinite` consume CPU -- measured at ~10% in Chrome, ~20-30% in Firefox. |
| `transition` (while transitioning) | **YES** (during transition) | Compositor interpolates values each frame. Stops when transition completes. |
| `will-change: transform` | No (by itself) | Promotes to compositor layer but does NOT cause continuous drawing. Only triggers frames when the property actually changes. |
| `will-change: opacity` | No (by itself) | Same as above -- layer promotion only, no continuous frames. |
| `filter` / `backdrop-filter` | No (static) | Only causes frames during animation. Static filters are composited once and cached. |
| CSS cursor blink (`@keyframes` with `infinite`) | **YES** | Every cursor blink cycle produces compositor frames. This is why JS-driven blink with `setInterval` + DOM mutation is actually WORSE (triggers paint), but CSS-only blink still causes compositor work. |

### CSS Transitions/Animations When Not Visually Changing

**Critical finding**: CSS `animation: infinite` causes compositor frames even when the element
is `opacity: 0`, `visibility: hidden`, or completely off-screen (unless the element is removed
from DOM entirely). The animation timeline still ticks.

However, `transition` declarations alone do NOT cause activity. A `transition: all 0.3s` rule
is inert until a property actually changes value. The compositor only activates during the
transition's interpolation window.

### position: fixed / position: sticky

- `position: fixed` elements are **promoted to their own compositor layer** (especially on
  high-DPI displays and mobile). This promotion is for scroll performance -- the compositor
  can reposition the fixed element without repaint during scrolling.
- `position: sticky` elements also get compositor layer promotion when their scroll container
  is composited.
- **Neither causes continuous compositing when idle.** They only cause compositor work during
  scroll events.
- However, implicit compositing can cascade: if a `z-index` sibling overlaps a fixed/sticky
  layer, it too gets promoted (see section 3).

### overflow: auto with Scrollable Content

- Scrollable containers (`overflow: auto/scroll`) are promoted to compositor layers for
  threaded scrolling (allowing scroll without main-thread involvement).
- **When idle (not scrolling), no continuous compositing occurs.** The compositor only
  produces frames during active scroll.
- The layer exists in GPU memory but costs zero compositor frames when static.

### Layered Elements (z-index Stacking)

- Static `z-index` stacking does NOT cause continuous compositing.
- However, z-index stacking CAN trigger **implicit compositing** (see section 3), which
  increases memory usage and compositor complexity.

---

## 3. What Triggers "Composite Layers" in Chrome DevTools

### Explicit Layer Promotion (Direct Compositing Reasons)

An element gets its own compositor layer when it has:
- `will-change: transform | opacity | filter`
- `transform: translateZ(0)` or `translate3d(0,0,0)` (the classic "force GPU" hack)
- An active CSS animation on `transform` or `opacity`
- `position: fixed` (on high-DPI or mobile)
- `<video>` with hardware-decoded content
- `<canvas>` with WebGL or accelerated 2D context
- `overflow: scroll/auto` (for composited scrolling)
- Composited CSS filters

### Implicit Compositing (The Silent Layer Explosion)

**This is the most insidious cause of excessive compositing.**

When a non-composited element overlaps (in paint order / z-index) with a composited element,
Chrome MUST promote the overlapping element to its own layer to preserve correct stacking.
This is called **implicit compositing** or **assumedOverlap**.

Example: Element A has `will-change: transform` (composited). Element B has a higher z-index
and visually overlaps A. Element B gets implicitly promoted to a compositor layer even though
it has no compositing reason of its own.

This can cascade: if Element C overlaps B, it also gets promoted. This is a **layer explosion**.

### Layer Squashing

Chrome mitigates layer explosions with **layer squashing**: multiple implicitly-promoted
elements that overlap the same composited layer get merged into a single backing texture.
This reduces memory but not compositor complexity entirely.

### How Many Layers Is Too Many?

There is no hard threshold. Each layer:
- Consumes GPU memory (proportional to its pixel dimensions)
- Adds compositor overhead for damage tracking and compositing
- On mobile, GPU memory is severely constrained

Guidance: Use Chrome DevTools > Layers panel to inspect. If you see dozens of layers on a
simple page, investigate. A typical well-optimized page has 5-15 layers.

### DevTools Diagnostics

- **Layers panel**: Shows all compositor layers, their sizes, memory, and compositing reasons.
- **Paint Flashing** (Rendering tab > Paint flashing): Green overlay on every repainted area.
- **Layer Borders** (Rendering tab > Layer borders): Orange borders on compositor layers.
- **Scrolling performance issues** (Rendering tab): Highlights elements that slow scrolling.

---

## 4. React-Specific Redraw Triggers

### Does React's Reconciler Cause Redraws When State Hasn't Changed?

**No, if correctly implemented.** React's reconciliation algorithm diffs the virtual DOM
against the previous virtual DOM. If the diff produces zero DOM mutations, zero browser
rendering work occurs.

However, React CAN cause unnecessary redraws through:

1. **Re-renders without `React.memo` / `useMemo`**: A parent re-render causes all children
   to re-render (virtual DOM diff), even if props haven't changed. If the diff produces
   identical output, no DOM mutation occurs and no browser redraw happens. But the JS
   reconciliation work itself consumes CPU.

2. **Object/array identity changes**: `<Child data={{x: 1}} />` creates a new object every
   render, defeating `React.memo` shallow comparison. This causes a real DOM mutation
   (React sees different props).

3. **Context value changes**: Any context value change re-renders ALL consumers, even if
   the specific value they use didn't change.

### Do useEffect Cleanup/Setup Cycles Cause Layout Thrashing?

**Yes, potentially.**

- `useEffect` runs asynchronously after paint -- generally safe.
- `useLayoutEffect` runs synchronously BEFORE paint, and if it reads layout properties
  (`offsetWidth`, `getBoundingClientRect`) and then writes styles, it causes **forced
  synchronous layout** (layout thrashing).
- Pattern: `useLayoutEffect` that measures DOM + sets state = double render + layout thrash.

### Does React's Event Delegation Keep the Compositor Warm?

**No.** React 17+ attaches a single event listener per event type to the React root container
(not `document`). These are passive listeners that:
- Do NOT cause compositor activity when idle
- Do NOT trigger style recalc or layout
- Only fire when actual user events occur

React 16 and earlier attached to `document`, but even that didn't cause compositor activity.
Event listener registration alone never causes rendering work.

### Virtual DOM Diff = Zero DOM Mutation = Zero Redraw?

**Correct.** If reconciliation produces zero DOM mutations:
- No style recalc
- No layout
- No paint
- No composite
- Zero rendering pipeline activity

The only cost is the JavaScript CPU time for the diff itself (which can be significant for
large component trees, but is invisible to the rendering pipeline).

---

## 5. CSS Properties: Free vs Expensive at Idle

### Compositor-Only (Cheapest -- Skip Layout and Paint)

Only these two properties can be animated without triggering layout or paint:
- `transform` (translate, rotate, scale, skew)
- `opacity`

When on a promoted layer, changes to these go directly to the compositor thread.
`filter` and `backdrop-filter` can also be compositor-only on promoted layers.

### Paint-Only (Skip Layout, Still Repaint)

These trigger paint but not layout:
- `color`, `background-color`, `background-image`
- `border-color`, `border-radius`
- `box-shadow`, `text-shadow`
- `outline`, `outline-color`
- `visibility`

### Layout-Triggering (Most Expensive)

These trigger layout, paint, AND composite:
- `width`, `height`, `min-*`, `max-*`
- `top`, `right`, `bottom`, `left`
- `margin`, `padding`
- `display`, `position`, `float`, `clear`
- `font-size`, `font-family`, `font-weight`
- `line-height`, `text-align`, `vertical-align`
- `overflow`, `white-space`, `word-wrap`
- `border-width` (not border-color)

### Are CSS Custom Properties Recalculated Every Frame?

**No, not inherently.** Static CSS custom properties are resolved once during style computation
and cached. They do NOT recalculate every frame.

However:
- Dynamically setting a custom property via JS (`element.style.setProperty('--x', value)`)
  in a `requestAnimationFrame` loop DOES trigger style recalc every frame.
- Custom properties are inherited, so changing one on a parent triggers style recalc for ALL
  descendants. Setting custom properties at the deepest possible level minimizes this cost.
- Chrome 83+ optimized custom property inheritance to avoid whole-tree recalculation, but
  descendant recalc still applies.

### Does `transition: all` Cause Continuous Style Recalc?

**No.** `transition: all 0.3s` is a declaration, not an active animation. It is completely
inert until a property value actually changes. The transition only activates during the
interpolation period (e.g., 0.3s) and then stops.

The performance concern with `transition: all` is different: when a property DOES change,
`all` means Chrome must check every animatable property for transition eligibility, which
is more work than specifying `transition: opacity 0.3s` explicitly. But this is a one-time
cost per change, not continuous.

### What About `:hover` Styles on Many Elements?

- `:hover` only triggers style recalc when the mouse actually moves over/off an element.
- While the mouse is stationary, `:hover` causes zero rendering work.
- If `:hover` changes a layout property (e.g., `width`), it triggers full pipeline.
- If `:hover` changes `transform` or `opacity` on a promoted layer, it's compositor-only.
- With many elements (hundreds), `:hover` style recalc on mouse move CAN be expensive, but
  only during mouse movement, never at idle.

---

## 6. Chrome Flags and Settings to Reduce Redraws

### Command-Line Flags

| Flag | Effect |
|------|--------|
| `--disable-gpu-compositing` | Forces software compositing. All rendering on CPU. Eliminates GPU compositor but makes everything slower. Useful for isolating GPU vs CPU rendering issues. |
| `--disable-frame-rate-limit` | Removes vsync cap. Compositor will run as fast as possible. Useful for benchmarking, increases power consumption. May not work in recent Chrome versions. |
| `--disable-gpu-vsync` | Disables GPU vsync synchronization. Used with `--disable-frame-rate-limit`. |
| `--disable-gpu` | Disables GPU acceleration entirely. Software rendering fallback. |
| `--enable-gpu-rasterization` | Forces GPU rasterization of paint commands (may already be default). |
| `--show-composited-layer-borders` | Draws borders around compositor layers (same as DevTools layer borders). |
| `--show-paint-rects` | Flashes green rectangles on repainted areas (same as DevTools paint flashing). |
| `--disable-threaded-compositing` | Runs compositor on main thread instead of its own thread. |

### chrome://flags Relevant Entries

| Flag | Description |
|------|-------------|
| `#enable-gpu-rasterization` | GPU rasterization of paint commands |
| `#enable-oop-rasterization` | Out-of-process rasterization |
| `#enable-zero-copy` | Zero-copy rasterization (tiles rasterized directly to GPU memory) |
| `#enable-raw-draw` | Direct paint without intermediate layer |

### DevTools Rendering Tab

- **Paint flashing**: Green overlay on every repainted region.
- **Layout shift regions**: Blue overlay on layout shifts.
- **Layer borders**: Orange/olive borders showing compositor layer boundaries.
- **FPS meter**: Real-time FPS counter and GPU memory usage.
- **Scrolling performance issues**: Highlights slow-scroll elements.
- **Core Web Vitals**: Overlay showing LCP, CLS, INP.

### Hardware Acceleration Toggle

`Settings > System > Use hardware acceleration when available`

Disabling this forces Chrome into software rendering mode. This eliminates GPU compositing
entirely but significantly increases CPU load and reduces rendering performance. Not
recommended for reducing redraws -- it just moves the work to CPU.

---

## 7. CSS Containment and content-visibility

### CSS Containment (`contain`)

The `contain` property tells the browser that an element's subtree is independent from
the rest of the document, enabling rendering optimizations.

| Value | Effect |
|-------|--------|
| `contain: layout` | Element's internal layout is independent. No layout changes inside affect outside (and vice versa). |
| `contain: paint` | Element's painting is clipped to its bounds. Nothing inside can visually overflow. Establishes a new stacking context and containing block. |
| `contain: size` | Element's size is independent of its children. Must have explicit width/height. |
| `contain: style` | Counter and quote scoping (minimal rendering impact). |
| `contain: strict` | Equivalent to `contain: layout paint size style`. Maximum containment. |
| `contain: content` | Equivalent to `contain: layout paint style`. Most useful general-purpose value. |

**How containment reduces rendering work:**
- **Layout containment**: When content inside the container changes, layout recalculation
  is scoped to that container's subtree. The browser knows changes inside cannot affect
  siblings or ancestors, so it skips their layout.
- **Paint containment**: Creates an independent paint layer. Repaints are scoped.
- **Size containment**: Browser can skip laying out children entirely if it already knows
  the container's size.

### content-visibility

| Value | Effect |
|-------|--------|
| `content-visibility: visible` | Default. No optimization. |
| `content-visibility: hidden` | Element's content is never rendered (like `display: none` but preserves layout space). |
| `content-visibility: auto` | Browser skips rendering for off-screen elements. Gains layout + paint + size containment when off-screen. Renders normally when near viewport. |

**Performance impact of `content-visibility: auto`:**
- Measured 7x rendering performance improvement on initial page load in web.dev benchmarks.
- Off-screen elements get zero rendering work: no style, no layout, no paint.
- Elements gain containment automatically when off-screen.
- Uses `IntersectionObserver` internally to detect viewport proximity.
- Elements emit `contentvisibilityautostatechange` event when visibility changes.

### Usage in the 11 Web Terminal Projects

**None of the 11 audited terminal projects use `contain` or `content-visibility`.**

This is a missed optimization opportunity, especially for:
- Terminal scrollback buffers (hundreds/thousands of off-screen rows)
- Chat message histories (claude-code-webui, CUI)
- Split-pane layouts where panels are independent

`content-visibility: auto` on scrollback rows would eliminate rendering work for off-screen
terminal history. `contain: strict` on terminal viewport containers would scope all layout
recalculations to the terminal itself.

---

## 8. requestIdleCallback vs requestAnimationFrame

### requestAnimationFrame (RAF)

- Callback fires once per frame, synchronized to display refresh (typically 60Hz/16.6ms).
- **Registration alone does NOT cause compositor activity.** The compositor only produces
  frames if there is damage. A registered RAF callback is just queued JS work.
- However, if a RAF callback makes DOM mutations, those mutations trigger the rendering
  pipeline. A RAF loop that reads layout properties and writes styles causes layout thrashing
  every frame.
- One-shot RAF (call RAF once, don't re-register in callback) = no continuous overhead.
- Looping RAF (re-register in callback) = JS runs every frame, but compositor only draws if
  the callback produces visual changes.

### requestIdleCallback (RIC)

- Callback fires when the browser is idle (no pending frame work, no pending user input).
- Designed for non-urgent background work: analytics, prefetching, non-critical state updates.
- Does NOT trigger any rendering pipeline activity by itself.
- Has a `deadline` parameter indicating how much idle time remains (typically 0-50ms).
- Can specify `timeout` to ensure callback runs within a maximum delay.

### Which Terminal Projects Use requestIdleCallback?

**None of the 11 audited projects use `requestIdleCallback`.**

All projects that batch rendering use `requestAnimationFrame` or `setTimeout`:
- xterm.js: `RenderDebouncer` uses RAF for coalesced row rendering
- DomTerm: RAF for `_updateDisplay`
- hterm: `setTimeout` for deferred redraws
- GoTTY: xterm.js internal RAF

`requestIdleCallback` would be appropriate for:
- Scrollback buffer pruning
- Analytics/telemetry sending
- Non-visible DOM updates (off-screen row recycling)
- Accessibility tree updates

### Does RAF Registration Alone Cause Compositor Activity?

**No.** Registering a `requestAnimationFrame` callback without doing any visual work in it
does not cause the compositor to produce frames. The compositor's `cc::Scheduler` only
triggers `BeginFrame` when there is pending damage (SetNeedsRedraw/SetNeedsCommit). A no-op
RAF callback has zero rendering cost (only the JS execution cost of calling the empty function).

---

## 9. Chrome Tracing for Measuring Idle Redraws

### Relevant Trace Categories

| Category | What it captures |
|----------|-----------------|
| `cc` | Compositor (cc) events: commits, draws, tile management |
| `cc.debug` | Detailed compositor debugging info |
| `cc.debug.scheduler` | Scheduler state machine decisions (why a frame was produced) |
| `cc.debug.scheduler.frames` | Per-frame scheduler decisions |
| `viz` | Viz (display compositor) events: surface aggregation, display draws |
| `viz.hit_testing` | Hit testing in viz |
| `gpu` | GPU command buffer events |
| `benchmark` | Frame timing benchmarks |
| `blink` | Blink rendering engine events (style, layout, paint) |
| `blink.animations` | Animation lifecycle events |
| `disabled-by-default-cc.debug` | Verbose compositor events (must enable explicitly) |
| `disabled-by-default-cc.debug.scheduler` | Verbose scheduler decisions |
| `disabled-by-default-viz.debug.overlay_planes` | Overlay plane decisions |

### Key Trace Events

| Event | Meaning |
|-------|---------|
| `Scheduler::BeginFrame` | Vsync signal received by compositor scheduler |
| `Scheduler::BeginImplFrame` | Compositor frame started (compositor thread) |
| `Scheduler::BeginMainFrame` | Main thread frame started |
| `cc::LayerTreeHostImpl::DrawLayers` | Compositor actually drawing layers |
| `viz::Display::DrawAndSwap` | Display compositor drawing final frame |
| `viz::DirectRenderer::DrawFrame` | Actual GPU draw commands issued |
| `ProxyImpl::ScheduledActionDraw` | Scheduler decided to draw |
| `ProxyImpl::ScheduledActionCommit` | Scheduler decided to commit from main thread |
| `ThreadProxy::ScheduledActionSendBeginMainFrame` | Main thread being asked to produce content |
| `InsideBeginImplFrame` | Time spent inside a compositor frame |
| `NeedsBeginFrameChanged` | Compositor toggled its need for vsync signals |

### How to Measure "Redraws Per Second" at Idle

**Method 1: chrome://tracing**

1. Open `chrome://tracing`
2. Click "Record" and select categories: `cc`, `viz`, `benchmark`, `disabled-by-default-cc.debug.scheduler`
3. Navigate to the target page, let it sit idle for 10+ seconds
4. Stop recording
5. Search for `DrawAndSwap` or `DrawLayers` events
6. Count events per second during the idle period
7. Zero events = truly idle compositor. Any events = something is causing damage.

**Method 2: Performance tab in DevTools**

1. Open DevTools > Performance tab
2. Start recording
3. Let page sit idle for 10 seconds
4. Stop recording
5. Look at the "Frames" lane -- each green bar is a compositor frame
6. In the "Main" thread lane, look for "Composite Layers" entries
7. Idle page should show very few or zero frames after initial render

**Method 3: FPS meter**

1. DevTools > Rendering tab > Frame Rendering Stats (FPS meter)
2. Shows real-time frame rate
3. Idle page should show 0 fps (no frames being produced)
4. Any non-zero reading at idle = something is forcing compositor frames

### What to Look For at Idle

- `NeedsBeginFrameChanged(true)`: Something activated the compositor. Search for the
  preceding event to find what requested it.
- `SetNeedsRedraw` without preceding JS: A CSS animation or compositor effect is causing
  damage without any main-thread involvement.
- `BeginMainFrame` at idle: The main thread is being asked to produce content. This means
  something (timer, RAF, event) is triggering main-thread rendering work.
- Continuous `DrawAndSwap` at idle: The viz display compositor is drawing frames. This means
  actual pixels are being composited every vsync. Investigate what layer has damage.

---

## Summary: What Causes Redraws at Idle (No JS Running)

### Causes (Confirmed)

1. **CSS `animation: infinite`**: Any infinite keyframe animation keeps the compositor producing
   frames, even if the element is invisible (`opacity: 0`). Measured at ~10% CPU in Chrome.
2. **CSS cursor blink via `@keyframes infinite`**: The cursor blink animation produces
   compositor frames every cycle.
3. **Active CSS transition**: During the interpolation period, the compositor produces frames.
   After completion, stops.
4. **`<video>` element playing**: Hardware-decoded video produces compositor frames for each
   video frame.
5. **Composited `<canvas>` with animation loop**: WebGL or 2D canvas with RAF loop.
6. **Scrolling (during scroll)**: Compositor-thread scroll produces frames during scroll input.

### Non-Causes (Confirmed)

1. **Static `will-change`**: Layer promotion without animation causes zero continuous frames.
2. **`transition: all` declaration**: Inert until property changes. Zero idle cost.
3. **`:hover` styles**: Only trigger on mouse movement, not at idle.
4. **Static CSS custom properties**: Resolved once, cached. No per-frame cost.
5. **`position: fixed/sticky`**: Layer promoted but zero idle frames (only during scroll).
6. **`overflow: auto` (not scrolling)**: Layer exists but compositor idle.
7. **RAF registration without visual work**: Zero compositor overhead.
8. **React event delegation**: Passive listeners cause zero rendering work.
9. **React reconciliation with no DOM diff**: Zero rendering pipeline activity.
10. **`z-index` stacking (static)**: No continuous compositing (but may cause implicit layer
    promotion increasing memory).

### Recommendations for Zero-Idle-Redraw Pages

1. **Never use `animation: infinite`** unless the element is visible and the animation serves
   a purpose. Remove animated elements from DOM when hidden.
2. **Use `content-visibility: auto`** for off-screen content (scrollback buffers, chat history).
3. **Use `contain: content`** on independent UI sections to scope rendering work.
4. **Prefer CSS `transition` over `animation`** for state changes -- transitions naturally stop.
5. **Stop cursor blink after timeout** (xterm.js does this: 5 minutes; DomTerm: 30 seconds).
6. **Audit layers** with DevTools Layers panel. Minimize implicit compositing.
7. **Use `requestIdleCallback`** for non-urgent work instead of `setTimeout`/`setInterval`.
8. **Specify transition properties explicitly** (`transition: opacity 0.3s`) instead of
   `transition: all`.
9. **Measure idle frames** with `chrome://tracing` using `cc.debug.scheduler` category.

---

## Sources

- [RenderingNG Architecture (Chrome for Developers)](https://developer.chrome.com/docs/chromium/renderingng-architecture)
- [How cc Works (Chromium Docs)](https://chromium.googlesource.com/chromium/src/+/master/docs/how_cc_works.md)
- [Life of a Frame (Chromium Docs)](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/life_of_a_frame.md)
- [Rendering Performance (web.dev)](https://web.dev/articles/rendering-performance)
- [Stick to Compositor-Only Properties (web.dev)](https://web.dev/articles/stick-to-compositor-only-properties-and-manage-layer-count)
- [Accelerated Rendering in Chrome (web.dev)](https://web.dev/articles/speed-layers)
- [GPU Accelerated Compositing in Chrome (chromium.org)](https://www.chromium.org/developers/design-documents/gpu-accelerated-compositing-in-chrome/)
- [Compositor Thread Architecture (chromium.org)](https://www.chromium.org/developers/design-documents/compositor-thread-architecture/)
- [content-visibility: the new CSS property (web.dev)](https://web.dev/articles/content-visibility)
- [Invisible CSS Animations CPU Cost (mortoray.com)](https://mortoray.com/highly-inefficient-invisible-animations-css-firefox-chrome-react/)
- [What Forces Layout/Reflow (Paul Irish)](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)
- [Recording Tracing Runs (chromium.org)](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/recording-tracing-runs/)
- [BlinkNG Deep Dive (Chrome for Developers)](https://developer.chrome.com/docs/chromium/blinkng)
- [CSS Containment (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Containment/Using)
- [Improving CSS Custom Properties Performance (Igalia)](https://blogs.igalia.com/jfernandez/2020/08/13/improving-css-custom-properties-performance/)
- [React 17 Event Delegation (Saeloun Blog)](https://blog.saeloun.com/2021/07/08/react-17-event-delagation/)
