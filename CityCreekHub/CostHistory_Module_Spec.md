# Cost History & Budget Builder — Module Spec (v2)
**Target:** City Creek Operations Hub (`City_Creek_Estimating_Hub.html`, 1.15 MB build)
**Purpose:** Turn CCC's existing 119-bid history into a normalized cost model that answers conceptual-stage budget questions for CMGC and design-build pursuits.

> **v2 supersedes v1.** v1 was written before inspecting the app and told you to build a sector taxonomy and import the bid spreadsheets. **Both already exist.** Corrected below.

---

## 1. Why this exists

At pursuit and conceptual stage (AACE Class 5 / Class 4), an owner asks "what does this run?" before drawings exist. That answer comes from parametric $/SF and normalized comparables, not a CSI takeoff.

The Hub holds 119 bids. They cannot answer that question today because they lack four fields: **gross SF, a populated project type, a construction midpoint, and a location.**

**Design principle:** the module must make false precision structurally difficult. Output is always a range with a stated accuracy class and `n`.

---

## 2. What already exists — do not rebuild

| Asset | State |
|---|---|
| `DATA.bids` | **119 records** across `2023` (23), `2024` (38), `2025` (46), `2026` (12). Every record has `project`, `date`, `amount`, `outcome`. |
| `DATA.projectTypes` | **31-item controlled vocabulary**, already good: Healthcare/Medical, Office, Retail, Education/School, Industrial/Warehouse, Religious, Tenant Improvement, Credit Union, Tilt-Up, Shell, Prefab Metal Building, etc. **Use this. Do not author a new taxonomy.** |
| `DATA.materialLabor` | 77 unit-cost rows, CSI-coded, with `vendor` and `updated` dates (2025-07 → 2026-05). CCC's own unit cost database. |
| `DATA.jobcosts` | 22 Estimate-vs-Actual records. **0 have actuals populated.** See §7. |
| `DATA.checklist` | 400 scope-gap items |
| `DATA.tradePartners` | 1,166 records |
| `jobSame()` + `notSameSeed` | Job identity matcher with permanent overrides |
| Data Health view | Ranked cleanup items with one-click fixes — the pattern to extend |

The spreadsheet import is **done.** `Bids List and $.xlsx` is already in the app. Do not re-import it.

---

## 3. The actual gap, measured

| Field | Coverage | Note |
|---|---|---|
| `projectType` | **66/119 (55%)** | 53 blank: 2023→8, 2024→14, 2025→27, 2026→4 |
| `sf` | **4/119 (3%)** | All four are 2026 |
| `duration` | 11/119 | Free text (`"6 Months"`) — needs parse to number |
| `gcsBidding` | 10/119 | Bid climate input |
| `constrType` | 11/119 | |
| `city` / location | **0/119** | Field does not exist |
| `ntpDate` / midpoint | **0/119** | Field does not exist |
| `lowBid` on losses | **0/119** | Field does not exist — see §6 |
| `jobcosts.actual` | **0/22** | See §7 |

**The cheapest, highest-leverage fix is the 53 blank `projectType` values.** No research required — Jamo recognizes every job by name. One sitting. Until that's done nothing groups, and every other field is wasted effort.

---

## 4. Schema additions

Extend the **existing bid record**. No parallel collection — `jobSame()` and Data Health already point at one identity model, and a second guarantees drift.

```
// --- Envelope (new) ---
sf                 number|null    // EXISTS, 3% populated. The unlock.
siteAcres          number|null
stories            number|null
workType           enum: 'New' | 'Addition' | 'Renovation' | 'TI' | 'Sitework Only'
constrType         string|null    // EXISTS, 11/119

// --- Location (new) ---
city               string
county             string
locationFactor     number         // default 1.00, see §5.2

// --- Schedule (new) ---
ntpDate            date|null
durationMonths     number|null    // migrate from free-text `duration`: "6 Months" -> 6
midpointDate       date           // COMPUTED, never entered:
                                  //   ntpDate + durationMonths/2
                                  //   fallback: date + 3mo + durationMonths/2, flagged estimated

// --- Market context (new) ---
lowBid             number|null    // capture on LOSSES — see §6
bidderCount        number|null    // migrate from `gcsBidding`

// --- Derived (computed, never stored) ---
costPerSF          = amount / sf
```

Already present and reusable as-is: `project`, `date`, `amount`, `outcome`, `projectType`, `type` (bid type), `monthlyGC`, `feePct`, `contingencyPct`, `avgBids`, `debriefNotes`.

### Completeness gating
A bid enters the comparables pool only with `sf`, `projectType`, `midpointDate`, and `amount`.

Everything else stays in the record and in Win/Loss analytics. Surface gaps in **Data Health**, ranked by recency, using the same exclusion pattern already established for Unknown bidder counts — excluded from averages, never silently averaged as zero.

---

## 5. Normalization engine

```
adjustedCostPerSF = baseCostPerSF × escalation × location × bidClimate × delivery
```

