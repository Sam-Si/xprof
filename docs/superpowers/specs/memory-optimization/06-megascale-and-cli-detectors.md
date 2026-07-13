# Megascale + Remaining CLI Detectors — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** `megascale_stats`, `megascale_perfetto` frontend, `detect_layout_mismatch_copies_tool`, `detect_unfused_reshapes_tool`, `detect_unnecessary_convert_reduce_tool`, `get_overview_tool`  
**Severity labels:** **H** | **M** | **L** only  
**Confidence labels:** **observed** | **hypothesized** only  
**Companion docs:** `00-synthesis`, `01-trace-viewer-path` (~1600 lines), `02-convert-serve-cache-path` (~1600 lines), `03-memory-tools-family` (~1200 lines), `04`–`05`

This note re-derives memory peaks for the megascale family and the remaining CLI/MCP detectors from **live repository sources**. Every major claim cites **`path:line`** (or `path:start-end`) at write time. No product code changes ship with this document.

---

## How to read this document

| Convention | Meaning |
|------------|---------|
| Severity **H** | Multi-MiB intermediate/wire, double compression of large bodies, full-host IR, all-module HLO, or N× full convert for a scalar/neighborhood need |
| Severity **M** | Measurable waste, secondary path, or multi-host cold cost that is bounded relative to Perfetto/HLO extremes |
| Severity **L** | Small absolute waste, optional path, or FE retention after panel close |
| Confidence **observed** | Confirmed by code path with `path:line` cites |
| Confidence **hypothesized** | Plausible from structure; needs measurement before implementation bets |
| Path cites | Repo-relative `file:line` or `file:start-end` |
| Double gzip | Inner writer already gzip; outer `respond()` gzips again |
| Full host IR | `XprofTrace` materializes all selected lines/events for one host before Perfetto encode |
| N× HLO loads | Per-candidate (or per-module) full `graph_viewer` text / full HloProto dump |
| Overview full JSON for scalars | CLI needs ~30 keys; convert emits full UI OverviewPage JSON |

**Out of scope for this note:** Trace Viewer LevelDB (doc 01), shared convert concurrency (doc 02), memory_profile/viewer family (doc 03), OpStats fan-out tables (doc 04), graph_viewer UI itself (doc 05) except as a dependency of detector neighborhood loads.

---

## 0. Why this family matters in the portfolio

Megascale + detectors are not the warm-up default pair (`DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')` at `registry.py:55`), but they hit three high-leverage failure modes that synthesis theme ranks care about:

1. **Double compression of already-gzipped binary** — Perfetto path (`perfetto_writer.cc:405-417` + `respond.py:109-111`).  
2. **Full-host intermediate IR** — `XSpace` arena + `XprofTrace` + `perfetto::Trace` + gzip `data_` coexist (`megascale_stats_processor.cc:69-80`).  
3. **CLI projection gap** — detectors and `get_overview` pay UI-scale converts for scalar/neighborhood answers (same theme as doc 03 `summary_only` / peak-only).

Relative to Trace Viewer (doc 01): Perfetto is single-host but **not** viewport-filtered. Relative to overview warm-up (doc 02/04): `get_overview_tool` reuses the same multi-host OpStats spine then throws away most JSON.

```text
                    ┌──────────────────────────────┐
                    │  megascale_stats (table)     │  cold multi-host DCN
                    │  severity M (typical)        │
                    └──────────────┬───────────────┘
                                   │ same tool tag
                    ┌──────────────▼───────────────┐
                    │  megascale_perfetto (binary) │  full host IR + double gzip
                    │  severity H                  │
                    └──────────────────────────────┘

  CLI detectors ──► all-module HLO / N× graph text / full hlo_op_profile
  get_overview  ──► full OverviewPage JSON → keep ~30 scalars
```

---

## Registry / wiring

| Concern | Path / detail | Cite |
|---------|---------------|------|
| Tool registry | `megascale_stats` ∈ `XPLANE_TOOLS` | `registry.py:33-52` |
| ALL_HOSTS | `megascale_stats` ∈ `XPLANE_TOOLS_ALL_HOSTS_SUPPORTED` | `registry.py:58-65` |
| Overview warm-up | `DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')` — overview always multi-host cold-start cost | `registry.py:55` |
| Options common | `perfetto` bool only; **`group_tiny_events` not plumbed** | `options/base.py:31-46` |
| Python convert | `megascale_stats` → `host_name` + `perfetto` only | `raw_to_tool_data.py:226-235` |
| C++ processor | `REGISTER_PROFILE_PROCESSOR("megascale_stats", …)` | `megascale_stats_processor.cc:92` |
| Perfetto IR | `megascale_perfetto/{xspace_loader,trace_processor,perfetto_writer,xprof_trace.h}` | package |
| DCN stats | `xplane_to_dcn_collective_stats.cc`, `process_megascale_dcn.cc` | package |
| FE stats | `frontend/app/components/megascale_stats/` | package |
| FE Perfetto | `frontend/app/components/megascale_perfetto/` | package |
| CLI tools | `plugin/xprof/cli/tools/*.py` | package |
| HLO helpers | `plugin/xprof/cli/internal/oss/hlo_tools.py` | package |
| CLI cache | SQLite, default expire 86400s | `decorators.py:23-60` |
| HTTP respond | default gzip when `content_encoding is None` | `respond.py:107-111` |
| ToolDataService | always `content_encoding=None` | `tool_data.py:131-135` |

### Option plumbing gap (memory-relevant)

| Layer | Behavior | Cite | Confidence |
|-------|----------|------|------------|
| FE sets query | `group_tiny_events` from route → URL | `megascale_perfetto.ts:53`, `:110-112` | observed |
| `build_common_params` | does **not** copy `group_tiny_events` | `options/base.py:31-46` | observed |
| `raw_to_tool_data` | only `host_name`, `perfetto` | `raw_to_tool_data.py:227-230` | observed |
| C++ | `GetParam<bool>(…, "group_tiny_events").value_or(true)` | `megascale_stats_processor.cc:74-76` | observed |

So the FE URL contract for disabling grouping is **dead code** today; C++ always defaults true when the param is missing. Default-true is good for size; the gap matters for experiments and for any future more aggressive thresholds.

---

## 1. megascale_stats (table / DCN Slack path)

### 1.1 End-to-end data path

```text
UI expand panel (isExpanded=true)
  → dataService.getData(session, megascale_stats, host)     # megascale_stats.ts:101
  → HostSelector: host may be ALL_HOSTS (supported)         # registry.py:64
  → ToolDataService → raw_to_tool_data megascale_stats
  → MegascaleStatsProcessor::ProcessSession (perfetto=false)
       GetDcnSlackAnalysisByHostName(session, hostname)     # processor.cc:83-85
         ConvertMultiXSpaceToDcnCollectiveStats             # cold: scan every host
           for each host:                                   # xplane_to_dcn…:44-67
             Arena + GetXSpace(host)
             ConvertXSpaceToDcnSlackAnalysis
             WriteBinaryProto <host>.dcn_collective_stats.pb
             combiner.Combine
           WriteBinaryProto ALL_HOSTS.dcn_collective_stats.pb  #:69-72
         warm: ReadBinaryProto hostname → DcnSlackAnalysis  #:154-156
       GenerateMegaScaleJson → "[ " + DataTable.ToJson + " ]"  # process_megascale_dcn.cc:142-145
  → ToolResult(content_encoding=None)                        # tool_data.py:134
  → respond: gzip.compress(JSON)                             # respond.py:109-111
  → FE: SimpleDataTable + google.visualization.DataTable
       dataInfo retained; onPanelClosed only flips flag      # megascale_stats.ts:139-141
```

