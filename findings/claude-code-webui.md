# claude-code-webui Analysis

**Repository:** https://github.com/sugyan/claude-code-webui
**Version:** 0.1.56
**Author:** sugyan
**License:** MIT

## 1. Tech Stack

### Backend (`backend/package.json`)
- **Runtime:** Node.js >= 20 (also supports Deno)
- **Framework:** Hono (v4.8.5) with `@hono/node-server`
- **Key dependency:** `@anthropic-ai/claude-code` v1.0.108 (fixed, not caret)
- **CLI:** commander v14
- **Logging:** @logtape/logtape + @logtape/pretty
- **Build:** esbuild (v0.25.6) via custom `scripts/build-bundle.js`
- **Dev tooling:** tsx, vitest, eslint, prettier, typescript v5.9

### Frontend (`frontend/package.json`)
- **Framework:** React 19.1 + React DOM 19.1
- **Router:** react-router-dom v7.6.2
- **Build:** Vite v7.1.6 with `@vitejs/plugin-react-swc`
- **CSS:** TailwindCSS v4 via `@tailwindcss/vite` plugin
- **Icons:** @heroicons/react v2.2
- **Date:** dayjs v1.11
- **Testing:** vitest, @testing-library/react, @playwright/test, jsdom
- **Types:** `@anthropic-ai/claude-code` v1.0.108 (devDependency for SDK types)

### Shared (`shared/types.ts`)
- TypeScript type definitions shared between frontend and backend
- Defines: StreamResponse, ChatRequest, AbortRequest, ProjectInfo, ConversationHistory

## 2. Rendering Pipeline

**Classification: DOM_ONLY (React component tree)**

### No Terminal Emulation Confirmed
- Zero references to xterm, xterm.js, terminal emulator, canvas-based rendering, or WebGL anywhere in frontend source code.
- The only "terminal" match is in `package-lock.json` as a transitive dependency name, not used in application code.

### How Output Is Rendered
Pure React component hierarchy rendering Claude SDK messages as styled HTML/DOM elements:

1. **`ChatMessages`** component receives an `AllMessage[]` array and maps each message to a typed React component via `renderMessage()`.
2. Message type discrimination uses type guard functions (`isChatMessage`, `isToolMessage`, `isThinkingMessage`, etc.).
3. Each message type has a dedicated component:
   - `ChatMessageComponent` - user/assistant text messages (renders in `<pre>` with `whitespace-pre-wrap font-mono`)
   - `SystemMessageComponent` - init, result, error (collapsible details)
   - `ToolMessageComponent` - tool invocations
   - `ToolResultMessageComponent` - tool results with preview (Edit diffs, Bash stdout/stderr, Grep matches)
   - `PlanMessageComponent` - plan mode output
   - `ThinkingMessageComponent` - extended thinking (collapsible, default expanded)
   - `TodoMessageComponent` - todo list with status icons
   - `LoadingComponent` - spinner with "Thinking..." pulse animation

4. All styling is TailwindCSS utility classes. No custom CSS beyond the 3-line `index.css` that imports Tailwind.

## 3. Update/Flush Mechanism

**Classification: REACT_STATE_DRIVEN_NDJSON_STREAM**

### Streaming Flow
1. Frontend sends POST to `/api/chat` with JSON body.
2. Backend returns `Content-Type: application/x-ndjson` (newline-delimited JSON) as a `ReadableStream`.
3. Frontend reads the stream using `response.body.getReader()` in a `while(true)` loop.
4. Each chunk is decoded, split by newlines, and each line is parsed as JSON.
5. `processStreamLine()` dispatches to `UnifiedMessageProcessor` which calls React state setters (`addMessage`, `updateLastMessage`, `setCurrentAssistantMessage`).
6. React re-renders the component tree on each state update.

### Key Detail
- This is NOT SSE (no `EventSource`). It is a raw fetch stream with NDJSON framing.
- No WebSocket either. Pure HTTP streaming via ReadableStream/getReader.
- Auto-scroll via `useEffect` on `[messages]` calling `scrollIntoView({ behavior: "smooth" })`.

## 4. Claude Code Integration

**Integration method: Official SDK (`@anthropic-ai/claude-code`)**

