# LLM-Tool Eval Viewer

A compact, end‑to‑end spec to guide prototyping of an interactive React UI that renders on top of the repo’s pytest outputs: `reports/<run>/<TestClass>/case.json` + `/artifacts/*` and (optionally) the merged index at `reports/<run>_merged/index.json`.

---

## 1) Objectives

* **Explore** results per eval case: inputs, code, logs, outputs, metrics, and externalized artifacts (images/CSVs).
* **Compare** across runs: same case across different runs; summarize drift.
* **Filter & triage**: by tool type, scenario/category, status, metric outcomes.
* **Share**: deep-links to runs/cases, export JSON/snippets.

## 2) Target Users & Primary Jobs

* **Eval engineer** (you): debug failing/unknown cases, verify metrics.
* **Tool author**: validate function responses + artifacts vs. expectations.
* **PM/lead**: scan high-level run status, trends.

## 3) Data Sources (from repo structure)

* **Per‑run worker folder**: `reports/<run_worker>/index.json` → quick status cards.
* **Merged runs**: `reports/<run_prefix>_merged/index.json` → aggregate across workers.
* **Per‑case**: `reports/<run_worker>/<TestClass>/case.json` (conforms to `schema/case.schema.json`).
* **Artifacts**: `reports/<run_worker>/<TestClass>/artifacts/*` (images/CSVs) referenced by relative paths inside `case.json > inference.function_response.artifacts[]`.

> The UI should not require write access. All content is read‑only from the disk (or an API that serves the same files).

## 4) Information Architecture

1. **Run Selector** (global)

   * Pick a run: dropdown + recent pins; optional search by prefix `gh_<id>_a<attempt>`.
   * If a merged index exists: toggle **Merged vs Raw workers**.
2. **Run Dashboard** (home for selected run)

   * KPI tiles: total cases, % pass/skip/fail/unknown; # tools; # scenarios.
   * Status bar by TestClass; sparkline per metric.
   * Quick filters (chips) + search.
3. **Case Browser**

   * Virtualized table/cards with: Case ID, Tool ID, TestClass, Status, top metric scores, tags (scenario/category), duration (if available), updated time.
   * Facets: Status, Tool ID, Scenario, Metric name, TestClass.
4. **Case Detail**

   * Header: Case ID, status pill, tool id, scenario tags, run/worker, TestClass, last modified.
   * **Tabs** (see §6 below): Overview • Inputs • Inference • Artifacts • Metrics • Logs • Code • JSON • Compare.
5. **Compare View**

   * Select 2+ runs; pivot on Case ID. Side‑by‑side diff for metrics/inference/artifacts.

## 5) Navigation & Deep Links

* URL pattern:

  * `/runs/:runId` → Run Dashboard
  * `/runs/:runId/cases` → Browser with query params for filters
  * `/runs/:runId/cases/:testClass` → Filter by class
  * `/runs/:runId/cases/:testClass/:caseId` → Case Detail
  * `/compare?caseId=...&runs=runA,runB` → Compare
* Copy‑link buttons in headers; preserves filters & tab.

## 6) Case Detail — Tabs & Content

### 6.1 Overview

* Status + reason (if available in metrics results).
* **Expectation snapshot** (from `case.expectations`).
* Mini cards: `function_call.name`, args preview, `tool_id`, scenario tags.
* Metrics summary table: metric, status, score, link to metric tab.
* Quick links: open JSON, open artifacts folder.

### 6.2 Inputs

* Left column: **Function Call** pretty‑printed (`case.function_call`), collapsible args.
* Right column: **Input Artifacts/Metadata**

  * If `case.expectations.input_artifacts` or `eval_case.artifact_mock` hints exist, list them.
  * Show artifact metadata table if provided (name, mimetype, size, source path).

### 6.3 Inference

* Status pill from `inference.status`.
* **Function Response** (cleaned): `tool_id`, `query`, `outputs[]` preview.
* **Text Summary**: `inference.text_summary` if present.
* **Code Output**: monospaced block; truncation with “expand”.

### 6.4 Artifacts

* Masonry **Gallery** for images; click → lightbox with zoom/download.
* **CSV Viewer** (DataFrame): interactive table (pagination + column pinning + CSV download). Source path shows from `artifacts[].path`.
* “Skipped binaries” listed with reason.

### 6.5 Metrics

* Grid/list of metric results from `_metric_sink` written into `case.json.metrics`.
* Each item: metric name, status (pass/skip/fail), score, reason, link to related sections (inputs/inference).
* Aggregate sparkline if repeated across cases.

### 6.6 Logs

* Collapsible sections for `logs` with search and copy; sticky toolbar: Find, Wrap, Jump to error.

### 6.7 Code

* **Monaco** read‑only editors for `code` + `code_output`; language autodetect; split view; copy/download.

### 6.8 JSON

* Two panes: left **case.json** tree viewer; right **schema** reference from `schema/case.schema.json` with `$ref` hover tips.
* JSONPath search; “Copy JSONPath” button.

### 6.9 Compare

* Pick baseline + target run (or quick “Prev run”).
* Diff for: status, metric scores, function\_response fields, artifact lists (added/removed/changed).
* Timeline chip to hop through runs.

