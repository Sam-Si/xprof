# Trace Viewer Path — Memory Use & Optimization Design Note

**Status:** analysis-only (no product code changes)  
**Date:** 2026-07-13  
**Scope:** XPlane on disk → plugin HTTP `/data` → convert (C++ streaming LevelDB + JSON/PB, Python non-stream JSON) → browser (Catapult iframe + Trace Viewer V2 WASM/WebGPU + Angular shell + stack_trace bridge)  
**Depth target:** deepest single-tool note in this tree (≥1200 lines of code-cited analysis)  
**Related notes:** `00-synthesis-memory-optimization.md` (portfolio), `02-convert-serve-cache-path.md` (shared convert/gzip), stack_trace adjacency below

This note re-derives the Trace Viewer memory story from **live repository sources** (not invented paths). Confidence labels are only **`observed`** (code exists and implies the behavior) or **`hypothesized`** (plausible peak/UX impact not yet measured). Severity is **H / M / L** for user-visible or peak-RAM impact, independent of confidence.

Every major claim below cites **`path:line`** from this workspace at write time.

---

## 0. Why Trace Viewer dominates the memory portfolio

Trace Viewer is unique among XProf tools:

| Dimension | Trace Viewer | Typical stats tool (overview / kernel_stats / …) |
|-----------|--------------|--------------------------------------------------|
| Host selection | Explicit multi-host CSV (`hosts=`) — **only** tools in `_MULTI_HOST_SELECTION_TOOLS` (`plugin/xprof/profile_plugin/tools/registry.py:99`) | ALL_HOSTS aggregate or single host |
| On-disk intermediate | 3 LevelDB SSTABLEs per host after first open (`streaming_trace_viewer_processor.cc:75-86`) | OpStats / tool-specific caches |
| Viewport | `start_time_ms` / `end_time_ms` / `resolution` re-fetch (`trace_view_options.cc:41-56`) | One-shot JSON table |
| Wire formats | Catapult JSON **or** zstd columnar `TraceDataResponse` (`format=pb`) (`streaming_trace_viewer_processor.cc:347-389`) | DataTable JSON |
| Browser model | Full timeline SoA + optional Catapult `tr.Model` + WASM heap | Angular DataTable graphs |
| First-open cost | Full XSpace → TraceEventsContainer → LevelDB write | Combined OpStats cache shared across tools |
| Warm-up | `DEFAULT_CACHE_TOOLS` includes `trace_viewer@` (`registry.py:55`) | `overview_page` only among stats tools |

**Implication:** optimizing Trace Viewer is not “another JSON table slim-down.” It is a multi-layer pipeline where peak RAM can appear in the **plugin process (C++)**, the **Python gzip layer**, and the **browser (JS + WASM)** simultaneously—and multi-host multiplies several of those peaks.

---

## 1. Full end-to-end data path

### 1.1 ASCII E2E diagram (production happy path: `trace_viewer@` + V2 JSON)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ DISK (session run_dir)                                                      │
│   <host>.xplane.pb                                                          │
│   after cold preprocess per host:                                           │
│     TRACE_LEVELDB / SSTABLE                                                 │
│     TRACE_EVENTS_METADATA_LEVELDB                                           │
│     TRACE_EVENTS_PREFIX_TRIE_LEVELDB                                        │
│   (paths via SessionSnapshot::MakeHostDataFilePath)                         │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ BROWSER — Angular shell                                                     │
│   frontend/.../trace_viewer/trace_viewer.ts                                 │
│   - hosts capped at 10 (sort + slice)  :556                                 │
│   - builds query: run, tag=trace_viewer@, hosts|host, filters, full_dma … │
│   - useTraceViewerV2 from query/localStorage  :179-194                      │
│   - dataService.getDataUrl → /data?...  :592-597                            │
│   ├─ V2: WASM loadTraceData(url)  [main.ts:1048]                            │
│   └─ Legacy: iframe → /trace_viewer_index.html?trace_data_url=…  :608-614  │
│       plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html (Catapult)   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTP GET /data
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ PLUGIN HTTP                                                                 │
│   api/data_api.py  DataApiMixin.data_route :112-121                         │
│     → parse ToolRequest (parse_request.py:13-31)                            │
│     → services/tool_data.py ToolDataService.get_tool_data :44-135           │
│         resolve run_dir, build_tool_params                                  │
│         tools/options/trace.py → trace_viewer_options :13-44                │
│           resolution default 8000, format, start/end_time_ms, event_name…  │
│         services/hosts.py HostSelector.select :27-115                       │
│           hosts_param CSV if supports_multi_host_selection                  │
│           host==ALL_HOSTS → all XPlanes                                     │
│           missing host/hosts → ALL hosts (legacy fallback :95-98)           │
│         convert.xspace_to_tool_data(asset_paths, tool, params)              │
│   convert/raw_to_tool_data.py :97-145                                       │
│     tool == 'trace_viewer@':                                                │
│       pywrap → C++ StreamingTraceViewerProcessor                            │
│       returns raw_data as-is (JSON string or pb bytes) :143                 │
│       content_type application/json | application/octet-stream :144-146     │
│     tool == 'trace_viewer':                                                 │
│       C++ Trace proto bytes → process_raw_trace → Python JSON join :127-137 │
│   http/respond.py respond() :29-118                                         │
│     DEFAULT: gzip.compress(body) + Content-Encoding: gzip :109-111          │
│     content_encoding arg never set by ToolDataService today :134            │
│     → **already-zstd pb is gzipped again**                                  │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ C++ convert — StreamingTraceViewerProcessor                                 │
│   REGISTER_PROFILE_PROCESSOR("trace_viewer@", …) :396                       │
│                                                                             │
│   ProcessSession (plugin local multi-host path) :58-158:                    │
│     for each host SERIAL (TODO b/452217676 parallel :72):                   │
│       if SSTABLE missing:                                                   │
│         Arena GetXSpace → PreprocessSingleHostXSpace :98-102                │
│         optional ProcessMegascaleDcn :103-105                               │
│         ConvertXSpaceToTraceEventsContainer :107-109                        │
│         StoreAsLevelDbTables (events + metadata + prefix trie) :120-124     │
│       LoadTraceEventsContainer(file_paths, viewport, resolution) :139-140   │
│       merged.Merge(host_container, host_id) :145                            │
│     SerializeAndSetOutput(merged) :151-152                                  │
│                                                                             │
│   Map/Reduce (worker service when XSpaceSize()>1) :h46-50:                  │
│     Map: per-host preprocess → SSTABLE path :160-235                        │
│     Reduce: parallel LoadTraceContainerForHost (MaxParallelism) :281-339    │
│             then serial Merge into one container :314-328                   │
│                                                                             │
│   SerializeAndSetOutput :342-391:                                           │
│     format=="pb": ConvertTraceDataToCompressedDeltaSeriesProto (zstd)       │
│     else: TraceEventsToJson → full std::string via IOBufferAdapter          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ gzip wire
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ BROWSER materialization                                                     │
│   V2 JSON path:                                                             │
│     fetch → response.json() → JS object graph (main.ts:711-717)             │
│     processTraceEvents → packer → FlameChartTimelineData SoA                │
│     entry_args: vector<flat_hash_map> per event (timeline.h:111)            │
│   V2 PB path (only if URL pathname ends with .pb — production /data does NOT):│
│     arrayBuffer → _malloc WASM copy → zstd decompress → TraceDataResponse   │
│     free decompressed early; free malloc buffer (main.ts:823-838)           │
│   Legacy Catapult:                                                          │
│     JSON → tr.Model; _loadedTraceEvents Set grows on pan/zoom merge         │
│   Event detail: second /data with event_name → force format=json            │
│   stack_trace_page: query-param bridge to Angular StackTraceSnippet         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Tool naming / catalog override

From `plugin/xprof/profile_plugin/tools/registry.py` and `tools/catalog.py`:

| Name | Meaning | Registration |
|------|---------|--------------|
| `trace_viewer` | Non-streaming (pre-TF 2.13 style) | `XPLANE_TOOLS` (`registry.py:34`) |
| `trace_viewer@` | Streaming / LevelDB-backed (TF 2.14+) | `XPLANE_TOOLS` (`registry.py:35`); `REGISTER_PROFILE_PROCESSOR` (`streaming_trace_viewer_processor.cc:396`) |

- If both available in a session, **catalog discards plain `trace_viewer`** so UI always prefers `@` (`catalog.py:85-89`).
- Cloud Vertex path may advertise plain `trace_viewer` only; streaming SSTABLE path then returns `UnimplementedError` “Cloud AI” (`streaming_trace_viewer_processor.cc:90-91`, `xplane_to_tools_data.cc:119-120`).
- Historical suffix notes in registry: `^` generated from XPlane, `#` gzip payload, `@` streaming (`registry.py:22-26`).

### 1.3 What “streaming” is and is not

| Product language | Reality (code) |
|------------------|----------------|
| “Streaming trace viewer” | LevelDB SSTABLE intermediate + viewport-filtered reload |
| HTTP chunked convert progress | **Not present** — single response after full serialize |
| Incremental LevelDB write visible to client | Client only sees final JSON/PB |
| Automatic PB for V2 | `use_pb` flag exists (`feature_flags.ts:20-24`); data URLs not switched to `format=pb` or `.pb` |
| Per-tile LRU on server | Each request reloads filtered events from LevelDB into a **new** container |

---

## 2. Line-cited constants table

All values verified in source. Memory implication is analytical (**observed** structure → **hypothesized** peak multiplier where noted).