Single-host selection is required for Perfetto navigation (`openPerfetto` copies host — `megascale_stats.ts:143-155`); the table path is the aggregate DCN Slack analysis (host param selects which cached pb to read after cold multi-host generate).

### 1.2 Cold multi-host DCN convert — sequential arenas

Cold path is **sequential per host** with a **fresh Arena per iteration**:

```text
for idx in 0..XSpaceSize-1:                    # xplane_to_dcn_collective_stats.cc:44-67
  hostname = GetHostname(idx)
  Arena arena
  xspace = GetXSpace(idx, &arena)
  if !HasDcnCollectiveStatsInXSpace → write NO_HOST marker, return false  #:50-57
  dcn = ConvertXSpaceToDcnSlackAnalysis(*xspace, …)
  WriteBinaryProto(hostname, dcn)
  combiner.Combine(dcn)
Finalize → WriteBinaryProto(ALL_HOSTS, …)       #:69-72
```

| Property | Implication | Severity | Confidence |
|----------|-------------|----------|------------|
| Sequential arenas | Peak ≈ max(X_i) + analysis + combiner state, **not** Σ X_i | **M** (vs parallel Σ) | observed |
| N host pb + ALL_HOSTS | Disk growth O(N); read path only one hostname | **L**–**M** | observed |
| Early exit on first host without MegaScale: | Writes `kNoHostIdentifier` and returns false | **L** | observed `:50-57` |
| Presence check may also walk XSpaces | `HasDcnCollectiveStatsInMultiXSpace` cold loop `:101-110` | **M** first open | observed |

`HasDcnCollectiveStatsInXSpace` looks for host-plane event metadata names starting with `"MegaScale:"` (`:80-90`).

### 1.3 JSON / DataTable materialization

`GetMegaScaleDataTable` builds **~16 columns** per rendezvous summary row (`process_megascale_dcn.cc:86-103`, row fill `:109-138`):

| Column group | Examples |
|--------------|----------|
| Identity | rendezvous_name, recv/send op, transfer_type |
| Timing | slack_time, observed_duration, stalls (send/recv/done), total_stall |
| Volume | occurrences, net_tx_bytes, required_bandwidth |

`GenerateMegaScaleJson` wraps `ToJson` as a one-element JSON array string (`:142-145`).

| # | Stage | Representation | Peak notes | Confidence |
|---|-------|----------------|------------|------------|
| S1 | Cold DCN convert | Sequential per-host Arena XSpace | Peak ≈ max(X_i) + analysis | observed |
| S2 | Cold combine | `DcnSlackAnalysisCombiner` + per-host pb | N host files + ALL_HOSTS | observed |
| S3 | Warm read | One `DcnSlackAnalysis` protobuf | Small vs XSpace; scales with #collectives | observed |
| S4 | JSON | `GetMegaScaleDataTable` → `ToJson` | One row × ~16 columns per rendezvous | observed |
| S5 | HTTP | raw JSON + gzip | Double buffer (shared path) | observed (`respond.py:109-111`) |
| S6 | FE | JS object + Google Charts DataTable | Held after panel close | observed (`megascale_stats.ts:139-141`) |

### 1.4 Severity assessment (table path)

Table path is **moderate** relative to Perfetto/trace. Cold multi-host convert is the real cost; warm path is light. Pathological rendezvous counts could grow the DataTable, but typical cost is dominated by XSpace parse, not JSON.

### 1.5 Opportunities — megascale_stats table

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| MS-1 | Lazier cold DCN: convert requested host first; defer ALL_HOSTS combine | **M** | hypothesized | M |
| MS-2 | Free FE `dataTable` / `dataInfo` on panel close / host change | **L** | observed | S |
| MS-3 | Skip re-gzip if body already compressed (shared) | **M** | observed | S |
| MS-4 | Bound / paginate table rows for pathological #rendezvous | **L** | hypothesized | M |
| MS-5 | Skip presence walk that reloads XSpaces when cache marker exists | **L** | observed | S |
| MS-6 | Stream combine without holding full per-host analysis in combiner longer than needed | **L** | hypothesized | L |

**Top for table path:** MS-1 (cold multi-host), then shared gzip policy (MS-3 / portfolio theme 3).

### 1.6 Relationship to Trace Viewer DCN

`ProcessMegascaleDcn` mutates XSpace for **trace** preprocess (`process_megascale_dcn.cc:58-81`) — adding host/device DCN traffic lines. That path is optional on TV (`enable_legacy_dcn`, doc 01). It is **not** the megascale_stats table path; table path uses `ConvertXSpaceToDcnSlackAnalysis` → disk pb → JSON. Do not conflate the two when sizing.

---

## 2. megascale_perfetto (Perfetto export path)

### 2.1 End-to-end data path

```text
UI /megascale_perfetto/:sessionId?host=H[&group_tiny_events=…]
  → buildPerfettoDataURL: tag=megascale_stats&perfetto=true&host=H
       # megascale_perfetto.ts:98-114
  → fetch /data → ToolDataService → raw_to_tool_data
       options = {host_name, perfetto} only          # raw_to_tool_data.py:227-230
  → MegascaleStatsProcessor (perfetto=true):
       Arena + GetXSpaceByName(H)                    # processor.cc:70-72
       XSpaceLoader::Load → XprofTrace               # :73  FULL host IR
       TraceProcessor::Process                       # :76-77
       PerfettoWriter::WriteToString(…, compressed_output=true)
         Arena perfetto::Trace
         GzipOutputStream level=1 → std::string data_  # perfetto_writer.cc:399-417
  → content_type application/octet-stream            # processor.cc:78
  → ToolResult content_encoding=None                 # tool_data.py:134
  → respond: gzip.compress(already-gzip body)        # respond.py:109-111  ★ DOUBLE GZIP
  → FE: response.blob() → arrayBuffer()              # megascale_perfetto.ts:128-129
       postMessage({perfetto: {buffer}}, origin)     # :168-173  NO transfer list → clone
```

**Note:** There is no separate `megascale_perfetto` convert tool — Perfetto reuses `megascale_stats` with `perfetto=true` (`megascale_perfetto.ts:105-106`).

### 2.2 Full host IR — XprofTrace structure

`XprofTrace` (`xprof_trace.h:106-127`) holds:

| Field | Source | Memory character |
|-------|--------|------------------|
| `string_table` | Interned arg keys/string values | Good for args; **not** used for event names |
| `tpu_fragments` | TPU planes, allowed lines only | `Steps`, `XLA Modules`, `XLA Ops`, `XLA TraceMe` (`xspace_loader.cc:58-66`) |
| `megascale_fragments` | CUSTOM Megascale Trace plane | **All** Megascale events (not line-filtered like TPU) |
| Counter tracks | rx/tx, bandwidth, inflight collective | Separate SoA-like vectors |

**Event.name is `std::string`, not StringId** (`xprof_trace.h:77-78`). Loader assigns:

```text
event.name = event_visitor.DisplayName() or Name()   # xspace_loader.cc:203-208, 271-276
```

