# Chrome Rendering Pipeline Audit: CUI

## Architecture Summary
- **Framework:** React 18 (full virtual DOM)
- **Transport:** SSE (Server-Sent Events) via fetch ReadableStream for streaming; REST API for history
- **Components:** React SPA with React Router, ~40+ web source files
- **No terminal:** Pure chat UI (no xterm.js)
- **Source:** `/c/dev/web-terminal-research/cui/src/web/` (client), `/c/dev/web-terminal-research/cui/src/` (server)

---

## 1. Forced Synchronous Layout

### Hits Found (7 unique locations)

| Location | Property | Context | Severity |
|----------|----------|---------|----------|
| `main.tsx:25` | `getComputedStyle` | Theme color meta tag update | LOW |
| `Composer.tsx:752` | `innerHeight` | Max textarea height calc | MEDIUM |
| `Composer.tsx:753` | `scrollHeight` | Textarea auto-resize | HIGH |
| `Dialog.tsx:22` | `innerWidth` | Mobile breakpoint check | LOW |
| `MessageList.tsx:30` | `scrollTop = scrollHeight` | Auto-scroll to bottom | MEDIUM |
| `WaveformVisualizer.tsx:44` | `getBoundingClientRect` | Canvas initialization | LOW |
| `TaskTool.tsx:31` | `scrollHeight` | Content scroll height read | LOW |

### Critical Forced Reflow Patterns

**Pattern 1: Textarea auto-resize (fires on every input change)**
`Composer.tsx:748-755`:
```
const adjustTextareaHeight = () => {
    textarea.style.height = 'auto';                                    // DOM WRITE
    const maxHeight = Math.floor(window.innerHeight * 0.8);            // LAYOUT READ
    textarea.style.height = `${Math.min(textarea.scrollHeight, maxHeight)}px`; // LAYOUT READ + DOM WRITE
};
```
Classic write-read forced reflow. `style.height = 'auto'` invalidates layout, then `scrollHeight` forces synchronous layout. Runs on every keystroke via `useEffect(() => adjustTextareaHeight(), [value])` and also on window resize.

**Pattern 2: Auto-scroll on every message change**
`MessageList.tsx:28-32`:
```
useEffect(() => {
    containerRef.current.scrollTop = containerRef.current.scrollHeight;
}, [messages]);
```
This fires whenever the `messages` array reference changes. During streaming, if each stream event creates a new message or mutates the array, this fires on every update. The `scrollTop = scrollHeight` read forces layout computation after React's DOM updates.

### Theme MutationObserver
`main.tsx:34-42`:
```
const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
        if (mutation.attributeName === 'data-theme') {
            setTimeout(updateThemeColor, 0);  // calls getComputedStyle
        }
    });
});
observer.observe(document.documentElement, { attributes: true });
```
Persistent MutationObserver on `<html>` element watching attribute changes. Only triggers `getComputedStyle` on theme change (rare), but the observer callback fires on any attribute change to `<html>`.

---

## 2. React Reconciliation

### Always-Mounted Components
During active conversation view:
- `App` -> `ChatApp` -> `PreferencesProvider` -> `StreamStatusProvider` -> `ConversationsProvider`
- `Layout` (sidebar + content area)
- `ConversationView` (main content)
- `ConversationHeader`
- `MessageList` (with all `MessageItem` children)
- `Composer` (text input area)

**Estimated always-mounted components: 15-25** (smaller than claudecodeui).

### useLayoutEffect Usage
**None found.** CUI does not use `useLayoutEffect` anywhere.

### useEffect Reading Layout Properties
- `Composer.tsx:758-760`: `useEffect(() => adjustTextareaHeight(), [value])` -- reads `scrollHeight` and `innerHeight` after writing `style.height`. Fires per keystroke.
- `MessageList.tsx:28-32`: `useEffect(() => { scrollTop = scrollHeight }, [messages])` -- reads `scrollHeight`. Fires per message update.

### Streaming Re-renders

**SSE stream processing:**
`useStreaming.ts:90-121`:
```
while (true) {
    const { done, value } = await reader.read();
    // ... parse JSON lines ...
    const event = JSON.parse(jsonLine);
    optionsRef.current.onMessage(event);  // calls handleStreamMessage
}
```

