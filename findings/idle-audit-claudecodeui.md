# Idle Activity Audit: claudecodeui

Static code analysis of `src/` and `server/` directories.
Date: 2026-03-26

## Summary Counts

| Category | Total Found | Runs During Idle | Infinite/Repeating |
|---|---|---|---|
| setInterval (client) | 4 | 3 | 3 |
| setInterval (server) | 1 | 1 | 1 |
| setTimeout (recursive / self-scheduling) | 1 | 1 | 1 |
| requestAnimationFrame | 3 | 0 | 0 |
| CSS @keyframes (infinite) | 2 definitions | conditionally | yes when active |
| Tailwind animate-* (infinite) | ~60 usages | ~10 | ~10 |
| MutationObserver | 0 | - | - |
| ResizeObserver | 3 | 3 (event-driven) | no (passive) |
| IntersectionObserver | 1 | 1 (event-driven) | no (passive) |
| WebSocket reconnect timer | 1 | 1 | 1 |
| xterm.js cursor blink | 1 | 1 | 1 |
| React useEffect with intervals | 4 | 3 | 3 |

**Idle Score: 5/10 (moderate idle activity)**

Primary idle offenders: sidebar sort-order polling (1s), sidebar clock update (60s), version check polling (5min), WebSocket reconnect loop, xterm cursor blink, and CSS animate-pulse/animate-ping on always-visible sidebar elements.

---

## 1. setInterval -- All Occurrences

### 1a. Sidebar sort-order polling
- **File:** `src/components/sidebar/hooks/useSidebarController.ts:177`
- **Interval:** 1000ms (1 second)
- **Purpose:** Polls localStorage for sort-order changes from other tabs
- **Runs during idle:** YES -- always, unconditionally
- **Cleared on idle:** NO -- only cleared on unmount
- **Disableable:** Not via config. Would need code change.
- **Severity:** HIGH -- 1Hz polling on every frame is wasteful. The `storage` event listener at line 169 already covers cross-tab sync, making this redundant.

### 1b. Sidebar clock update
- **File:** `src/components/sidebar/hooks/useSidebarController.ts:124`
- **Interval:** 60000ms (1 minute)
- **Purpose:** Updates `currentTime` state for relative timestamps ("5 min ago")
- **Runs during idle:** YES -- always
- **Cleared on idle:** NO -- only cleared on unmount
- **Disableable:** No config option
- **Severity:** LOW -- once per minute, triggers re-render of sidebar timestamps

### 1c. ClaudeStatus elapsed timer
- **File:** `src/components/chat/view/subcomponents/ClaudeStatus.tsx:68`
- **Interval:** 1000ms
- **Purpose:** Updates elapsed time counter during AI response generation
- **Runs during idle:** NO -- guarded by `if (!isLoading) return`
- **Cleared on idle:** YES -- cleanup runs when isLoading becomes false
- **Disableable:** N/A (already idle-safe)

### 1d. ClaudeStatus animation phase
- **File:** `src/components/chat/view/subcomponents/ClaudeStatus.tsx:81`
- **Interval:** 500ms
- **Purpose:** Cycles animated dots ("Thinking...", "Processing...") during AI response
- **Runs during idle:** NO -- guarded by `if (!isLoading) return`
- **Cleared on idle:** YES
- **Disableable:** N/A (already idle-safe)

### 1e. Version check polling
- **File:** `src/hooks/useVersionCheck.ts:83`
- **Interval:** 300000ms (5 minutes)
- **Purpose:** Fetches GitHub API for latest release version
- **Runs during idle:** YES -- always, unconditionally
- **Cleared on idle:** NO -- only on unmount
- **Disableable:** No config. Would need code change.
- **Severity:** LOW -- 5-min interval is infrequent, but generates network traffic during idle

### 1f. Server: Codex session cleanup
- **File:** `server/openai-codex.js:406`
- **Interval:** 300000ms (5 minutes)
- **Purpose:** Cleans up completed Codex sessions older than 30 minutes
- **Runs during idle:** YES -- always (server-side, no display impact)
- **Cleared on idle:** Never cleared (module-level, runs for server lifetime)
- **Disableable:** No
- **Severity:** NONE for display (server-side only)

---

## 2. setTimeout (Recursive / Self-Scheduling)

### 2a. WebSocket reconnection
- **File:** `src/contexts/WebSocketContext.tsx:86`
- **Interval:** 3000ms delay, then re-calls `connect()`
- **Purpose:** Reconnects WebSocket after disconnection
- **Runs during idle:** YES -- if connection drops, retries every 3s indefinitely
- **Cleared on idle:** Only if component unmounts or connection succeeds
- **Disableable:** No
- **Severity:** MEDIUM during disconnect -- generates connection attempts every 3s. Idle when connected.

### Non-recursive setTimeout (not idle concerns):
- `src/hooks/useProjectsState.ts:220` -- 500ms one-shot to clear loading progress
- `src/shared/view/ui/Tooltip.tsx:63` -- tooltip show delay, one-shot
- `src/components/chat/view/subcomponents/ChatComposer.tsx` -- various one-shot delays
- `src/components/shell/view/Shell.tsx:167` -- one-shot focus delay
- Server-side: various one-shot timeouts for process cleanup, debounce

