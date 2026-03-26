# shellinabox -- Hidden Chrome Redraw Trigger Audit

Source paths searched: `shellinabox/vt100.jspp`, `shellinabox/styles.css`, `shellinabox/*.css`

---

## 1. CSS Transitions

**None.** Zero CSS transitions anywhere in the codebase. This is a pre-CSS3 era project (2008-2010). All animation is done via JavaScript.

## 2. will-change

**None.** Zero usage.

## 3. position: fixed/sticky/absolute -- Compositor Layers

**Heavy usage of position: absolute -- many potential compositor layers:**

Terminal core:
- `#cursor` -- `position: absolute; left: 0px; top: 0px; z-index: 1` (the cursor element)
- `#scrollable` -- `position: relative` (scroll container)
- `#lineheight` -- `position: absolute; visibility: hidden` (measurement element)
- `#reconnect` -- `position: absolute; z-index: 2` (reconnect button)
- `#cursize` -- `position: absolute; z-index: 2` (resize indicator)
- `.hidden` -- `position: absolute; top: -10000px; left: -10000px` (offscreen storage)

Overlays:
- `#menu` -- `position: absolute; z-index: 3`
- `#menu .popup` -- `position: absolute`
- `#keyboard` -- `position: absolute; z-index: 3`
- `#keyboard .box` -- `position: absolute` (with box-shadow, border-radius, opacity)
- `#kbd_button` -- `position: fixed; z-index: 0; visibility: hidden`

**Verdict: ~3-4 compositor layers at idle.** The cursor (z-index: 1), the scroll container, and the hidden measurement element. Menu/keyboard/reconnect are hidden via visibility or display:none and shouldn't create active layers. The `.hidden` elements at -10000px may still create layers depending on Chrome's heuristics.

## 4. CSS Containment

**None.** Zero `contain:` usage. Expected for a 2008-era project.

## 5. Scrollbar Type

**Browser-native scrollbar:**
- `#scrollable { overflow-x: hidden; overflow-y: scroll }` -- standard native scrollbar
- No custom scrollbar implementation

**Verdict: No JS scrollbar overhead. Native scrollbar compositing only.**

## 6. Cursor Element

**Absolutely-positioned `<pre>` element, JavaScript-driven blink:**

The cursor is a `<pre>` tag:
```html
<pre id="cursor">&nbsp;</pre>
```

CSS:
```css
#cursor { position: absolute; left: 0px; top: 0px; overflow: hidden; z-index: 1; }
#cursor.bright { background-color: black; color: white; }
#cursor.dim { background-color: white; opacity: 0.2; }
#cursor.inactive { border: 1px solid; margin: -1px; }
```

**Blink mechanism: `setInterval` at 500ms, toggling className:**
```js
this.cursorInterval = setInterval(..., 500);
// toggles between 'bright' and 'dim' class names
this.cursor.className = this.cursor.className == 'bright' ? 'dim' : 'bright';
```

Position updated via inline styles:
```js
this.cursor.style.left = pixelX + 'px';
this.cursor.style.top = pixelY + 'px';
```

Size locked to character dimensions:
```js
this.cursor.style.width = this.cursorWidth + 'px';
this.cursor.style.height = this.cursorHeight + 'px';
```

**Verdict: ACTIVE REDRAW TRIGGER.** The setInterval fires every 500ms and toggles the cursor class between `bright` (opaque) and `dim` (opacity: 0.2). This triggers:
1. Class name change -- style recalc
2. `background-color` change -- Paint
3. `opacity` change -- Composite (opacity changes are compositor-friendly but still require work)

Unlike hterm, there is NO CSS transition on the cursor, so the change is instantaneous (one frame, not interpolated). This is cheaper than hterm's 100ms transition.

## 7. CSS Custom Properties

**None.** Zero `var(--)` usage. All colors are hardcoded in CSS or set via inline styles. This is a pre-custom-properties era project.

**Verdict: No custom property invalidation cost whatsoever.**

## 8. Box-shadow / border-radius

**On overlay elements only:**
- `#keyboard .box` -- `border-radius: 10px; box-shadow: 4px 4px 6px #222222; opacity: 0.85`
- `#keyboard b, i, s, u` -- `border-radius: 5px; box-shadow: 2px 2px 3px #222222`
- `#keyboard .selected` -- `box-shadow: 0px 0px 3px #222222`

**No box-shadow or border-radius on terminal content or cursor.**

**Verdict: Zero idle cost. Keyboard overlay is hidden (`display: none`) at idle.**

