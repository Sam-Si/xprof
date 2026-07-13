# Memory Tools Family — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `memory_profile`, `memory_viewer` (UI + convert)  
**CLI:** `get_memory_profile_tool`, `get_peak_allocations_tool`  
**Companion:** family summary in `00-synthesis-memory-optimization.md`; convert/serve/cache shared path in `02-convert-serve-cache-path.md`

---

## How to read this document

| Convention | Meaning |
|------------|---------|
| Severity **H** | High: multi-MiB intermediate/wire or N× full convert for scalar need |
| Severity **M** | Medium: measurable waste, bounded or secondary path |
| Severity **L** | Low: small absolute, or optional path only |
| Confidence **observed** | Confirmed by code path with `path:line` cites |
| Confidence **hypothesized** | Plausible from structure; needs measurement |
| Path cites | Repo-relative `file:line` or `file:start-end` |
| Dead wire | Proto field emitted on wire; FE consumer = **NONE** |

Scope is **memory family only**: runtime memory profile (XPlane host alloc events) and static HLO memory viewer (buffer assignment / heap simulator). No product code changes in this note.

---

## Registry placement

Source: [`plugin/xprof/profile_plugin/tools/registry.py`](plugin/xprof/profile_plugin/tools/registry.py)

| Concern | Membership / policy | Cite |
|---------|---------------------|------|
| **XPLANE_TOOLS** | `memory_profile`, `memory_viewer` both listed | `registry.py:33-52` |
| **HLO_TOOLS** | `memory_viewer` only (`frozenset(['graph_viewer', 'memory_viewer'])`) | `registry.py:73` |
| **ALL_HOSTS_SUPPORTED / ONLY** | **Neither** memory tool | `registry.py:58-70` |
| **DEFAULT_CACHE_TOOLS** | Not warmed (`overview_page`, `trace_viewer@` only) | `registry.py:55` |
| **Single-XSpace enforce** | `MemoryProfileProcessor` rejects `XSpaceSize() != 1` | `memory_profile_processor.cc:41-45` |
| **Options builder** | `tools/options/memory.py` → registered **only** for `memory_viewer` | `options/registry.py:17-22`, `memory.py:11-16` |
| **Python convert** | [`plugin/xprof/convert/raw_to_tool_data.py`](plugin/xprof/convert/raw_to_tool_data.py) | `raw_to_tool_data.py:163-168`, `:211-225` |
| **C++ processors** | `MemoryProfileProcessor`, `MemoryViewerProcessor` (`REGISTER_PROFILE_PROCESSOR`) | `memory_profile_processor.cc:62`, `memory_viewer_processor.cc:101` |
| **Unified path** | `UnifiedMemoryViewerProcessor` (HLO session snapshot) | `unified_memory_viewer_processor.cc:33-85` |
| **C++ convert cores** | `xplane_to_memory_profile.cc`, `hlo_proto_to_memory_visualization_utils.cc` | headers default `max_num_snapshots=1000`, `kSmallBufferSize=16KiB` |
| **Protos** | `plugin/xprof/protobuf/memory_profile.proto`, `memory_viewer_preprocess.proto` | — |
| **FE** | `frontend/app/components/memory_profile/`, `memory_viewer/` | — |
| **CLI** | `get_memory_profile_tool.py`, `get_peak_allocations_tool.py` | — |

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

## 1. memory_profile — full algorithm walk

### 1.1 End-to-end data path

```text
HTTP /data?tag=memory_profile&host=H
  → raw_to_tool_data.py: assert len(xspace_paths) == 1          # :163-168
  → xspace_wrapper / ProfileProcessor path
  → MemoryProfileProcessor::ProcessSession                     # memory_profile_processor.cc:39-59
       google::protobuf::Arena
       SessionSnapshot::GetXSpace(0, &arena)
       PreprocessSingleHostXSpace(xspace, step_grouping=true,
                                  derived_timeline=false)      # :53-54
       ConvertXSpaceToMemoryProfileJson(*xspace, &json)        # xplane_to_memory_profile.cc:585-594
         FindPlaneWithName(kHostThreadsPlaneName)
         ConvertXPlaneToMemoryProfile(*host_plane,
             max_num_snapshots=1000)                           # header default :42-43
           GenerateMemoryProfile: every kMemoryAllocation /
             kMemoryDeallocation XEvent → full snapshot
           ProcessMemoryProfileProto → sample + peak active
           version = 1
         GenerateHloModules(xspace) from device kXlaModuleLineName
         ConvertProtoToJson(always_print_fields_with_no_presence=true)
  → SetOutput(json, application/json)
  → respond gzip
  → FE MemoryProfile.parseData → MemoryProfileBase holds full proto
```

**Sources:** `xprof/convert/memory_profile_processor.cc`, `xprof/convert/xplane_to_memory_profile.{h,cc}`, `plugin/xprof/protobuf/memory_profile.proto`, `frontend/app/components/memory_profile/`.

### 1.2 Host / multi-host policy

| Policy | Behavior | Confidence |
|--------|----------|------------|
| Processor | `XSpaceSize() != 1` → `InvalidArgumentError` | **observed** `memory_profile_processor.cc:41-45` |
| Registry | Not in ALL_HOSTS sets | **observed** `registry.py:58-70` |
| FE | `numHosts` + `host_id` query param; host plane still single-XSpace convert | **observed** `memory_profile.ts:90-97` |
| CLI multi-host | `get_memory_profile` falls back: fetch without host, then iterate `get_hosts` until capacity+peak valid | **observed** `get_memory_profile_tool.py:196-242` |

**Severity on multi-host sessions:** each host convert is independent; worst case CLI walks N hosts serially, each paying full convert. UI typically one host at a time.

### 1.3 Stage A — GenerateMemoryProfile (what is kept)

Entry: `GenerateMemoryProfile(const XPlane* host_trace)` at `xplane_to_memory_profile.cc:88-180`.

Algorithm:

1. Create `XPlaneVisitor` over host plane (`CreateTfXPlaneVisitor`).
2. `ForEachLine` → `ForEachEvent`:
   - Read event type; **keep only** `HostEventType::kMemoryAllocation` or `kMemoryDeallocation` (`:59-65`, `:96-101`). All other host events discarded immediately.
   - Build `MemoryAggregationStats` + `MemoryActivityMetadata` from XStats (`:103-164`):
     - Stats: `kBytesReserved`→stack, `kBytesAllocated`→heap, `kBytesAvailable`→free, `kFragmentation`, `kPeakBytesInUse`.
     - Metadata: `kRequestedBytes`, `kAllocationBytes`, `kAddress`, `kTfOp`, `kGroupId`→step_id, `kRegionType`, `kDataType` (via `DataTypeString`), `kTensorShapes`.
     - Allocator key `memory_id`: from `kIndexOnHost` / `kDeviceOrdinal` (int→string) or `kAllocatorName` (string).
   - `UpdateProfileSummary(stats, event.OffsetPs(), summary)` (`:67-85`): tracks lifetime peak and **profiling-window peak** when `stack+heap >= peak_stats.peak_bytes_in_use`; writes `peak_stats_time_ps` and `memory_capacity = stack+heap+free`.
   - **Append** full `MemoryProfileSnapshot` to `memory_profile_per_allocator[memory_id].memory_profile_snapshots` (`:171-176`).

**Kept at this stage (per allocator):** every alloc/dealloc event as a complete snapshot (time + aggregation_stats + activity_metadata with string fields).

**Discarded:** non-memory host events; no downsampling yet; no peak-active table yet; deallocation metadata not yet filled from matching alloc.

| Property | Value | Confidence |
|----------|-------|------------|
| Complexity | O(events) time and space | **observed** |
| Peak intermediate | Full snapshot list before any prune | **observed** |
| String density | `tf_op_name`, `region_type`, `data_type`, `tensor_shape` per event | **observed** `:148-162` |

### 1.4 Stage B — ProcessMemoryProfileProto

Entry: `ProcessMemoryProfileProto(max_num_snapshots, MemoryProfile*)` at `:491-531`.

Global steps:

1. `num_hosts = 1` (`:493`).
2. Collect non-empty allocator keys into `memory_ids`, sort (`:495-501`).
3. Per allocator:
   a. **Sort** `memory_profile_snapshots` by `time_offset_ps` ascending (`:509-513`).
   b. `UpdateStepId` (`:186-198`): invalid step (`-1`) at start → 0; at boundaries/end → previous valid + 1.
   c. `UpdateDeallocation` (`:201-236`): walk sorted snapshots; map `address → alloc metadata*`; on FREE copy `tf_op_name`, `region_type`, `data_type`, `tensor_shape` from matching ALLOC; erase address. Unmatched FREE logs VLOG(2).
   d. `SampleMemoryProfileTimeline` → fill `sampled_timeline_snapshots` (see §1.5).
   e. `GetPeakMemoryStep(peak_bytes_in_use, …)` (`:239-252`): scan **all original snapshots** for last match of heap+stack == peak; return that snapshot's `step_id`.
   f. `ProcessActiveAllocations(peak_step_id, …)` (see §1.6).
   g. `SaveActiveAllocationSnapshots` rewrites field 1 to peak-referenced only (see §1.7).