### 5.1 Escalation — to midpoint, never to today
```
escalation = index[target.midpointDate] / index[historical.midpointDate]
```
Escalating to the current date instead of the construction midpoint systematically underprices every project. **Enforce this in code; do not leave it to discipline.**

Index is a user-maintained `{year, quarter, factor, source}` table. Seed 1.00 at current quarter; Jamo populates outward. The `source` string goes into the Basis of Estimate verbatim.

`materialLabor.updated` dates span only 5 distinct values — **not** a usable time series. Do not derive an index from it.

### 5.2 Location
Ratio of target city factor to historical city factor. Default 1.00, displayed explicitly — an unpopulated factor must read `1.00`, never be silently absent. If an RSMeans subscription is active, its published city cost index is the defensible basis to type in (manual entry only — §8).

### 5.3 Bid climate
Derive from `bidderCount` against the rolling median for that project type. Records with null `bidderCount` are **excluded from climate correction**, not assumed average.

### 5.4 Delivery method
User-editable factor table keyed off the existing `type` field, default 1.00. Do not invent values — they come from Jamo's judgment or from observed spread once volume supports it.

### 5.5 Output contract
Never return a scalar:
```
{ low, high, median, n, comparables[], assumptions[], accuracyClass, excludedCount }
```
**If `n < 3`, return comparables only and suppress the range entirely.** Three points is not a distribution.

---

## 6. Keep the losses

Outcome distribution is roughly 20 wins to 76 losses all-time. That is not a weakness in the dataset — **it is the dataset.** Wins tell you what CCC will do work for; losses tell you what the market will do work for, and the market number is the honest one to hand an owner.

Add `lowBid` capture to the Win/Loss Debrief (which already collects spread and competitor intel) and feed it into a "market range" shown alongside the "CCC range."

---

## 7. The actuals gap — flag this to Jamo

`jobcosts` has 22 records and **zero populated actuals.** The view is scaffolded and empty.

This is the highest-value data in the entire system and it is currently 0% captured. A bid amount records what CCC *thought* a job would cost. An actual records what it *did* cost. Cost history built on actuals is something no published database can compete with; cost history built on bid amounts is a record of estimates.

Not a Phase 1 blocker, but it needs an owner and a routine — likely a close-out step with accounting rather than an estimating task.

---

## 8. Hard constraints

- **Never fabricate** a unit cost, $/SF, escalation rate, location factor, or markup. Missing values render as labeled placeholders that cannot be mistaken for data.
- **No RSMeans data in the Hub database.** The Gordian license permits downloading insubstantial portions into a spreadsheet for a specific project estimate and explicitly prohibits use in an archival or searchable database. Typing one location factor with its source cited is fine; importing line items is not.
- **Single-record identity.** Cost history extends the bid record.
- Preserve existing conventions: vanilla JS, localStorage (`cccEstimatingHub_v3`), no external dependencies, all mutations through `saveStore()`, scroll/focus preservation across re-renders, dark-mode CSS variables defined, undo via `pushUndo()`.
- New view registers in the `NAV` array under **Estimating** or **Reference**, with a badge count following existing patterns.

---

## 9. Phasing

| Phase | Build | Unlocks |
|---|---|---|
| **0** | Data Health queue for the 53 blank `projectType` values, one-click assign from the existing 31-item list | Everything downstream. Half a day of Jamo's time. |
| **1** | Add envelope + location + schedule fields; parse `duration` → `durationMonths`; compute `midpointDate`; derive `costPerSF`; extend bid entry form; Data Health completeness ranking | Benchmarks view with real $/SF ranges |
| **2** | Escalation index table + normalization engine + comparable picker | Numbers quotable forward with a defensible basis |
| **3** | Division-level cost capture (seed from `Cost History\` folders on disk); % of total | Missing-scope sanity flag |
| **4** | Budget Builder + BOE generator | The owner-facing deliverable |

**Phase 0 first, and it is not optional.** Backfilling SF on records that have no project type produces a pile of numbers that cannot be grouped into anything.

Backfill SF selectively — two densest types (Tenant Improvement at 26 records, Healthcare/Medical at 14), most recent first. Do not attempt all 119. That project dies in week two, every time.

---

## 10. Definition of done — Phases 0 & 1

- [ ] Data Health surfaces all 53 untyped bids with inline assign from `projectTypes`
- [ ] Bid entry captures `sf`, `workType`, `city`, `durationMonths` at creation, not after
- [ ] `duration` free text migrated to numeric `durationMonths` without data loss
- [ ] `midpointDate` computes automatically; estimated midpoints visibly flagged as such
- [ ] `lowBid` added to Win/Loss Debrief
- [ ] Benchmarks view renders $/SF by project type with `n` stated on every figure
- [ ] Any type with `n < 3` shows comparables only, no range
- [ ] No regression: existing 119 bid records render unchanged in Bid Log and Analytics
