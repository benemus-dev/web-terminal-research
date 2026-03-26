# Rendering Pipeline Audit: claude-code-webui

Audited: `frontend/src/` and `backend/`
Date: 2026-03-26

---

## 1. React Re-renders at Idle

### State variables (useState)

**ChatPage.tsx** (root chat component):
- `projects` (ProjectInfo[]) -- set once on mount via fetch, stable at idle
- `isSettingsOpen` (boolean) -- only changes on user click
- `searchParams` -- from react-router, stable at idle

**useChatState.ts** -- 8 state variables:
- `messages` (AllMessage[]) -- changes per streaming chunk (see section 6)
- `input` (string) -- only on user typing
- `isLoading` (boolean) -- only on request start/end
- `currentSessionId`, `currentRequestId`, `hasShownInitMessage`, `hasReceivedInit`, `currentAssistantMessage` -- only during active streaming

**At idle, none of these change.** No polling, no timers that update state.

### Effects that fire at idle

**TimestampComponent.tsx** -- THIS IS THE ONLY IDLE ACTIVITY:
- When `mode === "relative"`, sets a `setInterval(updateTime, 60000)` per instance
- Each interval tick calls `setDisplayTime()` which triggers a re-render of that TimestampComponent
- **Currently**: mode defaults to `"absolute"`, so this interval is NOT active in normal chat messages
- If relative mode were used, each visible timestamp would independently fire every 60s, causing N independent re-renders (no batching via shared timer -- the TODO comment on line 30 acknowledges this)

**ChatPage.tsx** -- global keydown listener:
- `useEffect` on line 424 adds `document.addEventListener("keydown", ...)` for ESC abort
- Fires on every keypress globally, but handler early-exits if `!isLoading`
- This is a passive listener, does NOT cause re-renders (no state updates unless abort happens)

**SettingsContext.tsx** -- theme effect:
- `useEffect` on `[settings, isInitialized]` -- fires once on mount, then only on settings change
- Modifies `document.documentElement.classList` (add/remove "dark") -- this IS a DOM mutation that triggers Style Recalc on the entire tree, but only on explicit theme toggle

### Context providers

One context: `SettingsContext` (theme + enterBehavior). Value is `useMemo`-ized, so consumers only re-render when settings actually change. At idle, this is stable.

### Verdict: idle state is clean

No re-renders at idle unless relative timestamps are enabled (they are not by default).

---

## 2. Forced Synchronous Layout from React

### Layout property reads

**ChatInput.tsx lines 80-89** -- textarea auto-resize effect:
```
textarea.style.height = "auto";                    // WRITE (invalidates layout)
const computedStyle = getComputedStyle(textarea);   // READ (forces layout)
const scrollHeight = Math.min(textarea.scrollHeight, maxHeight);  // READ
textarea.style.height = `${scrollHeight}px`;        // WRITE
```
This is a **classic layout thrash pattern**: write -> read -> write. The `getComputedStyle()` after `style.height = "auto"` forces a synchronous layout computation. This fires on every keystroke (dependency: `[input]`).

**Severity**: Low in practice -- only one element measured, no loop, only on user typing. But it IS a forced sync layout on every input change.

### scrollIntoView

**ChatMessages.tsx lines 34-37**:
```
messagesEndRef.current.scrollIntoView({ behavior: "smooth" });
```
Called inside `useEffect(() => { scrollToBottom() }, [messages])`.

`scrollIntoView` with `behavior: "smooth"` triggers layout computation (browser must know current scroll position and target element position). The `smooth` variant starts a scroll animation managed by the browser, which causes continuous composite operations over ~300-500ms.

**Frequency**: fires on EVERY messages state change. During streaming, this means every chunk. See section 6.

### useLayoutEffect

Not used anywhere in the codebase. Good.

### Commented-out near-bottom check (ChatMessages.tsx lines 40-50)

The `isNearBottom()` function that reads `scrollTop`, `scrollHeight`, `clientHeight` is commented out. If re-enabled, it would add another forced layout read per scroll.

---

## 3. DOM Mutations from React Reconciliation

### Always-mounted components (chat view)

