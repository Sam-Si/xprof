# Trace Viewer Path — Memory Use & Optimization Design Note

**Status:** analysis-only (no product code changes)  
**Date:** 2026-07-13  
**Scope:** XPlane on disk → plugin HTTP `/data` → convert (C++ streaming LevelDB + JSON/PB, Python non-stream JSON) → browser (Catapult iframe + Trace Viewer V2 WASM/WebGPU + Angular shell + stack_trace bridge)  
**Depth target:** deepest single-tool note in this tree (richer than 04/05/06)  
**Related notes:** `00-synthesis-memory-optimization.md` (portfolio), `02-convert-serve-cache-path.md` (shared convert/gzip), stack_trace adjacency below

This note re-derives the Trace Viewer memory story from **live repository sources** (not invented paths). Confidence labels are only **`observed`** (code exists and implies the behavior) or **`hypothesized`** (plausible peak/UX impact not yet measured). Severity is **H / M / L** for user-visible or peak-RAM impact, independent of confidence.

---

## 0. Why Trace Viewer dominates the memory portfolio

Trace Viewer is unique among XProf tools:

| Dimension | Trace Viewer | Typical stats tool (overview / kernel_stats / …) |
|-----------|--------------|--------------------------------------------------|
| Host selection | Explicit multi-host CSV (`hosts=`) — **only** tools in `_MULTI_HOST_SELECTION_TOOLS` | ALL_HOSTS aggregate or single host |
| On-disk intermediate | 3 LevelDB SSTABLEs per host after first open | OpStats / tool-specific caches |
| Viewport | `start_time_ms` / `end_time_ms` / `resolution` re-fetch | One-shot JSON table |
| Wire formats | Catapult JSON **or** zstd columnar `TraceDataResponse` (`format=pb`) | DataTable JSON |
| Browser model | Full timeline SoA + optional Catapult `tr.Model` + WASM heap | Angular DataTable graphs |
| First-open cost | Full XSpace → TraceEventsContainer → LevelDB write | Combined OpStats cache shared across tools |
| Warm-up | `DEFAULT_CACHE_TOOLS` includes `trace_viewer@` | `overview_page` only among stats tools |

**Implication:** optimizing Trace Viewer is not “another JSON table slim-down.” It is a multi-layer pipeline where peak RAM can appear in the **plugin process (C++)**, the **Python gzip layer**, and the **browser (JS + WASM)** simultaneously—and multi-host multiplies several of those peaks.

---

## 1. Full end-to-end data path

### 1.1 ASCII E2E diagram (production happy path: `trace_viewer@` + V2 JSON)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ DISK (session run_dir)                                                      │
│   <host>.xplane.pb                                                          │
│   after cold preprocess per host:                                           │
│     <host>.TRACE_LEVELDB / .SSTABLE naming via StoredDataType               │
│     <host> TRACE_EVENTS_METADATA_LEVELDB                                    │
│     <host> TRACE_EVENTS_PREFIX_TRIE_LEVELDB                                 │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ BROWSER — Angular shell                                                     │
│   frontend/.../trace_viewer/trace_viewer.ts                                 │
│   - hosts capped at 10 (sort + slice)                                       │
│   - builds query: run, tag=trace_viewer@, hosts|host, filters, full_dma … │
│   - useTraceViewerV2 from query/localStorage                                │
│   - dataService.getDataUrl → /data?...                                      │
│   ├─ V2: WASM loadTraceData(url)  [main.ts]                                 │
│   └─ Legacy: iframe → /trace_viewer_index.html?trace_data_url=…             │
│       plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html (Catapult)   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTP GET /data
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ PLUGIN HTTP                                                                 │
│   api/data_api.py  DataApiMixin.data_route / data_impl                      │
│     → parse ToolRequest                                                     │
│     → services/tool_data.py ToolDataService.get_tool_data                   │
│         resolve run_dir, build_tool_params                                  │
│         tools/options/trace.py → trace_viewer_options                       │
│           resolution default 8000, format, start/end_time_ms, event_name…  │
│         services/hosts.py HostSelector.select                               │
│           hosts_param CSV if supports_multi_host_selection                  │
│           host==ALL_HOSTS → all XPlanes                                     │
│           missing host/hosts → ALL hosts (legacy fallback)                  │
│         convert.xspace_to_tool_data(asset_paths, tool, params)              │
│   convert/raw_to_tool_data.py                                               │
│     tool == 'trace_viewer@':                                                │
│       pywrap → C++ StreamingTraceViewerProcessor                            │
│       returns raw_data as-is (JSON string or pb bytes)                      │
│       content_type application/json | application/octet-stream              │
│     tool == 'trace_viewer':                                                 │
│       C++ Trace proto bytes → process_raw_trace → Python JSON join          │
│   http/respond.py respond()                                                 │
│     DEFAULT: gzip.compress(body) + Content-Encoding: gzip                   │
│     content_encoding arg never set by ToolDataService today                 │
│     → **already-zstd pb is gzipped again**                                  │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ C++ convert — StreamingTraceViewerProcessor                                 │
│   REGISTER_PROFILE_PROCESSOR("trace_viewer@", …)                            │
│                                                                             │
│   ProcessSession (plugin local multi-host path):                            │
│     for each host SERIAL (TODO b/452217676 parallel):                       │
│       if SSTABLE missing:                                                   │
│         Arena GetXSpace → PreprocessSingleHostXSpace                        │
│         optional ProcessMegascaleDcn                                        │
│         ConvertXSpaceToTraceEventsContainer                                 │
│         StoreAsLevelDbTables (events + metadata + prefix trie)              │
│       LoadTraceEventsContainer(file_paths, viewport, resolution)            │
│       merged.Merge(host_container, host_id)                                 │
│     SerializeAndSetOutput(merged)                                           │
│                                                                             │
│   Map/Reduce (worker service when XSpaceSize()>1):                          │
│     Map: per-host preprocess → SSTABLE path                                 │
│     Reduce: parallel LoadTraceContainerForHost (MaxParallelism threads)     │
│             then serial Merge into one container                            │
│                                                                             │
│   SerializeAndSetOutput:                                                    │
│     format=="pb": ConvertTraceDataToCompressedDeltaSeriesProto (zstd)       │
│     else: TraceEventsToJson → full std::string via IOBufferAdapter          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ gzip wire
┌───────────────────────────────────▼─────────────────────────────────────────┐
│ BROWSER materialization                                                     │
│   V2 JSON path:                                                             │
│     fetch → response.json() → JS object graph                               │
│     processTraceEvents → packer → FlameChartTimelineData SoA                │
│     entry_args: vector<flat_hash_map> per event (TODO b/474668991)          │
│   V2 PB path (only if URL pathname ends with .pb — production /data does NOT):│
│     arrayBuffer → _malloc WASM copy → zstd decompress → TraceDataResponse   │
│     free decompressed early; free malloc buffer                             │
│   Legacy Catapult:                                                          │
│     JSON → tr.Model; _loadedTraceEvents Set grows on pan/zoom merge         │
│   Event detail: second /data with event_name → force format=json            │
│   stack_trace_page: query-param bridge to Angular StackTraceSnippet         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Request parameter surface (what actually drives memory)

