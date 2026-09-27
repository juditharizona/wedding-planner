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

Everything lives in the single `<script>` block of `index.html`, organized under `// ── SECTION ──` comment banners.

- **State**: `state` holds only saved data (checklist, guests, budget, venues, vendors, settings); each is loaded from/saved to its own `localStorage` key via `STORE`/`load`/`save`, and `saveAll()` persists everything. UI-only state lives separately in `ui` and is never saved: `tab`, `open` (the one open card, as `'kind:id'`), `editing` (open card is in edit mode), `adding` (kind whose add form is showing), `addSection`, `guestFilter`.
- **Migration**: `migrate()` runs at startup and upgrades older saved data (vendor `price` + linked budget lines with `vendorId` → `vendor.estimated`/`vendor.actual`; stray `expanded` flags removed; range estimates like "1,500–2,500" → first number). Keep it idempotent and extend it whenever the data shape changes, so users never lose data.
- **Render loop**: `render()` calls the current tab's `renderX()` (template-literal HTML strings) and replaces `#content`'s innerHTML. Events are **delegated once** at init (`onClick`, `onKeyDown` on `#content`) using `data-act` attributes — do not re-bind listeners per render. Tabs are bound once in `initTabs()`.
- **One card pattern for every list**: `itemCard({kind, id, head, view, form, flat, directEdit})` renders tasks, guests, vendors, budget items and venues. Tap the head (`data-act="toggle"`) to open/close; open shows read-only `view` + Edit/Delete; Edit shows `form` + Save/Cancel. Only one card is open at a time. Tasks use `directEdit` (opening = editing). Add forms are the same card with id `'new'` (Save button reads "Add").
- **Forms**: build with `field()`, `selectField()`, `chips()` (chips update the DOM locally, no re-render, so typed text isn't lost). `readForm()` collects every `[data-f]`; `saveItem(kind, id, root)` validates (required fields, `isDuplicate()`, numeric amounts) and commits. Enter in a single-line input saves.
- **Safety**: deleting always goes through `confirmDelete()` (a styled bottom sheet — second approval). While a card is being edited or an add form is open, opening another card, ticking tasks, or switching tabs is blocked with a toast (`busyWith`/`blockIfBusy`) so unsaved changes can't be lost. Use `toast()` for feedback.
- **Finance**: vendors carry `estimated`/`actual` directly; manual costs live in `state.budget`. `financeItems()` merges both into one shape for totals, the donut chart and the "All items" list (sorted: confirmed costs high→low, then estimates high→low). `diffInfo()` gives the ▲/▼/✓ over/under-estimate marker.
- **Drag and drop**: `initDragDrop()` (called after each guests render) adds long-press touch dragging of guest cards between the side lists; it sets `suppressClick` so the drop doesn't also open the card.
- **Styling**: CSS custom properties on `:root`; shared classes `.item`, `.item-head`, `.item-body`, `.details`, `.form`, `.btn-*`, `.chip`. Reuse them rather than adding one-off inline styles or new colors.
