# City Creek Operations Hub — project briefing

Single-file offline business-intelligence app for a Utah commercial GC estimator (Jamo, City Creek Construction). Everything — CSS, data, and ~9,000 lines of vanilla JS — lives inside **City_Creek_Estimating_Hub.html**. No server, no build tools, no frameworks. It must stay a single file that opens by double-clicking.

## Non-negotiables
- **One self-contained HTML file.** No external requests, CDNs, or imports. Works fully offline.
- **User data lives in localStorage** under key `cccEstimatingHub_v3`. The file ships seed data; the user's edits overlay it. **Never rename the storage key** — that would orphan months of his real bid data.
- Brand: dark green `#3D5A3D`, tan `#C4B896`, light + dark themes. Taglines "Built With Intention" / "Intentional Excellence".
- The user is not a terminal person. Keep everything double-click simple.

## File anatomy (in order)
1. `<style>` — all CSS, organized by `/* ===== Section ===== */` comments.
2. Second `<script>` holds everything else:
   - `const DATA = {…}` — a single large JSON object (seed data: bids by year, active board, contracted, wip, pmProjects, jobcosts, ~1,150 tradePartners, costs, checklist, goals2026, divNames, notSameSeed…).
   - After it, the app: helpers → edit engine → matcher → views → render.

## Core architecture
- `V = {}` — registry of view functions; `V.someView()` returns an HTML string. `render()` sets `#main.innerHTML` from `V[state.view]`. 29 views across 7 nav groups.
- **Edit engine**: `rows(name)` merges seed rows + user edits + added − deleted. `setF(coll,id,key,val,rerender)` writes an edit. Inputs are built with helpers `tin/tmoney/tnum/tpctv/tdate/osel` that call `setF` on change. `STORE` is the persisted overlay (`saveStore()` writes localStorage).
- **render() preserves scrollTop + focused-element index + caret** when re-rendering the same view; view switches reset to top. Don't break this — it was a hard-won fix.
- **Undo**: `_doDelete` pushes to `STORE.undo` (max 40); Ctrl/Cmd+Z; "Recently Deleted" panel in Backup view. Seed-row deletes only hide (restorable), added-row deletes remove.
- **Job identity matching** (`jobSame`): job-number prefixes (`2610-`, `2527.`), phase tags (Phase 2+, "(2 Buildings)", "PR N" = separate scopes; Phase 1 = base job), token overlap ≥75%, and user overrides (`STORE.notSame` + `DATA.notSameSeed` — pairs marked "different jobs"; these always win). This matcher links bids ↔ contracted ↔ WIP ↔ Projects ↔ closeout.
- **Approved COs/PRs** (collection `changeOrders`) roll into WIP revised contract automatically via `wipApprovedChanges()`; `revisedContractOverride` is the manual escape hatch.
- **Timeline estimator** (`wipEstimate`): projects committed cost / earned revenue / calendar-year revenue from contract + start + duration + % complete. Rows it fills get `estimated:true` and an "est" badge; manual numbers always win.
- The **ticker** (`tickerItems()`) derives everything live — never hardcode numbers or years in it.

## Editing conventions
- Views return template strings; event handlers are inline `onclick="fnName(...)"` with globals on `window`.
- Escape user text with `esc()` (HTML) / `att()` (attributes) — always.
- Money via `money()/moneyK()`, percentages via `pct()`. `setPct` takes "7" for 7%.
- New styling goes in `<style>` with a section comment; respect both themes (`html.dark`).

## Testing (how this app has been verified so far)
Headless Playwright against `file://`: load, iterate all views setting `state.view` + `render()`, assert `#main h1` exists and zero `pageerror`s, light + dark. Interactive checks drive real inputs (note: card project names are `<input>` values — match on `.value`, not innerText; a test calling `.focus()` scrolls, which can look like an app bug but isn't).

## Working with the user
Jamo is a construction estimator, not a developer. He describes outcomes ("PRs should have their own section") and expects working, tested results delivered as the updated single file. He values: honest data (flag estimates as estimates, thin samples as thin), undoability, and nothing that silently overwrites what he typed. When a heuristic gets a job-match wrong he'll tell you — wire it into the override system rather than special-casing names.