### 1.5 Stage C — SampleMemoryProfileTimeline (kept vs discarded)

Entry: `:435-488`.

```text
original = memory_profile_snapshots  # still full list
if |original| > max_num_snapshots (default 1000):
  width = |original| / max_num_snapshots
  count1 = max_num_snapshots * (width + 1) - |original|
  count2 = max_num_snapshots - count1
  max_box_filter(width, count1, start=0):
    for each box of width W:
      pick snapshot with MAX (heap+stack) in box
      COPY full snapshot into sampled_timeline_snapshots
  max_box_filter(width+1, count2, start=width*count1)
else:
  sampled_timeline_snapshots = copy of all original
```

| Output field | Content | Used by FE? |
|--------------|---------|-------------|
| `sampled_timeline_snapshots` | ≤1000 **full** snapshots (stats **and** activity_metadata) | Yes — timeline chart; version==1 path |
| Original list (still in field 1 until Save) | All events | Intermediate only for peak path |

**Important nuance (observed):** Timeline chart series lines use aggregation_stats primarily, **but** tooltips read `activity_metadata` (`memory_timeline_graph.ts:249-293`). Slimming samples must retain at least tooltip-needed metadata **or** FE must stop showing per-event tooltips.

**Also observed:** summary panel counts alloc/dealloc **from sampled** timeline when `version==1` (`memory_profile_summary.ts:40-68`) — so counts are **approximate**, not true full-trace counts.

### 1.6 Stage D — ProcessActiveAllocations (peak active)

Entry: `:330-401`.

```text
unmapped_allocation_bytes = peak_stats.heap_allocated_bytes
active_alloc_map: address → (snapshot_index, metadata*)
for i in 0..snapshots (time-sorted original list):
  if snapshot.time_offset_ps > peak_stats_time_ps: break
  if metadata.step_id != peak_bytes_profile_step_id: continue
  if ALLOCATION:
    active_alloc_map[address] = {i, &metadata}
    unmapped_allocation_bytes -= allocation_bytes
  else DEALLOCATION:
    if address in map: erase
    else: unmapped_deallocation_bytes += allocation_bytes
    unmapped_allocation_bytes += allocation_bytes
unmapped_allocation_bytes -= unmapped_deallocation_bytes
active_allocs = values of map
InsertSpecialAllocations(unmapped, step_id):
  if unmapped > 0: special row "unused preallocated device memory"
  if stack_bytes > 0: special row "stack"
sort active_allocs by MetadataComparator:
  (-allocation_bytes, -requested_bytes, tf_op, region, dtype, shape)
merge identical rows → ActiveAllocation {snapshot_index, special_index, num_occurrences}
```

| Kept for peak table | Discarded from peak table |
|---------------------|---------------------------|
| Allocs live at peak time within peak step | Events after peak time |
| Special rows for unmapped heap + stack | Allocs freed before peak in that step |
| Merged identical metadata rows | Events in other steps |

Special indices: `snapshot_index < 0` → `special_index = -snapshot_index - 1`; else `special_index = -1` (`:386-390`).

### 1.7 Stage E — SaveActiveAllocationSnapshots (final field 1 rewrite)

Entry: `:405-431`.

1. Collect pointers to snapshots referenced by `active_allocations` where `snapshot_index >= 0`.
2. Remap those allocations' indices to dense `0..n-1`.
3. Replace `memory_profile_snapshots` with **only** those peak-referenced snapshots.

**Post-process residency per allocator:**

| Field | Content |
|-------|---------|
| `memory_profile_snapshots` | Peak-referenced only (often << events) |
| `sampled_timeline_snapshots` | ≤1000 full samples |
| `active_allocations` | Merged peak rows |
| `special_allocations` | Unmapped + stack pseudo-rows |
| `profile_summary` | Peak stats + capacity + lifetime peak |

**Still coexists until process end:** full original list + sampled list during ProcessMemoryProfileProto — full list only dropped when Save rewrites field 1.

### 1.8 Stage F — version, HLO modules, JSON

- `ConvertXPlaneToMemoryProfile` sets `version = 1` to signal new sampling (`:556-559`). FE uses this to pick `sampledTimelineSnapshots` vs legacy field 1 (`memory_timeline_graph.ts:130-136`).
- `GenerateHloModules` (`:562-583`): for each TPU device plane, walk `kXlaModuleLineName` events → `HloModule{id, name, start_time_ps, end_time_ps, plane_name}`.
- `ConvertProtoToJson` (`:533-548`): **`always_print_fields_with_no_presence = true`** → emits zeros/empty strings for absent fields. **Proto size driver.**

### 1.9 Materialization table (memory_profile)

| # | Stage | Representation | Peak / waste notes | Confidence |
|---|-------|----------------|--------------------|------------|
| M1 | XSpace load | Full host XSpace in Arena | Dominant input size | **observed** |
| M2 | `GenerateMemoryProfile` | **All** alloc/dealloc as snapshots | **O(events)** intermediate; full metadata strings | **observed** `:88-180` |
| M3 | `UpdateDeallocation` | `flat_hash_map<address, meta*>` | Copies strings onto free events | **observed** `:201-236` |
| M4 | Sort | In-place sort full list | Temporary residency | **observed** `:509-513` |
| M5 | Sample timeline | ≤1000 **full** snapshot copies | Metadata on every sample; tooltips use it | **observed** `:435-488`, FE tooltip |
| M6 | Peak active + Save | Shrink field 1 to peak refs | Good prune; dual lists until Save | **observed** `:330-431` |
| M7 | JSON | always_print | Empty field bloat | **observed** `:536-539` |
| M8 | Wire + FE | Gzipped JSON → Angular object | All allocators held; UI shows one | **observed** `memory_profile_base.ts:4-21` |
| M9 | HLO modules | repeated HloModule | Usually small vs snapshots | **observed** `:562-583` |
| M10 | Options | `max_num_snapshots` not plumbed | Always 1000 default | **observed** header `:42-43`; processor no options |

### 1.10 Severity assessment (memory_profile)

| Scenario | Peak driver | Severity |
|----------|-------------|----------|
| Short profile, few alloc events | XSpace + JSON | **L**–**M** |
| Long training window, dense allocator events | M2 full list; then M5+M7 of 1000 full samples × allocators | **H** |
| Many allocators (multi-device) | Map of PerAllocator each with sampled timeline | **H** |
| CLI scalar summary only | Same as UI convert (no summary mode) | **H** waste relative to need |

### 1.11 Opportunities (memory_profile MP-*)

| ID | Opportunity | Hotspot | Severity | Confidence |
|----|-------------|---------|----------|------------|
| MP-1 | Two-pass/streaming peak tracking without storing every event as full snapshot; keep peak bookkeeping + reservoir for timeline | M2 intermediate | **H** | hypothesized |
| MP-2 | Slim timeline samples: stats + minimal tooltip metadata; drop unused metadata fields (`address`, full shape when tooltip optional) | M5 wire/FE | **H** | **observed** partial (tooltip still needs some metadata) |
| MP-3 | Disable `always_print_fields_with_no_presence` for memory_profile JSON | M7 | **M** | **observed** |
| MP-4 | Plumb `max_num_snapshots` via ToolOptions / memory.py (and CLI) | M5, options gap | **M** | **observed** gap |
| MP-5 | Per-allocator lazy convert: only process selected memory_id | M8 multi-allocator | **M** | hypothesized |
| MP-6 | String interning for repeated tf_op_name / tensor_shape / data_type | M2/M6 | **M** | hypothesized |
| MP-7 | Binary protobuf or column-oriented timeline on wire | M7/M8 | **M** | hypothesized |
| MP-8 | Free unused allocator maps on FE when selection changes | M8 | **L** | hypothesized |
| MP-9 | Skip GenerateHloModules when overlay not needed | M9 | **L** | hypothesized |
| MP-10 | Sample during generate; never build O(events) RepeatedPtrField of full snapshots | M2+M5 | **H** | hypothesized |

**Top:** MP-2, MP-1/MP-10, MP-3, MP-4.

---

## 2. memory_viewer — full algorithm walk

### 2.1 End-to-end data path

