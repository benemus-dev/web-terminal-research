# Chrome Redraw Trigger Audit: claude-code-webui

Audited: `frontend/src/` (all .tsx, .ts, .css files)
Date: 2026-03-26

## Summary

The previous audit found "zero JS idle activity" but this CSS/DOM-level audit reveals
significant hidden redraw sources. The project uses Tailwind CSS heavily with
backdrop-blur, transition-all, and semi-transparent backgrounds on always-visible
elements -- all of which create compositor layers and force continuous compositing work.

**Estimated always-visible compositor layers at idle (chat view): 8-12**

---

## 1. CSS Transitions on Always-Visible Elements

**Total transition instances: 28**

### Always-visible (WORST -- active at idle):
| Class | File | Element | Problem |
|-------|------|---------|---------|
| `transition-colors duration-300` | ChatPage.tsx:437 | Root `<div>` (min-h-screen) | Always in DOM. Forces style recalc on any color property change (dark mode, inherited) |
| `transition-all duration-200` | ChatInput.tsx:225 | Textarea input | Always visible. `transition-all` monitors ALL properties |
| `transition-colors` | ChatInput.tsx:255 | Working directory button | Always visible |
| `transition-all duration-200` | SettingsButton.tsx:11 | Settings gear button | Always visible |
| `transition-all duration-200` | HistoryButton.tsx:11 | History button | Always visible |
| `transition-colors duration-300` | DemoPage.tsx:353 | Demo root div | Always in DOM when demo active |
| `transition-colors duration-300` | ProjectSelector.tsx:70 | Project selector root | Always in DOM when on project page |

### Conditional (less concerning):
- ChatPage.tsx:445,454 -- Back buttons (only when `isHistoryView`/`isLoadedConversation`)
- ChatPage.tsx:559 -- "New conversation" button (only on error state)
- ChatInput.tsx:233,242 -- Abort/Send buttons (conditional on `isLoading`)
- PermissionInputPanel.tsx:212,249,286 -- Permission options (only when permission requested)
- PlanPermissionInputPanel.tsx:134,171,208 -- Plan options (only when plan approval)
- SettingsModal.tsx:52 -- Close button (only when modal open)
- GeneralSettings.tsx:37,68 -- Theme buttons (only when settings open)
- HistoryView.tsx:135 -- History cards (only in history view)
- ProjectSelector.tsx:90 -- Project list items (only on project page)

### Critical issue: `transition-all`
**9 instances** of `transition-all` found. This is the worst possible transition value --
it tells Chrome to watch ALL CSS properties for changes, including inherited ones.
On elements with `backdrop-blur-sm`, this creates a pathological case where the compositor
must re-evaluate blur + transparency on every style recalculation.

Files with `transition-all`:
- ChatInput.tsx: 3 instances (textarea, abort button, send button)
- ChatPage.tsx: 2 instances (back buttons -- conditional)
- SettingsButton.tsx: 1 instance (always visible)
- HistoryButton.tsx: 1 instance (always visible)
- GeneralSettings.tsx: 2 instances (conditional)
- PermissionInputPanel.tsx: 3 instances (conditional)
- PlanPermissionInputPanel.tsx: 3 instances (conditional)

---

## 2. CSS `will-change`

**Count: 0**

No explicit `will-change` declarations. However, Tailwind's `backdrop-blur-sm`
and `transition-all` implicitly promote elements to compositor layers, achieving
the same effect without the developer's awareness.

---

## 3. CSS `transform` and `opacity` on Static Elements

**`opacity` instances: 16 (all static, always in DOM during chat)**

Most are on MessageComponents.tsx elements that use fixed opacity values like
`opacity-90`, `opacity-80`, `opacity-70`, `opacity-60`. These are static Tailwind
classes, not animated, so they promote to compositor layers but do NOT cause
continuous redraws by themselves.

However, combined with `backdrop-blur-sm` on parent containers, any compositor
layer with reduced opacity forces Chrome to composite in a specific order,
preventing layer merging optimizations.

**`transform` instances: 0** (no explicit transforms found)

---

## 4. `backdrop-filter` and `filter`

**`backdrop-blur-sm` instances: 9**

This is the single most expensive CSS feature in this codebase.

### Always-visible (CRITICAL):
| File | Line | Element | Impact |
|------|------|---------|--------|
| ChatInput.tsx:225 | Textarea | Always visible. Blur requires reading pixels behind element every frame |
| ChatMessages.tsx:82 | Messages container | Always visible. This is the LARGEST element with backdrop-blur |
| SettingsButton.tsx:11 | Settings button | Always visible |
| HistoryButton.tsx:11 | History button | Always visible |

### Conditional:
| File | Line | Element |
|------|------|---------|
| ChatPage.tsx:445 | Back button (history view only) |
| ChatPage.tsx:454 | Back button (loaded conversation only) |
| SettingsModal.tsx:41 | Modal overlay (when open) |
| PermissionInputPanel.tsx:172 | Permission panel (when permission) |
| PlanPermissionInputPanel.tsx:105 | Plan panel (when plan) |

