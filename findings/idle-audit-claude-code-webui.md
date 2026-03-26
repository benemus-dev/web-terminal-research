# Idle Activity Audit: claude-code-webui

**Date:** 2026-03-26
**Scope:** `frontend/src/` and `backend/` (excluding node_modules)
**Method:** Static code analysis only

## Summary

| Category | Count | Idle Impact |
|---|---|---|
| setInterval | 1 | MINOR (60s, conditional) |
| setTimeout (recursive) | 1 pattern | NONE (demo mode only) |
| setTimeout (one-shot) | ~10 | NONE (event-driven, not idle) |
| requestAnimationFrame | 0 | -- |
| CSS @keyframes | 0 (custom) | -- |
| Tailwind animate-* classes | 4 uses | MINOR (only during loading) |
| CSS transitions | ~25 uses | NONE (hover-only, not idle) |
| Network polling/keepalive | 0 | -- |
| MutationObserver | 0 | -- |
| ResizeObserver | 0 | -- |
| IntersectionObserver | 0 | -- |
| WebSocket | 0 | -- |
| React useEffect with timers | 1 | MINOR (same as setInterval #1) |

**IDLE SCORE: CALM**

Zero continuous activity at idle. The only repeating timer is a 60-second relative-timestamp updater, and it only runs when timestamps are in "relative" mode (which is not the default). All CSS animations (Tailwind built-in `animate-spin`, `animate-pulse`) are conditional on active loading state and stop when idle.

---

## Detailed Findings

### 1. setInterval

**[F1] TimestampComponent relative-time updater**
- **File:** `frontend/src/components/TimestampComponent.tsx:32`
- **Code:** `const interval = setInterval(updateTime, 60000);`
- **Interval:** 60,000ms (1 minute)
- **Purpose:** Updates relative timestamps ("2 minutes ago") on chat messages
- **Runs during idle?** Only if `mode === "relative"`. Default mode is `"absolute"`, so this does NOT run in the default configuration. When active, one interval per message component using relative mode.
- **Cleared?** Yes, `return () => clearInterval(interval)` in useEffect cleanup (line 33)
- **Disableable?** Yes, use `mode="absolute"` (the default)

No other `setInterval` calls exist in frontend/src or backend.

### 2. setTimeout (recursive / self-rescheduling)

**[F2] Demo typing animation**
- **File:** `frontend/src/hooks/useDemoAutomation.ts:163`
- **Code:** `typingIntervalRef.current = setTimeout(typeNextCharacter, delay + pauseDelay);`
- **Also at:** lines 395, 453, 456, 473, 579, 623
- **Pattern:** setTimeout that calls itself to type next character
- **Interval:** Variable (~20-100ms per character)
- **Runs during idle?** NO. Only runs on the `/demo` page during active demo playback. Not loaded in normal chat usage.
- **Cleared?** Yes, `clearTimeout` on refs in cleanup
- **Disableable?** Yes, don't visit `/demo`

### 3. setTimeout (one-shot, non-recursive)

**[F3] DemoPage style injection cleanup** -- `DemoPage.tsx:76` -- 100ms, one-shot, demo only
**[F4] DemoPage button click delays** -- `DemoPage.tsx:264-276` -- 200ms each, user-triggered
**[F5] ChatInput IME composition** -- `ChatInput.tsx:148` -- `setTimeout(..., 0)`, race condition fix, event-triggered

None of these run during idle. All are event-driven one-shots.

### 4. requestAnimationFrame

**None found.** Zero RAF usage anywhere in the codebase.

### 5. CSS Animations

No custom `@keyframes` definitions in any CSS file. The only CSS file is `frontend/src/index.css` (3 lines: Tailwind import + dark mode variant). No `tailwind.config.*` file exists (uses Tailwind v4 with CSS-based config).

**Tailwind built-in animation classes used:**

**[F6] animate-spin (loading spinner)**
- `MessageComponents.tsx:395` -- 4x4 spinner in LoadingComponent ("Thinking...")
- `ChatPage.tsx:526` -- 8x8 spinner ("Loading conversation history...")
- `HistoryView.tsx:58` -- 8x8 spinner ("Loading project/conversations...")
- **Runs during idle?** NO. Only rendered when `isLoading === true`. Removed from DOM when loading completes.
- **Animation:** Tailwind `animate-spin` = `@keyframes spin { to { transform: rotate(360deg) } }` at 1s linear infinite. GPU-composited (transform-only).

**[F7] animate-pulse (thinking text)**
- `MessageComponents.tsx:396` -- "Thinking..." text opacity pulse
- **Runs during idle?** NO. Only rendered inside LoadingComponent during active streaming.
- **Animation:** Tailwind `animate-pulse` = `@keyframes pulse { 50% { opacity: .5 } }` at 2s cubic-bezier infinite.

### 6. CSS Transitions

~25 instances of `transition-colors`, `transition-all` with `duration-200` or `duration-300`. All are hover/focus state transitions on buttons and interactive elements. These are event-driven (mouse enter/leave) and produce zero activity at idle. No infinite or auto-triggering transitions.

### 7. Network Timers / Streaming

**[F8] NDJSON streaming (chat)**
- **File:** `ChatPage.tsx:185-227` (frontend), `backend/handlers/chat.ts:104-132` (backend)
- **Pattern:** `response.body.getReader()` with `while(true) { reader.read() }` loop
- **Runs during idle?** NO. Only active during a chat request. The ReadableStream is demand-driven (backpressure). No keepalive pings, no polling, no reconnection timers.
- **Backend:** `Connection: keep-alive` header on NDJSON response, but this is HTTP-level and only active during streaming. No setTimeout/setInterval in backend code at all.

**No polling anywhere.** All data fetches (projects list, conversation history) are one-shot `fetch()` calls triggered by navigation/mount, not recurring.

### 8. Observers

**Zero.** No MutationObserver, ResizeObserver, or IntersectionObserver usage anywhere in frontend/src or backend.

### 9. React useEffect Summary

All useEffects found are either:
- **Mount-only** (`[]` deps): Settings initialization (`SettingsContext.tsx:13`), project loading (`ChatPage.tsx:388`), keyboard listeners (`SettingsModal.tsx:19`)
- **State-driven** (specific deps): Theme application, auto-scroll on message change, textarea height adjustment on input change
- **The only timer-containing useEffect** is [F1] above (TimestampComponent)

No useEffect runs any recurring work at idle.

---

## Architecture Notes

- **No WebSocket.** Despite CHANGELOG mentioning "WebSocket-like streaming," actual implementation is HTTP fetch + NDJSON ReadableStream. Clean request/response lifecycle.
- **No service worker** or background sync detected.
- **No localStorage polling** or cross-tab communication timers.
- **Textarea auto-resize** (`ChatInput.tsx:80-91`) runs on `[input]` dependency only, not on timer.
- **Auto-scroll** (`ChatMessages.tsx:53-55`) runs on `[messages]` dependency only, not on timer.

## What Could Be Wrong

- Tailwind v4 may inject additional animation keyframes via its runtime that aren't visible in source CSS. Would need to inspect computed styles in browser DevTools.
- Third-party dependencies (React Router, Heroicons) were not audited -- they could theoretically register observers or timers, though this is unlikely for these libraries.
- The `capture-screenshots.ts` script has a 2000ms setTimeout but this is a build/CI tool, not runtime code.
