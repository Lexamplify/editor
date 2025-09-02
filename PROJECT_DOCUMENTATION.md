## Project Documentation

### 1) Project Overview

Umo Editor (package name: "lexteam-editor") is an open-source rich-text document editor built with Vue 3 and Tiptap 2. It provides pagination, Markdown input rules, a comprehensive toolbar, block-level editing, AI assistant, multiple export formats (HTML/Image/PDF/Word), watermarking, page layout controls, and multi-language support. The UI uses TDesign for Vue. The project is bundled and exposed as a library via Vite and emits TypeScript types.

- Frameworks: Vue 3, Vite 4, TypeScript 5
- Core editor: Tiptap 2 with many official/pro extensions
- UI: `tdesign-vue-next`
- Utilities: `@vueuse/core`, `vue-i18n`, `tippy.js`, `dom-to-image-more`, `highlight.js`, `lowlight`, `katex`, `mermaid`, `plyr`, etc.

Entry point and app init

- `index.html` mounts the Vue app to `#app` and loads `src/main.ts`.
- `src/main.ts` creates the app with `src/app.vue`, installs the plugin `useUmoEditor` (from `src/components/index.ts`) with global options, and mounts the app.

```ts
// src/main.ts (abridged)
import App from './app.vue'
import { useUmoEditor } from './components'
const app = createApp(App)
app.use(useUmoEditor, {})
app.mount('#app')
```

Key scripts (package.json)

- dev: `vite --port 9000 --open --force`
- build: `vue-tsc --noEmit && vite build` (library build)
- test: `vitest`

### 2) File Structure & Architecture

```bash
editor/
├─ index.html
├─ vite.config.ts
├─ tsconfig.json
├─ vitest.config.ts
├─ eslint.config.ts
├─ README.md
├─ public/
│  └─ favicon.png
├─ dist/                  # Library build output (ES module + CSS)
├─ types/                 # Emitted type declarations for consumers
│  ├─ components.d.ts
│  ├─ imports.d.ts
│  ├─ index.d.ts
│  └─ src/
│     └─ components/
├─ src/
│  ├─ app.vue             # Demo app showcasing <umo-editor />
│  ├─ main.ts             # App bootstrap
│  ├─ i18n.ts             # vue-i18n setup (en-US)
│  ├─ assets/
│  │  ├─ icons/           # SVGs registered by vite-plugin-svg-icons
│  │  ├─ images/          # Static images (logo, watermark, etc.)
│  │  └─ styles/          # Less styles and theme overrides
│  ├─ components/         # Library components
│  │  ├─ index.ts         # Plugin install + named exports
│  │  ├─ index.vue        # UmoEditor (root editor UI)
│  │  ├─ editor/index.vue # Tiptap EditorContent wrapper
│  │  ├─ toolbar/         # Toolbar implementations (classic/ribbon + menus)
│  │  ├─ container/       # Page container, TOC, search/replace, print
│  │  ├─ menus/           # Bubble/context menus and submenus
│  │  ├─ statusbar/       # Status bar (zoom, layout, preview, lang)
│  │  ├─ ai/assistant/    # AI assistant input
│  │  ├─ modal.vue, tooltip.vue, icon.vue, page-options.vue
│  │  └─ color-picker.vue
│  ├─ composables/        # App-wide composables (state, i18n helper, dialogs)
│  │  ├─ state.ts         # Persistent state via @vueuse/core useStorage
│  │  ├─ i18n.ts          # i18n helpers (t, l, useI18n)
│  │  └─ *.ts
│  ├─ extensions/         # Tiptap extensions and registry
│  │  ├─ index.ts         # getDefaultExtensions(), inputAndPasteRules()
│  │  └─ */ (nodes, marks, menus, tools: table, image, callout, etc.)
│  ├─ locales/            # Translation JSON
│  ├─ options/            # Default options, dicts, AI options, webpages
│  ├─ utils/              # Helpers (options merge, content transform, etc.)
│  └─ types/              # Ambient types for the library
└─ package.json
```

Folder purposes