### Why this matters:
`backdrop-blur-sm` compiles to `backdrop-filter: blur(4px)`. Chrome must:
1. Create a separate compositor layer for the element
2. On every composite, read the pixels BEHIND the element
3. Apply a Gaussian blur to those pixels
4. Composite the blurred result with the element's semi-transparent background

The ChatMessages container (`bg-white/70 ... backdrop-blur-sm`) is especially
problematic because it covers most of the viewport and its content scrolls.
Every scroll event forces a full backdrop-filter recomputation.

**No `filter:` property found** (only backdrop-filter via Tailwind).

---

## 5. `position: fixed` and `position: sticky`

**`fixed` instances: 1**
- SettingsModal.tsx:41 -- Modal overlay (`fixed inset-0 z-50`). Only when modal is open.

**`sticky` instances: 0**

This is relatively clean. No always-visible fixed-position elements.

---

## 6. Tailwind Animation Utilities

**Active animation instances: 4**

| Class | File | Line | Element | Condition |
|-------|------|------|---------|-----------|
| `animate-spin` | ChatPage.tsx:526 | Loading spinner | Only during `isLoading && !historyError` |
| `animate-spin` | HistoryView.tsx:58 | Loading spinner | Only during history loading |
| `animate-spin` | MessageComponents.tsx:395 | Thinking spinner | Only during `isLoading` |
| `animate-pulse` | MessageComponents.tsx:396 | "Thinking..." text | Only during `isLoading` |

All animations are conditional on loading state. When idle, none are active.
This is correctly implemented.

**`duration-` instances: ~20** (all paired with `transition-*`, covered in section 1)
**`ease-` instances: 0** (Tailwind defaults used)

---

## 7. Box-shadow on Always-Visible Elements

**`shadow-sm` always visible: 4**
- ChatInput.tsx:225 -- Textarea (`shadow-sm`)
- ChatMessages.tsx:82 -- Messages container (`shadow-sm`)
- SettingsButton.tsx:11 -- Settings button (`shadow-sm`)
- HistoryButton.tsx:11 -- History button (`shadow-sm`)

**`shadow-md` / `shadow-xl` conditional: ~8** (hover states, modal)

**`hover:shadow-md` on always-visible: 3**
- SettingsButton.tsx, HistoryButton.tsx, ChatInput.tsx send button

Box-shadow alone is paint-only (no compositor promotion). But when combined with
`backdrop-blur-sm` on the same element (which DOES promote to compositor layer),
the shadow must be painted on every composite frame. The 4 always-visible elements
with both `shadow-sm` AND `backdrop-blur-sm` are the worst offenders.

**`border-radius` (rounded-*) always visible: ~6** -- same concern when on compositor layers.

---

## 8. Scroll Containers

**Scroll containers: 3**

| File | Line | Element | Always visible? |
|------|------|---------|----------------|
| ChatMessages.tsx:82 | Messages area | YES -- main scroll container |
| HistoryView.tsx:130 | History grid | Only in history view |
| SettingsModal.tsx:60 | Settings content | Only when modal open |

The ChatMessages scroll container is the most concerning because it ALSO has
`backdrop-blur-sm`. Scrolling this container forces Chrome to recompute the
backdrop blur on every scroll frame. This is a known Chrome performance issue.

---

## 9. SVG Icons Always in DOM

**SVG instances: 4 locations**
- ChatPage.tsx:537 -- Error icon (conditional on `historyError`)
- HistoryView.tsx:72,102 -- Error/empty icons (conditional on error/empty state)
- HistoryView.tsx:151 -- Clock icon in history cards (conditional on history view)

All SVGs are conditional -- none are always in the DOM during normal chat.
No SVG filters or animations found. This is clean.

The `ChevronLeftIcon` imported in ChatPage.tsx is likely from a library (heroicons)
and rendered as inline SVG, but only conditionally.

---

## 10. React Component Structure

### Always-mounted components in chat view:
1. `ChatPage` -- root page component
2. `ChatMessages` -- message scroll area
3. `ChatInput` -- input textarea + buttons
4. `SettingsButton` -- gear icon
5. `HistoryButton` -- history icon (custom, not from library)
6. `EmptyState` (inside ChatMessages, when no messages)

### Layout-forcing effects:

**ChatInput.tsx:80-91** -- `useEffect` that reads `textarea.scrollHeight` on every
`input` change. This forces a synchronous layout reflow. The pattern is:
```
textarea.style.height = "auto";       // triggers layout invalidation
textarea.scrollHeight;                 // forces synchronous layout
textarea.style.height = `${h}px`;     // triggers another layout
```
This is a forced reflow on every keystroke. Not an idle issue but poor during typing.

**ChatMessages.tsx:53-55** -- `useEffect` that calls `scrollIntoView({ behavior: "smooth" })`
on every message change. The smooth scroll triggers Chrome's scroll animation
compositor, which runs for ~300ms after each message.

**No `useLayoutEffect` found** -- good.

**No `getBoundingClientRect` / `offsetHeight` / `clientWidth` reads found in
active code** (one commented-out block in ChatMessages.tsx:42-50).

