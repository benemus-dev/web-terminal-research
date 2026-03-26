# Chrome Rendering Pipeline Audit: claudecodeui

## Architecture Summary
- **Framework:** React 18 (full virtual DOM)
- **Terminal:** xterm.js with FitAddon (in Shell component)
- **Transport:** WebSocket (for chat streaming + shell terminal)
- **Components:** Large React component tree (~314 source files)
- **Source:** `/c/dev/web-terminal-research/claudecodeui/src/`

---

## 1. Forced Synchronous Layout

### Hits Found (17 unique locations)

| Location | Property | Context | Severity |
|----------|----------|---------|----------|
| `useChatComposerState.ts:740` | `scrollHeight` | Textarea auto-resize | HIGH |
| `useChatComposerState.ts:741` | `getComputedStyle` | Read lineHeight after height write | HIGH |
| `useChatComposerState.ts:827` | `scrollHeight` | Input handler auto-resize | HIGH |
| `useChatComposerState.ts:831` | `getComputedStyle` | Read lineHeight after height write | HIGH |
| `useChatComposerState.ts:896` | `scrollHeight` + `getComputedStyle` | Transcript handler | MEDIUM |
| `useChatComposerState.ts:386-387` | `scrollTop`/`scrollLeft` | Sync overlay scroll | LOW |
| `useChatSessionState.ts:209` | `scrollTop = scrollHeight` | Scroll to bottom | MEDIUM |
| `useChatSessionState.ts:224-225` | `scrollTop`/`scrollHeight`/`clientHeight` | isNearBottom check | LOW |
| `useChatSessionState.ts:238-239` | `scrollHeight`/`scrollTop` | loadOlderMessages | MEDIUM |
| `useChatSessionState.ts:270,273` | `scrollTop` | handleScroll | LOW |
| `useChatSessionState.ts:285-286` | `scrollHeight`/`scrollTop` | useLayoutEffect scroll restore | HIGH |
| `useChatSessionState.ts:524` | `scrollIntoView` | Search scroll to element | LOW |
| `useChatSessionState.ts:575` | `scrollHeight`/`scrollTop` | Every-render scroll tracking | HIGH |
| `ChatComposer.tsx:154` | `getBoundingClientRect` | Command menu positioning | LOW |
| `CommandMenu.tsx:113-114` | `getBoundingClientRect` x2 | Menu/item rect for scroll | LOW |
| `CommandMenu.tsx:116` | `scrollIntoView` | Selected item visibility | LOW |
| `MainContentHeader.tsx:23-24` | `scrollLeft`/`scrollWidth`/`clientWidth` | Tab scroll indicators | LOW |
| `EditorSidebar.tsx:51` | `clientWidth` | Sidebar width calc | LOW |
| `useEditorSidebar.ts:75` | `getBoundingClientRect` | Container rect for drag | LOW |
| `FileContextMenu.tsx:35-40` | `innerWidth`/`innerHeight` | Context menu positioning | LOW |
| `useDeviceSettings.ts:14` | `innerWidth` | Mobile breakpoint check | LOW |
| `useQuickSettingsDrag.ts:142,228` | `innerHeight` | Drag constraints | LOW |

### Critical Forced Reflow Patterns

**Pattern 1: Textarea auto-resize (fires on EVERY keystroke during input)**
`useChatComposerState.ts:739-742`:
```
textareaRef.current.style.height = 'auto';                              // DOM WRITE
textareaRef.current.style.height = `${textareaRef.current.scrollHeight}px`; // LAYOUT READ (forced reflow)
const lineHeight = parseInt(window.getComputedStyle(...).lineHeight);    // LAYOUT READ
```
This write-read-read pattern forces synchronous layout on every character typed. The `useEffect` on `[input]` (line 734) means this runs after every keystroke React re-render.

**Duplicate effect:** Lines 746-752 have ANOTHER `useEffect` on `[input]` that does `style.height = 'auto'` again. Two effects run sequentially on the same dependency.

**Pattern 2: Every-render scroll position tracking**
`useChatSessionState.ts:572-577`:
```
useEffect(() => {
    // No dependency array filter -- runs on EVERY render
    const container = scrollContainerRef.current;
    scrollPositionRef.current = { height: container.scrollHeight, top: container.scrollTop };
});
```
This `useEffect` has **no dependency array optimization for this branch** -- it reads `scrollHeight` and `scrollTop` on every single React render cycle. During streaming, this means layout reads on every state update.

**Pattern 3: useLayoutEffect scroll restore**
`useChatSessionState.ts:281-288`:
```
useLayoutEffect(() => {
    const newScrollHeight = container.scrollHeight;  // LAYOUT READ
    container.scrollTop = top + ...;                 // DOM WRITE
}, [chatMessages.length]);
```
`useLayoutEffect` runs synchronously before browser paint. This blocks painting until scroll restoration completes. Fires whenever `chatMessages.length` changes.

---

## 2. React Reconciliation

