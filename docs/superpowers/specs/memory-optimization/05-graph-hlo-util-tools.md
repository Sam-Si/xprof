# Graph / HLO / Utilization Tools — Memory Optimization Opportunity Notes

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `graph_viewer`, `inference_profile`, `utilization_viewer`, `smart_suggestion`, `perf_counters`, `pod_viewer`  
**CLI (related):** `get_graph_viewer_tool`, `get_utilization_viewer_tool`, `get_smart_suggestions_tool`, `get_kpi_metrics_tool`, `get_top_hlo_ops_tool` (+ `hlo_tools` via graph_viewer)

## Registry placement

Source: [`plugin/xprof/profile_plugin/tools/registry.py`](plugin/xprof/profile_plugin/tools/registry.py)

| Concern | Membership |
|---------|------------|
| **XPLANE_TOOLS** | All six tools listed |
| **HLO_TOOLS** | `graph_viewer` (+ `memory_viewer`; covered in `03-memory-tools-family.md`) |
| **XPLANE_TOOLS_ALL_HOSTS_SUPPORTED** | `pod_viewer` only (of this set) |
| **XPLANE_TOOLS_ALL_HOSTS_ONLY** | `pod_viewer`, `smart_suggestion` (also `overview_page`) |
| **Multi-host selection** | None of these six (`trace_viewer` / `trace_viewer@` only) |
| **Explicit single-XSpace** | `utilization_viewer` (C++ rejects `XSpaceSize() != 1`) |
| **Not ALL_HOSTS** | `graph_viewer`, `inference_profile`, `perf_counters`, `utilization_viewer` — host picker / first host / all paths per tool |

Dispatch:

| Layer | Path |
|-------|------|
| Python options | `tools/options/graph.py` (graph_viewer); smart_suggestion `refresh_suggestion` in `raw_to_tool_data.py` |
| Python convert | [`plugin/xprof/convert/raw_to_tool_data.py`](plugin/xprof/convert/raw_to_tool_data.py) |
| C++ router | [`xprof/convert/xplane_to_tools_data.cc`](xprof/convert/xplane_to_tools_data.cc) |
| HLO subpath | [`xprof/convert/hlo_to_tools_data.cc`](xprof/convert/hlo_to_tools_data.cc), [`xprof/convert/xplane_to_hlo.cc`](xprof/convert/xplane_to_hlo.cc) |
| Shared pipeline | See `02-convert-serve-cache-path.md` (gzip double-buffer, no concurrent convert cap) |

```text
XPlane host files
  ├─ HLO extract (tool_names / first graph open) → <module>.hlo_proto.pb
  │     └─ graph_viewer (HLO_TOOLS): HloProto → HloModule IR → HTML/DOT/txt/pb
  ├─ OpStats cache ALL_HOSTS.op_stats.pb
  │     ├─ pod_viewer (ALL_HOSTS_ONLY)
  │     └─ smart_suggestion rules (via ToolDataProvider)
  ├─ inference_profile: parallel per-host InferenceStats → sample → DataTable JSON
  ├─ utilization_viewer: single host XSpace → counter metrics DataTable
  └─ perf_counters: sequential hosts → raw counter rows DataTable
```

---

## 1. graph_viewer

### Data path

