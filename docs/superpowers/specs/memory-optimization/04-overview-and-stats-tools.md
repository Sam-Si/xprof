# Overview & Stats Tools Family — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `overview_page`, `input_pipeline_analyzer`, `framework_op_stats`, `kernel_stats`, `hlo_stats`, `op_profile`, `roofline_model`  
**Companion:** family summary in `00-synthesis-memory-optimization.md`; convert/serve/cache shared path in `02-convert-serve-cache-path.md`; non-OpStats memory tools in `03-memory-tools-family.md`

---

## How to read this document

| Convention | Meaning |
|------------|---------|
| Severity **H** | High: multi-host peak, unbounded wire, default warm-up, or multi-GB risk |
| Severity **M** | Medium: measurable 2× waste or browser heap; moderate fix effort |
| Severity **L** | Low: polish / secondary / optional path |
| Confidence **observed** | Confirmed by code path with `path:line` cites |
| Confidence **hypothesized** | Plausible from structure; needs measurement |
| Path cites | Repo-relative `file:line` or `file:start-end` |
| OpStats hub | Combined multi-host `OpStats` proto is the shared intermediate for all seven tools |

Scope is **this OpStats fan-out family only**. No product code changes in this note.

---

## 0. Executive summary

All seven tools are **OpStats fan-out tools**: one expensive multi-host extract of `OpStats` (always with metrics + step_db + kernel_stats_db), then tool-specific JSON projections.

### Peak memory drivers (family)

| Rank | Driver | Severity | Confidence | Primary cites |
|------|--------|----------|------------|---------------|
| 1 | `std::vector<OpStats> all_op_stats` holds **all hosts** before combine | **H** | observed | `multi_xplanes_to_op_stats.cc:50-66`, `:131-137` (Reduce same pattern) |
| 2 | `OpStatsOptions` always all-true in cache path | **H** | observed | `multi_xplanes_to_op_stats.cc:121-124`, `base_op_stats_processor.cc:111-115` |
| 3 | Unbounded wire tables (`kernel_stats`, `hlo_stats`) | **H** | observed | `xplane_to_kernel_stats_db.cc:125-149`, `op_stats_to_hlo_stats.cc:136-140` |
| 4 | Overview inference second full multi-XSpace pass | **H** | observed | `overview_page_processor.cc:53-65`, `multi_xspace_to_inference_stats.cc:117-165` |
| 5 | DEFAULT warm-up always builds overview → forces OpStats early | **H** | observed | `registry.py:55` |
| 6 | FE pageSize is presentation-only; full tables resident | **M** | observed | `kernel_stats_table.ts:111`, `hlo_stats.ts:129-142` |
| 7 | Dual payloads (framework idle modes; roofline infeed true/false) | **M** | observed | `op_stats_to_tf_stats.cc:125-128`, `roofline_model_processor.cc:45-50` |

### Family invariant (observed)

```text
Every tool pays full OpStats extract cost (metrics + steps + kernels)
even when the UI view only needs a slice.
Cache hit on ALL_HOSTS OpStats amortizes recompute across tools,
but peak convert and wire remain tool-specific.
```

---

## 1. Registry placement

Source: [`plugin/xprof/profile_plugin/tools/registry.py`](plugin/xprof/profile_plugin/tools/registry.py)

| Concern | Membership / policy | Cite |
|---------|---------------------|------|
| **XPLANE_TOOLS** | All seven listed | `registry.py:33-52` (`overview_page` :37, `input_pipeline_analyzer` :38, `framework_op_stats` :39, `kernel_stats` :40, `op_profile` :42, `hlo_stats` :43, `roofline_model` :44) |
| **DEFAULT_CACHE_TOOLS** | `('overview_page', 'trace_viewer@')` — overview always warmed | `registry.py:55` |
| **ALL_HOSTS_SUPPORTED** | overview_page, input_pipeline_analyzer, framework_op_stats, kernel_stats | `registry.py:58-66` |
| **ALL_HOSTS_ONLY** | overview_page only among this family | `registry.py:68-70` |
| **Not in ALL_HOSTS UI sets** | op_profile, hlo_stats, roofline_model | by exclusion from :58-70 |
| **Python dispatch** | `raw_to_tool_data.py` → `xspace_wrapper_func` | `raw_to_tool_data.py:146-190` |
| **C++ base** | `BaseOpStatsProcessor` Map/Reduce/ProcessSession | `base_op_stats_processor.{h,cc}` |
| **Combined cache** | `ConvertMultiXSpaceToCombinedOpStatsWithCache` | `multi_xplanes_to_op_stats.cc:116-159` |
| **Host selection** | `HostSelector` | `services/hosts.py:24-115` |
| **Proto hub** | `plugin/xprof/protobuf/op_stats.proto` | — |

### 1.1 Python dispatch table (`raw_to_tool_data.py`)

| Tool | Branch lines | Notes |
|------|--------------|-------|
| `overview_page` | `:146-149` | JSON via xspace_wrapper |
| `input_pipeline_analyzer` | `:150-153` | JSON |
| `framework_op_stats` | `:154-157` | JSON |
| `kernel_stats` | `:159-162` | JSON |
| `op_profile` | `:173-177` | sets `group_by` default `'program'` |
| `hlo_op_profile` | `:178-182` | same as op_profile grouping |
| `hlo_stats` | `:183-186` | JSON |
| `roofline_model` | `:187-190` | JSON |

No per-tool slim options are passed on these branches (unlike graph_viewer / memory_viewer which build option dicts). **observed** gap.

### 1.2 ALL_HOSTS policy matrix (this family)

| Tool | ALL_HOSTS_SUPPORTED | ALL_HOSTS_ONLY | Empty-host HostSelector | Typical peak driver |
|------|---------------------|----------------|-------------------------|---------------------|
| overview_page | yes | **yes** | all planes (`hosts.py:94-97`) | N× XSpace → OpStats + optional N× inference |
| input_pipeline_analyzer | yes | no | all planes if empty | N× OpStats; unbounded steps |
| framework_op_stats | yes | no | all planes if empty | N× OpStats; wire capped 500×2 |
| kernel_stats | yes | no | all planes if empty | N× OpStats; **uncapped** kernel rows |
| hlo_stats | no | no | all planes if empty (**pitfall**) | Per-host preferred; **uncapped** HLO rows |
| op_profile | no | no | all planes if empty (**pitfall**) | Tree cap 100; OpStats full |
| roofline_model | no | no | all planes if empty (**pitfall**) | Cap 1000; dual convert + FE copies |

**HostSelector empty-host behavior** (`hosts.py:94-97`): when neither `host` nor `hosts_param` is set, **all** XPlane files are selected. Combined with `:107-110` (require host only when `tool not in ALL_HOSTS_ONLY` **and** asset_paths empty — which is not the empty-host case), non-ALL_HOSTS tools can still convert multi-host if the client omits host.

---

## 2. Family shared architecture — OpStats fan-out

### 2.1 Shared short path (all seven)

