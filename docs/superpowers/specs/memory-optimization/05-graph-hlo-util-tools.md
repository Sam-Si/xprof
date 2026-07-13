# Graph / HLO / Utilization Tools — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `graph_viewer`, `inference_profile`, `utilization_viewer`, `smart_suggestion`, `perf_counters`, `pod_viewer`  
**CLI (related):** `get_graph_viewer_tool`, `get_utilization_viewer_tool`, `get_smart_suggestions_tool`, `get_kpi_metrics_tool`, `get_top_hlo_ops_tool`, `hlo_tools` (`get_hlo_module_content`, `get_hlo_neighborhood`)  
**Companion specs:** convert/serve/cache shared path in `02-convert-serve-cache-path.md`; HLO sibling `memory_viewer` in `03-memory-tools-family.md`; OpStats family in `04-overview-and-stats-tools.md`; synthesis in `00-synthesis-memory-optimization.md`

---

## How to read this document

| Convention | Meaning |
|------------|---------|
| Severity **H** | High: multi-MiB intermediate/wire, N× full convert, or UI-scale payload forced for scalar/CLI need |
| Severity **M** | Medium: measurable waste, bounded path, or secondary multiplier |
| Severity **L** | Low: small absolute cost, optional path, or already mitigated |
| Confidence **observed** | Confirmed by code path with `path:line` (or `path:start-end`) cites |
| Confidence **hypothesized** | Plausible from structure; needs measurement on real multi-host / large-module sessions |
| Path cites | Repo-relative `file:line` or `file:start-end` |
| CLI waste | Pattern where agent/CLI needs O(1)–O(10) scalars but convert returns full UI tables/HTML/trees |

Scope is **this tool set only**. No product code changes in this note. Shared convert/gzip/cache issues are referenced from `02-` and ranked again only when they dominate these tools.

---

## Registry placement

Source: [`plugin/xprof/profile_plugin/tools/registry.py`](plugin/xprof/profile_plugin/tools/registry.py)

| Concern | Membership / policy | Cite |
|---------|---------------------|------|
| **XPLANE_TOOLS** | All six tools listed | `registry.py:33-52` |
| **HLO_TOOLS** | `graph_viewer` (+ `memory_viewer`; covered in `03-`) | `registry.py:73` |
| **XPLANE_TOOLS_ALL_HOSTS_SUPPORTED** | `pod_viewer` only (of this set) | `registry.py:58-65` |
| **XPLANE_TOOLS_ALL_HOSTS_ONLY** | `pod_viewer`, `smart_suggestion` (+ `overview_page`) | `registry.py:68-70` |
| **DEFAULT_CACHE_TOOLS** | Not warmed (`overview_page`, `trace_viewer@` only) | `registry.py:55` |
| **Multi-host selection** | None of these six (`trace_viewer` / `trace_viewer@` only) | `registry.py:99` |
| **Explicit single-XSpace** | `utilization_viewer` rejects `XSpaceSize() != 1` | `xplane_to_tools_data.cc:370-375` |
| **Options builder** | `tools/options/graph.py` for graph_viewer | `graph.py:12-33` |
| **Python convert** | [`plugin/xprof/convert/raw_to_tool_data.py`](plugin/xprof/convert/raw_to_tool_data.py) | `raw_to_tool_data.py:169-252` |
| **C++ router** | [`xprof/convert/xplane_to_tools_data.cc`](xprof/convert/xplane_to_tools_data.cc) | `xplane_to_tools_data.cc:341-385` |
| **HLO subpath** | `hlo_to_tools_data.cc`, `xplane_to_hlo.cc`, `graph_viewer_processor.cc` | see §1 |
| **OpStats subpath** | `multi_xplanes_to_op_stats.cc` → pod_viewer / smart_suggestion | see §4, §6 |
| **names_only shortcut** | `perf_counters` + `names_only=1` bypasses convert | `counter_names.py:28-41`, `tool_data.py:64-66` |

### Dispatch layers

| Layer | Path | Role |
|-------|------|------|
| Python options | `tools/options/graph.py` | Packs `graph_viewer_options` (width default **3**) |
| Python convert | `raw_to_tool_data.py` | Tool branches; smart_suggestion `refresh_suggestion`; graph content-type |
| C++ router | `xplane_to_tools_data.cc` | `ConvertMultiXSpacesToToolData` switch |
| HLO processor | `graph_viewer_processor.cc` | REGISTER_PROFILE_PROCESSOR `"graph_viewer"` |
| HLO extract | `xplane_to_hlo.cc` | XSpace → `*.hlo_proto.pb` on disk |
| Shared pipeline | `02-convert-serve-cache-path.md` | gzip double-buffer, no concurrent convert cap |

### Family short path (all six)

```text
XPlane host files (session dir)
  ├─ HLO extract (tool_names / first graph open)
  │     ConvertMultiXSpaceToHloProto → N× <module>.hlo_proto.pb
  │     └─ graph_viewer (HLO_TOOLS): HloProto → HloModule IR → HTML/DOT/txt/pb
  ├─ OpStats cache ALL_HOSTS.op_stats_v2.pb
  │     ├─ pod_viewer (ALL_HOSTS_ONLY)
  │     └─ smart_suggestion rules (via ToolDataProviderImpl)
  ├─ inference_profile: parallel per-host InferenceStats → sample → DataTable JSON
  ├─ utilization_viewer: single host XSpace → counter metrics DataTable
  └─ perf_counters: sequential hosts → raw counter rows DataTable
```

**Observed family invariants:**

1. UI-scale outputs are the only convert modes for most tools (no server `summary` / `metrics_only`).  
2. CLI tools that need scalars still force full UI materialization (`get_utilization_viewer` CSV, `get_kpi_metrics` dual full tools, `get_top_hlo_ops` full op_profile tree).  
3. Multi-host peak is dominated by **vector of full OpStats** (`multi_xplanes_to_op_stats.cc:50-66`) for pod + smart_suggestion (+ overview family).  
4. Graph peak is dominated by **proto + HloModule IR coexistence** plus DOT→HTML wrap, not by gzip alone.

---

## 1. graph_viewer

### 1.1 End-to-end data path

```text
UI / CLI
  → graph_viewer_options {type, node_name, module_name|program_id,
                          graph_width, show_metadata, merge_fusion, format}
  → raw_to_tool_data.py:191-210
       options = graph_viewer_options; use_saved_result default True
       content_type: octet-stream for pb/pbtxt/json/txt; text/html for type=graph
  → GraphViewerProcessor::ProcessSession                     # graph_viewer_processor.cc:105-124
       GetHloProto:
         GetHloProtoByOptions(module_name | program_id)      # xplane_to_hlo.cc:167-180
         fallback GetHloProtoByNodeName (scan all *.hlo_proto.pb)
       ConvertHloProtoToGraphViewer(*hlo_proto, options)     # graph_viewer_processor.cc:78-103
            type=graph: ConvertHloProtoToGraph
                 ConvertHloProtoToModule → Plot
                 RenderNeighborhoodAround ALWAYS kDot first
                 WrapDotInFormat → HTML/url/dot
            type=custom_call: ConvertHloProtoToModule + GetCustomCallText
            type=adj_nodes: GetAdjacentNodes (full module + adjacency JSON)
            type=short_txt|long_txt: ConvertHloProtoToStringView → HloModule::ToString
            type=pb: SerializeAsString (no IR)
            type=pbtxt: TextFormat PrintToString (no IR)
            type=json: Unimplemented
       SetOutput(..., "application/json")                    # ALWAYS application/json
  → FE: iframe Graphviz HTML + separate op_profile fetch (limit 300) for hover
  → CLI: get_graph_viewer defaults short_txt; hlo_tools may request long_txt
```

**Registration:** `REGISTER_PROFILE_PROCESSOR("graph_viewer", GraphViewerProcessor)` at `graph_viewer_processor.cc:126`.

**Legacy dual path:** router also calls `ConvertHloProtoToToolData` for `"memory_viewer" || "graph_viewer"` at `xplane_to_tools_data.cc:349-350` → `hlo_to_tools_data.cc:76-120` (module_name/program_id required; **no** node-name fallback).

### 1.2 HLO extract (cold path shared with memory_viewer)

Entry: `ConvertMultiXSpaceToHloProto` at `xplane_to_hlo.cc:183-209`.

