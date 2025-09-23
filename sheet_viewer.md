# Run Review Grid — Dense Row Design & Interactions (Designer Spec)

A focused, research‑grade screen to triage many cases quickly. Desktop‑only. No extra flair; information density and speed first.

---

## 1) Purpose & Primary Use Cases

* **Bulk scan** an evaluation run’s outputs like a spreadsheet, seeing images, key text, and statuses at a glance.
* **Spot anomalies** (bad metrics, errors, unexpected artifacts) and drill into full details without losing context.
* **Filter by labels/markers** and categories, sort by metrics/status/time, and **keyboard‑triage** case‑by‑case.

---

## 2) Page Model (Single Screen)

**Master–Detail Workspace** with a **dense “Review Grid”** on the left and a **Case Detail** inspector on the right. Detail can be collapsed; grid can show an **inline expanded row** with fuller content (without switching panes).

* Header: Run selector, global search, merged/raw toggle.
* Left Sidebar: facets (Status, Tool, Class/TestClass, Scenario, Labels/Markers, Metrics, Has Artifacts).
* **Center Split**: left = Review Grid; right = Case Detail tabs. Resizable divider. Focus modes: Grid‑only / Detail‑only.

---

## 3) Review Grid — Row Structure (Collapsed)

Goal: fit **2–3 rows** vertically on a 1440× desktop, each row showing rich context.

**Row height target (collapsed):** \~300–360px (adjustable via density toggle).
**Grid columns (visual cells within the row):**

1. **Preview Strip / Artifact Carousel**

   * Size: **500×200** viewport inside the row (fixed height).
   * Thumbnails with left/right chevrons; hover shows image index; click opens lightbox.
   * If CSV artifact present, show a small **CSV chip** (count) and hover tooltip (columns/rows).

2. **Question / Args Block**

   * First line: **Function name** (`function_call.name`) + Tool ID pill.
   * Body: **Primary arg** (e.g., `query` or first text arg) truncated to 2 lines.
   * Metadata line: filenames from args (if any) or `function_call.args.*` keys inline; greyed **history** snippet (if provided in expectations/description).
   * Labels/Markers: chips (Scenario, Class markers, tags); overflow `+N` with tooltip list.

3. **Input Artifact Details (Mini‑Table)**

   * For dataframes: small table **2–3 rows**: name • shape • columns (first 3) • size (if known).
   * For binaries/skipped: list with icon + reason.

4. **Code / Output Summary**

   * **Code snippet**: first \~8–12 lines (monospace).
   * Divider line.
   * **Output summary**: brief `text_summary` or `code_output` first \~4 lines; fallback to `function_response.outputs[0..]` keys.

5. **Eval Status & Metrics**

   * Status pill from `inference.status` (success/error/unknown).
   * Metric list (2–3 most important): name • score • pass/fail/skip pill.
   * **Reason** excerpt for failing metric (1 line, tooltip expands).
   * Timestamp and Worker chip (when Raw).

6. **Row Actions**

   * Buttons: **Expand inline**, **Open in Detail**, **Compare**, **Copy link**.
   * Checkbox for multi‑select (optional; for batch export).

---

## 4) Inline Expanded Row (Partial Details)

When **Expand inline** is clicked, the row grows to \~520–640px height and reveals deeper context **without leaving the grid**:

* **Artifacts panel** widens; shows more thumbnails or a 2×2 grid under the carousel.
* **Code block** shows full cell with scroll (height‑limited area).
* **Logs excerpt** (first N lines with search input).
* **JSON sneak peek**: tree viewer of `function_response` top‑level keys.
* **CTA**: “Open Full Case Details” button remains.

Inline expansion should **not** collapse other expanded rows automatically (user controls expansion), but cap at e.g., 2 expanded at once with a subtle hint if more are attempted.

---

## 5) Case Detail Pane (Full Inspector)

* Mirrors the tab suite: Overview • Inputs • Inference • Artifacts • Metrics • Logs • Code • JSON.
* **Tab persistence** across row changes: if user selects another row in Grid, the same tab stays active.
* Opening from an expanded row scroll‑locks the grid and focuses the detail; Esc returns focus.

---

## 6) Interactions & Keyboard