## 9. contentEditable

**None.** shellinabox does NOT use contentEditable. Input is handled via a hidden input field and keydown/keypress event listeners on the container.

**Verdict: No contentEditable compositor overhead.** This is a significant advantage -- Chrome does not maintain an internal editing caret timer.

## 10. Inline Styles vs Classes

**Extremely heavy inline style usage -- the most aggressive of all three projects:**

Every character span has inline `style.cssText`:
```js
span.style.cssText = style;  // style is a string like "color: white; background-color: black;"
```

Line elements:
```js
line.style.height = this.cursorHeight + 'px';
```

The style merge/split operations (vt100.jspp ~1760-1860) manipulate `span.style.cssText` directly, comparing style strings for equality to merge adjacent spans:
```js
if (span.style.cssText == sibling.style.cssText) { // merge }
```

Cursor positioning: 2 inline style writes per cursor move (left, top).

Console visibility:
```js
this.console[1-this.currentScreen].style.display = 'none';
this.console[this.currentScreen].style.display = '';
```

CSS transforms for zoom:
```js
this.console[this.currentScreen].style[transform] = style;
this.cursor.style[transform] = style;
this.space.style[transform] = style;
```

**Verdict: Every character attribute change writes a full `style.cssText` string. During output, this generates far more style recalc work than a class-based approach. At idle, no writes occur. The cssText string comparison for span merging is a clever optimization that reduces node count but doesn't reduce style recalc cost during output.**

## 11. DOM Node Count (80x24 terminal)

**Estimate: 100-400 nodes for typical screen**

Structure per line:
- 1 `<pre>` per line = 24 pre elements (or `<div>` in some modes)
- 1+ `<span>` per attribute run within each line (shellinabox creates new spans for every color change)
- Wide character spans are display: inline-block

Fixed structural nodes:
- `#vt100` container
- `#scrollable` (scroll container)
- `#console` (normal screen)
- `#alt_console` (alternate screen, display: none)
- `#cursor` (positioned pre)
- `#lineheight` (measurement, hidden)
- `#padding` (scroll spacer, hidden)
- `#reconnect` (button)
- `#cursize` (resize indicator)
- `#menu` (hidden)
- `#keyboard` (hidden)
- `#kbd_button` (hidden)
- `<input>` for keyboard input

Scrollback:
- shellinabox keeps scrollback lines in the DOM (not virtual)
- `.scrollback` class lines remain as DOM nodes
- `.hidden` class puts old content at -10000px (still in DOM!)

**For plain 80x24 text: ~70 nodes (24 pre + 24 span + ~20 structural)**
**For colored output: 200-400 nodes (many spans per line)**
**With scrollback: potentially 1000+ nodes** (all scrollback stays in DOM)

**Verdict: DOM-heaviest of the three at scale.** No virtual scrolling means scrollback accumulates real DOM nodes. The `.hidden` class moves them offscreen but doesn't remove them. Chrome must still account for these in style recalc and layout, even though they're not painted.

---

## Summary: Idle Redraw Triggers

| Trigger | Present | Active at Idle | Cost |
|---------|---------|---------------|------|
| CSS transitions | No | No | None |
| will-change | No | No | None |
| Cursor blink (setInterval 500ms) | Yes | Yes (always when focused) | **Moderate** -- class toggle every 500ms, no transition interpolation |
| contentEditable | No | No | **None** -- significant advantage |
| CSS custom properties | No | No | None |
| Box-shadow on terminal | No | No | None |
| Inline style writes | Yes (heavy) | No (only during output) | None at idle |
| Scrollback DOM bloat | Yes | Yes (increases recalc scope) | **Low but growing** -- layout/style recalc cost rises with scrollback size |

**Overall idle redraw profile: LOW.** The only active idle trigger is the cursor blink setInterval at 500ms, which does an instantaneous class toggle (no CSS transition interpolation). No contentEditable timer. No CSS animations. No custom property invalidation.

**Key advantages over DomTerm and hterm:**
1. No contentEditable -- eliminates Chrome's internal caret timer entirely
2. No CSS transitions on cursor -- blink is a clean binary class swap, one repaint frame per toggle
3. No CSS custom properties -- zero style invalidation chain risk
4. No CSS animations at all

**Key disadvantage:**
1. Scrollback stays in DOM -- over time, the DOM tree grows unbounded, increasing the cost of any style recalc or layout that does occur (including the cursor blink class toggle)
