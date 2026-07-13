# Synthesis: XProf Memory Optimization Opportunities

**Date:** 2026-07-13  
**Status:** Analysis / design only — **no product code changes in this campaign**  
**Audience:** Engineers prioritizing RAM-sensitive work as memory costs rise  
**Companion docs:** `01`–`06` in this directory

---

## (a) Tool inventory

### Shared data path (all tools)

```text
On-disk XPlane / HLO
  → SessionResolver + HostSelector
  → convert (raw_to_tool_data / pywrap / C++ processors)
  → ToolDataService (+ result cache version policy)
  → HTTP respond (JSON/bytes + default gzip)
  → Angular tool component (or CLI client)
```

**Sources:** `plugin/xprof/profile_plugin/tools/registry.py`, `services/tool_data.py`, `api/data_api.py`, `http/respond.py`, `convert/raw_to_tool_data.py`.

### XPLANE_TOOLS (registry)

| Tool | Design note | Host policy highlight |
|------|-------------|------------------------|
| `trace_viewer` | 01 | multi-host selection; legacy full JSON |
| `trace_viewer@` | 01 | multi-host; LevelDB streaming |
| `overview_page` | 04 | ALL_HOSTS only when multi-host |
| `input_pipeline_analyzer` | 04 | ALL_HOSTS supported |
| `framework_op_stats` | 04 | ALL_HOSTS supported |
| `kernel_stats` | 04 | ALL_HOSTS supported |
| `memory_profile` | 03 | single-host |
| `pod_viewer` | 05 | ALL_HOSTS only |
| `op_profile` | 04 | not in ALL_HOSTS UI sets |
| `hlo_stats` | 04 | not in ALL_HOSTS UI sets |
| `roofline_model` | 04 | not in ALL_HOSTS UI sets |
| `inference_profile` | 05 | multi-host parallel |
| `memory_viewer` | 03 | HLO (+ XPlane extract) |
| `graph_viewer` | 05 | HLO_TOOLS |
| `megascale_stats` | 06 | ALL_HOSTS supported |
| `perf_counters` | 05 | multi-host events |
| `utilization_viewer` | 05 | single-host enforced |
| `smart_suggestion` | 05 | ALL_HOSTS only; multi-tool cold |

### Cross-cutting surfaces

| Surface | Design note |
|---------|-------------|
| convert → serve → cache | 02 |
| tools_cache / generate_cache | 02 |
| megascale_perfetto FE | 06 |
| stack_trace_page / stack_trace_snippet | 01 (adjacent) |
| CLI tools | 03, 05, 06 |

### CLI tools

| CLI | Design note |
|-----|-------------|
| get_memory_profile_tool | 03 |
| get_peak_allocations_tool | 03 |
| get_overview_tool | 06 |
| get_graph_viewer_tool | 05 |
| get_utilization_viewer_tool | 05 |
| get_smart_suggestions_tool | 05 |
| get_kpi_metrics_tool | 05 |
| get_top_hlo_ops_tool | 05 |
| detect_layout_mismatch_copies_tool | 06 |
| detect_unfused_reshapes_tool | 06 |
| detect_unnecessary_convert_reduce_tool | 06 |

---

## (b) Trace-viewer path (deep dive summary)

Full detail: **`01-trace-viewer-path.md`**.

### Critical path memory story

1. **First open multi-host:** Σ XSpaces → full TraceEventsContainer → LevelDB write (20 MiB blocks) — highest server peak.  
2. **Steady-state:** LevelDB filtered load by time span + resolution — good design, undermined if `num_events < 500k` disables visibility or `resolution=0`.  
3. **Wire:** Default JSON + **gzip always** in `respond()`; `format=pb` + zstd exists but **`use_pb` not wired** to `/data` URLs.  
4. **Browser V2:** JSON object graph + WASM copy + SoA with per-event `entry_args` (TODO to drop).  
5. **Legacy Catapult:** merges tiles into unbounded model.

### Top trace wins (ranked)

| Rank | ID | Win | Severity | Confidence |
|------|-----|-----|----------|------------|
| 1 | T1 | Ship format=pb end-to-end for V2 | H | observed gap |
| 2 | T2 | Do not gzip already-zstd PB | H | observed |
| 3 | T3 | Slim `entry_args` / lazy args | H | observed TODO |
| 4 | T4 | Bound multi-host first-open | H | observed |
| 5 | T5/T17 | Keep visibility on; reduce overfetch | M | observed |

---

## (c) Ranked cross-cutting opportunities (global themes)

