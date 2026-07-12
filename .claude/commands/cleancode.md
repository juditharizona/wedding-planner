---
description: Scan the whole app for bugs, dead code, and inefficient patterns, then clean them up
argument-hint: [optional focus area, e.g. "guests" or "sw.js"]
---

Do a full cleanup pass over this repo (vanilla JS/HTML/CSS in a single `index.html` plus `sw.js` — no build step, no tests, no linter to lean on). If `$ARGUMENTS` names a focus area (a tab/section like "guests", "finance", "venues", "settings", or a file like "sw.js"), scope the scan to that; otherwise scan everything.

## 1. Scan first, don't edit blind

Read the relevant sections of `index.html` and `sw.js` in full before changing anything — partial reads miss cross-references (a function or CSS class used far from where it's read).

Look for:

**Dead code**
- Functions, variables, and `const` config objects that are never called/referenced anywhere in the file.
- CSS classes/rules defined in `<style>` with no matching element in any of the `renderX()` template strings (search for the class name as a string, since markup is generated, not static).
- HTML `id`s or `data-*` attributes with no matching `getElementById`/`querySelector`/event binding in `bindEvents()`, and vice versa (a selector in `bindEvents()` with no matching markup).
- Leftover files referenced by nothing (check `manifest.json` and `sw.js`'s `FILES` array against what's actually in the repo).

**Correctness bugs**
- Event handlers bound to elements that no longer exist after a markup change.
- State mutations that don't call `saveAll()` before `render()` (silently lost on reload).
- Guest/vendor/budget cross-sync drift — anywhere `syncVendorToBudget`/`removeVendorFromBudget` should be called but isn't.
- Off-by-one or duplicate-entry bugs in the `isDuplicate()` checks.
- `parseInt`/`parseFloat` on `dataset` values without validating the source attribute exists.

**Efficiency**
- Judge this within the app's existing architecture (documented in `CLAUDE.md`): full string-template render + full re-bind on every state change is the deliberate pattern here — don't propose a virtual-DOM/diffing rewrite or a framework. Instead look for waste *inside* that pattern:
  - Repeated `state.guests.filter()`/`.reduce()`/`.find()` passes over the same array that could be combined into one pass.
  - `document.querySelectorAll` run inside a loop instead of once outside it.
  - Recomputing something in `renderHeader()` and again in the tab's `renderX()` that could be computed once and passed/shared.
  - Anything that rebuilds a big HTML string just to throw most of it away (e.g. filtering after building instead of before).

## 2. Fix it

- Dead code and clear efficiency wins: remove/fix directly.
- Anything that changes observable behavior (a real correctness bug): fix it, but call it out explicitly in your summary — don't bury a behavior change inside a "cleanup" commit description.
- Follow the codebase's existing conventions (see `CLAUDE.md`): inline styles via template strings, CSS custom properties for colors, `data-*` hooks + `bindEvents()` for interactivity, single `state` object persisted via `saveAll()`.
- Don't introduce new abstractions, helper files, or build tooling to do this — this app is intentionally a single static file.

## 3. Verify

- For anything touching rendering or event binding, actually run the app (`python3 -m http.server` + a quick Playwright pass, or the `/verify` skill) and click through the affected tab(s) — don't rely on reading the code alone.
- If `index.html` or `sw.js` changed, bump `APP_VERSION`/`APP_UPDATED` in `index.html` and `CACHE` in `sw.js` together, per the project's standing rule.

## 4. Report

Summarize what was dead code (removed), what was a real bug (fixed, with the behavior change called out), and what was an efficiency tweak — don't let these blur together in the final summary.
