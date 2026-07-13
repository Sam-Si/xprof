# Memory Tools Family — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only  
**Tools:** memory_profile, memory_viewer, get_memory_profile_tool, get_peak_allocations_tool

## Registry placement

| Concern | Path |
|---------|------|
| XPLANE tools | `plugin/xprof/profile_plugin/tools/registry.py` — both memory tools single-host (not ALL_HOSTS sets) |
| HLO | memory_viewer ∈ HLO_TOOLS |
| Options | `tools/options/memory.py` — only `view_memory_allocation_timeline` for memory_viewer |
| Convert | `raw_to_tool_data.py`; C++ `xplane_to_tools_data.cc`, `xplane_to_memory_profile.cc`, `hlo_proto_to_memory_visualization_utils.cc` |
| Protos | `plugin/xprof/protobuf/memory_profile.proto`, `memory_viewer_preprocess.proto` |

---

## 1. memory_profile

### Data path
XSpace → GetXSpace → GenerateMemoryProfile (all alloc/dealloc snapshots) → Process (max 1000 samples) → peak active allocations → JSON → FE charts.

### Materialization
| # | What | Risk |
|---|------|------|
| M2 | All events as MemoryProfileSnapshot before prune | O(events) intermediate |
| M5 | sampled_timeline_snapshots ≤1000 **full** snapshots | Wire/FE bloat if metadata retained |
| M7 | MessageToJsonString with always_print | JSON bloat |
| M8 | FE holds full MemoryProfileProto | Browser heap |

### Opportunities
| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| MP-1 | Two-pass peak without storing all events | **H** | hypothesized |
| MP-2 | Slim timeline samples (stats only, no activity_metadata) | **H** | observed |
| MP-3 | Disable always_print empty fields | **M** | observed |
| MP-4 | Expose max_num_snapshots via options | **M** | observed |
| MP-6 | String interning for repeated op names | **M** | hypothesized |

**Top:** MP-2, MP-1, MP-3

---

## 2. memory_viewer

### Data path
HloProto → heap simulator → max_heap (+ **duplicate** max_heap_by_size) → program-order vectors + instruction names → PreprocessResult JSON → FE (re-sorts maxHeap client-side).

### Materialization
| # | What | Waste |
|---|------|-------|
| V3 | Per-event vectors size = #sim events + string names | Dominates wire |
| V4 | std::list + remove on free | O(n) CPU/allocator |
| V6 | Dual max_heap sorts in proto | **Dead weight** — FE recomputes |
| V9 | DOT/HTML allocation timeline | Huge when enabled |

### Opportunities
| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| MV-1 | Drop dual-sorted max_heap from wire | **H** | observed |
| MV-2 | Downsample program-order timelines | **H** | hypothesized |
| MV-3 | Lazy hlo_instruction_names | **H** | hypothesized |
| MV-4 | Replace std::list | **M** | observed |
| MV-6 | Cache PreprocessResult per module×memory_space | **H** | hypothesized |
| MV-7 | Bound timeline HTML/DOT | **M** | observed |

**Top:** MV-1, MV-2/3, MV-6, MV-4

---

## 3. CLI get_memory_profile_tool

Path: full `memory_profile.json` convert → parse to **scalar summary only** → SQLite cache 86400s. Multi-host fallback re-converts per host.

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GMP-1 | summary_only convert mode | **H** | observed |
| GMP-3 | Smarter host short-circuit | **M** | hypothesized |

---

## 4. CLI get_peak_allocations_tool

Path: list modules → **for each module full memory_viewer convert** → aggregate top buffers. **N+1 critical**.

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GPA-1 | Stop serial full convert per module | **Critical** | observed |
| GPA-2 | peak-only convert mode | **Critical** | observed |
| GPA-4 | Cheap total-based rank then deep-dive top-K | **H** | hypothesized |

---

## Family shared opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| FAM-1 | Convert view modes: full \| ui \| summary \| peak_buffers | **Critical** | observed |
| FAM-2 | On-disk result cache keyed by module/view | **H** | hypothesized |
| FAM-3 | JSON bloat policy | **H** | observed |
| FAM-5 | Complete memory.py options (max_snapshots, summary_only) | **M** | observed |
| FAM-6 | CLI must not force UI-scale protos | **H** | observed |

### Family priority order
1. FAM-1 / GPA-2 / GMP-1 — summary & peak modes  
2. MV-1 + MP-2 — delete dead/duplicate wire fields  
3. GPA-1 — eliminate N× full converts  
4. FAM-2 — durable preprocess cache  
5. MV-4 + MP-1 — intermediate residency  

## Paths cited
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/tools/options/memory.py`
- `plugin/xprof/convert/raw_to_tool_data.py`
- `xprof/convert/xplane_to_memory_profile.cc`
- `xprof/convert/hlo_proto_to_memory_visualization_utils.cc`
- `xprof/convert/memory_profile_processor.cc`
- `xprof/convert/memory_viewer_processor.cc`
- `plugin/xprof/protobuf/memory_profile.proto`
- `plugin/xprof/protobuf/memory_viewer_preprocess.proto`
- `frontend/app/components/memory_profile/`
- `frontend/app/components/memory_viewer/memory_usage/memory_usage.ts`
- `plugin/xprof/cli/tools/get_memory_profile_tool.py`
- `plugin/xprof/cli/tools/get_peak_allocations_tool.py`
