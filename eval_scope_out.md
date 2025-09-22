# LLM-Tool Eval Pytest Proposal + Reporting
---

# repo layout

```
repo/
  core/
    types.py
  schema/
    case.schema.json
  utils/
    reporting_capture.py
  tests/
    base_tool_test.py
    conftest.py
    test_analysis_code_execution_simple.py   # example
  scripts/
    merge_report_runs.py
```

---

# core/types.py

```python
# core/types.py
from __future__ import annotations
from typing import Any, Dict, List, Literal, Optional
from pydantic import BaseModel, Field

class FunctionCall(BaseModel):
    name: str
    args: Dict[str, Any] = Field(default_factory=dict)

class EvalCase(BaseModel):
    case_id: str
    description: Optional[str] = None
    tool_id: str
    function_call: FunctionCall
    expectations: Dict[str, Any] = Field(default_factory=dict)
    # Optional test-time helpers (used by mocks)
    artifact_mock: Optional[Dict[str, Any]] = None  # e.g., {"dataframes": [pd.DataFrame,...]}

# inference/function_response are stored as plain dicts in case.json for flexibility
```

---

# schema/case.schema.json (optional, helpful in CI)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Tool Eval Case Report",
  "type": "object",
  "required": ["schema_version", "case", "inference", "metrics"],
  "properties": {
    "schema_version": { "type": "string", "enum": ["1.0"] },
    "case": {
      "type": "object",
      "required": ["case_id","tool_id","function_call"],
      "properties": {
        "case_id": { "type": "string" },
        "description": { "type": "string" },
        "tool_id": { "type": "string" },
        "function_call": {
          "type": "object",
          "required": ["name","args"],
          "properties": {
            "name": { "type": "string" },
            "args": { "type": "object" }
          }
        },
        "expectations": { "type": "object" }
      },
      "additionalProperties": true
    },
    "inference": {
      "type": "object",
      "required": ["status","function_response"],
      "properties": {
        "id": { "type": "string" },
        "status": { "type": "string", "enum": ["success","error","unknown"] },
        "logs": { "type": "string" },
        "code": { "type": "string" },
        "code_output": { "type": "string" },
        "text_summary": { "type": "string" },
        "function_response": {
          "type": "object",
          "properties": {
            "tool_id": { "type": "string" },
            "query": { "type": "string" },
            "outputs": { "type": "array", "items": { "type": "object" } },
            "artifacts": {
              "type": "array",
              "items": {
                "type": "object",
                "required": ["kind"],
                "properties": {
                  "kind": { "type": "string", "enum": ["image","csv","binary"] },
                  "mimetype": { "type": "string" },
                  "path": { "type": "string" },
                  "skipped": { "type": "boolean" }
                },
                "additionalProperties": true
              }
            }
          },
          "additionalProperties": true
        }
      },
      "additionalProperties": true
    },
    "metrics": { "type": "array", "items": { "type": "object" } }
  },
  "additionalProperties": false
}
```

---

# utils/reporting\_capture.py

```python
# utils/reporting_capture.py
from __future__ import annotations
import base64, json, re
from pathlib import Path
from typing import Dict, Any, List, Tuple

IMAGE_MIMES = {"image/png", "image/jpeg", "image/jpg", "image/webp"}
CSV_MIMES = {"text/csv", "application/csv"}

def _sanitize(stem: str) -> str:
    return re.sub(r"[^A-Za-z0-9_.-]+", "_", stem)[:128] or "artifact"

def _decode_to_file(b64: str, out: Path) -> None:
    out.write_bytes(base64.b64decode(b64))

def externalize_artifacts_minimal(
    function_response: Dict[str, Any],
    out_dir: Path,
) -> Tuple[Dict[str, Any], List[str]]:
    """
    Returns (cleaned_function_response, written_paths).
    Writes only images and CSVs; replaces base64 blobs with file refs.
    Leaves outputs/code/text inline.
    """
    written: List[str] = []
    out_dir.mkdir(parents=True, exist_ok=True)

    fr = json.loads(json.dumps(function_response))  # deep copy
    parts = fr.get("artifact_parts")
    if not parts:
        fr.setdefault("artifacts", [])
        fr.pop("artifact_parts", None)
        return fr, written

    filenames = parts.get("filenames") or []
    payloads  = parts.get("bytes") or []
    mimetypes = parts.get("mimetypes") or parts.get("mime_types") or []

    file_refs = []
    for idx, b64 in enumerate(payloads):
        mt = mimetypes[idx] if idx < len(mimetypes) else "application/octet-stream"
        fname = filenames[idx] if idx < len(filenames) else f"artifact_{idx}"
        stem = _sanitize(Path(fname).stem)

        if mt in IMAGE_MIMES:
            ext = ".png" if mt == "image/png" else ".jpg"
            path = out_dir / f"{stem}{ext}"
            _decode_to_file(b64, path)
            file_refs.append({"kind":"image", "mimetype":mt, "path": str(path.relative_to(out_dir.parent))})
            written.append(str(path))
        elif mt in CSV_MIMES:
            path = out_dir / f"{stem}.csv"
            _decode_to_file(b64, path)
            file_refs.append({"kind":"csv", "mimetype":mt, "path": str(path.relative_to(out_dir.parent))})
            written.append(str(path))
        else:
            file_refs.append({"kind":"binary", "mimetype":mt, "skipped": True})

    fr["artifacts"] = file_refs
    fr.pop("artifact_parts", None)
    return fr, written
