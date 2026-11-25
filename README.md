dk-application — Sales Performance Dual-POV Ledger
A single-file, zero-dependency sales analytics web application (`index.html`) built for Head Sales / Sales Directors (national helicopter view) and Sales Managers (territory-scoped operational view). It parses raw CSV sales ledgers entirely client-side, applies automated data hygiene, and renders interactive inline `<svg>` visualizations, peer-median benchmarks, and bilingual field briefings.
---
1. Agent Execution Context & Non-Negotiables
> **Read this section before modifying any code.** Every edit made by an AI coding agent must preserve these architectural invariants:
Single-File Self-Containment: All CSS, HTML markup, and vanilla JavaScript live inside `index.html`. Do not introduce npm packages, bundlers, TypeScript build steps, or external chart/CSS CDNs (only Google Fonts `Inter` is linked, with system sans fallbacks).
Client-Side Privacy & Ingestion: CSV parsing, deduplication, canonicalization, and aggregation run 100% in browser memory inside an IIFE (`(() => { 'use strict'; ... })()`). Never send ledger rows to external endpoints.
Mandatory XSS Escaping: Every CSV-derived string (`region`, `branch`, `oname`, `oid`, `sm`, `brand`, `otype`, `TAB.name`) must pass through `esc()` before interpolation into `innerHTML` or SVG `<title>`/`<text>` nodes.
Return-Line Accounting Semantics: Credit notes / returns (`net < 0`, `qty < 0`, or invoice numbers matching `/^(rtr|retur)/i`) are flagged `ret: true`. They reduce net sales (`sum(rows)`) but must not inflate unique invoice counts (`live = rows.filter(r => !r.ret)`).
Centralized Event Delegation: Interactive controls (`[data-pov]`, `[data-area]`, `[data-dim]`, `[data-clear]`, `[data-rankdir]`, `[data-rankcat]`, `[data-outmode]`, `[data-donut]`, `[data-lang]`, `[data-outlet]`, `[data-sort]`, `[data-showall]`) are handled by a single `document.addEventListener('click', ...)` dispatcher. Do not attach per-row listeners inside render loops.
Financial Alignment: Apply `.tabular` (`font-variant-numeric: tabular-nums`) to all numeric KPIs, chart labels, percentages, and table cells, and `.num` (`text-align: right`) to numeric table columns.
---
2. Quick Start & Local Preview
Entrypoint: `index.html` (renamed/deployed from `sales-performance-ledger-v3.html`).
Run Locally: Open `index.html` directly via `file://` or serve statically:
```bash
  npx serve .
  # or
  python3 -m http.server 8080
  ```
