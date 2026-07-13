# Memory Tools Family — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `memory_profile`, `memory_viewer` (UI + convert)  
**CLI:** `get_memory_profile_tool`, `get_peak_allocations_tool`  
**Companion:** family summary in `00-synthesis-memory-optimization.md`; convert/serve/cache shared path in `02-convert-serve-cache-path.md`

---

## Registry placement

Source: [`plugin/xprof/profile_plugin/tools/registry.py`](plugin/xprof/profile_plugin/tools/registry.py)

| Concern | Membership / policy |
|---------|---------------------|
| **XPLANE_TOOLS** | `memory_profile`, `memory_viewer` both listed |
| **HLO_TOOLS** | `memory_viewer` only (`frozenset(['graph_viewer', 'memory_viewer'])`) |
| **ALL_HOSTS_SUPPORTED / ONLY** | **Neither** memory tool — UI/host selection is per-host (or module list for viewer) |
| **DEFAULT_CACHE_TOOLS** | Not warmed (`overview_page`, `trace_viewer@` only) |
| **Single-XSpace enforce** | `MemoryProfileProcessor` rejects `XSpaceSize() != 1` |
| **Options builder** | `tools/options/memory.py` → registered only for `memory_viewer` |
| **Python convert** | [`plugin/xprof/convert/raw_to_tool_data.py`](plugin/xprof/convert/raw_to_tool_data.py) |
| **C++ processors** | `MemoryProfileProcessor`, `MemoryViewerProcessor` (`REGISTER_PROFILE_PROCESSOR`) |
| **C++ convert cores** | `xplane_to_memory_profile.cc`, `hlo_proto_to_memory_visualization_utils.cc` |
| **Protos** | `plugin/xprof/protobuf/memory_profile.proto`, `memory_viewer_preprocess.proto` |
| **FE** | `frontend/app/components/memory_profile/`, `memory_viewer/` |
| **CLI** | `plugin/xprof/cli/tools/get_memory_profile_tool.py`, `get_peak_allocations_tool.py` |

### Shared family short path

```text
UI / CLI
  ├─ memory_profile (single host)
  │    HostSelector → 1× XSpace
  │    → MemoryProfileProcessor
  │         Arena + GetXSpace + PreprocessSingleHostXSpace(step_grouping)
  │         → ConvertXSpaceToMemoryProfileJson
  │              host plane → GenerateMemoryProfile (ALL alloc/dealloc events)
  │              → ProcessMemoryProfileProto (sample ≤1000, peak active allocs)
  │              → MessageToJsonString(always_print_fields_with_no_presence=true)
  │    → gzip JSON → FE holds MemoryProfileProto object graph
  │
  └─ memory_viewer (module × memory_space)
       module list: memory_viewer.json without module → comma-separated names
       → GetHloProtoByOptions (disk *.hlo_proto.pb or extract from XSpace)
       → ConvertHloProtoToMemoryViewer / ConvertHloProtoToPreprocessResult
            heap simulator walk → program-order vectors + dual max_heap*
            optional DOT/HTML allocation timeline
       → MessageToJsonString (default options) | text/html timeline
       → FE MemoryUsage re-sorts maxHeap client-side; charts hold full arrays
```

**Observed family invariant:** UI-scale protos/JSON are the only convert outputs. CLI tools that need a handful of scalars (`peak_bytes`, `total_hbm_mib`, top buffers) still force full UI materialization (and for peak allocations, **once per module**).

---

## 1. memory_profile

### End-to-end data path