```

---

# tests/base\_tool\_test.py (patch-only + “evaluate helper”)

```python
# tests/base_tool_test.py
from __future__ import annotations
import asyncio
from typing import Any, Awaitable, Callable, List

import pytest
from pytest import MonkeyPatch
from utils.reporting_capture import externalize_artifacts_minimal

# patch these once for ALL tools that dereference via these modules
PATCH_TARGETS_GLOBAL: List[str] = [
    "genie.utils.load_artifacts",     # <- your central utils entrypoint
    # add more shared utils if needed:
    # "genie.utils.safety.sanitize_code",
]

class BaseToolTestClass:
    """
    - Class-scoped patching of shared utils (or per-class added targets).
    - Runs tool once per class/eval_case (sync or async).
    - Externalizes images/CSVs to files.
    - Provides `evaluate_and_record(...)` helper so test bodies stay tiny.
    """

    # ---- override if you need tool-specific use-site patches in a subclass ----
    def patch_targets(self, function_tool) -> List[str]:
        return PATCH_TARGETS_GLOBAL

    # ---- convert df -> tool part (override if your project has a concrete type) ----
    def df_to_part(self, df, name: str):
        return {"name": name, "df": df}

    # ---- shared mock(s) used by patches ----
    def mock_load_artifacts(self, eval_case, filenames):
        dfs = getattr(getattr(eval_case, "artifact_mock", None), "dataframes", []) or []
        names = filenames or [f"df_{i}.csv" for i in range(len(dfs))]
        return [self.df_to_part(df, nm) for df, nm in zip(dfs, names)]

    # ---- class-scoped monkeypatch infra ----
    def _class_monkeypatch(self, request) -> MonkeyPatch:
        mp = MonkeyPatch()
        request.addfinalizer(mp.undo)
        return mp

    def _apply_patches(self, request, eval_case):
        mp = self._class_monkeypatch(request)
        mocks = {
            "genie.utils.load_artifacts": lambda ctx, fnames: self.mock_load_artifacts(eval_case, fnames),
            # "genie.utils.safety.sanitize_code": lambda code: code,
        }
        for target in self.patch_targets(getattr(request, "function_tool", None)):
            if target in mocks:
                mp.setattr(target, mocks[target], raising=True)

    # ---- core: run tool and capture ----
    @pytest.fixture(scope="class")
    async def tool_inference(self, request, function_tool: Callable[..., Awaitable[dict]] | Callable[..., dict], eval_case, class_report_dir):
        # 1) patch utils once per class/eval_case
        self._apply_patches(request, eval_case)

        # 2) run tool (await if coroutine)
        out = function_tool(**eval_case.function_call.args)
        raw = await out if asyncio.iscoroutine(out) else out

        # 3) externalize artifacts
        fr_dict = (raw.get("function_response") or {})
        fr_clean, _ = externalize_artifacts_minimal(fr_dict, class_report_dir / "artifacts")

        inf = {
            "id": raw.get("id"),
            "status": raw.get("status", "unknown"),
            "logs": raw.get("logs"),
            "code": raw.get("code"),
            "code_output": raw.get("code_output"),
            "text_summary": raw.get("text_summary"),
            "function_response": fr_clean,
        }

        # stash for final writer
        setattr(request.cls, "_inference_result", inf)
        setattr(request.cls, "_eval_case", eval_case)
        setattr(request.cls, "_report_dir", class_report_dir)
        return inf

    # ---- tiny evaluation helper to standardize test bodies ----
    async def evaluate_and_record(self, metric, tool_inference, eval_case, sink: list):
        res = await metric.evaluate(tool_inference, eval_case) if hasattr(metric, "evaluate_async") else metric.evaluate(tool_inference, eval_case)
        sink.append(res.dict() if hasattr(res, "dict") else dict(res))
        # default assertion policy
        assert (res.get("status") if isinstance(res, dict) else res.status) in ("pass", "skip"), (res.get("reason") if isinstance(res, dict) else getattr(res, "reason", ""))