| Step | Behavior | Cite | Confidence |
|------|----------|------|------------|
| Cache check | If any `*.hlo_proto.pb` exists in session dir, return (skip re-extract) | `xplane_to_hlo.cc:192-202` | **observed** |
| Cold extract | For each XSpace: Arena GetXSpace → `hlo_proto_map.AddHloProtosFromXSpace` | `xplane_to_hlo.cc:56-60` | **observed** |
| Write | One file per module: `<module>.hlo_proto.pb` | `xplane_to_hlo.cc:76-87` | **observed** |
| Empty | Write `NO_MODULE.hlo_proto.pb` sentinel | `xplane_to_hlo.cc:64-72` | **observed** |
| tool_names trigger | Router runs extract on `"tool_names"` with TODO to create only when needed | `xplane_to_tools_data.cc:355-362` | **observed** |

**Severity H cold multi-host:** every host XSpace is fully loaded into Arena to harvest modules even if only one module is later viewed.

### 1.3 Module load paths

| API | Behavior | Peak risk | Cite | Confidence |
|-----|----------|-----------|------|------------|
| `GetHloProtoByModuleName` | Read single binary proto from disk | One module | `xplane_to_hlo.cc:95-102` | **observed** |
| `GetHloProtoByProgramId` | Scan filenames; fuzzy `StrContains(module_name, program_id)` | List dir + 1 load | `xplane_to_hlo.cc:135-163` | **observed** |
| `GetHloProtoByOptions` | module_name else program_id else error | — | `xplane_to_hlo.cc:167-180` | **observed** |
| `GetHloProtoByNodeName` | **For every** `*.hlo_proto.pb`: load full proto; walk all computations×instructions until name match | Worst-case **all modules** resident sequentially | `xplane_to_hlo.cc:105-132` | **observed** |
| Processor fallback | If options miss module, parse node_name and call node-name scan | Enables silent O(N_modules) | `graph_viewer_processor.cc:62-74` | **observed** |

### 1.4 Type-specific convert (materialization)

Params: `ParseGraphViewerParams` at `hlo_proto_to_graph_view.cc:295-355`.

| type | IR needed? | Output | Peak notes | Cite | Confidence |
|------|------------|--------|------------|------|------------|
| `graph` | **Yes** | DOT→HTML/url/dot | Proto + HloModule + DOT + HTML coexist mid-wrap | `hlo_proto_to_graph_view.cc:423-429`, `:489-498` | **observed** |
| `custom_call` | **Yes** | custom-call text | Full module for one instruction | `graph_viewer_processor.cc:87-96` | **observed** |
| `adj_nodes` | **Yes** | small JSON operand/user names | Full module for adjacency | `hlo_proto_to_graph_view.cc:371-420` | **observed** |
| `short_txt` | **Yes** | `HloPrintOptions::ShortParsable` | IR + string | `hlo_proto_to_graph_view.cc:464-473` | **observed** |
| `long_txt` | **Yes** | verbose + `print_large_constants` | Often larger than pb | `hlo_proto_to_graph_view.cc:471`, params.verbose | **observed** |
| `pb` | **No** | `SerializeAsString` | Proto only | `hlo_proto_to_graph_view.cc:459-460` | **observed** |
| `pbtxt` | **No** | TextFormat | Proto + text | `hlo_proto_to_graph_view.cc:443-451` | **observed** |
| `json` | — | Unimplemented | Error | `hlo_proto_to_graph_view.cc:439-441` | **observed** |

**Critical graph path detail:** `RenderGraphNeighborhoodAround` always renders **kDot first**, then `WrapDotInFormat` (`hlo_proto_to_graph_view.cc:493-498`). HTML wrap embeds the entire DOT inside a large HTML template (`graphviz_helper.h:39-129` `$DOT` substitution; `WrapDotInFormat` `:147-160`).

**Default width mismatch:**

| Layer | Default `graph_width` | Cite |
|-------|----------------------|------|
| C++ `kDefaultWidth` | **3** | `tool_options.h:40` |
| Options builder (Python) | **3** if missing/invalid | `graph.py:19-20` |
| FE | **1** | `graph_viewer.ts:86` |
| CLI `get_graph_viewer` | **1** (omit param if 1) | `get_graph_viewer_tool.py:20`, `:88-89` |

Width 3 neighborhoods can explode DOT/HTML size vs width 1.

### 1.5 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| HLO extract (cold) | All XSpaces → HloProtoMap → N× `.hlo_proto.pb` | Full multi-host XSpace walk once | **observed** `xplane_to_hlo.cc:52-90` |
| Load module | Full `xla::HloProto` in memory | Large modules tens–hundreds of MB | **observed** |
| IR materialization | `ConvertHloProtoToModule` → `unique_ptr<HloModule>` | **Proto + full IR coexist** (proto is `const&` retained by caller) | **observed** `hlo_proto_to_graph_view.cc:426-428` |
| type=graph | DOT string → HTML wrapper with embedded DOT | Peak ≈ IR + DOT + HTML | **observed** `graphviz_helper.h:39-129` |
| type=long_txt | Full module text (+ large constants if verbose) | Often larger than pb | **observed** |
| Node-name fallback | Scan every `*.hlo_proto.pb` until name match | Worst-case load **all** modules | **observed** `GetHloProtoByNodeName` |
| Processor content-type | `SetOutput(..., "application/json")` even for HTML body | Mismatch; Python layer may override for router path | **observed** `graph_viewer_processor.cc:122` vs `raw_to_tool_data.py:199-204` |
| FE | iframe DOM + full SVG; `opProfileLimit` default 300 | GRAPH_HTML_THRESHOLD = 1e6 **warn only** | **observed** `graph_viewer.ts:50`, `:92`, `:815-820` |
| CLI | full payload; client `max_lines` only on hlo_tools | Server still materializes full long_txt | **observed** `hlo_tools.py:136-158` |

### 1.6 Materialization waste

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| G1 | Proto → HloModule for almost every type except raw `pb`/`pbtxt` | Dominant convert peak | **H** | **observed** |
| G2 | Graph path always renders **kDot first**, then HTML/url wrap | Temporary double graph string | **H** | **observed** |
| G3 | `AddGraphMetadata` (PLATFORM_GOOGLE ME path): parse JSON graph → dump again | Extra full copy | **M** | **observed** `hlo_proto_to_graph_view.cc:100-112` |
| G4 | Processor sets `application/json` even for HTML graph body | Content-type mismatch; still full string | **L** | **observed** |
| G5 | FE loads **op_profile** (limit 300) on every graph session for hover | Second large convert path | **H** | **observed** `graph_viewer.ts:92`, `:141` |
| G6 | Default `graph_width`: FE=1, options builder=3, C++ default=3 | Width 3 neighborhood can explode HTML | **H** | **observed** |
| G7 | No durable cache of rendered neighborhood / txt per (module, node, width) | Reconvert on pan/reselect | **H** | **observed** (no cache key in processor) |
| G8 | `GetHloProtoByNodeName` has no instruction→module index | O(modules) full loads | **M** | **observed** |
| G9 | `long_txt` + `print_large_constants` | Multi-MB text for agents | **H** | **observed** |

### 1.7 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| GV-1 | Cache rendered graph/txt keyed by module×node×width×flags on disk | **H** | observed | M |
| GV-2 | Bound graph HTML size server-side (hard fail / auto-shrink width) — FE only warns at 1 MB | **H** | observed | S–M |
| GV-3 | Avoid full module IR for `pb`/`pbtxt` (already) and partial for adjacency-only | **M** | observed | M |
| GV-4 | Stream or zstd compress large HTML/txt; skip double gzip policy (shared 02) | **H** | hypothesized | M |
| GV-5 | Lazy op_profile: fetch metrics for visible nodes only, not limit=300 tree | **H** | observed | M |
| GV-6 | Index instruction→module at HLO extract time; kill full-module scan fallback | **M** | observed | M |
| GV-7 | Cap default graph_width=1 end-to-end (align FE / options / C++) | **M** | observed | S |
| GV-8 | Free HloProto after IR build (or build IR without retaining both) | **M** | hypothesized | M |
| GV-9 | CLI `summary` / neighborhood-only modes; never force `long_txt` without max_lines server-side | **M** | observed | S |
| GV-10 | ME path (`ConvertHloProtoToMeGraph`) avoid parse-dump metadata copy | **L** | observed | S |

**Top:** GV-2, GV-5, GV-1, GV-7, GV-9

### 1.8 Frontend notes

Source: [`frontend/app/components/graph_viewer/graph_viewer.ts`](frontend/app/components/graph_viewer/graph_viewer.ts)

| Behavior | Detail | Cite | Confidence |
|----------|--------|------|------------|
| Default width | `graphWidth = 1` | `:86` | **observed** |
| Op profile limit | `opProfileLimit = 300` | `:92` | **observed** |
| Query parse | `graph_width` from URL or 1 | `:174` | **observed** |
| Threshold | `GRAPH_HTML_THRESHOLD = 1000000` bytes | `:50` | **observed** |
| On load | Measures iframe HTML length; **warns** if over threshold | `:812-820` | **observed** |
| No hard reject | User can still receive multi-MB HTML | — | **observed** |
| Hover metrics | Injects runtime colors from op profile into iframe | `injectRuntimeData` path | **observed** |
| Module list | Separate request for modules / default graph options from op profile | `:141` | **observed** |

