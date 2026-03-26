# claude-code-remote - Static Analysis Findings

**Repo:** github.com/buckle42/claude-code-remote
**Version:** N/A (script collection, no versioned package)
**License:** Not specified

---

## 1. Tech Stack

- **Language:** Bash (scripts), Python 3 (voice wrapper)
- **Framework:** FastAPI + Uvicorn (Python web server)
- **Terminal serving:** ttyd (external binary, installed via Homebrew)
- **Session persistence:** tmux
- **Network:** Tailscale VPN (WireGuard)
- **Platform:** macOS only (uses `caffeinate`, `launchd`, Homebrew paths)
- **Dependencies:** ttyd, tmux, FastAPI, uvicorn, python-multipart, Tailscale

No npm, no Node.js, no frontend build system. This is a shell script + Python project.

---

## 2. Rendering Pipeline

**Terminal emulator via ttyd's built-in xterm.js:**

- ttyd serves a web page with an embedded xterm.js terminal
- The voice wrapper (`voice-wrapper.py`) embeds ttyd in an `<iframe>`
- ttyd handles all terminal rendering internally -- its own bundled xterm.js
- The wrapper page is server-rendered HTML from a Python f-string (no build step, no framework)
- Image compression uses `<canvas>` + `getContext('2d')` for client-side photo resizing before upload

**Renderer classification:** ttyd's embedded xterm.js (canvas-based terminal emulator inside iframe).

---

## 3. Update/Flush Mechanism

**ttyd WebSocket push:**

- ttyd internally uses WebSocket to stream PTY output to its xterm.js frontend
- The voice wrapper does NOT intercept or process terminal output -- it only sends input
- Text input goes: `fetch('/send')` -> FastAPI -> `tmux send-keys` -> tmux session -> ttyd picks up output
- No RAF, no batching -- ttyd handles its own rendering pipeline internally

**Voice wrapper updates:**
- Vanilla JavaScript, no framework -- direct DOM manipulation
- `visibilitychange` event listener reloads iframe on tab wake (auto-reconnect)

---

## 4. Claude Code Integration

**No direct integration -- fully indirect via tmux:**

- User types in the text input or dictates via iOS
- Text is sent via REST POST to `/send` endpoint
- FastAPI handler runs `tmux send-keys -t claude -l <text>` then `tmux send-keys Enter`
- Claude Code CLI runs inside the tmux session as a regular interactive process
- The wrapper has NO knowledge of Claude Code -- it is a generic terminal remote access tool
- "New session" button sends `/exit` then `claude` after 1.5s delay
- "Resume" button sends `/exit` then `claude --resume` after 1.5s delay

**No SDK, no subprocess management, no streaming parsing.** Claude Code is just a program running in tmux.

---

## 5. Transport

- **REST API** (FastAPI) for input injection: `/send`, `/key`, `/copy`, `/upload`
- **WebSocket** used internally by ttyd (between ttyd server and its xterm.js client in iframe)
- The voice wrapper page communicates with FastAPI via `fetch()` calls
- All services bind exclusively to Tailscale IP -- not exposed to LAN or internet

---

## 6. PTY

**Yes, via tmux + ttyd:**

- tmux creates the PTY session (`tmux new-session -s claude`)
- ttyd attaches to the tmux session via `tmux-attach.sh` wrapper script
- ttyd serves the PTY over WebSocket to the browser
- The voice wrapper does NOT spawn its own PTY -- it injects keystrokes into the existing tmux session via `tmux send-keys`

---

## 7. Font Rendering

**ttyd terminal (configured in start script):**
- Font family: `'Menlo, Monaco, Consolas, monospace, Apple Color Emoji, Segoe UI Emoji'`
- Font size: 14px
- Line height: 1.2
- Cursor: block, blinking
- Scrollback: 10000

**Voice wrapper UI:**
- Quick-key buttons: `'Menlo', monospace` at 14px
- Text input: `-apple-system, system-ui, sans-serif` at 16px
- Copy overlay textarea: `Menlo, monospace` at 14px, line-height 1.4
- Dark theme: `#1a1a1a` background throughout

---

## 8. Strain Assessment: MEDIUM

- ttyd's xterm.js uses canvas rendering (not WebGL) -- CPU-driven glyph painting
- iframe embedding adds a layer of composition (browser must composite iframe content)
- Auto-reconnect reloads entire iframe on visibility change -- causes flash/redraw
- 14px Menlo at 1.2 line height is compact but readable
- Dark theme throughout reduces overall brightness
- No anti-flicker measures -- raw terminal output appears character-by-character as tmux delivers it
- The copy overlay uses a full-screen textarea with monospace text -- acceptable for brief use
- Risk factors: canvas-based xterm.js rendering (not GPU-accelerated), iframe reload flicker on phone wake, no configurable font size from the UI

---

## 9. Key Files

1. **`scripts/voice-wrapper.py`** -- Complete FastAPI app: HTML UI, REST endpoints for tmux injection, file upload
2. **`scripts/start-remote-cli.sh`** -- Service orchestrator: starts ttyd, voice wrapper, caffeinate, watchdog
3. **`scripts/tmux-attach.sh`** -- tmux session creation/attachment with env var cleanup
4. **`scripts/stop-remote-cli.sh`** -- Clean shutdown of all services
5. **`CLAUDE.md`** -- Setup instructions context for Claude Code assistance
