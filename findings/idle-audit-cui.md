# CUI Idle Activity Audit

**Date:** 2026-03-26
**Scope:** `src/` directory, static analysis only
**Method:** Exhaustive search for all timer, animation, observer, and polling patterns

## Summary

| Category | Total | Runs During Idle | Severity |
|---|---|---|---|
| setInterval | 4 | 2 (server-side SSE heartbeats) | LOW |
| setTimeout (recursive/re-scheduling) | 2 | 0 | NONE |
| requestAnimationFrame | 2 | 0 | NONE |
| CSS animations (Tailwind) | 7 uses | 4 conditional on state | MEDIUM |
| Network timers (SSE heartbeat) | 2 | 2 (server-side) | LOW |
| Reconnection timers | 2 | 0 (only on disconnect) | NONE |
| MutationObserver | 1 | 1 | LOW |
| IntersectionObserver | 1 | 1 | LOW |
| ResizeObserver | 0 | - | NONE |
| Service Worker | 1 | 0 (no polling/sync) | NONE |
| React useEffect polling | 0 | - | NONE |

**Idle Score: 8/10 (good)**
Most activity is event-driven or state-gated. The only truly idle-persistent items are server-side SSE heartbeats (invisible to display) and one MutationObserver watching for theme attribute changes. No client-side timers or RAF loops run during idle. CSS animations only fire during active loading/streaming states.

---

## 1. setInterval (4 occurrences)

### 1a. SSE heartbeat - StreamManager (server-side)
- **File:** `src/services/stream-manager.ts:254`
- **Interval:** 30,000ms (30s)
- **Purpose:** Sends SSE comment `:heartbeat` to all connected clients to prevent connection timeout
- **Runs during idle:** YES (whenever any SSE client is connected)
- **Cleared:** Yes, `stopHeartbeat()` called when no clients remain
- **Display impact:** NONE (server-side only, writes SSE comment, no pixel change)
- **Disableable:** Only by disconnecting all SSE clients

### 1b. SSE heartbeat - Log stream (server-side)
- **File:** `src/routes/log.routes.ts:67`
- **Interval:** 30,000ms (30s)
- **Purpose:** Keeps log stream SSE connection alive
- **Runs during idle:** YES (while log stream tab is open)
- **Cleared:** Yes, on `req.close` event (line 72-74)
- **Display impact:** NONE (server-side only)
- **Disableable:** Close the log stream connection

### 1c. Audio recording duration counter
- **File:** `src/web/chat/hooks/useAudioRecording.ts:108`
- **Interval:** 100ms
- **Purpose:** Updates displayed recording duration every 100ms
- **Runs during idle:** NO (only during active audio recording)
- **Cleared:** Yes, on `stopRecording()` (line 141-143)
- **Display impact:** Updates duration text while recording only
- **Disableable:** Automatic - stops when recording stops

### 1d. Config polling (test-only)
- **File:** `src/services/config-service.ts:366`
- **Interval:** 50ms
- **Purpose:** Polls config file for changes in test environment
- **Runs during idle:** Only in `NODE_ENV=test`
- **Cleared:** Yes, in `stopWatching()`
- **Display impact:** NONE (server-side, test-only)
- **Disableable:** N/A - production uses `fs.watch` instead (event-driven)

---

## 2. setTimeout (recursive/re-scheduling patterns)

### 2a. MCP permission polling loop
- **File:** `src/mcp-server/index.ts:193`
- **Interval:** 1,000ms (POLL_INTERVAL)
- **Purpose:** Polls for permission decision in MCP tool handler
- **Runs during idle:** NO (only while a permission request is pending, max 1 hour timeout)
- **Cleared:** Exits loop when permission granted/denied/timeout
- **Display impact:** NONE (server-side MCP handler)
- **Disableable:** N/A - request-scoped

### 2b. SSE reconnection timers
- **File:** `src/web/chat/hooks/useStreaming.ts:133` (5,000ms)
- **File:** `src/web/chat/hooks/useMultipleStreams.ts:228` (1,000ms base, exponential backoff, max 3 retries)
- **Purpose:** Reconnect to SSE stream after unintentional disconnect
- **Runs during idle:** NO (only after connection failure, finite retries)
- **Display impact:** NONE (network reconnection)
- **Disableable:** Automatic - limited retries, visibility-gated