| Constant | Value | Path:line | Memory implication |
|----------|-------|-----------|--------------------|
| Default `resolution` (plugin options) | **8000** | `plugin/xprof/profile_plugin/tools/options/trace.py:20` | Density binning default if FE omits; fewer events than resolution=0 |
| Default `format` (C++ TraceViewOption) | **`"json"`** | `xprof/convert/trace_view_options.h:39`; parsed default `trace_view_options.cc:55-56` | PB is opt-in; production path pays full JSON string |
| Default `format` force on `event_name` | **`"json"`** | `tools/options/trace.py:30-33` | Detail fetch never uses compressed PB |
| `kDisableStreamingThreshold` | **500_000** events | `trace_view_options.cc:114` | Below → visibility **disabled** (`trace_events.h:241-247`) |
| LevelDB `block_size` (store) | **20 MiB** | `trace_events.cc:416`, `:563` | Large table blocks in plugin RSS during open/store |
| LevelDB `block_size` (load) | **20 MiB** | `trace_events.h:205` | Read-side buffer granularity |
| LevelDB `block_size` (prefix trie) | **20 MiB** | `prefix_trie.cc:91`, `:119` | Search-path open cost |
| Event chunk size | **10_000** | `trace_events.cc:569`; `lite_trace_events.cc:341` | Parallel store granularity |
| `kMaxBufferedChunks` | **20** | `trace_events.cc:570`; `lite_trace_events.cc:342` | Up to 20 chunks buffered concurrently during write |
| Zstd compression level | **1** (fast) | `delta_series/zstd_compression.cc:21-23` | Lower CPU, larger compressed PB vs higher levels |
| V2 `ZOOM_RATIO` | **8** | `frontend/.../trace_viewer_v2/main.ts:12` | Resolution overfetch ×8 bins |
| V2 `FETCH_RATIO` | **3.0** | `main.ts:20` | Time-span overfetch ×3 |
| `kFetchRatio` | **3.0f** | `timeline/constants.h:188` | C++ side fetch width (pan/zoom re-request) |
| `kPreserveRatio` | **2.0f** | `timeline/constants.h:190` | Keep loaded window = viewport×2 before refetch |
| `kRefetchZoomRatio` | **8.0f** | `timeline/constants.h:193` | When density is too coarse → refetch |
| `MIN_EVENT_WIDTH` | **2** px | `main.ts:36` | Resolution bin size base |
| `HEADING_WIDTH` / `kDefaultLabelWidth` | **250** px | `main.ts:27`; `timeline/constants.h:61` | Subtracted from canvas width for bin count |
| `kMinFetchDurationMicros` | **1000** | `constants.h:186` | Floor on fetch duration (prevents tiny deep-zoom fetches) |
| `kMinDurationMicros` | **1e-6** | `constants.h:182` | Max zoom-in floor (1 ps) |
| FE multi-host cap | **10** | `trace_viewer.ts:556` `slice(0, 10)` | Max hosts per request from Angular shell |
| `use_pb` default | **false** | `feature_flags.ts:20-24` | PB pipeline off by default |
| `use_trace_viewer_v2` | query/localStorage | `trace_viewer.ts:179-194`; `constants.ts:124` | V2 vs Catapult iframe |
| `DEFAULT_CACHE_TOOLS` | `overview_page`, `trace_viewer@` | `registry.py:55` | Warm-up includes streaming TV |
| Multi-host selection tools | `trace_viewer`, `trace_viewer@` | `registry.py:99` | Only tools with `hosts=` CSV |
| Worker multi-host gate | `XSpaceSize() > 1` | `streaming_trace_viewer_processor.h:46-50` | Enables Map/Reduce path |
| Search debounce | **300** ms | `trace_viewer_container.ts:388` | UI only; does not bound server result size |
| Event args cache fuzzy match | **50_000** µs | `trace_viewer.ts:831` | Start-time equality window for cache hit |
| respond() default encoding | **gzip** | `http/respond.py:109-111` | Always compress unless `content_encoding` set |
| `content_encoding` from ToolDataService | **`None`** | `tool_data.py:134` | Forces gzip path always for TV |
| `load_metadata` default on viewport load | **`false`** | `trace_events.h:941` | Viewport loads skip metadata SSTABLE (args not bulk-loaded) |
| Interned empty string index | **0** | `delta_series/..._proto.cc:22-25` | PB string table reserved slot |
| Catapult hash key parts | `pid$ts[$z]` | `tf-trace-viewer.html:1936-1940` | Set key size ~ tens of bytes/event |
| Fuzzy slice match epsilon | **0.001** ms | `tf-trace-viewer.html:1956` | Args merge into existing slices |
| `kEventHeight` | **23** px | `constants.h:62` | Render height; not heap |
| `kEventMinimumDrawWidth` | **2** px | `constants.h:63` | Matches MIN_EVENT_WIDTH intent |
| `kEventNavigationMaxDurationMicros` | **5_000_000** | `constants.h:200` | Zoom-to-event viewport cap |
| `kEventNavigationZoomFactor` | **2.5** | `constants.h:204` | Expand around selected event |
| Filter config query param | `trace_filter_config` | `constants.ts:109` | Can force denser server filters |
| Feature flag storage prefix | `xprof_ff_` | `constants.ts:189`; `main.ts:988` | `use_pb` etc. persisted |
| Color palettes | 7 named | `constants.ts:194-202` | UI only |
| WASM heap metric | `window.wasmMemoryBytes` | `main.ts:888-890` | HEAPU8.length after process |
| ProcessSession host loop | serial | `streaming_trace_viewer_processor.cc:72-73` | Peak = max host + merged |
| Reduce thread count | `min(num_hosts, MaxParallelism())` | `streaming_trace_viewer_processor.cc:297-298` | Peak ≈ sum concurrent containers |
| Non-stream multi-host | **rejected** (`XSpaceSize()!=1`) | `xplane_to_tools_data.cc:88-91` | Plain `trace_viewer` is single-host only |

### 2.1 Derived formulas (from constants)

```text
# V2 resolution bins requested (when canvas known, no selected_group_ids):
resolution = round((canvasWidth - HEADING_WIDTH) / MIN_EVENT_WIDTH) * ZOOM_RATIO
           = round((W - 250) / 2) * 8

# Example W=1920 → bins ≈ (1670/2)*8 ≈ 6680  (near plugin default 8000)
# Example W=0 or no canvas → resolution stays 0 → no duration downsampling

# Time range overfetch (main.ts expandUrlTimeRange):
expanded_duration = viewport_duration * FETCH_RATIO(3)
# C++ MaybeRequestData also uses kFetchRatio=3 for refetch windows

# Multi-host event budget (hypothesized):
events ≈ f(viewport×FETCH_RATIO, resolution_bins×ZOOM_RATIO, filters) × N_hosts
```

---

## 3. Request parameter matrix → code path activation

Built by `build_trace_viewer_options` (`tools/options/trace.py:13-44`) and parsed by `GetTraceViewOption` (`trace_view_options.cc:39-84`). Load branching is in `LoadTraceEventsContainer` (`trace_view_options.cc:86-122`).

| Param | Default / source | Activates path | Memory effect | Confidence |
|-------|------------------|----------------|---------------|------------|
| `resolution` | Plugin default **8000** (`trace.py:20`); C++ default string `"0"` if missing (`trace_view_options.cc:45-46`); V2 overwrites via `updateUrlWithResolution` (`main.ts:651-676`) | Viewport load visibility filter | Higher → more events through density filter; **0** disables duration downsampling (`trace_viewer_visibility.cc:38-39`) | observed |
| `start_time_ms` / `end_time_ms` | C++ default `"0.0"` (`trace_view_options.cc:41-44`); V2 expands by `FETCH_RATIO=3` (`main.ts:695-706`) | `MilliSpan` for `TraceVisibilityFilter` | Overfetch ×3 time span on each load | observed |
| `format` | `"json"` (`trace_view_options.cc:55-56`); `event_name` **forces json** (`trace.py:30-33`) | `SerializeAndSetOutput` pb vs JSON (`streaming_trace_viewer_processor.cc:347-389`) | PB skips catapult JSON + uses zstd; still gzipped by Python | observed |
| `event_name` + `duration_ms` + `unique_id` | Empty/0 defaults | `ReadFullEventFromLevelDbTable` (`trace_view_options.cc:91-97`) | Full metadata for **one** event; always JSON | observed |
| `search_prefix` | `""` | `SearchInLevelDbTable` if trie exists (`trace_view_options.cc:98-108`) | Matching set materialization; different from viewport load | observed |
| `search_metadata` | `false` (`trace_view_options.cc:57-58`) | Search options `{.search_metadata=…}` | May pull metadata for matches | observed |
| `full_dma` | `false` (`trace.py:21`) | TraceOptions details | Can force denser / unfiltered data via details flags | observed |
| `enable_legacy_dcn` | `false` (`trace.py:22`) | `ProcessMegascaleDcn` on cold preprocess (`streaming_trace_viewer_processor.cc:103-105`) | Extra events on first open | observed |
| `hosts` (CSV) | FE cap 10 (`trace_viewer.ts:556`) | `HostSelector` multi-host branch (`hosts.py:69-78`) | Linear multi-host merge cost | observed |
| `host` | Single host or `ALL_HOSTS` | `hosts.py:79-84` | ALL_HOSTS = every XPlane | observed |
| *(neither host nor hosts)* | — | Legacy all-hosts (`hosts.py:95-98`) | **Dangerous accidental full session** | observed |
| `use_saved_result` | `true` (`parse_request.py:29`; `raw_to_tool_data.py:126`) | Result cache policy in ToolDataService | Affects whether convert re-runs / cache version write | observed |
| `tag` | tool name | `parse_request.py:27` maps `tag` → `req.tool` | Selects `trace_viewer` vs `trace_viewer@` | observed |
| `run` / `session_path` / `run_path` | session resolve | `tool_data.py:74-81` | Locates run_dir + assets | observed |
| `trace_filter_config` | FE optional | FILTER_CONFIG (`constants.ts:109`) → TraceOptions filters | Can densify or shrink event set | observed |
| `selected_group_ids` | absent | Forces `resolution=0` in V2 (`main.ts:660`) | **Full density** for filtered group view | observed |
| `use_pb` (feature flag) | **false**, localStorage `xprof_ff_use_pb` | **Does not alter data URL today** | Dead switch for memory path | observed |

### 3.1 Load path decision tree (server)

```text
LoadTraceEventsContainer (trace_view_options.cc:86-122):
  if event_name non-empty:
      → ReadFullEventFromLevelDbTable(metadata + events, name, start_ps, dur_ps, unique_id)
  else if search_prefix non-empty:
      if prefix_trie file exists:
          → SearchInLevelDbTable(..., search_metadata)
      else: no-op (empty container)
  else:
      → TraceVisibilityFilter(MilliSpan(start,end), resolution, options)
      → LoadFromLevelDbTable(..., kDisableStreamingThreshold=500000)
         load_metadata default false
```

### 3.2 Serialize path decision tree

```text
SerializeAndSetOutput (streaming_trace_viewer_processor.cc:342-391):
  if format == "pb":
      ConvertTraceDataToCompressedDeltaSeriesProto → SetOutput(bytes, octet-stream)
  else:
      TraceEventsToJson → std::string → SetOutput(json, application/json)
```

### 3.3 Python branch after C++

```text
raw_to_tool_data.xspace_to_tool_data (raw_to_tool_data.py:127-146):
  if tool == 'trace_viewer':
      if format=='pb': data=raw_data, octet-stream
      else: data=process_raw_trace(raw_data)  # ParseFromString + join JSON
  if tool == 'trace_viewer@':
      data=raw_data always  # no Python re-parse
      content_type json or octet-stream by format
```

---

## 4. Full call stack: HTTP `/data` → C++ return

### 4.1 Shared HTTP → service stack (both tools)

```text
1. DataApiMixin.data_route
     plugin/xprof/profile_plugin/api/data_api.py:112-121
     try:
       data, content_type, content_encoding = self.data_impl(request)
       return respond(data, content_type, content_encoding=content_encoding)

2. DataApiMixin.data_impl
     data_api.py:39-68
     req = tool_request_from_args(request.args)   # parse_request.py:13-31
     result = self._tool_data.get_tool_data(req, session_path=..., run_path=...,
                                            logdir=..., run_dir_cache=..., cache_lock=...)
     return result.data, result.content_type, result.content_encoding

3. tool_request_from_args
     parse_request.py:13-31
     hosts CSV → tuple; tag→tool; use_saved_result bool; raw_args all str

4. ToolDataService.get_tool_data
     tool_data.py:44-135
     a. try_counter_names_only early exit
     b. resolve_run_dir (sessions)
     c. should_use_saved_result → build_tool_params(req, use_saved_result)
        - options/registry maps trace_viewer / trace_viewer@ → build_trace_family
        - build_trace_viewer_options packs resolution/format/event_name/...
     d. HostSelector.select → asset_paths + selected_hosts
     e. params['hosts'] = selection.selected_hosts
     f. convert.xspace_to_tool_data(asset_paths, tool, params)
     g. return ToolResult(data, content_type, content_encoding=None)  # :131-135

5. HostSelector.select
     hosts.py:69-98 priority:
       hosts_param + multi-host tool → exact CSV list (FileNotFound if missing)
       host == ALL_HOSTS → all xplanes
       host in map → single
       host unknown → maybe FileNotFound
       not host and not hosts_param → ALL hosts (legacy)

6. xspace_to_tool_data
     raw_to_tool_data.py:97-145
     → pywrap xspace_to_tools_data / ProfileProcessor path
```

### 4.2 Stack for `tag=trace_viewer@` (streaming)