### 1.9 CLI notes

| Entry | Default | Waste | Cite |
|-------|---------|-------|------|
| `get_graph_viewer` | `output_type="short_txt"`, `graph_width=1` | Still full convert; returns entire body | `get_graph_viewer_tool.py:10-121` |
| `hlo_tools.get_hlo_module_content` | Fetches `long_txt` or `short_txt`; **client** truncates `max_lines=2000` | Server materializes full module text before truncate | `hlo_tools.py:92-158` |
| `hlo_tools.get_hlo_neighborhood` | Fetches full module text, then BFS client-side | Full long/short_txt for a local neighborhood | `hlo_tools.py:221-272` |
| Registration | `xprof_cli.py` maps `get_graph_viewer`, `get_hlo_module_content`, `get_hlo_neighborhood` | — | `xprof_cli.py:50-51,67` |

**CLI waste pattern (classic):** agent needs neighborhood of 1 instruction → server builds full HloModule IR + full module text → client truncates. Severity **H**, confidence **observed**.

### 1.10 Graph viewer residual risks if optimized

| Opt | Correctness | UX |
|-----|-------------|-----|
| Graph width hard cap | Miss distant nodes | Must re-request with larger width |
| Server max_lines on long_txt | Incomplete dumps | Agents must page |
| Drop node-name fallback without index | Op links without module break | Need extract-time index first |
| Lazy op_profile | Missing hover metrics until fetch | Acceptable progressive enhancement |

---

## 2. inference_profile

### 2.1 End-to-end data path

```text
Session XPlanes (all hosts in selection)
  → InferenceStatsProcessor::ProcessSession                  # inference_stats_processor.cc:37-55
  → ConvertMultiXSpaceToInferenceStats                       # multi_xspace_to_inference_stats.cc:117-167
       thread_pool_size = min(MaxParallelism(), xspaces_size)  # :125-127
       for each host i (parallel):
         Arena GetXSpace(i)
         PreprocessSingleHostXSpace(step_grouping=true, derived_timeline=false)
         GetNonOverlappedStepEvents (GPU path)
         GenerateInferenceStats → per_host_inference_stats[i]
       sequential CombineInferenceStatsResult per host
       RegroupInferenceStatsByModel
       SampleInferenceStats → sampled_* (percentiles)
  → InferenceStatsToDataTableJson (metadata + per-model tables)
  → FE: array of Google DataTables; holds all models at once
```

**Registration:** `REGISTER_PROFILE_PROCESSOR("inference_profile", InferenceStatsProcessor)` at `inference_stats_processor.cc:57`.

**Router:** `xplane_to_tools_data.cc:366-367` / helper `:297-307`.

**Python:** `raw_to_tool_data.py:236-239` (no special options beyond params).

**Not** in ALL_HOSTS sets (`registry.py:58-70`) — host policy is whatever HostSelector passes; multi-host still converts all selected XSpaces in parallel.

### 2.2 Sampling constants (wire lean, intermediate fat)

Source: `inference_stats_sampler.cc:181-190`

| Constant | Value | Meaning |
|----------|-------|---------|
| `kWantedPercentiles` | 50, 75, 90, 99, 99.9, 99.99 (±error bands) | 6 bands |
| `kMaxNumDataSelectedPerPercentile` | **10** | Cap samples per band |
| Theoretical max samples / model | ≤ 6×10 = 60 requests + 60 batches | Plus aggregates |

Sampling happens **after** full request/batch lists are built and combined (`multi_xspace_to_inference_stats.cc:162-165`). Full unsampled details remain in `InferenceStats` while sampling runs and while JSON is built from sampled views — intermediate residency is the problem, not the wire.

### 2.3 Parallelism

| Property | Value | Cite | Confidence |
|----------|-------|------|------------|
| Thread pool | `min(tsl::port::MaxParallelism(), xspaces_size)` | `multi_xspace_to_inference_stats.cc:125-127` | **observed** |
| Per host | Full XSpace + preprocess + GenerateInferenceStats | `:130-150` | **observed** |
| Peak model | **Σ host peaks** if all threads run convert simultaneously | structural | **observed** structure / **hypothesized** magnitude |
| Join | `executor->JoinAll()` then sequential combine | `:152-160` | **observed** |

No semaphore bounds convert concurrency against other UI tools (shared with 02-doc concurrent convert concern).

### 2.4 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| Per-host convert | Full XSpace + all `RequestDetail` / `BatchDetail` | Parallel hosts → **Σ host peaks** | **observed** |
| Combined proto | Full unsampled request lists per model | Dominates before sample | **observed** |
| Sampling | 6 percentile bands × ≤10 samples | Wire much smaller than intermediate | **observed** |
| JSON | Joined DataTable JSON array | Sampled only + aggregates + tensor patterns | **observed** `InferenceStatsToDataTableJson` |
| FE | All model tables retained | Multi-model multiplies | **observed** `inference_profile.ts:45-49`, `:116-156` |

### 2.5 Materialization waste

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| I1 | Retain all request_details until after sample | Peak ≫ wire | **H** | **observed** |
| I2 | Unbounded host parallelism | Multi-host RSS spike | **H** | **observed** |
| I3 | No disk cache of InferenceStats / sampled JSON | Recompute on column change | **M** | **observed** (no cache in processor) |
| I4 | FE holds all model DataTables | Client heap | **M** | **observed** |
| I5 | Tensor pattern tables with HTML percentile text | Extra strings | **L** | **observed** `GenerateTensorPatternPercentileText` |
| I6 | Overview also can re-run InferenceStats (see 04 OV-3) | Cross-tool double convert | **H** | **observed** (overview path) |

### 2.6 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| IP-1 | Online / streaming sample: never retain all request_details | **H** | observed | L |
| IP-2 | Bound host parallelism (semaphore) instead of `min(MaxParallelism, N)` | **H** | observed | S |
| IP-3 | Drop full request_details after sample (clear before JSON) | **H** | observed | S |
| IP-4 | Per-model / percentile-column incremental re-fetch (today full recompute on column change via options) | **M** | hypothesized | M |
| IP-5 | FE: load selected model tables only | **M** | observed | M |
| IP-6 | Disk cache of combined InferenceStats / sampled JSON | **M** | hypothesized | M |
| IP-7 | Multi-host correctness path (FE warns b/474172782) — avoid loading N hosts when wrong | **M** | hypothesized | M |
| IP-8 | Share InferenceStats with overview_page to kill second sweep | **H** | observed gap | M |

**Top:** IP-1 / IP-3, IP-2, IP-5, IP-8

Wire is already relatively lean (sampled). **Peak is intermediate residency**, not JSON size.

### 2.7 Frontend notes

Source: `frontend/app/components/inference_profile/inference_profile.ts`

- Holds `allRequestTables`, `allBatchTables`, `allTensorPatternTables` arrays (`:45-49`).  
- `parseData` builds a `google.visualization.DataTable` per model for each table type (`:116-156`).  
- Column changes re-request with `request_column` / `batch_column` options → full C++ recompute (processor reads options at `:44-49`).

### 2.8 CLI notes

No dedicated `get_inference_profile` thin CLI tool in `plugin/xprof/cli/tools/`. Agents that need inference KPIs typically go through overview / KPI paths (see §7). If raw tool is fetched via generic client, full DataTable JSON is returned — severity **M–H** depending on model count, confidence **hypothesized** for agent traffic.

---

## 3. utilization_viewer

### 3.1 End-to-end data path

```text
Exactly 1 XSpace (enforced)
  → xplane_to_tools_data.cc:370-385
       if XSpaceSize() != 1 → InvalidArgumentError
       Arena GetXSpace(0)
       PreprocessSingleHostXSpace(step_grouping=true, derived_timeline=true)
       ConvertXSpaceToUtilizationViewer
  → xplane_to_utilization_viewer.cc:143-312
       For each TPU plane:
         per sample line → counters_map → TpuCounterUtil
         ComputeTpuGenericTcUnitUtilization / Bandwidth / ICI / SC
         One DataTable row per (sample × metric)
       Columns: host, device, sample, node, name, achieved, peak, unit
       data_table.ToJson()
  → FE: full table → FilterDataProcessor copies per core for unit/bandwidth/HBM charts
  → CLI get_utilization_viewer: fetch tqx=csv → parse ALL rows → ~10 scalar % metrics
```

**Python:** `raw_to_tool_data.py:244-247`.

**Device support:** only `"TPU v7x"` and `"TPU v6 Lite"` (`xplane_to_utilization_viewer.cc:112-118`). Other devices skip planes → empty-ish table.