```text
UI memory_viewer
  → getModuleList(session)  // memory_viewer.json, no module → CSV names
  → getDataByModuleNameAndMemorySpace(memory_viewer, module, color)
  → raw_to_tool_data.py options:                                    # :211-225
       module_name, program_id, view_memory_allocation_timeline,
       memory_space
  → MemoryViewerProcessor::ProcessSession                          # :51-98
       GetHloProtoByOptions(session, options)
       MemoryViewerOption {
         memory_color, small_buffer_size=kSmallBufferSize (16KiB),
         timeline_option.render_timeline / timeline_noise
       }
       ConvertHloProtoToMemoryViewer →
         ConvertHloProtoToPreprocessResultJson                     # utils.cc:1193-1215
           ConvertHloProtoToPreprocessResult                       # :1166-1191
           if render_timeline: HTML via RenderTimelineGraph(DOT)
           else: MessageToJsonString(result)  // NO always_print
  → content_type application/json | text/html
  → FE MemoryViewer holds preprocess; MemoryUsage re-sorts maxHeap
```

Also: `hlo_to_tools_data.cc:71-113` glue; `UnifiedMemoryViewerProcessor` (`unified_memory_viewer_processor.cc`) for unified HLO sessions — same option parse + convert.

### 2.2 Host / module policy

| Policy | Behavior | Confidence |
|--------|----------|------------|
| Registry | ∈ HLO_TOOLS + XPLANE_TOOLS | **observed** |
| Module list | Separate fetch returns CSV of module names | **observed** FE `memory_viewer.ts:82-108`, CLI `_get_module_names` |
| Per request | One module × one `memory_space` color | **observed** |
| Timeline option | `view_memory_allocation_timeline` in `build_memory_family` | **observed** `memory.py:14-15` |
| `timeline_noise` | C++ supported; not in Python options builder | **observed** gap |
| `memory_profile` options | **No** family builder — defaults only | **observed** `options/registry.py` keys |

### 2.3 Stage A — HloProtoBufferWrapper Init

Class: `hlo_proto_to_memory_visualization_utils.cc:218-397`.

1. Index all HLO instructions by name and unique id across computations (`:296-300`).
2. Index logical buffer protos by id (`:304-307`).
3. `ComputeLogicalBufferUnpaddedSizes` for unpadded size map (`:309-312`).
4. For each buffer allocation: build `BufferAllocationStruct`; for each assigned logical buffer build `LogicalBufferStruct` (shape resolve, offset, unpadded size) (`:314-338`).
5. Attach heap_simulator_trace_id to allocations by inspecting first event buffer of each trace (`:340-352`).

Trace selection for a color (`GetHeapSimulatorTraceId`):

- Prefer non-indefinite allocation's stored trace id for that color (`:385-396`).
- Fallback: trace with most events whose logical buffer color matches (`:357-381`).

### 2.4 Stage B — ProcessHeapSimulatorTrace (std::list + dual peak lists)

Entry: `:683-768`. Stats struct: `:552-681`.

```text
stats.SetSimulatorTraceEventSize(trace.events_size())
for event in trace.events():
  UpdateOnSimulatorEvent(event):
    push heap_size_bytes_timeline
    push unpadded_heap_size_bytes_timeline
    push hlo_instruction_name_timeline  # string per event
    mark seen logical buffer + allocation
  if ALLOC:
    logical_buffer.inc()
    IncreaseMemoryUsage(lb, init_buffer_span=true):
      logical_buffers.push_back(id)           # std::list program order
      heap_size += size; unpadded += unpadded
      if new peak:
        peak_heap_size_position = timeline.size-1
        peak_logical_buffers = logical_buffers  # LIST COPY
        peak_canonical_to_display_id = map snapshot
      init span [now, end)
  else if FREE:
    ref_count = dec(); if 0: DecreaseMemoryUsage(canonical):
      logical_buffers.remove(id)              # O(n) list::remove
      heap_size -= size; end span
  else if SHARE_WITH:
    share_with(canonical); if ref_count==1:
      map root_canonical → display sharer id
      IncreaseMemoryUsage(root, init_buffer_span=false)
FinalizeMemoryUsage:
  push final heap sizes + empty instruction name
  require seen_buffer_allocations.size() == 1
```

**Why `std::list`:** comment at `:653-655` — preserve **program order** of live buffers so peak snapshot order matches birth order for bar chart. FREE uses `list::remove` → linear scan (`:604`).

**Dual peak structures (observed):**

| Structure | Role |
|-----------|------|
| `std::list<int64_t> logical_buffers` | Live set in program order during sim |
| `std::list<int64_t> peak_logical_buffers` | Copy of live set at max heap |
| `canonical_to_display_id` / `peak_canonical_to_display_id` | SHARE_WITH display name remapping |

Invalid trace id → skip sim, return Ok; rest of pipeline still runs (`:689-696`).

### 2.5 Stage C — CreatePeakUsageSnapshot / PeakUsageSnapshot

`:770-854`.

1. For each indefinite lifetime logical buffer (best-sized assigned buffer per indefinite allocation):
   - Add allocation size to `indefinite_memory_usage_bytes`.
   - **Only memory_color == 0 (HBM)** adds HeapObject to bar chart (`:847-849`); VMEM skips indefinite bars.
2. `FinalizeBufferUsage`: for each id in `peak_logical_buffers` (display remapped), `AddHeapObject`:
   - If size < `small_buffer_size` (16KiB): accumulate into single "small (<N bytes)" object.
   - Else: `MakeHeapObject` with label, shape, tf_op, opcode, source_info, color index.

### 2.6 Stage D — GeneratePreprocessResult (PreprocessResult fields)

Entry: `:1069-1162`.

| PreprocessResult field | How filled | Size driver |
|------------------------|------------|-------------|
| `module_name`, `entry_computation_name` | HloModule name fields | Small |
| `max_heap` | peak HeapObjects program order | Peak buffers × string fields |
| `max_heap_by_size` | Sort copy of max_heap by size desc | **Duplicate of max_heap** |
| `max_heap_to_by_size`, `by_size_to_max_heap` | Index maps | O(peak) ints |
| `heap_sizes`, `unpadded_heap_sizes` | MiB timeline + indefinite add_mib | **#events+1 doubles** |
| `hlo_instruction_names` | Per-event strings + final "" | **#events+1 strings** |
| `peak_heap_mib`, `peak_unpadded_heap_mib`, `peak_heap_size_position` | Peak scalars | Small |
| `logical_buffer_spans` | map id→{start,limit} for seen buffers with span | O(seen buffers) |
| Special notes | `entry_computation_parameters_mib`, `non_reusable_mib`, `maybe_live_out_mib`, `total_buffer_allocation_mib`, `indefinite_buffer_allocation_mib`, `indefinite_lifetimes` | Nested trees for indefinite |
| Scoped VMEM | `max_scoped_vmem_*` from backend_config JSON (color==1 only) | Small |

Indefinite memory is **added** to every timeline point and to peak MiB (`:1117-1137`). Scoped VMEM is **not** added to peak/total (comment `:1152-1155`) — FE header adds it for display.

### 2.7 Stage E — ConvertAllocationTimeline (optional)

`:856-1000`. Only materializes `allocation_timeline` DOT string when building rects; used when `render_timeline` true → HTML (`:1202-1209`). When false, JSON path sets `allocation_timeline` to `entry_url` stub (`:1211`).

- Graph canvas `1<<12` × `1<<12` points; skip sub-pixel rects (`:947-949`).
- Indefinite allocations excluded from timeline.
- Optional dummy node when `timeline_noise` (`:991-995`).

### 2.8 Which FE fields actually read PreprocessResult

Primary consumer: `MemoryUsage.initMemoryUsageFromPrecomputed` (`memory_usage.ts:102-176`) and `memory_viewer_main.ts` charts.