Zero-Data Smoke Test: Click "Load sample FMCG ledger" (`#btnSample`) on the initial dropzone screen to hydrate 25 multi-region, multi-brand, multi-month records (including a return line `RTR-1001`).
---
3. Repository & Codebase Topology
```text
index.html
├── <style>                   # Design tokens (:root), hairline cards, responsive 7/5 & 4-col grids
├── <header class="topbar">   # Sticky utility bar: SL badge, dynamic scope breadcrumb, POV toggle, Columns/Load CSV
├── <main id="loaderView">    # Drag-and-drop CSV file picker, sample loader, localStorage reopen, raw paste box
├── <main id="mapperView">    # Interactive column-mapping fallback when date or net sales headers are ambiguous
├── <main id="appView">       # Main analytical workspace:
│   ├── Filter Strip          # Manager territory pills (#managerAreaBar), cascading dropdowns, active filter chips
│   ├── 4-Metric KPI Strip    # Actual Net Sales, Invoices & Velocity, Active Outlets, Dynamic Top Entity
│   ├── Analytical Grid (7/5) # Left: Primary Bar Chart, Mix Donut + Legend, Daily Brush Timeline | Right: Rankings
│   ├── Cross-Tab Matrix      # Region × Brand volume heatmap in Rp millions (Director POV only)
│   ├── Account Ledger        # Searchable, filterable, sortable outlet table benchmarked vs. channel median
│   └── Bottom Utility Row    # Bilingual (ID/EN) Telegram/WhatsApp briefing generator & Data Hygiene audit log
└── <script>                  # Single strict-mode IIFE:
    ├── Constants & Parsers   # FIELDS, MON/MONID, BRAND_COLORS, OTHER_COLOR, parseDate(), num(), parseCSV()
    ├── State Containers      # TAB, M, RAW, S, F, UI
    ├── Ingestion & Hygiene   # loadText() -> ingest() -> autoMap() -> build()
    ├── Query & Aggregation   # pass(), sel(), selPrev(), group(), outletAgg(), median(), getBenchmarkMeta()
    ├── Render Pipeline       # render() -> renderHeader() .. renderDQ()
    └── Event & Brush Binding # Delegated click handler, select/input listeners, pointer-capture SVG brush
```
---
4. State Architecture & Data Contracts
Application state is partitioned into five module-scoped objects inside the IIFE:
```ts
/** Raw parsed CSV table before schema mapping */
interface RawTable {
  name: string;
  head: string[];
  rows: string[][];
}

/** Column index mapping (-1 if unmapped) */
interface ColumnMap {
  date: number;   // Required
  net: number;    // Required
  inv: number;
  region: number;
  branch: number;
  oid: number;
  oname: number;
  otype: number;
  sm: number;
  brand: number;
  qty: number;
}

/** Normalized transaction record produced by build() */
interface SalesRecord {
  t: number;       // UTC epoch day (Math.floor(Date.UTC(y, m-1, d) / 86400000))
  m: string;       // YYYY-MM month key
  inv: string;     // Invoice number
  region: string;  // Canonicalized region name (e.g., 'Jabar' -> 'Jawa Barat')
  branch: string;  // Canonicalized branch/city
  oid: string;     // Outlet ID (falls back to oname)
  oname: string;   // Majority-vote canonical outlet name per oid
  otype: string;   // Channel / outlet type (e.g., 'Modern Trade', 'Wholesaler', 'General Trade')
  sm: string;      // Canonicalized salesperson name
  brand: string;   // Canonicalized brand name
  net: number;     // Net sales in integer IDR (negative for returns)
  ret: boolean;    // True if return/credit note
  line: number;    // 1-based source CSV line number
}

/** Active dataset & metadata container (S) */
interface DatasetState {
  name: string;
  rows: SalesRecord[];
  has: {
    region: boolean;
    branch: boolean;
    brand: boolean;
    sm: boolean;
    type: boolean;
    outlet: boolean;
    inv: boolean;
  };
  q: DataQualityReport;
  tMin: number;
  tMax: number;
  months: string[];      // Sorted unique YYYY-MM keys
  totalOutlets: number;  // Distinct oid count across full dataset
}

/** Active data filter slice (F) */
interface FilterState {
  t0: number;            // Start UTC epoch day
  t1: number;            // End UTC epoch day
  month: string | null;  // Selected YYYY-MM period or null for custom/full range
  region: string | null;
  brand: string | null;
  sm: string | null;
  type: string | null;
}

/** Presentation & interaction state (UI) */
interface UIState {
  pov: 'director' | 'manager';
  donutDim: 'brand' | 'region' | 'type';
  rankDir: 'top' | 'lowest';
  rankCat: 'outlets' | 'brands' | 'salespeople';
  outMode: 'total' | 'peer';
  lang: 'id' | 'en';
  sortCol: 'name' | 'region' | 'type' | 'inv' | 'net' | 'rel';
  sortDir: 'asc' | 'desc';
  showAll: boolean;
  q: string;
  statusFilter: 'ALL' | 'Above Median' | 'Near Median' | 'Below Median';
  dedup: boolean;
}
```
Deterministic Execution Flow
Ingestion (`loadText` $\rightarrow$ `ingest`): Strips UTF-8 BOM, auto-detects delimiter (`,`, `;`, or `\t`) via `detectDelim()`, parses RFC-4180 quoted rows via `parseCSV()`, and runs `autoMap()` against bilingual English/Indonesian header aliases (`FIELDS`). If `date` or `net` is unmapped, opens `#mapperView`.
Hygiene & Normalization (`build`):
Deduplicates exact row matches when `UI.dedup === true`.
Parses `YYYY-MM-DD` and `DD/MM/YYYY` dates (`parseDate`) and Indonesian/international currency strings (`num`).
Canonicalizes casing/spacing and regional shorthands (`REGALIAS`: `jabar`, `jateng`, `jatim`) by frequency majority vote (`canon()`).
Resolves outlet naming drift so every `oid` maps to a single canonical `oname`.
Caches raw CSV in `localStorage` under `'sales-ledger-last'`.
Synchronous Render (`render()`): Calls in exact order:
`renderHeader()` $\rightarrow$ `renderFilterStrip()` $\rightarrow$ `renderKPIs()` $\rightarrow$ `renderPrimaryBarChart()` $\rightarrow$ `renderDonutCard()` $\rightarrow$ `renderStrip()` $\rightarrow$ `renderOperationalRankings()` $\rightarrow$ `renderMatrix()` $\rightarrow$ `renderLedger()` $\rightarrow$ `renderSummary()` $\rightarrow$ `renderDQ()`.
---
5. Key Functions & DOM Event Hooks
Core Function Contracts
Symbol	Purpose	Contract
`switchPOV(pov)`	Toggles `'director'` vs `'manager'`	In `'manager'` mode, hides `"All Regions"` option, auto-selects top region if `F.region` is null, switches donut off `'region'`, and scopes `#filterSalesperson`.
`sel(skip?)`	Primary filtered query	Returns `S.rows` matching `[F.t0, F.t1]` and all active dimensions in `F` except keys listed in `skip`.
`selPrev(skip?)`	Prior-month comparison query	If `F.month` is active and `prevMonth(F.month)` exists in `S.months`, returns prior month rows with identical dimension filters.
`outletAgg(rows)`	Outlet rollups & peer median	Groups `rows` by `oid`, counts non-return invoices (`nInv`), computes channel median per `otype`, and attaches `rel = o.net / med[o.type]`.
`getBenchmarkMeta(relPct)`	Peer benchmark classifier	Maps `rel * 100` to `{ label, badgeClass, barColor, textColor }`.
`makeSummary(L)`	Field briefing generator	Builds a single-paragraph Bahasa Indonesia (`'id'`) or English (`'en'`) summary reflecting `UI.pov` and `F`.
Delegated `data-*` Attribute Registry
Attribute	Target Elements	Behavior on Click
`data-pov="director|manager"`	Header segmented toggle	Calls `switchPOV(pov)`.
`data-area="<Region>"`	`#territoryPills` buttons	Sets `F.region`, clears `F.sm`, rebuilds salesperson dropdown, re-renders.
`data-dim="<dim>" data-val="<val>"`	Bar chart groups, donut slices/rows, matrix headers, ranking rows	Toggles `F[dim]` (`null` if already selected) and re-renders. Blocked on `dim === 'region'` in Manager POV.
`data-clear="<key>"`	`#activeChips` filter tags	Clears `month`/`range` or `F[key]` and re-renders.
`data-rankdir="top|lowest"`	Operational Rankings header	Sets `UI.rankDir` and calls `renderOperationalRankings()`.
`data-rankcat="outlets|brands|salespeople"`	Operational Rankings tabs	Sets `UI.rankCat` and calls `renderOperationalRankings()`.

