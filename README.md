# dk-application — Sales Performance Dashboard

A single-file sales performance ledger (`index.html`) supporting **Head Sales / Sales Director** (national helicopter view) and **Sales Manager** (area-scoped operational view) workflows. Designed for fast human scanning and deterministic parsing by coding agents, using zero external chart dependencies, `tabular-nums` alignment, and a single centralized state object.

---

## Quick Overview

* **Entrypoint:** `index.html` — self-contained HTML, CSS, and vanilla JavaScript with no build step, package manager, or bundler required.
* **Local Preview:** Open `index.html` directly in any browser or serve the directory statically (e.g., `npx serve .` or `python -m http.server`).
* **Primary Perspectives (POVs):**
  * **Head Sales / Director (`pov = 'director'`):** National coverage across all regions (`ALL`), regional quota vs. actual grouped bar chart, and brand mix.
  * **Sales Manager (`pov = 'manager'`):** Area-scoped view (`Jakarta`, `Bandung`, `Surabaya`, `Yogyakarta`, `Medan`) via quick territory pills or the region selector, switching the primary bar chart to individual salesperson achievement.
* **Core Capabilities:** Reporting period switcher (`MTD`, `Q4`, `YTD`), cascading region/salesperson/brand filters, Top vs. Lowest operational rankings (outlets, brands, salespeople), and a sortable account ledger with search.

---

## Project Structure & State Flow

```text
index.html
├── <style>        # CSS custom properties, hairline borders, tabular numerals, responsive grids
├── <body>         # Header + POV toggle, Area scope bar, Filter bar, KPI strip, Charts, Rankings, Ledger
└── <script>
    ├── Data & Config   # RAW_RECORDS, PERIOD_MULTIPLIERS, BRAND_COLORS
    ├── State           # state = { pov, period, region, salesperson, brand, rankMode, search, sortKey, sortDir }
    ├── Data Pipeline   # getPeriodRecords() -> getFilteredRecords()
    ├── UI Sync         # syncDropdownOptions(), setPOV(pov)
    ├── Renderers       # renderKPIs(), renderBarChart(), renderDonutChart(), renderRankings(), renderLedger()
    └── Orchestrator    # renderAll()
```

### State & Render Contract

Every filter, POV, period, or sort interaction mutates `state` and runs a synchronous UI refresh:

1. `syncDropdownOptions()` validates `state.brand` and rebuilds `filterSalesperson` options from rows matching `state.region` so a salesperson from another territory cannot remain selected.
2. `getPeriodRecords()` applies `PERIOD_MULTIPLIERS[state.period]` (`MTD`, `Q4`, `YTD`) to base `RAW_RECORDS` and computes `attainment = (actual / quota) * 100`.
3. `getFilteredRecords()` filters records by `state.region`, `state.salesperson`, and `state.brand`.
4. `renderKPIs`, `renderBarChart`, `renderDonutChart`, `renderRankings`, and `renderLedger` consume the filtered records and update the DOM.

---

## Key Functions & Data Contracts

| Symbol | Role | Input / Output Contract |
| :--- | :--- | :--- |
| `RAW_RECORDS` | Base `MTD` sales dataset | Array of `SalesRecord` objects in IDR |
| `PERIOD_MULTIPLIERS` | Period scaling factors | Maps `'MTD' \| 'Q4' \| 'YTD'` to `{ quota, actual }` multipliers |
| `getPeriodRecords()` | Computes period totals | Returns records with scaled `quota`, `actual`, and `attainment` percentage |
| `getFilteredRecords()` | Applies active filters | Returns period-scaled records matching `region`, `salesperson`, and `brand` |
| `syncDropdownOptions()` | Keeps selectors valid | Rebuilds `filterBrand` and `filterSalesperson` options when `region` changes |
| `setPOV(pov)` | Switches executive/area mode | Updates `state.pov`, toggles `#territoryBar`, scopes `state.region`, calls `renderAll()` |
| `renderAll()` | Master UI refresh | Runs `syncDropdownOptions` $\rightarrow$ all five section renderers |

---

## Domain Rules & Thresholds

* **Currency & Scale:** Monetary values are stored as integer IDR amounts (`actual`, `quota`) and formatted via `formatIDR()` (`Rp ...B`, `Rp ...M`, or locale IDR) and `formatMillions()`.
* **Attainment Status (`getStatusInfo(attainment)`):**
  * **On Track:** `attainment >= 100` (`badge-good`, green)
  * **At Risk:** `85 <= attainment < 100` (`badge-warn`, amber)
  * **Behind:** `attainment < 85` (`badge-bad`, red)
* **Chart Grouping (`renderBarChart`):** Groups bars by `salesperson` when `state.pov === 'manager'` or `state.region !== 'ALL'`; otherwise groups by `region`.

---

## Data Schema

Each item in `RAW_RECORDS` conforms to the following interface:

```ts
interface SalesRecord {
  id: number;
  region: 'Jakarta' | 'Bandung' | 'Surabaya' | 'Yogyakarta' | 'Medan';
  salesperson: string;
  outlet: string;
  brand: string;
  quota: number;  // Base MTD quota in IDR
  actual: number; // Base MTD actual sales in IDR
}
```

---

## Agent & Maintainer Conventions

* **Preserve Zero-Dependency SVG Charts:** Keep `renderBarChart` and `renderDonutChart` as inline `<svg>` builders rather than introducing external charting libraries.
* **Keep Financial Alignment Consistent:** Apply `.tabular` (`font-variant-numeric: tabular-nums`) to numeric readouts and `.num-col` (`text-align: right`) to numeric table columns.
* **Avoid Stale Filter Combinations:** Always run `syncDropdownOptions()` before filtering when `state.region` changes, and reset `state.salesperson = 'ALL'` on territory switches.
* **Connecting Live Backend Data:** Replace `RAW_RECORDS` and `PERIOD_MULTIPLIERS` with API-sourced data matching `SalesRecord`, then invoke `renderAll()`.
