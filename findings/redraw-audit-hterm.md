# hterm -- Hidden Chrome Redraw Trigger Audit

Source paths searched: `hterm/js/*.js` (all CSS is injected inline from JS)

Note: hterm has NO external CSS files. All styles are injected programmatically via `document.createElement('style')` in `hterm_terminal.js` and inline `style.cssText` assignments in `hterm_scrollport.js`.

---

## 1. CSS Transitions

**Found: YES -- on cursor and menu elements**

- `hterm_terminal.js:1890` -- cursor node: `transition: opacity, background-color 100ms linear`
  - This is on the CURSOR ELEMENT. Every cursor show/hide/blink cycles through this transition.
  - 100ms is fast but still causes Chrome to run interpolation frames.
- `hterm_terminal.js:1818` -- context menu: `transition-duration: 200ms` (on `<menu>` element, display: none at idle)
- `hterm_notifications.js:44` -- notification toast: `transition: opacity 180ms ease-in`

**Verdict: ACTIVE REDRAW TRIGGER on cursor blink.** The cursor transition fires continuously during blink. The menu and notification transitions only fire on user interaction.

## 2. will-change

**None.** Zero usage.

## 3. position: fixed/sticky/absolute -- Compositor Layers

**Multiple absolutely-positioned elements:**

Terminal core:
- `hterm_scrollport.js:539` -- screen container: `position: absolute`
- `hterm_scrollport.js:691` -- selection overlay: `position: fixed`
- `hterm_scrollport.js:775` -- top select bag: `position: fixed`
- `hterm_scrollport.js:826` -- character measure element: `position: absolute`
- `hterm_terminal.js:1880` -- **cursor node: `position: absolute`**
- `hterm_terminal.js:1909` -- scroll blocker: `position: absolute` (top: -99px, offscreen)
- `hterm_terminal.js:1817` -- context menu: `position: absolute`

Non-terminal:
- `hterm_notifications.js:42` -- notification: `position: absolute`
- `hterm_accessibility_reader.js:29` -- a11y live region: `position: absolute`

**Verdict: ~4-5 compositor layers at idle.** The cursor, screen container, selection overlay, and scroll blocker all maintain separate layers. The scroll blocker is offscreen but still allocated.

## 4. CSS Containment

**None.** Zero `contain:` usage.

## 5. Scrollbar Type

**Browser-native scrollbar with JS visibility toggle:**
- `hterm_scrollport.js:651` -- `overflow-y: scroll; overflow-x: hidden` on screen element
- `hterm_scrollport.js:2183` -- `this.screen_.style.overflowY = 'scroll'` (show scrollbar)
- `hterm_scrollport.js:2187` -- `this.screen_.style.overflowY = 'hidden'` (hide scrollbar)

**Verdict: Native scrollbar, toggled via inline style. The toggle itself triggers a layout recalc when scrollbar visibility changes.**

## 6. Cursor Element

**Dedicated absolutely-positioned div, NOT inside the text flow:**

```js
this.cursorNode_ = this.document_.createElement('div');
this.cursorNode_.className = 'cursor-node';
// position: absolute, positioned via CSS calc() with custom properties
this.document_.body.appendChild(this.cursorNode_);
```

Position uses CSS `calc()` with custom properties:
```css
left: calc(var(--hterm-screen-padding-size) +
    var(--hterm-charsize-width) * var(--hterm-cursor-offset-col));
top: calc(var(--hterm-screen-padding-size) +
    var(--hterm-charsize-height) * var(--hterm-cursor-offset-row));
```

**Cursor blink: JavaScript setTimeout, NOT CSS animation.**
- `hterm_terminal.js:3129-3157` -- `setCursorBlink()` manages `this.timeouts_.cursorBlink`
- `hterm_terminal.js:4143-4157` -- blink callback uses `setTimeout` with configurable cycle (default 100ms on, 100ms off)
- Toggles `cursorNode_.getAttribute('visible')` between `'true'` and `'false'`
- The `[visible="false"]` rule sets `opacity: 0`
- Combined with `transition: opacity 100ms linear` on the cursor node

**Verdict: ACTIVE REDRAW TRIGGER.** The cursor blink is a JS setTimeout (every 100ms default!) that:
1. Changes a DOM attribute (`visible`)
2. Triggers the CSS `opacity` transition (100ms)
3. Chrome interpolates opacity over ~6 frames at 60fps
4. This means the cursor is ALWAYS animating when blinking -- there's never a truly idle frame

The 100ms default cycle is extremely aggressive. At 60fps, Chrome is interpolating cursor opacity on ~10 of every 12 frames.

## 7. CSS Custom Properties

**Extensive usage -- ~20+ custom properties on `:root`:**

Defined in injected stylesheet:
- `--hterm-charsize-width`, `--hterm-charsize-height` -- character dimensions
- `--hterm-blink-node-duration` -- blink speed (0.7s)
- `--hterm-mouse-cursor-default/text/pointer/style` -- mouse cursor variants
- `--hterm-screen-padding-size` -- terminal padding
- `--hterm-color-0` through `--hterm-color-255` -- full 256-color palette
- `--hterm-foreground-color`, `--hterm-background-color`
- `--hterm-cursor-color` -- cursor color
- `--hterm-cursor-offset-col`, `--hterm-cursor-offset-row` -- cursor position