Built by `build_trace_viewer_options` (`tools/options/trace.py`) and V2 URL mutators (`main.ts`):

| Param | Default / source | Memory effect |
|-------|------------------|---------------|
| `resolution` | Plugin default **8000**; V2 overwrites via `updateUrlWithResolution` = `round((canvasWidth-250)/2) * ZOOM_RATIO(8)` | Higher → more events through visibility filter; **0** disables density downsampling |
| `start_time_ms` / `end_time_ms` | V2 expands viewport by `FETCH_RATIO=3` around visible range | Overfetch ×3 time span on each load |
| `format` | `"json"` in C++ `GetTraceViewOption` unless set; `event_name` **forces json** | PB path skips catapult JSON + Python join |
| `event_name` + `duration_ms` + `unique_id` | Selection detail fetch | Full metadata read for one event; always JSON |
| `search_prefix` / `search_metadata` | Prefix trie path | Different load path; may still materialize matching set |
| `full_dma` / filters | Tool options / FILTER_CONFIG | Can force denser / unfiltered data |
| `hosts` | FE cap 10 | Linear multi-host merge cost |
| `host` | Single host or `ALL_HOSTS` | ALL_HOSTS = every XPlane |

### 1.3 Tool naming / catalog override

From `tools/registry.py` and `tools/catalog.py`:

- `trace_viewer` = non-streaming (pre-TF 2.13 style; still registered).
- `trace_viewer@` = streaming / LevelDB-backed (TF 2.14+).
- If both available in a session, **catalog discards plain `trace_viewer`** so UI always prefers `@`.
- Cloud Vertex path may advertise plain `trace_viewer` only (`xplane_to_tool_names.cc` branch on cloud flag)—streaming SSTABLE path then may be unimplemented (`UnimplementedError` “Cloud AI”).

---

## 2. Stage-by-stage materialization

Peak memory is the **sum of concurrent live representations**, not the size of the final wire payload alone.

| # | Stage | Representation | Peak notes | Severity | Confidence |
|---|-------|----------------|------------|----------|------------|
| S0 | On-disk XPlane | `*.xplane.pb` | Hundreds of MB–multi-GB multi-host sessions | H (disk) | observed |
| S1 | `SessionSnapshot.GetXSpace(i, &arena)` | Arena-backed `XSpace` | Full host decode; arena freed after preprocess scope ends in ProcessSession | H | observed (`streaming_trace_viewer_processor.cc`) |
| S2 | `PreprocessSingleHostXSpace` | Mutates XSpace in place (+ step grouping, derived timeline) | Extra derived lines/events on same arena object | M | observed |
| S3 | Optional `ProcessMegascaleDcn` | XSpace mutation for legacy DCN | Extra events when `enable_legacy_dcn` | M | observed |
| S4 | `ConvertXSpaceToTraceEventsContainer` | Arena `TraceEventsContainer` + name table + raw args | Peak ≈ XSpace + full event set **before** LevelDB write; largest single-host convert spike | **H** | observed (`xplane_to_trace_container.cc`) |
| S5 | `StoreAsLevelDbTables` | Parallel chunk serialize (`kChunkSize=10000`, `kMaxBufferedChunks=20`) + LevelDB `block_size=20 MiB` | Up to 20 chunks buffered × serialized KV pairs; 20 MiB table blocks | **H** | observed (`trace_events.cc`, `lite_trace_events.cc`) |
| S6 | Cold SSTABLE on disk | 3 tables/host: events, metadata, prefix trie | Disk spill amortizes later loads; first open pays full S1–S5 | M (disk) / H (first RAM) | observed |
| S7 | `LoadTraceEventsContainer` | Filtered `TraceEventsContainer` in RAM | Visibility+resolution reduce events when `num_events ≥ 500_000`; below threshold visibility **disabled** (load denser) | **H** | observed (`trace_view_options.cc`, `trace_events.h` DoLoad) |
| S8 | Multi-host merge | `merged.Merge(std::move(host), host_id)` | ProcessSession: host container freed after merge; Reduce: **all host containers live until JoinAll** then merge | **H** | observed |
| S9 | JSON serialize | `std::string trace_viewer_json` via `IOBufferAdapter` | Full catapult-style event array as one contiguous string; coexists with merged container until SetOutput | **H** | observed (`SerializeAndSetOutput`) |
| S10 | format=pb serialize | `TraceDataResponse` proto → serialize → zstd compress | Intermediate uncompressed proto + compressed bytes; columnar deltas + interned strings | M–H | observed (delta_series header) |
| S11 | Python non-stream (`trace_viewer`) | `Trace` proto parse + `''.join(TraceEventsJsonStream)` | Double materialization: C++ Trace string → Python proto → chunked JSON → joined str | **H** | observed (`raw_to_tool_data.py`, `trace_events_json.py`) |
| S12 | `respond()` gzip | Raw body + `gzip.compress(body)` | Both live until Response constructed; **zstd pb gzipped again** | **H** | observed (`respond.py`; `content_encoding=None` from tool_data) |
| S13 | HTTP transfer | gzip bytes | Wire size; browser inflate buffers | M | observed |
| S14 | V2 JSON parse | Full JS object graph (`traceEvents[]`) | Dominant browser heap for production path | **H** | observed (`loadJsonDataInternal` → `JSON`/fetch json) |
| S15 | V2 WASM process | `processTraceEvents` packs into C++ SoA | JS graph may still be live during pack; WASM heap grows (`ALLOW_MEMORY_GROWTH`); `window.wasmMemoryBytes` records HEAPU8 length | **H** | observed (`main.ts`) |
| S16 | V2 PB parse | ArrayBuffer + WASM malloc copy + zstd out + proto | Better density; decompressed string explicitly `swap`’d free; **pathname `.pb` gate unused for `/data`** | H if enabled; gap today | observed (`trace_pb_event_parser.cc`, `main.ts`) |
| S17 | `FlameChartTimelineData` | SoA vectors + **`entry_args` maps** | TODO to fetch args from backend; maps dominate per-event heap | **H** | observed (`timeline.h` b/474668991) |
| S18 | Timeline fetch window | `kFetchRatio=3`, `kPreserveRatio=2`, `kRefetchZoomRatio=8` | Holds last fetch range; refetch on deep zoom keeps multiple generations briefly | M | observed (`constants.h`, `timeline.cc`) |
| S19 | Legacy Catapult model | `tr.Model` + `_loadedTraceEvents` Set of hash keys | Unbounded growth on pan/zoom **merge** path | **H** | observed (`tf-trace-viewer.html`) |
| S20 | Event args cache (Angular) | `eventArgsCache: Map<string, Record>` | Unbounded if user selects many events | M | observed (`trace_viewer.ts`) |
| S21 | stack_trace_page | Query strings: stack_trace, hlo_module, hlo_op, source | URL + component field retention; secondary to parent tool | L–M | observed |