### Always-Mounted Components
The component tree during active chat includes:
- `App` -> `AppContent` -> `ChatInterface` -> `ChatMessagesPane` -> per-message `MessageComponent`
- `ChatComposer` (always mounted)
- `ClaudeStatus` (always mounted during session)
- `ThinkingModeSelector` (always mounted)
- `TokenUsagePie` (always mounted)
- `PermissionRequestsBanner` (always mounted when permissions pending)
- Shell terminal (if Shell tab is active -- xterm.js container)
- `MainContentHeader` with tab switcher
- Sidebar with project list

**Estimated always-mounted components: 30-50** depending on active view.

### useLayoutEffect Usage
**1 instance found:**
- `useChatSessionState.ts:281` -- Scroll position restore after loading older messages. Runs synchronously before paint. Depends on `[chatMessages.length]`.

### useEffect Reading Layout Properties
- `useChatSessionState.ts:572-577` -- Reads `scrollHeight`/`scrollTop` on every render (no deps)
- `useChatComposerState.ts:734-743` -- Reads `scrollHeight` and `getComputedStyle` on `[input]` change
- `useChatComposerState.ts:746-752` -- Reads nothing but writes `style.height` on `[input]` change

### Streaming Re-renders

**Stream delta buffering (critical optimization present):**
`useChatRealtimeHandlers.ts:184-201`:
```
if (msg.kind === 'stream_delta') {
    streamBufferRef.current += text;
    if (!streamTimerRef.current) {
        streamTimerRef.current = window.setTimeout(() => {
            sessionStore.updateStreaming(sid, accumulatedStreamRef.current, provider);
        }, 100);  // 100ms buffer
    }
    return;
}
```

Stream deltas are buffered with a **100ms debounce timer**. This means:
- During streaming at ~50 tokens/sec: ~10 React re-renders/sec (not 50)
- Each re-render triggers the full message list reconciliation
- Each re-render triggers the every-render `useEffect` that reads scroll position

**Estimated re-renders during streaming: 10/sec** (from stream buffer) + additional renders from status updates, scroll events.

### State Update Batching
React 18 batches state updates in event handlers and effects automatically. However:
- The streaming handler uses `setTimeout` which in React 18 IS batched (improvement over React 17)
- Multiple `setState` calls in `useChatRealtimeHandlers` (setIsLoading, setCanAbortSession, setClaudeStatus, setPendingPermissionRequests) are batched into one render

---

## 3. DOM Mutations During Idle

### Cursor Blink (Shell tab)
When Shell tab is active with xterm.js: cursor blink depends on xterm.js renderer used. claudecodeui does not explicitly set a renderer addon -- likely uses default DOM renderer.
- DOM renderer: ~2 mutations/sec for cursor blink
- If WebGL addon were loaded: 0 mutations/sec

### Status Indicators
- `ClaudeStatus` component: only re-renders when `claudeStatus` state changes (not idle)
- `TokenUsagePie`: static between updates
- No polling intervals found in chat components

### Version Check
`useVersionCheck.ts:83`: `setInterval(checkVersion, 5 * 60 * 1000)` -- HTTP fetch every 5 minutes. If version changed, triggers a state update and re-render. Otherwise, no DOM mutation.

### Other Idle Activity
- `MainContentHeader.tsx:31-33`: ResizeObserver on tab container -- only fires on actual resize, not idle
- No heartbeat/ping DOM writes found

**Idle DOM mutations (chat view): 0/sec**
**Idle DOM mutations (shell view): ~2/sec** (cursor blink via DOM renderer)

---

## 4. Scroll Event Handling

### Chat scroll handler
`useChatSessionState.ts:262-279`:
```
const handleScroll = useCallback(async () => {
    const nearBottom = isNearBottom();  // reads scrollTop, scrollHeight, clientHeight
    setIsUserScrolledUp(!nearBottom);   // React state update

    if (container.scrollTop < 100) {    // reads scrollTop
        await loadOlderMessages(container);  // may read scrollHeight, scrollTop
    }
});
```
This handler:
1. Reads 3 layout properties (scrollTop, scrollHeight, clientHeight)
2. Triggers React state update (`setIsUserScrolledUp`)
3. May trigger async message loading which reads 2 more layout properties

**No forced reflows** in the scroll handler itself -- reads only, no preceding writes.

### Auto-scroll during streaming
`useChatSessionState.ts:579-595`:
```
useEffect(() => {
    if (autoScrollToBottom && !isUserScrolledUp) {
        setTimeout(() => scrollToBottom(), 50);  // scrollTop = scrollHeight
    }
    // or: container.scrollTop = prevTop + heightDiff
});
```
During streaming, this fires after every render (when new messages arrive). The `scrollToBottom()` sets `scrollTop = scrollHeight` which is a write. The preceding render may have changed DOM, making this a potential forced reflow on the next layout read.

### scrollIntoView
- `useChatSessionState.ts:524`: `scrollIntoView({ block: 'center', behavior: 'smooth' })` -- only on search navigation, not during streaming
- `CommandMenu.tsx:116`: `scrollIntoView({ block: 'nearest', behavior: 'smooth' })` -- only when navigating command menu with keyboard

