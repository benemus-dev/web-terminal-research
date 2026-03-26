# Chrome Actual Redraw Causes: What REALLY Triggers Rendering

Research date: 2026-03-26
Sources: Chromium source code, design docs, chromium.org, Chrome DevTools blog, web.dev

---

## 1. Chrome's BeginFrame Scheduling

### What triggers Chrome to send a BeginFrame signal?

BeginFrame is the vsync-aligned signal that flows from the **display compositor (viz)** through the **renderer compositor (cc)** to the **main thread (Blink)**. The pipeline is:

```
Display/OS vsync → viz BeginFrameSource → cc::Scheduler → Blink main thread
```

The cc::Scheduler subscribes/unsubscribes to BeginFrame via `StartOrStopBeginFrames()`:

```cpp
void Scheduler::StartOrStopBeginFrames() {
    bool needs_begin_frames = state_machine_->ShouldSubscribeToBeginFrames();
    if (needs_begin_frames == observing_begin_frame_source_) return;

    if (needs_begin_frames) {
        observing_begin_frame_source_ = true;
        begin_frame_source_->AddObserver(this);
    } else {
        observing_begin_frame_source_ = false;
        begin_frame_source_->RemoveObserver(this);
        compositor_timing_history_->BeginImplFrameNotExpectedSoon();
    }
}
```

### Does Chrome send BeginFrame continuously or on demand?

**On demand.** Chrome uses a demand-driven model. The observer calls `SetNeedsBeginFrames(true)` on the source to indicate it is active, and `SetNeedsBeginFrames(false)` when idle. The source only triggers BeginFrame while observers are subscribed.

When a page goes idle for a long period, the compositor sends `BeginMainFrameNotExpectedSoon` — a signal that explicitly tells the main thread "no frames are coming, do idle work."

### What is cc::Scheduler and when does it decide to produce a frame?

`cc::Scheduler` is the compositor-thread component that manages frame production timing. It delegates state decisions to `SchedulerStateMachine`. The scheduler:
- Receives BeginFrame signals from viz
- Decides whether to send `BeginMainFrame` to the main thread
- Manages deadlines for main thread responses
- Falls back to compositor-only draws if the main thread is slow

### ShouldSubscribeToBeginFrames — the gatekeeper

This is the critical method that determines whether Chrome is actively receiving vsync ticks:

```cpp
bool SchedulerStateMachine::ShouldSubscribeToBeginFrames() const {
    if (!HasInitializedLayerTreeFrameSink())
        return false;
    if (!visible_)
        return false;
    return BeginFrameRequiredForAction() ||
           BeginFrameNeededForVideo() ||
           ProactiveBeginFrameWanted();
}
```

Three conditions can keep BeginFrame subscribed:
1. **BeginFrameRequiredForAction()** — actual work pending
2. **BeginFrameNeededForVideo()** — video element playing
3. **ProactiveBeginFrameWanted()** — speculative/latency-hiding

### BeginFrameRequiredForAction — when actual work exists

```cpp
bool SchedulerStateMachine::BeginFrameRequiredForAction() const {
    if (forced_redraw_state_ == ForcedRedrawOnTimeoutState::WAITING_FOR_DRAW)
        return true;
    return needs_redraw_ ||
           needs_one_begin_impl_frame_ ||
           (needs_begin_main_frame_ && !defer_begin_main_frame_) ||
           needs_impl_side_invalidation_;
}
```

Each flag:
- `needs_redraw_` — set by `SetNeedsRedraw()`, tree activation, checkerboard animations
- `needs_one_begin_impl_frame_` — set by `SetNeedsOneBeginImplFrame()`, used for scrollbar fade animations
- `needs_begin_main_frame_` — set when Blink calls `SetNeedsAnimate`, `SetNeedsUpdate`, or `SetNeedsCommit`
- `needs_impl_side_invalidation_` — impl-side tile invalidation

### ProactiveBeginFrameWanted — the latency hider (THIS is why "idle" pages still get frames)