```text
HTTP /data?tag=memory_profile&host=H
  → raw_to_tool_data.py: assert len(xspace_paths) == 1
  → xspace_wrapper / ProfileProcessor path
  → MemoryProfileProcessor::ProcessSession
       google::protobuf::Arena
       SessionSnapshot::GetXSpace(0, &arena)
       PreprocessSingleHostXSpace(xspace, step_grouping=true, derived_timeline=false)
       ConvertXSpaceToMemoryProfileJson(*xspace, &json)
         FindPlaneWithName(kHostThreadsPlaneName)
         ConvertXPlaneToMemoryProfile(*host_plane, max_num_snapshots=1000)  // default
           GenerateMemoryProfile: every kMemoryAllocation / kMemoryDeallocation XEvent
             → MemoryProfileSnapshot {time, aggregation_stats, activity_metadata}
             UpdateProfileSummary peak tracking per allocator
           ProcessMemoryProfileProto:
             sort snapshots by time_offset_ps
             UpdateStepId / UpdateDeallocation (address→alloc metadata map)
             SampleMemoryProfileTimeline(max_num_snapshots) → sampled_timeline_snapshots
             GetPeakMemoryStep + ProcessActiveAllocations
             SaveActiveAllocationSnapshots (rewrite memory_profile_snapshots to peak-only refs)
           version = 1
         GenerateHloModules(xspace) from device kXlaModuleLineName events
         ConvertProtoToJson(always_print_fields_with_no_presence=true)
  → SetOutput(json, application/json)
  → respond gzip
  → FE MemoryProfile.parseData → MemoryProfileBase holds full MemoryProfileProto
       timeline graph reads sampledTimelineSnapshots
       breakdown table reads activeAllocations + memoryProfileSnapshots / specialAllocations
```

**Sources:**  
`xprof/convert/memory_profile_processor.cc`,  
`xprof/convert/xplane_to_memory_profile.{h,cc}` (`max_num_snapshots = 1000` default),  
`plugin/xprof/protobuf/memory_profile.proto`,  
`frontend/app/components/memory_profile/`.

### Host / multi-host policy

| Policy | Behavior | Confidence |
|--------|----------|------------|
| Processor | `XSpaceSize() != 1` → `InvalidArgumentError` | **observed** |
| Registry | Not in ALL_HOSTS sets | **observed** |
| FE | `numHosts` + `host_id` query param; host plane still single-XSpace convert | **observed** (`memory_profile.ts`) |
| CLI multi-host | `get_memory_profile` falls back: fetch without host, then iterate `get_hosts` until capacity+peak valid | **observed** |

**Severity on multi-host sessions:** each host convert is independent; worst case CLI walks N hosts serially, each paying full convert. UI typically one host at a time.

### Materialization table

| # | Stage | Representation | Peak / waste notes | Confidence |
|---|-------|----------------|--------------------|------------|
| M1 | XSpace load | Full host XSpace in Arena | Dominant input size | **observed** |
| M2 | `GenerateMemoryProfile` | **All** alloc/dealloc events as `MemoryProfileSnapshot` (per allocator) | **O(events)** intermediate before prune; each snapshot carries full `MemoryActivityMetadata` strings (tf_op, shape, dtype, region) | **observed** |
| M3 | `UpdateDeallocation` | `flat_hash_map<address, MemoryActivityMetadata*>` over all snapshots | Extra map; copies string fields onto free events | **observed** |
| M4 | Sort | In-place sort of full snapshot list | Temporary residency of unsorted+sorted | **observed** |
| M5 | `SampleMemoryProfileTimeline` | Copies ≤1000 **full** snapshots into `sampled_timeline_snapshots` (max-box filter) | **Still includes `activity_metadata` on every sample** though timeline chart primarily needs aggregation_stats + time | **observed** |
| M6 | Peak active path | `ProcessActiveAllocations` then `SaveActiveAllocationSnapshots` shrinks `memory_profile_snapshots` to only indices referenced by `active_allocations` | Good: full timeline not kept in field 1 after process; **but** both full list and sampled list coexist until end of `ProcessMemoryProfileProto` | **observed** |
| M7 | JSON | `MessageToJsonString` with `always_print_fields_with_no_presence = true` | Emits empty/default fields; string bloat on every snapshot | **observed** (`ConvertProtoToJson` in `xplane_to_memory_profile.cc`) |
| M8 | Wire + FE | Gzipped JSON → parsed `MemoryProfileProto` in Angular | Component holds full map `memoryProfilePerAllocator` for all allocators even when UI shows one | **observed** (`MemoryProfileBase.data`) |
| M9 | HLO modules list | `repeated HloModule` name + time range from device planes | Usually small vs snapshots | **observed** |
| M10 | Options | `max_num_snapshots` **not** plumbed through processor/options/`memory.py` | Always hardcoded 1000 default on public API | **observed** |

### Proto field residency (post-process, per allocator)

From `memory_profile.proto` `PerAllocatorMemoryProfile`:

