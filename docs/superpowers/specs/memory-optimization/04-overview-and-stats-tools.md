# Overview & Stats Tools Family — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** overview_page, input_pipeline_analyzer, framework_op_stats, kernel_stats, hlo_stats, op_profile, roofline_model

## Registry placement

| Concern | Path / policy |
|---------|----------------|
| XPLANE tools | `plugin/xprof/profile_plugin/tools/registry.py` — all seven ∈ `XPLANE_TOOLS` |
| ALL_HOSTS supported | overview_page, input_pipeline_analyzer, framework_op_stats, kernel_stats (`XPLANE_TOOLS_ALL_HOSTS_SUPPORTED`) |
| ALL_HOSTS only | overview_page only among this family (`XPLANE_TOOLS_ALL_HOSTS_ONLY`) |
| Not in ALL_HOSTS UI sets | op_profile, hlo_stats, roofline_model (UI lists per-host only when multi-host) |
| Default warm-up | `DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')` — overview always warmed |
| Python dispatch | `plugin/xprof/convert/raw_to_tool_data.py` → `xspace_to_tools_data` |
| C++ dispatch | `xprof/convert/xplane_to_tools_data.cc` + ProfileProcessor path |
| Shared OpStats base | `xprof/convert/base_op_stats_processor.{h,cc}` / `op_stats_processor.cc` |
| Combined cache | `ConvertMultiXSpaceToCombinedOpStatsWithCache` in `multi_xplanes_to_op_stats.cc` |
| Host selection | `plugin/xprof/profile_plugin/services/hosts.py` (`HostSelector`) |
| Host list for UI | `plugin/xprof/profile_plugin/tools/filenames.py` `hosts_from_xplane_filenames` |
| Proto hub | `plugin/xprof/protobuf/op_stats.proto` |

### Shared short path (all seven)

```text
HTTP /data → HostSelector (1..N XPlanes)
  → raw_to_tool_data.py (tool branch; use_saved_result default True)
  → C++ ConvertMultiXSpaceToCombinedOpStatsWithCache
       options ALWAYS:
         generate_op_metrics_db = true
         generate_step_db = true
         generate_kernel_stats_db = true
       cache: ALL_HOSTS.op_stats.pb (and/or per-host OP_STATS)
  → per-tool ProcessCombinedOpStats / ConvertOpStatsTo*
  → JSON string (DataTable JSON or MessageToJsonString)
  → respond() gzip
  → Angular component holds full parsed object graph
```

**Observed family invariant:** every tool pays the full OpStats extract cost (metrics + steps + kernels) even when the UI view only needs a slice. Cache hit on `ALL_HOSTS` OpStats amortizes recompute across tools, but peak convert and wire still tool-specific.

---

## 1. overview_page

### Short path
`raw_to_tool_data.py` (`overview_page`) → `OverviewPageProcessor` / `ConvertMultiXSpacesToOverviewPage` → `ConvertMultiXSpaceToCombinedOpStatsWithCache` → `ConvertOpStatsToOverviewPage` → (if not training) **`ConvertMultiXSpaceToInferenceStats` re-reads all XSpaces** → `OverviewPageToJson` (array of DataTables) → FE `frontend/app/components/overview_page/`.

### ALL_HOSTS
- **ALL_HOSTS_ONLY** when multi-host: UI offers only aggregate; warm-up always all hosts.  
- HostSelector empty host → all XPlanes (same as ALL_HOSTS).  
**Severity: H multi-host first paint.**

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| OpStats combine | All host OpStats resident before merge (`multi_xplanes_to_op_stats.cc` TODO to stream-combine) | observed |
| Inference branch | Second full multi-XSpace pass for latency (not from OpStats cache) | observed |
| Wire | Compact DataTable JSON (analysis, steptime/input, env, tips, inference, diagnostics) — smaller than full OpStats | observed |
| FE | Holds tuple of DataTables; step-time graph uses embedded input-pipeline analysis | observed |
| Warm-up | Always in `DEFAULT_CACHE_TOOLS` → forces full-session OpStats build early | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| OV-1 | Stream-combine OpStats one host at a time (fix multi_xplanes TODO) | **H** | observed |
| OV-2 | OpStatsOptions per tool (overview may skip `kernel_stats_db`) | **M** | hypothesized |
| OV-3 | Cache InferenceStats / attach to OpStats so overview avoids second XSpace sweep | **H** | observed gap |
| OV-4 | Bound or summarize steptime series embedded in overview JSON for long profiles | **M** | hypothesized |
| OV-5 | Warm-up: write per-host OpStats incrementally; avoid holding N hosts + combined | **H** | hypothesized |
| OV-6 | Slim recommendation HTML strings / tip tables | **L** | hypothesized |