### 2.1 Severity heatmap by process

```text
Plugin C++ peak (cold multi-host Reduce):
  [XSpace arena × concurrent Maps] + [TraceEventsContainer × N threads]
  + [merged container] + [JSON string OR TraceDataResponse+zstd]
  + [gzip buffer in Python]

Browser peak (V2 JSON):
  [gzip inflate] + [JSON object graph] + [WASM SoA + entry_args]
  + [optional previous timeline until replace]

Browser peak (legacy Catapult streaming):
  [full model accumulation across pan/zoom]  ← often worse long session
```

### 2.2 Visibility / streaming threshold semantics (critical)

`LoadTraceEventsContainer` constructs `TraceVisibilityFilter(MilliSpan(start,end), resolution, …)` and passes `kDisableStreamingThreshold = 500000` into `LoadFromLevelDbTable`.

In `DoLoadFromLevelDbTable`:

```text
filter_by_visibility =
    (threshold == -1) OR (!trace.has_num_events()) OR (num_events >= threshold);

if (!filter_by_visibility) {
  visibility_filter->UpdateVisibility(0);  // disable density filtering
}
```

**Observed consequences:**

1. Traces with **fewer than 500k events** get **no visibility downsampling** → load denser than large traces for the same viewport/resolution. Counter-intuitive: “small” traces can still inflate browser RAM if they pack heavy args.
2. `resolution=0` (V2 when `selected_group_ids` filter present, or default before canvas width known) means “no duration-based downsampling” even when visibility is enabled.
3. “Streaming” in product language means **re-query LevelDB with a new timespan/resolution**, not HTTP chunked transfer of convert progress.

---

## 3. Streaming (`trace_viewer@`) vs non-streaming (`trace_viewer`)

| Aspect | `trace_viewer` (non-stream) | `trace_viewer@` (stream / LevelDB) |
|--------|----------------------------|-------------------------------------|
| Registration | XPLANE_TOOLS | XPLANE_TOOLS; catalog prefers this |
| Multi-host C++ | `ConvertXSpaceToTraceEvents` requires **XSpaceSize()==1** | ProcessSession loop or Map/Reduce multi-host |
| First request | XSpace → Trace events **string** (C++) → Python `process_raw_trace` | XSpace → container → **3 SSTABLEs** → filtered load → JSON/PB |
| Later pan/zoom | Full reconvert unless external result cache | SSTABLE hit; load only filtered slice |
| Viewport params | Limited / ignored in legacy path | `start/end_time_ms`, `resolution`, search, event_name |
| Output path | Always through Python JSON join for non-pb | C++ JSON string **or** zstd PB; Python does **not** re-parse `@` |
| Disk intermediates | None (unless tool result cache) | 3 LevelDB tables per host |
| Worker service | N/A single-host assumption | `ShouldUseWorkerService` when `XSpaceSize() > 1` |
| Cloud AI | May be the only option | SSTABLE paths may be Unimplemented |
| Peak first-open | XSpace + Trace string + Python Trace proto + joined JSON + gzip | XSpace + container + LevelDB write buffers + load + JSON/PB + gzip |
| Peak subsequent | Same as first if no cache | LevelDB read + filtered container + serialize + gzip |

### 3.1 Non-stream double materialization (detail)

`raw_to_tool_data.py` for `tool == 'trace_viewer'`:

1. C++ returns serialized Trace / events string (`ConvertXSpaceToTraceEventsString`).
2. Unless `format=='pb'`, Python runs `process_raw_trace`:
   - `trace_events_old_pb2.Trace().ParseFromString(raw_trace)`
   - `''.join(TraceEventsJsonStream(trace))` yielding `json.dumps` per event with trailing commas and a fake `{}` terminator.