```text
HTTP /data → HostSelector (1..N XPlanes)                    # hosts.py:27-115
  → raw_to_tool_data.py (tool branch; use_saved_result default True)
  → C++ ProfileProcessor / BaseOpStatsProcessor
       ├─ ProcessSession path:
       │    ConvertMultiXSpaceToCombinedOpStatsWithCache    # multi_xplanes_to_op_stats.cc:116
       │      options ALWAYS:
       │        generate_op_metrics_db = true               # :122
       │        generate_step_db = true                     # :123
       │        generate_kernel_stats_db = true             # :124
       │      cache: ALL_HOSTS OP_STATS binary proto
       │    → ProcessCombinedOpStats (tool-specific)
       │
       └─ Map/Reduce path (worker service multi-host):
            Map: per-host ConvertXSpaceToOpStats + write OP_STATS  # base_op_stats_processor.cc:98-121
            Reduce: load ALL per-host OpStats into vector          # :131-137
                    CombineAllOpStats → ALL_HOSTS OP_STATS
                    ProcessCombinedOpStats
  → JSON string (DataTable JSON or MessageToJsonString)
  → respond() gzip                                                  # see 02-convert-serve-cache-path.md
  → Angular component holds full parsed object graph
```

### 2.2 The vector-OpStats TODO (critical multi-host peak)

**File:** `xprof/convert/multi_xplanes_to_op_stats.cc`

```text
:50-52  // TODO(profiler): Change the combiner to convert and combine one
        // OpStats at a time, to reduce peak memory usage.
:53-54  std::vector<OpStats> all_op_stats;
        all_op_stats.reserve(session_snapshot.XSpaceSize());
:55-66  for each XSpace i:
          Arena + GetXSpace
          PreprocessSingleHostXSpace(step_grouping=true, derived_timeline=true)
          ConvertXSpaceToOpStats → push_back into all_op_stats
:78-105 ComputeStepIntersection (max steps = UINT32_MAX)  # :91-93
        CombineAllOpStats → *combined_op_stats
```

**Same pattern in Reduce** (`base_op_stats_processor.cc:131-155`):

```text
:131-137  vector<OpStats> all_op_stats; load every map_output file
:140-149  OpStatsInfo pointers into vector (size stable — comment :142-144)
:151-155  step intersection unbounded + CombineAllOpStats
```

**Peak model (hypothesized math, structure observed):**

```text
peak_rss ≈ Σ_i size(OpStats_i) + size(combined_op_stats)
         + Σ_i transient(XSpace_i + Arena) during loop
         + step_intersection working set
         + tool JSON string

With N hosts of roughly equal OpStats size S:
  current:  ~ N·S + combined(~S_merged)  during combine
  ideal streaming combine: ~ 2·S + combined  (host i + running merge)
```

**Severity: H** multi-host. **Confidence: observed** for the vector residency; **hypothesized** for exact GB scaling without heap profiles.

### 2.3 OpStatsOptions — always full

| Call site | Flags | Cite |
|-----------|-------|------|
| `ConvertMultiXSpaceToCombinedOpStatsWithCache` | metrics+step+kernel all `true` | `multi_xplanes_to_op_stats.cc:121-124` |
| `BaseOpStatsProcessor::Map` | same all `true` | `base_op_stats_processor.cc:111-115` |
| Struct defaults | all `false` if caller omits | `xplane_to_op_stats.h:31-36` |

**Implication:** the default struct would allow per-tool slim builds, but **every production cache path overrides to full**. Tools that only need `kernel_stats_db` still pay metrics+steps; tools that only need metrics still pay kernels.

| Tool | Likely minimum DBs (hypothesized) | Still pays (observed always-full) |
|------|-----------------------------------|-----------------------------------|
| overview_page | metrics + step (+ env/diagnostics) | + kernel |
| input_pipeline_analyzer | step + host metrics | + kernel |
| framework_op_stats | metrics (+ kernel for TensorCore col on GPU) | full |
| kernel_stats | kernel only | + metrics + step |
| hlo_stats | device metrics | + kernel + step |
| op_profile | device metrics | + kernel + step |
| roofline_model | device metrics + perf_env | + kernel + step |

### 2.4 Cache file model

| Artifact | Identifier | Writer | Reader |
|----------|------------|--------|--------|
| Per-host OpStats | hostname | `Map` `WriteBinaryProto` `base_op_stats_processor.cc:119-120` | `Reduce` load; `AreAllOpStatsCached` |
| Combined OpStats | `kAllHostsIdentifier` | cache miss path `:144-146`; Reduce `:157-159` | `ConvertMultiXSpaceToCombinedOpStatsWithCache` hit `:129-134` |

**Cache hit path still materializes full combined `OpStats` into RAM** before projection — disk amortization of **CPU**, not of **peak RSS for serve**. **observed** `:132-134`.

**ShouldUseWorkerService** (`base_op_stats_processor.cc:174-187`): multi-host (`XSpaceSize() > 1`) uses worker when `use_saved_result` false OR not all per-host caches present. Single host always local.

### 2.5 Step intersection unbounded

```text
ComputeStepIntersectionToMergeOpStats(all_op_stats_info,
    std::numeric_limits<uint32_t>::max())   # multi_xplanes_to_op_stats.cc:91-93
                                           # base_op_stats_processor.cc:151-152
```

No cap on merged step count → long profiles amplify `step_db` in combined OpStats and any tool that projects per-step rows (input_pipeline, overview steptime). **observed**.

### 2.6 Wire format classes

| Class | Tools | Encode path | Row bound |
|-------|-------|-------------|-----------|
| DataTable JSON array / object | overview, input_pipeline, framework_op_stats, kernel_stats, hlo_stats, roofline | `DataTable::ToJson` | tool-specific |
| Proto MessageToJsonString | op_profile | `always_print_fields_with_no_presence=true` | tree children_per_node 100 |

### 2.7 Frontend residency pattern (family)

Common pattern across table tools:

1. HTTP JSON → `JSON.parse` → `SimpleDataTable` object graph.  
2. `new google.visualization.DataTable(simple)` — often a **second** full copy.  
3. Table chart `pageSize: N` — **draws** N rows; does **not** free the rest.  
4. Filters via `DataView` sometimes clone via `toDataTable()` (another copy).

| Tool | pageSize | Full residency | Cite |
|------|----------|----------------|------|
| kernel_stats | 100 | yes | `kernel_stats_table.ts:111` |
| hlo_stats | 10 (× tables) | yes | `hlo_stats.ts:129`, `:142` |
| framework_op_stats | UI paging if present | dual tables + NgRx | `stats_table_data_provider.ts:114-115` clone |
| input_pipeline | charts + detail | full step series | `input_pipeline.ts:85-92` |
| overview_page | multi DataTable tuple | full | `overview_page_module.ts` |
| op_profile | tree | Map cache per group_by | `op_profile.ts:43-100` |
| roofline_model | multi DataTables | raw + program + op + scatters | `roofline_model.ts:84-107` |

### 2.8 Shared fan-out diagram