### 3.2 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| XSpace + preprocess | Full host plane + **derived_timeline=true** | Heavier than memory_profile (which uses derived_timeline=false) | **observed** `xplane_to_tools_data.cc:379-380` vs memory path |
| Counter expand | One DataTable row per (sample × metric) | Sample count can be large | **observed** `:297-308` |
| JSON | Full flat table | UI-scale | **observed** `:312` |
| FE charts | Multiple `FilterDataProcessor` per core | Multiplier per core × chart type | **observed** `utilization_viewer.ts:155-208` |
| CLI | Full CSV then discard most rows | **UI-scale payload for scalars** | **observed** `get_utilization_viewer_tool.py:254-309` |

### 3.3 Materialization waste

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| U1 | Per-sample rows with no server aggregate mode | Large tables for long profiles | **H** | **observed** |
| U2 | CLI fetches full CSV for ~11 percent metrics | Classic scalar waste | **H** | **observed** |
| U3 | derived_timeline=true may be overkill for counters | Extra preprocess cost | **M** | **hypothesized** |
| U4 | FE FilterDataProcessor per core × chart | Client copies | **M** | **observed** |
| U5 | No host/device/node server filter | Client filters after full convert | **M** | **observed** (CLI filters client-side `:145-178`) |
| U6 | Hardcoded peak bandwidth / frequency tables | Correctness not memory | **L** | **observed** |

### 3.4 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| UV-1 | CLI / `summary` convert mode: aggregate % only (no per-sample rows) | **H** | observed | M |
| UV-2 | Server-side host/device/node + sample downsample filters | **H** | observed | M |
| UV-3 | Skip derived_timeline if only counters needed (if safe) | **M** | hypothesized | M |
| UV-4 | FE: one DataView; no per-chart full copies | **M** | hypothesized | M |
| UV-5 | Cache utilization DataTable on disk per host | **L** | hypothesized | S |
| UV-6 | Mean/max aggregate per metric across samples as default UI mode | **M** | hypothesized | M |

**Top:** UV-1, UV-2

Not ALL_HOSTS; severity mainly sample cardinality × CLI full fetch.

### 3.5 CLI waste analysis (full fetch for scalars) — detailed

Source: [`plugin/xprof/cli/tools/get_utilization_viewer_tool.py`](plugin/xprof/cli/tools/get_utilization_viewer_tool.py)

| Step | What happens | Lines |
|------|--------------|-------|
| Cache decorator | `@decorators.cached(expire=86_400)` — 24h client cache | `:254` |
| Fetch | `client.fetch(tool_name="utilization_viewer.json", tqx="out:csv")` | `:271-275` |
| Decode | Entire CSV to string | `:290-305` |
| Parse | `csv.DictReader` all rows into list of dicts | `:129-137` |
| Filter | Client host/device/node filters on full list | `:145-178` |
| Reduce | Compute ~11 scalars: HBM, ICI R/W, vector ALU, scalar, Vmem/Cmem, XLU, MXU, idleness | `:191-229` |
| Output | JSON of percentages (+ optional `metrics` map) | `:217-237` |

**Needed by agent:** ~11 floats.  
**Paid:** full single-host convert (XSpace + derived timeline + per-sample metrics) + full CSV wire + full client parse.  
**Severity: H** | **Confidence: observed**

This is the same pattern as GMP-1 / peak-allocations waste in `03-memory-tools-family.md`.

### 3.6 Frontend notes

Source: `frontend/app/components/utilization_viewer/utilization_viewer.ts`

- Loads full tool data once (`:122-129`).  
- Builds per-core unit, bandwidth, and HBM `FilterDataProcessor` instances (`:155-208`).  
- Holds master `dataTable` plus filtered processors — client-side only paging/filter.

---

## 4. smart_suggestion

### 4.1 End-to-end data path

```text
ALL_HOSTS_ONLY host policy (registry)
  → raw_to_tool_data.py:248-252
       options['refresh_suggestion'] = params.get('refresh_suggestion', False)
  → SmartSuggestionProcessor::ProcessSession                 # smart_suggestion_processor.cc:81-129
       if !refresh && cache hit:
         ReadBinaryProto SMART_SUGGESTION / ALL_HOSTS
         → ALL_HOSTS.smart_suggestion.pb                     # repository.cc:55
       else:
         ToolDataProviderImpl(session_snapshot)
           GetOpStats → ConvertMultiXSpaceToCombinedOpStatsWithCache
           GetOverviewPage / InputPipeline / OpProfile from OpStats
           GetMemoryProfile via full tool convert + JSON→proto parse
           GetEventTimeFractionAnalyzer via tool convert (binary parse)
         SmartSuggestionEngine + RegisterAllRulesFor3P
         WriteBinaryProto SMART_SUGGESTION
  → MessageToJsonString(always_print_fields_with_no_presence=true)
  → FE HTML cards (suggestion_text may contain HTML)
  → CLI: get_smart_suggestions → HTTP, strip_html=True, cached 86400s
```

**Registration:** `smart_suggestion_processor_registration` at `smart_suggestion_processor.cc:62-63`.

**Map/Reduce:** Unimplemented (`:67-78`) — only ProcessSession path.

### 4.2 ToolDataProviderImpl cost graph

Source: `xprof/convert/smart_suggestion/tool_data_provider_impl.h`

| Provider method | How data is obtained | Peak impact | Cite | Confidence |
|-----------------|---------------------|-------------|------|------------|
| `GetOpStats` | `ConvertMultiXSpaceToCombinedOpStatsWithCache` | N host OpStats vector on miss | `:137-144` | **observed** |
| `GetOverviewPage` | `ConvertOpStatsToOverviewPage` from cached OpStats | Secondary; needs OpStats | `:57-67` | **observed** |
| `GetInputPipelineAnalysisResult` | `ConvertOpStatsToInputPipelineAnalysis` | Secondary | `:69-80` | **observed** |
| `GetOpProfile` | `ConvertOpStatsToOpProfile` full tree | Secondary from OpStats | `:112-122` | **observed** |
| `GetMemoryProfile` | **Full** `ConvertMultiXSpacesToToolDataWithProfileProcessor("memory_profile")` then **JSON → proto** | Double materialization; single-host memory tool | `:125-132`, `:148-165` | **observed** |
| `GetEventTimeFractionAnalyzerResult` | Full tool convert; **ParseFromString** (binary path special-cased) | Extra XSpace walk | `:84-95`, `:154-158` | **observed** |

**JSON round-trip tax:** for tools other than event_time_fraction_analyzer, `GetToolDataHelper` takes JSON string and `JsonStringToMessage` (`:161-165`). Memory profile pays: XSpace→MemoryProfile→JSON→MemoryProfile.

### 4.3 Report size vs convert peak

Proto: `plugin/xprof/protobuf/smart_suggestion.proto`

```text
SmartSuggestion { rule_name, suggestion_text }
SmartSuggestionReport { repeated SmartSuggestion suggestions }
```

| Property | Value | Confidence |
|----------|-------|------------|
| Wire report | Few strings | **observed** |
| JSON opts | `always_print_fields_with_no_presence = true` | **observed** `smart_suggestion_processor.cc:116-119` |
| Disk cache | `.smart_suggestion.pb` under ALL_HOSTS | **observed** `repository.cc:55` |
| Refresh | `refresh_suggestion` true/1 bypasses cache | **observed** `:83-97` |

**Asymmetry:** output is tiny; cold path can be one of the largest multi-tool convert stacks in the product.

### 4.4 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| OpStats combine | `vector<OpStats>` for **all** hosts then combine | Explicit TODO to stream-combine | **observed** `multi_xplanes_to_op_stats.cc:50-66` |
| Memory profile side convert | Full memory_profile tool + JSON string + parse back | Double materialization | **observed** |
| Event time analyzer | Another full convert | Extra XSpace walks | **observed** |
| Report | Few `SmartSuggestion` strings | Tiny | **observed** |
| JSON opts | always_print | Minor | **observed** |
| Warm cache | Read small pb only | Cold→warm cliff | **observed** |

### 4.5 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| SS-1 | Stream OpStats combine (shared with pod_viewer / overview) — drop all_op_stats vector | **H** | observed | L |
| SS-2 | Prefer binary proto paths in ToolDataProvider; **no JSON round-trip** | **H** | observed | M |
| SS-3 | Rule-specific minimal signals (duty cycle, % bound) without full MemoryProfile / OpProfile trees | **H** | hypothesized | L |
| SS-4 | Single-flight + durable report cache (already partial); share warm OpStats with UI | **M** | observed | S–M |
| SS-5 | Lazy rule evaluation: stop after K suggestions; skip unused providers | **M** | hypothesized | M |
| SS-6 | FE already small; strip HTML server-side for CLI (done via `strip_html`) | **L** | observed | — |
| SS-7 | Default never refresh on CLI; document refresh cost | **M** | observed | S |