Each SSE line is parsed and dispatched immediately via `onMessage`. There is **no buffering/debounce** of stream events.

`useConversationMessages.ts:69-204`:
`handleStreamMessage` calls `setMessages`, `setToolResults`, `setChildrenMessages` etc. on each stream event. React 18 batches these within the same microtask, but each `reader.read()` cycle is a separate microtask.

**Estimated re-renders during streaming:**
- If the LLM streams at ~50 tokens/sec and messages come as individual SSE lines: potentially **up to 50 re-renders/sec**
- However, SSE messages are chunked by the fetch ReadableStream. Chunks may contain multiple lines, reducing actual render count
- Realistic estimate: **15-30 re-renders/sec** during active streaming

Each re-render of `MessageList` triggers:
1. Full message array reconciliation (filter, group, map)
2. `useEffect([messages])` -> `scrollTop = scrollHeight` (auto-scroll)
3. Component reconciliation for all visible `MessageItem` components

### State Update Batching
React 18 automatic batching applies within each `reader.read()` callback. Multiple `setState` calls within one `handleStreamMessage` invocation are batched. But each `reader.read()` cycle is a new microtask, so each chunk triggers a separate render.

---

## 3. DOM Mutations During Idle

### No Cursor Blink
CUI is a chat UI with no terminal. The text input cursor blink is handled by the browser natively (CSS `caret-color`) -- no JS DOM mutations.

### CSS Animations (Loading Indicators)
`MessageList.tsx:98-100`:
```
<span className="animate-bounce [animation-delay:-0.32s]" />
<span className="animate-bounce [animation-delay:-0.16s]" />
<span className="animate-bounce" />
```
Loading dots use CSS `animate-bounce`. These are **CSS animations handled by the compositor** -- no JS involvement, no layout/paint, just composite. Only visible when `isLoading && displayMessages.length === 0`.

`MessageList.tsx:124`: `animate-pulse` for streaming indicator dot. Same -- CSS animation, compositor-only.

### Status Indicators
- `ConversationHeader`: Static between updates
- No polling intervals in chat hooks
- `useStreaming.ts:154-170`: Visibility change listener (only fires on tab focus/blur)

### WaveformVisualizer (Audio Recording)
`WaveformVisualizer.tsx:126-135`:
```
const animate = useCallback((currentTime: number) => {
    if (!isRecording) return;
    if (shouldUpdate) drawWaveform();  // canvas draw, not DOM
    animationFrameRef.current = requestAnimationFrame(animate);
});
```
Only active during audio recording. Uses `requestAnimationFrame` loop at 15fps (`frameRate = 15`). Draws to `<canvas>`, not DOM. **No DOM mutations, no layout/paint -- just canvas compositing.**

When not recording: RAF loop is stopped (`cancelAnimationFrame`). **0 activity.**

### Audio Recording Duration Timer
`useAudioRecording.ts:108`: `setInterval(() => ..., 1000)` -- updates recording duration counter. Only active during recording, not idle.

### MutationObserver
`main.tsx:34-42`: Watches `<html>` attributes. At idle, no attributes change -- observer callback never fires.

**Idle DOM mutations: 0/sec**

---

## 4. Scroll Event Handling

### No onScroll handler in MessageList
`MessageList.tsx` does NOT attach an `onScroll` handler. The container just has `overflow-y-auto` for native scrolling.

### Auto-scroll
`MessageList.tsx:28-32`:
```
useEffect(() => {
    containerRef.current.scrollTop = containerRef.current.scrollHeight;
}, [messages]);
```
This is a `useEffect` (not `onScroll`). It fires after React render when messages change. During streaming, this fires on every message update.

**No scroll event handler at all** -- CUI does not track scroll position, does not implement infinite scroll/pagination, and does not have an "isUserScrolledUp" guard. This means:
- Auto-scroll fires even if user has scrolled up (poor UX)
- No layout reads from scroll handlers (good for performance)

### scrollIntoView
Not used anywhere in CUI web source.

---

## 5. Resize Handling

### Composer textarea resize
`Composer.tsx:763-769`:
```
useEffect(() => {
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
}, []);
```
`handleResize` calls `adjustTextareaHeight()` which does the write-read forced reflow pattern. **1 forced reflow per resize.**