- `src/components`: All Vue single-file components composing the editor UI and UX.
- `src/extensions`: Tiptap extensions (nodes, marks, features) bundled into the editor.
- `src/composables`: Reusable logic (state persistence, i18n helpers, popups).
- `src/options`: Default `UmoEditorOptions`, dictionaries, and option schema/merging.
- `src/utils`: Cross-cutting utilities: `getOpitons`, content transforms, ID/shortcuts.
- `src/assets`: Less styles, icons registered as sprites, and static images.
- `types/`: Generated declaration files for consumers of the library build.

Highlighted files

- `vite.config.ts`: Library build config, Vue macros, auto-import, components resolver, svg-icons, DTS.
- `index.html`: Demo host page that mounts the app.
- `src/main.ts`: Creates the Vue app and installs the editor plugin.

### 3) Components Documentation

Key component: UmoEditor

- File: `src/components/index.vue`
- Purpose: The root editor component that composes toolbar, page container (with Tiptap editor content), and statusbar. Provides/provides state and exposes editor API methods to parents via `defineExpose`.
- Props: Accepts all keys defined by `propsOptions` from `src/options/index.ts`, which mirrors `UmoEditorOptions` (toolbar, page, document, ai, echarts, webPages, templates, file, importWord, users, theme, locale, etc.). Props are merged with defaults and global plugin options via `getOpitons`.
- State: Uses Vue `provide/inject` and `@vueuse/core` `useStorage` through `useState(...)` for persisted slices: `toolbar`, `document`.
- Emits: `created`, `changed`, `changed:*` (toolbar/page variants), `focus`, `blur`, `print`, `saved`, `destroy`, `menuChange`, `contentError`.
- Important methods (exposed): `setOptions`, `setToolbar`, `setLayout`, `setPage`, `setWatermark`, `setDocument`, `setContent`, `insertContent`, `startTypewriter/stopTypewriter/getTypewriterState`, `getContent` (html/text/json), `getImage` (blob/jpeg/png), `getVanillaHtml`, `saveContent`, `getContentExcerpt`, `useEditor/getEditor`, `getTableOfContents`, selection helpers, `setLocale`, `setTheme`, `setReadOnly`, `focus/blur`, `toggleFullscreen`, `reset`, `destroy`, bookmark helpers, and message helpers (`useAlert/Confirm/Message`).
- Dependencies: Imports `propsOptions` and `getOpitons` from `src/options` and `src/utils/options`, `i18n`, tiptap selection utils, dom-to-image, `@tool-belt/type-predicates`, and TDesign locale packs. Provides many contexts used by child components.

Editor content wrapper

- File: `src/components/editor/index.vue`
- Purpose: Initializes the Tiptap `Editor` with `getDefaultExtensions()` and binds to the DOM via `<EditorContent />`. Syncs content to `document` state with throttling.
- Key props/state: Uses provided `options`, `page`, `uploadFileMap`, and `document` state.
- Hooks: Creates/destroys the Editor, onUpdate throttles syncing, dynamically loads KaTeX CSS when math extension is enabled.

Toolbar

- File: `src/components/toolbar/index.vue`
- Purpose: Renders the toolbar in either ribbon or classic mode, with dynamic menu slots and actions (status popup, toggle mode, save/restore from cache).
- State: Reads `$toolbar` from persistent state; tracks `savedAt`, online status; emits `menu-change` to parent.
- Dependencies: Uses `status` popup, TDesign components, `useState`, injected `editor`, and `timeAgo` util.

Page container

- File: `src/components/container/page.vue`
- Purpose: Provides the zoomable page area with margins, watermark overlay, and header/footer gutters. Hosts the `editor` component, TOC, image viewer, back-to-top, search/replace, and print.
- State: Reads `page` to compute page size/orientation and zoom, tracks image preview state, recalculates height when content/layout changes.

Status bar

- File: `src/components/statusbar/index.vue`
- Purpose: Controls layout (page/web), zoom, preview mode with fullscreen, language dropdown, toggles spellcheck, shortcuts drawer, and quick actions.
- State: Uses `page`, `$document`, `editor`, and injected `setLocale`/`reset`.

Bubble menu

- File: `src/components/menus/bubble/index.vue`
- Purpose: A floating bubble menu using Tiptap `BubbleMenu`, switching between editing menus and AI assistant input; updates tippy placement based on assistant visibility.

Other notable components

- `src/components/menus/*`: Rich set of bubble/context menu actions for links, images, files, nodes, and tags.
- `src/components/toolbar/*/(base|insert|table|tools|page|export)`: Menu groups with many actions mapped to Tiptap commands.