### Backend invocation (`backend/handlers/chat.ts`)
```typescript
import { query } from "@anthropic-ai/claude-code";

for await (const sdkMessage of query({
  prompt: processedMessage,
  options: {
    abortController,
    executable: "node",
    pathToClaudeCodeExecutable: cliPath,
    resume: sessionId,        // session continuity
    allowedTools,
    cwd: workingDirectory,
    permissionMode,
  },
})) {
  yield { type: "claude_json", data: sdkMessage };
}
```

- Uses the **SDK's `query()` function**, not raw CLI subprocess spawning.
- The SDK internally manages the Claude Code CLI process.
- `cliPath` is auto-detected via `backend/cli/validation.ts` (`detectClaudeCliPath()`).
- Abort support via standard `AbortController` passed to SDK options.
- Session continuity: first message creates session, subsequent messages pass `session_id` via `options.resume`.

### Frontend type usage
- Frontend imports SDK types (`SDKMessage`) from `@anthropic-ai/claude-code` (devDependency) for type checking only.
- Message type discrimination: system, assistant, result, user.

## 5. Transport

**Classification: HTTP Streaming (NDJSON over fetch)**

- **Frontend -> Backend:** HTTP POST with JSON body (`/api/chat`)
- **Backend -> Frontend:** NDJSON stream (`application/x-ndjson`) via ReadableStream
- **Abort:** HTTP POST to `/api/abort/:requestId`
- **Other endpoints:** REST (GET `/api/projects`, GET `/api/projects/:name/histories`)
- **No WebSocket.** No SSE (EventSource). No socket.io.
- Backend runs on Hono framework (port 8080), frontend on Vite dev server (port 3000) with proxy.

## 6. Font Rendering

- **Body text:** TailwindCSS defaults (system font stack via `font-sans` implicitly)
- **Code/output content:** `font-mono` (TailwindCSS monospace stack: `ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace`)
- **No custom fonts loaded.** No @font-face declarations. No Google Fonts.
- Message content rendered in `<pre className="whitespace-pre-wrap text-sm font-mono leading-relaxed">`
- Tool results and collapsible details use `text-xs font-mono leading-relaxed`
- No tailwind.config file found -- uses TailwindCSS v4 defaults with Vite plugin.

## 7. Strain Assessment

**LOW**

Rationale:
- Pure DOM rendering with standard React reconciliation -- no canvas, no WebGL, no compositing layers.
- TailwindCSS utility classes produce straightforward CSS -- no complex animations beyond a spinner and pulse.
- `font-mono` with `leading-relaxed` and `whitespace-pre-wrap` is readable.
- Dark mode support (`dark:` variants) reduces strain in low-light conditions.
- Auto-scroll uses `behavior: "smooth"` -- not jarring.
- No rapid DOM mutations during streaming -- React batches state updates.
- Message content is static once rendered (append-only list).
- The only dynamic update during streaming is appending/updating the current assistant message.

## 8. Key Files

1. **`/c/dev/web-terminal-research/claude-code-webui/backend/handlers/chat.ts`** - Claude Code SDK integration, NDJSON streaming, the core backend handler.
2. **`/c/dev/web-terminal-research/claude-code-webui/frontend/src/components/ChatPage.tsx`** - Main chat page with fetch streaming loop (`getReader` + NDJSON parsing).
3. **`/c/dev/web-terminal-research/claude-code-webui/frontend/src/components/MessageComponents.tsx`** - All message type React components (ChatMessage, ToolResult, Thinking, etc.).
4. **`/c/dev/web-terminal-research/claude-code-webui/frontend/src/hooks/streaming/useStreamParser.ts`** - Stream line parsing and SDK message dispatching.
5. **`/c/dev/web-terminal-research/claude-code-webui/frontend/src/components/chat/ChatMessages.tsx`** - Message list rendering with type-based dispatch and auto-scroll.

## Summary

claude-code-webui is a chat UI (not a terminal emulator) that wraps the official `@anthropic-ai/claude-code` SDK. The backend (Hono/Node.js) calls the SDK's `query()` function and streams SDK messages as NDJSON over HTTP. The frontend (React 19 + Vite + TailwindCSS) reads the stream via `fetch().body.getReader()`, parses each JSON line, and renders typed React components for each message kind. All rendering is DOM-based with no canvas, WebGL, or terminal emulation. Eye strain risk is low.