Args keys/values go through `string_table.Intern` (`xspace_loader.cc:97-128`). So duplicate op names pay full string copies per event — classic IR bloat.

SkipStat already drops flops, kernel_details, source_stack, etc. (`xspace_loader.cc:37-55`) — partial projection, but still full time-range of selected lines.

### 2.3 Peak coexistence window (server)

During `ProcessSession` perfetto branch, the following can coexist until function return:

| Object | Created at | Cite |
|--------|------------|------|
| `Arena` + full host `XSpace` | GetXSpaceByName | `processor.cc:70-72` |
| `XprofTrace` (full IR) | XSpaceLoader::Load | `:73` |
| TraceProcessor in-place mutations | Process() | `:76-77` |
| `Arena` + `perfetto::protos::Trace` | WriteToString | `perfetto_writer.cc:402-404` |
| gzip `std::string* output` (`data_`) | GzipOutputStream | `:405-417` |
| Python bytes + second gzip | respond | `respond.py:109-111` |

This is the **full host IR + encode** story: no viewport, no device subset, no streaming write of IR chunks.

### 2.4 Double gzip — exact chain (must-cite)

| Step | What | Cite | Confidence |
|------|------|------|------------|
| 1 | C++ writes gzip level=1 into `data_` | `perfetto_writer.cc:405-417` (`compressed_output=true` from `processor.cc:79-80`) | observed |
| 2 | Python convert returns raw bytes, content_type octet-stream | `raw_to_tool_data.py:231-235` | observed |
| 3 | ToolDataService sets `content_encoding=None` always | `tool_data.py:131-135` | observed |
| 4 | `respond()` sees falsy encoding → `gzip.compress(body)` again | `respond.py:107-111` | observed |
| 5 | Client receives double-gzipped body as single Content-Encoding: gzip | — | observed (browser may fail or inflate once depending on stack) |

**Severity H**, **confidence observed**. Fix is small (set encoding identity/gzip correctly, or skip outer compress for precompressed tools) and multiplies with Trace PB double-compress (doc 01 T2, doc 02 encoding hygiene).

Evidence that truthy `content_encoding` skips re-gzip: `http_respond_test.py` comment path (`respond.py` branch `:107-108`; test notes identity path).

### 2.5 TraceProcessor memory-relevant behaviors

| Behavior | Detail | Cite | Severity impact |
|----------|--------|------|-----------------|
| group_tiny_events | Default true; collapses sub-threshold events | `trace_processor.h:17`, `processor.cc:74-76` | reduces IR/output when enabled |
| Sort / run_id / flows | In-place + maps | `trace_processor.cc` Process | temporary maps |
| Counters | Delta series for bw | counter tracks in IR | moderate |
| URL message for disable | Suggests `&group_tiny_events=false` | `trace_processor.cc:192` | dead without plumbing |

### 2.6 Materialization by stage (Perfetto)

| # | Stage | Representation | Peak notes | Confidence |
|---|-------|----------------|------------|------------|
| P1 | GetXSpace | Arena full host `XSpace` | Dominates for large multi-chip hosts | observed |
| P2 | XSpaceLoader | `XprofTrace` full event copy (selected lines) | **Full host IR**; `Event.name` not interned | observed (`xprof_trace.h:77-78`, `xspace_loader.cc:203-208`) |
| P3 | Args | StringTable interns keys/string values | Better than raw XStats; still O(events × args) | observed |
| P4 | Megascale buffer | fragment maps of tracks | Temporary list/map overhead during load | observed |
| P5 | TraceProcessor | sort, tiny-event collapse, flows, counters | Grouping reduces events when enabled | observed |
| P6 | Perfetto proto | Full `perfetto::protos::Trace` on Arena | ≈ expanded IR size | observed |
| P7 | Inner gzip | `data_` string | XSpace + IR + Trace + gzip coexist | observed |
| P8 | Outer gzip | `respond.py` | **Double compress** when encoding unset | observed |
| P9 | Browser | blob + ArrayBuffer (+ postMessage clone) + Perfetto iframe heap | Dominant client peak; empty buffer → "Trace data too big" | observed (`megascale_perfetto.ts:128-135`) |

### 2.7 Frontend Perfetto retention / clone

```text
const blob = await response.blob();                 # :128
const trace = await blob.arrayBuffer();             # :129
…
perfettoWindow.postMessage(
  { perfetto: { buffer: trace, title, url, keepApiOpen: true } },
  this.perfettoOrigin,
);                                                  # :168-173
// No second argument transfer list → structured clone of ArrayBuffer
```

| ID | Issue | Severity | Confidence |
|----|-------|----------|------------|
| FE-MS-2 | postMessage without transfer list clones buffer | **H** | observed |
| FE-MS-3 | Cross-origin `ui.perfetto.dev` — no control of viewer heap after handoff | **M** | observed constraint |
| FE-MS-5 | blob + arrayBuffer may overlap until GC | **M** | hypothesized |
| FE-MS-6 | Empty body mapped to "Trace data too big" (may also mean OOM/ truncation) | **M** | observed message `:130-135` |

### 2.8 Opportunities — megascale_perfetto

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| MP-1 | Set `content_encoding` (or skip outer gzip) for pre-gzipped Perfetto | **H** | observed | S |
| MP-2 | Stream / chunk Perfetto write; avoid XSpace+IR+Trace+gzip simultaneously | **H** | hypothesized | L |
| MP-3 | Intern `Event.name` (StringId) like args | **H** | observed | M |
| MP-4 | Viewport / time-range / device subset export (not full host) | **H** | hypothesized | L |
| MP-5 | FE: transfer ArrayBuffer in `postMessage` transfer list; drop blob after `arrayBuffer` | **H** | observed | S |
| MP-6 | Plumb `group_tiny_events` end-to-end; optional aggressive thresholds | **M** | observed | S |
| MP-7 | Drop non-essential lines/args earlier (already skips some StatTypes) | **M** | hypothesized | M |
| MP-8 | Disk/LevelDB intermediate like `trace_viewer@` for re-open | **M** | hypothesized | L |
| MP-9 | Server size budget → explicit error vs empty body | **M** | hypothesized | M |
| MP-10 | Share encoding fix with zstd PB traces (doc 01 T2) | **H** | observed | S |
| MP-11 | Free XSpace arena before building perfetto Trace (lifetime split) | **H** | hypothesized | M |

**Top:** MP-1, MP-5, MP-3, MP-2/MP-4.

---

## 3. Frontend retention (both megascale UIs)

| ID | Opportunity | Severity | Confidence | Cite |
|----|-------------|----------|------------|------|
| FE-MS-1 | `onPanelClosed` does not clear `dataInfo` / Dashboard `dataTable` | **L** | observed | `megascale_stats.ts:139-141` |
| FE-MS-2 | Perfetto clone without transfer keeps peak high | **H** | observed | `megascale_perfetto.ts:168-173` |
| FE-MS-3 | Cross-origin Perfetto iframe — no control of viewer heap | **M** | observed | constraint |
| FE-MS-4 | Host switch does not abort in-flight fetch | **L** | hypothesized | no AbortController in stats `:101-124` |
| FE-MS-7 | Stats `dataInfo` reassigned with full SimpleDataTable on each load | **L** | observed | `:117-120` |
| FE-MS-8 | Perfetto `ngOnDestroy` cleans ping only — no buffer ref held by design | **L** | observed | `:192-202` |