```text
                    ┌─────────────────────────────┐
   N × XSpace  ───► │ ConvertXSpaceToOpStats      │
                    │ (metrics+step+kernel ALWAYS)│
                    └─────────────┬───────────────┘
                                  │ vector holds N OpStats  ◄── FAM-OS-1 TODO
                                  ▼
                    ┌─────────────────────────────┐
                    │ CombineAllOpStats           │
                    │ → ALL_HOSTS.op_stats.pb     │
                    └─────────────┬───────────────┘
                                  │ one combined OpStats in RAM
          ┌───────────┬───────────┼───────────┬───────────┬───────────┬───────────┐
          ▼           ▼           ▼           ▼           ▼           ▼           ▼
     overview   input_pipe   framework   kernel_stats  hlo_stats  op_profile  roofline
     (+infer?)  (steps)      (500×2)     (uncapped)    (uncapped) (limit 100) (1000×2)
          │           │           │           │           │           │           │
          └───────────┴───────────┴───────────┴───────────┴───────────┴───────────┘
                                  ▼
                         gzip JSON → FE full graph
```

### 2.9 Ranked shared opportunities (FAM-OS)

| ID | Opportunity | Hotspot | Confidence | Affects |
|----|-------------|---------|------------|---------|
| FAM-OS-1 | **Incremental combine**: convert host → merge → drop host OpStats (fix multi_xplanes TODO :50-52) | **H** | observed | all multi-host |
| FAM-OS-2 | **Per-tool OpStatsOptions** (don't always build metrics+step+kernel) | **H** | observed | all; biggest KS/IP/OV |
| FAM-OS-3 | **Wire row caps** where missing (kernel_stats, hlo_stats) | **H** | observed | KS, HS |
| FAM-OS-4 | **Lazy heavy fields** (HLO expressions, recommendation HTML) | **H** | hypothesized | HS, OV |
| FAM-OS-5 | **Host policy hygiene**: empty host ≠ all planes for non-aggregate tools; require explicit host for OP/HS/RF | **M** | observed | OP, HS, RF |
| FAM-OS-6 | **Tool-result disk cache** beyond OpStats (JSON keyed by tool+host+options) | **M** | hypothesized | all |
| FAM-OS-7 | **JSON bloat policy**: no always_print; DataTable omit empty cells | **M** | observed | OP, tables |
| FAM-OS-8 | **FE presentation ≠ storage**: pageSize only draws; virtualize or server page | **M** | observed | KS, HS, FO |
| FAM-OS-9 | **Concurrent convert cap** with warm-up (overview is DEFAULT_CACHE_TOOLS) | **H** | hypothesized | OV + siblings |
| FAM-OS-10 | **InferenceStats sharing** so overview doesn't re-read XSpaces | **H** | observed | OV |
| FAM-OS-11 | Cap step intersection / step_db for UI tools | **M** | hypothesized | IP, OV |
| FAM-OS-12 | Stream-parse FE JSON (avoid full SimpleDataTable + gviz clone) | **M** | hypothesized | table tools |
| FAM-OS-13 | Single combined OpStats load shared across concurrent tool requests | **M** | hypothesized | multi-tool open |
| FAM-OS-14 | mmap / arena-backed OpStats cache load (reduce peak on hit) | **L** | hypothesized | all |

---

## 3. overview_page

### 3.1 End-to-end data path

```text
HTTP /data?tag=overview_page[&host=ALL_HOSTS]
  → HostSelector: multi-host → ALL_HOSTS only in UI; empty host → all planes
  → raw_to_tool_data.py:146-149
  → OverviewPageProcessor::ProcessSession | ProcessCombinedOpStats
       ConvertMultiXSpaceToCombinedOpStatsWithCache     # overview_page_processor.cc:88-89
       ConvertOpStatsToOverviewPage(combined)           # :51, :92
       if !is_training:
         ConvertMultiXSpaceToInferenceStats(session,    # :58-59, :97-98
             "", "", &inference_stats)
           # multi_xspace_to_inference_stats.cc:117-165
           # parallel per-host: GetXSpace + Preprocess + GenerateInferenceStats
           # vector<InferenceStats> per_host size N     # :122
         ComputeInferenceLatencyResult → overview.inference_latency
       OverviewPageToJson → array of DataTables         # :70, :106
  → gzip → FE frontend/app/components/overview_page/
       holds analysis, steptime/input, env, tips, inference, diagnostics tables
```

**Sources:** `overview_page_processor.cc:45-112`, `op_stats_to_overview_page.{h,cc}`, `multi_xspace_to_inference_stats.cc:117-165`, `unified_overview_page_processor.cc`.

### 3.2 ALL_HOSTS / warm-up

| Policy | Behavior | Confidence |
|--------|----------|------------|
| ALL_HOSTS_ONLY | UI offers only aggregate when multi-host | **observed** `registry.py:68-70` |
| DEFAULT_CACHE_TOOLS | Always warm-up overview on session open | **observed** `registry.py:55` |
| HostSelector empty | all XPlanes | **observed** `hosts.py:94-97` |
| Peak severity | **H** multi-host first paint + warm-up races | **observed** structure |

Warm-up of overview forces full-session OpStats build early, which **helps** sibling tools on cache hit but **hurts** peak if concurrent with trace_viewer@ warm-up (`DEFAULT_CACHE_TOOLS` pair). See `02-convert-serve-cache-path.md` concurrency section.

### 3.3 Materialization stages

| Stage | What is kept | Discarded / not used later | Confidence |
|-------|--------------|----------------------------|------------|
| A. Per-host XSpace preprocess | step_grouping + derived_timeline | raw events after OpStats extract | **observed** `multi_xplanes_to_op_stats.cc:62-63` |
| B. Per-host OpStats | full metrics+step+kernel | — | **observed** options :122-124 |
| C. Vector all_op_stats | N full OpStats simultaneously | after combine (still peak) | **observed** :53-66 |
| D. Combined OpStats | merged DBs | intermediate hosts may still live until vector destructs | **observed** |
| E. OverviewPage proto | analysis, recommendation, env, input pipeline summary, diagnostics | most kernel_stats_db unused | **hypothesized** field use |
| F. Inference branch (non-training) | N× XSpace reload + InferenceStats + latency | full inference detail if only latency chart needed | **observed** gap |
| G. OverviewPageToJson | compact DataTable JSON multi-table | smaller than raw OpStats | **observed** |
| H. FE | tuple of DataTables; step-time graph embeds input-pipeline analysis | — | **observed** |

### 3.4 Inference dual-pass detail

`ConvertMultiXSpaceToInferenceStats` (`multi_xspace_to_inference_stats.cc:117-165`):

1. Allocates `vector<InferenceStats> per_host_inference_stats(xspaces_size)` — another N-way residency (`:122`).  
2. Thread pool up to `MaxParallelism()` (`:125-127`) — **parallel** peak may exceed serial OpStats path.  
3. Each task: Arena + GetXSpace + Preprocess (step_grouping=true, derived_timeline=**false**) + GenerateInferenceStats (`:131-148`).  
4. Join, CombineInferenceStatsResult, Regroup, Sample (`:152-165`).

**Does not reuse** the OpStats cache or step_db already built. **Severity H**, **confidence observed**.

### 3.5 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| OV-1 | Stream-combine OpStats one host at a time (fix multi_xplanes TODO) | **H** | observed |
| OV-2 | OpStatsOptions per tool (overview may skip `kernel_stats_db`) | **M** | hypothesized |
| OV-3 | Cache InferenceStats / attach to OpStats so overview avoids second XSpace sweep | **H** | observed gap |
| OV-4 | Bound or summarize steptime series embedded in overview JSON for long profiles | **M** | hypothesized |
| OV-5 | Warm-up: write per-host OpStats incrementally; avoid holding N hosts + combined | **H** | hypothesized |
| OV-6 | Slim recommendation HTML strings / tip tables | **L** | hypothesized |
| OV-7 | Serialize InferenceStats to disk on first compute (shared with inference_profile) | **M** | hypothesized |
| OV-8 | Defer inference latency until FE tab visible | **L** | hypothesized |
| OV-9 | Cap concurrent warm-up (overview + trace) process-wide | **H** | hypothesized |

**Top:** OV-1, OV-3, OV-5 / FAM-OS-9.

---

## 4. input_pipeline_analyzer

### 4.1 End-to-end data path

```text
HTTP /data?tag=input_pipeline_analyzer[&host=H|ALL_HOSTS]
  → raw_to_tool_data.py:150-153
  → InputPipelineProcessor
       ConvertMultiXSpaceToCombinedOpStatsWithCache | Map/Reduce
       ConvertOpStatsToInputPipelineAnalysis(combined)   # input_pipeline_processor.cc:46, :56
       InputPipelineAnalysisResultToDataTableJson        # :47, :59
         → host table + device tables + recommendation + max infeed
         + per-step detail rows from step_db (unbounded)
  → FE frontend/app/components/input_pipeline/
       summary + host/device detail + max infeed charts
       device_side_analysis_detail_data_provider clones DataView → DataTable
```

### 4.2 ALL_HOSTS

| Mode | Peak | Confidence |
|------|------|------------|
| ALL_HOSTS aggregate | full multi-host OpStats + all steps | **observed** supported `registry.py:59` |
| Per-host | one XPlane OpStats | **observed** |
| Severity | **H** ALL_HOSTS; **M** per-host | — |

### 4.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Needs | Primarily `step_db` (+ metrics for host ops / tf.data stats) | hypothesized |
| Still builds | Full kernel_stats + full op_metrics always | **observed** options always-true |
| Wire | Per-step detail over **unbounded** step count (no kMax in convert) | **observed** loops over `step_details` / `grouped_by_step` in `op_stats_to_input_pipeline_analysis.cc` e.g. `:168`, `:226`, `:595` |
| Step merge | intersection max = UINT32_MAX | **observed** multi_xplanes `:91-93` |
| FE | Full tables in component state; charts re-slice columns client-side | **observed** |
| FE copy | `device_side_analysis_detail_data_provider.ts` `toDataTable()` after DataView | **observed** |

### 4.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| IP-1 | OpStatsOptions: step_db + host metrics only | **H** | hypothesized |
| IP-2 | Cap / downsample per-step rows for UI (summary + top-K outliers) | **H** | hypothesized |
| IP-3 | Prefer per-host default when multi-host (don't auto all-hosts) | **M** | observed policy gap |
| IP-4 | Cache tool JSON separately from OpStats so step-table rebuild is free | **M** | hypothesized |
| IP-5 | Cap step_db at convert time for UI (keep full for CLI if needed) | **M** | hypothesized |
| IP-6 | FE: avoid DataView.toDataTable() full clone | **L** | observed |

**Top:** IP-1, IP-2.

---

## 5. framework_op_stats

### 5.1 End-to-end data path

```text
HTTP /data?tag=framework_op_stats[&host=...]
  → raw_to_tool_data.py:154-157
  → FrameworkOpStatsProcessor
       OpStats via cache/Map-Reduce
       ConvertOpStatsToTfStats(combined)                 # framework_op_stats_processor.cc:44, :54
         kMaxNumOfOps = 500                              # op_stats_to_tf_stats.cc:45
         host + device tables each top-500
         *with_idle and *without_idle both generated     # :125-128
       TfStatsToDataTableJson → "[with_idle, without_idle]"  # :208-214
  → FE frontend/app/components/framework_op_stats/
       NgRx store + pie charts + stats tables
       stats_table_data_provider clones DataTable         # :114-115
```

### 5.2 ALL_HOSTS

- ∈ **ALL_HOSTS_SUPPORTED** (`registry.py:60`). Aggregate merges metrics across hosts.  
- **Severity: M–H** (bounded top-500 helps wire; OpStats peak still H multi-host).

### 5.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Cap | Top 500 ops per side — good wire bound | **observed** `op_stats_to_tf_stats.cc:45`, `:79`, `:101` |
| Dual tables | with_idle + without_idle nearly 2× JSON | **observed** `:125-128`, `:208-214` |
| Intermediate | CreateTfMetricsDbFromDeviceOpMetricsDb + GroupKernelReportsByOpName | **observed** (call chain in same file) |
| FE | NgRx keeps full data + optional diff run; pie processors clone views | **observed** / **hypothesized** diff |
| Table UI | Presentation paging only if present; full rows still in memory | **hypothesized** paging; **observed** clone |

### 5.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| FO-1 | Emit only the idle mode FE needs (toggle server-side) | **M** | observed |
| FO-2 | Skip kernel_stats_db when TensorCore column not needed (non-GPU) | **M** | hypothesized |
| FO-3 | Diff mode: don't hold two full runs if only delta needed | **M** | hypothesized |
| FO-4 | Shared OpStats cache already helps — ensure host-scoped cache for per-host UI | **M** | observed |
| FO-5 | Lower kMaxNumOfOps via tool option for large sessions | **L** | hypothesized |
| FO-6 | FE: stop `dataTable.clone()` if format-in-place is safe | **L** | observed |

**Top:** FO-1, FO-2.

---

## 6. kernel_stats

### 6.1 End-to-end data path

```text
HTTP /data?tag=kernel_stats[&host=...]
  → raw_to_tool_data.py:159-162
  → KernelStatsProcessor
       ConvertMultiXSpaceToCombinedOpStatsWithCache      # kernel_stats_processor.cc:40-41
       KernelStatsToDataTableJson(combined.kernel_stats_db())  # :42-43, :51-52
         GenerateKernelStatsDataTable:
           for (const auto& report : kernel_stats_db.reports())  # xplane_to_kernel_stats_db.cc:125
             AddRow — NO max_records, NO top-K
  → FE frontend/app/components/kernel_stats/
       common_data_store holds SimpleDataTable
       kernel_stats_table pageSize: 100                  # kernel_stats_table.ts:111
       createDataTable builds full gviz DataTable        # :80-85
```

### 6.2 ALL_HOSTS

- ∈ **ALL_HOSTS_SUPPORTED** (`registry.py:61`). Combine merges kernel reports.  
- **Severity: H** for large GPU sessions (unique kernels × launch configs).

### 6.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Build | Always on every OpStats path (`generate_kernel_stats_db=true`) even for non-kernel tools | **observed** |
| Wire | **All** `kernel_stats_db.reports()` → rows (name, dims, op_name, durations…) | **observed** `:125-149` |
| Columns | 15 columns including block/grid dim strings | **observed** `:100-116` |
| FE | Full SimpleDataTable in store; Google chart table pages 100 rows | **observed** |
| Waste | Full OpMetrics + step_db still built for this tool | **observed** |
| Sort | Reports expected total_duration descending (DCHECK) | **observed** `:146-148` |

**Contrast with siblings that cap:**

| Tool | Cap constant | Cite |
|------|--------------|------|
| framework_op_stats | `kMaxNumOfOps = 500` | `op_stats_to_tf_stats.cc:45` |
| roofline_model | `kMaxNumRecords = 1000` | `op_stats_to_roofline_model.cc:59` |
| op_profile | `op_profile_limit = 100` | `op_profile_processor.cc:53` |
| **kernel_stats** | **none** | `xplane_to_kernel_stats_db.cc:125` |
| **hlo_stats** | **none** (`SortedOpMetricsDb` default max_records=-1) | `op_metrics_to_record.h:33-34`, `op_stats_to_hlo_stats.cc:136-137` |

### 6.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| KS-1 | Top-K / cumulative-time cutoff on wire (mirror FO `kMaxNumOfOps` / roofline 1000) | **H** | observed gap |
| KS-2 | OpStatsOptions: kernel_stats_db only for this tool | **H** | hypothesized |
| KS-3 | Server-side filter (name substring) + pagination params | **M** | hypothesized |
| KS-4 | FE: virtualize / drop non-visible columns from resident table | **M** | hypothesized |
| KS-5 | Don't regenerate metrics/step DBs when serving from full OP_STATS cache (lazy field use) | **L** | hypothesized |
| KS-6 | Cumulative 99% self-time cutoff instead of hard K | **M** | hypothesized |

**Top:** KS-1, KS-2.

---

## 7. hlo_stats

### 7.1 End-to-end data path

```text
HTTP /data?tag=hlo_stats[&host=...]
  → raw_to_tool_data.py:183-186
  → HloStatsProcessor / UnifiedHloStatsProcessor
       OpStats via cache
       ConvertOpStatsToHloStats(combined)                # op_stats_to_hlo_stats.cc:127-141
         SortedOpMetricsDb(hlo_metrics_db)  // max_records default -1 = ALL
         AddHloStatsRecordsRecursively (+ SparseCore children)
       HloStatsToDataTableJson
         CreateHloStatsDataTable — 29 columns including
         full hlo_op_expression text                     # :155-190, :200-231
  → FE frontend/app/components/hlo_stats/
       table + pie + flop chart; pageSize: 10 UI only    # hlo_stats.ts:129, :142
       expression often hidden but still in payload
```

### 7.2 ALL_HOSTS

- **Not** in ALL_HOSTS_SUPPORTED/ONLY → multi-host UI lists per host.  
- Empty host still selects **all** XPlanes (`hosts.py:94-97`) → accidental multi-host convert possible.  
- **Severity: H** (unbounded rows + long expressions); multi-host accidental **H**.

### 7.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Rows | One row per HLO metric (+ SC children); expressions can be multi-KB | **observed** recursive add `:88-122` |
| Sort | Full sort of metrics_db then emit all | **observed** `:136-139` |
| Wire columns | ~29 columns including expressions, source_info, BW fields | **observed** `:155-190` |
| Expression twice | name parsed from expression **and** full expression cells | **observed** `:205-206` |
| FE | Full table held; expression column often hidden but still downloaded | **observed** / **hypothesized** hide |
| pageSize | 10 — presentation only | **observed** |

### 7.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| HS-1 | Top-K by self-time (e.g. 500–2000) like framework_op_stats | **H** | observed gap |
| HS-2 | Lazy long expression: name-only list + detail-by-id fetch | **H** | hypothesized |
| HS-3 | OpStatsOptions trim (device metrics only; skip kernel_stats) | **M** | hypothesized |
| HS-4 | Require explicit host when multi-host (no empty-host all-planes) | **M** | observed |
| HS-5 | FE strip unused columns after parse | **L** | hypothesized |
| HS-6 | Omit hlo_expression from default columns; optional `?fields=` | **H** | hypothesized |
| HS-7 | Cap SparseCore child expansion | **M** | hypothesized |

**Top:** HS-1, HS-2, HS-4.

---

## 8. op_profile

### 8.1 End-to-end data path

```text
HTTP /data?tag=op_profile|hlo_op_profile&group_by=program|...
  → raw_to_tool_data.py:173-182  (sets options['group_by'])
  → OpProfileProcessor / UnifiedOpProfileProcessor
       ConvertMultiXSpaceToCombinedOpStatsWithCache
       ConvertOpStatsToOpProfile(..., op_profile_limit=100, group_by)
                                                      # op_profile_processor.cc:50-53, :74-77
       MessageToJsonString with
         always_print_fields_with_no_presence = true  # :55-56, :79-80
  → FE frontend/app/components/op_profile/
       opProfileDataCache = Map<string, OpProfileProto>  # op_profile.ts:43
       fetchData caches each group_by                    # :81-100
```

### 8.2 ALL_HOSTS

- Not in ALL_HOSTS UI sets; typically one host. Empty host → all XPlanes (same pitfall as hlo_stats).  
- **Severity: M** wire (capped tree); **H** if multi-host OpStats forced.

### 8.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Cap | children_per_node / op_profile_limit 100 — good | **observed** `:53`, `op_profile_builder.cc:796-797` |
| JSON | always_print inflates empty metrics fields | **observed** `:55-56` |
| FE | Caches each group_by result for session; tree still full proto | **observed** `op_profile.ts:43-100` |
| Regroup | Changing group_by re-runs convert (unless FE cached) over same OpStats | **observed** |
| OpStats DBs | Still full metrics+step+kernel for a metrics-only tree | **observed** |

### 8.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| OP-1 | Disable always_print empty fields | **M** | observed |
| OP-2 | Cache OpProfile JSON keyed by group_by on disk | **M** | hypothesized |
| OP-3 | Build profile from flat metrics without full step/kernel DBs | **M** | hypothesized |
| OP-4 | FE: free unused group_by cache entries / cap Map size | **L** | observed |
| OP-5 | Expose lower op_profile_limit via tool options | **L** | hypothesized |
| OP-6 | Require explicit host multi-host | **M** | observed |

**Top:** OP-1, OP-3.

---

## 9. roofline_model

### 9.1 End-to-end data path

```text
HTTP /data?tag=roofline_model[&host=...]
  → raw_to_tool_data.py:187-190
  → RooflineModelProcessor
       ConvertMultiXSpaceToCombinedOpStatsWithCache
       ConvertOpStatsToRooflineModel(combined, true)     # roofline_model_processor.cc:45-46
       ConvertOpStatsToRooflineModel(combined, false)    # :47-48
       MergeFrom records without-infeed into with-infeed # :49-50
       RooflineModelToDataTableJson
         SortedOpMetricsDb(db, kMaxNumRecords=1000)      # op_stats_to_roofline_model.cc:59, :209
  → FE frontend/app/components/roofline_model/
       dataTableRaw + dataTableProgram + dataTableOp
       + scatterDataProgram + scatterDataOp              # roofline_model.ts:84-107
       child analyses re-JSON / re-parse for pie charts  # operation_level_analysis.ts:137-145
```

### 9.2 ALL_HOSTS

- Not in ALL_HOSTS UI sets; empty host → all planes pitfall.  
- **Severity: M** (capped 1000) + FE multi-DataTable duplication.

### 9.3 Materialization notes

| Stage | What | Confidence |
|-------|------|------------|
| Dual convert | Two full roofline passes over same metrics | **observed** processor `:45-50` |
| Cap | Top 1000 ops | **observed** `kMaxNumRecords = 1000` |
| Merge | Records from both passes concatenated | **observed** |
| FE | dataTableRaw + filtered program/op + scatter copies → multiplies heap | **observed** |
| FE re-serialize | `JSON.parse(this.dataTable.toJSON())` for pie providers | **observed** `operation_level_analysis.ts:137-138` |

### 9.4 Opportunities

| ID | Opportunity | Hotspot | Confidence |
|----|-------------|---------|------------|
| RF-1 | Single pass with include_infeed flag as column/filter | **M** | observed |
| RF-2 | Avoid cloning full DataTables; use DataView filters | **M** | hypothesized |
| RF-3 | OpStatsOptions without kernel_stats if unused | **L** | hypothesized |
| RF-4 | Lower default kMaxNumRecords or progressive load | **L** | hypothesized |
| RF-5 | Stop toJSON/parse round-trip for child charts | **M** | observed |
| RF-6 | Require explicit host multi-host | **M** | observed |

**Top:** RF-1, RF-2.

---

## 10. Cross-tool comparison matrices

### 10.1 Wire bounds

| Tool | Intermediate cap | Wire row bound | Dual payload | always_print |
|------|------------------|----------------|--------------|--------------|
| overview_page | OpStats full | compact multi-table | + inference pass | no |
| input_pipeline_analyzer | OpStats full | unbounded steps | no | no |
| framework_op_stats | OpStats full | 500 host + 500 device | with/without idle | no |
| kernel_stats | OpStats full | **unbounded** reports | no | no |
| hlo_stats | OpStats full | **unbounded** HLO + children | no | no |
| op_profile | OpStats full | tree limit 100 | no | **yes** |
| roofline_model | OpStats full | 1000 | infeed true+false | no |

### 10.2 Convert cost vs UI need (hypothesized fit)

| Tool | Metrics DB | Step DB | Kernel DB | Fit score |
|------|------------|---------|-----------|-----------|
| overview_page | need | need | mostly no | poor (pays kernel) |
| input_pipeline_analyzer | partial | need | no | poor |
| framework_op_stats | need | no | GPU maybe | medium |
| kernel_stats | no | no | need | **worst** (pays metrics+step) |
| hlo_stats | need | no | no | poor |
| op_profile | need | no | no | poor |
| roofline_model | need | no | no | poor |

### 10.3 FE pageSize vs residency

| Tool | pageSize | Resident structure | Extra copies |
|------|----------|--------------------|--------------|
| kernel_stats | 100 | SimpleDataTable + gviz DataTable | filter toDataTable |
| hlo_stats | 10 | SimpleDataTable + multi chart providers | pie shares provider carefully (`hlo_stats.ts:496-498` comment) |
| framework_op_stats | varies | NgRx + clone | preProcessDataTable clone |
| input_pipeline | n/a charts | multi SimpleDataTable | detail toDataTable |
| overview_page | n/a | multi SimpleDataTable | per chart |
| op_profile | n/a tree | Map of full protos | per group_by |
| roofline_model | n/a | 5+ DataTables | toJSON/parse |

---

## 11. Measurement plan

Goal: turn **hypothesized** items into **observed** with numbers; prioritize H fixes with measured ROI.

### 11.1 Workloads

| ID | Workload | Why |
|----|----------|-----|
| W1 | Single-host GPU, short (≤50 steps) | Baseline wire sizes |
| W2 | Single-host GPU, long (≥1k steps, many unique kernels) | Unbounded KS + IP steps |
| W3 | Single-host TPU, large HLO (many ops, long expressions) | HS expression bloat |
| W4 | Multi-host 8×, medium | Vector OpStats peak (FAM-OS-1) |
| W5 | Multi-host 32×, medium | Scaling curve N·S |
| W6 | Non-training / inference session | Overview dual XSpace pass (OV-3) |
| W7 | Cold session open (warm-up) | overview + trace_viewer@ concurrency |
| W8 | Hot session, click all seven tools | Cache hit RSS still high? |

### 11.2 Server metrics

| Metric | How | Maps to |
|--------|-----|---------|
| Peak RSS of plugin process | `/proc` or macOS `footprint` around convert | FAM-OS-1, OV-5 |
| Per-stage timers | existing LOG(INFO) spans in multi_xplanes / processors | already partially instrumented |
| `sizeof` / `ByteSizeLong()` of OpStats | temporary debug counters (analysis branch only) | FAM-OS-2 field costs |
| Per-DB sizes | `op_metrics_db`, `step_db`, `kernel_stats_db` ByteSizeLong | KS-2 / IP-1 |
| JSON output bytes pre-gzip | `SetOutput` string size | wire caps |
| Gzipped Content-Length | HTTP | user-visible transfer |
| Concurrent convert count | wrap processor entry | FAM-OS-9 |
| Cache hit vs miss | existing LOG cache hit lines | amortization |

### 11.3 Client metrics

| Metric | How | Maps to |
|--------|-----|---------|
| JS heap after tool load | Chrome Performance memory | FAM-OS-8 |
| Object retained size of SimpleDataTable | heap snapshot | KS/HS |
| Number of google.visualization.DataTable instances | snapshot | RF-2 |
| opProfileDataCache Map entries | snapshot | OP-4 |
| Time to interactive after /data | Performance panel | UX coupling |

### 11.4 Experiments (analysis-only design; no product commits)

| Exp | Change (local branch) | Success signal |
|-----|----------------------|----------------|
| E1 | Stream-combine prototype (drop host after merge) | peak RSS multi-host ↓ ≥30% on W4/W5 |
| E2 | OpStatsOptions kernel_stats_db=false for overview | OpStats ByteSizeLong ↓; overview JSON identical |
| E3 | kernel_stats top-1000 wire | JSON bytes ↓; UI still useful |
| E4 | hlo_stats top-1000 + drop expression column | JSON bytes ↓ sharply on W3 |
| E5 | Cache InferenceStats to disk | second overview load no XSpace reread; W6 peak ↓ |
| E6 | always_print=false on op_profile | JSON bytes ↓; FE still renders |
| E7 | Single roofline pass | convert time ↓ ~2×; JSON same schema |
| E8 | Host required for hlo_stats multi-host | no accidental N-host convert |

### 11.5 Logging / probes already present (reuse)

| Probe | Location |
|-------|----------|
| ConvertMultiXSpacesToCombinedOpStats start/host/finish | `multi_xplanes_to_op_stats.cc:46-111` |
| Cache hit/miss WithCache | `:130-151` |
| Overview ProcessCombinedOpStats stages | `overview_page_processor.cc:49-76` |
| OpStats cache per host in legacy processor | `op_stats_processor.cc:84-91` |

### 11.6 Acceptance bar for “done measuring”

- W4/W5 peak RSS attributed: % in vector OpStats vs combined vs JSON.  
- Per-tool wire size table filled with real bytes (pre/post gzip).  
- FAM-OS-2: measured size of kernel_stats_db and step_db share of OpStats for W2.  
- HS expression: p50/p99 string length and contribution to JSON.  
- Document which opportunities move from hypothesized → observed.

### 11.7 Suggested measurement sequence

1. **Instrument sizes only** (ByteSizeLong + JSON length) — no algorithm change.  
2. **W1 vs W2 vs W3** wire table — confirm KS/HS unbounded.  
3. **W4 vector peak** — validate FAM-OS-1 priority.  
4. **W6 inference** — validate OV-3.  
5. **W7 warm-up concurrency** — validate FAM-OS-9.  
6. **Prototype E1/E3/E4** offline for ROI ranking.

---

## 12. Family priority order (recommended)

1. **FAM-OS-1** — incremental OpStats combine (multi-host peak).  
2. **FAM-OS-2** — per-tool generate flags (stop building unused DBs).  
3. **KS-1 / HS-1** — wire top-K for unbounded tables.  
4. **OV-3 / FAM-OS-10** — stop second XSpace pass on overview inference.  
5. **HS-2 / FAM-OS-4** — lazy long text fields.  
6. **FAM-OS-5** — host selection safety for non-ALL_HOSTS tools.  
7. **OP-1 / FO-1 / RF-1** — dual payload / always_print / dual roofline cleanup.  
8. **FAM-OS-8** — FE virtualization after wire is bounded.  
9. **FAM-OS-9** — warm-up concurrency with DEFAULT_CACHE_TOOLS.  
10. **IP-2** — step series downsample for long profiles.

### Phase sketch (analysis only)

| Phase | Focus | Expected peak lever |
|-------|-------|---------------------|
| P0 | Measure §11 | evidence |
| P1 | FAM-OS-1 + FAM-OS-2 | server multi-host RSS |
| P2 | KS-1, HS-1, HS-2 | wire + FE heap |
| P3 | OV-3, FAM-OS-10 | inference / overview |
| P4 | FAM-OS-5, FO-1, OP-1, RF-1 | policy + dual payloads |
| P5 | FAM-OS-8 FE | browser after wire fixed |

---

## 13. Severity & confidence legends

### Severity

- **H** — multi-host peak, unbounded wire, or default-path warm-up cost likely to OOM or multi-GB.  
- **M** — meaningful 2× waste or browser heap, fixed with moderate effort.  
- **L** — polish / secondary.

### Confidence

- **observed** — directly evidenced by code paths cited.  
- **hypothesized** — structural inference without measured heap profiles.

---

## 14. Paths cited

### Plugin / registry / dispatch

- `plugin/xprof/profile_plugin/tools/registry.py:33-70` — XPLANE_TOOLS, DEFAULT_CACHE_TOOLS, ALL_HOSTS sets  
- `plugin/xprof/profile_plugin/tools/filenames.py` — hosts_from_xplane_filenames  
- `plugin/xprof/profile_plugin/services/hosts.py:24-115` — HostSelector  
- `plugin/xprof/convert/raw_to_tool_data.py:146-190` — tool branches  

### Convert (shared OpStats)

- `xprof/convert/multi_xplanes_to_op_stats.cc:42-159` — vector TODO `:50-52`, always-full options `:121-124`, cache `:116-159`  
- `xprof/convert/base_op_stats_processor.cc:98-187` — Map full options `:111-115`, Reduce vector `:131-155`, ProcessSession `:164-171`, ShouldUseWorkerService `:174-187`  
- `xprof/convert/base_op_stats_processor.h:36-69` — Map/Reduce/ProcessCombinedOpStats interface  
- `xprof/convert/op_stats_processor.cc` — legacy parallel path / cache checks  
- `xprof/convert/xplane_to_op_stats.h:31-36` — OpStatsOptions defaults  
- `xprof/convert/op_metrics_to_record.h:33-34` — SortedOpMetricsDb max_records default -1  
- `xprof/convert/xplane_to_tools_data.cc` — alternate convert entry / op_profile always_print  
- `plugin/xprof/protobuf/op_stats.proto` — hub schema  

### Convert (per tool)

- `xprof/convert/overview_page_processor.cc:45-112`  
- `xprof/convert/op_stats_to_overview_page.{h,cc}`  
- `xprof/convert/multi_xspace_to_inference_stats.cc:117-165`  
- `xprof/convert/unified_overview_page_processor.cc`  
- `xprof/convert/input_pipeline_processor.cc:35-59`  
- `xprof/convert/op_stats_to_input_pipeline_analysis.{h,cc}`  
- `xprof/convert/framework_op_stats_processor.cc:44-57`  
- `xprof/convert/op_stats_to_tf_stats.cc:45`, `:125-128`, `:208-214`  
- `xprof/convert/kernel_stats_processor.cc:36-55`  
- `xprof/convert/xplane_to_kernel_stats_db.cc:98-156`  
- `xprof/convert/hlo_stats_processor.cc` / `unified_hlo_stats_processor.cc`  
- `xprof/convert/op_stats_to_hlo_stats.cc:88-237`  
- `xprof/convert/op_profile_processor.cc:41-92`  
- `xprof/convert/unified_op_profile_processor.cc`  
- `xprof/convert/op_stats_to_op_profile.{h,cc}`  
- `xprof/convert/op_profile_builder.cc:796-797`  
- `xprof/convert/roofline_model_processor.cc:39-70`  
- `xprof/convert/op_stats_to_roofline_model.cc:59`, `:209`, dual-use via processor  
- `xprof/convert/unified_tools_registration.cc`  

### Frontend

- `frontend/app/components/overview_page/` — multi DataTable tuple  
- `frontend/app/components/input_pipeline/` — `input_pipeline.ts`, detail data providers  
- `frontend/app/components/framework_op_stats/` — NgRx + `stats_table_data_provider.ts`  
- `frontend/app/components/kernel_stats/kernel_stats.ts`  
- `frontend/app/components/kernel_stats/kernel_stats_table/kernel_stats_table.ts:80-111`  
- `frontend/app/components/hlo_stats/hlo_stats.ts:129-142`, `:494-498`  
- `frontend/app/components/op_profile/op_profile.ts:43-100`  
- `frontend/app/components/roofline_model/roofline_model.ts:84-107`  
- `frontend/app/components/roofline_model/operation_level_analysis/operation_level_analysis.ts:137-145`  
- `frontend/app/services/data_dispatcher/data_dispatcher_base.ts`  
- `frontend/app/store/common_data_store/`  
- `frontend/app/store/framework_op_stats/`  

### Related specs

- `docs/superpowers/specs/memory-optimization/00-synthesis-memory-optimization.md`  
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md` (gzip, warm-up concurrency, ALL_HOSTS amplification)  
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (non-OpStats memory tools)  

---

## 15. Opportunity master index

| ID | Tool/Family | Severity | Confidence |
|----|-------------|----------|------------|
| FAM-OS-1 | family | H | observed |
| FAM-OS-2 | family | H | observed |
| FAM-OS-3 | family | H | observed |
| FAM-OS-4 | family | H | hypothesized |
| FAM-OS-5 | family | M | observed |
| FAM-OS-6 | family | M | hypothesized |
| FAM-OS-7 | family | M | observed |
| FAM-OS-8 | family | M | observed |
| FAM-OS-9 | family | H | hypothesized |
| FAM-OS-10 | family | H | observed |
| FAM-OS-11 | family | M | hypothesized |
| FAM-OS-12 | family | M | hypothesized |
| FAM-OS-13 | family | M | hypothesized |
| FAM-OS-14 | family | L | hypothesized |
| OV-1 … OV-9 | overview_page | H/M/L | mixed |
| IP-1 … IP-6 | input_pipeline_analyzer | H/M/L | mixed |
| FO-1 … FO-6 | framework_op_stats | M/L | mixed |
| KS-1 … KS-6 | kernel_stats | H/M/L | mixed |
| HS-1 … HS-7 | hlo_stats | H/M/L | mixed |
| OP-1 … OP-6 | op_profile | M/L | mixed |
| RF-1 … RF-6 | roofline_model | M/L | mixed |

---

## 16. Appendix A — Algorithm walk: multi-host OpStats combine

### A.1 ConvertMultiXSpacesToCombinedOpStats

Entry: `multi_xplanes_to_op_stats.cc:42-114`.

1. Log start with `XSpaceSize()` (`:46-48`).  
2. **TODO** acknowledge peak issue (`:50-52`).  
3. Reserve vector size N (`:53-54`).  
4. For i in 0..N-1 (`:55-71`):  
   - Arena-scoped XSpace load (`:60-61`).  
   - Preprocess step_grouping + derived_timeline (`:62-63`).  
   - `ConvertXSpaceToOpStats(*xspace, options)` (`:64-65`).  
   - **push_back** keeps OpStats alive for all i (`:66`).  
   - Arena destroyed → XSpace freed; OpStats retained.  
5. Build `OpStatsInfo` pointer vector (`:78-84`).  
6. `ComputeStepIntersectionToMergeOpStats(..., max())` (`:91-93`) — **no step cap**.  
7. `CombineAllOpStats` into `combined_op_stats` (`:105`).  
8. Vector destructs after return — peak includes N + combined until then.

### A.2 ConvertMultiXSpaceToCombinedOpStatsWithCache

Entry: `:116-159`.

1. Force full `OpStatsOptions` (`:121-124`).  
2. If ALL_HOSTS OP_STATS file exists → ReadBinaryProto into `combined_op_stats` (`:129-134`).  
3. Else run A.1 + WriteBinaryProto (`:140-151`).  
4. Return; caller still holds full combined proto in RAM.

### A.3 BaseOpStatsProcessor Map/Reduce

**Map** (`base_op_stats_processor.cc:98-121`):

1. If per-host cache exists → return path only (`:104-106`).  
2. Else copy XSpace, preprocess, full options Convert, WriteBinaryProto (`:108-120`).  
3. Does not hold other hosts — **good for per-host peak during Map**.

**Reduce** (`:124-161`):

1. Load **all** map_output OpStats into vector (`:131-137`) — **same N-way peak as A.1**.  
2. Pointer stability comment (`:142-144`).  
3. Unbounded step intersection (`:151-152`).  
4. Combine + write ALL_HOSTS (`:154-159`).  
5. `ProcessCombinedOpStats` while combined (+ possibly vector until end) live (`:161`).

**ProcessSession** (`:164-171`): WithCache + ProcessCombinedOpStats — single-process path for tools.

### A.4 Peak timeline sketch (multi-host cold)

```text
time →
Map host0: [XSpace0][OpStats0] → disk
Map host1: [XSpace1][OpStats1] → disk
...
Reduce:    [OpStats0|OpStats1|...|OpStatsN-1][combined][tool JSON]
            ◄──────── peak RSS typically here ────────►
ProcessSession cold:
  loop i:  [XSpace_i][OpStats_i] + [OpStats0..i-1 still in vector]
  combine: [all OpStats][combined]
  tool:    [combined][JSON]
```

Streaming combine (FAM-OS-1) would collapse the vector span to O(1) host OpStats + running merge.

---

## 17. Appendix B — Per-tool ProcessCombinedOpStats one-liners

| Tool | Processor method behavior | Output |
|------|---------------------------|--------|
| overview_page | ConvertOpStatsToOverviewPage; maybe InferenceStats; OverviewPageToJson | multi DataTable JSON |
| input_pipeline_analyzer | ConvertOpStatsToInputPipelineAnalysis → DataTableJson | multi table JSON |
| framework_op_stats | ConvertOpStatsToTfStats → TfStatsToDataTableJson | `[idle, no_idle]` |
| kernel_stats | KernelStatsToDataTableJson(kernel_stats_db) | single table JSON |
| hlo_stats | ConvertOpStatsToHloStats → HloStatsToDataTableJson | single table JSON |
| op_profile | ConvertOpStatsToOpProfile limit 100 → MessageToJsonString always_print | Profile JSON |
| roofline_model | Convert ×2 (infeed T/F) merge → RooflineModelToDataTableJson | table + diagnostics JSON |

---

## 18. Appendix C — Empty-host / ALL_HOSTS interaction cases

| Client request | Tool class | HostSelector result | Convert scope |
|----------------|------------|---------------------|---------------|
| host=ALL_HOSTS | SUPPORTED | all planes (`hosts.py:79-81`) | multi |
| host=ALL_HOSTS | ONLY | all planes | multi |
| host=H valid | any | one plane (`:82-84`) | single |
| host=H invalid | not ONLY | FileNotFoundError (`:90-92`) | — |
| host=H invalid | ONLY | may fall through | empty paths edge |
| no host | any with xplanes | **all planes** (`:94-97`) | multi **even if tool not SUPPORTED** |
| no host, no assets | not ONLY | FileNotFoundError require host (`:107-109`) | — |

**Implication for OP/HS/RF:** UI may send a host, but any code path omitting host multiplies cost. FAM-OS-5.

---

## 19. Appendix D — Why cache hit is not free RAM

On OpStats cache hit:

1. Full combined proto deserialized (`ReadBinaryProto`) — O(size of OpStats).  
2. Tool projection allocates new JSON string — O(wire).  
3. Gzip may allocate compressed buffer alongside (see 02).  
4. FE still holds full parse.

Cache eliminates **re-walk of XPlanes** and **recompute of metrics**, not **resident size of the hub proto**. Slimming OpStatsOptions at write time (FAM-OS-2) or projection-only caches (FAM-OS-6) address residual RAM.

---

## Log line

```
OVERVIEW_STATS|DONE|04-overview-and-stats-tools.md|OpStats fan-out family: multi-host vector combine TODO + always-full OpStatsOptions dominate; kernel/hlo unbounded wire; overview dual XSpace inference; FE pageSize presentation-only; measurement plan W1-W8
```
