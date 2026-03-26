# Redraw Audit: WeTTY -- Hidden Chrome Compositor Triggers

## Summary

WeTTY is nearly as clean as ttyd. The main concerns are two always-present `position: absolute` elements (`#options`, `#functions`) that create compositor layers even when visually inert, and the xterm viewport scrollbar override.

---

## 1. CSS Transitions on Always-Visible Elements

**Count: 0 always-visible**

No `transition` properties in any of the SCSS files (`styles.scss`, `terminal.scss`, `overlay.scss`, `options.scss`, `variables.scss`).

The xterm_config `style.css` is a separate configuration page, not the terminal UI.

---

## 2. will-change Usage

**Count: 0**

None.

---

## 3. position: fixed/sticky/absolute Elements

**Count: 3 always-present (2 compositor layers)**

- `#terminal` (`terminal.scss:4`): `position: relative` -- establishes containing block, not a separate layer.
- `#overlay` (`overlay.scss:8`): `position: absolute`, `z-index: 100` -- always in DOM but `display: none` by default. **Not a compositor layer when hidden.**
- `#options` (`options.scss:5-8`): `position: absolute`, `z-index: 20`, 16x16px -- **always in DOM and visible** (gear icon). Creates a compositor layer.
- `#options .toggler` (`options.scss:14`): `position: absolute` nested. Same layer as parent.
- `#options .editor` (`options.scss:33`): `position: relative`, `display: none` by default.
- `#functions` (`options.scss:55`): `position: fixed`, `z-index: 20` -- **always in DOM**. Creates a compositor layer even if empty.
- `#functions .onscreen-buttons` (`options.scss:73`): `display: none` by default.

**Compositor layers when idle: 3** (document, xterm canvas, #options/functions fixed element)

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

**Count: 1 explicit override**

- `.xterm .xterm-viewport` (`styles.scss:23`): `overflow-y: hidden` -- deliberately disables xterm's built-in scrollbar. This prevents the scrollbar paint surface.
- `body` (`styles.scss:12`): `overflow: hidden` on html/body.

**Verdict:** Clean -- the scrollbar override actually reduces paint work.

---

## 7. Tailwind Animation/Transition Classes

**N/A** -- WeTTY uses plain SCSS, no Tailwind.

---

## 8. Box-shadow on Permanent Elements

**Count: 0**

No `box-shadow` in any SCSS file.

---

## 9. SVG Icons Always in DOM

**Count: 0 from CSS audit**

The toggler element uses text content (gear character), not SVG. No SVG icons in stylesheets.

---

## 10. Custom Scrollbar CSS

**Count: 0**

No `::-webkit-scrollbar` styling. The approach is to hide the scrollbar entirely via `overflow-y: hidden`.

---

## 11. prefers-reduced-motion Support

**Status: ABSENT**

No `@media (prefers-reduced-motion)` anywhere.

---

## Compositor Layer Estimate (Idle State)

| Layer | Source | Always Present |
|-------|--------|----------------|
| Root document | html/body | Yes |
| xterm canvas | xterm.js | Yes |
| #options element | position: absolute + z-index | Yes (16x16px, minimal) |
| #functions element | position: fixed + z-index | Yes (but may be empty) |

**Total: ~3-4 layers**

---

## Idle Rendering Pipeline Quietness

**Rating: VERY QUIET**

When idle:
- No CSS transitions running
- No animations
- No backdrop-filter
- No box-shadows
- No scroll containers with visible scrollbars
- `#overlay` is display:none (no cost)
- `#options` is tiny (16x16) -- negligible compositor cost
- xterm cursor blink is the only redraw source (~1.67 Hz)

WeTTY's deliberate decision to hide the xterm scrollbar (`overflow-y: hidden`) is actually beneficial -- it eliminates a paint surface that xterm.js would otherwise maintain.