## 7) Global Components (React/TS)

* **RunSelect**: combobox + recent; optional “load folder/URL”.
* **StatusPill**: `success | error | unknown | pass | fail | skip`.
* **FacetBar**: filter chips + reset.
* **VirtualTable**: TanStack Table with sticky headers; server‑ or file‑backed.
* **CodeViewer**: Monaco wrapper; supports `language`, `value`, `diff?: {original, modified}`.
* **JsonTree**: tree + search + copy path.
* **ArtifactGallery**: grid + lightbox; accepts `artifacts: {kind, mimetype, path}[]` and a base path resolver.
* **CsvDataGrid**: lazy‑load CSV → DataFrame view with column ops.
* **MetricBadge** and **MetricList**.
* **EmptyState** and **ErrorBoundary**.

## 8) States & Edge Cases

* **No merged index** → fall back to worker runs.
* **Missing artifacts** → show placeholders + raw `artifact_parts` (if any remain elsewhere).
* **Large JSON/logs** → chunked/virtualized rendering; search index built on web worker.
* **Unknown schema versions** → banner: “Unsupported schema vX; attempting best‑effort render.”
* **Cross‑run diff when case absent** → mark as “missing in run B”.

## 9) Filters & Taxonomy

* Derive categories from:

  * `case.tool_id`
  * inferred **scenario** from file path (e.g., `tests/test_data/<group>/...`) or `case.description` keywords.
  * metric names presence.
* Expose quick filters: **Status**, **Tool**, **Scenario**, **Metric**, **TestClass**.

## 10) Sample Data Mapping (from `case.json`)

* **Header**

  * `case.case_id`, `case.tool_id`, `case.description`
  * `inference.status`
* **Function Call** → `case.function_call.{name,args}`
* **Outputs** → `inference.function_response.outputs[]`
* **Artifacts** → `inference.function_response.artifacts[]`
* **Metrics** → `metrics[]` (each with `metric`, `status`, `score`, `reason`…)

## 11) UX Flows

### Flow A: Triage a failing case

1. Select run → Case Browser (filter Status=error/fail).
2. Open case → Overview → Metrics (see failed metric reason).
3. Jump to Inputs/Inference/Artifacts to inspect; open images/CSV.
4. Copy deep link to share with a comment.

### Flow B: Compare runs for a case

1. From Case Detail → Compare tab.
2. Pick baseline (prev run) and target (current).
3. Review metric deltas + artifact diff; export diff as JSON.

### Flow C: Gallery inspection

1. Artifacts tab → view grid → open lightbox.
2. Toggle “Show CSVs” → switch to DataGrid; download.

## 12) Visual Layout Hints (for Figma/Miro)

* **Global shell**: left sidebar (Run selector, facets), main content (Browser/Detail), right drawer (JSON/Schema quick peek).
* **Case Browser**: card + table hybrid — dense, keyboard navigable; 12‑col grid; 3‑up cards on wide screens.
* **Detail**: Header summary bar; tabs below; content uses cards with rounded‑2xl, soft shadows.

## 13) Tech Notes (for later implementation)

* **Routing**: React Router.
* **State**: TanStack Query for file fetching; Zustand for UI state.
* **UI Kit**: shadcn/ui + Tailwind.
* **Tables**: TanStack Table.
* **Charts**: Recharts (metric trends).
* **Code**: Monaco editor (diff + folding + search).
* **Perf**: Web workers for JSON/log search; react‑virtual for huge tables.
* **File access**: abstract `FileProvider` to support `file://`, static hosting, or small API.

## 14) Accessibility & Quality

* Keyboard‑first nav; logical tab order.
* High‑contrast mode; prefers‑color‑scheme.
* Live region announcements for tab switches.
* All imagery with alt = filename or description.

## 15) Instrumentation (optional)

* Track time‑to‑first‑render, CPU/heap for very large case.json.
* Event: filter changes, tab views, compare runs used.

## 16) Nice‑to‑Haves

* **Saved Views**: persist filter combos.
* **Run labels**: allow user nicknames.
* **Inline diffs** inside table for quick scan (delta badges).

## 17) Open Questions for Designer

* Density scale (compact vs comfy) defaults?
* Do we want a right‑side JSON inspector always present or as a popover?
* Thumbnail strategy for very large artifact sets (>200 images)?

## 18) Example Case Header (data → UI)

* **Case**: `analysis_code_execution/simple/py_eval_001`
* **Tool**: `analysis_code_execution`
* **Status**: `success`
* **Metrics**: `accuracy: pass (1.0)`; `safety: skip`.
* **Artifacts**: `3 images, 1 csv` (lightbox + grid)

---

### Deliverables for Prototype

1. **Run Selector** (header) + **Run Dashboard** with KPIs.
2. **Case Browser** with facets and virtualized list.
3. **Case Detail** with the full tab set (Overview/Inputs/Inference/Artifacts/Metrics/Logs/Code/JSON/Compare).
4. **Compare** flow for a single case across two runs.

> This brief is intentionally implementation‑ready but focused on UX scopes. Engineers can derive the component tree and props; designers can sketch flows and screens quickly.