| Field | Post-process content | UI consumer |
|-------|----------------------|-------------|
| `memory_profile_snapshots` | Peak-referenced snapshots only (after `SaveActiveAllocationSnapshots`) | Breakdown table (via `ActiveAllocation.snapshot_index`) |
| `sampled_timeline_snapshots` | ≤1000 full snapshots | Timeline area chart |
| `active_allocations` | Merged peak rows (`num_occurrences`) | Breakdown table |
| `special_allocations` | Unmapped heap + stack pseudo-rows | Breakdown table |
| `profile_summary` | Peak stats + capacity + lifetime peak | Summary panel |

### Severity assessment

| Scenario | Peak driver | Severity |
|----------|-------------|----------|
| Short profile, few alloc events | XSpace + JSON | **L–M** |
| Long training window, dense allocator events | **M2 full snapshot list** before sample; then M5+M7 JSON of 1000 full samples × allocators | **H** |
| Many allocators (multi-device) | Map of PerAllocator profiles each with own sampled timeline | **H** |
| CLI scalar summary only | Same as UI convert (no summary mode) | **H** waste relative to need |

### Opportunities

| ID | Opportunity | Hotspot | Severity | Confidence |
|----|-------------|---------|----------|------------|
| MP-1 | Two-pass (or streaming) peak tracking without storing every event as a full `MemoryProfileSnapshot`; keep only peak bookkeeping + reservoir for timeline | M2 intermediate | **H** | hypothesized |
| MP-2 | Slim timeline samples: emit **stats-only** snapshots (time + aggregation_stats); drop `activity_metadata` from `sampled_timeline_snapshots` | M5 wire/FE | **H** | **observed** (timeline FE uses stats; metadata needed only for peak table via separate field) |
| MP-3 | Disable `always_print_fields_with_no_presence` for memory_profile JSON (or use compact print options) | M7 | **M** | **observed** |
| MP-4 | Plumb `max_num_snapshots` via `ToolOptions` / `memory.py` (and CLI) so dense sessions can drop to e.g. 256 | M5, options gap | **M** | **observed** gap |
| MP-5 | Per-allocator lazy convert: only process selected `memory_id` (FE already selects one) | M8 multi-allocator | **M** | hypothesized |
| MP-6 | String interning / dictionary encoding for repeated `tf_op_name`, `tensor_shape`, `data_type` in peak snapshots | M2/M6 string dupes | **M** | hypothesized |
| MP-7 | Binary protobuf or column-oriented timeline on wire instead of JSON doubles/strings | M7/M8 | **M** | hypothesized |
| MP-8 | Free unused allocator maps on FE when selection changes; avoid holding full proto if only one allocator shown | M8 | **L** | hypothesized |
| MP-9 | Skip `GenerateHloModules` when FE does not need module overlay (option) | M9 | **L** | hypothesized |
| MP-10 | Avoid double-resident full snapshot list: sample timeline during generate pass, never build O(events) `RepeatedPtrField` | M2+M5 | **H** | hypothesized |

**Top:** MP-2, MP-1/MP-10, MP-3, MP-4.

---

## 2. memory_viewer

### End-to-end data path

```text
UI memory_viewer
  → getModuleList(session)  // memory_viewer.json, no module → comma names
  → getDataByModuleNameAndMemorySpace(memory_viewer, module, color)
  → raw_to_tool_data.py options:
       module_name, program_id, view_memory_allocation_timeline, memory_space
  → MemoryViewerProcessor::ProcessSession
       GetHloProtoByOptions(session, options)   // may load large HloProto
       MemoryViewerOption {
         memory_color, small_buffer_size=16KiB,
         timeline_option.render_timeline / timeline_noise
       }
       ConvertHloProtoToMemoryViewer → ConvertHloProtoToPreprocessResultJson
         HloProtoBufferWrapper: index logical buffers, allocations, traces
         ProcessHeapSimulatorTrace(memory_color):
           for each HeapSimulatorTrace::Event:
             UpdateOnSimulatorEvent → push heap_size, unpadded, hlo_instruction_name
             ALLOC/FREE/SHARE_WITH with std::list active buffers (O(n) remove on FREE)
             snapshot peak_logical_buffers at max heap
         CreatePeakUsageSnapshot: indefinite buffers + peak heap objects
         GeneratePreprocessResult:
           dual max_heap + max_heap_by_size + index maps
           heap_sizes / unpadded_heap_sizes / hlo_instruction_names size = #events+1
           logical_buffer_spans map
           NoteSpecialAllocations (indefinite_lifetimes tree)
         ConvertAllocationTimeline (if render): DOT rects up to graph 4096×4096
         MessageToJsonString(result)  // no always_print flag here
         OR RenderTimelineGraph(DOT) → HTML when view_memory_allocation_timeline
  → content_type application/json | text/html
  → FE MemoryViewer holds preprocess result
       MemoryUsage.createMaxHeapIndex() RE-SORTS maxHeap by size and padding
       (ignores wire maxHeapBySize / index maps for primary charts)
       program_order_chart: full heapSizes + hloInstructionNames arrays
```