`TraceEventsJsonStream` is **conceptually** streaming (iterator of chunks) but **call site joins everything** into one `str` before `respond()` gzip. Opportunity: stream chunks into gzip writer without a full join (T6/T7 family).

### 3.2 Stream path does not use Python JSON converter

For `trace_viewer@`, success path sets `data = raw_data` directly (JSON already produced in C++, or binary PB). That avoids the Python proto re-parse—but still pays C++ full `std::string` JSON and Python gzip.

### 3.3 What “streaming” is **not**

| Not this | Reality |
|----------|---------|
| HTTP chunked convert progress | Single response after full serialize |
| Incremental LevelDB write visible to client | Client only sees final JSON/PB |
| Automatic PB for V2 | `use_pb` flag exists; data URLs not switched to format=pb or `.pb` |
| Per-tile LRU on server | Each request reloads filtered events from LevelDB into a new container |

---

## 4. Multi-host / ALL_HOSTS impact

### 4.1 Policy matrix

| Mechanism | Code | Behavior |
|-----------|------|----------|
| Multi-host CSV | `supports_multi_host_selection` → only `trace_viewer` / `trace_viewer@` | `hosts=h1,h2,…` → exact list; missing host → FileNotFoundError |
| ALL_HOSTS | `host == ALL_HOSTS` | All XPlanes in run_dir |
| Empty host | `not host and not hosts_param` | **Also all hosts** (legacy) |
| FE cap | `hostsList.sort().slice(0, 10)` | Silent truncation to 10 hosts |
| Host list UI | `filenames.py` / hosts API | Trace tools not in ALL_HOSTS_SUPPORTED set the same way as overview—FE still can pass many hosts |
| Worker path | `ShouldUseWorkerService` if `XSpaceSize() > 1` | Map per host; Reduce parallel load |

**Dangerous combo (observed):** FE fails to pass `hosts`/`host` correctly → HostSelector falls into “use all hosts” → convert attempts entire session even when user thought single host.

### 4.2 ProcessSession vs Reduce peak shapes

**ProcessSession** (serial host loop):

```text
peak ≈ max_over_hosts(preprocess_peak(host))
       + size(merged_so_far)
       + serialize(merged_final)
```

Comment TODO `b/452217676`: parallelize hosts—improves latency but risks raising peak toward Reduce shape if naive.

**Reduce** (parallel load):

```text
peak ≈ sum_over_concurrent_hosts(filtered_container(host))
       + merged
       + serialize
```

Thread count = `min(num_hosts, MaxParallelism())`. For 10 hosts with large filtered windows, **sum of concurrent containers** is the dominant hypothesized OOM mode on the plugin/worker.

### 4.3 Merge semantics

`merged_trace_container.Merge(std::move(trace_container), i + 1)` remaps device/process IDs by host index. Merged container holds **union** of filtered events from all hosts for the viewport—multi-host is not “metadata only.”

### 4.4 Multi-host × overfetch interaction

V2 expands time range by `FETCH_RATIO=3` and resolution by `ZOOM_RATIO=8` **per request**, then multiplies by N hosts. Approximate event budget:

```text
events ≈ f(viewport × FETCH_RATIO, resolution_bins × ZOOM_RATIO) × N_hosts
```

Severity **H** when N=10 and resolution falls back to 0 (full density).

---

## 5. Frontend retention (V2 + Catapult + container + stack_trace)

### 5.1 Angular shell — `trace_viewer.ts`

Holds across session:

| Object | Lifetime | Notes |
|--------|----------|-------|
| `hostList` | Navigation | Host names only — small |
| `featureFlags` + localStorage | Session | Includes `use_pb` (default false) — **not wired into data URL** |
| `eventArgsCache` | Component life | Grows with distinct selected events |
| `hloAdjacentNodesCache` | Component life | Adjacent-nodes JSON per op |
| `traceViewerModule` | Until destroy | Full WASM module; `shutdownTraceViewerV2()` on destroy |
| `processes` maps | After load | Process name maps from WASM |
| Query string / navigationEvent | Current run | Rebuilds data URL |

**Observed gap:** `use_pb` appears in feature flag dialog and localStorage (`xprof_ff_use_pb`) but `update()` builds `getDataUrl` without `format=pb`. V2 loader only takes PB branch when `urlObj.pathname.endsWith('.pb')`, while plugin serves `/data?...` — production V2 is **JSON-only** unless external URL rewriting exists outside this tree.

### 5.2 Trace Viewer V2 — `main.ts` + timeline + parsers

**Load pipeline (JSON):**

1. `loadTraceData(url)` dedupes identical URL via `currentLoadingPromise`.
2. Abort: if URL changes mid-flight, `isAbortRequested` skips process (good) but **fetch may still complete and hold ArrayBuffer/JSON until GC**.
3. `updateUrlWithResolution`: bins = `(clientWidth - HEADING_WIDTH) / MIN_EVENT_WIDTH` × `ZOOM_RATIO(8)`.
4. Optional `expandUrlTimeRange` with `FETCH_RATIO(3)`.
5. `fetch` → `response.json()` → validate `traceEvents` → `processTraceEvents`.

**Load pipeline (PB, if pathname `.pb`):**

1. `arrayBuffer` of body (after browser gunzip of HTTP layer).
2. `_malloc` + `HEAPU8.set` — **duplicate** of buffer in WASM heap.
3. `processCompressedTraceEvents` → zstd decompress → parse `TraceDataResponse` → free decompressed string early → SoA.
4. `_free(dataPtr)` in `finally`.

**Resident timeline model (`FlameChartTimelineData`):**

