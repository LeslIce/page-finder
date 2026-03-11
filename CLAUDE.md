# CLAUDE.md

## Project Overview

**Page Finder** (单词笔记本) is a lightweight single-file HTML5 PWA for recording and indexing handwritten vocabulary notebook entries. It targets language learners who maintain physical vocabulary notebooks and need a digital index for quick lookup.

Live at: https://leslice.github.io/page-finder/

## Architecture

### Single-File Design

The entire application lives in `index.html` (~2,800 lines). All HTML, CSS, and JavaScript are contained in one file — this is an intentional design constraint for simple deployment to GitHub Pages and PWA installation on iPhone home screens.

**Do not split this into multiple files** unless explicitly requested.

### Technology Stack

- **Vanilla JavaScript (ES6+)** — no frameworks, no build step, zero external dependencies
- **IndexedDB** for persistent client-side storage
- **Pure CSS3** with CSS Grid, Flexbox, and CSS variables (mint color palette)
- **PWA** capabilities via dynamically generated manifest and app icons from inline SVG
- **OpenAI-compatible API** integration (Silicon Flow) for optional word definitions

### Key Architectural Patterns

**State object** — Central `state` object holds all app state (DB connection, cached data, UI state):
```
state.db, state.notebooks, state.words, state.currentView, state.currentNotebookId, etc.
```

**DOM element cache** — `els` object caches DOM references, populated at init time.

**View system** — 5 views (shelf, list, entry, settings, search) toggled via `showView(name)`. Each is a `<section class="view">`.

**Data flow** — In-memory caches (`state.notebooks`, `state.words`) synced with IndexedDB via Promise-wrapped helpers (`getAllRecords`, `getRecord`, `putRecord`, `deleteRecord`). Call `refreshCaches()` to reload from DB.

**IIFE wrapper** — All code runs inside a single IIFE with `"use strict"`.

### Database Schema (IndexedDB)

- **notebooks**: `id` (auto), `name`, `order` (indexed), `createdAt`
- **words**: `id` (auto), `notebookId` (indexed), `word`, `page`, `isAnalysis`, `createdAt` (indexed)
- **appState**: `id` ("main"), `lastNotebookId`, `lastSortMode`, `lastFilterMode`, `apiKey`, `modelName`

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

### API Integration

- OpenAI-compatible chat endpoint (default: Silicon Flow / Qwen3-8B)
- User-configurable API key and model in settings
- Concurrent request management with cancellation tokens
- Network availability check before scheduling API calls

## File Structure

```
page-finder/
├── index.html    # Complete application (HTML + CSS + JS)
├── README.md     # User-facing documentation (Chinese)
└── CLAUDE.md     # This file
```