**Sources:**  
`xprof/convert/memory_viewer_processor.cc`,  
`xprof/convert/hlo_to_tools_data.cc`,  
`xprof/convert/hlo_proto_to_memory_visualization_utils.{h,cc}`,  
`plugin/xprof/protobuf/memory_viewer_preprocess.proto`,  
`frontend/app/components/memory_viewer/memory_usage/memory_usage.ts`.

### Host / module policy

| Policy | Behavior | Confidence |
|--------|----------|------------|
| Registry | ∈ HLO_TOOLS + XPLANE_TOOLS | **observed** |
| Module list | Separate fetch returns CSV of module names (HLO protos on disk / extract) | **observed** (`memory_viewer.ts`, peak CLI `_get_module_names`) |
| Per request | One module × one `memory_space` color | **observed** |
| Timeline option | `view_memory_allocation_timeline` only option in `build_memory_family` | **observed** (`tools/options/memory.py`) |
| `timeline_noise` | Supported in C++ options / processor; not in Python options builder | **observed** gap |

### Materialization table

| # | Stage | Representation | Peak / waste notes | Confidence |
|---|-------|----------------|--------------------|------------|
| V1 | HloProto load | Full module buffer assignment + traces | Dominant input; shared with graph_viewer extract cache | **observed** |
| V2 | `HloProtoBufferWrapper` | Maps id→`LogicalBufferStruct` / `BufferAllocationStruct` with shapes | Full index of all buffers for color; coexists with proto | **observed** |
| V3 | Heap sim timelines | `vector<int64>` ×2 + `vector<string>` instruction names, length ≈ **#sim events** | **Dominates wire** for large modules; strings often repeat op names | **observed** (`HeapSimulatorStats`) |
| V4 | Active set | `std::list<int64_t> logical_buffers` + copy `peak_logical_buffers` | FREE uses `list::remove` → **O(n)** per free; peak copy clones entire list | **observed** |
| V5 | Peak heap objects | `vector<HeapObject>` at peak (+ small-buffer collapse <16KiB) | Includes label, shape, tf_op, opcode, source_info | **observed** |
| V6 | Dual sorted max_heap | Proto emits `max_heap` **and** `max_heap_by_size` + both index arrays | **Dead weight on wire**: FE `createMaxHeapIndex` re-sorts from `maxHeap` only | **observed** (`memory_usage.ts` lines 182–214; C++ `GeneratePreprocessResult` 1079–1115) |
| V7 | `logical_buffer_spans` | map buffer_id → {start,limit} for all seen buffers | Useful for charts; size O(buffers) | **observed** |
| V8 | `indefinite_lifetimes` | Nested `BufferAllocation` with `LogicalBuffer` children | Extra detail tree on every JSON | **observed** |
| V9 | Allocation timeline | DOT string of rects (`graph_width/height = 1<<12`) → HTML via `WrapDotInHtml` when enabled | **Huge** optional path; separate full convert | **observed** (`ConvertAllocationTimeline`, `RenderTimelineGraph`) |
| V10 | JSON encode | Full `PreprocessResult` via `MessageToJsonString` | Doubles as text; long repeated strings | **observed** |
| V11 | FE MemoryUsage | Clones maxHeap into by-size and by-padding arrays | **Third** sort order (padding) never on wire — good; but full arrays in component forever | **observed** |
| V12 | FE program order chart | Holds full `heapSizes` / names arrays | No downsample | **observed** (`program_order_chart`) |

### Severity assessment

| Scenario | Peak driver | Severity |
|----------|-------------|----------|
| Small module, short sim trace | HloProto + modest JSON | **L–M** |
| Large module, long heap_simulator_traces | V3 vectors + V6 dual heap + V10 JSON | **H** |
| Timeline HTML enabled | V9 DOT/HTML on top of preprocess | **H–Critical** for huge traces |
| Switching memory space / module | Full re-convert each time (no durable preprocess cache beyond generic tool cache) | **H** multi-view sessions |

