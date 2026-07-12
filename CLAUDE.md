# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page wedding planning PWA. There is no build step, no package manager, and no test suite — the entire app is `index.html` (inline `<style>` + inline `<script>`, vanilla JS, no framework, no dependencies) plus a minimal `sw.js` service worker for offline caching. `manifest.json` and the two icon PNGs make it installable as a PWA.

## Running it

There's no dev server or build command. Serve the directory statically and open it, e.g.:

```bash
python3 -m http.server 8934
```

then visit `http://localhost:8934/index.html`. Opening the file directly via `file://` mostly works but the service worker (which registers with the absolute path `sw.js`) and the `/wedding-planner/` absolute paths in `manifest.json`/`sw.js`'s `FILES` list assume it's deployed at that subpath — keep that in mind if paths ever need editing.

No lint/test/build commands exist in this repo.

## Every push: bump the version

`index.html` defines `APP_VERSION` and `APP_UPDATED` near the top of the `<script>` block — these drive the "App updated … · vNN" text in the page footer. `sw.js` separately defines `CACHE = 'wedding-planner-vNN'`, which controls cache invalidation for installed PWAs (bumping it is what makes an already-installed app actually fetch the new `index.html`).

**Both must be bumped together on every push that touches `index.html` or `sw.js`**, even for small fixes — they are two independent constants in two different files with no automated sync, and have drifted out of sync before.

## Architecture

Everything lives in the single `<script>` block of `index.html`, organized under `// ── SECTION ──` comment banners. The pattern is consistent throughout:

- **State**: a single `state` object (checklist, guests, budget, venues, vendors, settings, plus UI-only fields like `tab`, `guestFilter`, `addGuestOpen`) is the source of truth. Each domain array is loaded from and saved to its own `localStorage` key (see `STORE` and `load`/`save`). `saveAll()` persists every domain at once and is called after almost every mutation.
- **Render loop**: `render()` picks the current tab (`state.tab`) and calls the matching `renderX()` function (`renderChecklist`, `renderGuests`, `renderFinance`, `renderVenues`, `renderSettings`), each of which returns an HTML string built via template literals. `render()` sets `#content`'s `innerHTML` to that string wholesale, then calls `bindEvents()` to (re)attach every event listener via `data-*` attribute selectors (`[data-toggle]`, `[data-edit-task]`, `[data-expand]`, etc.). There is no diffing/virtual DOM — every state change does a full re-render of the current tab's HTML and a full re-bind of events. When adding a new interactive element, give it a `data-*` hook and add its listener in `bindEvents()`.
- **Header** (`renderHeader()`) and **tabs** (`initTabs()`) are outside the `render()`/`bindEvents()` cycle — the header updates a few DOM nodes directly (title, summary pills, decorative flowers) and tab buttons are bound once at init, not re-bound per render.
- **Expand/collapse pattern**: list items (guest cards, budget rows, vendor rows) track an `expanded` boolean on the item itself; clicking toggles it and un-expands all siblings in that list before re-rendering, so only one item is open at a time. Guest cards render their expanded detail panel inline immediately after the card in the HTML string (see `expandedPanel()` in `renderGuests()`), and the surrounding `.guest-columns` grid drops from two columns to one (`.has-expanded` class) while anything is expanded so the panel isn't squeezed into a half-width column.
- **Cross-domain sync**: vendors and budget are linked — `syncVendorToBudget()` creates/updates a budget line item whenever a vendor's price changes, and `removeVendorFromBudget()` cleans up the corresponding budget entry when a vendor is deleted. Budget items created this way carry a `vendorId` back-reference.
- **Modals/inline forms**: `showModal(html, onSubmit)` is the shared pattern for popup forms (task edit, venue edit) — it injects an overlay and calls `onSubmit(overlayEl)` on submit, which should return `false` to keep the modal open (e.g. validation failure) or `true` to close it.
- **Duplicate prevention**: `isDuplicate()` / `dupAlert()` are the shared helpers for warning on duplicate names/entries when adding items (guests, tasks, etc.) — reuse them rather than writing new checks.
- **Drag and drop**: guest cards support touch-based reordering between Bride/Groom/Both columns via `initDragDrop()`, called after every render when on the guests tab. It's a custom touch-event implementation (not the HTML5 drag-and-drop API), since the app targets mobile PWA usage.
- **Styling**: all styles are inline in the `<head>` `<style>` block using CSS custom properties defined on `:root` (`--pink`, `--blue`, `--green`, `--bg`, `--card`, `--text`, `--muted`, `--border`, etc.) plus per-element inline `style="..."` attributes generated in the template strings for one-off styling. Reuse the existing custom properties rather than hardcoding new colors.
