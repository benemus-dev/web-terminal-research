# Redraw Audit: claudecodeui -- Hidden Chrome Compositor Triggers

## Summary

claudecodeui is the heaviest project by far. It has **399 transition/animation occurrences across 106 files**, aggressive backdrop-filter usage on always-visible elements, multiple always-on compositor layers, and extensive custom scrollbar styling. However, it does respect `prefers-reduced-motion`.

---

## 1. CSS Transitions on Always-Visible Elements

**Count: HIGH -- transitions on almost every DOM element**

The most impactful finding is in `index.css:122-197`:

```css
/* GLOBAL: transition: none on *, then re-enabled for interactive elements */
* { transition: none; }

/* Re-enabled for buttons, links, inputs -- ALWAYS PRESENT */
button, a, input, textarea, select, [role="button"], .transition-all {
    transition: all 150ms cubic-bezier(0.4, 0, 0.2, 1);
}

/* EVERY structural element gets color transitions -- ALWAYS ACTIVE */
body, div, section, article, aside, header, footer, nav, main,
h1, h2, h3, h4, h5, h6, p, span, blockquote, ... {
    transition: background-color 200ms, border-color 200ms, color 200ms;
}
```

This means **every single div, span, p, header, footer, nav** element in the page has an active `transition` property. Chrome's style system must track transition eligibility on thousands of elements even when nothing is animating. While transitions only trigger compositing when a property actually changes, the style tracking overhead is non-trivial.

Additional always-visible transition classes in components:
- `.sidebar-transition`: `transform 300ms, opacity 300ms` (sidebar)
- `.details-chevron`: `transform 200ms` (tool details, always in DOM)
- `.message-transition`: `opacity 300ms, transform 300ms` (chat messages, always in DOM)

**Named animations always defined:**
- `spin` keyframe (index.css:6, 767) -- used by `.animate-spin` and `.loading-spinner`
- `settings-fade-in` keyframe (index.css:909)
- `search-flash` keyframe (index.css:923)

---

## 2. will-change Usage

**Count: 1 always-active**

- `.loading-spinner` (`index.css:783`): `will-change: transform` -- forces compositor layer promotion. This class is used on loading spinners which are conditional, but the CSS rule is always parsed.

---

## 3. position: fixed/sticky/absolute Elements

**Count: 30+ fixed elements across components (most conditional)**

Always-visible fixed/sticky elements:
- **QuickSettingsHandle** (`QuickSettingsHandle.tsx:55`): `fixed` positioned drag handle, always rendered. Creates a compositor layer.
- **Mobile nav** (`ChatComposer.tsx:168`): `max-sm:fixed max-sm:bottom-0` -- fixed on mobile, always visible.
- **SidebarContent** uses backdrop-blur which forces layer creation.
- **BranchesView** (`BranchesView.tsx:116`): `sticky top-0` header -- always creates a compositor layer.

Conditional fixed elements (modals, overlays -- ~20 instances):
- VersionUpgradeModal, Settings, CodeEditor, PrdEditor, ImageViewer, ConfirmActionModal, NewBranchModal, TaskMasterSetupModal, TaskHelpModal, CreateTaskModal, ProjectCreationWizard, FolderBrowserModal, ProviderLoginModal, etc.

**Compositor layers when idle: 5-8** depending on which panels are open.

---

## 4. backdrop-filter / filter Usage

**Count: 27 occurrences -- SIGNIFICANT**

Always-visible backdrop-filter elements:
- `.nav-glass` (`index.css:264-266`): `backdrop-filter: blur(20-24px) saturate(1.6-1.8)` -- applied to navigation. **Always visible, always composited.**
- `SidebarContent` (`SidebarContent.tsx:96`): `backdrop-blur-sm` -- always visible sidebar.
- `SidebarCollapsed` (`SidebarCollapsed.tsx:30`): `backdrop-blur-sm` -- always visible collapsed sidebar.
- `MainContentStateView` (`MainContentStateView.tsx:14`): `backdrop-blur-sm` -- header area, always visible.
- `ChatComposer` (`ChatComposer.tsx:280`): `backdrop-blur-sm` -- always visible input area.
- `ClaudeStatus` (`ClaudeStatus.tsx:113`): `backdrop-blur-md` -- visible during active sessions.
- `ConversationView` (CUI, but similar pattern): `backdrop-blur-sm` on sticky footer.

Conditional backdrop-filter (modals):
- ~15 modal overlays use `backdrop-blur-sm` but only when open.

**Impact:** Each `backdrop-filter` element forces Chrome to:
1. Create a separate compositor layer
2. Render all content behind it to a texture
3. Apply the blur shader every frame the layer is visible
4. Re-blur on any scroll or layout change in content behind it

This is the single largest source of hidden GPU work in this project.

---

## 5. CSS Containment

**Status: COMPLETELY ABSENT**

No `contain:` or `content-visibility:` anywhere in the 937-line index.css or any component. Given the scale of this app (100+ components, dozens of scroll containers), this is a significant missed optimization.

---

## 6. Scroll Containers

**Count: 51 occurrences of overflow-auto/scroll across 43 files**