| Rank | Theme | Why it multiplies across tools | Severity | Confidence | Primary docs |
|------|-------|--------------------------------|----------|------------|--------------|
| 1 | **Projection modes** (`full` / `ui` / `summary` / `peak_*`) so CLI/KPI never build UI protos | CLI detectors, overview, memory, util, top-HLO all pay full convert | Critical | observed | 03, 05, 06 |
| 2 | **Concurrent convert cap** (shared semaphore: `/data` + cache worker) | max_workers=1 for cache only; UI tabs unbounded | H | hypothesized | 02 |
| 3 | **Encoding hygiene** — no double compression; free raw after gzip; identity for precompressed | respond() always gzips; Perfetto + trace pb | H | observed | 01, 02, 06 |
| 4 | **ALL_HOSTS / multi-host combine without holding all intermediates** | OpStats vector of all hosts; explicit TODOs | H | observed | 02, 04, 05 |
| 5 | **Disk cache quota + single-flight** | Unbounded op_stats files; stampede on invalidation | H–M | observed/hypothesized | 02 |
| 6 | **Stream / viewport / top-K on wire** | Unbounded kernel reports, HLO rows, request_details | H | observed | 01, 04, 05 |
| 7 | **Avoid double materialization** (dict→json→gzip; json.loads for CSV; JSON round-trip for suggestions) | CSV path; smart_suggestion | H–M | observed | 02, 05 |
| 8 | **Frontend virtualization / no accumulation** | Catapult merge; full tables with pageSize draw-only | M | observed | 01, 04 |
| 9 | **Dead fields / dual sorts** | memory_viewer dual max_heap; dual idle stats | M–H | observed | 03, 04 |
| 10 | **HLO load amortization** (detectors / peak / graph) | N× module loads; N× graph_viewer text | Critical | observed | 03, 05, 06 |

### Theme diagram

```text
                    ┌─────────────────────┐
                    │  Projection modes   │  ← biggest systemic lever
                    └──────────┬──────────┘
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
   CLI / KPI              UI full view          Warm-up cache
   summary_only           top-K / viewport      DEFAULT_CACHE_TOOLS
         │                     │                     │
         └─────────────────────┴─────────────────────┘
                               │
              Concurrent cap + encode hygiene + host batching
                               │
                    Disk quota / single-flight / reuse OpStats
```

---

## (d) Phased recommendations (analysis roadmap — not an implementation PR)

### Phase 0 — Measure (1–2 weeks)

- Peak RSS + wall time for: multi-host `trace_viewer@`, ALL_HOSTS `overview_page`, `get_peak_allocations`, Perfetto megascale, one detector tool.  
- Wire size histograms for memory_viewer PreprocessResult and kernel_stats JSON.  
- Confirm production V2 traffic is JSON (not pb).

### Phase 1 — High ROI, low design risk (S/M effort)

1. **Encoding:** skip gzip when payload already compressed (trace pb, Perfetto compressed_output).  
2. **Wire dead weight:** drop dual max_heap sorts (memory_viewer); slim memory_profile timeline samples.  
3. **CLI:** summary_only / peak_only flags for overview, memory_profile, utilization, peak allocations.  
4. **Feature flag:** wire `use_pb` → `format=pb` for V2 data URL.

### Phase 2 — Convert pipeline architecture (M/L)

1. Shared convert concurrency limit across `/data` + generate_cache.  
2. OpStats combine streaming / free hosts early (multi-host).  
3. Projection API on convert for OpStats-backed tools (skip unused DBs).  
4. LevelDB/visibility tuning for trace; bound multi-host first open.

### Phase 3 — Product/UX memory (M/L)

1. Frontend: no Catapult accumulation; slim V2 SoA; transfer lists for Perfetto.  
2. Detectors: lazy HLO load; neighborhood from proto not N× graph text.  
3. Disk cache quotas + single-flight convert keys.  
4. Deprecate legacy `trace_viewer` full JSON path.

### Explicit non-goals of this document set

- No C++/Python/TS patches shipped here.  
- No proprietary large-trace benchmarks.  
- Severities labeled **hypothesized** where unmeasured.

---

## Campaign method (how this analysis was produced)

| Wave | Concurrent | Focus | Docs |
|------|------------|-------|------|
| 1 | 3 | Trace path; convert/serve; memory family | 01–03 |
| 2 | 3 | Overview/stats; graph/HLO/util; megascale+CLI | 04–06 |
| 3 | controller | Inventory coverage + synthesis | 00 |

**Constraint:** max **3** concurrent research subagents; waves advanced on completion (finite campaign; not infinite 10‑minute forever loop after inventory complete).

**Log:** `{SCRATCH}/subagent-campaign.log`

---

## Index of design docs

| File | Content |
|------|---------|
| `00-synthesis-memory-optimization.md` | This synthesis |
| `01-trace-viewer-path.md` | Full trace viewer path deep dive |
| `02-convert-serve-cache-path.md` | Shared convert/serve/cache |
| `03-memory-tools-family.md` | memory_profile/viewer + peak CLI |
| `04-overview-and-stats-tools.md` | OpStats fan-out tools |
| `05-graph-hlo-util-tools.md` | graph, inference, util, smart, perf, pod |
| `06-megascale-and-cli-detectors.md` | megascale/Perfetto + detector CLIs |

---

## Confidence legend

- **observed:** Confirmed in source during this campaign.  
- **hypothesized:** Plausible from architecture; needs measurement before implementation bets.