---

## 3. requestAnimationFrame

### 3a. File mentions cursor positioning
- **File:** `src/components/chat/hooks/useFileMentions.tsx:203`
- **Purpose:** One-shot RAF to position cursor after inserting file mention
- **Runs during idle:** NO -- triggered only by user action
- **Severity:** NONE

### 3b. AskUserQuestionPanel mount animation
- **File:** `src/components/chat/tools/components/InteractiveRenderers/AskUserQuestionPanel.tsx:22`
- **Purpose:** One-shot RAF to set `mounted` state for entry animation
- **Runs during idle:** NO -- one-shot on mount
- **Severity:** NONE

### 3c. Shell terminal focus
- **File:** `src/components/shell/view/Shell.tsx:166`
- **Purpose:** One-shot RAF to focus terminal on mount
- **Runs during idle:** NO -- one-shot on mount
- **Severity:** NONE

No RAF loops found. All are one-shot.

---

## 4. CSS Animations

### 4a. @keyframes spin (infinite)
- **File:** `src/index.css:6,767`
- **Classes:** `.animate-spin`, `.loading-spinner`
- **Duration:** 1s linear infinite
- **Used by:** ~20 components for loading spinners (Loader2 icons, RefreshCw icons)
- **Runs during idle:** Only when loading state is active (conditionally rendered). Most are guarded by `isLoading`, `isRefreshing`, etc.
- **Severity:** LOW when idle -- spinners unmount when loading completes. But if ANY loading state stalls, a spinner runs forever.

### 4b. @keyframes settings-fade-in (finite)
- **File:** `src/index.css:909`
- **Duration:** 150ms ease-out (NOT infinite)
- **Severity:** NONE

### 4c. @keyframes search-flash (finite)
- **File:** `src/index.css:923`
- **Duration:** 4s ease-out (NOT infinite)
- **Severity:** NONE

### 4d. Tailwind animate-pulse (infinite)
- **Built-in Tailwind:** `animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite`
- **Usages found:** ~12 occurrences across components
- **ALWAYS-VISIBLE idle offenders:**
  - `src/components/sidebar/view/subcomponents/SidebarFooter.tsx:46,67` -- version update notification dot. Pulses PERMANENTLY if update available.
  - `src/components/sidebar/view/subcomponents/SidebarCollapsed.tsx:74` -- same update dot in collapsed sidebar
  - `src/components/sidebar/view/subcomponents/SidebarSessionItem.tsx:69` -- active session indicator. Pulses while session is "active" (could be long-lived).
  - `src/components/settings/view/tabs/agents-settings/AgentListItem.tsx:99` -- agent status dot, pulses when connecting
- **Conditionally rendered (OK):**
  - `SubagentContainer.tsx:76` -- only during subagent execution
  - `AssistantThinkingIndicator.tsx:23-27` -- only during AI response
  - `MicButtonView.tsx:24,57` -- only during recording/transcription
  - `SidebarProjectSessions.tsx:41-44` -- only during session loading skeleton
  - `GitPanelHeader.tsx` -- only during git operations
- **Severity:** MEDIUM -- the sidebar update dot and active session dot pulse infinitely during idle if conditions are met

### 4e. Tailwind animate-ping (infinite)
- **Built-in Tailwind:** `animation: ping 1s cubic-bezier(0, 0, 0.2, 1) infinite`
- **Usages:**
  - `src/components/chat/view/subcomponents/ClaudeStatus.tsx:123` -- green dot ping during AI response. Guarded by `isLoading`.
  - `src/components/mic-button/view/MicButtonView.tsx:78,82` -- recording/transcription indicator. Guarded by state.
  - `src/components/plugins/view/PluginSettingsTab.tsx:41` -- plugin running indicator. Active while plugin process runs.
- **Severity:** LOW -- all are conditionally rendered

### 4f. Tailwind animate-bounce (infinite)
- **Built-in Tailwind:** `animation: bounce 1s infinite`
- **Usages:**
  - `src/components/auth/view/AuthLoadingScreen.tsx:21` -- loading dots during auth. Only during auth flow.
- **Severity:** NONE during normal idle

### 4g. prefers-reduced-motion
- **File:** `src/index.css:220-229`
- **Status:** RESPECTED -- all animations forced to 0.01ms with iteration-count: 1 when `prefers-reduced-motion: reduce` is set
- **This is good.** Users can disable all animations via OS setting.

---

## 5. Network Timers

### 5a. WebSocket reconnection loop
- **File:** `src/contexts/WebSocketContext.tsx:86-89`
- **Interval:** 3000ms recursive setTimeout
- **Runs during idle:** Only when disconnected. When connected, WebSocket is event-driven (no polling/ping from client side).
- **No client-side keepalive/ping found.** The server has no `ws.ping()` interval either (checked `server/index.js` and all server files).
- **Severity:** LOW when connected, MEDIUM when disconnected