### Non-recursive setTimeout instances (one-shot, NOT idle concerns):
- `src/web/chat/components/Composer/Composer.tsx:595,612,737,811` -- cursor positioning (0ms)
- `src/web/chat/components/ConversationView/ConversationView.tsx:141` -- focus input (100ms)
- `src/web/chat/components/Home/Home.tsx:55,160,166` -- focus input (50-100ms)
- `src/web/chat/components/JsonViewer/JsonViewer.tsx:20` -- "Copied!" reset (2,000ms)
- `src/web/chat/components/MessageList/MessageItem.tsx:94` -- copy badge reset (2,000ms)
- `src/web/chat/components/PreferencesModal/PreferencesModal.tsx:102` -- status clear (3,000ms)
- `src/web/chat/components/PreferencesModal/NotificationTab.tsx:117` -- test sent reset (3,000ms)
- `src/web/inspector/InspectorApp.tsx:634,656` -- "Copied!" reset (2,000ms)
- `src/services/claude-process-manager.ts:212,220,282,536` -- process lifecycle (server-side)

---

## 3. requestAnimationFrame (2 occurrences)

### 3a. Audio data visualization RAF loop
- **File:** `src/web/chat/hooks/useAudioRecording.ts:98`
- **Purpose:** Reads frequency data from AnalyserNode, updates `audioData` state for waveform display
- **Runs during idle:** NO (only during active audio recording, gated by `analyserRef.current` check)
- **Cleared:** `cancelAnimationFrame()` on `stopRecording()` (line 191-193)
- **Display impact:** Drives WaveformVisualizer canvas at ~60fps while recording
- **Disableable:** Automatic - stops when recording stops

### 3b. WaveformVisualizer animation loop
- **File:** `src/web/chat/components/WaveformVisualizer/WaveformVisualizer.tsx:134,151`
- **Purpose:** Canvas drawing loop for waveform bars, throttled to 15fps internally
- **Runs during idle:** NO (gated by `isRecording` prop, line 149)
- **Cleared:** `cancelAnimationFrame()` when `isRecording` becomes false (line 152-153)
- **Display impact:** Canvas redraws at 15fps effective rate while recording
- **Disableable:** Automatic - controlled by `isRecording` state

---

## 4. CSS Animations (7 uses)

### 4a. Loading dots (animate-bounce) -- CONDITIONAL
- **File:** `src/web/chat/components/MessageList/MessageList.tsx:98-100`
- **Elements:** 3 bouncing dots with staggered delays (-0.32s, -0.16s, 0s)
- **Runs during idle:** NO (conditional: `isLoading && displayMessages.length === 0`)
- **Display impact:** 3 dots bouncing infinitely while loading initial messages
- **Disableable:** Disappears when loading completes

### 4b. Streaming indicator (animate-pulse) -- CONDITIONAL
- **File:** `src/web/chat/components/MessageList/MessageList.tsx:124`
- **Element:** 2.5x2.5 dot pulsing during streaming
- **Runs during idle:** NO (conditional: `isStreaming && !hasLoadingToolUse`)
- **Display impact:** Single dot opacity pulsing (~2s CSS animation cycle)
- **Disableable:** Disappears when streaming ends

### 4c. Tool icon pulse (animate-pulse) -- CONDITIONAL
- **File:** `src/web/chat/components/MessageList/MessageItem.tsx:199`
- **Element:** Tool use icon pulses while tool is executing
- **Runs during idle:** NO (conditional: `shouldBlink` which requires pending tool result)
- **Display impact:** Icon opacity pulsing during tool execution
- **Disableable:** Stops when tool result arrives

### 4d. Audio processing spinner (animate-spin) -- CONDITIONAL
- **File:** `src/web/chat/components/Composer/Composer.tsx:935`
- **Element:** Loader2 icon spinning during audio processing
- **Runs during idle:** NO (conditional: `audioState === 'processing'`)
- **Display impact:** Spinner rotation
- **Disableable:** Disappears when processing completes

### 4e. Submit button spinner (animate-spin) -- CONDITIONAL
- **File:** `src/web/chat/components/Composer/Composer.tsx:1052`
- **Element:** Loader2 icon in submit button
- **Runs during idle:** NO (conditional: `isLoading`)
- **Display impact:** Spinner rotation while submitting
- **Disableable:** Disappears when submission completes

### 4f. Task "Running" status (animate-pulse + shimmer) -- CONDITIONAL
- **File:** `src/web/chat/components/Home/TaskItem.tsx:187`
- **Element:** Text with pulse animation, optionally shimmer (2s linear infinite)
- **Runs during idle:** PARTIALLY -- if a task is actively running and visible on Home screen, this pulses/shimmers continuously. Not idle in the "nothing happening" sense, but the animation runs even if the user is not looking.
- **Display impact:** Text opacity pulse + gradient shimmer on "Running" status
- **Disableable:** Only when task completes

