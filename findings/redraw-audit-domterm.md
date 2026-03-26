# DomTerm -- Hidden Chrome Redraw Trigger Audit

Source paths searched: `hlib/*.css`, `hlib/*.js`

---

## 1. CSS Transitions

**Found: YES -- in layout chrome and menus, NOT in terminal body**

- `jsMenus.css`: 6 transitions on menu opacity (30ms-250ms). Menus are `position: fixed` with `visibility` toggling. These fire only on menu open/close, not at idle.
- `domterm-layout.css` / `goldenlayout-base.css`: transitions on `.lm_dropTargetIndicator` (200ms) and `.lm_controls > li` opacity (300ms). These are tab/pane management UI, not the terminal output area.

**Verdict: No idle redraw cost.** All transitions are on UI chrome that is hidden/inactive during normal terminal use.

## 2. will-change

**None.** Zero usage anywhere in the codebase.

## 3. position: fixed/sticky/absolute -- Compositor Layers

**Significant usage, but stratified:**

Terminal-relevant:
- `span.tail-hider` -- `position: absolute` (command folding indicators, right-aligned)
- `span.error-exit-mark` -- `position: absolute` (error dots)
- `div.domterm-show-info` -- `position: absolute`, z-index: 100 (info overlay)
- `span.focus-caret-mark`, `div.focus-caret-line` -- `position: absolute` (cursor positioning elements)

Layout chrome (goldenlayout):
- ~20 `position: absolute` rules for tab management, splitters, drag targets, resize handles

Fixed overlays:
- `.domterm-show-info` variant -- `position: fixed` (2 rules)
- `jsMenus` -- `position: fixed` (menu overlay)
- `dt-titlebar` -- `position: fixed`

**Verdict: ~3-4 compositor layers in the terminal area at idle** (caret mark, caret line, possible tail-hiders on visible lines). GoldenLayout adds more if panes are split, but those are structural.

## 4. CSS Containment

**None.** Zero `contain:` usage. This is a missed optimization -- every DOM change in the terminal area triggers layout recalc up the tree.

## 5. Scrollbar Type

**Browser-native scrollbar.**
- `div.dt-buffers { overflow-y: scroll }` -- standard native scrollbar
- `div.dt-buffer { overflow-x: hidden; overflow-y: hidden }` -- buffers themselves don't scroll
- The scroll container is `dt-buffers`, Chrome renders its native scrollbar.

**Verdict: No custom scrollbar JS overhead, but native scrollbar can trigger compositing on scroll.**

## 6. Cursor Element

**Complex multi-element cursor system:**

- `span[std="caret"]` -- the actual caret element, styled with `caret-color: transparent` (hides native caret)
- `div.focus-caret` -- container div, `position: absolute`
- `span.focus-caret-mark` -- positioned marker
- `div.focus-caret-line` -- line highlight, `position: absolute`
- Terminal itself has `contentEditable = true` on `topNode` (line 3570 in terminal.js)
- Caret position updated via inline styles: `.style.top`, `.style.left`, `.style.bottom`, `.style.right` (terminal.js ~6900)

**Blinking cursor: CSS animation**
```
animation: blinking-caret 1.5s steps(1) 0s var(--caret-blink-count);
```
- Uses `steps(1)` -- binary on/off, no interpolation
- `--caret-blink-count` defaults to 20 iterations then stops
- Two variants: `blinking-caret` and `blinking-caret-lineend`
- **Keyframes toggle `background-color` and `color`** -- these are Paint properties

**Verdict: ACTIVE REDRAW TRIGGER when cursor is blinking.** Every 750ms (half of 1.5s cycle), Chrome must repaint the caret element. However, this stops after 20 cycles (30 seconds), then the cursor goes solid. After that, no idle redraw from cursor.

## 7. CSS Custom Properties

**Heavy usage -- 15+ custom properties on terminal elements:**

Layout-critical:
- `--char-width`, `--char-height`, `--wchar-width` -- used on every `dt-cluster` span
- `--background-color`, `--foreground-color` -- used everywhere
- `--caret-color`, `--caret-accent-color` -- cursor styling

Changed at runtime via JS:
- `terminal.js:526` -- `this.topNode.style.setProperty(setting.cssVariable, value)` on settings change
- `domterm-parser.js:1855` -- sets `--dt-{colorname}` properties from escape sequences
- `domterm-parser.js:2348` -- removes color properties

**Verdict: Custom properties on the root element cause full style recalc when changed.** At idle, no changes occur. During terminal output, color escape sequences can trigger `setProperty` on the topNode, which invalidates all computed styles using those variables.

## 8. Box-shadow / border-radius