### 5b. Version check HTTP polling
- **File:** `src/hooks/useVersionCheck.ts:83`
- **Interval:** 300000ms (5 min)
- **Target:** GitHub API `https://api.github.com/repos/{owner}/{repo}/releases/latest`
- **Runs during idle:** YES
- **Severity:** LOW

---

## 6. Observers

### 6a. ResizeObserver -- Editor sidebar
- **File:** `src/components/code-editor/view/EditorSidebar.tsx:71`
- **Purpose:** Detects parent element width changes for responsive layout
- **Runs during idle:** Passive (event-driven, fires only on actual resize)
- **Cleaned up:** YES (disconnect in cleanup)
- **Severity:** NONE

### 6b. ResizeObserver -- Main content header
- **File:** `src/components/main-content/view/subcomponents/MainContentHeader.tsx:31`
- **Purpose:** Updates scroll state for header shadow
- **Runs during idle:** Passive
- **Cleaned up:** YES
- **Severity:** NONE

### 6c. ResizeObserver -- Shell terminal
- **File:** `src/components/shell/hooks/useShellTerminal.ts:222`
- **Purpose:** Refits terminal dimensions on container resize (debounced 50ms)
- **Runs during idle:** Passive
- **Cleaned up:** YES
- **Severity:** NONE

### 6d. IntersectionObserver -- Message auto-expand
- **File:** `src/components/chat/view/subcomponents/MessageComponent.tsx:82`
- **Purpose:** Auto-expands tool use messages when they scroll into view
- **Runs during idle:** Passive (only fires on scroll intersection changes)
- **Cleaned up:** YES (disconnect in cleanup)
- **Severity:** NONE

No MutationObservers found.

---

## 7. xterm.js Inherited Timers

### 7a. Cursor blink
- **File:** `src/components/shell/constants/constants.ts:16`
- **Setting:** `cursorBlink: true`
- **Interval:** xterm.js default 600ms (setInterval internally in xterm.js core, inherited by WebGL addon)
- **Runs during idle:** YES -- whenever terminal is mounted and visible, cursor blinks via internal setInterval even when user is not typing
- **Cleared on idle:** NO -- runs as long as terminal is mounted
- **Disableable:** YES -- set `cursorBlink: false` in TERMINAL_OPTIONS
- **Severity:** MEDIUM -- constant 600ms timer + GPU redraws (WebGL addon). The WebGL addon renders cursor blink by redrawing the cursor cell each cycle.

### 7b. WebGL addon render loop
- **File:** `src/components/shell/hooks/useShellTerminal.ts:100`
- **Addon:** `@xterm/addon-webgl` loaded at line 100
- **Behavior:** The WebGL addon does NOT use a requestAnimationFrame loop. It renders on-demand (on data, resize, blink). However, cursor blink triggers periodic WebGL redraws at 600ms intervals.
- **Severity:** Tied to cursor blink above

---

## 8. React useEffect with Intervals

All setInterval usages are inside useEffect hooks. Already catalogued above:

| Hook | File | Interval | Idle? |
|---|---|---|---|
| useSidebarController (sort poll) | useSidebarController.ts:177 | 1000ms | YES |
| useSidebarController (clock) | useSidebarController.ts:124 | 60000ms | YES |
| ClaudeStatus (elapsed) | ClaudeStatus.tsx:68 | 1000ms | NO |
| ClaudeStatus (animation) | ClaudeStatus.tsx:81 | 500ms | NO |
| useVersionCheck | useVersionCheck.ts:83 | 300000ms | YES |

---

## Idle Activity Profile

When the application is fully idle (no AI response, no loading, no user interaction, terminal visible):

| Source | Frequency | GPU Impact |
|---|---|---|
| Sidebar sort-order poll | 1 Hz | React re-render if changed |
| Sidebar clock update | 0.017 Hz | React re-render (sidebar) |
| Version check HTTP | 0.003 Hz | Network only |
| xterm cursor blink | 1.67 Hz | WebGL cell redraw |
| animate-pulse (sidebar dots) | 0.5 Hz CSS | GPU composite layer |
| WebSocket (connected) | 0 Hz | Event-driven only |
| ResizeObservers | 0 Hz | Event-driven only |

**Total idle timers firing: ~3.2 events/second** (dominated by cursor blink and sort-order polling)

---

## Recommendations for Display Strain Reduction

1. **Remove sort-order polling** (useSidebarController.ts:177) -- the `storage` event listener already handles cross-tab sync. The 1Hz poll is redundant.
2. **Set `cursorBlink: false`** (constants.ts:16) -- eliminates the highest-frequency idle timer and its WebGL redraws.
3. **Gate version check on visibility** -- wrap the interval in a `document.visibilityState === 'visible'` check, or use `document.addEventListener('visibilitychange')` to pause/resume.
4. **Replace animate-pulse on persistent elements** with a static indicator -- the sidebar update dot and active session dot pulse forever. A static colored dot conveys the same information without perpetual animation.
5. **prefers-reduced-motion is respected** (index.css:220) -- this is already correct and will disable all CSS animations for users who enable the OS-level setting.