```cpp
bool SchedulerStateMachine::ProactiveBeginFrameWanted() const {
    if (!visible_) return false;

    // Commit pending — want to draw quickly after commit
    if ((begin_main_frame_state_ != BeginMainFrameState::IDLE) &&
        !defer_begin_main_frame_)
        return true;

    // Pending tree exists — want to draw after activation
    if (has_pending_tree_) return true;

    // Tile preparation needed — may enable activation
    if (needs_prepare_tiles_) return true;

    // Just drew — likely to draw again soon (prevents SetNeedsBeginFrame jitter)
    if (did_attempt_draw_in_last_frame_) return true;

    // Last commit was empty — but another might come
    if (last_commit_had_no_updates_) return true;

    // Active scrolling/pinching — keep frames flowing
    if (tree_priority_ == SMOOTHNESS_TAKES_PRIORITY) return true;

    return false;
}
```

**Key insight**: The `did_attempt_draw_in_last_frame_` and `last_commit_had_no_updates_` conditions mean Chrome keeps requesting frames for **at least one extra frame** after any draw or empty commit. This prevents latency spikes from toggling BeginFrame subscription on/off too aggressively.

### Can Chrome truly stop BeginFrame for a visible page?

**YES**, but only when ALL of these are false:
- No `needs_redraw_`
- No `needs_one_begin_impl_frame_`
- No `needs_begin_main_frame_`
- No `needs_impl_side_invalidation_`
- No video playing
- No pending tree
- No pending tiles
- No recent draw attempt
- No recent empty commit
- No active scroll/pinch interaction
- Not in forced redraw state

In practice, a truly static page with no input elements, no animations, no video, and no user interaction **can** reach zero BeginFrames. But many common page features prevent this.

---

## 2. Sources of Redraws That Application Code CANNOT Control

### 2.1 Caret (cursor) blink in `<input>`, `<textarea>`, `contenteditable`

**This is the single biggest source of idle redraws on pages with input elements.**

From Chromium source (`third_party/blink/renderer/core/editing/frame_caret.cc`):

```cpp
void FrameCaret::CaretBlinkTimerFired(TimerBase*) {
    SetVisibleIfActive(!IsVisibleIfActive());
    ScheduleVisualUpdateForPaintInvalidationIfNeeded();
}
```

The caret blink:
1. Uses a repeating timer at the OS-configured blink interval (typically 500ms = 1Hz toggle = 2 state changes/sec)
2. Each toggle calls `SetVisibleIfActive()` which changes an opacity effect node
3. **Optimization path**: If `DirectlyUpdateCompositedOpacityValue()` succeeds, the update goes directly to the compositor without main-thread paint invalidation — but it STILL produces a compositor frame
4. **Fallback path**: If direct update fails, it triggers `SetNeedsNonCompositedPaintInvalidation()` AND `SetPaintArtifactCompositorNeedsUpdate()` — full main-thread involvement

The opacity is set to **0.001f** (not 0.0f) when "hidden" to ensure the compositor keeps rendering quads for the caret layer. This means even the "hidden" state produces rendering work.

**Bottom line**: A focused `<input>` or `<textarea>` guarantees at least 2 compositor frames per second from caret blink alone, even on an otherwise completely static page.

Mozilla's Firefox bug [1724405](https://bugzilla.mozilla.org/show_bug.cgi?id=1724405) documented identical behavior: "The caret keeps blinking forever, and each time the caret visibility changes, there's activity in the main thread, the Renderer thread and the Compositor thread." Firefox's fix was `ui.caretBlinkCount` — stop blinking after N cycles.

**Chromium has a TODO in frame_caret.h**: "Consider moving the timer into the compositor thread" — acknowledging this is architecturally suboptimal.

### 2.2 Scrollbar hover/fade animations

Scrollbars in Chrome exist in multiple forms:
- **Desktop painted scrollbars**: Theme code is not thread-safe, so thumb/track are painted and rastered into bitmaps on the main thread, then emitted as quads on the compositor thread
- **Aura scrollbar fade**: Animates layer opacity via compositor (uses `SetNeedsOneBeginImplFrame`)
- **Overlay scrollbars (macOS)**: Fade in/out animations on scroll stop

