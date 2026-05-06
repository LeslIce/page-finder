# CLAUDE.md

## Project Overview

**Page Finder** (单词笔记本) is a lightweight single-file HTML5 PWA for recording and indexing handwritten vocabulary notebook entries. It targets language learners who maintain physical vocabulary notebooks and need a digital index for quick lookup.

Live at: https://leslice.github.io/page-finder/

## Architecture

### Single-File Design

The entire application lives in `index.html` (~2,860 lines). All HTML, CSS, and JavaScript are contained in one file — this is an intentional design constraint for simple deployment to GitHub Pages and PWA installation on iPhone home screens.

**Do not split this into multiple files** unless explicitly requested.

Approximate breakdown:
- HTML structure: ~215 lines
- CSS: ~750 lines (lines 14–950)
- JavaScript: ~1,650 lines (inside a single IIFE, lines ~1199–2856)

### Technology Stack

- **Vanilla JavaScript (ES6+)** — no frameworks, no build step, zero external dependencies
- **IndexedDB** (`VocabNotebook` database, version 1) for persistent client-side storage
- **Pure CSS3** with CSS Grid, Flexbox, and CSS variables (mint color palette)
- **PWA** capabilities via dynamically generated manifest and app icons from inline SVG
- **OpenAI-compatible API** integration (DeepSeek / deepseek-v4-flash) for optional word definitions

### Key Architectural Patterns

**IIFE wrapper** — All code runs inside a single IIFE with `"use strict"`.

**State object** — Central `state` object holds all app state (23 properties):
```
state.db                       // IndexedDB connection
state.notebooks                // Cached notebook records array
state.words                    // Cached word records array
state.currentView              // Active view name string
state.currentNotebookId        // Selected notebook ID
state.sortMode                 // "page" | "alpha" | "time"
state.sortReversed             // Boolean reverse sort
state.filterMode               // "all" | "mainOnly"
state.searchOrigin             // "shelf" | "list" (where search was opened from)
state.pendingHighlightWordId   // Word ID to highlight after render
state.editingWordId            // Word being edited
state.modalResolver            // Promise resolver for dialog
state.searchMeaningController  // AbortController for API requests
state.searchMeaningTimer       // Debounce timeout for API calls
state.searchMeaningToken       // Numeric cancellation token
state.listSwipeMountTimer      // Swipe animation setup timeout
state.listHighlightTimer       // Highlight effect timeout
state.listSwipeGuardUntil      // Timestamp guard against accidental swipes
state.entryViewportRaf         // RAF ID for viewport tracking
state.entryViewportTimers      // Array of timeouts for viewport sync
state.entryKeyboardLift        // Numeric offset for keyboard avoidance
state.entryFocusMode           // "page" | "word"
```

**DOM element cache** — `els` object caches ~34 DOM references, populated by `cacheElements()` at init time. All DOM element IDs are referenced through this cache.

**View system** — 5 views toggled via `showView(name)`, each a `<section class="view">`:
1. `shelfView` — Notebook grid/library
2. `listView` — Word list for a notebook (sort/filter/swipe-to-delete)
3. `entryView` — New word entry form
4. `settingsView` — API key/model config, data import/export
5. `searchView` — Global search with API meaning lookups

Plus two overlay modals: `modalOverlay` (confirmation dialogs) and `editOverlay` (word editing).

**Data flow** — In-memory caches (`state.notebooks`, `state.words`) synced with IndexedDB via Promise-wrapped helpers (`getAllRecords`, `getRecord`, `putRecord`, `deleteRecord`, `addRecord`). Call `refreshCaches()` to reload from DB.

### Database Schema (IndexedDB: `VocabNotebook` v1)

- **notebooks**: `id` (auto-increment keyPath), `name`, `order` (indexed), `createdAt` (indexed)
- **words**: `id` (auto-increment keyPath), `notebookId` (indexed), `word`, `page`, `isAnalysis`, `createdAt` (indexed)
- **appState**: `id` (string keyPath, always `"main"`), `lastNotebookId`, `lastSortMode`, `lastFilterMode`, `lastSortReversed`, `apiKey`, `modelName`