* Navigation: ↑/↓ moves selection; → focuses Detail; ← returns to Grid.
* `/` focuses search; `[` and `]` cycle Detail tabs.
* `x` toggles inline expand on the selected row.
* Lightbox for images supports arrow keys and zoom.
* Chips: click = add filter; right‑click = Only/Exclude.

---

## 7) Sorting, Filtering, and Grouping

* Sort by: Status, Primary Metric score, Updated time, Tool, Class, Scenario. Multi‑sort allowed.
* Group by: Scenario or Class/TestClass to form **sections** in the grid (sticky section headers showing counts and pass rate).
* Quick filters (chip strip above grid) mirror sidebar.

---

## 8) States & Edge Cases

* **No artifacts**: show placeholder image area; CSV chip may still appear.
* **Only CSVs**: carousel swaps for a CSV preview tile (headers + sample rows).
* **Large code/logs**: truncate with gradient; explicit “Expand inline” to reveal more.
* **Unknown schema version**: subtle banner in row and in detail.
* **Many labels**: wrap to two lines, then `+N` overflow with tooltip.

---

## 9) Visual Specs (for drawing)

* Row container: card with 16–20px padding; 16px gap between cells; 20px radius; subtle shadow.
* Column widths guideline (collapsed row):

  * Carousel 500×200 (fixed block).
  * Args/Question 30–35% min‑width.
  * Input‑artifact mini‑table 20–25% (auto shrink).
  * Code/Output 25–30%.
  * Status/Metrics 220–280px.
  * Actions 120–160px.
* Typography: body 14–15px; mono 13–14px; chips 12–13px.
* Status colors consistent with rest of app.

---

## 10) Data Mapping (from `case.json`)

**Row fields → JSON:**

* **Function name** → `case.function_call.name`
* **Tool ID** → `case.tool_id`
* **Primary arg** → heuristic: in `case.function_call.args`, prefer `query` else first string arg.
* **Filenames** → any arg keys containing `file`/`path`; join names.
* **History snippet** → `case.description` or an `expectations.history` field if present.
* **Labels/Markers** → from `case.expectations.labels` (if available) or inferred by file path.
* **Input artifact details** → from `inference.function_response.artifacts[]` of kind `csv`; display `path`, infer columns by sampling (designer shows placeholder).
* **Image carousel** → `artifacts[]` of kind `image` with `path` + `mimetype`.
* **Code snippet** → `inference.code` (first lines).
* **Output summary** → `inference.text_summary` or `inference.code_output` (truncated).
* **Status pill** → `inference.status`.
* **Metrics** → `metrics[]` (choose top N via fixed order or first in list).
* **Fail reason** → `metrics[].reason` of first failing metric.
* **Worker** → from run context (if Raw).
* **Timestamps** → from run index (designer can show placeholder time).

---

## 11) Compare Entry Points

* **Per‑row**: Compare button.
* **Inline expanded**: delta badge hints if previous run exists for this case.
* Opens Compare screen with baseline = previous run; target = current run; anchors to same section (e.g., Artifacts).

---

## 12) Performance Considerations (design‑visible)

* Virtualized grid (show skeleton rows while loading).
* Lazy load artifact thumbnails; show image ratio boxes to avoid layout shift.
* Inline expanded rows fetch additional chunks on demand.
* Keep 60fps scrolling; avoid heavy shadows/transitions.

---

## 13) Acceptance Criteria

* On a 1440px‑wide desktop, the grid shows **2–3 rich rows** fully without scrolling, each with: carousel, args/labels, input‑artifact mini‑table, code/output snippet, status/metrics, actions.
* Tapping **Expand inline** reveals deeper context but keeps the user in the grid; switching selection preserves the active **Detail** tab.
* Keyboard triage lets a user review 10 consecutive cases in under a minute (image previews, code/output glances, status/metric reads).
* Deep links preserve selected case, inline expansion state (if feasible), current filters, and active detail tab.

---

## 14) What to Draw (Designer Checklist)

1. **Collapsed row** with all cells filled (images+csv chip+labels+code+metrics).
2. **Collapsed row with no images** (CSV preview tile).
3. **Inline expanded row** with enlarged artifacts panel + logs excerpt + JSON peek.
4. **Grid + Detail** split screen; demonstrate tab persistence when moving selection.
5. **Keyboard hints** overlay and quick‑action hover states.
6. **Zero‑state** (no matches) and **unknown schema banner**.
