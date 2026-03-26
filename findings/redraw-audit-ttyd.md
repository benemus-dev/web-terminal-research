# Redraw Audit: ttyd -- Hidden Chrome Compositor Triggers

## Summary

ttyd is remarkably minimal. Almost no CSS-level redraw triggers exist when idle. The only significant finding is the overlay addon's inline transition and the modal's `position: fixed` layer.

---

## 1. CSS Transitions on Always-Visible Elements

**Count: 0 always-visible, 1 conditional**

- The overlay addon (`overlay.ts:19`) sets inline CSS `-webkit-transition: opacity 180ms ease-in` on a dynamically created div. This element is **only appended to DOM when showing a message** (resize overlay, connection status) and is **removed after timeout**. Not always-visible.
- No `transition` properties in `index.scss` or `modal.scss`.

**Verdict:** CLEAN -- no persistent transition watchers.

---

## 2. will-change Usage

**Count: 0**

No `will-change` anywhere in the codebase.

---

## 3. position: fixed/sticky/absolute Elements

**Count: 2 conditional, 0 always-visible**

- `.modal` (`modal.scss:9`): `position: fixed` -- only rendered when `show` prop is true (file upload modal). Conditional.
- `.modal-background` (`modal.scss:16`): `position: absolute` within the fixed modal. Conditional.
- Overlay addon div (`overlay.ts:17`): `position: absolute` -- only appended during message display, then removed.

**Compositor layers when idle: 1** (just the main document)

---

## 4. backdrop-filter / filter Usage

**Count: 0**

None.

---

## 5. CSS Containment

**Status: ABSENT**

No `contain:` or `content-visibility:` anywhere. The terminal container has `overflow: hidden` on body which provides implicit containment of paint.

---

## 6. Scroll Containers

**Count: 1 conditional**

- `.modal-content` (`modal.scss:25`): `overflow: auto` -- only exists when modal is shown.
- `body` has `overflow: hidden` -- no scrollbar rendered.
- xterm.js itself manages its own viewport scrollbar internally (via canvas/WebGL).

**Verdict:** No scroll containers generating paint when idle.

---

## 7. Tailwind Animation/Transition Classes

**N/A** -- ttyd uses Preact with plain SCSS, no Tailwind.

---

## 8. Box-shadow on Permanent Elements

**Count: 1 conditional**

- `.file-cta` (`modal.scss:59`): `box-shadow: none` -- explicitly set to none.
- No box-shadow on any always-visible element.

---

## 9. SVG Icons Always in DOM

**Count: 0**

No SVG icons in the codebase. ttyd is text-only UI.

---

## 10. Custom Scrollbar CSS

**Count: 0**

No `::-webkit-scrollbar` styling. No `scrollbar-width` or `scrollbar-color`.

---

## 11. prefers-reduced-motion Support

**Status: ABSENT**

No `@media (prefers-reduced-motion)` anywhere.

---

## Compositor Layer Estimate (Idle State)

| Layer | Source | Always Present |
|-------|--------|----------------|
| Root document | html/body | Yes |
| xterm canvas/WebGL | xterm.js addon | Yes |

**Total: 2 layers** (document + xterm rendering surface)

---

## Idle Rendering Pipeline Quietness

**Rating: VERY QUIET**

When idle (no terminal output, no user interaction):
- No CSS transitions running
- No animations defined
- No backdrop-filter compositing
- No box-shadows requiring paint
- No scroll containers with visible scrollbars
- xterm.js cursor blink is the only source of redraws (handled by xterm internally, ~1.67 Hz)

The only thing Chrome's compositor needs to do is composite the xterm canvas layer with the document layer. No CSS-driven repaints occur at all during idle.