```text
UI / CLI → graph_viewer_options {type, node_name, module_name|program_id, graph_width, show_metadata, …}
  → GetHloProtoByOptions | fallback GetHloProtoByNodeName
  → ConvertHloProtoToGraphViewer
       type=graph: ConvertHloProtoToModule → RenderNeighborhoodAround DOT → WrapDotInHtml
       type=short_txt|long_txt: ConvertHloProtoToModule → HloModule::ToString
       type=pb|pbtxt: serialize proto (pb no IR; pbtxt text)
       type=adjacent_nodes: full module + adjacency JSON
  → application/json | text/html | octet-stream
FE: iframe Graphviz HTML + separate op_profile fetch for hover metrics
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| HLO extract (cold) | All XSpaces → HloProtoMap → N× `.hlo_proto.pb` on disk | Full multi-host XSpace walk once | **observed** (`xplane_to_hlo.cc`) |
| Load module | Full `xla::HloProto` in memory | Large modules tens–hundreds of MB | **observed** |
| IR materialization | `ConvertHloProtoToModule` → `unique_ptr<HloModule>` | **Proto + full IR coexist** | **observed** (`hlo_proto_to_graph_view.cc`) |
| type=graph | DOT string → HTML wrapper with embedded DOT | Peak ≈ IR + DOT + HTML | **observed** (`graphviz_helper.h`) |
| type=long_txt | Full module text (+ large constants if verbose) | Often larger than pb | **observed** |
| Node-name fallback | Scan every `*.hlo_proto.pb` until name match | Worst-case load **all** modules | **observed** (`GetHloProtoByNodeName`) |
| FE | iframe DOM + full SVG; `opProfileLimit` default 300 | GRAPH_HTML_THRESHOLD = 1e6 **warn only** | **observed** (`graph_viewer.ts`) |
| CLI | `get_graph_viewer` defaults `short_txt`; `hlo_tools` may request `long_txt` | Truncation only client-side (`max_lines`) | **observed** |

### Materialization waste

| # | What | Risk |
|---|------|------|
| G1 | Proto → HloModule for almost every type except raw `pb` | Dominant convert peak |
| G2 | Graph path always renders **kDot first**, then HTML/url wrap | Temporary double graph string |
| G3 | `AddGraphMetadata` (internal): parse JSON graph → dump again | Extra full copy (PLATFORM_GOOGLE ME path) |
| G4 | Processor sets `application/json` even for HTML graph body | Content-type mismatch; still full string |
| G5 | FE loads **op_profile** (limit 300) on every graph session for hover | Second large convert path |
| G6 | Default `graph_width`: FE=1, options builder=3, C++ default=3 | Width 3 neighborhood can explode HTML |
| G7 | No durable cache of rendered neighborhood / txt per (module, node, width) | Reconvert on pan/reselect |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| GV-1 | Cache rendered graph/txt keyed by module×node×width×flags on disk | **H** | observed | M |
| GV-2 | Bound graph HTML size server-side (hard fail / auto-shrink width) — FE only warns at 1 MB | **H** | observed | S–M |
| GV-3 | Avoid full module IR for `pb` / partial for adjacency-only | **M** | observed | M |
| GV-4 | Stream or zstd compress large HTML/txt; skip double gzip policy (shared) | **H** | hypothesized | M |
| GV-5 | Lazy op_profile: fetch metrics for visible nodes only, not limit=300 tree | **H** | observed | M |
| GV-6 | Index instruction→module at HLO extract time; kill full-module scan fallback | **M** | observed | M |
| GV-7 | Cap default graph_width=1 end-to-end (align FE / options / C++) | **M** | observed | S |
| GV-8 | Free HloProto after IR build (or build IR without retaining both) | **M** | hypothesized | M |
| GV-9 | CLI `summary` / neighborhood-only modes; never force `long_txt` without max_lines server-side | **M** | observed | S |

**Top:** GV-2, GV-5, GV-1, GV-7

### Frontend / CLI notes

- [`frontend/app/components/graph_viewer/graph_viewer.ts`](frontend/app/components/graph_viewer/graph_viewer.ts) — iframe load poll, inject runtime colors from op profile.
- [`plugin/xprof/cli/tools/get_graph_viewer_tool.py`](plugin/xprof/cli/tools/get_graph_viewer_tool.py) — thin HTTP client; full payload returned.
- [`plugin/xprof/cli/internal/oss/hlo_tools.py`](plugin/xprof/cli/internal/oss/hlo_tools.py) — `get_hlo_module_content` / neighborhood via `graph_viewer.json`.

---

## 2. inference_profile

### Data path

```text
Session XPlanes (all hosts in selection)
  → parallel: GetXSpace(arena) → Preprocess → GenerateInferenceStats per host
  → CombineInferenceStatsResult → RegroupInferenceStatsByModel
  → SampleInferenceStats (percentiles) → copy into InferenceStats.sampled_*
  → InferenceStatsToDataTableJson (metadata + per-model request/batch/tensor tables)