### Opportunities

| ID | Opportunity | Hotspot | Severity | Confidence |
|----|-------------|---------|----------|------------|
| MV-1 | Drop `max_heap_by_size`, `max_heap_to_by_size`, `by_size_to_max_heap` from wire (FE already re-sorts) | V6 | **H** | **observed** |
| MV-2 | Downsample program-order timelines (max-box or stride) for UI; keep peak position accurate | V3 | **H** | hypothesized |
| MV-3 | Lazy / optional `hlo_instruction_names` (fetch on hover, or index into shared string table) | V3 | **H** | hypothesized |
| MV-4 | Replace `std::list` + `remove` with vector + swap-erase or freelist; avoid O(n) FREE | V4 CPU/allocator churn | **M** | **observed** |
| MV-5 | Slim `HeapObject`: omit redundant `label` if FE builds from name+shape; optional source_info | V5 | **M** | hypothesized |
| MV-6 | Durable cache of `PreprocessResult` keyed by `(module, memory_color, small_buffer_size, version)` on disk | Re-open / CLI reuse | **H** | hypothesized |
| MV-7 | Bound timeline HTML/DOT: max rects, progressive render, or skip sub-pixel only is already partial — add hard cap + summary | V9 | **M** | **observed** partial (sub-pixel skip) |
| MV-8 | Peak-only convert mode: skip program-order vectors and spans; emit peak scalars + max_heap only | CLI / summary UI | **H** | hypothesized (needed by GPA) |
| MV-9 | Plumb `timeline_noise`, `small_buffer_size` through `memory.py` | options completeness | **L** | **observed** gap |
| MV-10 | Avoid building dual index maps in C++ when wire drops them (MV-1) | V6 CPU | **M** | **observed** once MV-1 lands |
| MV-11 | FE: release previous module preprocess when module changes | V11 | **L** | hypothesized |
| MV-12 | Column-pack heap_sizes as binary blob / base64 float32 | V3/V10 | **M** | hypothesized |

**Top:** MV-1, MV-2/MV-3, MV-8, MV-6, MV-4.

---

## 3. CLI get_memory_profile_tool

### End-to-end data path

```text
get_memory_profile(session_id)  [@decorators.cached expire=86400]
  → CachedXprofClient.fetch(tool_name="memory_profile.json", format=json[, host])
  → full UI memory_profile convert + JSON (same as section 1)
  → _parse_profile_data: walk formats → MemoryProfileMetrics scalars only:
       memory_capacity, peak_hbm_bytes, stack, heap, free, fragmentation
  → emit ~6 GiB/% fields JSON
  Multi-host fallback:
       if peak/capacity invalid → client.get_hosts → for each host re-fetch full JSON
```

**Source:** [`plugin/xprof/cli/tools/get_memory_profile_tool.py`](plugin/xprof/cli/tools/get_memory_profile_tool.py).

### Materialization table

| # | Stage | Representation | Notes | Confidence |
|---|-------|----------------|-------|------------|
| C1 | Convert | Full MemoryProfile JSON (sampled timelines, active allocs, modules) | **Orders of magnitude larger than output** | **observed** |
| C2 | Parse | `json.loads` entire string then read `memoryProfilePerAllocator.*.profileSummary` | Transient full object graph in Python | **observed** |
| C3 | Output | ~few hundred bytes scalar JSON | Good | **observed** |
| C4 | Cache | SQLite decorator 86400s on **scalar** result | Convert still paid cold; warm CLI hits are cheap | **observed** (`@decorators.cached`) |
| C5 | Multi-host | Serial full converts until first “valid” host | Worst-case N× C1 | **observed** |

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GMP-1 | `summary_only` / `view=summary` convert mode: peak stats per allocator only, no snapshots/timeline | **H** | **observed** mismatch need vs payload |
| GMP-2 | Server-side host preference: pick first host with memory events without returning full JSON per attempt | **M** | hypothesized |
| GMP-3 | Smarter multi-host short-circuit (parallel first-success; or metadata probe) | **M** | hypothesized |
| GMP-4 | Parse streaming JSON / protobuf binary for summary fields only | **L** | hypothesized |
| GMP-5 | Cache intermediate full JSON keyed by session×host to share with UI (family cache) | **M** | hypothesized |