```text
7a. C++ ProfileProcessor factory → StreamingTraceViewerProcessor
      REGISTER_PROFILE_PROCESSOR("trace_viewer@", ...)  :396

8a. If worker service eligible (XSpaceSize()>1)  h:46-50:
      Map(host) per host:
        if SSTABLE exists → return path
        else: copy XSpace, Preprocess, optional MegascaleDcn,
              ConvertXSpaceToTraceEventsContainer, StoreAsLevelDbTables
              return SSTABLE path  :183-235
      Reduce(map_output_files):
        thread pool min(N, MaxParallelism)
        parallel LoadTraceContainerForHost → vector of StatusOr<Container>
        JoinAll; serial Merge; SerializeAndSetOutput  :281-339

9a. Else ProcessSession (local plugin path) :58-158:
      for i in 0..XSpaceSize-1 serial:
        resolve 3 SSTABLE paths (Cloud AI → Unimplemented if missing)
        if events SSTABLE missing:
          Arena GetXSpace → Preprocess → optional DCN
          ConvertXSpaceToTraceEventsContainer
          StoreAsLevelDbTables (3 files)
        LoadTraceEventsContainer → host container
        merged.Merge(move(host), i+1)
      SerializeAndSetOutput(merged)

10a. SerializeAndSetOutput :342-391
       format pb → ConvertTraceDataToCompressedDeltaSeriesProto
         GenerateResponse (columnar deltas + interned strings)
         SerializeToString → ZstdCompression::Compress (level 1)
         SetOutput(compressed, application/octet-stream)
       else → TraceEventsToJson into std::string via IOBufferAdapter
         SetOutput(json, application/json)

11a. Python raw_to_tool_data :138-146
       data = raw_data  # no re-parse
       content_type json or octet-stream

12a. respond() :107-111
       content_encoding is None → gzip.compress(body); Content-Encoding: gzip
```

### 4.3 Stack for `tag=trace_viewer` (non-streaming)

```text
7b. xplane_to_tools_data.cc ConvertXSpaceToTraceEvents :85-101
      require XSpaceSize()==1 else InvalidArgument
      Arena GetXSpace → Preprocess
      ConvertXSpaceToTraceEventsString(*xspace, &content)
      return content  # Trace proto wire bytes (not catapult JSON yet)

8b. Python process_raw_trace (raw_to_tool_data.py:41-45)
      Trace.ParseFromString(raw_trace)
      ''.join(TraceEventsJsonStream(trace))
        # yields header, per-event json.dumps, fake {} terminator
        # (trace_events_json.py:97-105)

9b. respond gzip as above
```

**Note:** Legacy branch in `ConvertXSpaceToTraceEvents` for `tool_name != "trace_viewer"` still implements single-host LevelDB+JSON (`xplane_to_tools_data.cc:102-168`) but multi-host production for `@` uses ProfileProcessor `ProcessSession`/`Map`/`Reduce`. The processor path is the multi-host authority.

### 4.4 Structures live at each stack frame (summary)

| Frame | Live memory (typical) | Severity |
|-------|----------------------|----------|
| GetXSpace arena | Full host XSpace | H cold |
| TraceEventsContainer (convert) | All events for host | H cold |
| StoreAsLevelDbTables buffers | ≤20 chunks × serialized KVs + 20MiB blocks | H cold |
| LoadTraceEventsContainer | Filtered events (or full if visibility off) | H warm |
| Reduce vector of containers | N concurrent filtered containers | H multi-host |
| merged container | Union of hosts | H |
| JSON std::string OR PB serialized+zstd | ≈ wire payload uncompressed/compressed | H / M |
| Python body + gzip body | Both until Response built | H |
| Browser inflate + JSON graph + WASM SoA | Often largest long-lived | H |

---

## 5. Stage-by-stage materialization

Peak memory is the **sum of concurrent live representations**, not the size of the final wire payload alone.

| # | Stage | Representation | Peak notes | Severity | Confidence |
|---|-------|----------------|------------|----------|------------|
| S0 | On-disk XPlane | `*.xplane.pb` | Hundreds of MB–multi-GB multi-host sessions | H (disk) | observed |
| S1 | `SessionSnapshot.GetXSpace(i, &arena)` | Arena-backed `XSpace` | Full host decode; arena freed after preprocess scope ends in ProcessSession (`streaming_trace_viewer_processor.cc:98-100`) | H | observed |
| S2 | `PreprocessSingleHostXSpace` | Mutates XSpace in place | Extra derived lines/events on same arena object | M | observed |
| S3 | Optional `ProcessMegascaleDcn` | XSpace mutation for legacy DCN | Extra events when `enable_legacy_dcn` | M | observed |
| S4 | `ConvertXSpaceToTraceEventsContainer` | Arena `TraceEventsContainer` + name table + raw args | Peak ≈ XSpace + full event set **before** LevelDB write | **H** | observed |
| S5 | `StoreAsLevelDbTables` | Parallel chunk serialize (`kChunkSize=10000`, `kMaxBufferedChunks=20`) + LevelDB `block_size=20 MiB` | Up to 20 chunks buffered × serialized KV pairs; 20 MiB table blocks | **H** | observed (`trace_events.cc:569-570`) |
| S6 | Cold SSTABLE on disk | 3 tables/host: events, metadata, prefix trie | Disk spill amortizes later loads; first open pays full S1–S5 | M (disk) / H (first RAM) | observed |
| S7 | `LoadTraceEventsContainer` | Filtered `TraceEventsContainer` in RAM | Visibility+resolution reduce events when `num_events ≥ 500_000`; below threshold visibility **disabled** | **H** | observed (`trace_view_options.cc:114-119`; `trace_events.h:241-247`) |
| S8 | Multi-host merge | `merged.Merge(std::move(host), host_id)` | ProcessSession: host container freed after merge; Reduce: **all host containers live until JoinAll** then merge | **H** | observed (`:145` vs `:299-328`) |
| S9 | JSON serialize | `std::string trace_viewer_json` via `IOBufferAdapter` | Full catapult-style event array as one contiguous string; coexists with merged container until SetOutput | **H** | observed (`:367-385`) |
| S10 | format=pb serialize | `TraceDataResponse` → serialize → zstd | Intermediate uncompressed proto + compressed bytes; columnar deltas + interned strings | M–H | observed (delta_series header `:111-119`) |
| S11 | Python non-stream (`trace_viewer`) | `Trace` proto parse + `''.join(TraceEventsJsonStream)` | Double materialization: C++ Trace string → Python proto → chunked JSON → joined str | **H** | observed (`raw_to_tool_data.py:41-45`, `trace_events_json.py:97-105`) |
| S12 | `respond()` gzip | Raw body + `gzip.compress(body)` | Both live until Response constructed; **zstd pb gzipped again** | **H** | observed (`respond.py:109-111`; `tool_data.py:134`) |
| S13 | HTTP transfer | gzip bytes | Wire size; browser inflate buffers | M | observed |
| S14 | V2 JSON parse | Full JS object graph (`traceEvents[]`) | Dominant browser heap for production path | **H** | observed (`main.ts:711-717`, `:841`) |
| S15 | V2 WASM process | `processTraceEvents` packs into C++ SoA | JS graph may still be live during pack; WASM heap grows (`ALLOW_MEMORY_GROWTH`); `window.wasmMemoryBytes` records HEAPU8 length | **H** | observed (`main.ts:873`, `:888-890`) |
| S16 | V2 PB parse | ArrayBuffer + WASM malloc copy + zstd out + proto | Better density; malloc free in finally; **pathname `.pb` gate unused for `/data`** | H if enabled; gap today | observed (`main.ts:802-838`) |
| S17 | `FlameChartTimelineData` | SoA vectors + **`entry_args` maps** | TODO to fetch args from backend; maps dominate per-event heap | **H** | observed (`timeline.h:106-111`) |
| S18 | Timeline fetch window | `kFetchRatio=3`, `kPreserveRatio=2`, `kRefetchZoomRatio=8` | Holds last fetch range; refetch on deep zoom | M | observed (`timeline.cc:3149-3212`) |
| S19 | Legacy Catapult model | `tr.Model` + `_loadedTraceEvents` Set of hash keys | Unbounded growth on pan/zoom **merge** path | **H** | observed (`tf-trace-viewer.html:1969-2005`) |
| S20 | Event args cache (Angular) | `eventArgsCache: Map<string, Record>` | Unbounded if user selects many events | M | observed (`trace_viewer.ts:202`, `:893`) |
| S21 | stack_trace_page | Query strings → component fields | URL + field retention; secondary to parent tool | L–M | observed (`stack_trace_page.ts:36-76`) |
| S22 | HLO adjacent nodes cache | `hloAdjacentNodesCache` Map | Grows with selected HLO ops | M | observed (`trace_viewer.ts:203-206`) |

### 5.1 Severity heatmap by process

```text
Plugin C++ peak (cold multi-host Reduce):
  [XSpace arena × concurrent Maps] + [TraceEventsContainer × N threads]
  + [merged container] + [JSON string OR TraceDataResponse+zstd]
  + [gzip buffer in Python]

Browser peak (V2 JSON):
  [gzip inflate] + [JSON object graph] + [WASM SoA + entry_args]
  + [optional previous timeline until replace via SetTimelineData move]

Browser peak (legacy Catapult streaming):
  [full model accumulation across pan/zoom]  ← often worse long session
```

### 5.2 Visibility / streaming threshold semantics (critical)

`LoadTraceEventsContainer` constructs `TraceVisibilityFilter(MilliSpan(start,end), resolution, …)` and passes `kDisableStreamingThreshold = 500000` into `LoadFromLevelDbTable` (`trace_view_options.cc:110-119`).

In `DoLoadFromLevelDbTable` (`trace_events.h:241-247`):

```text
filter_by_visibility =
    (threshold == -1) OR (!trace.has_num_events()) OR (num_events >= threshold);

if (!filter_by_visibility) {
  visibility_filter->UpdateVisibility(0);  // disable density filtering
}
```

**Observed consequences:**

1. Traces with **fewer than 500k events** get **no visibility downsampling** → load denser than large traces for the same viewport/resolution. Counter-intuitive: “small” traces can still inflate browser RAM if they pack heavy args.
2. `resolution=0` (V2 when `selected_group_ids` filter present, or default before canvas width known) means “no duration-based downsampling” even when visibility is enabled (`trace_viewer_visibility.cc:38-39`; `main.ts:657-660`).
3. `load_metadata` defaults **false** on viewport load (`trace_events.h:941`) — good for bulk args; selection path uses `ReadFullEventFromLevelDbTable` instead.
4. “Streaming” means **re-query LevelDB with a new timespan/resolution**, not HTTP chunked transfer.

### 5.3 Visibility algorithm (density downsampling)

`TraceViewerVisibility::Visible` (`trace_viewer_visibility.cc:30-42`):

1. Instant visible_span → always visible (cannot usefully filter).
2. Event outside span → not visible.
3. `resolution_ps == 0` → always visible (no downsampling).
4. Else `VisibleAtResolution`:
   - **Counters** (no resource_id): keep if distance from last visible counter ≥ resolution_ps.
   - **Complete events**: keep if duration ≥ resolution_ps OR gap from last end at same nesting depth ≥ resolution_ps.
   - **Flows**: first event visibility shared across flow_id; FLOW_END erases from map to bound flow map growth (`:100-103`).

Resolution in picoseconds is derived from requested `resolution` (bins) and visible duration via `TraceVisibilityFilter` (`trace_viewer_visibility.h:179-202` area).

---

## 6. Wire format comparison: JSON vs PB vs gzip layering

### 6.1 JSON path (production default)