### TimestampComponent.tsx -- Potential idle timer:
When `mode === "relative"`, creates a `setInterval` every 60 seconds. This is
an idle-time wakeup. Not a redraw trigger per se (text update is cheap), but
worth noting. Currently only used if relative timestamps are enabled.

---

## 11. CSS Containment

**`contain:` instances: 0**
**`content-visibility:` instances: 0**

Complete absence. No CSS containment is used anywhere. This means:
- Every layout change propagates to the root
- No paint containment -- changes in message list can invalidate header paint
- No size containment -- adding messages can trigger layout on unrelated elements

This is a significant missed optimization opportunity.

---

## 12. `@media (prefers-reduced-motion)`

**Instances: 0**

Not respected anywhere. Users with reduced-motion preferences will still get:
- All `transition-*` animations
- `animate-spin` loading spinners
- `animate-pulse` thinking text
- `scrollIntoView({ behavior: "smooth" })` smooth scrolling

---

## Compositor Layer Estimate (Idle Chat View)

| Element | Reason | Always present? |
|---------|--------|----------------|
| Root document | Default | Yes |
| ChatPage root div | `transition-colors` | Yes |
| ChatMessages container | `backdrop-blur-sm` + `overflow-y-auto` | Yes |
| ChatInput textarea | `backdrop-blur-sm` + `transition-all` | Yes |
| SettingsButton | `backdrop-blur-sm` + `transition-all` | Yes |
| HistoryButton | `backdrop-blur-sm` + `transition-all` | Yes |
| Each message with `opacity-*` | Potential promotion | Yes (per message) |

**Minimum layers at idle: 6 fixed + N per message with opacity classes**

With 20 messages visible, estimate **8-12 compositor layers**.

---

## Severity Ranking

### CRITICAL (causes work on every composite frame):
1. **`backdrop-blur-sm` on ChatMessages container** -- Largest element, covers viewport,
   has scroll. Forces pixel readback + blur on every composite. This alone can cause
   measurable GPU work at idle if DWM triggers composites.
2. **`backdrop-blur-sm` on ChatInput textarea** -- Second largest always-visible element.
3. **`transition-all` on always-visible elements** (SettingsButton, HistoryButton, ChatInput) --
   Forces Chrome to watch all properties for transitions, preventing optimization.

### HIGH (creates unnecessary compositor layers):
4. **`backdrop-blur-sm` on SettingsButton and HistoryButton** -- Small elements but still
   create dedicated compositor layers with blur.
5. **`transition-colors duration-300` on root div** -- 300ms transition on the root element
   means any color inheritance change triggers a 300ms animation across the entire tree.

### MEDIUM (potential issues):
6. **No CSS containment** -- Layout changes propagate everywhere.
7. **No `prefers-reduced-motion` support** -- Cannot disable animations for sensitive users.
8. **`shadow-sm` + `backdrop-blur` combo** -- Forces shadow paint on compositor layer.

### LOW (correctly gated):
9. **`animate-spin`/`animate-pulse`** -- Only during loading, correctly conditional.
10. **Fixed positioning** -- Only on modal overlay, correctly conditional.

---

## Recommendations

### Immediate wins (no visual change):

1. **Replace `transition-all` with specific properties everywhere:**
   ```
   transition-all duration-200  -->  transition-colors duration-200
   ```
   Or more precisely: `transition-[background-color,border-color,box-shadow]`
   This prevents Chrome from watching all 400+ CSS properties.

2. **Remove `backdrop-blur-sm` from ChatMessages and ChatInput:**
   These are the two largest elements. Replace with solid backgrounds:
   ```
   bg-white/70 backdrop-blur-sm  -->  bg-white dark:bg-slate-800
   ```
   The blur effect on a chat message area provides negligible visual value
   but forces continuous GPU work.

3. **Remove `backdrop-blur-sm` from SettingsButton and HistoryButton:**
   Small buttons do not benefit from background blur. Use solid backgrounds.

4. **Add CSS containment to message container:**
   ```css
   .messages-container {
     contain: layout style paint;
     content-visibility: auto;
   }
   ```
   This prevents message list changes from triggering layout/paint outside
   the container.

5. **Add `@media (prefers-reduced-motion)` to index.css:**
   ```css
   @media (prefers-reduced-motion: reduce) {
     *, *::before, *::after {
       animation-duration: 0.01ms !important;
       transition-duration: 0.01ms !important;
     }
   }
   ```

6. **Remove `transition-colors duration-300` from root div:**
   The root page div does not need a 300ms color transition. Dark mode
   switching can be instant.

### Deeper fixes:

7. **Replace `scrollIntoView({ behavior: "smooth" })` with instant scroll
   or use `requestAnimationFrame` + manual easing** to avoid Chrome's
   built-in smooth scroll compositor animation.

8. **Add `contain: content` to individual message components** to prevent
   layout thrashing when new messages arrive.

9. **Consider removing all semi-transparent backgrounds (`bg-white/70`,
   `bg-white/80`)** -- these force alpha blending on every composite.
   Solid backgrounds are free.