**Top:** GMP-1 (enables FAM-1), then GMP-3.

---

## 4. CLI get_peak_allocations_tool

### End-to-end data path

```text
get_peak_allocations(session_id, limit=10, min_size_mib=1.0, …)
  [@decorators.cached expire=86400]
  → _get_module_names:
       fetch memory_viewer.json (no module) → CSV module list
  → _fetch_modules_data: **for each module**
       fetch memory_viewer.json?module_name=M
         → full ConvertHloProtoToPreprocessResult JSON
       parse totalBufferAllocationMib
       _parse_and_aggregate_buffers:
         prefer bufferAssignment.logicalBuffers (often absent in preprocess JSON)
         else maxHeap[] → aggregate name.* / Others < min_size
  → heapq.nlargest(limit) by total_hbm_mib
  → markdown or compact modules JSON
```

**Source:** [`plugin/xprof/cli/tools/get_peak_allocations_tool.py`](plugin/xprof/cli/tools/get_peak_allocations_tool.py).

### Critical observation: N+1 full converts

| Step | Convert cost | Needed fields |
|------|--------------|---------------|
| Module list | Light (names) | names |
| Module *i* of N | **Full** heap sim + program-order vectors + dual max_heap + JSON | `totalBufferAllocationMib` + peak buffer list (`maxHeap` or logical buffers) |
| Aggregate | Client-side only | top-K modules |

If a session has 50 modules, the tool performs **1 list + 50 full memory_viewer converts** before ranking. Limit only truncates **output**, not work.

### Materialization table

| # | Stage | Representation | Notes | Confidence |
|---|-------|----------------|-------|------------|
| P1 | Module list fetch | CSV string | Cheap | **observed** |
| P2 | Per-module convert | Full `PreprocessResult` JSON | **N× UI path** | **observed** loop in `_fetch_modules_data` |
| P3 | Python parse | Full dict per module | Discards heap_sizes / spans / dual indices | **observed** |
| P4 | Aggregation | Instruction name regex + size buckets | CPU modest vs convert | **observed** |
| P5 | Output cache | Scalar-ish ranked list 86400s | Cold start catastrophic | **observed** |

### Fallback parsing detail

- Prefers `bufferAssignment.logicalBuffers` — **not** a field on `PreprocessResult` proto (memory_viewer preprocess does not emit classic buffer assignment dump).  
- Practical path: **`maxHeap`** (`logicalBufferSizeMib`, `instructionName`).  
- So even peak-buffer listing depends on peak heap objects already built for UI bar chart — still pays full sim + timelines to get there.

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GPA-1 | Stop serial full convert per module: batch peak mode or multi-module API | **Critical** | **observed** |
| GPA-2 | `view=peak_buffers` convert: peak_heap_mib + max_heap (+ totals), **no** program-order vectors / dual sorts / spans | **Critical** | **observed** |
| GPA-3 | Two-phase: cheap per-module **total only** rank, then deep-dive top-K modules | **H** | hypothesized |
| GPA-4 | Reuse disk HloProto + preprocess cache across modules (MV-6) so repeated CLI/UI share | **H** | hypothesized |
| GPA-5 | Parallel module fetches with concurrency bound | **M** | hypothesized (reduces latency, not total RAM if unbounded) |
| GPA-6 | Emit max_heap-only option aligned with FE bar chart (still skip timelines) | **H** | hypothesized |

**Top:** GPA-1 + GPA-2 (family-critical).

---

## 5. Frontend surfaces (memory family)

### memory_profile FE

| Component | Path | Holds | Notes | Confidence |
|-----------|------|-------|-------|------------|
| Shell | `memory_profile.ts` | Full `MemoryProfileProto` | Host id loop from `numHosts`; no release on leave beyond destroy | **observed** |
| Base | `memory_profile_base.ts` | `data`, `memoryIds`, selection | All allocators in `data` | **observed** |
| Timeline | `memory_timeline_graph/` | Reads `sampledTimelineSnapshots` + `hloModules` | Google AreaChart; full sample series | **observed** |
| Breakdown | `memory_breakdown_table/` | Active allocations at peak | Indexes into peak snapshots | **observed** |
| Summary | `memory_profile_summary/` | Peak stats | Small | **observed** |

### memory_viewer FE

