# Per‑Screen Design Briefs (Desktop SPA)

This document breaks down each routeable view for the desktop single‑page application. It focuses on what to draw and how it should behave at a UX/UI level, leaving implementation details out.

---

## Global Shell (applies to all screens)

**Header (64px)**: App name/logo • Run Selector (dropdown with recent) • Global Search trigger (⌘/Ctrl+K) • Help • Theme.
**Left Sidebar (280–320px)**: Facet filters (Status, Tool, Class/TestClass, Scenario, Labels/Markers, Metrics). Collapsible.
**Main**: 12‑column grid; comfy density; cards w/ 16–24px padding; status colors consistent.
**Right Drawer (optional)**: JSON quick view or notes; slide‑over.

Interactions: chips filter on click; right‑click “Only this/Exclude”. Sticky: tabs (on Case Detail), filter header.

---

## 1) Run Dashboard

**Goal**: Give a fast snapshot of the selected run and jump‑off points by category and problem areas.

### Layout

1. **Header strip**: Run Selector (also shows timestamp) • toggle **Merged/Raw** • quick filters chips.
2. **KPI Row** (4–6 cards): Total Cases • Pass Rate • Fails • Skips • Unknown • #Tools • #Classes. Each card shows value + small trend sparkline.
3. **Breakdowns (Tabs)**: `By Scenario` • `By Tool` • `By Class`.

   * **By Scenario**: 100% stacked bar chart (Status per Scenario) + compact table beneath.
   * **By Tool**: small multiples (one per tool) with pass/fail/skip bars.
   * **By Class**: matrix heatmap (Class × Status).
4. **Recent Activity**: list of latest failing/unknown cases with status pill, case id, labels/markers, quick actions (Open • Compare • Copy link).

### Interactions

* Click any chart segment to apply/append a filter.
* Hover tooltips with counts and percentages.
* Card click navigates to Case Browser with pre‑applied filter.
* Empty state with guidance if no merged index.

### States to design

* Dense vs comfy cards.
* Few vs many categories (overflow handling, horizontal scroll for charts).
* All‑green run (celebratory empty state for Recent Activity).

### Acceptance

* From Dashboard, a user can reach filtered Case Browser in ≤1 click.
* KPI values + chart totals align.

---

## 2) Case Browser

**Goal**: Find and triage cases quickly with powerful filter/sort and glanceable labeling.

### Layout

**Toolbar**: search (placeholder: “Search case id, tool, class, label…”) • View switch (Table↔Cards) • Density toggle • Column chooser • Export • Clear filters.
**Sidebar facets**: Status • Tool • Class/TestClass • Scenario • Labels/Markers • Metrics • Has Artifacts.
**Main (Table view)**: Virtualized data grid with columns:

* Status (pill)
* Case ID (link)
* Tool ID
* Class/TestClass
* Primary Scenario (chip)
* Labels/Markers (chips; +N overflow)
* Top Metric(s) (name + score pill)
* Worker (if Raw)
* Last Updated

**Main (Card view)**: 3–4 column grid, each card shows status ribbon, Case ID, Tool, Scenario chip, 2–3 labels/markers, one key metric, optional artifact thumbnail.

### Interactions

* Click chip to filter; right‑click chip → Only/Exclude.
* Sorting on any column; multi‑sort with modifier key.
* Row hover quick actions: Open • Compare • Copy link.
* Keyboard: j/k to move; Enter opens; c compares (if a baseline is chosen).
* Saved Views (optional): preset filters in toolbar.

### States to design

* Zero results (suggest reset).
* Many labels on a row (overflow).
* Narrow window (columns collapse with priority rules).
* Raw vs Merged runs (Worker column presence).

### Acceptance

* Filtering by a Label/Marker updates counts and table instantly.
* Switching views preserves filters and scroll position.

---

## 3) Case Detail

**Goal**: Complete inspection of a single case, with fast jumps between data types and a clear status narrative.

### Header

* Case ID (prominent) + status pill.
* Tool ID.
* Scenario & Labels/Markers (chips).
* Run + Worker.
* Actions: **Compare**, **Copy link**, **Open JSON**.

### Tabs & Sections

1. **Overview**

   * Expectation snapshot block (short list).
   * Metrics summary table (metric • status pill • score).
   * Artifact count (e.g., 3 images, 1 CSV) with jump links.

