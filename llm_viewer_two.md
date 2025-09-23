# Designer Brief — Desktop Prototype (Figma/Miro)

**Purpose:** Translate the engineering‑oriented brief into a visual + interaction spec for high‑fidelity desktop prototypes.

> **App model:** A **single‑page application** (SPA) with **multiple screens/views** (routeable). Everything happens in one browser tab; navigation swaps the main content pane. Desktop‑only for v1.

---

## 0) Screen Inventory (Sitemap)

1. **Run Dashboard** — high‑level overview/KPIs after selecting a run.
2. **Case Browser** — filterable, sortable list/grid of cases with labels/markers.
3. **Case Detail** — deep dive into a single case via tabs.
4. **Compare** — side‑by‑side across runs for the same case.
5. **Global Search** (overlay) — jump to run/case/class quickly.

All screens share a persistent **Header** (Run selector + search) and optional **Left Sidebar** (filters/facets), except Compare which can use a wider canvas.

---

## 1) Global Layout (Desktop)

* **Header (64px)**: left = app name/logo; center = Run selector; right = Global search \[⌘/Ctrl+K], Help, Theme.
* **Left Sidebar (280–320px)**: collapsible. Shows facets (Status, Tool, Class, Scenario, Labels/Markers, Metrics). Sticky footer with “Reset filters”.
* **Main Content**: 12‑column grid; comfortable density.
* **Right Drawer (optional, 360px)**: JSON quick view or notes; slides over Main Content.

Breakpoints (desktop only):

* **1440+**: 1200–1280px content width.
* **≥1200**: 1080–1160px.

---

## 2) Visual Language

* Clean, data‑forward UI. Neutral base; use color **only** for status/metric emphasis.
* Cards with soft shadows, 16–20px radius, 16–24px padding.
* High‑contrast mode variant.
* Typography scale: H1 28–32, H2 22–24, body 14–16, mono 13–14.

Status colors (consistent set):

* **success/pass** (green), **fail/error** (red), **skip** (grey), **unknown** (amber), **running** (blue, future).

---

## 3) Labels & Markers (Categories)

**Goal:** Make category metadata 1st‑class: visible, filterable, sortable.

* **Chips**: small, rounded; types: *Scenario*, *Class marker*, *Argumentative tag*, *Tool type*. Icons by type (e.g., 🏷, 🧪, 🧩).
* **Chip interactions**: hover tooltip → definition; click → add filter; ⨯ removes.
* **Sorting**: by label count, alphabetical, predefined priority.
* **Stats surfaces**:

  * On **Run Dashboard**: breakdown bars (100% stacked) of status by Scenario/Tool/Class.
  * On **Case Browser**: column for primary Scenario; group by tag via toolbar.
  * On **Case Detail**: small donut showing this case’s labels; click opens related cases panel.

---

## 4) Screen Specs

### 4.1 Run Dashboard

**Header strip** with Run selector, “Merged vs Raw” toggle, timestamp.

**KPI Row** (4–6 cards):

* Total Cases • Pass Rate • Fails • Skips • Unknown • #Tools • #Classes

**Breakdown Section** (tabs):

* **By Scenario** — bar chart + table (status counts per scenario).
* **By Tool** — small multiples.
* **By Class** — heatmap (Class × Status).
* Each chart has legends, hover details, and “click to filter” interaction.

**Recent Activity**: list of newest failing/unknown cases with badges and labels.

**Saved Views** (optional): quick filter presets.

**Empty States**: “No merged index found → pick raw run”.

---

### 4.2 Case Browser

**Toolbar**: search input (ID, tool, label, class), view switch (Table ↔ Cards), density toggle (Comfy/Compact), column picker, export (CSV/JSON of list view), clear filters.

**Table View** (default): virtualized data grid.
Columns:

* Status (pill)
* Case ID (link)
* Tool ID
* Class / TestClass
* Primary Scenario (chip)
* Labels/Markers (multi‑chip; 1 line, overflow +N)
* Top Metric(s) (name + score)
* Worker (if raw run)
* Last Updated

**Row hover**: quick actions → Open, Compare, Copy link.

**Card View**: 3–4 columns grid; each card has status ribbon, Case ID, Tool/Scenario chips, 1–2 key metrics, and a small artifact thumbnail if present.

**Facet Panel** (Sidebar):

* Status, Tool, Class/TestClass, Scenario, Labels/Markers, Metrics, Has Artifacts (yes/no).

**Bulk actions** (optional): export selected, add temporary label, open compare with selection (max 2).

**Empty/Zero results**: friendly hint to clear filters.