`data-outmode="total|peer"`	Outlet ranking basis pills	Toggles net sales vs. same-type median ranking and re-renders rankings.
`data-donut="brand|type|region"`	Donut card dimension pills	Sets `UI.donutDim` and calls `renderDonutCard()`.
`data-outlet="<oid>"`	Outlet rows in Operational Rankings	Populates `#q` search box with `oid`, filters ledger, and smooth-scrolls to `#ledgerWrap`.
`data-sort="<col>"`	`#ledgerWrap` `<th>` headers	Toggles `UI.sortCol` / `UI.sortDir` and calls `renderLedger()`.
`data-lang="id|en"`	Briefing card language toggle	Sets `UI.lang` and calls `renderSummary()`.
---
6. Domain Rules & Visual Semantics
Currency Formatting:
`rpCompact(n)` $\rightarrow$ `Rp 1.45B`, `Rp 245M`, `Rp 85K` (used in KPIs, chart axes, legend totals).
`rpExact(n)` $\rightarrow$ `Rp 245,000,000` (used in hover tooltips).
`rpID(n)` $\rightarrow$ `Rp 1,45 miliar`, `Rp 245,0 juta` (used in Bahasa Indonesia Telegram/WhatsApp summaries).
Peer Median Benchmarks (`getBenchmarkMeta`):
Above Median: `relPct >= 100` $\rightarrow$ Emerald badge (`#047857`), Teal bar (`#0F766E`).
Near Median: `75 <= relPct < 100` $\rightarrow$ Amber badge (`#B45309`), Amber bar (`#D97706`).
Below Median: `relPct < 75` $\rightarrow$ Red badge (`#B91C1C`), Red bar (`#B91C1C`).
Donut Highlight Synchronization (`renderDonutCard`):
Top 5 categories use `BRAND_COLORS` (`#0F766E`, `#2563EB`, `#D97706`, `#4F46E5`, `#059669`); remaining categories aggregate into `'Others'` rendered in `OTHER_COLOR` (`#64748B`).
When a dimension value `F[dim]` is active, all non-selected slices (including `'Others'`) dim to `opacity="0.25"` and their corresponding legend rows dim to `opacity: 0.45`.
Interactive Timeline Brush (`#strip`):
Pointer drag (`pointerdown` $\rightarrow$ `pointermove` $\rightarrow$ `pointerup` with `setPointerCapture`) sets `F.month = null` and updates `[F.t0, F.t1]` via `requestAnimationFrame`. Single click without drag resets `[F.t0, F.t1]` to `[S.tMin, S.tMax]`.
---
7. Agent Playbooks & Verification Checklist
Common Extension Recipes
Adding a New Filterable CSV Dimension (e.g., `sku_category`):
Add `{ k: 'cat', label: 'Category', al: ['category', 'kategori'] }` to `FIELDS`.
Normalize it in `build()` (`canon('cat')`) and add `cat: ix.cat >= 0` to `S.has`.
Add `cat: null` to `F`, add a check in `pass()`, add `<select id="filterCat">` in `#appView`, and wire `populateFilterControls()` + `renderFilterStrip()`.
Connecting a Live API Endpoint:
Fetch CSV text or JSON records from your API and either pass CSV text directly to `loadText(csvString, 'api-sync.csv')` or populate `TAB` + `M` and call `build(); showApp();`.
Post-Edit Agent Verification Checklist
Before concluding any task that modifies `index.html`, verify:
Zero Console Errors: Open `index.html` in a headless browser / preview, click `#btnSample`, and assert `console.error` count is `0`.
POV Switching: Toggle `#btnRoleManager` and `#btnRoleDirector`; confirm `#managerAreaBar` appears in Manager mode, `#matrixSection` hides in Manager mode, and `#filterSalesperson` only lists reps from the active `F.region`.
Chart & Legend Sync: Click a slice or row in `#donutCard`; confirm non-selected arcs (including `Others`) dim to `0.25` opacity and clicking the active chip in `#activeChips` restores full view.
Deduplication & Returns: Toggle `#dedup` in the Data Hygiene panel and confirm `S.rows.length` and `#dq` audit notes update deterministically without breaking invoice counts.