| Proto field (JSON camelCase) | FE consumer | Notes |
|------------------------------|-------------|-------|
| `allocationTimeline` | MemoryUsage → timelineUrl | Or builds `/data?...view_memory_allocation_timeline=true` |
| `totalBufferAllocationMib` | MemoryUsage / header / CLI GPA | **Yes** |
| `peakHeapMib` | MemoryUsage / header | **Yes** |
| `peakUnpaddedHeapMib` | padding overhead calc | **Yes** |
| `entryComputationParametersMib` | totalArgumentSizeBytes | **Yes** |
| `indefiniteBufferAllocationMib` | temp size calcs | **Yes** |
| `maxScopedVmemAllocationMib` | header | **Yes** |
| `maxScopedVmemInstructionName` | header | **Yes** |
| `peakHeapSizePosition` | peak band on chart | **Yes** |
| `heapSizes` | program_order_chart | **Yes** full array |
| `unpaddedHeapSizes` | program_order_chart | **Yes** |
| `hloInstructionNames` | program_order tooltips | **Yes** |
| `logicalBufferSpans` | buffer selection / CSV download | **Yes** |
| `maxHeap[]` | bar charts after FE map | **Yes** — primary peak list |
| `maxHeapBySize[]` | **NONE** | FE re-sorts (`createMaxHeapIndex`) |
| `maxHeapToBySize` | **NONE** | FE rebuilds |
| `bySizeToMaxHeap` | **NONE** | FE rebuilds |
| `moduleName` | viewer shell / CSV filename | Partial |
| `entryComputationName` | **NONE** (interface only) | Dead wire |
| `nonReusableMib` | **NONE** | Dead wire |
| `maybeLiveOutMib` | **NONE** | Dead wire |
| `indefiniteLifetimes[]` | **NONE** | Dead wire nested tree |
| HeapObject.`label` | **NONE** | FE builds display from name+shape |
| HeapObject.`numbered` color | Replaced | FE assigns `nColor++` |
| HeapObject.`shapeString`, `instructionName`, `tfOpName`, `groupName`, `opCode`, `logicalBufferId`, `sourceInfo`, sizes | FE | Used |

### 2.9 Materialization table (memory_viewer)

| # | Stage | Representation | Peak / waste notes | Confidence |
|---|-------|----------------|--------------------|------------|
| V1 | HloProto load | Full buffer assignment + traces | Dominant input | **observed** |
| V2 | Wrapper maps | id→LogicalBufferStruct / BufferAllocationStruct | Full index coexists with proto | **observed** |
| V3 | Heap sim timelines | 2×vector&lt;int64&gt; + vector&lt;string&gt; ≈ #events | Dominates wire | **observed** `:661-664` |
| V4 | Active set | std::list + peak list copy | FREE O(n); peak copy clones list | **observed** `:657-658`, `:604` |
| V5 | Peak HeapObjects | vector at peak (+ small collapse) | Rich strings per buffer | **observed** |
| V6 | Dual max_heap on wire | max_heap + max_heap_by_size + indices | **Dead weight** — FE re-sorts | **observed** `memory_usage.ts:182-214`, C++ `:1079-1115` |
| V7 | logical_buffer_spans | map all seen | Useful; O(buffers) | **observed** |
| V8 | indefinite_lifetimes | Nested BufferAllocation tree | **No FE reader** | **observed** |
| V9 | Timeline DOT/HTML | 4096×4096 layout | Huge optional path | **observed** |
| V10 | JSON encode | Full PreprocessResult | Doubles as text; repeated strings | **observed** `:1213` |
| V11 | FE MemoryUsage | maxHeap ×3 sort orders | Padding sort only client-side (good); holds forever | **observed** |
| V12 | FE program order | Full series | No downsample | **observed** `program_order_chart.ts` |

### 2.10 Severity assessment (memory_viewer)

| Scenario | Peak driver | Severity |
|----------|-------------|----------|
| Small module, short sim trace | HloProto + modest JSON | **L**–**M** |
| Large module, long heap_simulator_traces | V3 + V6 + V10 | **H** |
| Timeline HTML enabled | V9 on top of preprocess | **H** |
| Switching memory space / module | Full re-convert each time | **H** multi-view |

### 2.11 Opportunities (memory_viewer MV-*)

| ID | Opportunity | Hotspot | Severity | Confidence |
|----|-------------|---------|----------|------------|
| MV-1 | Drop `max_heap_by_size` + index maps from wire (FE re-sorts) | V6 | **H** | **observed** |
| MV-2 | Downsample program-order timelines; keep peak position accurate | V3 | **H** | hypothesized |
| MV-3 | Lazy/optional hlo_instruction_names or string table | V3 | **H** | hypothesized |
| MV-4 | Replace std::list + remove with vector + freelist | V4 | **M** | **observed** |
| MV-5 | Slim HeapObject: omit `label`; optional source_info | V5 | **M** | **observed** (label unused) / hypothesized (source_info) |
| MV-6 | Durable PreprocessResult cache (module, color, small_buffer, version) | Re-open / CLI | **H** | hypothesized |
| MV-7 | Bound timeline HTML/DOT rects hard cap | V9 | **M** | **observed** partial |
| MV-8 | Peak-only convert: skip program-order vectors/spans | CLI / summary UI | **H** | hypothesized (needed by GPA) |
| MV-9 | Plumb timeline_noise, small_buffer_size through memory.py | options | **L** | **observed** gap |
| MV-10 | Stop building dual index maps in C++ after MV-1 | V6 CPU | **M** | **observed** once MV-1 |
| MV-11 | FE: release previous preprocess on module change | V11 | **L** | hypothesized |
| MV-12 | Column-pack heap_sizes as binary/base64 float32 | V3/V10 | **M** | hypothesized |
| MV-13 | Omit dead scalars: non_reusable, maybe_live_out, indefinite_lifetimes, entry_computation_name | V8 | **M** | **observed** |

**Top:** MV-1, MV-2/MV-3, MV-8, MV-6, MV-4, MV-13.

---

## 3. Dead wire fields map (proto → FE consumer or NONE)

### 3.1 MemoryProfile family

| Proto path | Emitted? | FE consumer | CLI summary? | Dead? |
|------------|----------|-------------|--------------|-------|
| `memory_profile_per_allocator[].profile_summary.*` | Yes | Summary panel | **Yes** (peak/cap/stack/heap/free/frag) | No |
| `...peak_stats.peak_bytes_in_use` | Yes | Summary | Yes | No |
| `...peak_stats.peak_bytes_in_use` lifetime vs window | Yes | Both lifetime + window shown | Peak window only | No |
| `...sampled_timeline_snapshots[]` | Yes (v1) | Timeline chart + incomplete-step check | No | No |
| `...sampled_timeline_snapshots[].activity_metadata.address` | Yes | **NONE** | No | **Yes** (address on samples) |
| `...sampled_timeline_snapshots[].activity_metadata` (partial) | Yes | Tooltip + alloc/dealloc **counts** | No | Partial |
| `...memory_profile_snapshots[]` (post-Save peak refs) | Yes | Breakdown table via index | No | No |
| `...memory_profile_snapshots[].activity_metadata.address` | Yes | **NONE** in breakdown | No | **Yes** for table path |
| `...active_allocations[]` | Yes | Breakdown | No | No |
| `...special_allocations[]` | Yes | Breakdown | No | No |
| `memory_ids` | Yes | Allocator selector | No (CLI walks map keys) | No |
| `num_hosts` | Yes | Host id loop | No | No (always 1 from convert) |
| `version` | Yes | Selects sampled vs legacy field | No | No |
| `hlo_modules[]` | Yes | Timeline overlay table | No | No |
| `hlo_modules[].plane_name` | Yes | Overlay display | No | No |
| RESERVATION / EXPANSION activity enum values | Proto only | Not produced by convert today | — | N/A |

Notes:

- Breakdown uses: `tfOpName`, `allocationBytes`, `requestedBytes`, `numOccurrences`, `regionType`, `dataType`, `tensorShape` (`memory_breakdown_table.ts:80-88`).
- Tooltip uses: memoryActivity, requested/allocation bytes, tfOp, stepId, regionType, dataType, tensorShape (`memory_timeline_graph.ts:249-293`).
- **Always-print JSON** inflates zeros/empty strings for all of the above even when unset (`xplane_to_memory_profile.cc:536-539`).

### 3.2 PreprocessResult family

| Proto path | Emitted? | FE consumer | Peak CLI? | Dead? |
|------------|----------|-------------|-----------|-------|
| `heap_sizes` | Yes | program_order_chart | No | No for UI; dead for CLI |
| `unpadded_heap_sizes` | Yes | program_order_chart | No | same |
| `hlo_instruction_names` | Yes | tooltips | No | same |
| `max_heap` | Yes | bar charts + FE re-sort + CLI buffers | **Yes** | No |
| `max_heap_by_size` | Yes | **NONE** | No | **Yes** |
| `max_heap_to_by_size` | Yes | **NONE** | No | **Yes** |
| `by_size_to_max_heap` | Yes | **NONE** | No | **Yes** |
| `logical_buffer_spans` | Yes | selection span + CSV | No | No for UI |
| `module_name` | Yes | shell / CSV | module key separate | Partial |
| `entry_computation_name` | Yes | **NONE** | No | **Yes** |
| `peak_heap_mib` | Yes | header | Optional | No |
| `peak_unpadded_heap_mib` | Yes | padding | No | No |
| `peak_heap_size_position` | Yes | peak band | No | No |
| `entry_computation_parameters_mib` | Yes | args size | No | No |
| `non_reusable_mib` | Yes | **NONE** | No | **Yes** |
| `maybe_live_out_mib` | Yes | **NONE** | No | **Yes** |
| `total_buffer_allocation_mib` | Yes | header + **GPA total** | **Yes** | No |
| `indefinite_buffer_allocation_mib` | Yes | temp calcs | No | No |
| `indefinite_lifetimes[]` | Yes nested | **NONE** | No | **Yes** |
| `allocation_timeline` | Stub URL or DOT | URL builder / HTML path | No | Partial |
| `max_scoped_vmem_*` | Yes if VMEM | header | No | No |
| HeapObject.`label` | Yes | **NONE** | No | **Yes** |
| HeapObject.`numbered` | Yes | overwritten FE | No | Effectively dead |
| HeapObject sizes/names/shape/tf_op/group/opcode/source | Yes | FE + CLI | buffers | No |