```

---

# tests/conftest.py (report dirs, eval case loader, sink + per-run index)

```python
# tests/conftest.py
from __future__ import annotations
import os, time, json
from pathlib import Path
import pytest

from core.types import EvalCase

def _compute_run_root() -> Path:
    gh_run = os.getenv("GITHUB_RUN_ID")
    gh_attempt = os.getenv("GITHUB_RUN_ATTEMPT", "1")
    worker = os.getenv("PYTEST_XDIST_WORKER")
    run = f"gh_{gh_run}_a{gh_attempt}" if gh_run else time.strftime("%Y-%m-%d_%H%MZ", time.gmtime())
    if worker:
        run = f"{run}_{worker}"
    root = Path("reports") / run
    root.mkdir(parents=True, exist_ok=True)
    return root

@pytest.fixture(scope="session")
def reports_root() -> Path:
    return _compute_run_root()

@pytest.fixture(scope="class")
def class_report_dir(request, reports_root: Path) -> Path:
    d = reports_root / request.node.name
    (d / "artifacts").mkdir(parents=True, exist_ok=True)
    return d

def get_eval_cases(path: str) -> list[EvalCase]:
    p = Path(path)
    if p.is_dir():
        cases = []
        for f in sorted(p.glob("*.json")):
            cases += json.loads(f.read_text())
    else:
        cases = json.loads(p.read_text())
    return [EvalCase(**c) for c in cases]

# ---- per-class sink & final writer: writes case.json ----
@pytest.fixture(scope="class")
def _metric_sink(request, class_report_dir: Path):
    buf: list[dict] = []

    def finalize():
        eval_case = getattr(request.cls, "_eval_case", None)
        inf = getattr(request.cls, "_inference_result", None)
        if not eval_case:
            return
        case_doc = {
            "schema_version": "1.0",
            "case": {
                "case_id": eval_case.case_id,
                "description": eval_case.description,
                "tool_id": eval_case.tool_id,
                "function_call": eval_case.function_call.dict(),
                "expectations": eval_case.expectations,
            },
            "inference": inf or {"status": "unknown"},
            "metrics": buf,
        }
        (class_report_dir / "case.json").write_text(json.dumps(case_doc, indent=2, ensure_ascii=False))

    request.addfinalizer(finalize)
    return buf

# ---- optional: per-worker (run folder) index for convenience ----
def pytest_sessionfinish(session, exitstatus):
    """
    Create an index.json inside this worker's run folder for quick browsing.
    The full multi-worker merge is done by scripts/merge_report_runs.py in CI.
    """
    try:
        reports_root = _compute_run_root()
        items = []
        for case_dir in sorted(reports_root.glob("*")):
            if not case_dir.is_dir():
                continue
            cj = case_dir / "case.json"
            if cj.exists():
                doc = json.loads(cj.read_text())
                summary = {
                    "node": case_dir.name,
                    "case_id": doc["case"]["case_id"],
                    "tool_id": doc["case"]["tool_id"],
                    "status": doc["inference"]["status"],
                    "metrics": [{"metric": m.get("metric"), "status": m.get("status"), "score": m.get("score")} for m in doc.get("metrics", [])]
                }
                items.append(summary)
        if items:
            (reports_root / "index.json").write_text(json.dumps({"cases": items}, indent=2))
    except Exception:
        # never fail the test session for indexing
        pass

# ---- optional: hint path into pytest-json-report, if plugin present ----
def pytest_json_modifyreport(json_report):
    try:
        root = _compute_run_root()
        for test in json_report.get("tests", []):
            class_name = test.get("nodeid","").split("::")[0].split(".")[-1]
            hint = f"{root}/{class_name}/case.json"
            test.setdefault("extras", []).append({"name":"case_report","format":"text","content":hint})
    except Exception:
        pass
```

---

# tests/test\_analysis\_code\_execution\_simple.py (example test)

```python
# tests/test_analysis_code_execution_simple.py
from __future__ import annotations
import pytest

from core.types import EvalCase
from tests.base_tool_test import BaseToolTestClass
from tests.conftest import get_eval_cases

# import your real tool callable and metrics
from genie.agents.tools import general_code_execution_tool
from my_metrics import METRICS  # list of metric objects w/ evaluate(...) or evaluate_async(...)