**Top:** SS-1 (shared), SS-2, SS-3

First open of smart_suggestion can **amplify** OpStats + memory_profile peaks on ALL_HOSTS.

### 4.6 CLI notes

Source: `plugin/xprof/cli/tools/get_smart_suggestions_tool.py`

| Property | Value | Cite |
|----------|-------|------|
| Cache | `@decorators.cached(expire=86400)` | `:11` |
| Fetch | `smart_suggestion_tools.fetch_smart_suggestions(session_id, strip_html=True)` | `:26-28` |
| Output | JSON list of suggestions | `:29` |

**CLI waste:** wire is small (good). **Cold server peak** is still multi-tool convert — agent first call can OOM or thrash even though response is KB-scale. Severity **H** cold / **L** warm; confidence **observed**.

### 4.7 Frontend notes

Source: `frontend/app/components/smart_suggestion/smart_suggestion_view.ts` — renders suggestion cards from report JSON. Payload size not the issue; first-open latency/RSS is.

---

## 5. perf_counters

### 5.1 End-to-end data path

```text
For each host (sequential):                              # xplane_to_perf_counters.cc:118-125
  GetXSpace(arena) → ConvertXSpaceToPerfCounters
    TPU/GPU planes → every event with kCounterValue → DataTable row
      (Host, Chip, Kernel, Sample, Counter, Value, Description, Set)
→ data_table.ToJson()
Special: names_only=1 handled OUTSIDE convert
  services/counter_names.py → try_counter_names_only
  tool_data.py:64-66 short-circuit
FE: full google.visualization.DataTable; pageSize 30–200 (UI page only)
```

**Router:** `xplane_to_tools_data.cc:353-354`.  
**Python:** `raw_to_tool_data.py:240-243`.  
**Not** in ALL_HOSTS sets — multi-host means sequential full table union.

### 5.2 Row expansion detail

Source: `xplane_to_perf_counters.cc:33-98`

| Field | Source | Notes |
|-------|--------|-------|
| Host | `session_snapshot.GetHostname(i)` | String per row |
| Chip | `kGlobalChipId` or `kDeviceId` | Skip plane if missing |
| Kernel | `line.Name()` | |
| Sample | `line.Id()` | |
| Counter | `AsciiStrToLower(event.Name())` | |
| Value | Hex cell from `kCounterValue` | |
| Description | `kPerformanceCounterDescription` | **Repeated string** |
| Set | `kPerformanceCounterSets` | **Repeated string** |

Comment at `:76-79` notes zeros are included (frontend filters) — no server-side zero drop.

### 5.3 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| Host loop | One XSpace at a time (arena) | Sequential peak ≈ max host (good vs inference) | **observed** |
| Rows | Unaggregated event-level | O(samples × counters); Description/Set strings repeated | **observed** |
| JSON | Full table | Large multi-host | **observed** `:127` |
| FE | Full table in JS heap | Client filter + pageSize only | **observed** `perf_counters.ts:34`, `:83-86` |
| names_only | Static name list, no XSpace | Cheap path | **observed** `counter_names.py` |

### 5.4 Materialization waste

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| P1 | Unaggregated samples for default UI | Huge tables | **H** | **observed** |
| P2 | Description/Set repeated every row | String bloat | **M** | **observed** |
| P3 | No host/chip/counter convert filters | Always full | **M** | **observed** |
| P4 | Multi-host sequential still builds one giant table | Wire = sum | **M** | **observed** |
| P5 | FE pages but holds full DataTable | Client heap | **M** | **observed** |
| P6 | names_only exists but default UI path unused | Missed win for pickers | **L** | **observed** |

### 5.5 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| PC-1 | Aggregate / downsample samples server-side (mean/max per counter) for UI default | **H** | observed | M |
| PC-2 | Intern Description/Set strings; optional omit Description on wire | **M** | observed | S |
| PC-3 | Host/chip/counter filters as convert options | **M** | observed | S |
| PC-4 | Columnar / protobuf wire instead of DataTable JSON | **M** | hypothesized | L |
| PC-5 | FE virtual scroll + progressive fetch | **M** | hypothesized | M |
| PC-6 | Multi-host: don't default to all hosts if UI only shows one | **L** | hypothesized | S |
| PC-7 | Drop zero values server-side (optional) | **L** | hypothesized | S |

**Top:** PC-1, PC-2, PC-3

### 5.6 CLI notes

No dedicated `get_perf_counters` thin tool in `cli/tools/`. Generic fetch gets full JSON. `names_only=1` is the only lean path and requires `device_type` (`counter_names.py:28-41`). Agents that need a few counter values still risk full table — severity **M–H**, confidence **hypothesized** for agent volume / **observed** for convert shape.

---

## 6. pod_viewer

### 6.1 End-to-end data path

```text
ALL_HOSTS_ONLY (registry.py:68-70)
  → raw_to_tool_data.py:169-172
  → PodViewerProcessor::ProcessSession                       # pod_viewer_processor.cc:37-56
       ConvertMultiXSpaceToCombinedOpStatsWithCache
            miss: for each host XSpace → OpStats; hold all;
                  CombineAllOpStats; write ALL_HOSTS.op_stats_v2.pb
       ConvertOpStatsToPodViewer
            PodStatsRecord per (step × core) + step_breakdown_events + diagnostics
       MessageToJsonString(always_print=true)
  → FE: full PodViewerDatabase → topology + stack bars
```

**Also:** `ProcessCombinedOpStats` path at `pod_viewer_processor.cc:58-70` for processors that already hold OpStats.

**Router helper:** `ConvertMultiXSpacesToPodViewer` at `xplane_to_tools_data.cc:217-221`.

### 6.2 OpStats combine hotspot (shared SS-1 / X-1)

Source: `multi_xplanes_to_op_stats.cc:42-114`

```text
// TODO(profiler): Change the combiner to convert and combine one OpStats at
// a time, to reduce peak memory usage.
std::vector<OpStats> all_op_stats;
all_op_stats.reserve(session_snapshot.XSpaceSize());
for each host:
  Arena GetXSpace → Preprocess(step_grouping, derived_timeline=true)
  ConvertXSpaceToOpStats → push_back full OpStats

StepIntersection with max_steps = numeric_limits<uint32_t>::max()  // UNBOUNDED
CombineAllOpStats(...)
```

| Property | Value | Cite | Confidence |
|----------|-------|------|------------|
| Cache file | `OP_STATS` → `.op_stats_v2.pb` ALL_HOSTS | `repository.cc:54`, cache path `:125-151` | **observed** |
| Cache options | Always metrics + step + kernel DBs | `:121-124` | **observed** |
| Step cap | `std::numeric_limits<uint32_t>::max()` | `:91-93` | **observed** |
| TODO | Stream-combine to reduce peak | `:50-52` | **observed** |

### 6.3 PodViewerDatabase construction

Source: `op_stats_to_pod_viewer.cc:51-61`

| Field | Source | Growth |
|-------|--------|--------|
| `device_type` | run_environment | scalar |
| `step_breakdown_events` | from PodStatsDatabase | small |
| `pod_stats_sequence` | per step × per core `PodStatsRecord` | **steps × cores** |
| `diagnostics` | `PopulateStepDiagnostics` | warnings |

Nested loop at `:36-45` moves one `PodStatsRecord` per (step, core).

### 6.4 Stage peak table

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| OpStats intermediate | N host OpStats + combined | **Same SS-1 hotspot**; TODO in code | **observed** |
| PodViewerDatabase | steps × cores maps | Grows with duration × topology | **observed** |
| JSON always_print | Empty fields expanded | Wire bloat | **observed** `pod_viewer_processor.cc:45-48` |
| FE | Full JSON retained | Topology graph + charts | **observed** `pod_viewer/` |
| Cache hit | Read combined OpStats only | Still full OpStats → full Pod JSON | **observed** |

### 6.5 Opportunities

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| PV-1 | Stream OpStats combine (shared SS-1) | **H** | observed | L |
| PV-2 | Cap steps in merge (`std::numeric_limits::max()` today) | **H** | observed | S |
| PV-3 | Disable always_print; slim bottleneck-only view mode | **M** | observed | S |
| PV-4 | Pod-only intermediate (skip full OpStats fields unused by pod) | **H** | hypothesized | L |
| PV-5 | FE: step window / virtualize topology | **M** | hypothesized | M |
| PV-6 | OpStatsOptions: skip kernel_stats_db if pod doesn't need it | **M** | hypothesized | M |

**Top:** PV-1, PV-2, PV-4

