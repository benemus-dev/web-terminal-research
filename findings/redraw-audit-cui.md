# Redraw Audit: CUI -- Hidden Chrome Compositor Triggers

## Summary

CUI (Claude UI) has a moderate level of hidden redraw triggers. It uses Tailwind CSS v4 with shadcn/ui components. The chat interface has multiple scroll containers, a few backdrop-filter uses, and transition classes on interactive elements. Less aggressive than claudecodeui but not as clean as the terminal-only projects.

---

## 1. CSS Transitions on Always-Visible Elements

**Count: MODERATE**

Global transition in `global.css:22`:
```css
body {
    transition: background-color var(--transition-normal), color var(--transition-normal);
}
```
`--transition-normal` = 250ms. This means the body element permanently has an active transition watcher. Any theme switch triggers a 250ms animation across the entire page.

A commented-out global transition exists (`global.css:5`):
```css
/* transition: background-color 0.15s ease, color 0.15s ease, border-color 0.15s ease; */
```
This was wisely disabled. However, the body-level transition remains.

Component-level transitions (Tailwind classes):
- `transition-all` on Login button (`Login.tsx:54`)
- `transition-all` on Composer container (`Composer.tsx:854`) -- always visible
- `transition-colors` on ConversationHeader elements (`ConversationHeader.tsx:82,91,174`) -- always visible
- `transition-all duration-300` on Composer wrapper -- always visible
- `transition-all` on DropdownSelector items -- conditional
- `transition-colors` on TaskTabs items (`TaskTabs.tsx:16,23,30`) -- conditional per view
- `transition-opacity duration-250` on Dialog overlay -- conditional
- `transition-all duration-250` on Dialog content -- conditional
- `transition-[color,box-shadow]` on badge and input components

Inspector view (`InspectorApp.tsx`) has `before:transition-transform` on ~18 collapsible section headers -- all always in DOM when inspector is open.

---

## 2. will-change Usage

**Count: 0**

No `will-change` anywhere.

---

## 3. position: fixed/sticky/absolute Elements

**Count: 6 fixed elements (mostly conditional)**

- `InspectorApp` (`InspectorApp.tsx:679`): `fixed top-0 left-0 right-0 bottom-0` -- full-screen inspector, always present when that view is active.
- `Login` (`Login.tsx:31`): `fixed inset-0` -- only on login page.
- Dialog overlay (`dialog.tsx:39`): `fixed inset-0 z-50` -- conditional.
- Dialog content (`dialog.tsx:61`): `fixed top-[50%] left-[50%] z-50` -- conditional.
- Custom Dialog overlay (`Dialog.tsx:70`): `fixed inset-0 z-[100]` -- conditional.
- Custom Dialog content (`Dialog.tsx:82`): `fixed z-[100]` -- conditional.
- `ConversationView` sticky footer (`ConversationView.tsx:296`): `sticky bottom-0` -- always visible in chat view.

**No sticky headers** on the main chat view (unlike claudecodeui).

---

## 4. backdrop-filter / filter Usage

**Count: 2**

- `ConversationView` (`ConversationView.tsx:296`): `backdrop-blur-sm` on sticky scroll-to-bottom area. **Always visible** when chat view is active.
- `Dialog` (`Dialog.tsx:75`): `backdrop-blur-lg` on modal overlay. Conditional.

Much lighter than claudecodeui. Only 1 always-visible backdrop-filter element.

---

## 5. CSS Containment

**Status: ABSENT**

No `contain:` or `content-visibility:` in `global.css`, `theme.css`, `index.css`, or any component.

---

## 6. Scroll Containers

**Count: ~40 across chat and inspector views**

Always-visible scroll containers (chat view):
- `MessageList` (`MessageList.tsx:58,67`): `overflow-y-auto` -- main chat scroll
- `Composer` textarea (`Composer.tsx:868`): `overflow-y-auto` -- always visible
- `TaskList` (`TaskList.tsx:169,177,185,194`): `overflow-y-auto` -- 4 tab panels, 1 visible at a time
- `PermissionDialog` (`PermissionDialog.tsx:32`): `overflow-y-auto` -- conditional

Inspector view (when active):
- Sidebar panel (`InspectorApp.tsx:681`): `overflow-y-auto`
- Main content (`InspectorApp.tsx:1435`): `overflow-y-auto`
- ~22 collapsible sections with `overflow-auto` -- all with `max-h-96`
- Log monitor (`LogMonitor.tsx:190`): `overflow-y-auto`

---