EVAL_CASE_PATH = "tests/test_data/analysis_code_execution/simple"

@pytest.fixture(scope="module")
def function_tool():
    return general_code_execution_tool  # sync or async callable

@pytest.fixture(scope="class", params=get_eval_cases(EVAL_CASE_PATH), ids=lambda ec: ec.case_id)
def eval_case(request) -> EvalCase:
    return request.param

@pytest.fixture(scope="class", params=METRICS, ids=lambda m: m.name)
def metric(request):
    return request.param

@pytest.mark.evals
@pytest.mark.tools
@pytest.mark.tools("analysis_code_execution")
class TestAnalysisCodeExecution(BaseToolTestClass):
    # If all tools dereference via genie.utils.*, the global PATCH_TARGETS_GLOBAL is enough.
    # Override patch_targets(...) only if a tool still uses a use-site import that must be patched.

    @pytest.mark.asyncio
    async def test_metric(self, tool_inference, eval_case, metric, _metric_sink):
        await self.evaluate_and_record(metric, tool_inference, eval_case, _metric_sink)
```

---

# scripts/merge\_report\_runs.py (combine xdist workers into one index)

```python
# scripts/merge_report_runs.py
"""
Usage:
  python scripts/merge_report_runs.py reports gh_12345_a1

It will scan subfolders like:
  reports/gh_12345_a1_gw0, reports/gh_12345_a1_gw1, ...
and write:
  reports/gh_12345_a1_merged/index.json
"""
from __future__ import annotations
import json, sys
from pathlib import Path

def main(root_dir: str, run_prefix: str):
    root = Path(root_dir)
    workers = sorted(root.glob(f"{run_prefix}_gw*"))
    merged = {"runs": [], "cases": []}
    for w in workers:
        merged["runs"].append(w.name)
        idx = w / "index.json"
        if idx.exists():
            doc = json.loads(idx.read_text())
            merged["cases"].extend(doc.get("cases", []))
        else:
            # fallback: scan case.json directly
            for case_dir in w.glob("*"):
                cj = case_dir / "case.json"
                if cj.exists():
                    d = json.loads(cj.read_text())
                    merged["cases"].append({
                        "node": case_dir.name,
                        "case_id": d["case"]["case_id"],
                        "tool_id": d["case"]["tool_id"],
                        "status": d["inference"]["status"],
                        "metrics": [{"metric": m.get("metric"), "status": m.get("status"), "score": m.get("score")} for m in d.get("metrics", [])]
                    })
    out_root = root / f"{run_prefix}_merged"
    out_root.mkdir(parents=True, exist_ok=True)
    (out_root / "index.json").write_text(json.dumps(merged, indent=2))
    print(f"merged report: {out_root / 'index.json'}")

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: python scripts/merge_report_runs.py <reports_root> <run_prefix>")
        sys.exit(2)
    main(sys.argv[1], sys.argv[2])
```

---

# CI tips (GitHub Actions)

```yaml
# .github/workflows/tests.yml (snippet)
- name: Run tests (xdist)
  run: |
    pytest -q -n auto --json-report --json-report-file=pytest-report.json

- name: Upload raw reports (always)
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: pytest-reports-raw
    path: |
      reports/**
      pytest-report.json

# Optionally merge all worker runs (only on GH)
- name: Merge report runs
  if: always()
  run: |
    RUN_PREFIX="gh_${{ github.run_id }}_a${{ github.run_attempt }}"
    python scripts/merge_report_runs.py reports "$RUN_PREFIX"

- name: Upload merged report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: pytest-reports-merged
    path: reports/*_merged/index.json
```

---

# how to add new tool evals (teammate UX)

1. create eval cases (JSON) in a folder; each has `tool_id`, `function_call`, and `expectations`.
2. write a test class:

   * provide `function_tool` fixture that returns your callable,
   * parametrize `eval_case` from the folder,
   * parametrize `metric` from your metric list,
   * subclass `BaseToolTestClass` and write:

```python
@pytest.mark.asyncio
async def test_metric(self, tool_inference, eval_case, metric, _metric_sink):
    await self.evaluate_and_record(metric, tool_inference, eval_case, _metric_sink)
```

3. (optional) if your tool still imports a loader at the use-site, override `patch_targets` to add that dotted path. otherwise the global `"genie.utils.load_artifacts"` patch covers you.

---

that’s the whole end-to-end: **pytest-param → class-scoped inference → per-metric tests → case.json + artifacts → per-worker index → merged index**. once you confirm you’re emitting `case.json` + `artifacts/*`, we can layer on Streamlit/HTML views against `index.json` without touching the test code.