Component import relationships (high-level)

- `app.vue` -> uses `<umo-editor />` (registered by plugin from `components/index.ts`).
- `components/index.vue` (UmoEditor) -> `toolbar`, `container/page.vue`, `statusbar`.
- `container/page.vue` -> `editor/index.vue` (the Tiptap instance), TOC/print/search.
- `editor/index.vue` -> `extensions/index.ts` (`getDefaultExtensions`).

### 4) Data Flow & State Management

- No Vuex/Pinia. The project uses `provide/inject` for cross-component state and `useState(key, options)` to bind slices to `localStorage` via `@vueuse/core` `useStorage`.
- Global plugin options are provided via `app.use(useUmoEditor, options)` and then merged with component props and defaults by `getOpitons`.
- Editor events propagate through watchers in `components/index.vue` and are re-emitted as component events.
- Parent-to-child data: `options` and derived states (`page`, `$document`, `$toolbar`) are provided to descendants. Child components react to these via `watch`/computed.

### 5) APIs / Services

External APIs

- The core project does not hardcode HTTP APIs. Instead it exposes hooks for host applications:
  - `onSave(content, page, document)`: async callback to persist content.
  - `onFileUpload(file)`: async callback to upload a file; editor inserts returned metadata.
  - `onFileDelete(id, url)`: callback for deletions.
  - `ai.assistant.onMessage()`: async callback to produce AI assistant responses.

Data fetching

- No `axios`/`fetch` utilities are included. Integrations are supplied by the host app through the callbacks above.

Services/utilities

- `src/utils/options.ts` (getOpitons) merges defaults, global plugin options, and component props using an `ObjectSchema` validator.
- `src/extensions/*` include logic for content insertion, file handling, ECharts, mentions, code blocks, tables, etc.

### 6) Styling & Assets

- CSS preprocessor: Less
- Styles: `src/assets/styles/*.less` with component-level imports in SFCs. TDesign variables are customized via CSS variables and Less.
- Icons: SVGs under `src/assets/icons` are bundled into a sprite with `vite-plugin-svg-icons` and used via `<icon name="..." />` and `.umo-icon` elements.
- Images: `src/assets/images` hosts watermark presets, logo, and UI media used in previews and exports.

### 7) Build & Configuration

Vite configuration (`vite.config.ts`)

- Plugins:
  - Vue + Vue Macros (Reactivity Transform)
  - `unplugin-auto-import` (auto imports from Vue and `@vueuse/core`), emits to `types/imports.d.ts`
  - `unplugin-vue-components` with `TDesignResolver`
  - `vite-plugin-svg-icons` for sprite registration
  - `vite-plugin-dts` to emit library types to `types/`
  - `vite-tsconfig-paths` for `@` alias
- Build: library mode with entry `src/components/index.ts`, name from `package.json`, output `dist/umo-editor.js` with `style.css` and externals (`vue`, tiptap packages, vueuse, etc.).
- Base: `/umo-editor` for the demo preview pathing.

Build scripts (package.json)

- `build`: typecheck (`vue-tsc`) then `vite build`
- `prepublishOnly`: runs build to produce `dist` and `types` for publishing

### 8) Suggested Improvements

- Strong types for `inject` usage: Introduce dedicated injection keys with symbols and type helpers to avoid many `any` casts and improve DX.
- Expand i18n coverage: Add missing locales (currently en-US only in `src/i18n.ts`) and surface locale-switching in demo.
- Option typing surfacing: Export a public `UmoEditorOptions` interface from a `.ts` file within `src/types` (in addition to emitted d.ts) to simplify host imports.
- Unit tests: Increase test coverage for `utils/options` schema validations and critical extensions; add component tests for toolbar behaviors.
- Async/task states: Centralize async UI feedback (e.g., `saveContent`) into a small service and provide consistent toasts/indicators.
- Accessibility: Audit ARIA roles/labels on toolbar buttons, menus, and dialogs.

## Quick Start (Library Consumer)

```ts
import { createApp } from 'vue'
import UmoEditor, { useUmoEditor } from 'lexteam-editor'
import 'lexteam-editor/style'

createApp({
  template: '<umo-editor />',
})
  .use(useUmoEditor, {
    /* global UmoEditorOptions */
  })
  .mount('#app')
```