- Structure-of-arrays for levels, times, names, pids, tids.
- **`entry_args`**: `vector<flat_hash_map<string,string>>` — explicit TODO to drop in favor of backend fetch (selection path already has `maybeFetchEventArgs` with `event_name`).
- Flow line maps and counter series by group index.
- Fetch/preserve ratios keep a super-viewport of data; deep zoom triggers higher-resolution refetch without necessarily freeing previous SoA until replace completes.

**Packer (`trace_event_packer`):** assigns visual levels; temporary vectors during pack add short-lived peak on top of event list.

### 5.3 Trace Viewer container — `trace_viewer_container.ts`

- Hosts WASM canvas vs Catapult iframe switch (`useTraceViewerV2`).
- Loading status events; search debounce 300ms; selected event property tables.
- Does not own the large model itself but keeps references to `traceViewerModule` and event detail tables (small vs timeline).
- Legacy path listens to mouseup/keydown on window for iframe interaction.

### 5.4 Legacy Catapult — `tf-trace-viewer.html`

Critical retention pattern `_filterKnownTraceEvents` / `_updateModel`:

- First load: new `tr.Model()`, empty `_loadedTraceEvents` Set.
- Subsequent pan/zoom merges: hash every event; **keep Set of all ever-seen keys**; merge new events into model; update args for duplicates via `_maybeUpdateEventArguments`.
- Set keys are strings over pid/tid/ts/name — **unbounded** as user explores full trace duration.

Long interactive sessions on large multi-host traces can exceed single-viewport RAM by multiples of the “one screen” budget.

### 5.5 Event detail secondary fetches

Selection triggers `maybeFetchEventArgs`:

- Query: `event_name`, `start_time_ms`, `duration_ms`, `unique_id`.
- Options builder **forces `format=json`** when `event_name` set.
- Backend `ReadFullEventFromLevelDbTable` pulls metadata SSTABLE for that event.
- Args cached in Angular Map; also feed HLO adjacent-nodes fetch (`graph_viewer` type path) — second tool convert possibly large.

This is the intended “lazy args” pattern; timeline still stores args eagerly in SoA (contradiction / incomplete migration).

### 5.6 stack_trace_page / stack_trace_snippet (bridge)

**Role:** Trace Viewer (especially non-Angular Catapult) cannot embed Angular `StackTraceSnippet` in-process; `stack_trace_page` is a route bridge. Also used from op_profile / hlo_stats / roofline deep-links.

Query params retained as component fields: `stack_trace`, `hlo_module`, `hlo_op`, `source`, `session_id`/`run`, `op_category`.

Source code service is checked for availability then not held as persistent file buffer in the page itself—but snippet components may load file slices via the service token.

Memory impact is **secondary** to parent Trace Viewer payload, but long `stack_trace` query strings inflate URL history and Angular fields; opening snippet in a **new tab** may leave the full Catapult/WASM model alive in the background tab (browser process), which is a UX/architecture retention issue.

See §11 for ST-* opportunities.

---

## 6. Ranked opportunities table

Confidence column is **only** `observed` or `hypothesized`.

| ID | Stage | Description | Severity | Confidence | Effort | Notes |
|----|-------|-------------|----------|------------|--------|-------|
| T1 | FE+BE | Wire `format=pb` end-to-end for V2 when `use_pb` true: set query param; treat `/data` octet-stream as compressed PB (not only `.pb` pathname) | **H** | observed | M | Flag + C++ path exist; FE gate broken |
| T2 | HTTP | Skip default gzip when body is already zstd PB; set Content-Encoding appropriately or use identity + zstd framing | **H** | observed | S | `respond.py` always gzips if content_encoding None |
| T3 | FE V2 | Drop or lazy-load `entry_args` maps; rely on `event_name` detail fetch | **H** | observed | M | TODO b/474668991; detail fetch already exists |
| T4 | BE multi-host | Bound Reduce concurrency (semaphore) / free host containers ASAP; backpressure on Map | **H** | observed | M–L | Parallel load holds N containers |
| T5 | BE visibility | Revisit `kDisableStreamingThreshold=500000` and resolution=0 full-load cases | **M** | observed | S | Small traces load denser; filters force res=0 |
| T6 | BE JSON | Stream JSON into compressor without one giant `std::string` | **M** | hypothesized | L | IOBufferAdapter appends to string today |
| T7 | Python non-stream | Eliminate `process_raw_trace` join; stream chunks or move JSON fully to C++ | **M** | observed | M | Only plain `trace_viewer` |
| T8 | BE merge | ProcessSession: free host container after merge (already move); Reduce: merge incrementally as threads complete | **M** | observed | M | Reduces concurrent peak |
| T9 | Legacy FE | Disable Catapult model accumulation when V2 is default; or hard-cap `_loadedTraceEvents` / replace model each viewport | **M** | observed | S–M | Set growth in tf-trace-viewer.html |
| T10 | FE fetch | Abort fetch via AbortController; drop mid-flight JSON; free previous SoA before installing new | **M** | observed partial | M | URL change aborts process step only |
| T11 | LevelDB | Tune `block_size` 20 MiB and `kMaxBufferedChunks` for write peak | **M** | observed | S | Trade I/O vs RAM |
| T12 | Multi-host FE | Lower host cap below 10 or progressive host load (1 then add) | **M** | observed | S | slice(0,10) already; still high |
| T13 | Overfetch | Reduce `ZOOM_RATIO` / `FETCH_RATIO` / `kRefetchZoomRatio` defaults | **M** | observed | S | 8× and 3× overfetch |
| T14 | PB + HTTP | Single compression: zstd only; document Content-Type application/octet-stream + encoding | **H** | observed | S | Complements T1/T2 |
| T15 | Cold preprocess | Parallel preprocess with memory budget; avoid holding XSpace + full container + chunk buffers naively | **H** | hypothesized | L | First-open dominated by S4–S5 |
| T16 | Metadata SSTABLE | Never attach full args in viewport load; only metadata table on selection | **H** | hypothesized | M | Aligns with T3; check load_metadata flags |
| T17 | HostSelector | Cap hostless all-hosts for trace tools; require explicit host | **M** | observed | S | Dangerous fallback |
| T18 | Worker service | Prefer ProcessSession serial shape when memory constrained vs Reduce parallel | **M** | hypothesized | M | Config/flag |
| T19 | Event args cache | LRU-bound `eventArgsCache` / adjacent nodes cache | **L** | observed | S | Unbounded Maps |
| T20 | Search path | Bound search result set size server-side | **M** | hypothesized | M | Prefix trie can explode |
| T21 | full_dma / filters | Document and cap density when full_dma or FILTER_CONFIG disables downsampling | **M** | hypothesized | S | UX vs RAM |
| T22 | WASM heap | Grow carefully; reset/recreate module on huge runs; report wasmMemoryBytes in UI | **M** | observed | M | HEAPU8 tracks reservation |
| T23 | Catalog | Keep forcing `@` over plain; retire non-stream path in OSS when safe | **L** | observed | L | Removes T7 path |
| T24 | Double gzip observability | Metrics for payload sizes pre/post gzip and zstd | **L** | hypothesized | S | Guides T2 |
| T25 | Timeline preserve | Lower `kPreserveRatio` aggressively after pan settles | **M** | hypothesized | S | constants.h |
| T26 | Multi-host JSON | Per-host JSON stream merge without full merged container | **H** | hypothesized | L | Architectural |
| T27 | Cloud AI | If forced non-stream, apply same PB/lazy-args wins | **M** | observed gap | M | Unimplemented SSTABLE |
| T28 | Warm-up | `DEFAULT_CACHE_TOOLS` includes `trace_viewer@` — ensure warm-up does not hold full JSON in process after write | **M** | hypothesized | M | Shared with 02 |
| T29 | Content-type | Ensure FE uses arrayBuffer path for octet-stream regardless of URL | **H** | observed | S | Part of T1 |
| T30 | ProcessSession parallel TODO | If implementing b/452217676, add memory semaphore first | **H** | observed | M | Prevents latent OOM |