### 6.6 Frontend notes

- `pod_viewer.ts` loads full database.  
- `pod_viewer_common.ts:249` `parseData` retains structure for topology_graph and stack_bar_chart.  
- Charts convert slices to google DataTables client-side — full pod sequence still resident.

### 6.7 CLI notes

No dedicated pod_viewer CLI scalar tool. Full JSON is the only mode. Agents needing "worst core" or "step breakdown summary" force full PodViewerDatabase — severity **M–H**, confidence **hypothesized** for traffic / **observed** for convert shape.

---

## 7. Related CLI tools (cross-tool waste)

### 7.1 Summary table

| CLI | Convert dependency | What agent needs | What is fetched | Memory issue | Severity | Confidence |
|-----|-------------------|------------------|-----------------|--------------|----------|------------|
| **get_graph_viewer** | Full `graph_viewer` | short_txt / neighborhood | Entire HTML/txt body | No server bound | **H** | observed |
| **get_hlo_module_content** | `graph_viewer` short/long_txt | text slice | Full module text server-side; client max_lines | Full IR + text | **H** | observed |
| **get_hlo_neighborhood** | full module text then client BFS | local neighborhood | Full long/short_txt | Full IR | **H** | observed |
| **get_utilization_viewer** | Full utilization CSV | ~11 % metrics | Full per-sample CSV | **Classic UI-scale force** | **H** | observed |
| **get_smart_suggestions** | smart_suggestion | few strings | Small JSON; cold = multi-tool | Peak on cold cache | **H** cold / **L** warm | observed |
| **get_kpi_metrics** | `get_overview` + `get_memory_profile` | ~6 scalars | Two full tool converts | Dual UI-scale | **H** | observed |
| **get_top_hlo_ops** | Full `hlo_op_profile` tree | top-K=10 leaves | Full tree → flatten → heapq | Full tree for K=10 | **H** | observed |

### 7.2 get_kpi_metrics — dual full convert

Source: `plugin/xprof/cli/tools/get_kpi_metrics_tool.py:16-93`

```text
get_overview(session_id)          → full overview JSON
get_memory_profile(session_id)    → full memory_profile JSON
json.loads both
extract:
  steptime_ms_average
  device_duty_cycle_percent
  mxu_utilization_percent
  flop_rate_utilization_relative_to_roofline
  peak_memory_usage_gib
  device_type, device_core_count
```

| Property | Detail | Confidence |
|----------|--------|------------|
| Needed | ~6 scalars + 2 env fields | **observed** |
| Paid | Full overview (OpStats + optional inference) + full memory_profile (all snapshots) | **observed** |
| Error handling | Memory failure → peak_hbm "N/A" still returns other KPIs | **observed** `:60-68` |
| Severity | **H** | **observed** |

Aligns with FAM-1 / overview summary modes in `03`/`04`.

### 7.3 get_top_hlo_ops — full tree for top-K

Source: `plugin/xprof/cli/tools/get_top_hlo_ops_tool.py:21-154`

| Step | Detail | Cite |
|------|--------|------|
| Cache | `@decorators.cached(expire=86400)` | `:21` |
| Fetch | `hlo_op_profile.json` full | `:34-38` |
| Parse | JSON or binary `op_profile_pb2.Profile` | `:49-86` |
| Traverse | Recursive leaf walk of `by_category` or `by_program` | `:88-124` |
| Materialize | `flat_ops = list(ops_iterable)` **all leaves** | `:124` |
| Top-K | `heapq.nlargest(limit, ...)` default limit=10 × 3 sorts | `:135-139` |

**Needed:** 30 ops max (10×3).  
**Paid:** full op_profile convert (OpStats + profile tree JSON/proto) + full flatten in Python.  
**Severity: H** | **Confidence: observed**

Note: server op_profile convert already uses `op_profile_limit=100` in one path (`xplane_to_tools_data.cc:249`) but CLI still receives a full tree structure and flattens all leaves client-side.

### 7.4 hlo_tools long_txt path

Source: `plugin/xprof/cli/internal/oss/hlo_tools.py:136-158`

| Step | Server | Client |
|------|--------|--------|
| Fetch | `type: long_txt` or `short_txt` for whole module | — |
| IR | Full HloModule + ToString | — |
| Truncate | None | `max_lines` default 2000 |
| Neighborhood | Same full text | BFS on parsed text |

**Severity H:** server work independent of `max_lines`.

### 7.5 CLI opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| CLI-GV-1 | Server `summary` / `top_k` / `neighborhood` flags for graph + op profile | **H** | observed |
| CLI-UV-1 | utilization `metrics_only` convert (UV-1) | **H** | observed |
| CLI-KPI-1 | overview + memory summary modes (align FAM-1 from memory family) | **H** | observed |
| CLI-TOP-1 | op_profile convert option `top_ops_limit` without full tree materialization | **H** | hypothesized |
| CLI-SS-1 | Rely on SMART_SUGGESTION disk cache; never refresh unless asked | **M** | observed |
| CLI-HLO-1 | Server-side max_lines / streaming text for long_txt | **H** | observed |
| CLI-HLO-2 | Native adj_nodes / neighborhood convert instead of full text BFS | **H** | observed |

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
| X-8 | OpStatsOptions per consumer (skip kernel DB when unused) | pod, smart_suggestion, overview family | **M** | hypothesized |
| X-9 | Share InferenceStats between overview and inference_profile | overview + inference | **H** | observed gap |
| X-10 | Binary tool-data provider APIs (no JSON) | smart_suggestion ToolDataProvider | **H** | observed |

### Shared preprocess flags (quick compare)

| Tool | step_grouping | derived_timeline | Cite |
|------|---------------|------------------|------|
| utilization_viewer | true | **true** | `xplane_to_tools_data.cc:379-380` |
| inference per-host | true | **false** | `multi_xspace_to_inference_stats.cc:142-143` |
| OpStats multi-host | true | **true** | `multi_xplanes_to_op_stats.cc:62-63` |
| memory_profile (03) | true | false | (sibling doc) |

Derived timeline doubles timeline materialization on util + OpStats paths — severity **M**, confidence **hypothesized** as pure waste without per-tool measurement.

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
| 12 | CLI-HLO-1 / GV-9 | Server max_lines / neighborhood modes | **H** | observed | M |
| 13 | X-5 | always_print off | **M** | observed | S |
| 14 | SS-3 | Minimal signals for rules | **H** | hypothesized | L |
| 15 | PC-2 / PC-3 | String intern + filters | **M** | observed | S |

### Top 5 wins

1. **Stream OpStats combine** — largest shared multi-host peak (pod + smart_suggestion + overview family). Explicit TODO at `multi_xplanes_to_op_stats.cc:50-52`.  
2. **Graph size bounds** — prevent multi-MB HTML/IR spikes; align width defaults (FE 1 vs C++/Python 3).  
3. **CLI summary modes** — utilization, KPI, top HLO ops, long_txt must not pull UI-scale JSON/text.  
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
| Cap pod steps | Incomplete long runs | Show diagnostics; allow opt-in full |
| Skip derived_timeline on util | Wrong counters if timeline-derived | Validate against golden tests |
| Lazy rule providers | Missing suggestions if short-circuit wrong | Order rules by cost |

---

## 11. Severity × confidence summary by tool

| Tool | Peak severity | Dominant stage | Wire severity | CLI waste | Confidence |
|------|---------------|----------------|---------------|-----------|------------|
| graph_viewer | **H** | HloModule IR + DOT/HTML; FE op_profile | **H** (HTML/long_txt) | **H** full text for slice | observed |
| inference_profile | **H** convert / **M** wire | Parallel XSpace + full request_details | **M** (sampled) | **M** (no thin CLI) | observed |
| utilization_viewer | **M** UI / **H** CLI | Per-sample rows; CLI full CSV | **M–H** | **H** 11 scalars | observed |
| smart_suggestion | **H** cold / **L** warm | Nested OpStats + side converts | **L** | **L** wire / **H** cold peak | observed |
| perf_counters | **M–H** | Unaggregated multi-host rows | **H** large | **M–H** if generic fetch | observed |
| pod_viewer | **H** multi-host | OpStats vector + step×core JSON | **H** | **M–H** no summary | observed |

---

## 12. Per-tool opportunity index (quick lookup)

### graph_viewer
GV-1 cache render · GV-2 HTML bound · GV-3 less IR · GV-4 compress · GV-5 lazy op_profile · GV-6 instruction index · GV-7 width=1 · GV-8 free proto · GV-9 CLI neighborhood · GV-10 ME metadata

### inference_profile
IP-1 stream sample · IP-2 bound threads · IP-3 clear details · IP-4 incremental columns · IP-5 FE model filter · IP-6 disk cache · IP-7 multi-host policy · IP-8 share with overview