1. `ChatPage` -- outer container, header, buttons
2. `ChatMessages` -- message list container + all message components
3. `ChatInput` -- textarea + submit button + permission mode bar
4. `SettingsButton`, `HistoryButton` -- header icons

### Per-message components

Each message in the list is one of: `ChatMessageComponent`, `SystemMessageComponent`, `ToolMessageComponent`, `ToolResultMessageComponent`, `PlanMessageComponent`, `ThinkingMessageComponent`, `TodoMessageComponent`.

These wrap either `MessageContainer` (simple div wrapper) or `CollapsibleDetails` (with expand/collapse state).

### Key issue: `messages.map(renderMessage)` re-runs on every state change

The `messages` array is the dependency for the message list. React's reconciliation diffs the entire list. Keys are `${message.timestamp}-${index}` which is stable for existing messages, so React should correctly identify that only the last message changed (or a new one was appended).

**However**: `updateLastMessage` in `useChatState.ts` line 49-56 creates a NEW array via `.map()`:
```
setMessages((prev) => prev.map((msg, index) =>
    index === prev.length - 1 && msg.type === "chat"
      ? { ...msg, content }
      : msg,
));
```
This creates a new array reference AND a new object for the last message on every streaming text chunk. React must:
1. Re-render `ChatMessages` (new `messages` prop reference)
2. Diff all message keys (O(n) comparison)
3. Re-render the last `ChatMessageComponent` (new object reference)
4. Update the `<pre>` text node DOM

For a conversation with 100 messages, that is 100 key comparisons per chunk.

### addMessage creates new array too

`addMessage` uses `setMessages((prev) => [...prev, msg])` -- spread creates a new array, same reconciliation cost.

---

## 4. Scroll Event Handlers

**No scroll event handlers anywhere in the codebase.**

No `onScroll` prop, no `addEventListener("scroll", ...)`. The commented-out `isNearBottom` was the planned scroll handler but was never wired up.

Auto-scroll is done purely via `scrollIntoView` in a `useEffect`, not via scroll event listening.

### Verdict: no scroll-driven layout reads. Clean.

---

## 5. ResizeObserver / IntersectionObserver

**Neither is used anywhere in the codebase.**

No virtualization, no lazy rendering, no intersection-based visibility detection.

### Implication

The entire message list is fully rendered in the DOM at all times. For long conversations (hundreds of messages), this means:
- Large DOM tree
- More work during React reconciliation
- More elements for the browser to composite

No ResizeObserver means window resize is handled purely by CSS flexbox/grid (which is correct and efficient).

---

## 6. The Streaming Render Path (CRITICAL)

### Flow: NDJSON line arrives

1. **Backend** (`chat.ts`): SDK message arrives -> JSON.stringify + "\n" -> enqueue to ReadableStream
2. **Frontend** (`ChatPage.tsx` line 214-227): `reader.read()` returns a chunk
3. Chunk is split by `\n` into lines
4. **For each line**: `processStreamLine(line, streamingContext)` is called synchronously in a `for` loop

### Per-line state updates for assistant text

When a `type: "assistant"` message with `text` content arrives:

1. `UnifiedMessageProcessor.handleAssistantText()`:
   - First text chunk: calls `context.addMessage(newChatMessage)` + `context.setCurrentAssistantMessage(msg)`
     - `addMessage` -> `setMessages(prev => [...prev, msg])` -- **React state update #1**
     - `setCurrentAssistantMessage` -> **React state update #2**
   - Subsequent text chunks: calls `context.updateLastMessage(updatedContent)` + `context.setCurrentAssistantMessage(updatedMsg)`
     - `updateLastMessage` -> `setMessages(prev => prev.map(...))` -- **React state update #1**
     - `setCurrentAssistantMessage` -> **React state update #2**

2. After the `for` loop over lines completes, React batches the state updates from that `reader.read()` chunk into one reconciliation (React 18 automatic batching applies here since this runs inside an async function, and the state updates within the synchronous `for` loop are batched).

### BUT: each reader.read() chunk triggers a separate render

The `while(true) { reader.read(); for(line of lines) { ... } }` loop means:
- Each network chunk (which may contain 1 or more NDJSON lines) triggers one React render cycle
- The browser's streaming typically delivers chunks at network-level granularity
- During fast streaming, this could be dozens of renders per second