## 7. Tailwind Animation/Transition Classes on Always-Visible Elements

**Always-visible:**
- `transition-all duration-300` on Composer container
- `transition-colors` on ConversationHeader (3 elements)
- `scrollbar-thin` Tailwind plugin classes on multiple containers

**Conditional (active during states):**
- `animate-spin` on Loader2 icons (loading states only)
- `animate-in`, `animate-out`, `fade-in-0`, `zoom-in-95` etc. on shadcn Dialog/Popover/Tooltip
- `animate-slide-up` on PermissionDialog
- `slide-in-from-top duration-300` on error banner

**WaveformVisualizer** (`WaveformVisualizer.tsx`): Uses `requestAnimationFrame` loop -- but only active when recording audio. Properly cancels on stop.

---

## 8. Box-shadow on Permanent Elements

**Count: MODERATE**

Always-visible:
- `Composer` (`Composer.tsx:854`): `shadow-sm` -- always visible input area
- `ConversationView` sticky area: no explicit shadow but has `backdrop-blur-sm`

Defined in theme but applied conditionally:
- `--shadow-sm`, `--shadow-md`, `--shadow-lg`, `--shadow-short` in `theme.css`
- `shadow-lg` on DropdownSelector popover (conditional)
- `shadow-[0_-4px_20px_...]` on Dialog mobile drawer (conditional)
- `shadow-[0_2px_8px_...]` on Login hover state (conditional)
- `shadow-none` explicitly set on several button variants

---

## 9. SVG Icons Always in DOM

**Count: 26 icon imports across 26 files**

Uses `lucide-react` icons. Always-visible icons in chat view:
- ConversationHeader: back arrow, menu icons
- Composer: send button, attachment, permission mode icons
- Home: task icons

All are simple path-based SVGs, no gradients or filters. Lower risk than claudecodeui due to fewer total icons.

---

## 10. Custom Scrollbar CSS

**Count: Tailwind `scrollbar-thin` plugin (7 usages)**

```
scrollbar-thin scrollbar-thumb-black/20 dark:scrollbar-thumb-white/20 scrollbar-track-transparent
scrollbar-thin scrollbar-thumb-transparent hover:scrollbar-thumb-border scrollbar-track-transparent
```

These use the Tailwind scrollbar plugin which generates `::-webkit-scrollbar` styles. Applied to:
- DropdownSelector list
- TaskList (4 instances)
- Composer textarea
- TaskTool content

No global `::-webkit-scrollbar` override in the CSS files (unlike claudecodeui).

---

## 11. prefers-reduced-motion Support

**Status: ABSENT**

No `@media (prefers-reduced-motion)` in `global.css`, `theme.css`, `index.css`, or any component file. The shadcn/ui components also don't include it.

This is a gap -- CUI has animations on dialogs, error banners, and tooltips that will run even when the user has requested reduced motion.

---

## Compositor Layer Estimate (Idle State -- Chat View)

| Layer | Source | Always Present |
|-------|--------|----------------|
| Root document | html/body | Yes |
| ConversationView sticky | sticky + backdrop-blur-sm | Yes |
| Composer | border + shadow-sm | Likely |
| MessageList scroll | overflow-y-auto | Yes |

**Total: 3-4 compositor layers** (idle, chat view)

Inspector view adds 2-3 more layers when active.

---

## Idle Rendering Pipeline Quietness

**Rating: MODERATE**

When idle in chat view:
1. **1 backdrop-filter** on the sticky ConversationView footer -- forces GPU compositing for that layer. However, it's `backdrop-blur-sm` (4px blur) which is lighter than claudecodeui's 20-24px blurs.
2. **Body-level transition** (250ms on background-color and color) -- style system tracking on the root element.
3. **Composer transition-all** -- transition eligibility tracked on the always-visible input area.
4. **3-4 compositor layers** -- reasonable for a chat app.
5. **No `prefers-reduced-motion` support** -- animations cannot be disabled by OS setting.

Compared to claudecodeui, CUI is significantly quieter:
- No universal structural-element transitions
- No global backdrop-filter on nav/sidebar
- No `will-change` usage
- Far fewer always-visible fixed elements
- 26 icon files vs 169

### Highest-impact fixes:
1. Add `@media (prefers-reduced-motion: reduce)` support.
2. Remove `backdrop-blur-sm` from the ConversationView sticky footer (replace with opaque background).
3. Remove `transition: background-color 250ms` from body -- only apply during actual theme switches.
4. Add `contain: content` to MessageList and TaskList scroll containers.