**CLI `_parse_and_aggregate_buffers` prefers `bufferAssignment.logicalBuffers`** (`get_peak_allocations_tool.py:184-200`) — **not** emitted by PreprocessResult. Practical path is always `maxHeap` fallback (`:202-215`). That branch is effectively dead for memory_viewer JSON.

---

## 4. CLI get_memory_profile_tool

### 4.1 End-to-end

```text
get_memory_profile(session_id)  [@decorators.cached expire=86400]
  → CachedXprofClient.fetch(tool_name="memory_profile.json", format=json[, host])
  → full UI memory_profile convert + JSON (§1)
  → _parse_profile_data: walk formats → MemoryProfileMetrics scalars only:
       memory_capacity, peak_hbm_bytes, stack, heap, free, fragmentation
  → emit ~6 GiB/% fields JSON
  Multi-host fallback:
       if peak/capacity invalid → client.get_hosts → for each host re-fetch full JSON
```

Source: `plugin/xprof/cli/tools/get_memory_profile_tool.py`.

### 4.2 Parse formats handled

| Format branch | Keys | Status |
|---------------|------|--------|
| Table | `cols`/`rows` | Raises "not fully supported" `:58-59` |
| Legacy summary | `memoryProfileSummary` + `peakBytesUsageHbm` | Supported `:62-88` |
| Peak MiB list | `peakMemoryUsageMiB` | Supported `:91-100` |
| Current UI | `memoryProfilePerAllocator` → `profileSummary.peakStats.peakBytesInUse` | Supported `:103-120` |

### 4.3 Materialization

| # | Stage | Representation | Notes | Confidence |
|---|-------|----------------|-------|------------|
| C1 | Convert | Full MemoryProfile JSON | Orders of magnitude &gt; output | **observed** |
| C2 | Parse | `json.loads` entire string | Transient full graph in Python | **observed** |
| C3 | Output | ~few hundred bytes scalars | Good | **observed** |
| C4 | Cache | SQLite 86400s on **scalar** result | Cold convert still full | **observed** |
| C5 | Multi-host | Serial full converts | Worst-case N× C1 | **observed** |

### 4.4 Opportunities (GMP-*)

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GMP-1 | `summary_only` / `view=summary` convert: peak stats only | **H** | **observed** mismatch |
| GMP-2 | Server-side host preference without full JSON per attempt | **M** | hypothesized |
| GMP-3 | Parallel first-success multi-host | **M** | hypothesized |
| GMP-4 | Streaming JSON / binary summary fields only | **L** | hypothesized |
| GMP-5 | Share full JSON cache with UI (family cache) | **M** | hypothesized |

**Top:** GMP-1, then GMP-3.

---

## 5. CLI get_peak_allocations_tool — N+1 loop detail

### 5.1 End-to-end

```text
get_peak_allocations(session_id, limit=10, min_size_mib=1.0, …)
  [@decorators.cached expire=86400]
  → _get_module_names:                                          # :260-298
       fetch memory_viewer.json (no module) → CSV module list
  → _fetch_modules_data: **for each module**                    # :301-368
       fetch memory_viewer.json?module_name=M
         → full ConvertHloProtoToPreprocessResult JSON
       parse totalBufferAllocationMib
       _parse_and_aggregate_buffers:
         prefer bufferAssignment.logicalBuffers (absent)
         else maxHeap[] → aggregate name.* / Others < min_size
  → heapq.nlargest(limit) by total_hbm_mib                     # :432-436
  → markdown or compact modules JSON
```

Source: `plugin/xprof/cli/tools/get_peak_allocations_tool.py`.

### 5.2 Exact loop structure

```text
module_names = split(CSV from memory_viewer without module)   # 1 light call
modules_data = []
for module in module_names:                                    # N heavy calls
  mod_data = client.fetch(
      tool_name="memory_viewer.json",
      session_id=session_id,
      format="json",
      module_name=module,
  )  # → MemoryViewerProcessor → full heap sim + dual max_heap + program-order vectors
  parsed = json.loads(mod_data)
  total_hbm_mib = parsed["totalBufferAllocationMib"]          # single scalar
  top_buffers = aggregate(parsed["maxHeap"] or logicalBuffers)
  modules_data.append(ModulePeakAllocations(...))
sorted = heapq.nlargest(limit, modules_data, key=total_hbm_mib)
  # limit truncates OUTPUT only — all N modules already converted
```

### 5.3 Per-iteration peak cost