| Layer | What | Path:line |
|-------|------|-----------|
| C++ container | Arena events, names, optional raw_data | after Load/Merge |
| C++ JSON string | Full catapult document via `TraceEventsToJson` + `IOBufferAdapter` | `streaming_trace_viewer_processor.cc:367-385` |
| Python | `data = raw_data` for `@` (bytes/str of JSON) | `raw_to_tool_data.py:143` |
| HTTP | `body.encode('utf-8')` if str; then **always** `gzip.compress(body)` | `respond.py:56-57`, `:109-111` |
| Browser | gunzip (Content-Encoding) → `response.json()` → object graph | `main.ts:711-717` |
| WASM | `processTraceEvents(jsonData)` packs SoA | `main.ts:873` |

**Coexistence peak:** merged container + JSON string (C++) then body + gzip(body) (Python) then inflate + JS graph + WASM SoA (browser).

### 6.2 PB / delta-series path (opt-in, mostly unwired)

| Layer | What | Path:line |
|-------|------|-----------|
| Columnar build | `TraceDataResponse` with complete/async/counter series; deltas of timestamps; interned strings | `delta_series/..._proto.h:102-166`; `.cc:87-149` |
| Serialize | `response.SerializeToString` | `.h:111-114` |
| Zstd | `ZstdCompression::Compress` level **1** | `zstd_compression.cc:17-29` |
| C++ output | `SetOutput(compressed, application/octet-stream)` | `streaming_trace_viewer_processor.cc:360` |
| Python | `data = raw_data`; content_type octet-stream | `raw_to_tool_data.py:144-146` |
| HTTP | **gzip.compress on already-zstd bytes** | `respond.py:109-111` + `tool_data.py:134` |
| Browser gate | **Only if `urlObj.pathname.endsWith('.pb')`** | `main.ts:802` |
| WASM | `_malloc` copy + `processCompressedTraceEvents` + `_free` | `main.ts:823-838` |

**Anti-pattern (observed):** double compression — zstd inside, gzip outside — wastes CPU and briefly holds **uncompressed_proto + zstd + gzip(zstd)** peaks in plugin process.

### 6.3 Non-stream Python JSON join

`TraceEventsJsonStream` (`trace_events_json.py:32-105`):

- Conceptually streaming iterator of chunks (header, `json.dumps` per event, trailing comma, fake `{}`).
- Call site **joins entire stream** into one `str` (`raw_to_tool_data.py:45`).
- Opportunity: stream chunks into gzip writer without full join (T6/T7 family).

### 6.4 Content-Type and Content-Encoding matrix

| Path | content_type | content_encoding set by service | respond() behavior |
|------|--------------|--------------------------------|--------------------|
| TV@ JSON | `application/json` | `None` | Force gzip header + compress |
| TV@ PB | `application/octet-stream` | `None` | Force gzip on zstd payload |
| TV non-stream JSON | `application/json` | `None` | gzip |
| Error / No Data | `text/plain` | n/a | gzip of short string |

`respond` docstring notes content_encoding can skip auto-gzip (`respond.py:44-46`) but ToolDataService never passes it (`tool_data.py:134`).

### 6.5 Density comparison (qualitative)

| Format | Redundancy | Args | Interning |
|--------|------------|------|-----------|
| Catapult JSON | Full name strings per event; decimal ts/dur | Nested `args` objects if present | None |
| TraceDataResponse PB | Columnar deltas; name_refs | Metadata series; counters via extractor | `interned_strings` map (`_proto.cc:28-37`) |
| After zstd level 1 | High compressibility on deltas | Compact | Intern table once |

**Hypothesized:** PB+zstd body often ≪ JSON for same event set; **but** double-gzip and unused FE path erase production wins today.

---

## 7. Streaming (`trace_viewer@`) vs non-streaming (`trace_viewer`)

| Aspect | `trace_viewer` (non-stream) | `trace_viewer@` (stream / LevelDB) |
|--------|----------------------------|-------------------------------------|
| Registration | XPLANE_TOOLS | XPLANE_TOOLS; catalog prefers this (`catalog.py:85-89`) |
| Multi-host C++ | `ConvertXSpaceToTraceEvents` requires **XSpaceSize()==1** (`xplane_to_tools_data.cc:88-91`) | ProcessSession loop or Map/Reduce multi-host |
| First request | XSpace → Trace events **string** (C++) → Python `process_raw_trace` | XSpace → container → **3 SSTABLEs** → filtered load → JSON/PB |
| Later pan/zoom | Full reconvert unless external result cache | SSTABLE hit; load only filtered slice |
| Viewport params | Limited / ignored in legacy path | `start/end_time_ms`, `resolution`, search, event_name |
| Output path | Always through Python JSON join for non-pb | C++ JSON string **or** zstd PB; Python does **not** re-parse `@` |
| Disk intermediates | None (unless tool result cache) | 3 LevelDB tables per host |
| Worker service | N/A single-host assumption | `ShouldUseWorkerService` when `XSpaceSize() > 1` |
| Cloud AI | May be the only option | SSTABLE paths may be Unimplemented |
| Peak first-open | XSpace + Trace string + Python Trace proto + joined JSON + gzip | XSpace + container + LevelDB write buffers + load + JSON/PB + gzip |
| Peak subsequent | Same as first if no cache | LevelDB read + filtered container + serialize + gzip |

### 7.1 Non-stream double materialization (detail)

1. C++ returns serialized Trace / events string (`ConvertXSpaceToTraceEventsString`).
2. Unless `format=='pb'`, Python runs `process_raw_trace`:
   - `trace_events_old_pb2.Trace().ParseFromString(raw_trace)` (`raw_to_tool_data.py:43-44`)
   - `''.join(TraceEventsJsonStream(trace))` (`:45`)

### 7.2 Stream path does not use Python JSON converter

For `trace_viewer@`, success path sets `data = raw_data` directly (`raw_to_tool_data.py:143`). Avoids Python proto re-parse—but still pays C++ full `std::string` JSON and Python gzip.

---

## 8. Multi-host: ProcessSession vs Reduce (code-cited peak differences)

### 8.1 Policy matrix

| Mechanism | Code | Behavior |
|-----------|------|----------|
| Multi-host CSV | `supports_multi_host_selection` → only `trace_viewer` / `trace_viewer@` (`registry.py:99`, `hosts.py:69`) | `hosts=h1,h2,…` → exact list; missing host → FileNotFoundError |
| ALL_HOSTS | `host == ALL_HOSTS` (`hosts.py:79-81`) | All XPlanes in run_dir |
| Empty host | `not host and not hosts_param` (`hosts.py:95-98`) | **Also all hosts** (legacy) |
| FE cap | `hostsList.sort().slice(0, 10)` (`trace_viewer.ts:556`) | Silent truncation to 10 hosts |
| Worker path | `ShouldUseWorkerService` if `XSpaceSize() > 1` (`streaming_trace_viewer_processor.h:46-50`) | Map per host; Reduce parallel load |
| ProcessSession parallel TODO | `b/452217676` (`streaming_trace_viewer_processor.cc:72`) | Latency vs peak tradeoff if naively parallelized |

**Dangerous combo (observed):** FE fails to pass `hosts`/`host` correctly → HostSelector falls into “use all hosts” → convert attempts entire session even when user thought single host.

### 8.2 ProcessSession peak shape (serial)

```text
# streaming_trace_viewer_processor.cc:72-149
for each host i:
  peak_i ≈ max(
    cold: size(XSpace_arena) + size(full TraceEventsContainer) + store_buffers,
    warm: size(filtered Load container)
  ) + size(merged_so_far after hosts 0..i)
# after loop:
peak_serialize ≈ size(merged_final) + size(JSON or PB+zstd)

ProcessSession_peak ≈ max_over_hosts(preprocess_or_load) + size(merged_so_far)
                      + serialize(merged_final)
```

Host container is `std::move`'d into merge (`:145`) — host container should free after merge. **Only one host load/preprocess peaks at a time**, plus growing merged.

### 8.3 Reduce peak shape (parallel load)

```text
# streaming_trace_viewer_processor.cc:297-328
num_threads = min(num_hosts, MaxParallelism())
vector<StatusOr<TraceEventsContainer>> trace_containers(num_hosts);  // all slots live
parallel for i: trace_containers[i] = LoadTraceContainerForHost(...)
JoinAll  // ALL successful containers still in vector
serial for i: merged.Merge(move(trace_containers[i]), i+1)

Reduce_peak ≈ sum_over_concurrent_hosts(filtered_container(host))
              + merged
              + serialize
```

Thread count can be all hosts up to hardware parallelism. For **10 hosts** with large filtered windows, **sum of concurrent containers** is the dominant **hypothesized** OOM mode on plugin/worker.

### 8.4 Merge semantics

`merged_trace_container.Merge(std::move(trace_container), i + 1)` remaps device/process IDs by host index (`:145`, `:326`). Merged container holds **union** of filtered events from all hosts for the viewport—multi-host is not “metadata only.”

### 8.5 Multi-host × overfetch interaction

V2 expands time range by `FETCH_RATIO=3` and resolution by `ZOOM_RATIO=8` **per request**, then multiplies by N hosts:

```text
events ≈ f(viewport × FETCH_RATIO, resolution_bins × ZOOM_RATIO) × N_hosts
```

Severity **H** when N=10 and resolution falls back to 0 (full density).

### 8.6 Map phase cold multi-host

Map may run per host on workers; each Map that misses SSTABLE holds XSpace copy (`temp_xspace = xspace` at `:207`) + full container + store buffers. If many Maps concurrent on one machine (**hypothesized** worker topology dependent), cold peak can exceed Reduce load peak.

---

## 9. Peak memory narratives by user action

### 9.1 First-open (cold SSTABLE missing, V2 JSON, multi-host ProcessSession)

**User action:** Navigate to Trace Viewer for a session never opened before.

**Structures that live when:**

| Phase | C++ | Python | Browser |
|-------|-----|--------|---------|
| Preprocess host 0 | XSpace arena + full container + write buffers (≤20 chunks, 20MiB blocks) | waiting | idle / spinner |
| After store host 0 | arena freed; SSTABLE on disk; load filtered; merge | waiting | idle |
| Hosts 1..N serial | same pattern; merged grows | waiting | idle |
| Serialize | merged + JSON string | — | idle |
| respond | — | JSON bytes + gzip bytes | fetch buffer |
| Parse/process | freed after response | freed | inflate + JSON graph + WASM SoA + entry_args |

**Peak location:** often **plugin cold preprocess** (S4–S5) for huge hosts; after warm, **browser JSON+WASM** for interactive use.  
**Severity:** H | **Confidence:** observed structure; magnitude hypothesized.

### 9.2 Pan / zoom (warm SSTABLE, V2)

**User action:** Pan outside preserve range or zoom past `kRefetchZoomRatio`.

**FE state:**

1. `Timeline::MaybeRequestData` (`timeline.cc:3149-3212`) computes preserve=viewport×2, fetch=viewport×3; if preserve not ⊆ last_fetch or zoomed_in_too_much → emit fetch event; set `is_incremental_loading_=true`.
2. Angular/WASM builds new URL with expanded `start_time_ms`/`end_time_ms` and resolution.
3. `loadTraceData(url)` (`main.ts:1048-1093`):
   - If same URL as current and promise exists → reuse promise.
   - Else set `currentDataUrl = url`, start new promise.
   - `isAbortRequested: () => url !== currentDataUrl` — if user pans again, processing is skipped after fetch, but **fetch may complete**.
4. `SetTimelineData` **moves** new SoA in (`timeline.cc:349-353`) — previous SoA released on assign (move).

**Live structures mid-refetch (worst):** old SoA + in-flight JSON graph + new SoA being packed.  
**Severity:** M–H | **Confidence:** observed dual-resident windows; abort partial.

### 9.3 Search

**User action:** type in search box (container debounces 300ms — `trace_viewer_container.ts:388`).

