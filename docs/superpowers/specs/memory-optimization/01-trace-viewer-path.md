# Trace Viewer Path — Memory Use & Optimization Design Note

**Status:** analysis-only (no product code)  
**Date:** 2026-07-13  
**Scope:** XPlane → plugin HTTP → convert (C++/Python) → browser (Catapult + Trace Viewer v2)

## 1. End-to-end data path

```text
DISK: <run_dir>/<host>.xplane.pb
  (+ streaming preprocess: <host>.SSTABLE, .metadata.SSTABLE, .trie.SSTABLE)
        │
BROWSER Angular (trace_viewer.ts / V2 main.ts)
  hosts=≤10, resolution, start/end_time_ms, tag=trace_viewer@ | trace_viewer
        │ HTTP GET /data
PLUGIN api/data_api.py → ToolDataService → HostSelector
  options: tools/options/trace.py
  convert: raw_to_tool_data.py → pywrap
  respond.py → ALWAYS gzip by default
        │
C++ StreamingTraceViewerProcessor (trace_viewer@)
  first hit: GetXSpace(arena) → Preprocess → TraceEventsContainer → LevelDB
  every request: LoadTraceEventsContainer (visibility/resolution) → merge hosts
    format=json: TraceEventsToJson → full std::string
    format=pb: zstd TraceDataResponse delta series
  non-stream trace_viewer: ConvertXSpaceToTraceEventsString → Python process_raw_trace
        │
BROWSER: inflate gzip → JSON.parse or WASM zstd/proto → SoA timeline / Catapult model
```

## 2. Materialization by stage

| Stage | Representation | Peak notes | Confidence |
|-------|----------------|------------|------------|
| On-disk XPlane | `*.xplane.pb` | Hundreds of MB–GB multi-host | observed |
| GetXSpace | Arena XSpace | Full host decode | observed (`streaming_trace_viewer_processor.cc`) |
| TraceEventsContainer | Arena events + name table | Peak ≈ XSpace + full event set before LevelDB | observed |
| LevelDB | 3 tables/host; 20 MiB blocks; 10k event chunks | Disk spill for streaming | observed (`trace_events.cc`) |
| Visibility | Timespan + resolution bins | Disabled if num_events < 500_000 | observed (`kDisableStreamingThreshold`) |
| Multi-host merge | Serial ProcessSession / parallel Reduce | Reduce peak ≈ sum of concurrent host containers | observed |
| JSON serialize | One `std::string` | Full catapult-style events | observed |
| format=pb | zstd TraceDataResponse | Intermediate proto + compressed | observed |
| Python non-stream | Trace proto + `''.join` JSON | Double materialization | observed (`trace_events_json.py`) |
| respond() | gzip.compress(body) | Raw + gzip coexist; **also gzips already-zstd pb** | observed (`http/respond.py`) |
| V2 JSON | Full JS object graph + WASM copy | Dominant browser heap | observed |
| V2 PB | ArrayBuffer → WASM → zstd → proto | Better, but `use_pb` default false & not wired | observed (`feature_flags.ts`) |
| V2 timeline | `FlameChartTimelineData` SoA | `entry_args` maps per event (TODO to drop) | observed (`timeline.h`) |
| Legacy Catapult | Model + `_loadedTraceEvents` Set | Unbounded growth on pan/zoom merge | observed |

## 3. Streaming vs non-streaming

| Aspect | `trace_viewer` | `trace_viewer@` |
|--------|----------------|-----------------|
| Hosts | Legacy C++ often 1 XSpace | Multi-host merge / Map-Reduce |
| First request | Full XSpace → Trace → Python JSON | Full XSpace → LevelDB → filtered load |
| Later | Full reconvert unless external cache | LevelDB hit |
| Viewport | Limited | start/end + resolution |
| Output | JSON via Python | JSON or format=pb |
| Disk | None | 3 SSTABLEs/host |

“Streaming” means **viewport re-fetch of LevelDB subset**, not HTTP chunked convert.

**Gap (observed):** `use_pb` flag exists but is not applied to data URLs; V2 checks `.pb` pathname while plugin uses `/data?...` → production likely JSON-only.

## 4. Multi-host impact

- `supports_multi_host_selection`: only `trace_viewer` / `trace_viewer@` (`registry.py`)
- FE caps hosts at **10** (`trace_viewer.ts`)
- Missing host falls back to **all hosts** (dangerous)
- ProcessSession sequential (lower peak); Reduce parallel (higher peak)

## 5. Frontend retention / virtualization opportunities

1. Wire `format=pb` + `use_pb` end-to-end
2. Skip double gzip on zstd pb
3. Lazy args (events vs metadata SSTABLE); slim `entry_args`
4. Stop Catapult model accumulation when V2 default
5. Abort in-flight fetches; optional LRU of K tiles
6. Cap hostless all-hosts for trace
7. Tune FETCH_RATIO / ZOOM_RATIO (3× / 8× overfetch)

## 6. Ranked opportunities