**Cursor position is driven by custom properties:**
- `hterm_terminal.js:907` -- `document.documentElement.style.setProperty(...)` updates `--hterm-cursor-offset-col/row`
- This means every cursor move writes to `:root` style, triggering full style invalidation

**Color attributes use custom properties:**
- `hterm_text_attributes.js:382` -- `rgb(var(--hterm-color-${source}))` for each colored span

**Verdict: HIDDEN REDRAW COST on cursor movement.** Writing `--hterm-cursor-offset-col/row` to `:root` invalidates all computed styles that reference any `--hterm-*` variable. Chrome must recalculate styles on the entire document. At idle this doesn't fire, but during output it fires on every cursor position change.

## 8. Box-shadow / border-radius

**Minimal:**
- `hterm_notifications.js:38` -- `border-radius: 12px` on notification toast (hidden at idle)
- `hterm_terminal.js:1810` -- `border-radius: 4px` on context menu (display: none at idle)
- `hterm_terminal.js:1814` -- `filter: drop-shadow(...)` on context menu (display: none at idle)

**Verdict: No idle cost. All box-shadow/border-radius elements are hidden during normal use.**

## 9. contentEditable

**YES -- on the screen element and paste target:**
- `hterm_scrollport.js:618` -- `this.screen_.setAttribute('contenteditable', 'true')` (the main screen div)
- `hterm_scrollport.js:832` -- `this.pasteTarget_.contentEditable = true` (paste helper)
- `hterm_scrollport.js:644` -- screen has `caret-color: transparent` to hide the editing caret

**Same implications as DomTerm:**
- Chrome's internal editing caret timer fires on contentEditable elements
- The caret is hidden via `caret-color: transparent` but the timer still runs
- Spell-check may be active (no `spellcheck="false"` found in the search)

**Verdict: HIDDEN REDRAW TRIGGER.** Same contentEditable compositor tick as DomTerm. The screen element is contentEditable for input handling, and Chrome's internal caret blink timer fires independently of hterm's own cursor blink.

## 10. Inline Styles vs Classes

**Predominantly inline styles, set via `.style.cssText` blocks:**

Structural elements:
- Screen container, row nodes, cursor, scroll blocker -- all positioned via inline `style.cssText`
- Entire CSS rule blocks written as string literals in JS

Per-character styling:
- `hterm_text_attributes.js` generates inline `style.color`, `style.backgroundColor`, `style.textDecoration` on span elements
- Each attribute change creates a new span with inline styles

State changes:
- Border color: `this.div_.style.borderColor = v`
- Dimensions: `this.div_.style.width = ...`
- `overflowY`: toggled inline between 'scroll' and 'hidden'

**Verdict: Very heavy inline style usage. Every colored character run creates a span with inline styles. hterm avoids CSS classes for text attributes entirely -- it's all inline. This maximizes style recalc cost during output.**

## 11. DOM Node Count (80x24 terminal)

**Estimate: ~80-200 nodes for typical screen**

Structure:
- `x-row` custom element per visible line = 24 elements
- 1+ `span` per text attribute run within each row
- Row nodes container div
- Screen div (contentEditable)
- Cursor div (absolutely positioned, separate from text)
- Top fold, bottom fold (scroll management)
- Scroll blocker div
- Accessibility reader div (offscreen)
- Paste target (offscreen)

hterm uses a **virtual scrolling** approach:
- Only visible rows exist in the DOM
- Scrollback rows are stored in JS arrays, not as DOM nodes
- Two "fold" elements (empty divs with height) simulate scroll position

**For plain 80x24 text: ~60 nodes (24 x-rows + 24 spans + ~12 structural)**
**For colored output: 100-200 nodes (multiple spans per row)**

**Verdict: Efficient node count due to virtual scrolling. Scrollback doesn't inflate DOM. But the per-row span count still scales with attribute complexity.**

---

## Summary: Idle Redraw Triggers

| Trigger | Present | Active at Idle | Cost |
|---------|---------|---------------|------|
| CSS cursor transition | Yes (100ms linear) | Yes (during blink) | **HIGH** -- continuous opacity interpolation |
| JS cursor blink timer | Yes (setTimeout 100ms) | Yes (always, unless disabled) | **HIGH** -- fires 10x/sec, triggers DOM attribute change + transition |
| contentEditable caret timer | Yes | Yes (always) | Minimal (transparent, but compositor tick) |
| CSS blink animation | Yes (for `text-decoration: blink`) | Only if blink text present | Low |
| Custom property writes | Yes (cursor position) | No (only during output) | None at idle |
| will-change | No | No | None |
| CSS containment | No | No | N/A (missed optimization) |

**Overall idle redraw profile: MODERATE.** The cursor blink is the dominant cost. hterm's 100ms default blink cycle combined with the CSS opacity transition means Chrome is nearly always interpolating the cursor opacity -- there are only ~2ms gaps between animation frames. The contentEditable timer adds a second hidden tick. Disabling cursor blink would reduce idle cost to near-zero (only the contentEditable tick remains).

**Key difference from DomTerm:** hterm's cursor blink never stops (it's infinite setTimeout), while DomTerm's CSS animation stops after 20 cycles (30 seconds). hterm will keep triggering redraws indefinitely if the terminal has focus.