---

## 7. Top 10 wins (ranked)

Ordered by **expected peak-RAM reduction × breadth of users × implementability**.

| Rank | ID | Win | Why first |
|------|-----|-----|-----------|
| 1 | **T1+T29** | Ship compressed PB pipeline for V2 | Removes catapult JSON object graph + huge `std::string` JSON; code already 80% present |
| 2 | **T2+T14** | Single compression layer for PB | Stops gzip(zstd(data)); cuts plugin peak and CPU |
| 3 | **T3+T16** | Slim resident timeline (no per-event args maps) | Browser SoA stays dense; selection already has detail API |
| 4 | **T4+T8** | Bound multi-host concurrent containers | Prevents Reduce OOM on 10-host sessions |
| 5 | **T13+T5** | Fetch less density by default | ZOOM/FETCH ratios + threshold/res=0 audit |
| 6 | **T17** | Require explicit host; kill accidental all-hosts | Cheap correctness+memory fix |
| 7 | **T9** | Stop Catapult accumulation (or force V2) | Long-session legacy leak |
| 8 | **T10** | AbortController + free previous model | Pan/zoom thrash |
| 9 | **T7** | Kill non-stream Python double parse | Remaining `trace_viewer` / cloud path |
| 10 | **T11+T15** | LevelDB write buffer / cold preprocess budget | First-open plugin OOM |

**Secondary but valuable:** T12 progressive hosts, T19 cache LRUs, T22 WASM reset, T28 warm-up hygiene.

---

## 8. Risks matrix

| Opt | Correctness risk | UX risk | Rollout note |
|-----|------------------|---------|--------------|
| T1 PB | Proto schema skew between plugin and WASM; missing fields (flows, details) | Blank timeline if fallback to JSON broken | Feature flag already `use_pb`; can stage |
| T2 no double gzip | Clients that always gunzip blindly | None if Content-Encoding correct; browsers handle gzip layer | Need integration tests for both JSON and PB |
| T3 drop entry_args | Selection missing fields until second RTT | Slight delay on select | Keep event_name path fast; cache LRU |
| T4 concurrency bound | Latency ↑ on multi-host | Spinner longer; still correct | Make limit configurable |
| T5 threshold/res | Over-filter might hide short events | “Missing” slices at high zoom | Prefer refetch-on-zoom (already exists) |
| T6 stream JSON | Partial response on crash harder to debug | Progressive paint possible later | Large engineering |
| T7 Python path | Golden tests for catapult JSON order | None if C++ JSON matches | Catalog prefers @ already |
| T9 Catapult | Break extensions depending on full model | Legacy users only | Gate on useTraceViewerV2 |
| T13 lower overfetch | More network RTTs on zoom | Stutter on slow nets | Tune ZOOM_RATIO 8→4 first |
| T17 require host | Breaks bookmarks without host | Explicit host picker errors | Soft warn then hard fail |
| T26 no merge container | Cross-host ID remap bugs | Wrong lanes | Architecture-level risk |
| ST-* stack_trace | Broken deep links if truncate too aggressively | Snippet incomplete | Cap with ellipsis + full fetch by id |

---

## 9. Explicit real paths list

All paths below were verified present in this workspace at write time.

### 9.1 Plugin / Python convert

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
| `plugin/xprof/profile_plugin/constants.py` |
| `xprof/pywrap/profiler_plugin_impl.cc` |

### 9.2 C++ convert / streaming

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

### 9.3 Frontend Trace Viewer

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

### 9.4 Catapult / plugin static viewer

| Path |
|------|
| `plugin/trace_viewer/tf_trace_viewer/tf-trace-viewer.html` |
| `plugin/trace_viewer/trace_viewer.html` |

### 9.5 Stack trace bridge