| Component | Path | Holds | Notes | Confidence |
|-----------|------|-------|-------|------------|
| Shell | `memory_viewer.ts` | `memoryViewerPreprocessResult`, module list | Module switch reloads full preprocess | **observed** |
| Main | `memory_viewer_main.ts` | Charts + MemoryUsage | Orchestrates controls | **observed** |
| MemoryUsage | `memory_usage.ts` | heap arrays + maxHeap ×3 sort orders | **Ignores wire maxHeapBySize** | **observed** |
| Program order | `program_order_chart.ts` | Full series | No downsample | **observed** |
| Max heap chart | `max_heap_chart.ts` | Bar chart from MemoryUsage | Peak buffers | **observed** |
| Buffer details | `buffer_details.ts` | Selected buffer | Small | **observed** |
| Timeline link | MemoryUsage builds URL with `view_memory_allocation_timeline=true` | Triggers **second** convert (HTML) | **observed** |

### FE opportunities (cross-tool)

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| FE-MP-1 | Request slim timeline schema when backend supports MP-2 | **H** | hypothesized (depends on MP-2) |
| FE-MV-1 | Stop depending on dual max_heap fields (already true) — delete dead proto fields safely | **H** | **observed** |
| FE-MV-2 | On module change, null previous preprocess before new fetch completes | **L** | hypothesized |
| FE-FAM-1 | Do not keep tool JSON in parent store after navigating away | **M** | hypothesized |

---

## Family shared opportunities

| ID | Opportunity | Severity | Confidence | Touches |
|----|-------------|----------|------------|---------|
| FAM-1 | Convert **view modes**: `full` \| `ui` \| `summary` \| `peak_buffers` (and timeline as separate mode) | **Critical** | **observed** (CLI forces full) | processors, options, CLI, optional FE |
| FAM-2 | On-disk result cache keyed by `tool × module × memory_space × view × version` (beyond generic session cache) | **H** | hypothesized | convert/repository |
| FAM-3 | JSON bloat policy: no `always_print` by default; optional compact; prefer binary for large arrays | **H** | **observed** (MP always_print; MV full vectors) | both tools |
| FAM-4 | Shared HloProto extract already helps memory_viewer + graph_viewer; ensure peak CLI reuses same files without re-extract | **M** | **observed** path exists | HLO extract |
| FAM-5 | Complete `memory.py` options: `max_snapshots`, `summary_only`/`view`, `small_buffer_size`, `timeline_noise` | **M** | **observed** gap | options registry |
| FAM-6 | CLI must not force UI-scale protos — call summary/peak modes only | **H** | **observed** | CLI + FAM-1 |
| FAM-7 | Document / enforce single-host for memory_profile in HostSelector UX (avoid accidental multi-path) | **L** | hypothesized | UI hosts |
| FAM-8 | Metrics: log convert peak RSS, JSON bytes, event counts for memory tools | **M** | hypothesized | observability |
| FAM-9 | Unified memory analysis lab path (`labs/curated/memory_analysis`) — keep out of prod until it replaces dual tools; avoid third full materializer | **L** | hypothesized | labs |

### Family priority order

1. **FAM-1 / GPA-2 / GMP-1** — summary & peak_buffers convert modes so CLI (and light UI) stop paying UI-scale protos.  
2. **MV-1 + MP-2** — delete dead/duplicate wire fields (dual max_heap; timeline metadata).  
3. **GPA-1** — eliminate N× full memory_viewer converts (batch or two-phase rank).  
4. **FAM-2 / MV-6** — durable preprocess cache per module×space.  
5. **MV-4 + MP-1/MP-10** — intermediate residency & list churn during convert.  
6. **FAM-3 / MP-3** — JSON print policy.  
7. **FAM-5 / MP-4** — options plumbing.  
8. **MV-7** — bound optional timeline HTML.

---

## Cross-links to other design notes

| Topic | Doc |
|-------|-----|
| Shared convert → cache → gzip → FE peak | `02-convert-serve-cache-path.md` |
| HLO extract shared with graph_viewer | `05-graph-hlo-util-tools.md` |
| CLI decorator cache pattern | `06-megascale-and-cli-detectors.md` |
| Portfolio prioritization | `00-synthesis-memory-optimization.md` |

---

## Appendix A — Proto field → consumer map

### MemoryProfile (runtime)