2. **Inputs**

   * Function Call card: name + args (collapsible, mono style).
   * Input artifacts/metadata table if present (file, type, size, path).
   * Optional notes area.

3. **Inference**

   * Status pill from inference.
   * Function response essentials (tool\_id, query, outputs preview).
   * Text summary (if any).
   * Code output (mono block, truncated with “Expand”).

4. **Artifacts**

   * Image grid with lightbox (zoom, pan, download).
   * CSV viewer: paged table, column controls, download.
   * “Skipped binaries” list (reason).

5. **Metrics**

   * Filterable list: status filter (All/Fail/Pass/Skip).
   * Each item: name • status pill • score • reason excerpt • jump links to related tab.

6. **Logs**

   * Searchable log viewer with toolbar (Find, Wrap, Copy, Jump to error).
   * Large-file handling: pagination or virtual scroll indicators.

7. **Code**

   * Read‑only viewer for `code` and `code_output` (split toggled).
   * Copy/download buttons.

8. **JSON**

   * Left: case.json tree; Right: schema reference with tooltips.
   * JSONPath search + “Copy path”.

### Interactions

* Sticky tabs; anchor links within page.
* Chips filter to related cases (opens right drawer listing related).
* Keyboard: \[ to previous tab, ] to next tab.

### States to design

* Case with no artifacts.
* Unknown schema version (banner).
* Exception logs vs empty logs.
* Very long args or outputs (collapse with progressive disclosure).

### Acceptance

* From a failing metric, user can reach the relevant section in ≤2 clicks.
* Lightbox provides zoom and file name; CSV viewer supports quick download.

---

## 4) Compare (Side‑by‑Side)

**Goal**: Understand changes across two runs for the same case, quickly spotting regressions or improvements.

### Layout

* **Selector Bar** (sticky): Baseline Run • Target Run • Case selector (or prefilled).
* **Two columns**: left = Baseline, right = Target.
* **Delta header**: status delta + top metrics deltas (▲▼ with numeric change).

### Sections

* **Status & Metrics**: table with metric rows: baseline → target; color‑coded deltas.
* **Function Response**: key‑field diff (highlight changed/added/removed).
* **Artifacts**: counts of added/removed/changed; thumbnail list with badges; click opens dual lightbox (left/right).
* **Notes** (optional) to annotate findings.

### Interactions

* Arrow keys to switch to next/prev case in current filtered set.
* “Export diff JSON”; deep links to each side’s Case Detail.

### States to design

* Missing case on one side (placeholder with message).
* Many artifacts (paginate thumbnails).
* All metrics unchanged (flat state with confirmation hint).

### Acceptance

* User can identify at least one regression/improvement in under 10 seconds on typical data.
* Exported diff reflects on‑screen highlights.

---

## 5) Global Search (Overlay)

**Goal**: Jump anywhere fast without losing context.

### Layout

* Centered modal overlay; large input; results list with sections: Runs • Cases • Classes • Tools • Labels/Markers.
* Each result: icon by type, primary text (id/name), secondary (run/worker or tags).
* Recent searches below.

### Interactions

* Open with ⌘/Ctrl+K; Esc closes.
* Arrow keys to navigate; Enter to open; modifiers to open in new tab (if app supports multiple tabs).
* Matching highlights show across id, labels, and descriptions.

### States to design

* No results (suggest search syntax).
* Slow search (skeletons).
* Keyboard‑only path.

### Acceptance

* From any screen, user can open overlay, type a case id, and navigate to Case Detail in ≤3 seconds.

---

## 6) Label/Marker System (Cross‑cutting)

Design a consistent chip language:

* **Types**: Scenario, Class marker, Argumentative tag, Tool type.
* **Shape/Color**: neutral outline by default; accent fill on selection; tiny icon to distinguish types.
* **Tooltips**: show definition/example.
* **Stats hooks**: in Dashboard charts; in Browser grouping; in Case Detail donuts.

---

## 7) Prototype Checklist

* Dashboard (with By Scenario tab + Recent Fails).
* Browser table (filters, chips, quick actions) + Card view.
* Case Detail (Overview, Inputs, Artifacts, Metrics, Logs tabs).
* Compare (side‑by‑side with deltas and artifact thumbs).
* Global Search overlay.
* Representative states (empty, many labels, no artifacts, missing compare side).

---

## 8) Copy Guide

Short, action‑oriented labels: “Open”, “Compare”, “Copy link”, “Export diff”. Avoid jargon; keep status wording consistent across screens.
