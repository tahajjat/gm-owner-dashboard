# Why `index.html` froze the browser — root cause & fix

## Symptom

Opening `index.html` (or reloading it after the first visit) made the tab
become unresponsive / "stop" — scrolling, clicks and the OS "page unresponsive"
warning would appear, effectively hanging the browser.

## Root cause

`index.html` is a single-file prototype that grew by repeatedly pasting new
`<script>` blocks at the bottom of the file over time. Two of those blocks
(originally around **lines 277–285** and **line 292**) implemented a *second,
independent* Bengali/English translator that was layered on top of an
already-working translator defined earlier in the file (lines 265–273).

The second translator did three things together, and the combination is what
caused the freeze:

1. **Forced English on every load.** A trailing script unconditionally ran:

   ```js
   localStorage.setItem('gmLang', 'en');
   ```

   on *every single page load*, regardless of what the user had selected.
   This meant that from the **second time** the file was opened onward,
   `localStorage.gmLang` was always `'en'` when the page started.

2. **A `MutationObserver` that re-triggers itself.** The translator installed:

   ```js
   const observer = new MutationObserver(() => { if (current === 'en') swap('en') });
   observer.observe(document.body, { subtree: true, childList: true });
   ```

   `swap('en')` walks **every leaf element in `<body>`** and rewrites its
   `textContent`. Rewriting `textContent` **is itself a DOM mutation**, so
   every time `swap('en')` ran, it queued a brand new batch of mutation
   records — which immediately re-invoked the very same observer callback.

3. **An expensive callback, run in a loop.** `swap()` rebuilt and **re-sorted
   a ~150-entry dictionary for every single element** it visited
   (`Object.entries(map).sort(...)` was inside the per-element loop). With
   the dashboard's many nav clicks, filters, and re-renders (`render()`,
   `renderOwnerBoard()`, `renderExtended()`, `renderBins()` all rewrite large
   chunks of `innerHTML`), each interaction triggered full-document rescans,
   and the observer's own writes kept re-queuing more rescans.

Because the observer was attached to `document.body` with `subtree: true`
*while the rest of the (very long) HTML document was still being parsed and
inserted* (several more large `<section>` blocks follow it in the file), and
because the app already had a completely separate, older translation system
still wired to the same `#languageToggle` button, the page entered a
self-sustaining cycle of whole-document text rewrites that pegged the main
thread and made the tab unresponsive — exactly the "browser stops" symptom
reported.

This is a textbook case of two anti-patterns compounding each other:

- Mutating the DOM from inside a `MutationObserver` callback without a guard,
  so the observer reacts to its own writes.
- Two independent, un-coordinated copies of the same feature (translation)
  left in the codebase after later edits, instead of the old one being
  removed.

## Why it didn't freeze on the very first load

On a completely empty `localStorage`, `current` started as `'bn'`, so the
observer's `if (current === 'en')` guard was never true and `swap()` was
never called — the bug only manifests once `localStorage.gmLang` had already
been forced to `'en'` by a previous visit (i.e. on reload / second run),
which matches "it stops when I run/open it" (after the first time).

## Fix

The safe, original translator (lines 265–273 in the pre-fix file) already
provides bilingual toggling correctly and only translates **once, on demand**
(button click) — it does not use a `MutationObserver` and is not expensive.

The fix removes the two redundant, buggy blocks entirely:

- The `swap()` / `bnEn` / `enBn` / `MutationObserver` IIFE.
- The trailing IIFE that force-set `localStorage.gmLang = 'en'` and disabled
  the language toggle on every load.

Nothing else in the file references `swap`, `bnEn`, `enBn`,
`window.gmChangeLanguage`, `window.gmChangeTheme`, or `MutationObserver`
(verified with a project-wide search), and the dark-mode toggle continues to
work because the original translator (lines 265–273) already wires up
`#themeToggle` independently. No functionality is lost — the bilingual
toggle now behaves the way it did before the later patches broke it, and the
forced/disabled English-only state is gone.

## Verification performed

- Confirmed (by reading the full file) that no other code depends on the
  removed identifiers (`grep` for `swap(`, `bnEn`, `enBn`,
  `MutationObserver`, `gmChangeLanguage`, `gmChangeTheme`).
- Confirmed `<script>`/`</script>` and `<style>`/`</style>` tag counts stay
  balanced (10/10 each) after the removal.
- Extracted every inline `<script>` block and ran `node --check` on each —
  all 10 blocks are syntactically valid after the edit.