FE: array of Google DataTables; holds all models at once
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| Per-host convert | Full XSpace + all `RequestDetail` / `BatchDetail` | Parallel hosts → **Σ host peaks** | **observed** (`multi_xspace_to_inference_stats.cc` thread pool) |
| Combined proto | Full unsampled request lists per model | Dominates before sample | **observed** (`inference_stats.cc` `BuildRequestDetails`) |
| Sampling | 6 percentile bands × ≤10 samples | Wire much smaller than intermediate | **observed** (`kMaxNumDataSelectedPerPercentile=10`) |
| JSON | Joined DataTable JSON array | Sampled only + aggregates | **observed** |
| FE | All model tables retained | Multi-model multiplies | **observed** (`inference_profile.ts`) |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| IP-1 | Online / streaming sample: never retain all request_details | **H** | observed | L |
| IP-2 | Bound host parallelism (semaphore) instead of `min(MaxParallelism, N)` | **H** | observed | S |
| IP-3 | Drop full request_details after sample (clear before JSON) | **H** | observed | S |
| IP-4 | Per-model / percentile-column incremental re-fetch (today full recompute on column change via options) | **M** | hypothesized | M |
| IP-5 | FE: load selected model tables only | **M** | observed | M |
| IP-6 | Disk cache of combined InferenceStats / sampled JSON | **M** | hypothesized | M |
| IP-7 | Multi-host correctness path (FE warns b/474172782) — avoid loading N hosts when wrong | **M** | hypothesized | M |

**Top:** IP-1 / IP-3, IP-2, IP-5

Wire is already relatively lean (sampled). **Peak is intermediate residency**, not JSON size.

---

## 3. utilization_viewer

### Data path

```text
Exactly 1 XSpace (enforced)
  → PreprocessSingleHostXSpace(step_grouping=true, derived_timeline=true)
  → ConvertXSpaceToUtilizationViewer
       TPU planes → per sample line → counters_map → UtilizationCounters metrics rows
  → DataTable JSON (host, device, sample, node, name, achieved, peak, unit)
FE: full table → FilterDataProcessor copies per core for unit/bandwidth/HBM charts
CLI get_utilization_viewer: fetch tqx=csv → parse all rows → ~10 scalar % metrics
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| XSpace + preprocess | Full host plane + derived timeline | Same class as other single-host tools | **observed** |
| Counter expand | One DataTable row per (sample × metric) | Sample count can be large | **observed** |
| FE charts | Multiple filtered DataTable clones | Multiplier per core | **observed** |
| CLI | Full CSV then discard most rows | **UI-scale payload for scalars** | **observed** |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| UV-1 | CLI / `summary` convert mode: aggregate % only (no per-sample rows) | **H** | observed | M |
| UV-2 | Server-side host/device/node + sample downsample filters | **H** | observed | M |
| UV-3 | Skip derived_timeline if only counters needed (if safe) | **M** | hypothesized | M |
| UV-4 | FE: one DataView; no per-chart full copies | **M** | hypothesized | M |
| UV-5 | Cache utilization DataTable on disk per host | **L** | hypothesized | S |

**Top:** UV-1, UV-2

Not ALL_HOSTS; severity mainly sample cardinality × CLI full fetch.

---

## 4. smart_suggestion

### Data path

```text
ALL_HOSTS_ONLY host policy
  → SmartSuggestionProcessor::ProcessSession
       if !refresh && cache hit: read ALL_HOSTS.smart_suggestion.pb
       else:
         ToolDataProviderImpl
           GetOpStats → ConvertMultiXSpaceToCombinedOpStatsWithCache  // N OpStats peak
           GetOverviewPage / InputPipeline / OpProfile from OpStats
           GetMemoryProfile via full tool convert + JSON→proto parse
           GetEventTimeFractionAnalyzer via tool convert
         SmartSuggestionEngine + rules → SmartSuggestionReport (small)
         WriteBinaryProto SMART_SUGGESTION
  → MessageToJsonString(always_print) → FE HTML cards
CLI: get_smart_suggestions → HTTP, cached 86400s decorator
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| OpStats combine | `vector<OpStats>` for **all** hosts then combine | Explicit TODO to stream-combine | **observed** (`multi_xplanes_to_op_stats.cc:51-52`) |
| Memory profile side convert | Full memory_profile tool + JSON string + parse back | Double materialization | **observed** (`tool_data_provider_impl.h`) |
| Event time analyzer | Another full convert | Extra XSpace walks | **observed** |
| Report | Few `SmartSuggestion` strings | Tiny | **observed** (`smart_suggestion.proto`) |
| JSON opts | `always_print_fields_with_no_presence = true` | Minor | **observed** |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| SS-1 | Stream OpStats combine (shared with pod_viewer / overview) — drop all_op_stats vector | **H** | observed | L |
| SS-2 | Prefer binary proto paths in ToolDataProvider; **no JSON round-trip** | **H** | observed | M |
| SS-3 | Rule-specific minimal signals (duty cycle, % bound) without full MemoryProfile / OpProfile trees | **H** | hypothesized | L |
| SS-4 | Single-flight + durable report cache (already partial); share warm OpStats with UI | **M** | observed | S–M |
| SS-5 | Lazy rule evaluation: stop after K suggestions; skip unused providers | **M** | hypothesized | M |
| SS-6 | FE already small; strip HTML server-side for CLI (done via `strip_html`) | **L** | observed | — |