### Dialog mobile check
`Dialog.tsx:22-26`:
```
const checkMobile = () => setIsMobile(window.innerWidth <= 768);
window.addEventListener('resize', checkMobile);
```
Reads `window.innerWidth` and sets state. **1 layout read + potential re-render per resize.**

**Total layout reads per resize: ~3** (textarea scrollHeight + innerHeight + Dialog innerWidth)

---

## 6. WebSocket Message Processing

CUI uses **SSE (Server-Sent Events)** via fetch ReadableStream, not WebSocket.

`useStreaming.ts:82-121`:
```
const reader = response.body.getReader();
while (true) {
    const { done, value } = await reader.read();
    const decoded = decoder.decode(value, { stream: true });
    buffer += decoded;
    const lines = buffer.split('\n');
    for (const line of lines) {
        const event = JSON.parse(jsonLine);
        optionsRef.current.onMessage(event);  // -> handleStreamMessage
    }
}
```

Each `reader.read()` may return multiple SSE lines. All lines within one chunk are processed synchronously, and React 18 batches the resulting state updates. However, each `reader.read()` is a new microtask.

**No explicit batching/debounce** like claudecodeui's 100ms timer. Each SSE chunk triggers a React re-render.

`handleStreamMessage` in `useConversationMessages.ts` calls multiple setState functions per event:
- `setMessages` (new array spread)
- `setToolResults` (object spread)
- `setChildrenMessages` (object spread)

These are batched within one event handling, but the new array/object spreads mean React will detect changes and re-render.

**Each stream event triggers 1 React re-render** (batched setState within same handler). Multiple events in one SSE chunk are still processed sequentially with intermediate state updates, but React 18 should batch them within the same microtask.

---

## 7. Estimated Pipeline Runs/Second

### Idle State (conversation loaded, no streaming)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 0 | No DOM changes |
| Layout | 0 | No layout-triggering operations |
| Paint | 0 | Nothing to repaint |
| Composite | 0 | Nothing to composite |

CUI is **completely silent** at idle -- 0 pipeline activity.

### During Chat Streaming

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 15-30 | React re-renders from unbuffered SSE events |
| Layout | 15-30 | Auto-scroll (scrollTop=scrollHeight) per message update |
| Paint | 15-30 | Message content updates |
| Composite | 15-30 | Layer recomposition |

**Key difference from claudecodeui:** No 100ms stream buffer. Each SSE event triggers a render. At high token throughput, this could reach 30+ renders/sec.

### During Chat Input (typing)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~10-30 | Per-keystroke re-render |
| Layout | ~10-30 | Forced reflow: height='auto' -> scrollHeight |
| Paint | ~10-30 | Textarea content update |
| Composite | ~10-30 | Layer recomposition |

Same forced reflow pattern as claudecodeui for textarea auto-resize.

---

## Key Findings

1. **No stream buffering is the biggest performance gap.** claudecodeui buffers stream deltas for 100ms; CUI renders every SSE event immediately. At 50 tokens/sec, CUI could trigger 15-30 renders/sec vs claudecodeui's ~10.

2. **Auto-scroll fires unconditionally** on every `[messages]` change with no "isUserScrolledUp" guard. During streaming, this means `scrollTop = scrollHeight` fires on every re-render -- forced layout computation every time.

3. **Textarea auto-resize** has the same write-read forced reflow pattern as claudecodeui, firing on every keystroke.

4. **No useLayoutEffect** -- all effects are asynchronous. This is better than claudecodeui for avoiding paint-blocking, but means scroll position may visually jump.

5. **Idle performance is excellent** -- 0 pipeline activity, no timers, no polling, no cursor blink.

6. **Smaller component tree** (~15-25 always-mounted vs claudecodeui's ~30-50) means faster reconciliation per render.

7. **No infinite scroll/pagination** -- all messages are rendered at once. For long conversations, this means React reconciles the entire message list on every update. With 100+ messages, this becomes expensive.

8. **WaveformVisualizer** is well-implemented -- uses canvas (not DOM) at 15fps, only active during recording, RAF-driven.

9. **MutationObserver on `<html>`** is benign at runtime but adds a persistent observer that fires on any html attribute change.

10. **SSE via fetch ReadableStream** is a clean transport choice -- no Socket.IO overhead, no WebSocket framing. But the lack of client-side buffering before rendering is the main weakness.