### utilization_viewer
UV-1 metrics_only · UV-2 server filters · UV-3 skip derived_timeline · UV-4 FE DataView · UV-5 disk cache · UV-6 aggregate UI default

### smart_suggestion
SS-1 stream OpStats · SS-2 binary providers · SS-3 minimal signals · SS-4 single-flight · SS-5 lazy rules · SS-6 strip HTML · SS-7 no default refresh

### perf_counters
PC-1 aggregate · PC-2 intern strings · PC-3 filters · PC-4 columnar · PC-5 progressive FE · PC-6 host default · PC-7 drop zeros

### pod_viewer
PV-1 stream OpStats · PV-2 step cap · PV-3 always_print off · PV-4 pod-only intermediate · PV-5 FE window · PV-6 skip kernel DB

### CLI
CLI-GV-1 · CLI-UV-1 · CLI-KPI-1 · CLI-TOP-1 · CLI-SS-1 · CLI-HLO-1 · CLI-HLO-2

### Cross-cutting
X-1 … X-10 (see §8)

---

## 13. Real repo paths cited

### Registry / plugin / Python convert
- `plugin/xprof/profile_plugin/tools/registry.py:33-99`
- `plugin/xprof/profile_plugin/tools/options/graph.py:12-33`
- `plugin/xprof/convert/raw_to_tool_data.py:169-252`
- `plugin/xprof/profile_plugin/services/counter_names.py:1-41`
- `plugin/xprof/profile_plugin/services/tool_data.py:64-66`

### C++ convert — graph / HLO
- `xprof/convert/xplane_to_tools_data.cc:217-385`
- `xprof/convert/hlo_to_tools_data.cc:52-120`
- `xprof/convert/xplane_to_hlo.cc:52-209`
- `xprof/convert/graph_viewer_processor.cc:62-126`
- `xprof/convert/hlo_proto_to_graph_view.cc:100-499`
- `xprof/convert/hlo_proto_to_graph_view.h:35-90`
- `xprof/convert/graphviz_helper.h:39-160`
- `xprof/convert/tool_options.h:30-42`

### C++ convert — inference
- `xprof/convert/inference_stats_processor.cc:37-57`
- `xprof/convert/multi_xspace_to_inference_stats.cc:117-167`
- `xprof/convert/inference_stats_sampler.cc:181-190`
- `xprof/convert/inference_stats.cc` (Generate / BuildRequestDetails)

### C++ convert — utilization / perf
- `xprof/convert/xplane_to_utilization_viewer.cc:143-312`
- `xprof/convert/xplane_to_perf_counters.cc:33-128`
- `xprof/convert/tpu_generic_utilization_utils.cc` (metric math)

### C++ convert — pod / OpStats / smart
- `xprof/convert/pod_viewer_processor.cc:37-70`
- `xprof/convert/op_stats_to_pod_viewer.cc:30-61`
- `xprof/convert/op_stats_to_pod_stats.cc`
- `xprof/convert/multi_xplanes_to_op_stats.cc:42-159`
- `xprof/convert/smart_suggestion_processor.cc:62-129`
- `xprof/convert/smart_suggestion/tool_data_provider_impl.h:52-180`
- `xprof/convert/smart_suggestion/all_rules.h`
- `xprof/convert/repository.cc:51-58`

### Protos
- `plugin/xprof/protobuf/smart_suggestion.proto`
- `plugin/xprof/protobuf/pod_stats.proto`
- `plugin/xprof/protobuf/inference_stats.proto` (via sources)
- `plugin/xprof/protobuf/op_stats.proto` (OpStats hub)

### Frontend
- `frontend/app/components/graph_viewer/graph_viewer.ts:50-92,174,815-820`
- `frontend/app/components/inference_profile/inference_profile.ts:45-156`
- `frontend/app/components/utilization_viewer/utilization_viewer.ts:110-230`
- `frontend/app/components/smart_suggestion/smart_suggestion_view.ts`
- `frontend/app/components/perf_counters/perf_counters.ts:34,83-86`
- `frontend/app/components/pod_viewer/pod_viewer.ts`
- `frontend/app/components/pod_viewer/pod_viewer_common.ts:249`

### CLI
- `plugin/xprof/cli/tools/get_graph_viewer_tool.py`
- `plugin/xprof/cli/tools/get_utilization_viewer_tool.py:254-309`
- `plugin/xprof/cli/tools/get_smart_suggestions_tool.py`
- `plugin/xprof/cli/tools/get_kpi_metrics_tool.py:16-93`
- `plugin/xprof/cli/tools/get_top_hlo_ops_tool.py:21-154`
- `plugin/xprof/cli/internal/oss/hlo_tools.py:92-272`
- `plugin/xprof/cli/xprof_cli.py:50-67`