**Path:**

1. FE emits search → URL/query with `search_prefix` (and optional `search_metadata`).
2. Server: `SearchInLevelDbTable` if trie exists (`trace_view_options.cc:98-108`).
3. V2: `loadSearchResults` (`main.ts:1096-1130`) → `setSearchResultsInWasm` / compressed variant — **separate** from main timeline load.
4. Search results retained in `search_results_` and reconciled on `SetTimelineData` (`timeline.cc:355-386`).

**Memory:** matching event set + main timeline still resident. Unbounded search result size is **hypothesized** risk if prefix is very short.  
**Severity:** M | **Confidence:** observed dual path; bound size hypothesized.

### 9.4 Event detail selection

**User action:** click a single event.

**Path:**

1. WASM dispatches selection; Angular `maybeFetchEventArgs` (`trace_viewer.ts:813-896`).
2. Cache scan with 50ms fuzzy start match (`:831`).
3. On miss: second `/data` with `event_name`, `start_time_ms`, `duration_ms`, `unique_id` — **format forced json** (`trace.py:30-33`).
4. Server: `ReadFullEventFromLevelDbTable` metadata path (`trace_view_options.cc:91-97`).
5. Args stored in unbounded `eventArgsCache` (`trace_viewer.ts:893`).
6. May trigger HLO adjacent-nodes fetch (`maybeFetchAdjacentNodes` `:909+`) — second tool convert.

**Live structures:** main timeline SoA (still has `entry_args` maps!) + small detail JSON + cache entries. Contradicts incomplete migration away from eager args (TODO `b/474668991` at `timeline.h:106-108`).  
**Severity:** M (detail); H if entry_args not dropped | **Confidence:** observed.

### 9.5 First-open vs pan comparison table

| Dimension | First-open cold | Pan/zoom warm |
|-----------|-----------------|---------------|
| XSpace decode | Yes (per missing host) | No |
| LevelDB write | Yes | No |
| LevelDB read | Yes | Yes |
| Multi-host Reduce risk | Yes if worker path | Yes (load N) |
| Browser dual model | Unlikely | Yes (old+new briefly) |
| Catapult Set growth | Seed only | Unbounded merge |

---

## 10. Frontend state machine (V2): MaybeRequestData / abort / replace timeline

### 10.1 Loading status enum events

Dispatched as `LOADING_STATUS_UPDATE_EVENT_NAME` with statuses used in `fetchAndProcessTraceData` (`main.ts:793-896`):

```text
IDLE → LOADING_DATA → PROCESSING_DATA → IDLE
                  ↘ ERROR
```

### 10.2 `loadTraceData` state

```text
state:
  currentDataUrl: string | null
  currentLoadingPromise: Promise<void> | null

loadTraceData(url):
  if url == currentDataUrl && currentLoadingPromise:
    return currentLoadingPromise          # dedupe
  currentDataUrl = url
  currentLoadingPromise = async:
    parse URL
    if time range in URL: expandUrlTimeRange (FETCH_RATIO)
    fetchAndProcessTraceData(
      isAbortRequested = () => url !== currentDataUrl
    )
    finally: if url==currentDataUrl: currentLoadingPromise=null
  return currentLoadingPromise
```

**Abort semantics (observed):** does **not** call `AbortController` on `fetch`. In-flight network and `response.json()` may fully allocate; only **post-fetch** processing checks `isAbortRequested` (`main.ts:804`, `:819`, `:842`, `:869`). Mid-flight memory is not cancelled.

### 10.3 `fetchAndProcessTraceData` branches

```text
updateUrlWithResolution(canvas)
if pathname ends with .pb:
  arrayBuffer → check abort → malloc → processCompressed → free
else:
  response.json() → check abort → validate isTraceData → processTraceEvents
measure traceProcessingTime
set window.wasmMemoryBytes = HEAPU8.length
dispatch IDLE
```

### 10.4 Timeline MaybeRequestData machine

From `timeline.cc:3149-3212` and comments `:3153-3161`:

```text
ranges:
  data_time_range_          full trace duration
  last_fetch_request_range_ last requested fetch window
  preserve = visible * kPreserveRatio(2)
  fetch    = visible * kFetchRatio(3)  (min duration kMinFetchDurationMicros)

guards:
  if is_incremental_loading_: return
  if not zoomed_in_too_much and last_fetch contains preserve: return
  zoomed_in_too_much := last_fetch.duration / fetch.duration > kRefetchZoomRatio(8)

actions:
  emit kFetchData with start/end ms
  last_fetch_request_range_ = fetch
  is_incremental_loading_ = true   # cleared when data arrives
```

### 10.5 Replace timeline

`SetTimelineData` (`timeline.cc:349-353`):

```text
UpdateLevelPositions(data);
timeline_data_ = std::move(data);  // old SoA destroyed here
// reconcile search_results_ loaded_index against new entry_event_ids
```

**Observed good:** move assignment frees previous SoA.  
**Observed gap:** JS JSON graph from previous fetch is GC-dependent, not explicit nulling.

### 10.6 Resolution computation machine

```text
updateUrlWithResolution (main.ts:651-676):
  resolution = 0  # default = full density
  if NOT selected_group_ids:
    if canvas and (clientWidth - 250) > 0:
      resolution = round(viewerWidth / 2) * 8
  params.set(resolution)
```

Before first layout (canvas width 0) or with group filter → **resolution=0** → no downsampling server-side (when visibility enabled).

### 10.7 Angular shell ownership

`trace_viewer.ts` holds across session:

| Object | Lifetime | Path:line |
|--------|----------|-----------|
| `hostList` | Navigation | `:178`, `:410` |
| `featureFlags` + localStorage | Session | `:242`, `:133-151` |
| `eventArgsCache` | Component life | `:202` |
| `hloAdjacentNodesCache` | Component life | `:203-206` |
| `traceViewerModule` | Until destroy | `:195`; shutdown `:622` |
| `processes` maps | After load | `:231` |

**Observed gap:** `use_pb` appears in feature flag dialog (`feature_flags.ts:20-24`) but `update()` builds `getDataUrl` without `format=pb` (`trace_viewer.ts:592-597`). V2 loader only takes PB branch when pathname ends `.pb` (`main.ts:802`). Production V2 is **JSON-only** unless external URL rewriting exists outside this tree.

### 10.8 Container search debounce

`trace_viewer_container.ts:386-394`: `search$.pipe(debounceTime(300), distinctUntilChanged)` → emit `searchEvents`. Does not cancel in-flight search HTTP.

---

## 11. Catapult accumulation algorithm (hash growth model)

Source: `plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html`.

### 11.1 Event hash key

```text
_getEventHashKey(event)  # :1927-1940
  key = str(pid) + "$" + (ts or 0) [+ "$" + z if present]
  metadata events without ts → key with ts=0 (single slot; OK for static metadata)
```

### 11.2 Filter / accumulate

```text
_filterKnownTraceEvents(traceEvents)  # :1969-1987
  if _loadedTraceEvents.size == 0:
    add all keys; return all events          # first batch seeds Set
  else:
    for each event:
      if key not in Set: add key; keep event
      else: _maybeUpdateEventArguments(event)  # merge args into existing slice
    return filtered (new only)
```

### 11.3 Model update

```text
_updateModel(jsonData, replaceModel)  # :1994+
  if first load OR replaceModel:
    new tr.Model(); new Set(); reset async maps, fullBounds, tasks
  else:
    delete metadata fields to avoid accumulation of header junk
  split counter events (bypass hash filter — RangeError risk :2071-2075)
  otherEvents = _filterKnownTraceEvents(otherEvents)
  importTraces into existing _model
```

### 11.4 Growth model

```text
Let F_i = event set returned by server for viewport fetch i (visibility-limited).
Let U_n = union of F_1 .. F_n after user explores.

|_loadedTraceEvents| ≈ |U_n|     # one string key per unique pid$ts[$z]
|tr.Model slices|    ≈ |U_n|     # plus process/thread metadata

Memory_keys ≈ |U_n| * avg_key_bytes   # typically tens of bytes
Memory_model ≈ |U_n| * avg_slice_bytes  # args, stacks, etc. — much larger

If user pans across full trace duration at high resolution,
U_n → entire visible density of trace (minus per-fetch visibility),
not "one screen".
```

**Severity H** for long multi-host legacy sessions. Counter events intentionally skip the Set to avoid huge numeric hash issues (`:2071-2075`) — they re-expand via `flatMap` entries each fetch.

### 11.5 Args merge on duplicate

`_maybeUpdateEventArguments` (`:1943-1963`) walks thread slices for `abs(slice.start - tsMs) < 0.001` and copies args — can densify existing slices without growing Set.

---

## 12. Compressed delta-series (PB) internals

### 12.1 Conversion pipeline

`ConvertTraceDataToCompressedDeltaSeriesProto` (`..._proto.h:82-119`):

1. Build counter extractor that parses `RawData` args for double/uint counters.
2. `DeltaSeriesProtoConverter` with intern map (empty string id 0).
3. `GenerateResponse`: metadata processes/threads; details flags; full timespan; for each track:
   - complete → deltas, durations, name_refs, event_metadata
   - async (flow_id) → async series
   - else counter series
4. Move interned strings into response.
5. `SerializeToString` → `ZstdCompression::Compress`.

### 12.2 Intermediate peaks (plugin)

```text
merged container  (still live)
+ TraceDataResponse arena-like fields (columnar vectors)
+ serialized_proto (uncompressed)
+ compressed_result (zstd)
# then Python:
+ gzip(compressed_result)
```

**Hypothesized** intermediate peak order: `max(container+response, response+serialized, serialized+compressed, compressed+gzip)`.

### 12.3 Browser PB path (when enabled)

```text
ArrayBuffer (gzip already stripped by browser)
+ WASM HEAPU8 copy (malloc)
+ zstd decompress buffer (freed early inside processCompressed)
+ SoA FlameChartTimelineData
```

Duplicate buffer in WASM is **observed** (`main.ts:825-829`).

---

## 13. stack_trace_page depth (real field names)

### 13.1 Role (observed)

Angular bridge so Trace Viewer (non-Angular Catapult path) and tools like op_profile / hlo_stats / roofline can show source/HLO snippets. Comment in component: particularly useful because Trace Viewer is not Angular but `StackTraceSnippet` is (`stack_trace_page.ts:10-13`).

**Not** an `XPLANE_TOOLS` entry — no streaming convert.

### 13.2 Field names and query params

| Component field | Query / route key | Path:line |
|-----------------|-------------------|-----------|
| `hloModule` | `hlo_module` | `stack_trace_page.ts:29,67` |
| `hloOp` | `hlo_op` | `:30,68` |
| `sourceFileAndLineNumber` | `source` | `:31,69` |
| `stackTrace` | `stack_trace` | `:32,70` |
| `sessionId` | route `sessionId` **or** query `session_id` **or** `run` | `:33,61,74` |
| `programId` | parsed from `hloModule` via `/\((.*?)\)/` | `:42,71-73` |
| `opCategory` | `op_category` | `:34,75` |
| `sourceCodeServiceIsAvailable` | service token `isAvailable()` | `:40,49-57` |

### 13.3 Template binding

`stack_trace_page.ng.html`:

- Table shows `hloModule`, `hloOp`.
- `<source-mapper>` binds: `sessionId`, `programId`, `opName=hloOp`, `opCategory`, `sourceFileAndLineNumber`, `stackTrace`.
- TODO `b/428947160` — load stack trace from backend (currently URL-borne).
- Container only renders if `sourceCodeServiceIsAvailable`.

### 13.4 Cross-links

