# Redraw Audit: GoTTY -- Hidden Chrome Compositor Triggers

## Summary

GoTTY is the simplest project audited. Minimal CSS, no framework, no build-time CSS processing. The only finding is a single always-present transition on the xterm overlay element and one `position: absolute` element.

---

## 1. CSS Transitions on Always-Visible Elements

**Count: 1 conditional**

`xterm_customize.css:18`:
```css
.xterm-overlay {
    transition: opacity 180ms ease-in;
}
```

The `.xterm-overlay` element is created in `xterm.ts:22` as a plain div. It is:
- Created once in the constructor
- **Appended to DOM only when `showMessage()` is called** (resize events)
- **Removed from DOM after timeout** (`setTimeout` in `showMessage`)

So the transition exists in the stylesheet but the target element is not always in the DOM. When it is present (briefly during resize), Chrome tracks the opacity transition.

---

## 2. will-change Usage

**Count: 0**

None.

---

## 3. position: fixed/sticky/absolute Elements

**Count: 1 conditional**

- `.xterm-overlay` (`xterm_customize.css:14`): `position: absolute` with `transform: translate(-50%, -50%)` for centering. Only in DOM during message display.

No fixed or sticky elements anywhere.

**Compositor layers when idle: 1-2** (document + xterm canvas if using canvas renderer)

---

## 4. backdrop-filter / filter Usage

**Count: 0**

None.

---

## 5. CSS Containment

**Status: ABSENT**

No `contain:` or `content-visibility:`.

---

## 6. Scroll Containers

**Count: 0 explicit**

- `index.css`: `html, body, #terminal` all set to `height: 100%; width: 100%` with no overflow properties.
- xterm.js manages its own internal viewport scroll.

---

## 7. Tailwind Animation/Transition Classes

**N/A** -- GoTTY uses plain CSS and TypeScript, no Tailwind.

---

## 8. Box-shadow on Permanent Elements

**Count: 0**

No `box-shadow` anywhere.

---

## 9. SVG Icons Always in DOM

**Count: 0**

No SVG icons. The UI is purely a terminal element.

---

## 10. Custom Scrollbar CSS

**Count: 0**

No `::-webkit-scrollbar` styling. No `scrollbar-width` or `scrollbar-color`.

---

## 11. prefers-reduced-motion Support

**Status: ABSENT**

No `@media (prefers-reduced-motion)` anywhere.

Not practically needed since the only transition (180ms opacity on overlay) is ephemeral and rare.

---

## Compositor Layer Estimate (Idle State)

| Layer | Source | Always Present |
|-------|--------|----------------|
| Root document | html/body/#terminal | Yes |
| xterm canvas | xterm.js internal | Yes (if canvas renderer) |

**Total: 1-2 layers**

Note: GoTTY uses an older version of xterm.js (pre-addon architecture). The terminal may use DOM rendering (no separate canvas layer) or canvas rendering depending on configuration.

---

## Idle Rendering Pipeline Quietness

**Rating: QUIETEST OF ALL FIVE**

When idle:
- No CSS transitions active (overlay not in DOM)
- No animations
- No backdrop-filter
- No box-shadows
- No scroll containers
- No fixed/sticky elements
- No SVG icons
- No custom scrollbar paint
- xterm cursor blink is the only redraw source

GoTTY achieves the theoretical minimum for a web terminal: one document layer, one optional canvas layer, and zero CSS-driven compositor work.

### Notes:
- GoTTY uses an older xterm.js version with the deprecated `.on()` event API
- The hterm alternative (`hterm.ts`) uses Google's hterm library which has its own rendering model
- Both terminal backends manage their own cursor blink internally
- The `user-select: none` on the overlay prevents selection-related repaints during message display