### 3.1 Stats panel expand gate

`update()` no-ops unless `isExpanded` (`megascale_stats.ts:92-95`). Good: collapsed panel does not fetch. Bad: once expanded once, closing does not free parsed tables (FE-MS-1).

---

## 4. CLI: detect_layout_mismatch_copies_tool

**Path:** `plugin/xprof/cli/tools/detect_layout_mismatch_copies_tool.py` (~1100 lines of detector logic)

### 4.1 Data path

```text
detect_layout_mismatch_copies(session, limit=100)
  → hlo_tools._fetch_debug_info(session)   // ALL modules HloProto
       # :858  (1P helper; not in OSS hlo_tools.py)
  → get_top_hlo_ops(session, limit=100)    // full hlo_op_profile convert + flatten
       # :864
  → for each module in debug_info.hlo_proto:  # :893
       build instr_by_id, users_by_id, callers, roots   // O(N) maps  # :896-918
       for EVERY opcode=="copy":                        # :920-922
         BFS upstream max_depth=5
         BFS downstream max_depth=5
         if both sides compute → emit bottleneck
  → json.dumps(indent=2)
```

### 4.2 OSS vs 1P note on `_fetch_debug_info`

| Surface | Behavior | Confidence |
|---------|----------|------------|
| Tool calls `_fetch_debug_info` | protected access at `:858` | observed |
| OSS `hlo_tools.py` | implements neighborhood via graph_viewer text; **no** `_fetch_debug_info` export in the reviewed OSS file | observed |
| Tests | mock `_fetch_debug_info` / DebugInfoCollection | observed |
| Production 1P | dump-all multi-module HLO into one collection | hypothesized for exact RPC shape; **observed** that tool requires all modules resident |

**Implication:** memory model for this tool is **all modules' HloProto in process at once**, then O(N) indices, then BFS over **every** copy.

### 4.3 Materialization

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| L1 | All modules' `HloModule` protos resident | Large multi-module sessions | **H** | observed |
| L2 | Connectivity maps per module | O(N) memory; CPU already O(N) not O(N²) | **M** | observed |
| L3 | BFS per **every** copy (not top-K filtered) | CPU heavy; visited sets per search | **H** (CPU) / **M** (RAM) | observed |
| L4 | Full `hlo_op_profile` via `get_top_hlo_ops` | Extra full-profile convert (cached 86400s) | **H** first hit | observed |
| L5 | Result list of sandwiched copies | Usually small vs L1 | **L** | observed |

### 4.4 Opportunities — layout mismatch

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| LMC-1 | Do not load all modules at once — stream module-by-module from `*.hlo_proto.pb` | **H** | observed | M |
| LMC-2 | Restrict candidates to top-K copies by bytes/time **before** BFS | **H** | observed | S |
| LMC-3 | Share one HLO index / debug_info cache across detector tools | **H** | hypothesized | L |
| LMC-4 | Early filter: only copies with layout mismatch / non-optimal minor dim | **M** | hypothesized | M |
| LMC-5 | Avoid full op_profile when only enriching metrics for matches | **M** | hypothesized | M |
| LMC-6 | OSS implement `_fetch_debug_info` via on-disk protos (lazy) not dump-all | **H** | observed | M |
| LMC-7 | Drop `indent=2` for large result lists (wire only) | **L** | observed | S |

**Top:** LMC-1, LMC-2, LMC-3.

---

## 5. CLI: detect_unfused_reshapes_tool

**Path:** `plugin/xprof/cli/tools/detect_unfused_reshapes_tool.py`

### 5.1 Data path — N× HLO / graph text loads

```text
detect_unfused_reshapes(session, limit=50)
  → get_top_hlo_ops → top_by_bytes_accessed          # :37
  → filter formatting categories (reshape/transpose/copy/…)  # :44-56
  → for each candidate:                              # :75
       guess module from name path                   # :83-88
       get_hlo_neighborhood(instr, radius=2, module)
         → graph_viewer long/short_txt for FULL module text
              # hlo_tools.py:266-275
         → parse all lines into adjacency            # :280+
         → BFS radius
       on miss: list_hlo_modules + try EVERY module neighborhood  # :102-120
  → json.dumps findings                              # :177-184
```

### 5.2 Why this is N× full HLO text

`get_hlo_neighborhood` (**cached 86400s** at `hlo_tools.py:220`) still:

1. Resolves module name against `*.hlo_proto.pb` files (`:251-264`).  
2. Fetches **full module text** via `graph_viewer.json` with `type: long_txt|short_txt` (`:266-275`).  
3. Parses **every line** of that full text into `line_by_name`, `operands_by_name`, `users_by_name` (`:280+`).  
4. Returns only the neighborhood string — but the convert + parse cost is **full module**.

Unfused detector multiplies this by candidates, and on miss by modules:

| Scenario | Converts | Cite |
|----------|----------|------|
| Happy path, module in op name | ~1 graph_viewer full text per candidate | `:93-98` |
| Module miss fallback | list modules once + **candidates × modules** neighborhood calls | `:102-120` |
| Cache hit | SQLite returns full neighborhood string; still large string RAM | `decorators.py` |

This is the classic **N× HLO load** anti-pattern (same spirit as `get_peak_allocations_tool` multi-module loop in doc 03).

### 5.3 Materialization / amplification

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| U1 | Full op_profile for top bytes | First hit; then CLI SQLite cache | **H** | observed |
| U2 | **Per-candidate full module HLO text** via `graph_viewer` | N× full text convert | **H** | observed (`hlo_tools.py:266-275`, tool `:93-98`) |
| U3 | Module scan fallback: candidates × modules × full text | Worst case explosion | **H** | observed (`:110-120`) |
| U4 | `list_hlo_modules` / neighborhood cached 86400s | Mitigates **repeat** calls only | **M** | observed |
| U5 | Text graph rebuild each neighborhood call | Even first analysis pays full parse | **H** | observed |

### 5.4 Opportunities — unfused reshapes

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| URS-1 | Binary neighborhood from HloProto (no full text dump) | **H** | observed | M–L |
| URS-2 | One module load + multi-instruction queries | **H** | observed | M |
| URS-3 | Cap module fallback search; require module in op name | **H** | observed | S |
| URS-4 | Reuse layout-mismatch style in-memory index (shared) | **H** | hypothesized | L |
| URS-5 | Lower default limit / only top formatting ops | **M** | observed | S |
| URS-6 | Cache module adjacency graph, not only final neighborhood string | **H** | hypothesized | M |

**Top:** URS-1, URS-2, URS-3.

---

## 6. CLI: detect_unnecessary_convert_reduce_tool

**Path:** `plugin/xprof/cli/tools/detect_unnecessary_convert_reduce_tool.py` (~700 lines)

### 6.1 Data path

```text
detect_unnecessary_convert_reduce(session, limit=50)
  → get_top_hlo_ops (top_by_time + top_by_bytes)     # dual lists enrichment
  → _fetch_debug_info → ALL modules
  → map modules by name
  → for each top op:                                   # candidate-driven
       resolve module + instruction
       lazy _HloModuleTracer (indices: instructions, consumers, fusion_callers, …)
            # class :77-110
       find reduce / fusion-inner reduce
       trace_upcast (DFS) + trace_downcast (DFS)
  → dedupe → json.dumps
```

Better than layout-mismatch: **candidate-driven** from top ops, not full copy scan. Still pays **all-modules HLO load** up front.