Any scrollbar state change (hover on/off, thumb position, fade animation) triggers paint invalidation. Even **hovering over a scrollbar** causes a repaint because the scrollbar thumb changes visual state (highlighted vs normal).

For non-composited scrollbars, invalidation affects the **entire PaintLayer**, not just the scrollbar area.

### 2.3 Mouse movement and compositor thread

From Chrome's [Inside look at modern web browser (part 4)](https://developer.chrome.com/blog/inside-browser-part4):

> If no input event listeners are attached to the page, the compositor thread can create a new composite frame completely independent of the main thread.

Mouse movement CAN cause compositor frames even without any JavaScript handlers because:
- Chrome tracks "Non-Fast Scrollable Regions" — areas with event handlers
- Mouse moves outside these regions are handled entirely on the compositor thread
- The compositor may still produce frames for cursor type changes (arrow → pointer → text)
- Cursor changes are handled at the browser/OS level but Chrome's compositor may still need to update hit-test regions

### 2.4 Chrome extensions and content scripts

Extensions with content scripts inject into the page's DOM and can:
- Add MutationObservers (don't directly cause redraws, but callbacks may modify DOM)
- Inject CSS that triggers layout/style recalculation
- Add event listeners that create "Non-Fast Scrollable Regions"
- Run periodic timers that touch the DOM
- Inject iframes or overlays that have their own compositor surfaces

Even if an extension does nothing visible, its **presence** as an injected script world can affect Chrome's scheduling.

### 2.5 DevTools attachment

Opening DevTools changes rendering behavior:
- The DevTools overlay domain (`Overlay` CDP domain) can draw highlight overlays
- "Frame Rendering Stats" overlay runs its own animation loop
- Paint flashing, layout shift regions, layer borders — all generate compositor frames
- Even with no overlays enabled, the DevTools inspector generates additional IPC that can trigger frame scheduling

### 2.6 GPU process and display compositor (viz)

The display compositor (viz) runs in the GPU process and has its own frame scheduling independent of content:
- **Surface aggregation**: Combines compositor frames from all sources (browser UI, renderer, video)
- **Pending surfaces**: If any surface submits a new frame, viz may trigger a draw
- The browser UI chrome (tabs, omnibox, bookmarks bar) is a separate compositor surface — UI updates there cause viz to composite even if page content hasn't changed

### 2.7 Speculative frame production

`ProactiveBeginFrameWanted()` keeps frames flowing after recent activity to hide latency. The `did_attempt_draw_in_last_frame_` flag means Chrome requests **at least one more frame** after any draw, creating a "tail" of unnecessary frames after genuine updates stop.

### 2.8 IntersectionObserver processing

IntersectionObserver callbacks run **after** rAF callbacks and layout in the rendering pipeline. While they use `requestIdleCallback` for notification delivery, the **intersection computation** itself happens as part of the frame lifecycle. Pages with IntersectionObservers may keep the frame pipeline slightly busier because the observer's threshold checks run during frame processing.

---

## 3. What the Freeze Script Catches vs Misses

### The script:
```javascript
// Intercepts requestAnimationFrame and holds callbacks
// Injects CSS: animation: none !important; transition: none !important; animation-play-state: paused !important
```

### What it DOES stop:
- JS animation loops using rAF
- CSS `animation` property animations
- CSS `transition` animations
- JS-driven style changes that would cause rAF-driven repaints
- Any library animation framework (GSAP, anime.js, Framer Motion, etc.)

### What it CANNOT stop:

| Source | Why it's unstoppable |
|--------|---------------------|
| Caret blink | Browser-internal timer in `FrameCaret`, not CSS animation. Uses `EffectPaintPropertyNode` opacity, not CSS `animation` |
| Scrollbar hover/fade | Browser-internal compositor animation, not CSS |
| Mouse movement compositor frames | Compositor thread responds to input events independently |
| `will-change` layers | Compositor maintains composited layers regardless of CSS animation state |
| Video elements | `BeginFrameNeededForVideo()` — dedicated path in scheduler |
| Chrome extension injected content | Runs in separate script world, not intercepted by page-level rAF override |
| Browser UI overlaps (omnibox dropdown, permission prompts) | Separate compositor surface, no page control |
| `<canvas>` with active WebGL context | GPU process may maintain swap chain |
| Composited scrolling | Compositor handles scroll independently of main thread |
| Favicon animation | Browser process, not renderer |
| `requestIdleCallback` | Different from rAF, not intercepted by rAF freeze |
| Autofill/autocomplete popups | Browser UI, separate surface |
| Selection highlight | Browser-internal paint, changes with mouse drag |
| `contenteditable` spell-check underlines | Browser-internal rendering |
| Print preview / PDF viewer | Separate renderer process |

---

## 4. What Causes Redraws on a TRULY Static Page

### Test case: `<p>Hello</p>` — nothing else

**Expected behavior**: Chrome SHOULD reach zero BeginFrames after initial paint.

After the page loads and initial paint completes:
1. `needs_redraw_` clears after first draw
2. `needs_begin_main_frame_` clears after commit
3. `did_attempt_draw_in_last_frame_` stays true for ONE extra frame
4. `last_commit_had_no_updates_` may keep one more frame
5. Then `ShouldSubscribeToBeginFrames()` returns false
6. Scheduler calls `RemoveObserver(this)` — **zero BeginFrames**

**But in practice**: Even this minimal page may still see occasional frames from:
- Browser UI updates (tab title, favicon loading)
- Garbage collection triggering layout
- Chrome's own internal timers (memory pressure checks, etc.)
- Display change detection (if monitor goes to sleep/wake)

### With `<textarea>` (focused)

**Continuous redraws at 2x caret blink rate** (typically ~2 fps from caret alone). The `FrameCaret` timer fires every ~500ms, toggling opacity, triggering `ScheduleVisualUpdateForPaintInvalidationIfNeeded()` which sets `needs_redraw_` or directly updates compositor, keeping BeginFrame subscribed indefinitely.

### With `overflow: auto` content

If content overflows and scrollbar is visible:
- Static scrollbar: no extra redraws (after initial paint)
- Hovering over scrollbar: triggers paint invalidation for thumb highlight state
- On platforms with overlay scrollbars (macOS): fade-out animation after last scroll generates frames until fully faded

### With `position: fixed` element

A `position: fixed` element gets its own composited layer. This alone doesn't cause redraws, but:
- The compositor maintains the layer in its tree
- Any scroll event requires the compositor to handle the fixed positioning
- The layer's existence makes `has_pending_tree_` more likely to be true during commits

---

## 5. Chrome Flags That Actually Reduce Idle Compositor Activity

### `--disable-gpu-vsync`
**What it does**: Decouples frame production from monitor refresh rate.
**Effect on idle redraws**: INCREASES frame production — removes the natural throttle. Frames are produced as fast as possible when any work is pending. **Counterproductive** for reducing idle redraws.

### `--disable-frame-rate-limit`
**What it does**: Similar to disable-gpu-vsync, removes frame rate cap.
**Effect on idle redraws**: Same as above — makes things WORSE, not better.

### `--disable-gpu-compositing`
**What it does**: Falls back to software compositing on the main thread.
**Effect on idle redraws**: Eliminates compositor-thread frame production but moves ALL rendering to main thread. May reduce idle compositor frames but **increases main thread overhead** and causes visual artifacts. NOT recommended.

### `--enable-begin-frame-scheduling`
**Status**: **Removed.** This was turned on by default around 2016 (Chromium code review 1939253002) and the flag was subsequently deleted. Unified BeginFrame scheduling is now the only mode.

### `--disable-threaded-compositing`
**What it does**: Runs compositor on the main thread instead of a separate thread.
**Effect on idle redraws**: Removes the compositor thread's independent frame scheduling. May reduce some speculative frames but makes ALL rendering synchronous — catastrophic for interactive performance. Scrolling, pinch-zoom, and CSS transforms all stall on main thread work.

### `chrome://flags/#enable-gpu-rasterization`
**What it does**: Rasterizes paint commands on the GPU instead of CPU.
**Effect on idle redraws**: No effect on idle frame scheduling. Only changes WHERE rasterization happens, not WHEN frames are produced.

### Verdict
**None of these flags help with idle redraws in any practical way.** The flags that reduce compositor activity do so at severe cost to interactive performance. The demand-driven BeginFrame scheduling is the correct architecture — the problem is the things that DEMAND frames, not the scheduling itself.

---

## 6. requestAnimationFrame vs requestIdleCallback

### Does registering a rAF callback cause Chrome to schedule a BeginFrame?

**YES.** Calling `requestAnimationFrame(callback)` causes Blink to call `SetNeedsAnimate()` on `LayerTreeHost`, which sets `needs_begin_main_frame_ = true` in the scheduler state machine, which makes `BeginFrameRequiredForAction()` return true, which makes `ShouldSubscribeToBeginFrames()` return true. One rAF registration = one BeginFrame subscription.

From [How cc Works](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/how_cc_works.md):
> Most webpages request [a frame] via requestAnimationFrame, which eventually calls SetNeedsAnimate on LayerTreeHost.

### Does registering requestIdleCallback cause a BeginFrame?

**NO.** `requestIdleCallback` is scheduled during idle periods and explicitly does NOT trigger frame production. It runs when the browser determines there is idle time BETWEEN frames or when no frames are being produced. It does not set any `needs_*` flags.

### What if rAF is registered but the callback does nothing visible?

The frame is STILL produced. Chrome cannot know in advance that the callback will be a no-op. The full pipeline runs:
1. BeginMainFrame sent to main thread
2. rAF callback runs (does nothing)
3. Style recalc, layout, paint run (find no changes)
4. Commit may be **aborted early** if cc detects no updates
5. `last_commit_had_no_updates_ = true` — keeps ONE more proactive frame

So a no-op rAF still causes at least **2 frames** (one for the callback, one proactive), even if the callback changes nothing. If the rAF re-registers itself (common pattern), it's continuous frames forever.

---

## 7. Chrome's Internal Timers That Cause Rendering Work

### Caret blink timer
- **Interval**: OS-configured (Windows: `GetCaretBlinkTime()`, typically 500ms)
- **Effect**: Toggles opacity on `EffectPaintPropertyNode`, triggers `ScheduleVisualUpdateForPaintInvalidationIfNeeded()`
- **When active**: Any time an `<input>`, `<textarea>`, or `contenteditable` element is focused
- **When stopped**: Focus lost, mouse button held down (suspended), page hidden

### Scrollbar fade timer (overlay scrollbars)
- **Platforms**: macOS (overlay scrollbars), Aura (Chromium's UI framework on Linux/Windows)
- **Effect**: Animates scrollbar opacity from 1.0 to 0.0 over ~300-800ms after last scroll
- **Mechanism**: Uses `SetNeedsOneBeginImplFrame` to request compositor frames for animation
- **When active**: After any scroll event, until fade completes

### Selection highlight
- No timer, but **any mouse movement during selection** (drag to select text) causes continuous paint invalidation of the selection highlight region

### Autofill/autocomplete UI
- Browser-internal popup, rendered as a separate compositor surface
- Appearance/disappearance causes viz to recomposite
- Keyboard navigation within the popup triggers redraws

### Composited scrolling momentum (fling)
- After a touch/trackpad fling, the compositor animates scroll position
- `tree_priority_ == SMOOTHNESS_TAKES_PRIORITY` keeps `ProactiveBeginFrameWanted()` true
- Continues until fling velocity reaches zero (can be several seconds)

### Smooth scroll animation
- `scroll-behavior: smooth` or `Element.scrollIntoView({behavior: 'smooth'})`
- Generates compositor animation frames until scroll destination reached

---

## 8. How to ACTUALLY Achieve Zero Redraws

### Is it possible?

**Theoretically yes, practically very difficult.** Chrome's architecture supports zero-frame idle states, but many common page features prevent reaching it.

### Requirements for true zero redraws:

1. **No focused input elements** — eliminates caret blink
2. **No CSS animations or transitions** — (the freeze script handles this)
3. **No JavaScript timers modifying DOM** — no `setInterval`/`setTimeout` touching visible state
4. **No rAF registration** — even no-op rAF keeps frames flowing
5. **No video or audio elements** — `BeginFrameNeededForVideo()`
6. **No active scrollbar animations** — wait for scrollbar fade to complete
7. **No Chrome extensions with content scripts** — or at least none that poll/modify DOM
8. **No IntersectionObserver with frequent threshold checks**
9. **No `will-change` on elements** — unnecessary compositor layer maintenance
10. **No pending image loads** — decoded image display triggers paint
11. **Mouse not moving over the page** — prevents compositor hit-test updates
12. **DevTools not attached**
13. **No `contenteditable` regions**
14. **No `<canvas>` elements with active context**
15. **No Web Workers posting to main thread**
16. **No Service Worker fetch events**
17. **No pending font loads** — font swap triggers repaint
18. **No `resize` observer registered** (layout checks on frame)

### What about `document.hidden` (background tab)?

**YES — background tabs achieve near-zero rendering.** Chrome stops `requestAnimationFrame` callbacks entirely in background tabs (since 2011). Additionally:
- `setInterval`/`setTimeout` throttled to max 1/second (since Chrome 57), further throttled to 1/minute for chained timers after 5 minutes (since Chrome 88)
- BeginFrame signals stop being sent to hidden renderers
- Rendering is fully paused — no compositor frames produced
- Audio-playing tabs are exempted from throttling
- WebSocket/WebRTC tabs are exempted from throttling

**Background tab is the ONLY reliable way to achieve zero redraws in Chrome.**

### Minimal page that achieves zero redraws (foreground):

```html
<!DOCTYPE html>
<html>
<head><style>body { margin: 0; overflow: hidden; }</style></head>
<body><p>Hello</p></body>
</html>
```

With:
- `overflow: hidden` — no scrollbars
- No input elements — no caret
- No JavaScript — no rAF/timers
- No CSS animations
- No extensions loaded
- Mouse stationary (or moved off the window)

This page SHOULD reach zero BeginFrames after initial paint + the proactive tail (1-2 extra frames).

---

## 9. Measuring Actual Redraws

### chrome://tracing (most authoritative)

1. Open `chrome://tracing`
2. Click **Record** → **Manually select settings**
3. Select categories: `cc`, `viz`, `gpu`, `benchmark`
4. Record for 5-10 seconds on the target page
5. Stop recording

**Key trace events to search for:**
- `Scheduler::BeginFrame` or `BeginImplFrame` — each is one vsync tick received by cc
- `Scheduler::BeginMainFrame` — sent to main thread (main thread damage exists)
- `DrawFrame` — actual draw to screen
- `SubmitCompositorFrame` — frame submitted from renderer to viz
- `DidNotProduceFrame` — renderer explicitly declined to produce a frame
- `Display::DrawAndSwap` — viz drew to screen
- `FrameSequenceTracker` — tracks named sequences (TouchScroll, RAF, CompositorAnimation)

**Counting frames per second:**
Count `DrawFrame` events in a 1-second window. On a truly idle page, this should be 0. On a page with a blinking caret, expect ~2/sec.

### DevTools Performance panel

- Record a performance trace
- Look at the **Compositor** track
- White/empty frames = "No changes" (idle)
- Green frames = rendered successfully
- Look at the **Frames** section for frame rate

### Frame Rendering Stats overlay

DevTools → Rendering tab → check "Frame rendering stats"
Shows real-time FPS counter. On idle pages, FPS should drop to 0 (though the overlay itself generates frames while visible — observer effect).

### JavaScript-side measurement

```javascript
let frameCount = 0;
function countFrames() {
    frameCount++;
    requestAnimationFrame(countFrames);
}
requestAnimationFrame(countFrames);
setInterval(() => {
    console.log(`Frames/sec: ${frameCount}`);
    frameCount = 0;
}, 1000);
```

**WARNING**: This measurement itself causes continuous BeginFrame because it registers rAF recursively. It measures "frames when rAF is active" not "frames when idle." Use chrome://tracing for true idle measurement.

### Performance Observer API

```javascript
const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
        console.log('Long animation frame:', entry.duration);
    }
});
observer.observe({ type: 'long-animation-frame', buffered: true });
```

This catches frames > 50ms but does NOT report normal-duration frames. Not useful for counting total frames.

---

## Summary: Why "Idle" Pages Still Redraw

The most common culprits, in order of likelihood:

1. **Caret blink** in any focused input element (~2 fps continuous)
2. **rAF loop** from page JavaScript or libraries (60 fps continuous)
3. **CSS animations/transitions** from the page or injected stylesheets
4. **Chrome extensions** modifying DOM or running timers
5. **Scrollbar fade animation** after recent scroll (~0.5-1 sec burst)
6. **ProactiveBeginFrameWanted** tail after any activity (1-2 extra frames)
7. **DevTools being open** (observer effect)
8. **Video elements** (even paused ones may maintain frame subscription)
9. **Mouse movement** triggering compositor hit-test updates
10. **Favicon/tab animations** (loading spinner, etc.)

The freeze script addresses #2 and #3 but cannot touch the others, which is why it "makes it a bit better" but doesn't eliminate redraws entirely.

---

## Sources

- [How cc Works — Chromium Docs](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/how_cc_works.md)
- [Life of a Frame — Chromium Docs](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/life_of_a_frame.md)
- [scheduler_state_machine.h — Chromium Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/cc/scheduler/scheduler_state_machine.h)
- [scheduler_state_machine.cc — Chromium Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/cc/scheduler/scheduler_state_machine.cc)
- [scheduler.cc — Chromium Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/cc/scheduler/scheduler.cc)
- [frame_caret.cc — Chromium Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/editing/frame_caret.cc)
- [frame_caret.h — Chromium Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/editing/frame_caret.h)
- [Chrome Compositor and Display Scheduling (2015 slides)](https://docs.google.com/presentation/d/1FpTy5DpIGKt8r2t785y6yrHETkg8v7JfJ26zUxaNDUg/htmlpresent)
- [Inside look at modern web browser (part 4)](https://developer.chrome.com/blog/inside-browser-part4)
- [Aligned input events — Chrome DevTools Blog](https://developer.chrome.com/blog/aligning-input-events)
- [requestAnimationFrame Scheduling For Nerds — Paul Irish](https://medium.com/@paul_irish/requestanimationframe-scheduling-for-nerds-9c57f7438ef4)
- [Background tabs in Chrome 57 — Chrome Blog](https://developer.chrome.com/blog/background_tabs)
- [Heavy throttling of chained JS timers — Chrome 88](https://developer.chrome.com/blog/timer-throttling-in-chrome-88)
- [Compositor Thread Architecture — chromium.org](https://www.chromium.org/developers/design-documents/compositor-thread-architecture/)
- [GPU Accelerated Compositing in Chrome — chromium.org](https://www.chromium.org/developers/design-documents/gpu-accelerated-compositing-in-chrome/)
- [RenderingNG Architecture — Chrome for Developers](https://developer.chrome.com/docs/chromium/renderingng-architecture)
- [RenderingNG deep-dive: BlinkNG — Chrome for Developers](https://developer.chrome.com/docs/chromium/blinkng)
- [Blink paint README — Chromium Source](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/core/paint/README.md)
- [display_scheduler.h — Chromium Source](https://chromium.googlesource.com/chromium/src/+/55dd5deaa0aa4d1d5878c246d51456a44bd73924/components/viz/service/display/display_scheduler.h)
- [Recording Tracing Runs — chromium.org](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/recording-tracing-runs/)
- [Chrome Frame Viewer How-To — chromium.org](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/using-frameviewer/)
- [Understanding about:tracing results — chromium.org](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/trace-event-reading/)
- [Towards an animation smoothness metric — web.dev](https://web.dev/articles/smoothness)
- [Long Animation Frames API — Chrome for Developers](https://developer.chrome.com/docs/web-platform/long-animation-frames)
- [Firefox Bug 1724405 — Caret blinking wastes CPU/power](https://bugzilla.mozilla.org/show_bug.cgi?id=1724405)
- [Unified BeginFrame scheduling code review (1939253002)](https://codereview.chromium.org/1939253002)
- [Remove non-unified begin frame scheduling (2296083002)](https://codereview.chromium.org/2296083002)
- [What forces layout/reflow — Paul Irish](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)
- [Page Visibility API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