| Path |
|------|
| `frontend/app/components/stack_trace_page/stack_trace_page.ts` |
| `frontend/app/components/stack_trace_page/stack_trace_page.ng.html` |
| `frontend/app/components/stack_trace_snippet/stack_trace_snippet.ts` |
| `frontend/app/components/stack_trace_snippet/stack_frame_snippet.ts` |

### 9.6 Adjacent (referenced for multi-format / megascale)

| Path |
|------|
| `frontend/app/components/megascale_perfetto/megascale_perfetto.ts` |
| `plugin/xprof/protobuf/trace_data_response.proto` |

---

## 10. Constants appendix

| Constant | Value | Where | Memory role |
|----------|-------|-------|-------------|
| Default `resolution` (plugin options) | **8000** | `tools/options/trace.py` | Density binning default if FE omits |
| Default `format` (C++) | **`json`** | `trace_view_options.cc` GetTraceViewOption | PB opt-in only |
| `kDisableStreamingThreshold` | **500_000** events | `trace_view_options.cc` | Below → visibility off |
| LevelDB `block_size` | **20 MiB** | `trace_events.cc`, `prefix_trie.cc`, load path | Read/write buffer granularity |
| Event chunk size | **10_000** | `trace_events.cc` `kChunkSize` | Parallel store granularity |
| `kMaxBufferedChunks` | **20** | `trace_events.cc` / `lite_trace_events.cc` | Write-side concurrent buffer cap |
| V2 `ZOOM_RATIO` | **8** | `main.ts` | Resolution overfetch |
| V2 `FETCH_RATIO` | **3.0** | `main.ts` | Time-span overfetch |
| `kFetchRatio` | **3.0f** | `timeline/constants.h` | C++ side fetch width |
| `kPreserveRatio` | **2.0f** | `timeline/constants.h` | Keep loaded window |
| `kRefetchZoomRatio` | **8.0f** | `timeline/constants.h` | When to refetch denser data |
| `MIN_EVENT_WIDTH` | **2** px | `main.ts` | Resolution bin size |
| `HEADING_WIDTH` | **250** px | `main.ts` / `kDefaultLabelWidth` | Subtracted from canvas width |
| `kMinFetchDurationMicros` | **1000** | `constants.h` | Floor on fetch duration |
| FE multi-host cap | **10** | `trace_viewer.ts` `slice(0, 10)` | Max hosts per request |
| `use_pb` default | **false** | `feature_flags.ts` | PB pipeline off by default |
| `use_trace_viewer_v2` | query/localStorage | `trace_viewer.ts` | V2 vs Catapult iframe |
| `DEFAULT_CACHE_TOOLS` | `overview_page`, `trace_viewer@` | `registry.py` | Warm-up includes streaming TV |
| Multi-host selection tools | `trace_viewer`, `trace_viewer@` | `registry.py` | Only tools with `hosts=` CSV |
| Worker multi-host gate | `XSpaceSize() > 1` | `streaming_trace_viewer_processor.h` | Enables Map/Reduce path |
| Search debounce | **300** ms | `trace_viewer_container.ts` | UI only |
| Event args cache fuzzy match | **50_000** µs | `trace_viewer.ts` | Start-time equality window |
| Tutorial rotation | **3_000** ms | container | Irrelevant to RAM |
| respond() default encoding | **gzip** | `http/respond.py` | Always compress unless overridden |

---

## 11. Adjacent UI: `stack_trace_page` / `stack_trace_snippet`

### 11.1 Role (observed)

Angular bridge so Trace Viewer (non-Angular Catapult path) and tools like op_profile / hlo_stats / roofline can show source/HLO snippets around a stack frame. Query params carry:

- `stack_trace`
- `hlo_module` (also parses program id from parentheses)
- `hlo_op`
- `source` (file + line)
- `session_id` / `run`
- `op_category`

This is **not** an `XPLANE_TOOLS` entry and does not run streaming convert. Memory cost is the **string payload in the URL/route** plus whatever `SourceCodeService` loads into the snippet, while the **parent** Trace Viewer (or stats tool) may remain mounted.

**Paths:**

- `frontend/app/components/stack_trace_page/stack_trace_page.ts`
- `frontend/app/components/stack_trace_page/stack_trace_page.ng.html`
- `frontend/app/components/stack_trace_snippet/stack_trace_snippet.ts`
- `frontend/app/components/stack_trace_snippet/stack_frame_snippet.ts`
- Referenced from `frontend/app/components/trace_viewer/constants.ts` (`stack_trace_page` route name)
- Embedded toggles exist in op_profile / hlo_stats / roofline components (parent tool tables stay resident)

### 11.2 Materialization

| Stage | What is held | Confidence |
|-------|--------------|------------|
| Route query | Full `stack_trace` + HLO module strings in component fields | observed |
| SourceCodeService | Temporary availability check; may load file slices into snippet | observed (service token; isAvailable subscription) |
| Parent tools | `showStackTrace` toggles keep parent tool tables resident while snippet open | observed (pattern in tool UIs) |
| Trace Viewer deep-link | New page/tab may leave Catapult/WASM session alive in background | hypothesized |

### 11.3 Opportunities (ST-*)

| ID | Stage | Opportunity | Severity | Confidence | Effort | Notes |
|----|-------|-------------|----------|------------|--------|-------|
| ST-1 | FE route | Cap / truncate very long `stack_trace` query strings before navigation (URL + history + component heap) | **L** | hypothesized | S | Keep hash/id; fetch full stack by session+op |
| ST-2 | FE bundle | Lazy-load `StackTraceSnippet` module only when panel/page open | **L** | hypothesized | S | Avoid main bundle retain of snippet deps |
| ST-3 | FE service | Release SourceCodeService file buffers when page destroyed; no global unbounded cache | **M** | hypothesized | M | Audit service implementation |
| ST-4 | FE IPC | Avoid duplicating HLO text already loaded by graph_viewer/op_profile — pass handles/IDs not full text | **M** | hypothesized | M | Align with HLO extract cache |
| ST-5 | Trace deep-link | Open snippet without retaining full Catapult model in background tab (message-passing / replace model) | **M** | hypothesized | M | Especially legacy iframe |
| ST-6 | URL size | Prefer `session_id` + `hlo_op` + source line over embedding entire stack frames | **L** | observed | S | Fields already support partial info |
| ST-7 | Parent tools | When stack panel opens inside op_profile/roofline, virtualize parent table rows | **L** | hypothesized | M | Secondary; parent is real cost |