**Top:** OV-1, OV-3, OV-5

---

## 2. input_pipeline_analyzer

### Short path
`raw_to_tool_data.py` → `InputPipelineProcessor` → combined OpStats → `ConvertOpStatsToInputPipelineAnalysis` → `InputPipelineAnalysisResultToDataTableJson` → FE `frontend/app/components/input_pipeline/` (summary + host/device detail + max infeed).

### ALL_HOSTS
- ∈ **ALL_HOSTS_SUPPORTED** (UI: ALL_HOSTS + per-host).  
- Per-host mode: one XPlane → lower peak. Aggregate: full multi-host OpStats.  
**Severity: H on ALL_HOSTS; M per-host.**

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Needs | Primarily `step_db` (+ metrics for host ops / tf.data stats) | hypothesized |
| Still builds | Full kernel_stats + full op_metrics always | observed |
| Wire | Per-step detail rows over **unbounded step count** (no kMax in convert) | observed |
| FE | Full tables in component state; charts re-slice columns client-side | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| IP-1 | OpStatsOptions: step_db + host metrics only | **H** | hypothesized |
| IP-2 | Cap / downsample per-step rows for UI (summary + top-K outliers) | **H** | hypothesized |
| IP-3 | Prefer per-host default when multi-host (don't auto all-hosts) | **M** | observed policy gap |
| IP-4 | Cache tool JSON separately from OpStats so step-table rebuild is free | **M** | hypothesized |

**Top:** IP-1, IP-2

---

## 3. framework_op_stats

### Short path
`raw_to_tool_data.py` → `FrameworkOpStatsProcessor` → OpStats → `ConvertOpStatsToTfStats` (`kMaxNumOfOps = 500` host + 500 device) → **dual** with_idle / without_idle tables → `TfStatsToDataTableJson` as JSON array of two tables → FE `framework_op_stats/` + NgRx store + pie charts / stats tables.

### ALL_HOSTS
- ∈ **ALL_HOSTS_SUPPORTED**. Aggregate merges metrics across hosts.  
**Severity: M–H** (bounded top-500 helps wire; OpStats peak still H).

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Cap | Top 500 ops per side — good wire bound | observed |
| Dual tables | with_idle + without_idle nearly 2× JSON | observed |
| Intermediate | `CreateTfMetricsDbFromDeviceOpMetricsDb` + `GroupKernelReportsByOpName` | observed |
| FE | NgRx keeps full data + optional diff run; pie processors clone views | observed |
| Table UI | Presentation paging only if present; full rows still in memory | hypothesized |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| FO-1 | Emit only the idle mode FE needs (toggle server-side) | **M** | observed |
| FO-2 | Skip kernel_stats_db when TensorCore column not needed (non-GPU) | **M** | hypothesized |
| FO-3 | Diff mode: don't hold two full runs if only delta needed | **M** | hypothesized |
| FO-4 | Shared OpStats cache already helps — ensure host-scoped cache for per-host UI | **M** | observed |

**Top:** FO-1, FO-2

---

## 4. kernel_stats

### Short path
`raw_to_tool_data.py` → `KernelStatsProcessor` → OpStats → **`KernelStatsToDataTableJson(combined_op_stats.kernel_stats_db())` with no row limit** → FE `kernel_stats/` via common_data_store; table `pageSize: 100` (UI only).

### ALL_HOSTS
- ∈ **ALL_HOSTS_SUPPORTED**. Combine merges kernel reports.  
**Severity: H** for large GPU sessions (unique kernels × launch configs).

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Build | Always on every OpStats path (`generate_kernel_stats_db=true`) even for non-kernel tools | observed |
| Wire | **All** `kernel_stats_db.reports()` → rows (name, dims, op_name, durations…) | observed |
| FE | Full SimpleDataTable in store; Google chart table pages 100 rows | observed |
| Waste | Full OpMetrics + step_db still built for this tool | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| KS-1 | Top-K / cumulative-time cutoff on wire (mirror FO `kMaxNumOfOps` / roofline 1000) | **H** | observed gap |
| KS-2 | OpStatsOptions: kernel_stats_db only for this tool | **H** | hypothesized |
| KS-3 | Server-side filter (name substring) + pagination params | **M** | hypothesized |
| KS-4 | FE: virtualize / drop non-visible columns from resident table | **M** | hypothesized |
| KS-5 | Don't regenerate metrics/step DBs when serving from full OP_STATS cache (lazy field use) | **L** | hypothesized |

**Top:** KS-1, KS-2

---

## 5. hlo_stats

### Short path
`raw_to_tool_data.py` → `HloStatsProcessor` / unified `UnifiedHloStatsProcessor` → OpStats → `ConvertOpStatsToHloStats` → **`SortedOpMetricsDb` with no max_records** + recursive SparseCore children → `HloStatsToDataTableJson` (includes full **hlo_op_expression** long text) → FE `hlo_stats/` (table + pie + flop chart; `pageSize: 10` UI only).

### ALL_HOSTS
- **Not** in ALL_HOSTS_SUPPORTED/ONLY → multi-host UI lists per host.  
- Empty host still selects **all** XPlanes (`hosts.py`) → accidental multi-host convert possible.  
**Severity: H** (unbounded rows + long expressions); multi-host accidental **H**.

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Rows | One row per HLO metric (+ SC children); expressions can be multi-KB | observed |
| Wire columns | ~29 columns including expressions, source_info, BW fields | observed |
| FE | Full table held; expression column often hidden but still downloaded | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| HS-1 | Top-K by self-time (e.g. 500–2000) like framework_op_stats | **H** | observed gap |
| HS-2 | Lazy long expression: name-only list + detail-by-id fetch | **H** | hypothesized |
| HS-3 | OpStatsOptions trim (device metrics only; skip kernel_stats) | **M** | hypothesized |
| HS-4 | Require explicit host when multi-host (no empty-host all-planes) | **M** | observed |
| HS-5 | FE strip unused columns after parse | **L** | hypothesized |

**Top:** HS-1, HS-2, HS-4

---

## 6. op_profile

### Short path
`raw_to_tool_data.py` (`op_profile` / `hlo_op_profile`, `group_by`) → `OpProfileProcessor` / `UnifiedOpProfileProcessor` → OpStats → `ConvertOpStatsToOpProfile(..., op_profile_limit=100, group_by)` → **`MessageToJsonString` with `always_print_fields_with_no_presence = true`** → FE `op_profile/` tree; **client cache Map per group_by**.

### ALL_HOSTS
- Not in ALL_HOSTS UI sets; typically one host. Empty host → all XPlanes (same pitfall as hlo_stats).  
**Severity: M** wire (capped tree); **H** if multi-host OpStats forced.

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Cap | children_per_node limit 100 — good | observed |
| JSON | always_print inflates empty metrics fields | observed |
| FE | Caches each group_by result for session; tree still full proto | observed |
| Regroup | Changing group_by re-runs convert (unless cached) over same OpStats | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| OP-1 | Disable always_print empty fields | **M** | observed |
| OP-2 | Cache OpProfile JSON keyed by group_by on disk | **M** | hypothesized |
| OP-3 | Build profile from flat metrics without full step/kernel DBs | **M** | hypothesized |
| OP-4 | FE: free unused group_by cache entries / cap Map size | **L** | observed |
| OP-5 | Expose lower op_profile_limit via tool options | **L** | hypothesized |

**Top:** OP-1, OP-3

---

## 7. roofline_model

### Short path
`raw_to_tool_data.py` → `RooflineModelProcessor` → OpStats → **two** `ConvertOpStatsToRooflineModel` (include infeed/outfeed true + false), merge records → `RooflineModelToDataTableJson` (`kMaxNumRecords = 1000`) → FE `roofline_model/` builds multiple `google.visualization.DataTable`s (raw, program, op, scatter).

### ALL_HOSTS
- Not in ALL_HOSTS UI sets; empty host → all planes pitfall.  
**Severity: M** (capped 1000) + FE multi-DataTable duplication.

### Materialization notes
| Stage | What | Confidence |
|-------|------|------------|
| Dual convert | Two full roofline passes over same metrics | observed |
| Cap | Top 1000 ops | observed |
| FE | dataTableRaw + filtered program/op + scatter copies → multiplies heap | observed |

### Opportunities
| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| RF-1 | Single pass with include_infeed flag as column/filter | **M** | observed |
| RF-2 | Avoid cloning full DataTables; use DataView filters | **M** | hypothesized |
| RF-3 | OpStatsOptions without kernel_stats if unused | **L** | hypothesized |
| RF-4 | Lower default kMaxNumRecords or progressive load | **L** | hypothesized |

**Top:** RF-1, RF-2

---

## Family shared themes

### Architecture
All seven tools are **OpStats fan-out tools**: one expensive multi-host extract, many JSON projections. Cache of `ALL_HOSTS` OpStats is the main amortization lever (`multi_xplanes_to_op_stats.cc`, `base_op_stats_processor.cc` Map/Reduce).

### Ranked shared opportunities

| ID | Opportunity | Hotspot | Confidence | Affects |
|----|-------------|---------|------------|---------|
| FAM-OS-1 | **Incremental combine**: convert host → merge → drop host OpStats (fix TODO) | **H** | observed | all multi-host |
| FAM-OS-2 | **Per-tool OpStatsOptions** (don't always build metrics+step+kernel) | **H** | observed | all; biggest KS/IP/OV |
| FAM-OS-3 | **Wire row caps** where missing (kernel_stats, hlo_stats) | **H** | observed | KS, HS |
| FAM-OS-4 | **Lazy heavy fields** (HLO expressions, recommendation HTML) | **H** | hypothesized | HS, OV |
| FAM-OS-5 | **Host policy hygiene**: empty host ≠ all planes for non-aggregate tools; align op_profile/hlo_stats/roofline with explicit host required | **M** | observed | OP, HS, RF |
| FAM-OS-6 | **Tool-result disk cache** beyond OpStats (JSON keyed by tool+host+options) | **M** | hypothesized | all |
| FAM-OS-7 | **JSON bloat policy**: no always_print; DataTable omit empty cells | **M** | observed | OP, tables |
| FAM-OS-8 | **FE presentation ≠ storage**: pageSize only draws; still holds full payload — virtualize or server page | **M** | observed | KS, HS, FO |
| FAM-OS-9 | **Concurrent convert cap** with warm-up (overview is DEFAULT_CACHE_TOOLS) | **H** | hypothesized | OV + siblings |
| FAM-OS-10 | **InferenceStats sharing** so overview doesn't re-read XSpaces | **H** | observed | OV |

### ALL_HOSTS policy matrix (this family)

| Tool | ALL_HOSTS_SUPPORTED | ALL_HOSTS_ONLY | Typical peak driver |
|------|---------------------|----------------|---------------------|
| overview_page | yes | **yes** | N× XSpace → OpStats + optional N× inference |
| input_pipeline_analyzer | yes | no | N× OpStats; unbounded steps |
| framework_op_stats | yes | no | N× OpStats; wire capped 500×2 |
| kernel_stats | yes | no | N× OpStats; **uncapped** kernel rows |
| hlo_stats | no | no | Per-host preferred; **uncapped** HLO rows |
| op_profile | no | no | Tree cap 100; OpStats full |
| roofline_model | no | no | Cap 1000; dual convert + FE copies |

### Family priority order
1. **FAM-OS-1** — incremental OpStats combine (multi-host peak).  
2. **FAM-OS-2** — per-tool generate flags (stop building unused DBs).  
3. **KS-1 / HS-1** — wire top-K for unbounded tables.  
4. **OV-3 / FAM-OS-10** — stop second XSpace pass on overview inference.  
5. **HS-2 / FAM-OS-4** — lazy long text fields.  
6. **FAM-OS-5** — host selection safety for non-ALL_HOSTS tools.  
7. **OP-1 / FO-1 / RF-1** — dual payload / always_print / dual roofline cleanup.  
8. **FAM-OS-8** — FE virtualization after wire is bounded.

### Severity legend
- **H** — multi-host peak, unbounded wire, or default-path warm-up cost likely to OOM or multi-GB.  
- **M** — meaningful 2× waste or browser heap, fixed with moderate effort.  
- **L** — polish / secondary.

### Confidence legend
- **observed** — directly evidenced by code paths cited.  
- **hypothesized** — structural inference without measured heap profiles.

---

## Paths cited

### Plugin / registry / dispatch
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/tools/filenames.py`
- `plugin/xprof/profile_plugin/services/hosts.py`
- `plugin/xprof/convert/raw_to_tool_data.py`

### Convert (shared)
- `xprof/convert/base_op_stats_processor.{h,cc}`
- `xprof/convert/op_stats_processor.cc`
- `xprof/convert/multi_xplanes_to_op_stats.{h,cc}`
- `xprof/convert/xplane_to_op_stats.{h,cc}`
- `xprof/convert/xplane_to_tools_data.cc`
- `xprof/convert/data_table_utils.cc`
- `plugin/xprof/protobuf/op_stats.proto`

### Convert (per tool)
- `xprof/convert/overview_page_processor.cc`
- `xprof/convert/op_stats_to_overview_page.{h,cc}`
- `xprof/convert/multi_xspace_to_inference_stats.{h,cc}`
- `xprof/convert/input_pipeline_processor.cc`
- `xprof/convert/op_stats_to_input_pipeline_analysis.{h,cc}`
- `xprof/convert/framework_op_stats_processor.cc`
- `xprof/convert/op_stats_to_tf_stats.{h,cc}`
- `xprof/convert/kernel_stats_processor.cc`
- `xprof/convert/xplane_to_kernel_stats_db.{h,cc}`
- `xprof/convert/hlo_stats_processor.cc`
- `xprof/convert/op_stats_to_hlo_stats.{h,cc}`
- `xprof/convert/unified_hlo_stats_processor.{h,cc}`
- `xprof/convert/op_profile_processor.cc`
- `xprof/convert/op_stats_to_op_profile.{h,cc}`
- `xprof/convert/op_profile_builder.{h,cc}`
- `xprof/convert/unified_op_profile_processor.{h,cc}`
- `xprof/convert/roofline_model_processor.cc`
- `xprof/convert/op_stats_to_roofline_model.{h,cc}`
- `xprof/convert/unified_tools_registration.cc`

### Frontend
- `frontend/app/components/overview_page/`
- `frontend/app/components/input_pipeline/`
- `frontend/app/components/framework_op_stats/`
- `frontend/app/components/kernel_stats/` (+ `kernel_stats_table/`)
- `frontend/app/components/hlo_stats/`
- `frontend/app/components/op_profile/`
- `frontend/app/components/roofline_model/`
- `frontend/app/services/data_dispatcher/data_dispatcher_base.ts`
- `frontend/app/store/common_data_store/`
- `frontend/app/store/framework_op_stats/`

### Related specs
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md` (gzip, warm-up concurrency)
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (non-OpStats memory tools)

---

## Log line

```
OVERVIEW_STATS|DONE|04-overview-and-stats-tools.md|OpStats fan-out family: multi-host combine + always-full OpStatsOptions dominate; kernel/hlo unbounded wire; overview dual XSpace inference path
```