Always-visible scroll containers:
- Chat messages pane (`ChatMessagesPane.tsx`): `overflow-y-auto` -- main content scroll
- Sidebar content: `overflow-y-auto` -- session list scroll
- File tree body: `overflow-y-auto`
- Code editor surface: `overflow-auto`
- Settings panels: multiple `overflow-y-auto`
- Task board content: `overflow-y-auto`

Each scroll container with visible content creates a paint surface. When content is larger than the container, Chrome maintains a scrollable layer.

---

## 7. Tailwind Animation/Transition Classes on Always-Visible Elements

**Count: HIGH**

Always-visible Tailwind transition classes found:
- `transition-colors` on ConversationHeader, TaskTabs, multiple buttons
- `transition-all duration-300` on Composer container (always visible)
- `transition-all` on Login button, DropdownSelector items
- `animate-spin` on Loader2 components (conditional, during loading)
- `animate-in`, `slide-in-from-*`, `fade-in-*` on dialog/popover components (conditional)
- `duration-200`, `duration-250`, `duration-300` on various Dialog transitions

---

## 8. Box-shadow on Permanent Elements

**Count: 91 occurrences across 55 files**

Always-visible shadows:
- `.nav-tab-active` (`index.css:271`): multi-layer box-shadow with glow effect
- `.mobile-nav-float` (`index.css:277`): multi-layer shadow with ring
- `.chat-input-expanded` (`index.css:835`): elevation shadow on textarea
- `Composer` (`Composer.tsx:854`): `shadow-sm` -- always visible
- Sidebar items with shadows on hover states

Each box-shadow on a composited layer forces Paint on every frame that layer is dirty.

---

## 9. SVG Icons Always in DOM

**Count: 169 icon imports across 122 files**

claudecodeui uses `lucide-react` icons extensively. These are inline SVG elements. Key always-visible SVG icons:
- Sidebar: folder icons, session icons, settings gear, collapse/expand chevrons
- Chat: send button icon, attachment icon, thinking mode icons
- File tree: file type icons (per-file SVGs)
- Navigation: tab icons

lucide-react icons are simple path-based SVGs without gradients or filters, so they don't cause per-frame paint on their own. However, when combined with `transition-colors` or `transition-opacity` classes, they participate in transition tracking.

---

## 10. Custom Scrollbar CSS

**Count: EXTENSIVE (index.css:316-380)**

```css
.scrollbar-thin::-webkit-scrollbar { width: 6px; height: 6px; }
.scrollbar-thin::-webkit-scrollbar-track { background: transparent; }
.scrollbar-thin::-webkit-scrollbar-thumb { background-color: ...; border-radius: 3px; }
```

Also:
- `.dark::-webkit-scrollbar` styles (index.css:358-374)
- `.chat-input-placeholder::-webkit-scrollbar` styles (index.css:802-819)
- `.scrollbar-hide::-webkit-scrollbar { display: none; }` (index.css:753)

Custom scrollbar styling creates additional paint surfaces per scroll container. With 51 scroll containers, this is multiplicative.

---

## 11. prefers-reduced-motion Support

**Status: RESPECTED (index.css:220-229)**

```css
@media (prefers-reduced-motion: reduce) {
    *, ::before, ::after {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
        scroll-behavior: auto !important;
    }
}
```

This correctly kills all animations and transitions. Users with this OS setting enabled will have a dramatically quieter compositor. However, it does NOT disable `backdrop-filter` or remove compositor layers.

---

## Compositor Layer Estimate (Idle State)

| Layer | Source | Always Present |
|-------|--------|----------------|
| Root document | html/body | Yes |
| Sidebar | backdrop-blur-sm, z-index | Yes |
| Navigation glass | backdrop-filter: blur(20px) | Yes |
| Main content header | backdrop-blur-sm | Yes |
| Chat composer | backdrop-blur-sm + border | Yes |
| Quick settings handle | position: fixed | Yes |
| Mobile nav (mobile) | position: fixed + shadow | Yes (mobile) |
| Chat messages scroll | overflow-y-auto | Yes |
| File tree scroll | overflow-y-auto | Conditional |
| Code editor | overflow-auto | Conditional |

**Total: 6-10 compositor layers** (idle, desktop)

---

## Idle Rendering Pipeline Quietness

**Rating: NOISY**

Even when completely idle (no user interaction, no AI activity):
1. **backdrop-filter on 4-5 always-visible elements** forces continuous GPU compositing work. Any sub-pixel scroll, layout shift, or theme-related property change behind these elements triggers re-blur.
2. **Universal CSS transitions** (`background-color 200ms` on every structural element) means Chrome's style system is tracking transition eligibility on hundreds of DOM nodes.
3. **Custom scrollbar paint surfaces** on 51 scroll containers.
4. **6-10 compositor layers** require per-frame compositing even when nothing moves.
5. The `spin` animation is only active during loading states (conditional).

The saving grace is `prefers-reduced-motion` support, which kills transitions/animations but not `backdrop-filter` compositing.

### Highest-impact fixes:
1. Remove `backdrop-filter` from always-visible elements (sidebar, nav, header, composer). Replace with opaque backgrounds.
2. Remove the universal structural-element transition rule (`body, div, section, ...`). Only apply transitions to elements that actually change color during user interaction.
3. Add `contain: content` to scroll containers and isolated components.
4. Consider `content-visibility: auto` for off-screen chat messages.