**Note:** Not an XPLANE convert tool; severity is generally **L–M** vs Trace Viewer convert path (**H**). Still important for multi-tool workflows and deep-link UX.

---

## 12. Worked peak scenarios (qualitative)

### 12.1 Single-host, warm SSTABLE, V2 JSON, default viewport

```text
LevelDB filtered load → merged(=single) → JSON string ≈ W
respond gzip: W + gzip(W)
browser: gunzip + JSON graph ≈ 2–4×W + WASM SoA ≈ 1–2×W
peak browser often > plugin
```

### 12.2 Ten-host, cold open, Reduce parallel, V2 JSON

```text
Map×10: each may peak at XSpace+container+20-chunk buffers (hopefully not simultaneous)
Reduce: up to 10 filtered containers concurrent + merge + JSON
plugin peak can exceed single-host by nearly N during Reduce
browser still multiplies by overfetch×N events
```

### 12.3 Legacy Catapult long session

```text
each pan adds to _loadedTraceEvents and tr.Model
after exploring full timespan: model ≈ entire trace density (visibility-limited per fetch but union grows)
can surpass V2 single-viewport budget by large factor
```

### 12.4 Ideal future (T1+T2+T3+T4)

```text
zstd PB only on wire (no gzip wrap)
WASM decompress → SoA without args maps
Reduce concurrency ≤ 2–4
browser peak dominated by visible SoA; plugin peak by one host container + PB buffer
```

---

## 13. Measurement recommendations (analysis only)

No product code in this note; suggested instrumentation for a follow-up:

1. **Plugin:** log sizes at: post-GetXSpace, post-container convert, post-LevelDB store, post-load (event count), post-JSON/PB, post-gzip (`respond`).
2. **Browser:** already exposes `window.wasmMemoryBytes`; add performance marks around `JSON.parse` / `processTraceEvents` (partially present: `traceProcessingTime`).
3. **A/B:** `use_pb` true vs false with same session; host count 1 vs 10; ZOOM_RATIO 8 vs 4.
4. **Catapult:** sample `_loadedTraceEvents.size` after N pan operations.
5. **Multi-host:** compare ProcessSession vs Reduce RSS on identical 8-host session.

---

## 14. Relationship to other design notes

| Note | Overlap with Trace Viewer |
|------|---------------------------|
| `00-synthesis` | Portfolio ranking; TV likely top browser + multi-host convert |
| `02-convert-serve-cache` | Shared `respond()` gzip, result cache, warm-up `DEFAULT_CACHE_TOOLS` |
| `04-overview-and-stats` | Shares HostSelector / ALL_HOSTS patterns but not multi-host CSV or LevelDB |
| `05-graph-hlo` | Adjacent-nodes fetch from TV selection; HLO text not TV core path |
| `06-megascale` | `ProcessMegascaleDcn` optional on TV preprocess; Perfetto upload path separate |

Trace Viewer should be optimized **before** micro-optimizing stats DataTables when the goal is “large multi-host interactive sessions don’t OOM.”

---

## 15. Implementation sequencing (design only)

Suggested phases if product work is later approved:

| Phase | Items | Exit criteria |
|-------|-------|---------------|
| P0 | T2 (gzip policy), T17 (host required), T13 ratio tune | No double-compress PB; no accidental all-hosts; lower default overfetch |
| P1 | T1+T29 (PB end-to-end), T14 | V2 production on PB with flag default flip plan |
| P2 | T3+T16 (args lazy), T10 abort | Timeline RSS drop on large JSON/PB |
| P3 | T4+T8 multi-host bounds | 10-host Reduce stable under memory limit |
| P4 | T9 Catapult, T7 non-stream retirement | Legacy path bounded |
| P5 | T6 stream JSON, T26 arch | Only if PB insufficient for remaining JSON clients |

---

## 16. Open questions (code-derived, not blocked)

1. Does any deployment rewrite `/data` to a `.pb` filename so V2 PB path already works? (**hypothesized** no in OSS.)
2. Does warm-up for `trace_viewer@` write SSTABLEs only, or also materialize full JSON into a result cache? (see note 02.)
3. Exact RSS of `entry_args` vs SoA timing columns on a 1M-event viewport — needs measurement (T3 impact size).
4. Is `load_metadata` true on viewport loads today for streaming JSON (would pull args early)? Confirm in `LoadFromLevelDbTable` call sites / defaults.
5. Cloud AI non-stream path usage share — prioritizes T7 vs T1.

---

## 17. Summary

Trace Viewer memory is a **three-process problem** (C++ convert, Python HTTP gzip, browser JS/WASM) with a **multi-host multiplier** and a **false “streaming” name** (viewport re-query of LevelDB, not chunked HTTP). The highest-confidence, highest-severity gaps are:

1. Production V2 still on **JSON** despite PB/zstd pipeline existing (`use_pb` unwired; pathname check).
2. **Double compression** of zstd PB via default gzip.
3. **Per-event args maps** in the WASM timeline despite selection detail API.
4. **Parallel Reduce** holding N host containers.
5. **Overfetch** (3× time, 8× resolution) and **accidental all-hosts**.

Closing T1–T4 plus T13/T17 is the minimum set that changes the asymptotic behavior of large multi-host interactive sessions; ST-* items keep the stack-trace bridge from undoing wins via secondary retention.

---

*End of design note. Analysis only — no product code changes.*