`constants.ts:129-132` defines `STACK_TRACE_TOOL_NAME = ['stack_trace_page', 'Source Code Snippet with IR Text']` for Trace Viewer cross-links.

### 13.5 Materialization / memory

| Stage | What is held | Confidence |
|-------|--------------|------------|
| Route query | Full `stack_trace` + HLO module strings in component fields | observed |
| SourceCodeService | Temporary availability check; may load file slices into snippet | observed |
| Parent tools | Parent tool tables stay resident while snippet open | observed (pattern) |
| Trace Viewer deep-link | New page/tab may leave Catapult/WASM session alive in background | hypothesized |

### 13.6 Opportunities (ST-*)

| ID | Stage | Opportunity | Severity | Confidence | Effort | Notes |
|----|-------|-------------|----------|------------|--------|-------|
| ST-1 | FE route | Cap / truncate very long `stack_trace` query strings | **L** | hypothesized | S | Keep hash/id; fetch full stack by session+op |
| ST-2 | FE bundle | Lazy-load `StackTraceSnippet` module only when open | **L** | hypothesized | S | |
| ST-3 | FE service | Release SourceCodeService file buffers on destroy | **M** | hypothesized | M | Audit service |
| ST-4 | FE IPC | Pass handles/IDs not full HLO text | **M** | hypothesized | M | Align with HLO extract cache |
| ST-5 | Trace deep-link | Open snippet without retaining full Catapult/WASM in background tab | **M** | hypothesized | M | Especially legacy iframe |
| ST-6 | URL size | Prefer `session_id` + `hlo_op` + source line | **L** | observed | S | Fields already support partial info |
| ST-7 | Parent tools | Virtualize parent table rows when stack panel open | **L** | hypothesized | M | Secondary |
| ST-8 | Backend TODO | Implement `b/428947160` load stack from backend; stop URL megastrings | **M** | observed TODO | M | Template TODO |

---

## 14. Anti-patterns already in code

These are **observed** bugs/gaps with direct memory or density impact—not proposed features.

### AP-1. Double gzip on already-zstd PB

- C++ zstd-compresses `TraceDataResponse` (`..._proto.h:116-118`).
- `ToolDataService` returns `content_encoding=None` (`tool_data.py:134`).
- `respond()` always gzip when encoding None (`respond.py:109-111`).
- **Effect:** plugin holds zstd body + gzip(zstd); browser gunzips to zstd then WASM zstd-decompresses.
- Severity **H** | Confidence **observed** | Effort S

### AP-2. `use_pb` feature flag unwired to data URL / pathname

- Flag defined default false (`feature_flags.ts:20-24`).
- Persisted via `xprof_ff_use_pb` (`trace_viewer.ts:274-279`; `main.ts:988`).
- Data URL construction does not set `format=pb` (`trace_viewer.ts:592-597`).
- V2 PB branch requires `.pb` pathname (`main.ts:802`) — production `/data` never matches.
- **Effect:** entire compressed columnar pipeline dead for production V2.
- Severity **H** | Confidence **observed** | Effort S–M

### AP-3. Visibility disable below 500k events

- `kDisableStreamingThreshold = 500000` (`trace_view_options.cc:114`).
- Below threshold: `UpdateVisibility(0)` (`trace_events.h:245-247`) → full density for timespan.
- **Effect:** “medium” traces can ship denser payloads than huge traces.
- Severity **M** | Confidence **observed** | Effort S

### AP-4. Accidental all-hosts fallback

- Missing both `host` and `hosts` → all XPlanes (`hosts.py:95-98`).
- Severity **H** | Confidence **observed** | Effort S

### AP-5. Abort without AbortController

- URL change only skips process step (`main.ts:804+`); fetch still allocates.
- Severity **M** | Confidence **observed** | Effort M

### AP-6. Eager `entry_args` despite detail API

- SoA stores `vector<flat_hash_map<string,string>>` per event (`timeline.h:111`) with TODO `b/474668991`.
- Selection already has `maybeFetchEventArgs` (`trace_viewer.ts:813`).
- Severity **H** | Confidence **observed** | Effort M

### AP-7. Catapult unbounded Set

- `_loadedTraceEvents` never capped on merge path (`tf-trace-viewer.html:1969-2005`).
- Severity **H** | Confidence **observed** | Effort S–M

### AP-8. Non-stream full join of streaming iterator

- `TraceEventsJsonStream` is iterable (`trace_events_json.py:97`) but joined (`raw_to_tool_data.py:45`).
- Severity **M** | Confidence **observed** | Effort M

### AP-9. Reduce holds all host containers until JoinAll

- `vector` of containers sized to N (`streaming_trace_viewer_processor.cc:299`); merge after JoinAll (`:311-326`).
- Severity **H** | Confidence **observed** | Effort M

### AP-10. Silent host truncation to 10

- `slice(0, 10)` (`trace_viewer.ts:556`) — user may not know remaining hosts omitted.
- Severity **M** (correctness + incomplete multi-host) | Confidence **observed** | Effort S

### AP-11. resolution=0 on first paint / group filter

- Default resolution 0 before canvas sized (`main.ts:657-672`); group ids force 0 (`:660`).
- Severity **M** | Confidence **observed** | Effort S

### AP-12. Unbounded Angular caches

- `eventArgsCache`, `hloAdjacentNodesCache` never LRU (`trace_viewer.ts:202-206`).
- Severity **L–M** | Confidence **observed** | Effort S

### AP-13. ProcessSession parallel TODO without memory budget

- `b/452217676` (`streaming_trace_viewer_processor.cc:72`) if implemented naively → ProcessSession peak approaches Reduce.
- Severity **H** (latent) | Confidence **observed** TODO | Effort M

---

## 15. Expanded opportunities T1–T45

Confidence column is **only** `observed` or `hypothesized`. Severity **H|M|L**. Prerequisites are other T-IDs.

| ID | Stage | Description | Severity | Confidence | Effort | Prerequisites | Notes |
|----|-------|-------------|----------|------------|--------|---------------|-------|
| T1 | FE+BE | Wire `format=pb` end-to-end for V2 when `use_pb` true: set query param; treat `/data` octet-stream as compressed PB (not only `.pb` pathname) | **H** | observed | M | — | Flag + C++ path exist; FE gate broken |
| T2 | HTTP | Skip default gzip when body is already zstd PB; set Content-Encoding appropriately or identity + zstd framing | **H** | observed | S | — | `respond.py` always gzips if None |
| T3 | FE V2 | Drop or lazy-load `entry_args` maps; rely on `event_name` detail fetch | **H** | observed | M | — | TODO b/474668991 |
| T4 | BE multi-host | Bound Reduce concurrency (semaphore) / free host containers ASAP; backpressure on Map | **H** | observed | M–L | — | Parallel load holds N containers |
| T5 | BE visibility | Revisit `kDisableStreamingThreshold=500000` and resolution=0 full-load cases | **M** | observed | S | — | Small traces load denser |
| T6 | BE JSON | Stream JSON into compressor without one giant `std::string` | **M** | hypothesized | L | — | IOBufferAdapter appends to string |
| T7 | Python non-stream | Eliminate `process_raw_trace` join; stream chunks or move JSON fully to C++ | **M** | observed | M | — | Only plain `trace_viewer` |
| T8 | BE merge | ProcessSession already moves; Reduce: merge incrementally as threads complete | **M** | observed | M | T4 | Reduces concurrent peak |
| T9 | Legacy FE | Disable Catapult model accumulation when V2 default; or hard-cap `_loadedTraceEvents` / replace model each viewport | **M** | observed | S–M | — | Set growth |
| T10 | FE fetch | AbortController; drop mid-flight JSON; free previous SoA before installing new | **M** | observed | M | — | Abort process-only today |
| T11 | LevelDB | Tune `block_size` 20 MiB and `kMaxBufferedChunks` for write peak | **M** | observed | S | — | Trade I/O vs RAM |
| T12 | Multi-host FE | Lower host cap below 10 or progressive host load | **M** | observed | S | — | slice(0,10) already |
| T13 | Overfetch | Reduce `ZOOM_RATIO` / `FETCH_RATIO` / `kRefetchZoomRatio` defaults | **M** | observed | S | — | 8× and 3× |
| T14 | PB + HTTP | Single compression: zstd only; document Content-Type + encoding | **H** | observed | S | T1,T2 | Complements T1/T2 |
| T15 | Cold preprocess | Parallel preprocess with memory budget; avoid XSpace+full container+chunks naively | **H** | hypothesized | L | T11 | First-open S4–S5 |
| T16 | Metadata SSTABLE | Never attach full args in viewport load; only metadata on selection | **H** | hypothesized | M | T3 | load_metadata already false default |
| T17 | HostSelector | Cap hostless all-hosts for trace tools; require explicit host | **M** | observed | S | — | Dangerous fallback |
| T18 | Worker service | Prefer ProcessSession serial shape when memory constrained vs Reduce | **M** | hypothesized | M | T4 | Config/flag |
| T19 | Event args cache | LRU-bound `eventArgsCache` / adjacent nodes cache | **L** | observed | S | — | Unbounded Maps |
| T20 | Search path | Bound search result set size server-side | **M** | hypothesized | M | — | Prefix trie explode |
| T21 | full_dma / filters | Cap density when full_dma or FILTER_CONFIG disables downsampling | **M** | hypothesized | S | T5 | UX vs RAM |
| T22 | WASM heap | Grow carefully; reset/recreate module on huge runs; report wasmMemoryBytes in UI | **M** | observed | M | — | HEAPU8 tracks reservation |
| T23 | Catalog | Keep forcing `@` over plain; retire non-stream path in OSS when safe | **L** | observed | L | T7 | Removes T7 path |
| T24 | Double gzip observability | Metrics for payload sizes pre/post gzip and zstd | **L** | hypothesized | S | T2 | Guides T2 |
| T25 | Timeline preserve | Lower `kPreserveRatio` after pan settles | **M** | hypothesized | S | T13 | constants.h |
| T26 | Multi-host JSON | Per-host JSON stream merge without full merged container | **H** | hypothesized | L | T1 | Architectural |
| T27 | Cloud AI | If forced non-stream, apply same PB/lazy-args wins | **M** | observed | M | T1,T7 | Unimplemented SSTABLE |
| T28 | Warm-up | Ensure DEFAULT_CACHE_TOOLS warm-up does not hold full JSON after write | **M** | hypothesized | M | — | Shared with 02 |
| T29 | Content-type | FE uses arrayBuffer path for octet-stream regardless of URL | **H** | observed | S | T1 | Part of T1 |
| T30 | ProcessSession parallel TODO | If implementing b/452217676, add memory semaphore first | **H** | observed | M | T4 | Prevents latent OOM |
| T31 | Zstd level | Evaluate zstd level >1 for PB size vs CPU at serialize | **L** | hypothesized | S | T1 | Currently level 1 |
| T32 | Counter flatten Catapult | Avoid re-flatMap full counter entries every merge fetch | **M** | hypothesized | M | T9 | :2076-2087 |
| T33 | Duplicate WASM malloc | Zero-copy or shared buffer for PB into WASM | **M** | observed | L | T1 | HEAPU8.set copy |
| T34 | Search abort | Cancel previous search HTTP on new query | **L** | hypothesized | S | T10 | Debounce only today |
| T35 | FE host UX | Warn when hosts truncated to 10 | **L** | observed | S | T12 | Silent slice |
| T36 | Resolution floor | Never send resolution=0 on first paint; use plugin default 8000 until canvas ready | **M** | observed | S | T5,T13 | main.ts default 0 |
| T37 | Gzip streaming | Stream gzip of JSON without full body buffer | **M** | hypothesized | L | T6 | respond.py |
| T38 | Worker Map isolation | One Map cold preprocess per machine at a time | **H** | hypothesized | M | T4,T15 | Topology-dependent |
| T39 | FlameChart SoA intern | Intern entry_names / arg keys in WASM | **M** | hypothesized | M | T3 | Reduces string dup |
| T40 | Flow maps bound | Cap flow_lines_by_flow_id retention when filters hide flows | **L** | hypothesized | S | — | timeline.h:122-124 |
| T41 | ProfileProcessor JSON path | Legacy ConvertXSpaceToTraceEvents streaming branch still builds JSON only (`xplane_to_tools_data.cc:155-167`) — ensure processor path is sole multi-host entry | **M** | observed | M | T23 | Avoid dual paths |
| T42 | Event detail host precision | Already uses pidToHostMap — ensure multi-host detail never re-fetches all hosts | **L** | observed | S | — | :857-866 |
| T43 | stack_trace URL | See ST-1/ST-6/ST-8 | **L–M** | observed | S | — | Secondary |
| T44 | Metrics: event counts | Log filtered event count after Load and after Merge | **L** | hypothesized | S | T24 | Measurement |
| T45 | Progressive multi-host FE | Load host 1 first, then append hosts 2..N with replaceModel semantics per host | **M** | hypothesized | L | T12,T8 | UX redesign |