---

### 4.3 Case Detail

**Header**: Case ID + status pill; Tool ID; Scenario + labels; Run + Worker; buttons: Compare, Copy link, Open JSON.

**Tabs** (sticky):

* **Overview** — snapshot of expectations, metrics summary, artifact count; action chips.
* **Inputs** — function call (name/args) and any input artifacts/metadata (tables with inline preview). Expand/collapse long args.
* **Inference** — function\_response (key fields), text summary, code output (truncated with “Expand”).
* **Artifacts** — image gallery (lightbox) and CSV viewer (table with paging, column controls, download). Show “Skipped binaries” list.
* **Metrics** — list/grid with statuses and scores; inline badges; filter to failing metrics.
* **Logs** — searchable log viewer; toolbar: Find, Wrap, Copy, Jump to error.
* **Code** — read‑only code viewer (split: code and output). Copy/download.
* **JSON** — two‑pane: JSON tree of case.json; right panel schema reference with tooltips.

**Right Drawer** (optional): quick JSON peek or notes.

---

### 4.4 Compare (Side‑by‑Side)

**Selector Bar**: Baseline run vs Target run; case picker (auto‑filled when entering from Case Detail).

**Layout**: Two synchronized columns; sticky header with status/score deltas.

**Sections with diffs**:

* Status + top metrics (delta badges, up/down arrows).
* Function response (field‑level highlights).
* Artifacts (added/removed/changed counts; thumbnail diff list).
* Metrics table (rows: metric, baseline score/status → target score/status).

**Controls**: Next/Prev case in filtered set; “Export diff JSON”; deep link to each side’s case.

---

### 4.5 Global Search (Overlay)

* Open with ⌘/Ctrl+K.
* Search across: runs, case IDs, tool IDs, classes, labels/markers.
* Results grouped by type; arrow keys to navigate; Enter to jump.

---

## 5) Interactions & Micro‑UX

* **Chips**: click to filter; right‑click → “Only this”, “Exclude this”.
* **Sticky elements**: Facet panel header; Case tabs; Compare selector.
* **Keyboard**: / to focus search, j/k to move selection in tables, Enter to open case, c to compare, g r to change run.
* **Loading**: skeletons for cards/tables; optimistic filter feedback.
* **Toasts**: copy link, export success.

---

## 6) States to Design

* Empty run (no cases).
* No artifacts vs many artifacts (grid overflows, lazy thumbnails).
* Oversized logs/code (truncate with expand; search in‑place).
* Unknown schema version banner.
* Missing case in target run (Compare shows placeholder panel).

---

## 7) Data Visualization Patterns

* **100% stacked bars** for status by category (Scenario/Tool/Class).
* **Heatmap** for Class × Status matrix.
* **Sparklines** for metric trends across runs (if time is available).
* **Delta badges** (▲▼ 0.12) with color by improvement/regression.

---

## 8) Components (for design kit)

* Header bar
* Run Selector (combobox)
* Facet/Filter chips & panel
* KPI cards
* Data grid (table) with sortable headers, column chooser
* Cards for Case
* Tabs (with counts)
* Lightbox gallery
* CSV table viewer
* Metric badge & list
* Log viewer (monospace, search)
* Code viewer (read‑only)
* JSON tree viewer
* Diff blocks (field‑level and table‑level)
* Toasts, skeletons, empty states

---

## 9) Copy & Terminology

* Use consistent nouns: **Run**, **Case**, **Class**, **Scenario**, **Label/Marker**, **Artifact**, **Metric**.
* Status dictionary: *success/error/unknown* (inference) vs *pass/fail/skip* (metrics).

---

## 10) Deliverables for Prototype

1. **Run Dashboard** with KPI row + one breakdown chart (By Scenario) and a Recent Fails list.
2. **Case Browser** table with filters and card toggle.
3. **Case Detail** with Overview, Inputs, Artifacts, Metrics, Logs tabs.
4. **Compare** screen with side‑by‑side layout showing deltas.
5. **Global Search** overlay.

Include at least one flow storyboard:

* from Dashboard → Browser (filter by Scenario) → Case Detail → Compare → back link.

---

## 11) Notes on Desktop Focus

* Optimize for wide screens; maintain readable line lengths.
* Dense data views (compact mode) should still hit 44px touch targets where clickable.
* No mobile or tablet variations in v1.

---

### Answer to “Single page or multiple?”

Use a **single‑page app with multiple screens** (routeable views). This keeps context, enables fast transitions (e.g., preserving filters), and supports deep‑linking to any run/case/tab.