DB helpers: `openDatabase()`, `getStore()`, `requestToPromise()`, `ensureAppState()`, `getAppState()`, `saveAppState(patch)`.

### CSS Variables

Defined in `:root`:

| Variable | Value | Purpose |
|---|---|---|
| `--mint-rgb` | `132, 203, 171` | RGB tuple for opacity variants |
| `--mint-solid` | `#84cbab` | Full opacity mint |
| `--mint` | `rgba(var(--mint-rgb), 0.82)` | Primary interactive color |
| `--mint-soft` | `rgba(var(--mint-rgb), 0.22)` | Soft backgrounds |
| `--mint-faint` | `rgba(var(--mint-rgb), 0.08)` | Very subtle tint |
| `--mint-flash-strong` | `rgba(var(--mint-rgb), 0.16)` | Highlight strong |
| `--mint-flash-weak` | `rgba(var(--mint-rgb), 0.09)` | Highlight weak |
| `--gray` | `#5f5d5d` | Secondary text |
| `--gray-deep` | `#4c4b4b` | Dark text |
| `--line` | `rgba(0, 0, 0, 0.16)` | Borders |
| `--button-gray` | `#cfcfd4` | Disabled buttons |
| `--keyboard-offset` | `0px` | iOS soft keyboard compensation |
| `--keyboard-lift` | `0px` | Viewport shift for keyboard avoidance |
| `--safe-*` | `env(safe-area-inset-*)` | Notch/home indicator safe areas |
| `--ui-sans` | Quicksand, system fonts | UI font stack |
| `--title-sans` | System fonts | Title font stack |

### Key Functions

**Lifecycle:**
- `init()` — App entry point; caches DOM, opens DB, restores previous state
- `cacheElements()` — Populates `els` object
- `setupAppChrome()` — Creates manifest and apple-touch-icon from inline SVG
- `bindStaticEvents()` — Attaches all event listeners
- `restoreApp()` — Restores last viewed notebook/view from appState

**Navigation:**
- `showView(name)` — Toggle view visibility
- `openShelf()` / `openNotebook(id, options)` / `openEntry()` / `openSettings()` / `openSearch(origin)`

**Rendering:**
- `renderShelf()` — Notebook grid with long-press delete (560ms threshold)
- `renderList(options)` — Word list with sorting, filtering, swipe actions, highlights
- `renderSearchResults()` — Search results with debounced API meaning lookups

**Data operations:**
- `refreshCaches()` — Reload notebooks/words from IndexedDB
- `createNotebook()` — Create notebook with auto-incremented order
- `handleEntrySave(mode)` — Save word ("stay" keeps form, "next" advances page)
- `saveEditedWord()` — Update existing word
- `deleteNotebookCascade(notebookId)` — Delete notebook + all words, reorder remaining
- `exportData()` — Export all data as timestamped JSON file
- `importData(file)` — Import JSON backup, clears all stores, reloads page

**API:**
- `fetchWordMeaning(word, apiKey, modelName, signal)` — Call DeepSeek OpenAI-compatible API with AbortSignal
- `scheduleSearchMeanings(query, meaningTargets, token)` — Debounced concurrent meaning lookups (320ms delay)
- `cancelMeaningRequests()` — Abort in-flight API calls and clear timers

**Swipe & touch:**
- `setupSwipe(item)` — Pointer event handlers for swipe-to-delete (120px threshold)
- `closeAllSwipes(exceptItem)` — Reset all swipe offsets

**Dialogs:**
- `showDialog({ title, text, buttons })` — Promise-based confirmation dialog
- `openEditModal(record)` / `closeEditModal()`

### API Integration