---

## 16. Top 10 wins (ranked)

Ordered by **expected peak-RAM reduction × breadth of users × implementability**.

| Rank | ID | Win | Why first |
|------|-----|-----|-----------|
| 1 | **T1+T29** | Ship compressed PB pipeline for V2 | Removes catapult JSON object graph + huge `std::string` JSON; code already ~80% present |
| 2 | **T2+T14** | Single compression layer for PB | Stops gzip(zstd(data)); cuts plugin peak and CPU |
| 3 | **T3+T16** | Slim resident timeline (no per-event args maps) | Browser SoA stays dense; selection already has detail API |
| 4 | **T4+T8** | Bound multi-host concurrent containers | Prevents Reduce OOM on 10-host sessions |
| 5 | **T13+T5+T36** | Fetch less density by default | ZOOM/FETCH ratios + threshold/res=0 audit |
| 6 | **T17** | Require explicit host; kill accidental all-hosts | Cheap correctness+memory fix |
| 7 | **T9** | Stop Catapult accumulation (or force V2) | Long-session legacy leak |
| 8 | **T10** | AbortController + free previous model | Pan/zoom thrash |
| 9 | **T7** | Kill non-stream Python double parse | Remaining `trace_viewer` / cloud path |
| 10 | **T11+T15** | LevelDB write buffer / cold preprocess budget | First-open plugin OOM |

**Secondary:** T12 progressive hosts, T19 cache LRUs, T22 WASM reset, T28 warm-up hygiene, ST-*.

---

## 17. Risks matrix

| Opt | Correctness risk | UX risk | Rollout note |
|-----|------------------|---------|--------------|
| T1 PB | Proto schema skew; missing fields (flows, details) | Blank timeline if JSON fallback broken | Feature flag already `use_pb`; stage |
| T2 no double gzip | Clients that always gunzip blindly | None if Content-Encoding correct | Integration tests JSON+PB |
| T3 drop entry_args | Selection missing fields until second RTT | Slight delay on select | Keep event_name path fast; cache LRU |
| T4 concurrency bound | Latency ↑ on multi-host | Spinner longer; still correct | Make limit configurable |
| T5 threshold/res | Over-filter might hide short events | “Missing” slices at high zoom | Prefer refetch-on-zoom (exists) |
| T6 stream JSON | Partial response on crash harder to debug | Progressive paint possible later | Large engineering |
| T7 Python path | Golden tests for catapult JSON order | None if C++ JSON matches | Catalog prefers @ already |
| T9 Catapult | Break extensions depending on full model | Legacy users only | Gate on useTraceViewerV2 |
| T13 lower overfetch | More network RTTs on zoom | Stutter on slow nets | Tune ZOOM_RATIO 8→4 first |
| T17 require host | Breaks bookmarks without host | Explicit host picker errors | Soft warn then hard fail |
| T26 no merge container | Cross-host ID remap bugs | Wrong lanes | Architecture-level risk |
| ST-* stack_trace | Broken deep links if truncate too aggressively | Snippet incomplete | Cap with ellipsis + full fetch by id |

---

## 18. Worked qualitative RSS scenarios

All numeric multipliers below are **hypothesized** (labeled). Structural sequencing is **observed**.

### 18.1 Assumptions (global, hypothesized)

Let:

- `X_h` = RSS of decoded XSpace for host h  
- `C_h` = RSS of full TraceEventsContainer for host h  
- `W` = write buffer peak ≈ `kMaxBufferedChunks × avg_chunk_bytes + block_size` (order: tens–hundreds of MB **hypothesized**)  
- `F_h(v,r)` = RSS of filtered container for viewport v, resolution r  
- `J` = RSS of JSON string for merged filtered events  
- `P_u` = uncompressed TraceDataResponse; `P_z` = zstd(P_u); `P_g` = gzip(P_z)  
- `G` = browser JS object graph ≈ k_json × J (**hypothesized** k_json ∈ [2,5])  
- `S` = WASM SoA + entry_args ≈ k_soa × events (**hypothesized** entry_args can dominate)

### 18.2 Small session (1 host, ~100k events, warm SSTABLE, V2 JSON)

**Assumptions:** num_events < 500k → visibility disabled → F ≈ denser load for full timespan if start/end span whole trace.

```text
Plugin peak ≈ F_1 + J + gzip(J)
Browser peak ≈ inflate + G + S
Typical: browser peak > plugin peak (hypothesized)
```

### 18.3 Medium session (1 host, ~2M events, warm, V2 JSON, default viewport)

**Assumptions:** visibility ON; resolution ~6000–8000; FETCH_RATIO 3.

```text
Plugin peak ≈ F_filtered + J + gzip(J)
Browser peak ≈ G + S  (G often largest)
If entry_args dense: S may exceed G (hypothesized)
```

### 18.4 Large multi-host (10 hosts, cold open, Reduce, V2 JSON)

```text
Map phase (per worker, hypothesized concurrent Maps):
  peak_map ≈ max or sum(X_h + C_h + W) depending on scheduling

Reduce phase (observed structure):
  peak_reduce ≈ sum_{h=1..T} F_h + merged + J + gzip
  where T = min(10, MaxParallelism())

Browser:
  peak_browser ≈ G(N_hosts events) + S
  N_hosts multiplies event count approximately linearly (hypothesized if similar hosts)
```

**Dominant OOM risk:** Reduce concurrent containers (plugin) **or** JSON graph (browser) — whichever hits first depends on host density and machine RAM (**hypothesized**).

### 18.5 Large multi-host ProcessSession (serial)

```text
peak ≈ max_h(X_h + C_h + W) + merged_final + J + gzip
# lower than Reduce by roughly factor of concurrency (hypothesized)
# longer wall time (observed serial loop)
```

### 18.6 Legacy Catapult long session

```text
after n pans across full duration:
  Set size ≈ |U_n| keys
  tr.Model ≈ union of all fetched events
peak_browser_legacy ≫ single-viewport V2 budget (hypothesized large factor)
```

### 18.7 Ideal future (T1+T2+T3+T4)

```text
wire: zstd PB only (no gzip wrap)
plugin: one host F + P_u→P_z (no J, no P_g)
browser: decompress → SoA without args maps
Reduce concurrency ≤ 2–4
peak dominated by visible SoA + one host container
```

### 18.8 Event-detail spam

```text
main S resident + eventArgsCache grows with selections
secondary graph_viewer adjacent nodes may spike convert RSS (hypothesized)
```

### 18.9 Search heavy prefix

```text
main timeline S + search result structure + optional second payload
if prefix="": undefined / empty; short prefix may match huge set (hypothesized)
```

---

## 19. Measurement plan (analysis only — no implementation)

Goal: replace hypothesized multipliers with measured RSS/heap. Hooks only.

### 19.1 Plugin / C++ metrics

| Metric | Where to hook | Unit |
|--------|---------------|------|
| XSpace arena bytes | After `GetXSpace` (`streaming_trace_viewer_processor.cc:99-100`) | bytes / RSS delta |
| Full container events + estimated bytes | After `ConvertXSpaceToTraceEventsContainer` (`:108-109`) | count + RSS |
| Store buffer peak | Inside `DoStoreAsLevelDbTables` around chunk submit (`trace_events.cc:569-674`) | max buffered chunks × sizes |
| SSTABLE file sizes | After store, `GetFileSize` on 3 paths | bytes disk |
| Loaded event count | After `LoadTraceEventsContainer` | count; also `filter_by_visibility` flag |
| Concurrent containers | Reduce: sizeof live StatusOr containers at JoinAll (`:311`) | count + RSS |
| Merged event count | After all merges | count |
| JSON string size | After `TraceEventsToJson` (`:379-380`) | `trace_viewer_json.size()` |
| PB uncompressed / compressed | After SerializeToString / Zstd (`..._proto.h:111-118`) | two sizes |
| Wall times | Existing LOG durations (`:125-127`, `:141-148`, `:381-382`) | already partial |

**RSS method (hypothesized tooling):** `/proc/self/statm` or `absl` memory sampling around phases; or tcmalloc heap profiler scopes.

### 19.2 Python / HTTP metrics

| Metric | Where | Unit |
|--------|-------|------|
| Body size pre-gzip | `respond.py` before `:111` | bytes |
| Body size post-gzip | after `gzip.compress` | bytes |
| content_type / format | ToolResult | label |
| Double-compress flag | format==pb && default gzip path | boolean counter |
| process_raw_trace join size | non-stream path only | bytes |

### 19.3 Browser metrics

| Metric | Where | Unit |
|--------|-------|------|
| `performance.measure('traceProcessingTime')` | already `main.ts:876-880` | ms |
| `window.wasmMemoryBytes` | already `:888-890` | bytes HEAPU8 |
| JS heap | `performance.memory.usedJSHeapSize` (Chrome) around before/after `response.json` and after `processTraceEvents` | bytes |
| Dual-resident window | sample heap while `is_incremental_loading_` true | bytes |
| Catapult `_loadedTraceEvents.size` | after each pan | count |
| `eventArgsCache.size` | after N selections | count |
| Network transfer size | PerformanceResourceTiming for `/data` | encoded/decoded body |

### 19.4 A/B scenarios to run

1. `use_pb` true vs false (after T1 wired) same session.  
2. Host count 1 vs 5 vs 10.  
3. ZOOM_RATIO 8 vs 4; FETCH_RATIO 3 vs 2.  
4. ProcessSession-only vs Reduce (force worker off/on).  
5. Traces with 100k vs 600k vs 5M events (threshold effect).  
6. V2 vs Catapult after 20 pan operations.  
7. Cold vs warm SSTABLE first-open.  
8. `selected_group_ids` (resolution=0) vs normal.

### 19.5 Success criteria (design-level)

| Scenario | Target signal (hypothesized until measured) |
|----------|-----------------------------------------------|
| PB e2e | Plugin peak and browser JS heap both drop vs JSON |
| No double gzip | post-gzip size ≈ zstd size (or identity) |
| entry_args drop | wasmMemoryBytes drop proportional to map overhead |
| Reduce bound | plugin RSS flat vs host count beyond concurrency limit |
| Host required | zero accidental all-host converts in logs |

### 19.6 What not to implement in this note

No product code, no permanent log spam in production paths without sampling, no changes to respond()/FE in this analysis pass.

---

## 20. Explicit real paths list

All paths below were verified present in this workspace at write time.

### 20.1 Plugin / Python convert