### Related specs
- `docs/superpowers/specs/memory-optimization/00-synthesis-memory-optimization.md`
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md`
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (HLO memory_viewer sibling)
- `docs/superpowers/specs/memory-optimization/04-overview-and-stats-tools.md` (OpStats / overview)

---

## Appendix A — Key constants

| Constant | Value | Where |
|----------|-------|-------|
| HLO_TOOLS | `graph_viewer`, `memory_viewer` | `registry.py:73` |
| ALL_HOSTS_ONLY (this set) | `pod_viewer`, `smart_suggestion` | `registry.py:68-70` |
| ALL_HOSTS_SUPPORTED (this set) | `pod_viewer` only | `registry.py:58-65` |
| DEFAULT_CACHE_TOOLS | overview + trace_viewer@ (not this set) | `registry.py:55` |
| kDefaultWidth | 3 | `tool_options.h:40` |
| Options builder default width | 3 | `graph.py:19-20` |
| FE default graphWidth | 1 | `graph_viewer.ts:86` |
| CLI get_graph_viewer graph_width | 1 | `get_graph_viewer_tool.py:20` |
| GRAPH_HTML_THRESHOLD | 1_000_000 bytes (warn only) | `graph_viewer.ts:50` |
| FE opProfileLimit | 300 | `graph_viewer.ts:92` |
| Graph type names | graph, adj_nodes, short_txt, long_txt, pb, pbtxt, json, custom_call | `tool_options.h:31-38` |
| kDefaultFormatString | `"url"` | `tool_options.h:39` |
| Inference percentiles | 50,75,90,99,99.9,99.99 | `inference_stats_sampler.cc:181-187` |
| Max samples / percentile | 10 | `kMaxNumDataSelectedPerPercentile` `:190` |
| Utilization hosts | exactly 1 XSpace | `xplane_to_tools_data.cc:371-375` |
| Utilization devices | TPU v7x, TPU v6 Lite | `xplane_to_utilization_viewer.cc:112-118` |
| CLI util/smart/top cache | 86400 s | decorators on CLI tools |
| OpStats merge step cap | `uint32 max` (unbounded) | `multi_xplanes_to_op_stats.cc:91-93` |
| JSON always_print | true | pod_viewer, smart_suggestion processors |
| SMART_SUGGESTION suffix | `.smart_suggestion.pb` | `repository.cc:55` |
| OP_STATS suffix | `.op_stats_v2.pb` | `repository.cc:54` |
| HLO proto suffix | `.hlo_proto.pb` | `xplane_to_hlo.cc:48` |
| hlo_tools max_lines default | 2000 (client only) | `hlo_tools.py:97` |
| get_top_hlo_ops limit default | 10 | `get_top_hlo_ops_tool.py:22` |
| FE perf_counters pageSizeOptions | 30, 50, 100, 200 | `perf_counters.ts:34` |
| Op profile limit (one convert path) | 100 | `xplane_to_tools_data.cc:249` |

---

## Appendix B — Content-type / format matrix (graph_viewer)

| type | Processor SetOutput | Python raw_to_tool_data content_type | Typical consumer |
|------|---------------------|--------------------------------------|------------------|
| graph | application/json (processor) | text/html | FE iframe |
| short_txt / long_txt | application/json | application/octet-stream (download types) | CLI / download |
| pb / pbtxt / json | application/json | application/octet-stream | download |
| adj_nodes | application/json | text/plain default branch | FE / tools |
| custom_call | application/json | text/plain default | FE |

Cite: `graph_viewer_processor.cc:122`, `raw_to_tool_data.py:191-204`.

Mismatch means intermediate layers may buffer as "JSON" while body is HTML — usually harmless for memory but relevant to caching keys / compressors.

---

## Appendix C — Host policy matrix (this set)

| Tool | ALL_HOSTS_SUPPORTED | ALL_HOSTS_ONLY | Single-XSpace enforce | Multi-host convert behavior |
|------|---------------------|----------------|----------------------|-----------------------------|
| graph_viewer | no | no | no (HLO from session dir) | HLO extract walks all hosts once |
| inference_profile | no | no | no | Parallel all selected hosts |
| utilization_viewer | no | no | **yes** (`!=1` error) | Reject multi |
| smart_suggestion | no | **yes** | n/a | OpStats all hosts |
| perf_counters | no | no | no | Sequential all hosts → one table |
| pod_viewer | **yes** | **yes** | n/a | OpStats all hosts |

Cite: `registry.py:58-70`, `xplane_to_tools_data.cc:370-375`.

---

## Appendix D — Cold vs warm cost cliffs

| Tool | Cold | Warm | Notes |
|------|------|------|-------|
| graph_viewer | HLO extract all hosts + IR + render | Read one `.hlo_proto.pb` + IR + render | No render cache |
| inference_profile | Parallel full convert every request | Same (no durable InferenceStats cache) | Column change = full recompute |
| utilization_viewer | Full single-host every request | Same (no tool disk cache) | CLI client cache 86400 only |
| smart_suggestion | OpStats + memory_profile + event analyzer + rules | Read `.smart_suggestion.pb` | Largest cliff |
| perf_counters | Sequential all hosts every request | Same | names_only bypass |
| pod_viewer | OpStats N-host vector | Read OpStats cache + Pod JSON rebuild | Pod JSON not separately cached |

---

## Appendix E — CLI waste pattern taxonomy

| Pattern | Example in this set | Fix ID |
|---------|---------------------|--------|
| Full table → few scalars | get_utilization_viewer | UV-1 / CLI-UV-1 |
| Dual full tools → few KPIs | get_kpi_metrics | CLI-KPI-1 |
| Full tree → top-K | get_top_hlo_ops | CLI-TOP-1 |
| Full module text → max_lines / neighborhood | hlo_tools | CLI-HLO-1 / CLI-HLO-2 / GV-9 |
| Tiny output, huge cold convert | get_smart_suggestions | SS-1 / SS-2 / SS-3 |
| Full HTML for inspect | get_graph_viewer type=graph | GV-2 / CLI-GV-1 |

Shared slogan (with 03-doc): **UI-scale is the only convert mode; CLI is a client-side reducer.**

---

## Appendix F — Worked peak narratives (qualitative)

### F.1 Multi-host pod_viewer first open (H)

1. UI selects ALL_HOSTS (only option).  
2. Cache miss on `ALL_HOSTS.op_stats_v2.pb`.  
3. For host 0..N-1: load XSpace, preprocess with derived_timeline, convert full OpStats, **retain all in `vector<OpStats>`**.  
4. Step intersection unbounded; combine into one OpStats.  
5. Write cache.  
6. Build PodViewerDatabase (steps × cores).  
7. `MessageToJsonString(always_print=true)`.  
8. Gzip + FE parse full topology.

Peak ≈ N × OpStats + combined + JSON (+ gzip buffers from 02-doc). Stream-combine (X-1) removes N× residency.

### F.2 Large module graph pan (H)

1. HLO already extracted.  
2. Load 100MB-class HloProto.  
3. ConvertHloProtoToModule → IR coexists with proto.  
4. RenderNeighborhoodAround radius=3 (if options default 3 from non-FE entry).  
5. DOT string multi-MB → WrapDotInHtml embeds DOT → HTML larger still.  
6. FE warns at 1MB but still injects into iframe; Graphviz WASM renders SVG.  
7. Parallel op_profile limit 300 for hover colors.

GV-2 + GV-7 + GV-5 cut the common failure mode.

### F.3 Agent get_utilization_viewer (H waste / M absolute)

1. Agent asks for HBM/MXU %.  
2. Server converts full host utilization table.  
3. CSV of every sample × metric transferred.  
4. Python parses all rows, filters host/device/node, averages.  
5. Returns ~11 numbers.

Absolute RSS may be smaller than pod multi-host, but **waste ratio** (bytes converted / bytes needed) is extreme — severity H for CLI architecture.

### F.4 smart_suggestion cold (H)

1. Cache miss on `.smart_suggestion.pb`.  
2. ToolDataProvider builds OpStats (same as F.1).  
3. Converts overview/input/op_profile from OpStats (extra trees).  
4. Invokes full memory_profile tool → JSON → parse MemoryProfile.  
5. Invokes event_time_fraction_analyzer convert.  
6. Runs rules → few strings → write tiny cache.

SS-2 + SS-3 turn this into "read OpStats cache + few metrics."

---

## Appendix G — Interaction with sibling specs

| Sibling | Interaction |
|---------|-------------|
| `02-convert-serve-cache-path` | Gzip double-buffer, no concurrent convert cap, respond() path — multiplies every tool here |
| `03-memory-tools-family` | HLO extract shared; memory_profile used inside smart_suggestion; same CLI scalar-waste pattern |
| `04-overview-and-stats-tools` | Shared OpStats vector + cache; overview may re-run InferenceStats; KPI uses overview |
| `06-megascale-and-cli-detectors` | Additional CLI detectors may re-fetch graph/HLO text — HLO amortization |
| `00-synthesis` | Ranks stream OpStats, CLI summary, HLO amortization globally |

---

## Appendix H — Suggested measurement checklist (no code)

| Probe | What to capture | Tools |
|-------|-----------------|-------|
| RSS delta | Peak RSS during first convert | pod, smart_suggestion, inference |
| Intermediate vs wire | OpStats total bytes vs Pod JSON bytes | pod |
| Request_details count | Before/after sample | inference |
| Graph HTML bytes | vs graph_width 1/2/3 | graph_viewer |
| CLI ratio | response_bytes / final_json_bytes | util, KPI, top_hlo |
| Cache hit | Time/RSS warm smart_suggestion vs cold | smart_suggestion |
| Thread count | Active convert threads under multi-host inference | inference |
| Description string share | Unique vs total Description cells | perf_counters |

Confidence upgrades from **hypothesized** → **observed** only after these probes.

---

## Appendix I — Non-goals / out of scope

- Product code changes (explicitly forbidden for this note).  
- Trace viewer streaming architecture (01-doc).  
- Megascale / DCN tools (06-doc) except shared HLO amortization references.  
- Correctness of TPU counter formulas (utilization math) — only residency.  
- UI visual redesign beyond memory implications of DataTable retention.

---

## Appendix J — One-line takeaways per tool

| Tool | One-liner |
|------|-----------|
| graph_viewer | Proto+IR+DOT+HTML stacked; FE only warns at 1MB; CLI truncates after full dump. |
| inference_profile | Wire is sampled; peak is all requests × parallel hosts. |
| utilization_viewer | Fine for single-host UI; CLI fetches the universe for eleven floats. |
| smart_suggestion | Output is a postcard; cold path is a freight train of other tools. |
| perf_counters | Event-level dump with repeated strings; aggregate is the missing default. |
| pod_viewer | Multi-host OpStats vector is the shared dragon; step×core JSON is the tail. |

---

## Appendix K — Opportunity ID ↔ primary cite map

| ID | Primary cite |
|----|--------------|
| GV-1 | `graph_viewer_processor.cc:105-124` (no cache) |
| GV-2 | `graph_viewer.ts:50,815-820` |
| GV-5 | `graph_viewer.ts:92` |
| GV-6 | `xplane_to_hlo.cc:105-132` |
| GV-7 | `tool_options.h:40` vs `graph_viewer.ts:86` vs `graph.py:20` |
| GV-9 | `hlo_tools.py:136-158` |
| IP-1/IP-3 | `multi_xspace_to_inference_stats.cc:162-165` + sampler caps |
| IP-2 | `multi_xspace_to_inference_stats.cc:125-127` |
| UV-1 | `get_utilization_viewer_tool.py:254-309` |
| UV-2 | `xplane_to_utilization_viewer.cc:297-308` |
| SS-1 / PV-1 / X-1 | `multi_xplanes_to_op_stats.cc:50-66` |
| SS-2 | `tool_data_provider_impl.h:148-165` |
| PC-1 | `xplane_to_perf_counters.cc:69-97` |
| PV-2 | `multi_xplanes_to_op_stats.cc:91-93` |
| CLI-KPI-1 | `get_kpi_metrics_tool.py:35-36` |
| CLI-TOP-1 | `get_top_hlo_ops_tool.py:34-38,124-139` |
| X-5 | `pod_viewer_processor.cc:45-48`, `smart_suggestion_processor.cc:116-119` |

---

## Appendix L — Document history

| Date | Mode | Change |
|------|------|--------|
| 2026-07-13 | analysis only | Initial opportunity notes (shorter form) |
| 2026-07-13 | analysis only | Thorough expansion: path:line cites, CLI waste, appendices; overwrite ≥800 lines target |

---

*End of analysis. No product code in this document.*