- **Endpoint:** `https://api.deepseek.com/chat/completions`
- **Default model:** `deepseek-v4-flash` (user-configurable in settings)
- **Auth:** Bearer token via `Authorization` header
- **Temperature:** 0.3
- **Cancellation:** `AbortController` + numeric `searchMeaningToken` (incremented per query; old responses ignored)
- **Network check:** `navigator.onLine` before scheduling; `window.addEventListener("online")` re-triggers lookups
- **Debounce:** 320ms delay before API execution
- **Graceful degradation:** Missing API key skips lookup entirely; API failures return empty string

### PWA Details

- Manifest dynamically generated in `setupAppChrome()` and injected as data URL
- App name: "单词笔记本", short name: "单词本"
- Display: standalone, theme color: `#84cbab`
- Icons: inline SVG (notepad/pencil) as data URL, declared for 192x192 and 512x512
- Apple-specific: `apple-touch-icon`, `apple-mobile-web-app-capable: yes`, `apple-mobile-web-app-title: 单词本`
- **No service worker** — intentional single-file limitation; relies on browser HTTP cache + IndexedDB for offline data

### Event Handling

- **Pointer events** (`pointerdown`/`pointermove`/`pointerup`/`pointercancel`) for swipe detection with `pointerId` tracking
- **Scroll** closes all open swipes (passive listener)
- **Long-press** (560ms) on notebooks triggers delete confirmation
- **Keyboard:** Enter saves settings; Escape closes edit modal
- **Input events** with immediate validation (page: digits-only, max 3 chars)
- **Click outside** overlay dismisses modals
- **Haptic feedback** via Vibration API where supported
- **Reduced motion** via `@media (prefers-reduced-motion: reduce)`

### Import/Export Format

```json
{
  "notebooks": [{ "id": 1, "name": "...", "order": 1, "createdAt": 1234567890 }],
  "words": [{ "id": 1, "notebookId": 1, "word": "...", "page": "001", "isAnalysis": false, "createdAt": 1234567890 }],
  "appState": { "id": "main", "lastNotebookId": 1, "lastSortMode": "page", "lastFilterMode": "all", "lastSortReversed": false, "apiKey": "", "modelName": "" }
}
```

Export creates `vocab_notebook_[timestamp].json`. Import clears all stores, re-inserts data, and reloads the page.

## Development

### Setup

No build step required. Open `index.html` in a browser (non-private mode — IndexedDB needs it).

### Deployment

Push to `master` — GitHub Pages serves `index.html` automatically. No CI/CD pipelines.

### Testing

No automated tests. Test manually in a browser, especially on mobile Safari for PWA/touch interactions.

### Linting & Formatting

No linting or formatting tools configured. Follow existing conventions:
- 2-space indentation
- camelCase for functions and variables
- Chinese comments for feature explanations
- DOM element IDs referenced via `els` cache

## Conventions

### Code Style

- All new code goes inside the existing IIFE in `index.html`
- Prefer `const`/`let` over `var`
- Use async/await for IndexedDB and API operations
- Chinese UI strings throughout (the app is in Chinese)
- Comments in Chinese where describing user-facing behavior

### UI/UX

- Mobile-first, iPhone-optimized (safe area insets, standalone PWA mode)
- iOS-style swipe gestures with custom pointer event handlers
- Mint green color palette via CSS variables
- Reduced motion support via `@media (prefers-reduced-motion: reduce)`
- Haptic feedback via Vibration API where supported

### Important Constraints

- **Single-file architecture** — everything stays in `index.html`
- **Zero dependencies** — no npm packages, no CDN libraries
- **Offline-capable** — must work without network after initial load
- **No service worker** — intentional limitation of single-file design
- **iPhone primary target** — test touch interactions, safe areas, standalone mode

## File Structure

```
page-finder/
├── index.html    # Complete application (HTML + CSS + JS, ~2,860 lines)
├── README.md     # User-facing documentation (Chinese)
└── CLAUDE.md     # This file
```
