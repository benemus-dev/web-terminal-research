# Chrome Compositor Redraw Audit Summary

The previous idle audit only checked JS timers. This audit checks what Chrome's rendering pipeline actually does -- CSS animations, compositor layers, backdrop-filter, contentEditable, transitions on always-visible elements.

**Key learning: Chrome's compositor is demand-driven, not always 60fps.** But certain CSS properties force it active continuously.

## Corrected Idle Rankings (Quietest to Noisiest)

| Rank | Project | JS Idle Score | Compositor Idle Score | Worst Hidden Trigger | Compositor Layers |
|------|---------|--------------|----------------------|---------------------|-------------------|
| 1 | **GoTTY** | MINOR (30s ping) | QUIETEST | 180ms overlay fade (rare) | 1-2 |
| 2 | **ttyd** | MINOR (5s ping) | VERY QUIET | WebGL canvas (single surface) | 2 |
| 3 | **WeTTY** | MINOR (3s ping) | VERY QUIET | 16x16px gear icon layer | 3-4 |
| 4 | **shellinabox** | BUSY (500ms setInterval) | QUIET | No transitions, no contentEditable, no CSS animations. JS cursor blink is a binary class toggle (no interpolation) | 2-3 |
| 5 | **DomTerm** | MINOR | MODERATE | contentEditable caret timer (Chrome internal, runs forever). CSS cursor blink stops after 30s | 3-4 |
| 6 | **xterm.js DOM** | MINOR | MODERATE | CSS cursor blink `infinite` for 5min. Custom scrollbar with opacity transitions. No `contain:` on rows | ~8 |
| 7 | **hterm** | CALM (JS) | NOISY | Cursor blink = setTimeout(100ms) + CSS `transition: opacity 100ms linear` = continuous compositor interpolation. contentEditable on `<x-screen>` | 4-6 |
| 8 | **CUI** | MINOR | MODERATE | `backdrop-blur-sm` on sticky footer. `transition-all` on composer. 40 scroll containers | 3-4 |
| 9 | **claude-code-webui** | CALM (JS) | NOISY | `backdrop-blur-sm` on main chat container (Gaussian blur every composite frame). 9x `transition-all`. Zero `prefers-reduced-motion` | 8-12 |
| 10 | **claudecodeui** | MODERATE | VERY NOISY | 399 transition/animation occurrences. 27 backdrop-filters (4-5 always-visible). Universal CSS transitions on `div, span, p, header`. 51 scroll containers. 91 box-shadows | 6-10 |

## The Three Biggest Hidden Triggers Found

### 1. `backdrop-filter: blur()` (worst offender)
Forces Chrome to read pixels behind the element and apply Gaussian blur on EVERY composite frame. Found on always-visible elements in:
- **claude-code-webui**: ChatMessages container + ChatInput textarea (the two largest viewport elements)
- **claudecodeui**: navigation, sidebar, header, composer (4-5 always-visible)
- **CUI**: sticky ConversationView footer

Projects that avoid it entirely: GoTTY, ttyd, WeTTY, shellinabox, DomTerm, hterm, xterm.js

### 2. `contentEditable` (silent killer)
Chrome's internal editing caret blink timer fires on any contentEditable element, even when:
- `caret-color: transparent` hides the caret visually
- The element is blurred (not focused)
- The JS cursor blink is disabled

Found in:
- **DomTerm**: caret `<span>` is contentEditable
- **hterm**: entire `<x-screen>` is contentEditable

Avoided by: shellinabox (uses hidden `<input>` instead), xterm.js (uses `<textarea>`)

### 3. CSS `transition` + frequent property changes
`transition: all` tells Chrome to monitor all 400+ CSS properties for changes. Even though it's inert when nothing changes, it prevents Chrome from optimizing style recalc (can't skip property checks).

Combined with cursor blink or hover states, transitions create continuous compositor interpolation:
- **hterm**: `transition: opacity 100ms` on cursor + 10Hz setTimeout = near-continuous compositing
- **claude-code-webui**: 9x `transition-all` on always-visible elements
- **claudecodeui**: universal transitions on every structural HTML element

## Universal Gap: No Project Uses CSS Containment

**Zero projects** use `contain:` or `content-visibility:` anywhere:

| Optimization | What it does | Who should use it | Who does |
|-------------|-------------|-------------------|---------|
| `contain: strict` | Isolates element from layout/paint of ancestors | Terminal row containers, scrollback | Nobody |
| `contain: layout paint` | Prevents layout recalc propagation | Sidebar, chat message list | Nobody |
| `content-visibility: auto` | Skips rendering of off-screen content entirely | Scrollback rows, old messages | Nobody |

This is the single biggest missed optimization. A terminal with 1000 scrollback rows forces Chrome to include all of them in layout recalc even though only ~24 are visible.

## Universal Gap: prefers-reduced-motion

| Project | Respects it? |
|---------|-------------|
| claudecodeui | YES (forces all animations to 0.01ms, iteration-count: 1) |
| All others | NO |

## Corrected "CALM" Assessment

My original audit called **claude-code-webui** "CALM -- zero idle activity." The corrected assessment:

**claude-code-webui is NOT calm.** It has:
- `backdrop-blur-sm` on the main chat container = Gaussian blur every composite frame
- 9x `transition-all` on always-visible elements = style recalc can't optimize
- Zero `prefers-reduced-motion` = no way to opt out
- Zero CSS containment = layout changes propagate to document root
- 8-12 compositor layers at idle

The truly quietest projects are the minimal terminal wrappers: **GoTTY** (1-2 layers, zero CSS effects), **ttyd** (2 layers, WebGL canvas is self-contained), and **WeTTY** (3-4 layers, minimal UI chrome).

## Chrome Debugging Commands

To verify idle redraw activity:
```
# In Chrome DevTools Console:
# Show paint rectangles (green flash on repaint)
# Rendering tab > Paint flashing

# Show compositor layers (orange borders)
# Rendering tab > Layer borders

# Show FPS meter (frames per second at idle should be 0)
# Rendering tab > Frame rendering stats

# Chrome tracing for compositor decisions:
# chrome://tracing > Record > Categories: cc.debug.scheduler, viz
# Key events at idle: DrawAndSwap, NeedsBeginFrameChanged, SetNeedsRedraw
```

## How to Fix Each Project

| Project | Fix | Visual Impact |
|---------|-----|---------------|
| claude-code-webui | Remove `backdrop-blur-sm` from ChatMessages/ChatInput. Replace `transition-all` with `transition-colors`. Add `@media (prefers-reduced-motion)` | Minimal -- blur is subtle, colors still animate |
| claudecodeui | Remove universal element transitions. Remove always-visible backdrop-filters. Already has `prefers-reduced-motion` | None with reduced-motion enabled |
| hterm | Replace cursor `transition: opacity` with `step-end` (binary toggle, no interpolation). Use hidden `<textarea>` instead of contentEditable | None -- cursor already blinks, just without interpolation |
| DomTerm | Move contentEditable to a hidden `<textarea>`. Already has good cursor blink (stops after 30s) | None |
| xterm.js DOM | Add `contain: strict` to row elements. Replace custom scrollbar with native. Already has 5min cursor timeout | Minor scrollbar visual change |
| CUI | Remove `backdrop-blur-sm` from footer. Replace `transition-all` on composer | Minimal |