| Proto path | Primary consumer | Needed for CLI summary? |
|------------|------------------|-------------------------|
| `memory_profile_per_allocator[].profile_summary` | FE summary, CLI | **Yes** |
| `...sampled_timeline_snapshots[]` | FE timeline | No |
| `...memory_profile_snapshots[]` (peak refs) | FE breakdown | No |
| `...active_allocations[]` | FE breakdown | No |
| `...special_allocations[]` | FE breakdown | No |
| `memory_ids`, `num_hosts`, `version` | FE selectors | Partial |
| `hlo_modules[]` | FE timeline overlay | No |

### PreprocessResult (static HLO)

| Proto path | Primary consumer | Needed for peak CLI? |
|------------|------------------|----------------------|
| `peak_heap_mib`, `peak_unpadded_heap_mib`, `peak_heap_size_position` | FE header | Optional |
| `total_buffer_allocation_mib`, `indefinite_*`, params/liveout | FE header, CLI total | **Yes (total)** |
| `max_heap[]` | FE bar chart, CLI buffers | **Yes (buffers)** |
| `max_heap_by_size[]` + index maps | **None (FE re-sorts)** | No |
| `heap_sizes[]`, `unpadded_heap_sizes[]`, `hlo_instruction_names[]` | FE program-order chart | No |
| `logical_buffer_spans` | FE buffer lifespan | No |
| `indefinite_lifetimes[]` | FE details | No |
| `allocation_timeline` | Timeline HTML path / URL stub | No |
| `max_scoped_vmem_*` | FE VMEM header | No |

---

## Appendix B — Code anchors (absolute under repo root)

| Area | Path |
|------|------|
| Registry | `plugin/xprof/profile_plugin/tools/registry.py` |
| Options | `plugin/xprof/profile_plugin/tools/options/memory.py` |
| Options registry | `plugin/xprof/profile_plugin/tools/options/registry.py` |
| Python convert branch | `plugin/xprof/convert/raw_to_tool_data.py` |
| Memory profile processor | `xprof/convert/memory_profile_processor.cc` |
| XPlane → profile | `xprof/convert/xplane_to_memory_profile.cc` |
| Memory viewer processor | `xprof/convert/memory_viewer_processor.cc` |
| HLO → preprocess | `xprof/convert/hlo_proto_to_memory_visualization_utils.cc` |
| HLO tool glue | `xprof/convert/hlo_to_tools_data.cc` |
| Protos | `plugin/xprof/protobuf/memory_profile.proto`, `memory_viewer_preprocess.proto` |
| FE profile | `frontend/app/components/memory_profile/` |
| FE viewer | `frontend/app/components/memory_viewer/` |
| FE MemoryUsage re-sort | `frontend/app/components/memory_viewer/memory_usage/memory_usage.ts` |
| CLI summary | `plugin/xprof/cli/tools/get_memory_profile_tool.py` |
| CLI peaks | `plugin/xprof/cli/tools/get_peak_allocations_tool.py` |
| CLI cache decorator | `plugin/xprof/cli/internal/decorators.py` |

---

## Appendix C — Severity legend (this doc)

| Tag | Meaning |
|-----|---------|
| **Critical** | Orders-of-magnitude waste or N× full converts on common CLI/UI paths |
| **H** | Dominant peak or wire bloat on large real profiles |
| **M** | Meaningful savings; secondary to Critical/H |
| **L** | Nice-to-have / small absolute win |

**Confidence:**  
- **observed** — behavior verified in cited source on 2026-07-13.  
- **hypothesized** — consistent with architecture; not instrumented with production traces in this analysis.

---

## Appendix D — Suggested acceptance probes (analysis-only; not implemented)

1. Instrument `ConvertXSpaceToMemoryProfileJson`: log `original_snapshot_count`, `sampled_count`, `json_bytes`, peak RSS.  
2. Instrument `ConvertHloProtoToPreprocessResultJson`: log `sim_events`, `heap_sizes_len`, `max_heap_len`, `json_bytes`, fraction of JSON from `hlo_instruction_names`.  
3. Time `get_peak_allocations` vs module count; confirm linear full converts.  
4. Diff wire size with/without dual `max_heap_by_size` (MV-1 dry-run).  
5. Diff sampled timeline JSON with activity_metadata stripped (MP-2 dry-run).

These probes validate ranking before any product change.