**Top:** SS-1 (shared), SS-2, SS-3

First open of smart_suggestion can **amplify** OpStats + memory_profile peaks on ALL_HOSTS.

---

## 5. perf_counters

### Data path

```text
For each host (sequential):
  GetXSpace(arena) → ConvertXSpaceToPerfCounters
    TPU/GPU planes → every event with kCounterValue → DataTable row
      (Host, Chip, Kernel, Sample, Counter, Value, Description, Set)
→ data_table.ToJson()
Special: names_only=1 handled outside convert (`services/counter_names.py`)
FE: full google.visualization.DataTable; pageSize 30–200 (UI page only)
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| Host loop | One XSpace at a time (arena) | Sequential peak ≈ max host | **observed** |
| Rows | Unaggregated event-level | O(samples × counters); Description/Set strings repeated | **observed** |
| JSON | Full table | Large multi-host | **observed** |
| FE | Full table in JS heap | Client filter only | **observed** (`perf_counters.ts`) |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| PC-1 | Aggregate / downsample samples server-side (mean/max per counter) for UI default | **H** | observed | M |
| PC-2 | Intern Description/Set strings; optional omit Description on wire | **M** | observed | S |
| PC-3 | Host/chip/counter filters as convert options | **M** | observed | S |
| PC-4 | Columnar / protobuf wire instead of DataTable JSON | **M** | hypothesized | L |
| PC-5 | FE virtual scroll + progressive fetch | **M** | hypothesized | M |
| PC-6 | Multi-host: don’t default to all hosts if UI only shows one | **L** | hypothesized | S |

**Top:** PC-1, PC-2, PC-3

---

## 6. pod_viewer

### Data path

```text
ALL_HOSTS_ONLY
  → ConvertMultiXSpaceToCombinedOpStatsWithCache
       miss: for each host XSpace → OpStats; hold all; CombineAllOpStats; write ALL_HOSTS.op_stats.pb
  → ConvertOpStatsToPodViewer
       PodStatsRecord per (step × core) + step_breakdown_events + diagnostics
  → MessageToJsonString(always_print)