| ID | Stage | Description | Severity | Confidence | Effort |
|----|-------|-------------|----------|------------|--------|
| T1 | FE+BE | Enable format=pb for V2 | **H** | observed gap | M |
| T2 | HTTP | Avoid gzip on already-zstd pb | **H** | observed | S |
| T3 | FE V2 | Drop/lazy-load entry_args | **H** | observed TODO | M |
| T4 | BE | Bound multi-host first-open (parallel with backpressure) | **H** | observed sequential TODO | L |
| T5 | BE | Audit kDisableStreamingThreshold / resolution=0 | **M** | observed | S |
| T6 | BE JSON | Stream JSON without full std::string | **M** | hypothesized | L |
| T7 | Python | Eliminate process_raw_trace double parse | **M** | observed | M |
| T8 | Reduce | Free host containers ASAP | **M** | observed | M |
| T9 | Legacy FE | Disable Catapult accumulation | **M** | observed | S–M |
| T10 | FE | Abort + free mid-pan payloads | **M** | partial | M |
| T11 | LevelDB | Tune 20 MiB block size | **M** | observed | S |
| T12 | Multi-host | Lower host cap / progressive load | **M** | observed | S |
| T17 | Overfetch | Reduce ZOOM/FETCH ratios | **M** | observed | S |

## 7. Top 5 wins

1. **T1** Ship compressed PB pipeline for V2  
2. **T2** Single compression layer for PB  
3. **T3** Slim resident timeline (no per-event args maps)  
4. **T4** Bound multi-host first-open peak  
5. **T17+T5** Fetch less density by default  

## 8. Risks

| Opt | Correctness | UX |
|-----|-------------|-----|
| T1 | Proto/version skew | Blank timeline if fallback broken |
| T2 | Clients assuming gzip-only | None if Content-Encoding correct |
| T3 | Selection needs second RTT | Slight select delay |
| T4 | SSTABLE write races | OOM if unbounded parallel |
| T17 | More round-trips | Zoom stutter on slow nets |

## 9. Real repo paths cited

- `plugin/xprof/convert/raw_to_tool_data.py`
- `plugin/xprof/convert/trace_events_json.py`
- `plugin/xprof/profile_plugin/tools/options/trace.py`
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/services/tool_data.py`
- `plugin/xprof/profile_plugin/services/hosts.py`
- `plugin/xprof/profile_plugin/api/data_api.py`
- `plugin/xprof/profile_plugin/http/respond.py`
- `xprof/pywrap/profiler_plugin_impl.cc`
- `xprof/convert/streaming_trace_viewer_processor.cc`
- `xprof/convert/trace_view_options.cc`
- `xprof/convert/trace_viewer/trace_events.cc`
- `xprof/convert/trace_viewer/trace_viewer_visibility.cc`
- `xprof/convert/trace_viewer/delta_series/trace_data_to_compressed_delta_series_proto.cc`
- `frontend/app/components/trace_viewer/trace_viewer.ts`
- `frontend/app/components/trace_viewer_container/trace_viewer_container.ts`
- `frontend/app/components/trace_viewer_v2/main.ts`
- `frontend/app/components/trace_viewer_v2/feature_flags.ts`
- `frontend/app/components/trace_viewer_v2/timeline/timeline.h`
- `frontend/app/components/trace_viewer_v2/trace_helper/trace_pb_event_parser.cc`
- `plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html`
- `frontend/app/components/megascale_perfetto/megascale_perfetto.ts`

## Appendix — Key constants

| Constant | Value | Where |
|----------|-------|-------|
| Default resolution | 8000 | `tools/options/trace.py` |
| V2 ZOOM_RATIO / FETCH | 8 / 3 | `main.ts`, `timeline/constants.h` |
| Disable visibility | 500_000 events | `trace_view_options.cc` |
| LevelDB block | 20 MiB | `trace_events.cc` |
| FE multi-host cap | 10 | `trace_viewer.ts` |
| use_pb default | false | `feature_flags.ts` |


## 10. Adjacent UI: `stack_trace_page` / `stack_trace_snippet`

**Role (observed):** Angular bridge so Trace Viewer (non-Angular Catapult path) and tools like op_profile / hlo_stats / roofline can show source/HLO snippets around a stack frame. Query params carry `stack_trace`, `hlo_module`, `hlo_op`, `source`, `session_id` — not a separate XPLANE convert tool.

**Paths:**
- `frontend/app/components/stack_trace_page/stack_trace_page.ts`
- `frontend/app/components/stack_trace_snippet/`
- Referenced from `frontend/app/components/trace_viewer/constants.ts` (`stack_trace_page`)
- Embedded toggles: `op_profile_base.ts`, `hlo_stats.ts`, `roofline_model/.../operation_level_analysis.ts`

### Materialization

| Stage | What is held | Confidence |
|-------|--------------|------------|
| Route query | Full stack_trace + hlo module strings in component fields | observed |
| SourceCodeService | Temporary availability check; may load file slices | observed (service token) |
| Parent tools | `showStackTrace` toggles keep parent tool tables resident while snippet open | observed |

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| ST-1 | Cap / truncate very long `stack_trace` query strings before navigation (URL + component heap) | **L** | hypothesized |
| ST-2 | Lazy-load `StackTraceSnippet` module only when panel open (avoid main bundle retain) | **L** | hypothesized |
| ST-3 | Release SourceCodeService file buffers when page destroyed (ensure no global cache growth) | **M** | hypothesized |
| ST-4 | Avoid duplicating HLO text already loaded by graph_viewer/op_profile — pass handles/IDs not full text | **M** | hypothesized |
| ST-5 | Trace Viewer deep-link: open snippet without retaining full Catapult model in background tab | **M** | hypothesized |

**Note:** Not an XPLANE_TOOLS entry; memory impact is secondary to the parent tool payload. Severity is generally **L–M** vs convert paths.