| Path |
|------|
| `plugin/xprof/convert/raw_to_tool_data.py` |
| `plugin/xprof/convert/trace_events_json.py` |
| `plugin/xprof/profile_plugin/tools/options/trace.py` |
| `plugin/xprof/profile_plugin/tools/options/registry.py` |
| `plugin/xprof/profile_plugin/tools/registry.py` |
| `plugin/xprof/profile_plugin/tools/catalog.py` |
| `plugin/xprof/profile_plugin/tools/filenames.py` |
| `plugin/xprof/profile_plugin/services/tool_data.py` |
| `plugin/xprof/profile_plugin/services/hosts.py` |
| `plugin/xprof/profile_plugin/api/data_api.py` |
| `plugin/xprof/profile_plugin/http/respond.py` |
| `plugin/xprof/profile_plugin/http/parse_request.py` |
| `plugin/xprof/profile_plugin/constants.py` |
| `xprof/pywrap/profiler_plugin_impl.cc` |

### 20.2 C++ convert / streaming

| Path |
|------|
| `xprof/convert/streaming_trace_viewer_processor.cc` |
| `xprof/convert/streaming_trace_viewer_processor.h` |
| `xprof/convert/trace_view_options.cc` |
| `xprof/convert/trace_view_options.h` |
| `xprof/convert/xplane_to_tools_data.cc` |
| `xprof/convert/xplane_to_trace_container.cc` |
| `xprof/convert/xplane_to_trace_container.h` |
| `xprof/convert/xplane_to_tool_names.cc` |
| `xprof/convert/trace_viewer/trace_events.cc` |
| `xprof/convert/trace_viewer/trace_events.h` |
| `xprof/convert/trace_viewer/lite_trace_events.cc` |
| `xprof/convert/trace_viewer/trace_events_to_json.cc` |
| `xprof/convert/trace_viewer/trace_events_to_json.h` |
| `xprof/convert/trace_viewer/trace_viewer_visibility.cc` |
| `xprof/convert/trace_viewer/trace_viewer_visibility.h` |
| `xprof/convert/trace_viewer/trace_options.cc` |
| `xprof/convert/trace_viewer/trace_options.h` |
| `xprof/convert/trace_viewer/prefix_trie.cc` |
| `xprof/convert/trace_viewer/delta_series/trace_data_to_compressed_delta_series_proto.cc` |
| `xprof/convert/trace_viewer/delta_series/trace_data_to_compressed_delta_series_proto.h` |
| `xprof/convert/trace_viewer/delta_series/zstd_compression.cc` |
| `xprof/convert/trace_viewer/delta_series/zstd_compression.h` |

### 20.3 Frontend Trace Viewer

| Path |
|------|
| `frontend/app/components/trace_viewer/trace_viewer.ts` |
| `frontend/app/components/trace_viewer/constants.ts` |
| `frontend/app/components/trace_viewer/utils.ts` |
| `frontend/app/components/trace_viewer/trace_viewer.ng.html` |
| `frontend/app/components/trace_viewer_container/trace_viewer_container.ts` |
| `frontend/app/components/trace_viewer_container/trace_viewer_container.ng.html` |
| `frontend/app/components/trace_viewer_v2/main.ts` |
| `frontend/app/components/trace_viewer_v2/feature_flags.ts` |
| `frontend/app/components/trace_viewer_v2/timeline/timeline.h` |
| `frontend/app/components/trace_viewer_v2/timeline/timeline.cc` |
| `frontend/app/components/trace_viewer_v2/timeline/constants.h` |
| `frontend/app/components/trace_viewer_v2/timeline/data_provider.cc` |
| `frontend/app/components/trace_viewer_v2/timeline/data_provider.h` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_pb_event_parser.cc` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_pb_event_parser.h` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_event_parser.cc` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_event_packer.cc` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_event_packer.h` |
| `frontend/app/components/trace_viewer_v2/trace_helper/trace_event_parser_core.cc` |
| `frontend/app/components/trace_viewer_v2/event_manager.cc` |
| `frontend/app/components/trace_viewer_v2/application.cc` |

### 20.4 Catapult / plugin static viewer

| Path |
|------|
| `plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html` |
| `plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer-helper.js` |
| `plugin/trace_viewer/trace_viewer.html` |

### 20.5 Stack trace bridge

| Path |
|------|
| `frontend/app/components/stack_trace_page/stack_trace_page.ts` |
| `frontend/app/components/stack_trace_page/stack_trace_page.ng.html` |
| `frontend/app/components/stack_trace_snippet/stack_trace_snippet.ts` |
| `frontend/app/components/stack_trace_snippet/stack_frame_snippet.ts` |

### 20.6 Adjacent (referenced)

| Path |
|------|
| `plugin/xprof/protobuf/trace_data_response.proto` |
| `xprof/convert/xplane_to_tools_data_with_profile_processor.cc` |

---

## 21. Relationship to other design notes

| Note | Overlap with Trace Viewer |
|------|---------------------------|
| `00-synthesis` | Portfolio ranking; TV likely top browser + multi-host convert |
| `02-convert-serve-cache` | Shared `respond()` gzip, result cache, warm-up `DEFAULT_CACHE_TOOLS` |
| `04-overview-and-stats` | Shares HostSelector / ALL_HOSTS patterns but not multi-host CSV or LevelDB |
| `05-graph-hlo` | Adjacent-nodes fetch from TV selection; HLO text not TV core path |
| `06-megascale` | `ProcessMegascaleDcn` optional on TV preprocess; Perfetto upload path separate |

Trace Viewer should be optimized **before** micro-optimizing stats DataTables when the goal is “large multi-host interactive sessions don’t OOM.”

---

## 22. Implementation sequencing (design only)

| Phase | Items | Exit criteria |
|-------|-------|---------------|
| P0 | T2 (gzip policy), T17 (host required), T13+T36 ratio/resolution floor | No double-compress PB; no accidental all-hosts; lower default overfetch |
| P1 | T1+T29 (PB end-to-end), T14 | V2 production on PB with flag default flip plan |
| P2 | T3+T16 (args lazy), T10 abort | Timeline RSS drop on large JSON/PB |
| P3 | T4+T8 multi-host bounds | 10-host Reduce stable under memory limit |
| P4 | T9 Catapult, T7 non-stream retirement | Legacy path bounded |
| P5 | T6 stream JSON, T26 arch | Only if PB insufficient for remaining JSON clients |

---

## 23. Open questions (code-derived, not blocked)

1. Does any deployment rewrite `/data` to a `.pb` filename so V2 PB path already works? (**hypothesized** no in OSS.)  
2. Does warm-up for `trace_viewer@` write SSTABLEs only, or also materialize full JSON into a result cache? (see note 02.)  
3. Exact RSS of `entry_args` vs SoA timing columns on a 1M-event viewport — needs measurement (T3 impact size).  
4. Is `load_metadata` ever true on viewport loads via any call site override? Default is false (`trace_events.h:941`); confirm all production call sites.  
5. Cloud AI non-stream path usage share — prioritizes T7 vs T1.  
6. Worker Map concurrency topology — how many cold preprocesses co-reside?  
7. Does catalog preference for `@` leave any UI still requesting plain `trace_viewer`?  

---

## 24. Summary

Trace Viewer memory is a **three-process problem** (C++ convert, Python HTTP gzip, browser JS/WASM) with a **multi-host multiplier** and a **false “streaming” name** (viewport re-query of LevelDB, not chunked HTTP).

Highest-confidence, highest-severity gaps:

1. Production V2 still on **JSON** despite PB/zstd pipeline existing (`use_pb` unwired; pathname check at `main.ts:802`).  
2. **Double compression** of zstd PB via default gzip (`respond.py:109-111` + `tool_data.py:134`).  
3. **Per-event args maps** in the WASM timeline (`timeline.h:111`) despite selection detail API (`trace_viewer.ts:813`).  
4. **Parallel Reduce** holding N host containers (`streaming_trace_viewer_processor.cc:299-311`).  
5. **Overfetch** (3× time, 8× resolution) and **accidental all-hosts** (`hosts.py:95-98`).  
6. **Visibility off below 500k events** (`trace_view_options.cc:114` + `trace_events.h:245-247`).  
7. **Catapult unbounded Set** on pan/zoom merge (`tf-trace-viewer.html:1969-2005`).

Closing T1–T4 plus T13/T17/T36 is the minimum set that changes the asymptotic behavior of large multi-host interactive sessions; ST-* items keep the stack-trace bridge from undoing wins via secondary retention.

---

## 25. Appendix A — Non-stream JSON chunk shape

Each event from `TraceEventsJsonStream._event` (`trace_events_json.py:78-95`):

```text
{pid, tid, name, ts_ms, ph: X|i, [dur], [s:t], [args: {...}]}
```

Plus metadata process/thread name and sort_index events (`:49-74`).  
Document wrapper: `displayTimeUnit:ns`, `highres-ticks`, `traceEvents:[...]` with trailing fake `{}` (`:99-105`).

---

## 26. Appendix B — ProcessSession host loop pseudocode with line refs

```text
ProcessSession (streaming_trace_viewer_processor.cc:58-158)
  merged = empty
  opts = GetTraceViewOption(options)           # :67-68
  profiler_opts = TraceOptionsFromToolOptions  # :69-70
  for i in [0, XSpaceSize):                    # :73
    paths = MakeHostDataFilePath ×3            # :75-86
    if paths missing: Unimplemented Cloud AI   # :88-91
    if !exists(events_sstable):                # :94
      arena; xspace = GetXSpace(i, arena)      # :98-100
      Preprocess(step_grouping, derived)       # :101-102
      if enable_legacy_dcn: ProcessMegascaleDcn# :103-105
      ConvertXSpaceToTraceEventsContainer      # :107-109
      StoreAsLevelDbTables ×3 files            # :120-124
    LoadTraceEventsContainer → host            # :139-140
    merged.Merge(move(host), i+1)              # :145
  SerializeAndSetOutput(merged, ...)           # :151-152
```

---

## 27. Appendix C — Reduce pseudocode with line refs

```text
Reduce (streaming_trace_viewer_processor.cc:281-339)
  require map_output_files non-empty           # :288-289
  opts = GetTraceViewOption(options_)          # :292-295
  n = map_output_files.size()
  threads = min(n, MaxParallelism())           # :297-298
  containers = vector<StatusOr<Container>>(n)  # :299
  pool: for i: containers[i]=LoadHost(...)     # :303-309
  JoinAll                                      # :311
  merged = empty
  for i: if ok: merged.Merge(move(c[i]), i+1)  # :316-327
  if no success: InternalError                 # :330-331
  SerializeAndSetOutput(merged, ...)           # :334-335
```

---

## 28. Appendix D — FE resolution worked examples

| Canvas CSS width | viewerWidth = W−250 | bins = round(w/2) | resolution = bins×8 |
|------------------|---------------------|-------------------|---------------------|
| 250 or less | ≤0 | n/a | **0** (full density) |
| 500 | 250 | 125 | 1000 |
| 1280 | 1030 | 515 | 4120 |
| 1920 | 1670 | 835 | 6680 |
| 2560 | 2310 | 1155 | 9240 |
| with selected_group_ids | — | — | **0** always |

Plugin default **8000** when FE omits param (`trace.py:20`) is near the 1920px case.

---

## 29. Appendix E — Confidence / severity legend

| Label | Meaning |
|-------|---------|
| observed | Code path exists; behavior follows from source |
| hypothesized | Plausible magnitude/impact; not measured in this note |
| H | Likely OOM or multi-GB class impact on large sessions |
| M | Significant RSS or long-session growth; not always fatal |
| L | Bounded or secondary; still worth hygiene |

---

*End of design note. Analysis only — no product code changes.*