FE: full PodViewerDatabase → topology + stack bars
```

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| OpStats intermediate | N host OpStats + combined | **Same SS-1 hotspot**; TODO in code | **observed** |
| PodViewerDatabase | steps × cores maps | Grows with duration × topology | **observed** (`op_stats_to_pod_viewer.cc`) |
| JSON always_print | Empty fields expanded | Wire bloat | **observed** |
| FE | Full JSON retained | Topology graph | **observed** (`pod_viewer/`) |

### Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| PV-1 | Stream OpStats combine (shared SS-1) | **H** | observed | L |
| PV-2 | Cap steps in merge (`std::numeric_limits::max()` today) | **H** | observed | S |
| PV-3 | Disable always_print; slim bottleneck-only view mode | **M** | observed | S |
| PV-4 | Pod-only intermediate (skip full OpStats fields unused by pod) | **H** | hypothesized | L |
| PV-5 | FE: step window / virtualize topology | **M** | hypothesized | M |

**Top:** PV-1, PV-2, PV-4

---

## 7. Related CLI tools

| CLI | Convert dependency | Memory issue | Severity | Confidence |
|-----|-------------------|--------------|----------|------------|
| **get_graph_viewer** | Full `graph_viewer` | Returns entire HTML/txt; no server bound | **H** | observed |
| **get_utilization_viewer** | Full `utilization_viewer` CSV → scalars | **Classic UI-scale force** (like GMP-1 pattern in memory tools) | **H** | observed |
| **get_smart_suggestions** | `smart_suggestion` | Peak on cold cache = multi-tool convert; decorator 86400 helps warm | **H** cold / **L** warm | observed |
| **get_kpi_metrics** | `get_overview` + `get_memory_profile` | Two full tool converts for ~6 scalars | **H** | observed |
| **get_top_hlo_ops** | Full `hlo_op_profile` tree → flatten → heapq top-K | Full tree for K=10; `@cached(86400)` | **H** | observed |
| **hlo_tools** (module text / neighborhood) | `graph_viewer` types | `long_txt` module dump; client max_lines only | **M–H** | observed |

### CLI opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| CLI-GV-1 | Server `summary` / `top_k` / `neighborhood` flags for graph + op profile | **H** | observed |
| CLI-UV-1 | utilization `metrics_only` convert (UV-1) | **H** | observed |
| CLI-KPI-1 | overview + memory summary modes (align FAM-1 from memory family) | **H** | observed |
| CLI-TOP-1 | op_profile convert option `top_ops_limit` without full tree materialization | **H** | hypothesized |
| CLI-SS-1 | Rely on SMART_SUGGESTION disk cache; never refresh unless asked | **M** | observed |

---

## 8. Cross-cutting / shared with convert-serve path

| ID | Opportunity | Applies to | Severity | Confidence |
|----|-------------|------------|----------|------------|
| X-1 | Stream-combine OpStats (kill `vector<OpStats>`) | pod_viewer, smart_suggestion, any OpStats consumer | **H** | observed |
| X-2 | HLO extract once + instruction index | graph_viewer, memory_viewer, tool_names | **M** | observed |
| X-3 | View modes: `full \| ui \| summary \| cli` on convert | all tools here | **H** | observed |
| X-4 | Cap concurrent converts (02-doc) | inference parallel + foreground UI | **H** | hypothesized |
| X-5 | Stop `always_print` JSON default | pod_viewer, smart_suggestion | **M** | observed |
| X-6 | Durable tool result cache keyed by options | graph, util, inference, perf | **H** | hypothesized |
| X-7 | CLI must not force UI-scale protos | UV, KPI, top_hlo, graph long_txt | **H** | observed |

---

## 9. Ranked priority (this tool set)

| Rank | ID | Description | Severity | Confidence | Effort |
|------|-----|-------------|----------|------------|--------|
| 1 | X-1 / SS-1 / PV-1 | Stream OpStats multi-host combine | **H** | observed | L |
| 2 | GV-2 + GV-7 | Bound graph HTML + default width=1 | **H** | observed | S |
| 3 | UV-1 / CLI-UV-1 | Utilization summary mode for CLI | **H** | observed | M |
| 4 | CLI-TOP-1 / CLI-KPI-1 | Top-ops & KPI without full trees | **H** | observed | M |
| 5 | IP-1 / IP-3 | Inference: sample without retaining all requests | **H** | observed | M–L |
| 6 | SS-2 | Kill ToolDataProvider JSON round-trips | **H** | observed | M |
| 7 | PC-1 | Aggregate perf_counters for default UI | **H** | observed | M |
| 8 | GV-5 | Lazy graph op_profile metrics | **H** | observed | M |
| 9 | GV-1 / X-6 | Disk cache rendered HLO views | **H** | hypothesized | M |
| 10 | PV-2 | Cap steps in pod merge | **H** | observed | S |
| 11 | IP-2 | Bound inference host parallelism | **H** | observed | S |
| 12 | X-5 | always_print off | **M** | observed | S |

### Top 5 wins

1. **Stream OpStats combine** — largest shared multi-host peak (pod + smart_suggestion + overview family).  
2. **Graph size bounds** — prevent multi-MB HTML/IR spikes; align width defaults.  
3. **CLI summary modes** — utilization, KPI, top HLO ops must not pull UI-scale JSON.  
4. **Inference intermediate drop** — keep wire sampled; free full request lists early; bound host threads.  
5. **Smart suggestion provider path** — binary/shared OpStats; no memory_profile JSON parse tax.

---

## 10. Risks

| Opt | Correctness | UX |
|-----|-------------|-----|
| Graph width hard cap | Miss distant nodes | Must re-request with larger width |
| Stream OpStats | Step intersection bugs | Rare empty pod views |
| Inference online sample | Percentile skew if reservoir bad | Slightly different sample sets |
| Util summary | Lose per-sample spikes | CLI/agents ok; UI needs full mode |
| always_print off | FE assuming empty keys | Usually fine if FE uses proto3 defaults |
| Top-ops without full tree | Miss nested leaf accounting | Need same leaf definition as FE |

---

## 11. Severity × confidence summary by tool

| Tool | Peak severity | Dominant stage | Confidence |
|------|---------------|----------------|------------|
| graph_viewer | **H** | HloModule IR + DOT/HTML; FE op_profile | observed |
| inference_profile | **H** (convert) / **M** (wire) | Parallel XSpace + full request_details | observed |
| utilization_viewer | **M** UI / **H** CLI waste | Per-sample rows; CLI full CSV | observed |
| smart_suggestion | **H** cold | Nested OpStats + side converts | observed |
| perf_counters | **M–H** | Unaggregated multi-host rows | observed |
| pod_viewer | **H** multi-host | OpStats vector + step×core JSON | observed |

---

## 12. Real repo paths cited

### Registry / plugin / Python convert
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/tools/options/graph.py`
- `plugin/xprof/convert/raw_to_tool_data.py`
- `plugin/xprof/profile_plugin/services/counter_names.py`