**On terminal elements: YES**
- `domterm-default.css:90` -- `border-radius: 0.3ex` on some inline element
- `domterm-default.css:235,262,312` -- `border-radius: 6px`, `4px` on overlay/info elements

**On layout chrome: heavy**
- GoldenLayout themes: extensive `box-shadow` on tabs, splitters, drag indicators
- `domterm-layout.css`: `box-shadow: 2px 2px 4px` on header
- `dt-titlebar`: `border-radius: 10px 10px 0px 0px`

**Verdict: Terminal body itself has minimal box-shadow/border-radius. Layout chrome has plenty, but it's outside the repaint region for terminal content.**

## 9. contentEditable

**YES -- this is DomTerm's primary input mechanism.**

- `terminal.js:3570` -- `topNode.contentEditable = true` (the entire terminal div)
- `terminal.js:4060` -- `caretNode.contentEditable = true`
- `terminal.js:4005` -- `miniBuffer.contentEditable = true`
- `terminal.js:408` -- `this.buffers.contentEditable = false` (buffers excluded)

**Chrome implications of contentEditable:**
- Chrome maintains an editing caret layer even though DomTerm hides it with `caret-color: transparent`
- The editing caret has its own blink timer (Chrome internal, ~500ms) that fires even when hidden
- Chrome creates additional internal layers for spell-check underlines (if not disabled)
- `contentEditable` forces Chrome to maintain a "composition" layer for IME input

**Verdict: HIDDEN REDRAW TRIGGER.** Chrome's internal editing caret timer fires every ~500ms on contentEditable elements, even if the visual caret is CSS-hidden. This is invisible in DevTools Paint profiler but shows up in `chrome://tracing` as compositor work. The actual redraw cost is minimal (the caret is transparent) but the compositor tick still fires.

## 10. Inline Styles vs Classes

**Heavy inline style usage:**
- `terminal.js` has 60+ instances of `.style.` manipulation
- Caret positioning: inline top/left/bottom/right (4 properties per cursor move)
- Line heights: `line.style.height = this.cursorHeight + 'px'`
- Buffer widths: `buffers[i].style.width = styleWidth`
- Opacity changes: `this.initial.style.opacity = "0.3"` (connection state)
- Transform: `style.setProperty("transform", ...)` for zoom
- Visibility toggling: `el.style.visibility = "hidden"` / `""` for wide chars
- Margin adjustments: `previous.style.marginRight = ...` for char spacing

**Also uses classes for state:**
- `.domterm-active`, `.paused`, `.focusmode`, `.markmode` -- state classes on container

**Verdict: Mixed approach. Inline styles dominate for positioning and sizing. Each cursor movement writes 4 inline style properties. During output, many span elements get inline styles for colors/attributes. This is more style-recalc-heavy than a pure class-based approach.**

## 11. DOM Node Count (80x24 terminal)

**Estimate: ~200-500 nodes for a typical screen**

Structure per line:
- 1 `div.domterm-pre` per line = 24 divs
- 1+ `span` per attribute run within a line (plain text = 1 span, colored = multiple)
- `span.dt-cluster` for wide characters (display: inline-block)
- `span[breaking="yes"]` for soft line breaks

Additional persistent nodes:
- `div.dt-buffers` > `div.dt-buffer` (2 -- normal + alternate screen)
- `div.focus-caret`, `span.focus-caret-mark`, `div.focus-caret-line`
- `span[std="caret"]`
- `div.domterm-show-info` (even if hidden)
- `div.below-menubar`, titlebar, menu structures

**For plain 80x24 text: ~75 nodes (24 pre-divs + 24 spans + wrappers)**
**For richly colored output: 200-500 nodes (multiple spans per line for each color change)**

**Verdict: DOM is proportional to attribute complexity, not fixed. A `ls --color` output can have 5-10 spans per line, pushing node count up. This is inherent to DOM-based terminals.**

---

## Summary: Idle Redraw Triggers

| Trigger | Present | Active at Idle | Cost |
|---------|---------|---------------|------|
| CSS transitions | Yes (menus only) | No | None |
| will-change | No | No | None |
| Cursor blink animation | Yes (CSS keyframes) | Yes, for 30s then stops | Low (Paint on caret element only) |
| contentEditable caret timer | Yes | Yes (always) | Minimal (transparent caret, but compositor tick fires) |
| CSS custom property changes | Yes (heavy) | No (only during output) | None at idle |
| Layout chrome box-shadow | Yes | No (static) | None |
| Inline style writes | Yes (heavy) | No (only during output/cursor move) | None at idle |

**Overall idle redraw profile: LOW but not zero.** The contentEditable timer is the one truly hidden trigger -- Chrome's internal editing caret blinks even when `caret-color: transparent`. The CSS cursor blink stops after 30 seconds. After that, only the contentEditable compositor tick remains.
