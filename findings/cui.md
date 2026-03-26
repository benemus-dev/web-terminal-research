# CUI (cui-server) - Static Analysis Findings

**Repo:** github.com/bmpixel/cui
**Version:** 0.6.3
**License:** Apache-2.0

---

## 1. Tech Stack

- **Language:** TypeScript (full stack)
- **Framework:** React 18 + Vite 7
- **Server:** Express (with vite-express for dev)
- **Styling:** Tailwind CSS 4, shadcn/ui pattern (Radix UI primitives)
- **Build:** Vite, TypeScript, tsc-alias for path mapping
- **Database:** better-sqlite3 (conversation cache)
- **Testing:** Vitest with happy-dom, 90%+ coverage target
- **Key deps:** `@anthropic-ai/claude-code` (CLI package), `@anthropic-ai/sdk`, `@modelcontextprotocol/sdk`, `prismjs`, `react-markdown`, `react-diff-viewer-continued`, `eventsource`, `pino` (logging), `@google/genai`
- **PWA support:** `vite-plugin-pwa` + service worker (`sw.ts`)

Multi-provider: Claude Code CLI, Anthropic SDK direct, Google Gemini.

---

## 2. Rendering Pipeline

**Chat UI only -- NOT a terminal emulator.**

- Pure React DOM rendering -- no xterm.js, no canvas terminal, no WebGL
- Messages rendered via `react-markdown` for markdown content
- Code blocks via `prism-react-renderer` / `prismjs` for syntax highlighting
- Diff viewing via `react-diff-viewer-continued`
- Canvas usage limited to `WaveformVisualizer` component (audio recording visualization only)
- JSON viewer component for tool call inspection
- Tool rendering components for displaying Claude tool use results

**Renderer classification:** React DOM only. Chat/messaging paradigm.

---

## 3. Update/Flush Mechanism

**Server-Sent Events (SSE) streaming with fetch ReadableStream:**

- Server: `StreamManager` sets `Content-Type: text/event-stream` headers, writes SSE events to `Response` objects
- Client: `useStreaming` hook uses `fetch()` + `response.body.getReader()` (ReadableStream API) -- NOT native EventSource
- Heartbeat every 30s to keep connections alive
- Auto-reconnect on disconnect (with `shouldReconnect` flag)
- React state updates from stream events trigger reconciler-driven DOM updates
- `streamEventMapper.ts` transforms raw stream events into UI-consumable format

**No RAF or explicit frame batching.** React handles DOM update coalescing via its reconciler.

---

## 4. Claude Code Integration

**CLI subprocess -- spawns `claude` binary:**

- `ClaudeProcessManager` uses `spawn()` from `child_process` to launch the Claude CLI executable
- Finds claude binary at `node_modules/.bin/claude` (installed via `@anthropic-ai/claude-code` npm package)
- Falls back to searching PATH if not found in node_modules
- Streams output via JSONL format: `JsonLinesParser` parses newline-delimited JSON from stdout
- Each conversation gets a unique `streamingId`
- Supports resume via `--resume` flag with session ID

**NOT using the SDK programmatically** -- it shells out to the CLI binary and parses its `--output-format stream-json` output.

**Also has:** `@anthropic-ai/sdk` direct integration and Google Gemini via `@google/genai` for alternative providers.

**MCP integration:** `@modelcontextprotocol/sdk` for tool server configuration, with MCP config generator service.

---

## 5. Transport

- **SSE (Server-Sent Events)** for Claude conversation streaming (server to client)
- **REST API** for conversation management, file system operations, permissions, commands
- No WebSocket used
- `streaming.routes.ts` exposes `/api/stream/:streamingId` endpoint

---

## 6. PTY

**No PTY.** Does not spawn a pseudo-terminal.

- Uses `child_process.spawn()` directly -- piped stdin/stdout/stderr
- No terminal emulation -- the output is structured JSONL, not raw terminal escape sequences
- No `node-pty` dependency

---

## 7. Font Rendering

**System font stack for UI:**
```
body: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen', 'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif
```

**Monospace for code:**
```
--font-mono: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace
```

**Font sizes (CSS variables):**
- xs: 11px, sm: 12px, base: 13px, md: 14px, lg: 16px
- Base font for code/tool output: 13px (quite small)

**Font smoothing:** `-webkit-font-smoothing: antialiased` enabled.

---

## 8. Strain Assessment: LOW

- Pure DOM rendering -- no canvas, no WebGL, no repaint flicker
- SSE streaming with React reconciler batching -- smooth incremental updates
- Well-structured CSS theme variables with proper dark mode support
- System fonts with antialiasing -- subpixel-rendered text
- 13px base font size is on the small side but standard for code UIs
- No terminal emulator means no cursor blink animation, no rapid character-by-character rendering
- Heartbeat mechanism prevents connection drops (no reconnection flicker)
- Risk factor: 13px monospace might cause strain for extended reading; no user-configurable font size visible in theme

---

## 9. Key Files

1. **`src/services/claude-process-manager.ts`** -- Claude CLI subprocess management, JSONL parsing, conversation lifecycle
2. **`src/services/stream-manager.ts`** -- SSE streaming to browser clients, heartbeat, multi-client support
3. **`src/web/chat/hooks/useStreaming.ts`** -- Client-side ReadableStream consumer for SSE
4. **`src/web/chat/styles/theme.css`** -- Complete CSS theme with font definitions and color variables
5. **`src/cui-server.ts`** -- Server entry point, route registration, service wiring