### C++ convert
- `xprof/convert/xplane_to_tools_data.cc`
- `xprof/convert/hlo_to_tools_data.cc`
- `xprof/convert/xplane_to_hlo.cc`
- `xprof/convert/graph_viewer_processor.cc`
- `xprof/convert/hlo_proto_to_graph_view.cc` / `.h`
- `xprof/convert/graphviz_helper.h`
- `xprof/convert/tool_options.h` (`kDefaultWidth = 3`, type names)
- `xprof/convert/inference_stats_processor.cc`
- `xprof/convert/multi_xspace_to_inference_stats.cc`
- `xprof/convert/inference_stats.cc` / `inference_stats_sampler.cc`
- `xprof/convert/xplane_to_utilization_viewer.cc`
- `xprof/convert/xplane_to_perf_counters.cc`
- `xprof/convert/pod_viewer_processor.cc`
- `xprof/convert/op_stats_to_pod_viewer.cc` / `op_stats_to_pod_stats.cc`
- `xprof/convert/multi_xplanes_to_op_stats.cc`
- `xprof/convert/smart_suggestion_processor.cc`
- `xprof/convert/smart_suggestion/tool_data_provider_impl.h`
- `xprof/convert/repository.h` (`kAllHostsIdentifier`)
- `xprof/convert/repository.cc` (`SMART_SUGGESTION` suffix)

### Protos
- `plugin/xprof/protobuf/smart_suggestion.proto`
- `plugin/xprof/protobuf/pod_stats.proto`
- `plugin/xprof/protobuf/inference_stats.pb.h` (via sources)

### Frontend
- `frontend/app/components/graph_viewer/graph_viewer.ts`
- `frontend/app/components/inference_profile/inference_profile.ts`
- `frontend/app/components/utilization_viewer/utilization_viewer.ts`
- `frontend/app/components/smart_suggestion/smart_suggestion_view.ts`
- `frontend/app/components/perf_counters/perf_counters.ts`
- `frontend/app/components/pod_viewer/pod_viewer.ts`

### CLI
- `plugin/xprof/cli/tools/get_graph_viewer_tool.py`
- `plugin/xprof/cli/tools/get_utilization_viewer_tool.py`
- `plugin/xprof/cli/tools/get_smart_suggestions_tool.py`
- `plugin/xprof/cli/tools/get_kpi_metrics_tool.py`
- `plugin/xprof/cli/tools/get_top_hlo_ops_tool.py`
- `plugin/xprof/cli/internal/oss/hlo_tools.py`
- `plugin/xprof/cli/xprof_cli.py`

### Related specs
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md`
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (HLO memory_viewer sibling)

---

## Appendix — Key constants

| Constant | Value | Where |
|----------|-------|-------|
| HLO_TOOLS | `graph_viewer`, `memory_viewer` | `registry.py` |
| ALL_HOSTS_ONLY (this set) | `pod_viewer`, `smart_suggestion` | `registry.py` |
| kDefaultWidth | 3 | `tool_options.h` |
| Options builder default width | 3 | `tools/options/graph.py` |
| FE default graphWidth | 1 | `graph_viewer.ts` |
| GRAPH_HTML_THRESHOLD | 1_000_000 bytes (warn) | `graph_viewer.ts` |
| FE opProfileLimit | 300 | `graph_viewer.ts` |
| Inference percentiles | 50,75,90,99,99.9,99.99 | `inference_stats_sampler.cc` |
| Max samples / percentile | 10 | `kMaxNumDataSelectedPerPercentile` |
| Utilization hosts | exactly 1 XSpace | `xplane_to_tools_data.cc` |
| CLI util/smart/top cache | 86400 s | decorators on CLI tools |
| OpStats merge step cap | `uint32 max` (unbounded) | `multi_xplanes_to_op_stats.cc` |
| JSON always_print | true | pod_viewer, smart_suggestion processors |