### 6.2 _HloModuleTracer indices

Per touched module, tracer builds (`:80-110`):

| Index | Purpose |
|-------|---------|
| `computations`, `instructions` | id → proto |
| `instruction_to_computation` | ownership |
| `fusion_callers` | called_comp → (comp, instr) |
| `consumers` | operand → users |
| `computation_parameters` | param number map |
| `instructions_by_name` | name lookup for top ops |

Tracer is **lazy per module name** once op hits it (pattern `if full_mod_name not in tracers` around `:569-570`) — good within-run reuse — but modules collection still loaded wholesale via debug_info.

### 6.3 Materialization

| # | What | Risk | Severity | Confidence |
|---|------|------|----------|------------|
| C1 | All-module HLO protos | Same as LMC L1 | **H** | observed |
| C2 | Tracer indices per touched module | O(N) per module; reused within run | **M** | observed |
| C3 | DFS upcast/downcast with visited | Bounded by allowed opcodes (`_ALLOWED_TRACING_OPCODES` `:16-59`) | **L**–**M** | observed |
| C4 | Top ops + full profile convert | Shared with other detectors | **H** first hit | observed |
| C5 | Timing logs for fetch vs core | Useful for production sizing | **L** | observed |

### 6.4 Opportunities — convert/reduce

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| UCR-1 | Lazy per-module HLO load only for modules in top-ops set | **H** | observed | M |
| UCR-2 | Shared HLO session index with other detectors | **H** | hypothesized | L |
| UCR-3 | Optional skip of top_ops dual lists (time+bytes doubles work) | **M** | hypothesized | S |
| UCR-4 | Bound DFS visited size / depth hard limits | **L** | hypothesized | S |
| UCR-5 | Metrics-only profile extract instead of full hlo_op_profile UI | **H** | hypothesized | M |

**Top:** UCR-1, UCR-2.

---

## 7. CLI: get_overview_tool

**Path:** `plugin/xprof/cli/tools/get_overview_tool.py`  
**Not covered in depth elsewhere** (overview appears as warm-up / ALL_HOSTS in doc 02 and table tools in doc 04).

### 7.1 Data path — full JSON for scalars

```text
get_overview(session)  [@cached 86400s]                 # :54-55
  → client.fetch(overview_page.json)                    # :71-75
       OverviewPageProcessor:
         ConvertMultiXSpaceToCombinedOpStatsWithCache   # overview_page_processor.cc:88-89
         ConvertOpStatsToOverviewPage                   # :92
         [!training] ConvertMultiXSpaceToInferenceStats # :93-102
         OverviewPageToJson  // full UI JSON sections   # :106
  → json.loads full overview                            # get_overview_tool.py:87
  → extract PERFORMANCE_SUMMARY_KEYS + RUN_ENVIRONMENT_KEYS
       (+ stat_/sc_/run_ prefixes)                      # :13-51, :114-128
  → discard rest → small summary JSON                   # :144-149
```

### 7.2 Keys actually kept (order-of-magnitude)

| Set | Count (approx) | Examples |
|-----|----------------|----------|
| `PERFORMANCE_SUMMARY_KEYS` | ~23 fixed + all `stat_*` / `sc_*` | steptime, mxu, duty cycle, HBM bw (`:13-36`) |
| `RUN_ENVIRONMENT_KEYS` | ~11 fixed + `run_*` | device_type, host_count, build_target (`:40-51`) |
| Dropped | Entire DataTable sections, diagnostics tables, most overview UI blocks | full list of sections parsed only to merge `p` dicts (`:91-94`) |

CLI therefore pays:

1. Multi-host combined OpStats (same as UI overview / warm-up).  
2. Full OverviewPage proto → JSON tables.  
3. Optional second multi-XSpace walk for inference latency (`overview_page_processor.cc:53-65`, `:93-102`).  
4. Python `json.loads` of the entire UI payload.  
5. Then throws away everything except ~30–100 scalars.

### 7.3 Materialization

| # | What | Waste | Severity | Confidence |
|---|------|-------|----------|------------|
| O1 | Multi-host combined OpStats | Dominant; shared with UI overview / warm-up | **H** | observed |
| O2 | Full OverviewPage JSON for CLI that needs ~30 scalars | **Dead weight on wire + parse** | **H** | observed (`:71-94` vs `:114-149`) |
| O3 | Inference path re-walks multi-XSpace | Extra peak for non-training | **M** | observed (`processor.cc:93-102`) |
| O4 | CLI holds full parsed list of sections then drops | Transient Python heap | **M** | observed |
| O5 | SQLite caches slim output (good) | First call still full cost | **L** (after warm) | observed |

### 7.4 Opportunities — get_overview

| ID | Opportunity | Severity | Confidence | Effort |
|----|-------------|----------|------------|--------|
| GOV-1 | `summary_only` / `cli` convert mode emitting scalars only | **H** | observed | M |
| GOV-2 | Reuse warm OpStats disk cache without re-serializing full OverviewPage UI tables | **H** | observed | M |
| GOV-3 | Skip inference latency block for CLI summary | **M** | hypothesized | S |
| GOV-4 | Shared policy with memory/util projection modes (doc 03 FAM/GMP themes) | **H** | observed | M |
| GOV-5 | Server-side field allowlist query param | **H** | hypothesized | M |

**Top:** GOV-1 (mirror memory-family `summary_only`).

### 7.5 Warm-up interaction

Because `overview_page` is in `DEFAULT_CACHE_TOOLS` (`registry.py:55`), interactive UI sessions often already paid O1. CLI first hit on a **cold** session still pays full multi-host convert. Projection mode still matters for:

- wire size to CLI/MCP agents,  
- Python parse heap,  
- inference double-walk,  
- cache stampede when many agents call get_overview in parallel (doc 02 single-flight theme).

---

## 8. Cross-cutting: detectors + HLO + top ops

| ID | Opportunity | Severity | Confidence | Primary evidence |
|----|-------------|----------|------------|------------------|
| DET-1 | Shared `SessionHloIndex`: lazy module map, single connectivity build | **H** | observed | LMC/UCR all-module load; URS N× text |
| DET-2 | Detectors must not force full `hlo_op_profile` UI proto — metrics-only extract | **H** | hypothesized | all three detectors call `get_top_hlo_ops` |
| DET-3 | Prefer on-disk `*.hlo_proto.pb` (`generate_hlo_protos`) over opaque `_fetch_debug_info` dump-all | **H** | observed | URS uses files; LMC/UCR use dump-all |
| DET-4 | Never fetch full graph_viewer text for structural graph queries | **H** | observed | `hlo_tools.py:266-275` |
| DET-5 | Single-flight convert when MCP tools fan-out in parallel | **H** | hypothesized | doc 02 concurrency |
| DET-6 | JSON `indent=2` for large inefficient_ops lists | **L** | observed | all detectors |
| DET-7 | Shared CLI decorator cache is result-only — does not share intermediate HLO index across tools | **H** | observed | `decorators.py` caches return strings |
| DET-8 | Align with peak allocations N+1 (doc 03) under one "HLO amortization" epic | **H** | observed | synthesis theme 10 |

### 8.1 Shared amplification math (illustrative)

Let:

- `M` = number of HLO modules  
- `C` = number of formatting/copy candidates after top-K filter  
- `P` = cost of full `hlo_op_profile` convert  
- `G(m)` = cost of full graph_viewer text for module `m`  
- `H` = cost of loading all modules' HloProto once  