### Per render cycle, what happens:

1. `ChatMessages` re-renders (new `messages` reference)
2. All message keys are diffed
3. Last message's `<pre>` text content is updated (DOM text node mutation)
4. `useEffect([messages])` fires -> `scrollIntoView({ behavior: "smooth" })`
5. `scrollIntoView` forces layout computation
6. Smooth scroll animation starts (compositing work for ~300-500ms)
7. If another chunk arrives before smooth scroll finishes, a NEW `scrollIntoView` call interrupts the previous one

### The scrollIntoView storm

During active streaming, `scrollIntoView({ behavior: "smooth" })` is called on EVERY render. Each call:
- Forces the browser to compute layout (element positions)
- Starts a scroll animation
- The next call arrives before the animation completes, canceling it and starting a new one
- This creates continuous layout + composite work

This is the single biggest rendering pipeline cost during streaming.

### State updates per NDJSON line (non-text types)

- `tool_use`: `addMessage(toolMessage)` -- 1 state update
- `tool_result`: `addMessage(toolResultMessage)` -- 1 state update
- `system/init`: `addMessage()` + `setHasReceivedInit(true)` + possibly `setHasShownInitMessage(true)` -- 2-3 state updates
- `result`: `addMessage()` + `setCurrentAssistantMessage(null)` -- 2 state updates

All batched by React 18 within the synchronous `for` loop.

---

## 7. Window Resize Handling

**No JavaScript resize handling.** Layout is purely CSS-driven:
- `min-h-screen`, `h-screen`, `flex`, `flex-1`, `overflow-y-auto`
- Responsive breakpoints via Tailwind (`sm:` prefix)

The textarea auto-resize effect (`useEffect([input])`) does NOT re-fire on window resize -- it only fires when `input` text changes.

### Verdict: window resize is handled correctly by CSS. No layout thrashing.

---

## 8. Idle State Analysis

### After streaming completes, what remains active:

1. **Global keydown listener** (ChatPage.tsx) -- passive, no-op when `!isLoading`
2. **TimestampComponent intervals** -- only if `mode === "relative"` (default is "absolute", so NOT active)
3. **React Router** -- internal history listener, negligible
4. **No polling, no WebSocket, no SSE keepalive**

### Nothing fires at idle. The page is completely quiet.

Verified: no `setInterval`, no `requestAnimationFrame`, no `setTimeout` loops, no WebSocket connections, no SSE event sources, no polling fetches.

---

## Summary: Top Rendering Pipeline Costs

### During streaming (active)

| Rank | Issue | Impact |
|------|-------|--------|
| 1 | `scrollIntoView({ behavior: "smooth" })` on every chunk | Forces layout + starts scroll animation per chunk. During fast streaming, creates continuous layout thrash as animations are constantly interrupted and restarted. |
| 2 | Full message list reconciliation per chunk | `messages.map()` creates new array, React diffs all keys. O(n) where n = total messages. |
| 3 | `backdrop-blur-sm` on message container | GPU-composited blur on the scrolling container. Every scroll frame requires re-compositing the blur effect. Used on: ChatMessages container, ChatInput, header buttons (8+ elements). |
| 4 | No message list virtualization | Entire DOM tree rendered. For 200+ message conversations, significant paint/composite surface. |

### During typing (user input)

| Rank | Issue | Impact |
|------|-------|--------|
| 1 | Textarea auto-resize layout thrash | write(height:auto) -> read(getComputedStyle + scrollHeight) -> write(height:Npx). Forced sync layout per keystroke. |

### At idle

**No rendering pipeline activity.** Clean idle.

### CSS properties causing compositor work

- `backdrop-blur-sm` -- forces compositing layer, blur recalculated on scroll/repaint
- `transition-all duration-200` -- on many elements; hover transitions are fine but `transition-all` transitions ALL properties including layout-triggering ones (width, height, padding)
- `animate-spin` on loading spinner -- runs only when `isLoading` is true, stops at idle
- `animate-pulse` on "Thinking..." text -- same, only during loading