- A backup of the original file was kept at
  `index.html.orig` (scratch copy) before editing, in case you want to diff.

### Manual check you can do

1. Open `index.html` in a browser.
2. Reload the page a few times, and click through the nav tabs, filters, and
   Approve/Cancel actions repeatedly.
3. The page should stay responsive throughout — no freeze, no
   "page unresponsive" prompt.

If you ever want a language toggle again, extend the existing, working
translator at the top of the script section instead of adding a second one.

---

## Follow-up bug: sections after "Data connection health" rendering under the sidebar

### Symptom

After the freeze fix above, the "Data connection health" section (and
everything before it) displayed correctly, but the sections that come after
it in the file — **Biznify Unified ERP · ভবিষ্যৎ একীভূত কন্ট্রোল**,
**Demo Data Coverage**, and **Owner Reports Center** — rendered flush against
the left edge of the page, overlapping/underneath the fixed left sidebar
(`<aside>`), instead of appearing in the main content column.

### Root cause

`<main>` is the element that carries the layout offset for the fixed
sidebar:

```css
aside{position:fixed; width:232px; inset:0 auto 0 0; ...}
main{margin-left:232px; ...}
```

`</main>` closes very early in the document (right after the "Companies"
table), long before most of the dashboard's extra sections exist in the
markup. Every later "bolted-on" section (Owner Command Board, Bin Card,
Group Workspace, Governance, etc.) is written as a raw `<section>` sitting
directly under `<body>`, **after** `</main>` — and each one is expected to
relocate itself into `<main>` with a small script right after its markup,
e.g.:

```js
document.querySelector('main').appendChild(section);
```

This pattern was applied consistently for every extra section **except
three**: `Biznify Unified ERP` (`data-section="biznify-future"`),
`Demo Data Coverage` (`#demoCoverage`), and `Owner Reports Center`
(`#reportsCenter`). Their scripts only added a nav button — they never moved
their section into `<main>`. Left as direct children of `<body>`, these three
sections got no `margin-left`, so they rendered starting at `x = 0`, i.e.
under/behind the fixed sidebar. Since "Data connection health" is the last
section that *does* get relocated (it's the second half of the
`group-workspace` pair), everything **after** it in the file was affected —
matching what was reported.

A second, related bug in the same three scripts: their nav buttons toggled
visibility with `x.style.display = ...` directly, instead of calling the
shared `setView()` function used by every other nav button. This left stray
inline `style.display` values behind that could keep other sections hidden
after visiting one of these three views.

### Fix

For each of the three sections, the button-setup script now also appends the
section into `<main>` before wiring the nav button, and the button reuses the
shared `setView()` navigation function instead of the ad hoc
`style.display` toggling:

```js
// Example (Reports Center) — same pattern applied to Biznify Unified ERP and Demo Data Coverage
const section = document.querySelector('#reportsCenter');
document.querySelector('main').appendChild(section);
const n = document.createElement('button');
n.dataset.view = 'reports';
n.textContent = '▤ Reports Center';
document.querySelector('#nav')?.appendChild(n);
n.onclick = () => setView('reports');
```

This matches exactly how every other extra section (Bin Card, Group
Workspace, Owner Command Board, Governance) already relocates itself, so the
three that were missed now behave the same way — correct left-column
placement, correct "active" nav highlighting, and no leftover inline styles
breaking navigation between views.

### Verification performed

- Confirmed each of the three target elements (`[data-section="biznify-future"]`,
  `#demoCoverage`, `#reportsCenter`) is unique in the document before
  targeting it with `querySelector`.
- Confirmed `<script>`/`</script>` and `<style>`/`</style>` tag counts stay
  balanced (10/10) after the edit.
- Re-extracted all 10 inline `<script>` blocks and ran `node --check` on
  each — all still syntactically valid.

### Manual check you can do

1. Open `index.html`.
2. Click through every nav item, including "Biznify ERP Future",
   "Demo Data Coverage", and "Reports Center" (scroll the sidebar/nav if
   needed — on desktop widths it's a vertical list, on narrow widths it's a
   horizontal scroll strip).
3. Each of those views should render in the same content column as the rest
   of the dashboard, indented past the sidebar the same as, e.g., "Overview".
4. Switch back and forth between one of these three views and a normal one
   (e.g. Overview) a few times — the correct view should always be visible
   with nothing hidden or stuck.