| Tool | Dominant cost model |
|------|---------------------|
| layout mismatch | `H + P + Σ_modules O(|instr| + |copies|·BFS)` |
| unfused reshapes | `P + Σ_c G(m_c)` or `P + C·M·G` fallback |
| convert/reduce | `H + P + Σ_touched O(|instr| + DFS)` |
| get_overview | `OpStats_all_hosts + OverviewJSON_full` |

Projection + lazy HLO collapse the `H` and `G(m)` terms.

---

## 9. Shared HTTP / convert path touchpoints (this family)

| Touchpoint | Behavior for this family | Cite | Severity |
|------------|--------------------------|------|----------|
| `content_encoding=None` | Forces outer gzip for Perfetto binary | `tool_data.py:134` | **H** |
| `respond` default gzip | Double buffer always | `respond.py:109-111` | **H** (with MP-1) |
| `build_common_params` | Drops `group_tiny_events` | `options/base.py:31-46` | **M** |
| `raw_to_tool_data` megascale | Does not forward group_tiny | `raw_to_tool_data.py:227-230` | **M** |
| HostSelector ALL_HOSTS | Supported for table path | `registry.py:64` | **M** cold |
| CLI SQLite 86400s | Amortizes repeat agent calls; not first peak | `decorators.py` | mitigates repeats |
| CSP frame-src perfetto | Allows `https://ui.perfetto.dev` | `respond.py:89-91` | N/A memory |

---

## 10. Ranked priority (this family)

| Rank | ID | Description | Severity | Confidence | Effort |
|------|-----|-------------|----------|------------|--------|
| 1 | MP-1 | Stop double-gzip of Perfetto payload | **H** | observed | S |
| 2 | URS-1/2 | Neighborhood without full HLO text × N | **H** | observed | M–L |
| 3 | LMC-1 / UCR-1 | Lazy / streaming per-module HLO | **H** | observed | M |
| 4 | GOV-1 | Overview summary_only for CLI | **H** | observed | M |
| 5 | MP-5 / MP-3 | FE transfer + intern event names | **H** | observed | S–M |
| 6 | DET-1 | Shared HLO index across detectors | **H** | hypothesized | L |
| 7 | LMC-2 | Top-K copy candidates only | **H** | observed | S |
| 8 | MP-2 / MP-4 | Subset / streaming Perfetto export | **H** | hypothesized | L |
| 9 | MS-1 | Lazier DCN ALL_HOSTS cold path | **M** | hypothesized | M |
| 10 | MP-6 | Plumb group_tiny_events | **M** | observed | S |
| 11 | DET-2 | Metrics-only top ops extract | **H** | hypothesized | M |
| 12 | GOV-3 | Skip inference walk for CLI summary | **M** | hypothesized | S |
| 13 | FE-MS-1 | Clear stats DataTable on panel close | **L** | observed | S |
| 14 | DET-6 | Compact JSON for detector results | **L** | observed | S |

---

## 11. Severity × confidence matrix (summary)

| Tool | Dominant risk | Severity | Confidence |
|------|---------------|----------|------------|
| megascale_stats (table) | Cold multi-host DCN convert | **M** | observed |
| megascale_perfetto | Full XSpace→IR→Trace + **double gzip** + FE clone | **H** | observed |
| detect_layout_mismatch | All-module HLO + all copies BFS | **H** | observed |
| detect_unfused_reshapes | **N× full graph_viewer text** | **H** | observed |
| detect_unnecessary_convert_reduce | All-module HLO (candidate-limited logic) | **H** | observed |
| get_overview_tool | **Full multi-host OpStats/Overview JSON for scalars** | **H** (first hit) | observed |

---

## 12. Detailed peak sketches (for measurement design)

### 12.1 Perfetto server RSS sketch (hypothesized shape; measure in Phase 0)

```text
RSS_peak ≈ size(XSpace_host)
         + size(XprofTrace)          // ≈ events × (name + args + flows)
         + size(perfetto::Trace)
         + size(gzip_level1)
         + size(python_bytes)
         + size(gzip_outer)          // double
```

Measurement hooks (analysis only — not implemented here):

- Log byte sizes of `data_` before return from C++.  
- Log `len(body)` before/after `gzip.compress` in respond for `tag=megascale_stats&perfetto=true`.  
- Browser Performance memory / arrayBuffer byteLength at postMessage time.

### 12.2 Unfused reshape convert count sketch

```text
converts ≈ 1 (top_ops) + C (neighborhood hits)
         + 1 (list modules if any miss)
         + C_miss × M (fallback)
```

Log candidate count, miss rate, and modules list length in detector runs.

### 12.3 Overview scalar ratio sketch

```text
ratio = sizeof(slim_json) / sizeof(overview_page.json)
```

Expect ratio ≪ 1 (hypothesized); confirm on multi-host production sessions.

---

## 13. Dependencies on other design notes

| This-family item | Depends on / shares with | Doc |
|------------------|--------------------------|-----|
| MP-1 double gzip | Shared respond encoding policy; also Trace PB | 01, 02 |
| MP-2/4 streaming export | Trace viewport philosophy | 01 |
| GOV-1 summary_only | Projection modes epic; OpStats combine | 02, 03, 04 |
| URS-1 binary neighborhood | graph_viewer convert cost analysis | 05 |
| DET-1 SessionHloIndex | peak allocations multi-module loop | 03 |
| MS-1 lazy DCN | ALL_HOSTS combine patterns | 02, 04 |
| DET-5 single-flight | convert concurrency semaphore | 02 |

### 13.1 What not to fix first here

- Do not micro-optimize Google Charts DataTable retention (FE-MS-1, **L**) before MP-1.  
- Do not redesign DCN combiner (MS-6) before measuring multi-host cold RSS.  
- Do not invent a second Perfetto tool name — fix options plumbing on existing `megascale_stats`.

---

## 14. Suggested implementation order (no code in this doc)

1. **MP-1** content_encoding for precompressed Perfetto (and share with zstd/pb traces).  
2. **URS-1/2** + **DET-4** kill full-text neighborhood for detectors.  
3. **LMC-1 / UCR-1 / DET-1 / DET-3** lazy modular HLO index.  
4. **GOV-1** overview summary mode for CLI / MCP.  
5. **MP-3 / MP-5** Perfetto IR + FE transfer polish.  
6. **LMC-2** top-K copies.  
7. **MS-1 / MP-4** scale multi-host / full-host exports.  
8. **MP-6** plumb group_tiny_events for experiments.  
9. **FE-MS-1 / DET-6** polish.

### 14.1 Exit criteria (design)

| Item | Exit criterion |
|------|----------------|
| MP-1 | `Content-Encoding` correct; single gzip layer; golden test for perfetto tag |
| URS-1/2 | Zero `graph_viewer` full-text calls for neighborhood on binary path |
| LMC-1 / UCR-1 | Peak RSS independent of modules not in candidate set |
| GOV-1 | CLI wire size within small multiple of slim JSON |
| MP-5 | postMessage uses transfer list; no structured-clone of full buffer |

---

## 15. Open questions (code-derived, not blocked)