### 4g. "Loading more tasks" pulse (animate-pulse) -- CONDITIONAL
- **File:** `src/web/chat/components/Home/TaskList.tsx:238`
- **Element:** "Loading more tasks..." text pulsing
- **Runs during idle:** NO (conditional: `loadingMore`)
- **Display impact:** Text opacity pulse while loading
- **Disableable:** Disappears when loading completes

### 4h. Notification processing spinner (animate-spin) -- CONDITIONAL
- **File:** `src/web/chat/components/PreferencesModal/NotificationTab.tsx:213`
- **Element:** Loader2 spinning during push subscription
- **Runs during idle:** NO (conditional: `webPushLoading`)
- **Display impact:** Spinner rotation
- **Disableable:** Disappears when done

### CSS Variables (defined but not actively animating)
- **File:** `src/web/chat/styles/theme.css:80,113`
- `--loading-shimmer` gradient defined in both light/dark themes. Not applied via any `@keyframes` in CSS. Only referenced inline in TaskItem.tsx (item 4f above).

### Tailwind custom animation
- **File:** `tailwind.config.js:9-23`
- `slide-up`: 0.2s ease-out, one-shot (not infinite). Used for entrance transitions. Not an idle concern.

---

## 5. Network Timers

### 5a-5b. SSE heartbeats
See items 1a and 1b above. Both are server-side, 30s interval, no display impact.

### 5c. SSE reconnection
See item 2b above. Client-side, only on disconnect, finite retries with exponential backoff.

---

## 6. Observers

### 6a. MutationObserver -- theme watcher
- **File:** `src/web/main.tsx:34-42`
- **Target:** `document.documentElement` attributes
- **Watches:** `data-theme` attribute changes
- **Runs during idle:** YES (always observing, but only fires callback on theme change)
- **Display impact:** NONE during idle (only updates `<meta name="theme-color">` when theme changes)
- **Disableable:** No, but zero cost when idle -- pure event listener, no polling

### 6b. IntersectionObserver -- infinite scroll
- **File:** `src/web/chat/components/Home/TaskList.tsx:149`
- **Target:** Load-more sentinel element at bottom of task list
- **Watches:** Intersection with scroll container (100px rootMargin, 0.1 threshold)
- **Runs during idle:** YES (observer is registered while TaskList is mounted)
- **Display impact:** NONE during idle (only triggers `onLoadMore` when sentinel scrolls into view)
- **Disableable:** No, but zero cost when idle -- browser-native, no polling

---

## 7. React Effects with Timers/Polling

No `useEffect` hooks contain `setInterval` or periodic polling. All effects are either:
- One-shot setup/teardown (event listeners, SSE connections)
- Event-driven (visibility change, focus)
- Cleanup-on-unmount patterns

### Focus/visibility refresh (NOT polling)
- **File:** `src/web/chat/components/Home/Home.tsx:69-93`
- **Purpose:** Refreshes conversation list on window focus or tab becoming visible
- **Runs during idle:** NO (only on focus/visibility events)
- **Display impact:** One-time data fetch, not continuous

---

## 8. Service Worker

### 8a. PWA Service Worker
- **File:** `src/web/sw.ts` (65 lines)
- **Registration:** `src/web/main.tsx:54-57`
- **Features:** Workbox precache + push notification handling + notification click
- **Background sync:** NONE
- **Background fetch:** NONE
- **Periodic polling:** NONE
- **Runs during idle:** Only responds to push events (externally triggered)
- **Display impact:** Shows notification on push event (not idle-initiated)
- **Disableable:** Can unregister SW

---

## Idle Activity Summary

### What runs when the screen is completely idle (no streaming, no recording, no loading):

1. **Server-side SSE heartbeats** (30s interval) -- invisible to display, keeps TCP alive
2. **MutationObserver** on `<html>` -- zero-cost event listener, never fires during idle
3. **IntersectionObserver** on task list sentinel -- zero-cost browser observer, never fires during idle
4. **Service Worker** -- dormant, no timers

### What does NOT run during idle:
- No `setInterval` on the client side
- No `requestAnimationFrame` loops
- No CSS animations (all are state-gated)
- No polling hooks
- No recursive `setTimeout` chains
- No background sync/fetch

### Potential concern:
If a task is actively "running" (status === 'ongoing'), the TaskItem shimmer/pulse animation runs continuously on the Home screen even if the user is not interacting. This is the only CSS animation that could persist for extended periods. All others are transient (loading, processing states).

### Config file watching (production):
Uses `fs.watch()` (event-driven, OS-level inotify/ReadDirectoryChangesW), not polling. Zero CPU cost during idle.