| Per module iteration pays | Actually needed for ranking | Actually needed for top buffers |
|---------------------------|-----------------------------|----------------------------------|
| Load HloProto | Yes | Yes |
| HloProtoBufferWrapper full index | Yes for sim | Yes |
| Full heap simulator (list + timelines) | Peak exists inside sim | Peak live set |
| Program-order vectors (#events doubles+strings) | **No** | **No** |
| Dual max_heap_by_size + index maps | **No** | **No** |
| logical_buffer_spans full map | **No** | **No** |
| indefinite_lifetimes tree | **No** | **No** |
| JSON of all above | **No** | **No** |
| `totalBufferAllocationMib` | **Yes** | — |
| `maxHeap[]` (or peak list) | — | **Yes** |

If a session has **50 modules**, tool performs **1 list + 50 full memory_viewer converts** before ranking. Cold start severity **H** (observed loop structure).

### 5.4 Materialization (GPA)

| # | Stage | Representation | Notes | Confidence |
|---|-------|----------------|-------|------------|
| P1 | Module list fetch | CSV string | Cheap | **observed** |
| P2 | Per-module convert | Full PreprocessResult JSON | **N× UI path** | **observed** `:323-329` |
| P3 | Python parse | Full dict per module | Discards heap_sizes / spans / dual indices | **observed** |
| P4 | Aggregation | Regex name.* + size buckets | CPU modest vs convert | **observed** `:164-258` |
| P5 | Output cache | Ranked list 86400s | Cold catastrophic | **observed** |

### 5.5 Opportunities (GPA-*)

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GPA-1 | Stop serial full convert per module: batch peak mode or multi-module API | **H** | **observed** |
| GPA-2 | `view=peak_buffers` convert: totals + max_heap only | **H** | **observed** |
| GPA-3 | Two-phase: cheap per-module total rank, deep-dive top-K | **H** | hypothesized |
| GPA-4 | Reuse disk HloProto + preprocess cache (MV-6) | **H** | hypothesized |
| GPA-5 | Parallel module fetches with concurrency bound | **M** | hypothesized |
| GPA-6 | Align max_heap-only with FE bar chart (still skip timelines) | **H** | hypothesized |

**Top:** GPA-1 + GPA-2.

---

## 6. summary_only / peak_buffers mode design (analysis only)

### 6.1 Goal

Projection modes so CLI (and light UI) stop paying UI-scale protos. Aligns with synthesis theme 1 (`00-synthesis-memory-optimization.md`).

Proposed modes (names illustrative):

| Mode | Primary consumers |
|------|-------------------|
| `full` / `ui` | Angular memory_profile / memory_viewer |
| `summary` | get_memory_profile CLI; overview-style KPI |
| `peak_buffers` | get_peak_allocations CLI; peak bar without program-order chart |
| `timeline` | Separate HTML allocation timeline (already quasi-mode) |

### 6.2 memory_profile — what can be skipped per stage

| Stage | full/ui | summary_only | Notes |
|-------|---------|--------------|-------|
| XSpace load + preprocess step_grouping | Required | Required | Peak needs step ids |
| GenerateMemoryProfile full snapshots | Required for UI | **Stream/skip store** if MP-1 | Summary needs peak stats only |
| UpdateDeallocation string copy | UI breakdown | **Skip** | Summary ignores ops |
| SampleMemoryProfileTimeline | UI chart | **Skip** | |
| ProcessActiveAllocations + special | UI table | **Skip** | |
| SaveActiveAllocationSnapshots | UI | **Skip** | |
| GenerateHloModules | UI overlay | **Skip** | |
| always_print JSON of full tree | UI | Emit tiny summary message | |
| Wire fields | All §3.1 live | `profile_summary` (+ maybe memory_ids) only | |

**Minimum summary payload (hypothesized):** per allocator `{memory_id, memory_capacity, peak_bytes_in_use, stack, heap, free, fragmentation, peak_stats_time_ps, peak_bytes_usage_lifetime}`. Matches CLI `_parse_profile_data` branch 4.

### 6.3 memory_viewer — what can be skipped per stage

| Stage | full/ui | peak_buffers | summary (totals only) |
|-------|---------|--------------|----------------------|
| HloProto load | Required | Required | Required |
| Wrapper Init | Required | Required | Required (or cheaper total API) |
| ProcessHeapSimulatorTrace full timelines | UI chart | **Track peak only** (no vector push of names) | Peak heap scalar only |
| hlo_instruction_names vector | UI | **Skip** | **Skip** |
| heap_sizes / unpadded vectors | UI | **Skip** | **Skip** |
| peak_logical_buffers → max_heap | UI + GPA | **Keep** | **Skip** if totals-only phase |
| dual max_heap_by_size + maps | Dead today | **Skip** | **Skip** |
| logical_buffer_spans | UI selection | **Skip** | **Skip** |
| NoteSpecialAllocations / indefinite_lifetimes | Partial FE | Totals yes, tree no | Totals only |
| ConvertAllocationTimeline | Timeline mode | **Skip** | **Skip** |
| Scoped VMEM scan | UI VMEM | Optional | Optional |

**Minimum peak_buffers payload (hypothesized):** `{module_name, total_buffer_allocation_mib, peak_heap_mib, indefinite_buffer_allocation_mib, max_heap[] slim}`. Matches GPA fields actually read.

**Two-phase GPA (hypothesized):**

1. All modules: `summary` totals only (or even disk-side total from buffer assignment without full sim if equivalent).
2. Top-K modules: `peak_buffers` for buffer lists.

### 6.4 Options plumbing gaps (observed)

| Option | C++ | Python builder | Processor |
|--------|-----|----------------|-----------|
| `view_memory_allocation_timeline` | Yes | memory.py only for memory_viewer | Yes |
| `timeline_noise` | Yes | **No** | Parsed if present |
| `small_buffer_size` | Yes (hardcoded kSmallBufferSize) | **No** | Hardcoded |
| `max_num_snapshots` | API default 1000 | **No** | Never passed |
| `summary_only` / `view` | **No** | **No** | **No** |
| memory_profile any options | N/A | **No family builder** | Ignores ToolOptions |

Cites: `memory.py:11-16`, `options/registry.py:17-22`, `memory_profile_processor.cc:39-59` (options unused), `memory_viewer_processor.cc:67-84`.

---

## 7. Opportunities matrix (MP / MV / GMP / GPA / FAM / FE)

Severity H|M|L only. Confidence observed|hypothesized only.

### 7.1 Combined table

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| MP-1 | Streaming peak without full snapshot store | **H** | hypothesized |
| MP-2 | Slim timeline samples | **H** | observed (partial) |
| MP-3 | Disable always_print for MP JSON | **M** | observed |
| MP-4 | Plumb max_num_snapshots | **M** | observed |
| MP-5 | Per-allocator lazy convert | **M** | hypothesized |
| MP-6 | String interning | **M** | hypothesized |
| MP-7 | Binary/column timeline | **M** | hypothesized |
| MP-8 | FE free unused allocators | **L** | hypothesized |
| MP-9 | Skip HloModules optional | **L** | hypothesized |
| MP-10 | Sample during generate | **H** | hypothesized |
| MV-1 | Drop dual max_heap wire fields | **H** | observed |
| MV-2 | Downsample program-order timelines | **H** | hypothesized |
| MV-3 | Optional instruction name strings | **H** | hypothesized |
| MV-4 | Replace std::list FREE O(n) | **M** | observed |
| MV-5 | Slim HeapObject label/source | **M** | observed/hypothesized |
| MV-6 | Durable preprocess cache | **H** | hypothesized |
| MV-7 | Bound timeline HTML | **M** | observed partial |
| MV-8 | Peak-only convert mode | **H** | hypothesized |
| MV-9 | Complete memory.py options | **L** | observed |
| MV-10 | Stop dual index build in C++ | **M** | observed |
| MV-11 | FE release prior module | **L** | hypothesized |
| MV-12 | Pack heap_sizes binary | **M** | hypothesized |
| MV-13 | Omit dead PreprocessResult fields | **M** | observed |
| GMP-1 | summary_only convert for MP | **H** | observed |
| GMP-2 | Host probe without full JSON | **M** | hypothesized |
| GMP-3 | Multi-host first-success | **M** | hypothesized |
| GMP-4 | Stream parse summary | **L** | hypothesized |
| GMP-5 | Share JSON cache with UI | **M** | hypothesized |
| GPA-1 | Stop N× full convert | **H** | observed |
| GPA-2 | peak_buffers convert mode | **H** | observed |
| GPA-3 | Two-phase rank then deep-dive | **H** | hypothesized |
| GPA-4 | Cache reuse across modules | **H** | hypothesized |
| GPA-5 | Parallel module fetch bound | **M** | hypothesized |
| GPA-6 | max_heap-only option | **H** | hypothesized |
| FAM-1 | View modes full/ui/summary/peak_buffers | **H** | observed |
| FAM-2 | On-disk result cache key tool×module×space×view | **H** | hypothesized |
| FAM-3 | JSON bloat policy family-wide | **H** | observed |
| FAM-4 | Ensure peak CLI reuses HloProto files | **M** | observed path exists |
| FAM-5 | Complete memory.py options surface | **M** | observed |
| FAM-6 | CLI call summary/peak modes only | **H** | observed |
| FAM-7 | Single-host UX for memory_profile | **L** | hypothesized |
| FAM-8 | Log convert peak RSS / JSON bytes / event counts | **M** | hypothesized |
| FAM-9 | Keep labs memory_analysis out of prod dual path | **L** | hypothesized |
| FE-MP-1 | Request slim timeline schema when backend ready | **H** | hypothesized |
| FE-MV-1 | Already ignores dual max_heap — safe to drop | **H** | observed |
| FE-MV-2 | Null preprocess before module switch | **L** | hypothesized |
| FE-FAM-1 | Drop tool JSON from store on navigate away | **M** | hypothesized |

### 7.2 Family priority order

1. **FAM-1 / GPA-2 / GMP-1** — summary & peak_buffers convert modes.  
2. **MV-1 + MP-2 + MV-13** — delete dead/duplicate wire fields.  
3. **GPA-1** — eliminate N× full memory_viewer converts.  
4. **FAM-2 / MV-6** — durable preprocess cache.  
5. **MV-4 + MP-1/MP-10** — intermediate residency & list churn.  
6. **FAM-3 / MP-3** — JSON print policy.  
7. **FAM-5 / MP-4** — options plumbing.  
8. **MV-7** — bound optional timeline HTML.

---

## 8. Proto size drivers

### 8.1 memory_profile.proto

| Driver | Mechanism | Severity | Confidence |
|--------|-----------|----------|------------|
| String fields on every snapshot | `tf_op_name`, `region_type`, `data_type`, `tensor_shape` (`memory_profile.proto:50-59`) | **H** at event scale | observed |
| always_print JSON | Emits defaults/empty (`xplane_to_memory_profile.cc:536-539`) | **M**–**H** | observed |
| Dual snapshot lists until Save | Full + sampled coexist | **H** intermediate | observed |
| Sampled list still full metadata | max_box copies whole snapshot | **H** wire | observed |
| Multi-allocator map | Each allocator gets own sampled list | **H** multi-device | observed |
| address uint64 on metadata | Unused by FE | **L**–**M** | observed |
| hlo_modules names | Usually small | **L** | observed |

### 8.2 memory_viewer_preprocess.proto

| Driver | Mechanism | Severity | Confidence |
|--------|-----------|----------|------------|
| `hlo_instruction_names` length #events+1 | repeated string (`proto:63`, filled `:1130-1131`) | **H** | observed |
| `heap_sizes` + `unpadded_heap_sizes` | repeated double ×2 | **H** | observed |
| Dual `max_heap` + `max_heap_by_size` | Full HeapObject duplicate | **H** | observed |
| HeapObject string fields | label, instruction_name, shape_string, tf_op_name, group_name, op_code | **M**–**H** | observed |
| HeapObject.source_info | file + stack_frame strings | **M** | observed |
| `indefinite_lifetimes` nested | BufferAllocation × LogicalBuffer trees | **M** (dead FE) | observed |
| `logical_buffer_spans` map | All seen buffers | **M** | observed |
| `allocation_timeline` DOT | When rendered; huge | **H** optional | observed |
| MessageToJsonString default | Doubles as decimal text | **M** | observed |
| Index int32 arrays dual maps | Smaller than HeapObject dup | **L**–**M** | observed |

### 8.3 Encoding comparison

| Tool | JSON options | Gzip at respond | Notes |
|------|--------------|-----------------|-------|
| memory_profile | always_print_fields_with_no_presence=true | Yes (serve path) | Worst print policy |
| memory_viewer | default MessageToJsonString | Yes | Still large vectors as text |
| timeline HTML | DOT → WrapDotInHtml | Yes | Separate content-type |

---

## 9. Frontend surfaces (detail)

### 9.1 memory_profile FE

| Component | Path | Holds | Notes | Confidence |
|-----------|------|-------|-------|------------|
| Shell | `memory_profile.ts` | Full MemoryProfileProto | Host id from numHosts; incomplete-step from last sampled stepId | observed |
| Base | `memory_profile_base.ts` | data, memoryIds, selection | All allocators in data | observed |
| Timeline | `memory_timeline_graph/` | sampledTimelineSnapshots + hloModules | AreaChart; tooltips use metadata | observed |
| Breakdown | `memory_breakdown_table/` | Peak active rows | Indexes peak snapshots | observed |
| Summary | `memory_profile_summary/` | Peak stats | Counts from **sampled** series | observed |

### 9.2 memory_viewer FE

| Component | Path | Holds | Notes | Confidence |
|-----------|------|-------|-------|------------|
| Shell | `memory_viewer.ts` | preprocess + module list | Module switch reloads full | observed |
| Main | `memory_viewer_main.ts` | charts + MemoryUsage | Cross-chart selection indices | observed |
| MemoryUsage | `memory_usage.ts` | heap arrays + maxHeap ×3 | **Ignores wire maxHeapBySize** | observed |
| Program order | `program_order_chart.ts` | Full series | No downsample | observed |
| Max heap chart | `max_heap_chart.ts` | Bar chart | Peak buffers | observed |
| Buffer details | `buffer_details.ts` | Selected buffer | Small | observed |
| CSV download | `max_heap_chart_downloader.ts` | spans + heapSizes | Uses logicalBufferId spans | observed |
| Timeline link | MemoryUsage URL | Second convert HTML | observed |

### 9.3 FE createMaxHeapIndex (re-sort proof for MV-1)

From `memory_usage.ts:182-214`:

1. Map maxHeap → indexed pairs.
2. Sort by sizeMiB desc → `maxHeapBySize`, `bySizeToMaxHeap`, `maxHeapToBySize`.
3. Sort by padding (size−unpadded) desc → `maxHeapByPaddingSize`, padding index maps.

Wire fields `maxHeapBySize` / `maxHeapToBySize` / `bySizeToMaxHeap` are never read. Padding order **never** on wire (FE-only) — correct design; size dual on wire is pure waste.

---

## 10. Measurement plan

### 10.1 memory_profile long traces

**Goal:** quantify M2 intermediate peak vs final wire size.

| Step | Action | Metrics |
|------|--------|---------|
| 1 | Select long training/session with dense host alloc events (multi-minute, multi-step) | — |
| 2 | Run convert under `/usr/bin/time -l` or allocator tracing | Peak RSS, wall time |
| 3 | Instrument (analysis only — temporary counters) event count, pre-sample snapshot bytes, post-Save field1 count, sampled count | counts |
| 4 | Capture gzipped response Content-Length vs ungzip JSON length | wire / inflate |
| 5 | Parse JSON in node/python; measure heap of object graph | FE-like residency |
| 6 | Breakdown JSON: % size from sampled_timeline_snapshots vs peak snapshots vs summary | field drivers |
| 7 | Repeat with artificially lowered max_num_snapshots (if patched) 1000/256/64 | sensitivity |
| 8 | Multi-allocator host vs single | map multiplier |

**Success criteria for later work:** identify whether M2 intermediate or M5 wire dominates; confirm MP-2/MP-10 ROI.

### 10.2 Multi-module peak CLI (get_peak_allocations)

| Step | Action | Metrics |
|------|--------|---------|
| 1 | Session with N∈{5,20,50} HLO modules | — |
| 2 | Cold cache: run get_peak_allocations(limit=10) | wall, peak RSS, #HTTP/tool fetches |
| 3 | Per-module: time + response bytes of memory_viewer.json | histogram |
| 4 | Fraction of JSON bytes in heap_sizes+names vs maxHeap vs other | dead weight % |
| 5 | Warm cache second run | decorator hit rate |
| 6 | Simulate two-phase: if only totals fetched first (manual), time to rank top-10 vs full N | GPA-3 bound |
| 7 | Parallelism experiment (analysis): N concurrent fetches with cap 4 vs serial | latency vs RSS |

**Success criteria:** prove N× full convert cost; estimate savings of peak_buffers mode (drop program-order vectors) as % of JSON and convert CPU.

### 10.3 Cross-family harness notes

- Reuse convert serve metrics from `02-convert-serve-cache-path.md` (gzip, cache stampede).
- Log fields suggested under FAM-8: `tool`, `module`, `memory_color`, `event_count`/`sim_events`, `json_bytes`, `peak_rss_delta`, `view_mode`.
- Do **not** rely on DEFAULT_CACHE_TOOLS warm-up (neither memory tool is listed).

### 10.4 Suggested synthetic fixtures

| Fixture | Construction | Stresses |
|---------|--------------|----------|
| MP-dense | Many kMemoryAllocation events same allocator | M2/M5 |
| MP-multi-alloc | Many memory_ids | map of timelines |
| MV-long-trace | Large heap_simulator_traces events | V3 strings |
| MV-wide-peak | Many large live buffers at peak | max_heap dual |
| GPA-many-mod | N small modules | N+1 loop |

---

## 11. Processors and glue (code map)

### 11.1 MemoryProfileProcessor

```text
memory_profile_processor.cc:39-59
  reject XSpaceSize != 1
  Arena GetXSpace(0)
  PreprocessSingleHostXSpace(step_grouping=true, derived_timeline=false)
  ConvertXSpaceToMemoryProfileJson
  SetOutput(application/json)
REGISTER_PROFILE_PROCESSOR("memory_profile", …)  # :62
```

**ToolOptions ignored** — no max_num_snapshots, no summary flag.

### 11.2 MemoryViewerProcessor

```text
memory_viewer_processor.cc:51-98
  GetHloProtoByOptions
  parse memory_space → memory_color
  parse view_memory_allocation_timeline, timeline_noise
  small_buffer_size = kSmallBufferSize
  ConvertHloProtoToMemoryViewer
  content_type json or text/html
REGISTER_PROFILE_PROCESSOR("memory_viewer", …)  # :101
```

### 11.3 UnifiedMemoryViewerProcessor

Same option semantics; `ProcessHlo` on unified session (`unified_memory_viewer_processor.cc:33-85`). Accepts `timeline` as alias for render flag (`:66-67`).

### 11.4 hlo_to_tools_data

- `ConvertHloProtoToMemoryViewer` → `ConvertHloProtoToPreprocessResultJson` (`:71-74`).
- `ConvertHloProtoToToolData` for memory_viewer / graph_viewer (`:76-120`).

### 11.5 Python raw_to_tool_data

- `memory_profile`: assert one path (`:163-168`).
- `memory_viewer`: builds options dict; sets content_type HTML when timeline (`:211-225`).

---

## 12. Algorithm pseudocode appendix

### 12.1 GenerateMemoryProfile → Process → Sample → Peak

```text
function ConvertXPlaneToMemoryProfile(host_plane, max_snapshots=1000):
  profile = empty MemoryProfile
  for event in host_plane if alloc or free:
    stats, meta = parse XStats
    UpdateProfileSummary(profile[allocator].summary, stats, t)
    profile[allocator].snapshots.append({t, stats, meta})
  ProcessMemoryProfileProto(max_snapshots, profile):
    profile.num_hosts = 1
    profile.memory_ids = sorted(nonempty allocators)
    for each allocator A:
      sort A.snapshots by t
      UpdateStepId(A)
      UpdateDeallocation(A)           # fill free metadata from alloc map
      SampleTimeline(A, max_snapshots) # → A.sampled_timeline_snapshots
      step = GetPeakMemoryStep(A.summary.peak_stats.peak_bytes_in_use, A)
      ProcessActiveAllocations(step, A)
      SaveActiveAllocationSnapshots(A.snapshots, A.active_allocations)
  profile.version = 1
  return profile
```

### 12.2 Heap simulator → PreprocessResult

```text
function ConvertHloProtoToPreprocessResult(hlo, option):
  wrapper = HloProtoBufferWrapper(hlo)  # index buffers, allocations, traces
  stats = HeapSimulatorStats(wrapper)
  ProcessHeapSimulatorTrace(wrapper, option.memory_color, stats)
  peak = PeakUsageSnapshot(wrapper, stats, option.small_buffer_size)
  CreatePeakUsageSnapshot(wrapper, color, peak)  # indefinite + peak objects
  result = PreprocessResult()
  GeneratePreprocessResult(wrapper, stats, peak, color, result)
  ConvertAllocationTimeline(..., result)  # may leave allocation_timeline empty
  return result

function ProcessHeapSimulatorTrace(...):
  list live  # program order
  for e in events:
    record timelines (heap, unpadded, name)
    ALLOC → live.push; maybe new peak snapshot of live
    FREE  → live.remove; close span
    SHARE_WITH → refcount + display map; maybe IncreaseMemoryUsage
  finalize trailing timeline point
```

### 12.3 N+1 peak allocations

```text
function get_peak_allocations(session, limit=10, min_size_mib=1.0):
  names = fetch_module_csv(session)          # 1 call
  data = []
  for m in names:                            # N calls — full UI convert each
    j = fetch_memory_viewer(session, m)
    data.append({
      module: m,
      total: j.totalBufferAllocationMib,
      buffers: aggregate(j.maxHeap, min_size_mib)
    })
  return nlargest(limit, data, key=total)    # limit does not reduce N
```

---

## 13. Frontend interfaces & versioning notes

- FE types: `frontend/app/common/interfaces/data_table.ts` re-exports `MemoryProfileProto`, `MemoryViewerPreprocessResult`.
- Memory profile `version==1` required for sampled timeline field selection; version 0 would read peak-pruned field 1 as timeline (wrong after Save).
- numHosts is set to 1 by convert always; multi-host is a **session/product** concern outside single convert (FE host_id / CLI host walk).
- Incomplete steps warning: last sampled snapshot `stepId < 1` (`memory_profile.ts:99-110`).

---

## 14. Risk / non-goals

| Item | Note |
|------|------|
| No product code in this doc | Analysis only |
| Do not change peak semantics blindly | ProcessActiveAllocations step filter is subtle |
| Do not drop timeline metadata without FE tooltip redesign | MP-2 dependency |
| labs/curated/memory_analysis | Parallel path; avoid third materializer in prod (FAM-9) |
| bufferAssignment.logicalBuffers in GPA | Dead preferred branch; do not design on it |

---

## Cross-links to other design notes

| Topic | Doc |
|-------|-----|
| Shared convert → cache → gzip → FE peak | `02-convert-serve-cache-path.md` |
| HLO extract shared with graph_viewer | `05-graph-hlo-util-tools.md` |
| CLI decorator cache pattern | `06-megascale-and-cli-detectors.md` |
| Portfolio prioritization | `00-synthesis-memory-optimization.md` |

---

## Appendix A — Proto field → consumer map (summary)

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
| `total_buffer_allocation_mib`, `indefinite_*` mib, params | FE header, CLI total | **Yes (total)** |
| `max_heap[]` | FE bar chart, CLI buffers | **Yes (buffers)** |
| `max_heap_by_size[]` + index maps | **NONE (FE re-sorts)** | No |
| `heap_sizes[]`, `unpadded_heap_sizes[]`, `hlo_instruction_names[]` | FE program-order chart | No |
| `logical_buffer_spans` | FE buffer lifespan | No |
| `indefinite_lifetimes[]` | **NONE** | No |
| `non_reusable_mib`, `maybe_live_out_mib`, `entry_computation_name` | **NONE** | No |
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
| XPlane → profile header | `xprof/convert/xplane_to_memory_profile.h` |
| Memory viewer processor | `xprof/convert/memory_viewer_processor.cc` |
| Unified memory viewer | `xprof/convert/unified_memory_viewer_processor.cc` |
| HLO → preprocess | `xprof/convert/hlo_proto_to_memory_visualization_utils.cc` |
| HLO → preprocess header | `xprof/convert/hlo_proto_to_memory_visualization_utils.h` |
| HLO tool glue | `xprof/convert/hlo_to_tools_data.cc` |
| Protos | `plugin/xprof/protobuf/memory_profile.proto`, `memory_viewer_preprocess.proto` |
| FE profile | `frontend/app/components/memory_profile/` |
| FE viewer | `frontend/app/components/memory_viewer/` |
| FE MemoryUsage re-sort | `frontend/app/components/memory_viewer/memory_usage/memory_usage.ts` |
| CLI summary | `plugin/xprof/cli/tools/get_memory_profile_tool.py` |
| CLI peaks | `plugin/xprof/cli/tools/get_peak_allocations_tool.py` |
| CLI cache decorator | `plugin/xprof/cli/internal/decorators.py` |

---

## Appendix C — Line-level cite index (selected)

| Topic | Cite |
|-------|------|
| max_num_snapshots default 1000 | `xplane_to_memory_profile.h:42-43` |
| GenerateMemoryProfile event filter | `xplane_to_memory_profile.cc:96-101` |
| UpdateProfileSummary peak | `xplane_to_memory_profile.cc:67-85` |
| Sample max-box filter | `xplane_to_memory_profile.cc:435-488` |
| ProcessActiveAllocations | `xplane_to_memory_profile.cc:330-401` |
| SaveActiveAllocationSnapshots | `xplane_to_memory_profile.cc:405-431` |
| always_print JSON | `xplane_to_memory_profile.cc:536-539` |
| version=1 | `xplane_to_memory_profile.cc:556-559` |
| Single XSpace enforce | `memory_profile_processor.cc:41-45` |
| std::list live + peak | `hlo_proto_to_memory_visualization_utils.cc:657-658` |
| list::remove on FREE | `hlo_proto_to_memory_visualization_utils.cc:604` |
| Peak list copy | `hlo_proto_to_memory_visualization_utils.cc:589` |
| Dual max_heap emit | `hlo_proto_to_memory_visualization_utils.cc:1079-1115` |
| Timeline vectors fill | `hlo_proto_to_memory_visualization_utils.cc:1120-1132` |
| kSmallBufferSize 16KiB | `hlo_proto_to_memory_visualization_utils.h:29` |
| FE re-sort maxHeap | `memory_usage.ts:182-214` |
| FE ignores wire by-size | `memory_usage.ts:161-175` (reads maxHeap only) |
| GPA N-loop fetch | `get_peak_allocations_tool.py:323-329` |
| GPA total field | `get_peak_allocations_tool.py:354` |
| GPA maxHeap fallback | `get_peak_allocations_tool.py:202-215` |
| GMP multi-host fallback | `get_memory_profile_tool.py:196-242` |
| memory.py options thin | `memory.py:11-16` |
| options registry keys | `options/registry.py:17-22` |

---

## Appendix D — Worked cost model (order-of-magnitude, hypothesized)

Let:

- E = alloc/dealloc events on host plane
- A = number of allocators with events
- S = min(E, 1000) samples per allocator
- B = peak live buffers (non-small)
- T = heap simulator events
- M = HLO modules in session

| Path | Intermediate | Wire (order) |
|------|--------------|--------------|
| memory_profile UI | O(E) snapshots then O(S) sample + O(B_peak_rows) | O(A × S × |snapshot|) JSON |
| memory_profile summary_only (proposed) | O(E) stream or O(1) peak | O(A) scalars |
| memory_viewer UI | O(T) timelines + O(B) heap objects ×2 | O(T) doubles+strings + O(B) objects |
| memory_viewer peak_buffers (proposed) | O(T) sim without name vector optional | O(B) objects + scalars |
| GPA today | M × (UI convert) | M × UI JSON (discarded) |
| GPA two-phase (proposed) | M × totals + K × peak_buffers | O(M) scalars + K × O(B) |

When T ≫ B and M ≫ K, GPA and full MV dominate family RAM/CPU. Confidence: **hypothesized** magnitudes; structure **observed**.

---

## Appendix E — Decision checklist before implementation PRs

1. Measure §10 fixtures first (FAM-8).
2. Prefer wire drops with zero FE change (MV-1, MV-13, MP-3).
3. Add view modes behind options; default remains full UI.
4. CLI must opt into summary/peak_buffers (FAM-6) — do not change UI default silently.
5. Preserve peak step semantics and SHARE_WITH display mapping when rewriting heap sim.
6. Coordinate FE tooltip if slimming MP sample metadata.
7. Cache keys must include view mode and small_buffer_size / max_snapshots.
8. Avoid introducing labs path as third production materializer.

---

*End of memory tools family analysis. No product code changes in this document.*