1. Does any proxy gunzip once so double-gzip is invisible but still CPU-heavy? (**hypothesized** environment-dependent.)  
2. What is the production distribution of megascale host XSpace sizes vs trace_viewer hosts?  
3. How often do unfused reshape candidates miss module name path (fallback M factor)?  
4. Is `_fetch_debug_info` already partial in any 1P deployment, or always full dump?  
5. Empty Perfetto body: true size limit vs worker OOM kill vs gzip failure?  
6. Should Perfetto export reuse LevelDB from `trace_viewer@` preprocess when DCN lines already materialised?  
7. For GOV-1, is OpStats disk cache sufficient to skip OverviewPage entirely for CLI?  

---

## 16. Test / golden touchpoints (for future work, not this campaign)

| Area | Existing signal | Cite |
|------|-----------------|------|
| respond identity encoding | unit test expects no re-gzip when encoding set | `http_respond_test.py` |
| Perfetto writer compress | WriteToString tests with `compressed_output=false` | `perfetto_writer_test.cc` |
| group_tiny_events | TraceProcessor tests true/false | `trace_processor_test.cc` |
| Detectors | heavy mocks of `_fetch_debug_info` / top ops | `plugin/xprof/cli/tests/` |
| Overview | CLI extracts p_dict keys | `get_overview_tool.py` itself |

Any MP-1 fix should extend golden characterization for `tag=megascale_stats&perfetto=true` content-encoding.

---

## 17. Paths cited

### 17.1 Megascale convert / FE

- `frontend/app/components/megascale_stats/megascale_stats.ts`  
- `frontend/app/components/megascale_stats/megascale_stats.ng.html`  
- `frontend/app/components/megascale_perfetto/megascale_perfetto.ts`  
- `frontend/app/components/megascale_perfetto/megascale_perfetto.ng.html`  
- `frontend/app/components/chart/dashboard/dashboard.ts`  
- `plugin/xprof/convert/raw_to_tool_data.py` (`:226-235`)  
- `plugin/xprof/profile_plugin/tools/options/base.py` (`:20-46`)  
- `plugin/xprof/profile_plugin/tools/registry.py` (`:33-70`)  
- `plugin/xprof/profile_plugin/services/tool_data.py` (`:131-135`)  
- `plugin/xprof/profile_plugin/services/hosts.py`  
- `plugin/xprof/profile_plugin/http/respond.py` (`:29-118`)  
- `xprof/convert/megascale_stats_processor.cc` (`:60-92`)  
- `xprof/convert/process_megascale_dcn.cc` (`:58-145`)  
- `xprof/convert/xplane_to_dcn_collective_stats.cc` (`:41-160`)  
- `xprof/convert/megascale_perfetto/xprof_trace.h` (`:77-127`)  
- `xprof/convert/megascale_perfetto/xspace_loader.cc` (`:37-66`, `:168+`, `:203-208`)  
- `xprof/convert/megascale_perfetto/trace_processor.cc` / `.h`  
- `xprof/convert/megascale_perfetto/perfetto_writer.cc` (`:399-424`)  

### 17.2 CLI detectors / overview

- `plugin/xprof/cli/tools/detect_layout_mismatch_copies_tool.py` (`:837+`)  
- `plugin/xprof/cli/tools/detect_unfused_reshapes_tool.py` (`:12-191`)  
- `plugin/xprof/cli/tools/detect_unnecessary_convert_reduce_tool.py` (`:77-110`, `:550+`)  
- `plugin/xprof/cli/tools/get_overview_tool.py` (`:13-158`)  
- `plugin/xprof/cli/tools/get_top_hlo_ops_tool.py`  
- `plugin/xprof/cli/internal/oss/hlo_tools.py` (`:220-275`)  
- `plugin/xprof/cli/internal/decorators.py`  
- `xprof/convert/overview_page_processor.cc` (`:46-111`)  
- `xprof/convert/overview_page_processor.h`  

### 17.3 Docs cross-ref

- `docs/superpowers/specs/memory-optimization/01-trace-viewer-path.md` (gzip / FE transfer patterns; ~1600 lines)  
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md` (ALL_HOSTS, overview warm-up, respond; ~1600 lines)  
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (summary_only / N+1 convert; ~1200 lines)  
- `docs/superpowers/specs/memory-optimization/00-synthesis-memory-optimization.md` (portfolio ranking)  
- `docs/superpowers/specs/memory-optimization/04-overview-and-stats-tools.md`  
- `docs/superpowers/specs/memory-optimization/05-graph-hlo-util-tools.md`  

---

## 18. Appendix A — Double gzip call stack (annotated)

```text
Browser fetch(perfettoDataUrl)
  GET /data?tag=megascale_stats&perfetto=true&host=H
    data_api.data_route
      ToolDataService.get_tool_data
        raw_to_tool_data.xspace_to_tool_data
          pywrap → MegascaleStatsProcessor::ProcessSession
            if perfetto:
              Arena; GetXSpaceByName                  # full host
              XprofTrace = XSpaceLoader::Load         # full IR
              TraceProcessor.Process
              PerfettoWriter::WriteToString(
                  trace, &data_, compressed_output=true)
                // inner gzip level 1 → data_         # ★ compress #1
          return bytes, application/octet-stream
        ToolResult(data, content_type, content_encoding=None)
      respond(data, content_type, content_encoding=None)
        body = gzip.compress(body)                    # ★ compress #2
        Content-Encoding: gzip
```

**Fix locus candidates (analysis):**

| Locus | Change idea | Risk |
|-------|-------------|------|
| `tool_data.py` | Set encoding from tool metadata | central |
| `raw_to_tool_data` | Return encoding for perfetto branch | local |
| `respond.py` | Detect gzip magic / skip if already compressed | heuristic risk |
| Processor | Write uncompressed; let respond gzip once | CPU trade |

Preferred design (hypothesized): **explicit encoding from convert**, not magic-byte sniffing.

---

## 19. Appendix B — Full host IR field checklist

From `xprof_trace.h` + loader:

| Keep today | Notes |
|------------|-------|
| TPU Steps / Modules / Ops / TraceMe | Allowed lines only |
| All Megascale Trace events | No line filter analogous to TPU |
| Interned string args (filtered stats) | SkipStat list applied |
| Event.name as std::string | **Intern candidate (MP-3)** |
| Flow / run_id from processor | Needed for Perfetto flows |
| Counters rx/tx/bw/inflight | Useful for Megascale analysis |

**Subset export ideas (MP-4, hypothesized):**

- Time range query params (mirror trace_viewer start/end).  
- Device id filter (`/device:TPU:k` only).  
- Drop XLA Ops line when only Megascale collective view needed.  
- Counter-only mode for bandwidth debugging.

---

## 20. Appendix C — N× HLO load sequence (unfused reshapes)

```text
get_top_hlo_ops(session)                    # 1× full op_profile (cold)
for candidate in formatting_ops:            # up to limit=50
  if module guessed:
    get_hlo_neighborhood(module, instr)     # 1× graph_viewer FULL text
  else:
    list_hlo_modules()                      # 1×
    for mod in all_modules:
      get_hlo_neighborhood(mod, instr)      # up to M × full text
      break if found
```

Binary alternative (URS-1/2 design sketch):

```text
index = SessionHloIndex.lazy(session)
for candidate in formatting_ops:
  mod = index.resolve_module(candidate)
  g = index.graph(mod)                      # one proto parse per mod
  neighborhood = g.bfs(instr, radius=2)     # no text dump