---

## 5. Resize Handling

### Chat view resize
- `useDeviceSettings.ts:55`: `window.addEventListener('resize', checkMobile)` -- reads `window.innerWidth`, sets state if mobile breakpoint changes. **1 layout read + potential re-render.**
- `EditorSidebar.tsx:68`: `window.addEventListener('resize', updateWidth)` -- reads `parentElement.clientWidth`. **1 layout read.**

### Shell terminal resize
`useShellTerminal.ts:222-241`:
```
const resizeObserver = new ResizeObserver(() => {
    clearTimeout(resizeTimeoutRef.current);
    resizeTimeoutRef.current = setTimeout(() => {
        currentFitAddon.fit();       // reads clientWidth/clientHeight, writes terminal dims
        sendSocketMessage(wsRef.current, { type: 'resize', cols, rows });
    }, TERMINAL_RESIZE_DELAY_MS);
});
```
**Debounced resize** -- FitAddon.fit() runs after delay. Single read-write cycle, not thrashing. Good practice.

**Total layout reads per resize (chat view): ~3** (innerWidth + clientWidth + any scroll position reads)
**Total layout reads per resize (shell view): ~5** (above + FitAddon reads)

---

## 6. WebSocket Message Processing

### Chat messages
Each WS message updates `latestMessage` state. `useChatRealtimeHandlers` processes it:
- `stream_delta`: Buffered with 100ms timer -> `sessionStore.updateStreaming()` -> React state update -> re-render
- `stream_end`, `complete`, `error`, `status`, `permission_request`: Immediate state updates (batched by React 18)
- `text`, `tool_use`, `tool_result`: Routed to session store -> triggers re-render

**Stream deltas are NOT rendered per-message.** The 100ms buffer coalesces ~5-10 deltas into one render.

**Non-delta messages trigger immediate React re-renders** (1 render per message, batched within same tick).

### Shell terminal messages
Shell WS messages go directly to xterm.js `terminal.write()` via `onData` handler. xterm.js batches internally. **No React involvement for shell data.**

---

## 7. Estimated Pipeline Runs/Second

### Idle State (Chat view, no streaming)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | 0 | No DOM changes |
| Layout | 0 | No layout-triggering operations |
| Paint | 0 | Nothing to repaint |
| Composite | 0 | Nothing to composite |

### Idle State (Shell view, cursor blinking)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~2 | xterm.js cursor blink (DOM renderer) |
| Layout | 0 | Cursor blink is style-only |
| Paint | ~2 | Repaint cursor region |
| Composite | ~2 | Recomposite terminal layer |

### During Chat Streaming (~50 tokens/sec)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~10 | React re-renders from 100ms buffer flush |
| Layout | ~10 | Scroll position reads + auto-scroll writes |
| Paint | ~10 | Message list content updates |
| Composite | ~10 | Layer recomposition |

Plus: the every-render `useEffect` reads `scrollHeight`/`scrollTop` on each of the ~10 renders, and auto-scroll writes `scrollTop`. This creates ~10 potential forced reflow cycles per second during streaming.

### During Chat Input (typing)

| Pipeline Stage | Runs/sec | Notes |
|---------------|----------|-------|
| Style Recalc | ~10-30 | Per-keystroke re-render + textarea resize |
| Layout | ~10-30 | Forced reflow from height='auto' then scrollHeight read |
| Paint | ~10-30 | Textarea content + height change |
| Composite | ~10-30 | Layer recomposition |

The textarea auto-resize pattern (write `height='auto'`, read `scrollHeight`, write `height=scrollHeight+'px'`, read `getComputedStyle`) fires on every keystroke. This is the highest-frequency forced reflow in the codebase.

---

## Key Findings

1. **Textarea auto-resize is the worst offender.** Write-read-read pattern on every keystroke: `style.height='auto'` -> `scrollHeight` -> `getComputedStyle`. Two duplicate `useEffect`s on `[input]` compound this.

2. **Every-render scroll tracking** (`useChatSessionState.ts:572-577`) reads layout properties on every React render with no dependency guard. During streaming at 10 renders/sec, this is 10 unnecessary layout reads/sec.

3. **Stream buffering at 100ms is effective** -- reduces 50 token/sec stream to ~10 renders/sec. Without it, the rendering cost would be 5x higher.

4. **useLayoutEffect scroll restore** blocks paint synchronously. Only fires on message count change (pagination), not during streaming -- acceptable.

5. **Shell terminal resize is well-debounced** -- uses ResizeObserver + setTimeout delay before FitAddon.fit().

6. **No idle DOM mutations in chat view.** The application is completely quiet when not streaming or typing.

7. **React component count is large** (~30-50 always-mounted) but reconciliation only runs when state actually changes. During idle, this cost is 0.

8. **The 100ms stream buffer timer** means latency between stream delta arrival and visual update is up to 100ms. This is a deliberate tradeoff of responsiveness for rendering efficiency.
