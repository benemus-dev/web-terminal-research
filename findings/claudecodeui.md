# claudecodeui - Static Analysis Findings

**Repo:** github.com/siteboon/claudecodeui
**Version:** 1.26.3
**License:** GPL-3.0

---

## 1. Tech Stack

- **Language:** TypeScript (frontend), JavaScript (server)
- **Framework:** React 18 + Vite 7
- **Server:** Express + WebSocket (`ws`)
- **Styling:** Tailwind CSS 3, PostCSS
- **Build:** Vite, TypeScript
- **Database:** better-sqlite3 (session persistence)
- **Key deps:** `@anthropic-ai/claude-agent-sdk`, `@xterm/xterm`, `node-pty`, `@uiw/react-codemirror`, `react-markdown`, `react-syntax-highlighter`, `@openai/codex-sdk`, i18next

Multi-provider: supports Claude, Cursor, OpenAI Codex, and Gemini CLI.

---

## 2. Rendering Pipeline

**Dual-mode UI:**

**A) Terminal emulator (Shell tab):**
- xterm.js v5.5 with WebGL addon (`@xterm/addon-webgl`) as primary renderer
- Falls back to Canvas if WebGL unavailable: `nextTerminal.loadAddon(new WebglAddon())` with try/catch
- FitAddon for auto-sizing, WebLinksAddon for clickable URLs
- Renders into a `<div>` container via `nextTerminal.open(terminalContainerRef.current)`

**B) Chat UI (Chat tab):**
- Standard React DOM rendering
- Messages rendered via `react-markdown` + `react-syntax-highlighter`
- Code blocks via CodeMirror (`@uiw/react-codemirror`) with minimap
- No canvas/WebGL in chat mode -- pure DOM

**Renderer classification:** Hybrid -- xterm.js WebGL canvas for terminal, React DOM for chat.

---

## 3. Update/Flush Mechanism

**Terminal mode:**
- WebSocket push: server sends PTY output directly to client WebSocket, xterm.js writes it immediately
- No RAF batching visible -- xterm.js handles its own internal frame scheduling
- Resize events debounced at 50ms (`TERMINAL_RESIZE_DELAY_MS`)

**Chat mode (Claude SDK path):**
- SDK streams events via WebSocket to client
- React state updates trigger reconciler-driven re-renders
- Messages appended to state array, React batches DOM updates

---

## 4. Claude Code Integration

**Two distinct integration paths:**

**Path A -- Claude Agent SDK (chat mode):**
- `@anthropic-ai/claude-agent-sdk` imported in `server/claude-sdk.js`
- Uses `import { query } from '@anthropic-ai/claude-agent-sdk'` -- direct SDK call, no subprocess
- Streams events back to frontend via WebSocket
- Supports tool approval flow with timeout (55s default)
- Session management with abort capability via `AbortController`

**Path B -- PTY shell (terminal mode):**
- `node-pty` spawns a real shell process
- User types `claude` in the terminal manually -- Claude Code CLI runs inside the PTY
- Server bridges PTY I/O to browser WebSocket
- PTY sessions tracked in `ptySessionsMap` with reconnection support

**Streaming:** WebSocket (`ws` library). Server sends JSON messages with `type` field. Client parses and dispatches.

---

## 5. Transport

- **WebSocket** for both terminal PTY I/O and chat SDK streaming
- Express REST API for project management, file operations, settings, git
- No SSE used

---

## 6. PTY

**Yes, spawns real PTY via node-pty.**

```
shellProcess = pty.spawn(shell, shellArgs, { ... })
```

- PTY sessions keyed by `${projectPath}_${sessionId}_${command}`
- Supports reconnection to existing PTY sessions
- Separate from Claude SDK path -- the PTY is for the "Shell" tab experience
- `postinstall` script runs `fix-node-pty.js` for platform compatibility

---

## 7. Font Rendering

**Terminal (xterm.js):**
- Font family: `'Menlo, Monaco, "Courier New", monospace'`
- Font size: 14px
- Cursor blink: enabled
- Tab stop width: 4
- Scrollback: 10000 lines
- Background: `#1e1e1e`, Foreground: `#d4d4d4`
- Full 16-color + extended ANSI color theme defined

**Chat mode:**
- Uses Tailwind defaults (system font stack)
- Code blocks via CodeMirror with One Dark theme

---

## 8. Strain Assessment: LOW-MEDIUM

- xterm.js WebGL renderer is GPU-accelerated -- smooth glyph rendering with minimal CPU repaint
- Canvas fallback available if WebGL fails
- Dark theme with good contrast (#d4d4d4 on #1e1e1e)
- Standard monospace fonts (Menlo/Monaco) with 14px size -- readable
- WebSocket push means no polling flicker
- Resize debounce prevents layout thrash
- The chat mode is pure DOM -- React reconciler handles batching
- Risk factor: WebGL addon can occasionally drop frames on weak GPUs; no explicit frame rate limiting visible

---

## 9. Key Files

1. **`server/index.js`** -- Main server: Express + WebSocket + PTY spawn + multi-provider routing
2. **`server/claude-sdk.js`** -- Claude Agent SDK integration, session management, tool approval
3. **`src/components/shell/hooks/useShellTerminal.ts`** -- xterm.js initialization, WebGL addon loading
4. **`src/components/shell/hooks/useShellConnection.ts`** -- WebSocket connection to PTY backend
5. **`src/components/shell/constants/constants.ts`** -- Terminal options: font, colors, theme