```

---

## 21. Appendix D — Overview full JSON for scalars

```text
OverviewPageProcessor
  OpStats combined  ← ALL hosts (with cache)
  OverviewPage page ← many sections / tables
  optional InferenceStats multi-xspace walk
  overview_page_json = OverviewPageToJson(page)   # FULL UI

get_overview_tool
  overview_data = json.loads(full_json)           # FULL parse
  for section in overview_data:
    all_p_dict.update(section["p"])
  keep keys in PERFORMANCE_SUMMARY_KEYS
       or startswith stat_|sc_|run_
       or RUN_ENVIRONMENT_KEYS
  return {performance_summary, run_environment}   # SLIM
```

**GOV-1 design sketch:** convert option `view=summary` emits only the `p` dict keys CLI needs (or a dedicated `OverviewSummary` proto), skipping DataTable rows entirely.

---

## 22. Appendix E — Severity rubric applied to this family

| Severity | Applied when… | Examples here |
|----------|---------------|---------------|
| **H** | Multi-MiB class waste, double compress large binary, all-module HLO, N× full module text, full Overview for scalars | MP-1, MP-2, P2 IR, URS-1, LMC-1, GOV-1 |
| **M** | Real cost but secondary vs H, or single cold multi-host sequential path | MS-1, MP-6, GOV-3 |
| **L** | FE retention, indent, presence short-circuit | FE-MS-1, DET-6, MS-5 |

Never use "Critical" — portfolio uses H/M/L only (synthesis rule).

---

## 23. Appendix F — Confidence rubric applied

| Confidence | Applied when… |
|------------|---------------|
| **observed** | Direct code path with line cites implies the memory behavior |
| **hypothesized** | Peak magnitude, production frequency, or best API shape needs measurement |

Examples:

- Double gzip: **observed** (both compress sites exist).  
- Transfer list savings in Chrome: **observed** clone path; magnitude **hypothesized** until measured.  
- Lazy DCN host-first win: **hypothesized** (depends on UI host selection frequency).

---

## 24. Appendix G — MCP / CLI fan-out scenario

Agents often chain:

```text
get_overview → get_top_hlo_ops → detect_* → get_hlo_neighborhood
```

Without single-flight and shared HLO index:

| Step | Convert paid |
|------|----------------|
| overview | multi-host OpStats + full JSON |
| top ops | full hlo_op_profile |
| layout mismatch | all-module HLO again |
| unfused | N× graph_viewer text |
| convert/reduce | all-module HLO again |

DET-1 + DET-2 + DET-5 + GOV-1 collapse this chain. This is why synthesis ranks **projection modes** and **HLO amortization** as global top themes, not "megascale-only" bugs.

---

## 25. Appendix H — Comparison: megascale_stats vs trace_viewer@

| Dimension | megascale_stats table | megascale_perfetto | trace_viewer@ (doc 01) |
|-----------|----------------------|--------------------|-------------------------|
| Host scope | ALL_HOSTS cold generate; read by host | Single host required | Multi-host CSV (cap 10 UI) |
| Intermediate | dcn_collective_stats.pb | None (ephemeral IR) | LevelDB SSTABLEs |
| Viewport | N/A (summary table) | Full host time range | start/end/resolution |
| Wire | JSON DataTable | gzip-of-gzip binary | JSON or zstd PB |
| Browser | Charts table | Perfetto iframe | V2 WASM / Catapult |
| Dominant peak | Cold multi-host XSpace | Full IR + double gzip + clone | First-open multi-host + browser SoA |

Perfetto should borrow **viewport + intermediate** ideas from Trace Viewer; table path should borrow **lazy host** ideas from OpStats combine discussions (doc 02/04).

---

## 26. Appendix I — File size / complexity map (repo)

| File | ~Lines | Role |
|------|--------|------|
| `detect_layout_mismatch_copies_tool.py` | ~1100 | Heaviest detector logic |
| `detect_unnecessary_convert_reduce_tool.py` | ~700 | Tracer + rules matrix |
| `trace_processor.cc` | ~840 | Perfetto IR transforms |
| `perfetto_writer.cc` | ~420 | Proto + gzip |
| `xspace_loader.cc` | ~360 | XSpace → IR |
| `detect_unfused_reshapes_tool.py` | ~190 | Thin; cost in hlo_tools |
| `get_overview_tool.py` | ~160 | Thin; cost in overview processor |
| `megascale_stats_processor.cc` | ~94 | Branch perfetto vs table |
| `megascale_perfetto.ts` | ~200 | Fetch + postMessage |
| `megascale_stats.ts` | ~160 | Panel + DataTable |

Complexity in detectors is **logic**; complexity in memory is often **dependencies** (graph_viewer, debug_info, respond).

---

## 27. Appendix J — Opportunity ID index

| ID | One-line |
|----|----------|
| MS-1 | Lazy cold DCN host-first |
| MS-2 | Free FE table on close |
| MS-3 | Shared skip re-gzip |
| MS-4 | Paginate rendezvous rows |
| MS-5 | Skip redundant presence XSpace walk |
| MS-6 | Slimmer combiner lifetime |
| MP-1 | Stop Perfetto double gzip |
| MP-2 | Stream / stage lifetimes |
| MP-3 | Intern Event.name |
| MP-4 | Viewport / device subset |
| MP-5 | postMessage transfer list |
| MP-6 | Plumb group_tiny_events |
| MP-7 | Drop more lines/args early |
| MP-8 | Disk intermediate for re-open |
| MP-9 | Explicit size budget errors |
| MP-10 | Share encoding fix with PB traces |
| MP-11 | Free XSpace before perfetto Trace |
| FE-MS-1..8 | Frontend retention / clone / abort |
| LMC-1..7 | Layout mismatch detector |
| URS-1..6 | Unfused reshapes detector |
| UCR-1..5 | Convert/reduce detector |
| GOV-1..5 | Overview CLI projection |
| DET-1..8 | Cross-detector HLO/top-ops |

---

## 28. Summary

The megascale + detector family contributes four **H**-severity, **observed** themes that synthesis must keep ranked high:

1. **Double gzip** of precompressed Perfetto (`perfetto_writer.cc:405-417` + `respond.py:109-111` + `tool_data.py:134`).  
2. **Full host IR** materialization without viewport (`megascale_stats_processor.cc:69-80`, `xprof_trace.h`, `xspace_loader.cc`).  
3. **N× HLO / graph text loads** in unfused reshape detection (`hlo_tools.py:266-275`, `detect_unfused_reshapes_tool.py:75-120`).  
4. **Full OverviewPage JSON for ~30 scalars** (`overview_page_processor.cc:88-106`, `get_overview_tool.py:71-149`).

Fix order favors **encoding hygiene (S)** and **projection / lazy HLO (M)** before large Perfetto architecture rewrites (L). No product code in this campaign — measurement in synthesis Phase 0 should instrument the four themes above first for this family.

---

## 29. Document maintenance

| Item | Value |
|------|-------|
| Last analysis date | 2026-07-13 |
| Mode | analysis only |
| Depth target | ≥800 lines of code-cited analysis |
| Related deepened notes | 01 ~1600, 02 ~1600, 03 ~1200 lines |
| Severity vocabulary | H \| M \| L only |
| Confidence vocabulary | observed \| hypothesized only |

When code moves, re-verify the four star cites: double gzip, full host IR, N× HLO, overview full JSON for scalars.
