# Megascale + Remaining CLI Detectors — Memory Optimization Opportunity Analysis

**Date:** 2026-07-13  
**Mode:** analysis only (no product code)  
**Tools:** megascale_stats, megascale_perfetto frontend, detect_layout_mismatch_copies_tool, detect_unfused_reshapes_tool, detect_unnecessary_convert_reduce_tool, get_overview_tool

---

## Registry / wiring

| Concern | Path |
|---------|------|
| Tool registry | `plugin/xprof/profile_plugin/tools/registry.py` — `megascale_stats` ∈ XPLANE tools; also `XPLANE_TOOLS_ALL_HOSTS_SUPPORTED` |
| Overview warm-up | `DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')` — overview always multi-host cold-start cost |
| Options | `plugin/xprof/profile_plugin/tools/options/base.py` — `perfetto` bool only; **`group_tiny_events` not plumbed** |
| Python convert | `plugin/xprof/convert/raw_to_tool_data.py` (`megascale_stats` → `host_name` + `perfetto`) |
| C++ processor | `xprof/convert/megascale_stats_processor.cc` (`REGISTER_PROFILE_PROCESSOR("megascale_stats", …)`) |
| Perfetto IR | `xprof/convert/megascale_perfetto/` (`xspace_loader`, `trace_processor`, `perfetto_writer`, `xprof_trace.h`) |
| DCN stats | `xprof/convert/xplane_to_dcn_collective_stats.cc`, `xprof/convert/process_megascale_dcn.cc` |
| FE stats | `frontend/app/components/megascale_stats/` |
| FE Perfetto | `frontend/app/components/megascale_perfetto/` |
| CLI tools | `plugin/xprof/cli/tools/*.py` |
| HLO helpers | `plugin/xprof/cli/internal/oss/hlo_tools.py` |
| CLI cache | `plugin/xprof/cli/internal/decorators.py` (SQLite, 86400s) |
| HTTP respond | `plugin/xprof/profile_plugin/http/respond.py` — default gzip when `content_encoding is None` |

---

## 1. megascale_stats (table / DCN Slack path)

### End-to-end data path

```text
UI host=ALL_HOSTS → expand panel → dataService.getData(session, megascale_stats, ALL_HOSTS)
  → HostSelector: all XPlanes when ALL_HOSTS
  → MegascaleStatsProcessor (perfetto=false)
       GetDcnSlackAnalysisByHostName(session, hostname)
         ConvertMultiXSpaceToDcnCollectiveStats  [cold: scan every host]
           for each host:
             Arena + GetXSpace(host)
             ConvertXSpaceToDcnSlackAnalysis
             WriteBinaryProto <host>.dcn_collective_stats.pb
             combiner.Combine
           WriteBinaryProto ALL_HOSTS.dcn_collective_stats.pb
         warm: ReadBinaryProto hostname → DcnSlackAnalysis
       GenerateMegaScaleJson → DataTable JSON array
  → respond: gzip(JSON)
  → FE: SimpleDataTable + google.visualization.DataTable (Dashboard.parseData)
```

Single-host selection is required for Perfetto; table path is explicitly **all-hosts aggregate** in the FE template (`megascale_stats.ng.html`).

### Materialization

| # | Stage | Representation | Peak notes | Confidence |
|---|-------|----------------|------------|------------|
| S1 | Cold DCN convert | Sequential per-host Arena XSpace | Peak ≈ max(X_i) + analysis, not Σ X_i (sequential arenas) | observed (`xplane_to_dcn_collective_stats.cc`) |
| S2 | Cold combine | `DcnSlackAnalysisCombiner` + per-host pb writes | N host files + ALL_HOSTS | observed |
| S3 | Warm read | One `DcnSlackAnalysis` protobuf | Small vs XSpace; scales with #collectives | observed |
| S4 | JSON | `GetMegaScaleDataTable` → `ToJson` string | One row × ~16 columns per rendezvous | observed (`process_megascale_dcn.cc`) |
| S5 | HTTP | raw JSON + gzip | Double buffer (shared path) | observed (`respond.py`) |
| S6 | FE | JS object + Google Charts DataTable | Held after panel close (`onPanelClosed` only flips flag) | observed (`megascale_stats.ts`) |

### Severity assessment

Table path is **moderate** relative to Perfetto/trace. Cold multi-host convert is the real cost; warm path is light.

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| MS-1 | Lazier cold DCN: only convert requested host first; defer ALL_HOSTS combine | **M** | medium |
| MS-2 | Free FE `dataTable` / `dataInfo` on panel close / host change | **L** | high |
| MS-3 | Skip re-gzip if body already compressed (shared) | **M** | high (shared with 02) |
| MS-4 | Bound / paginate table rows for pathological #rendezvous | **L** | hypothesized |

**Top:** MS-1 (cold multi-host), then shared gzip policy.

---

## 2. megascale_perfetto (Perfetto export path)

### End-to-end data path

```text
UI /megascale_perfetto/:sessionId?host=H[&group_tiny_events=…]
  → fetch /data?tag=megascale_stats&perfetto=true&host=H
  → MegascaleStatsProcessor:
       Arena + GetXSpaceByName(H)          // full host XPlane
       XSpaceLoader::Load → XprofTrace     // full event copy (selected lines)
       TraceProcessor::Process             // sort, run_id, group tiny, flows, counters
       PerfettoWriter::WriteToString(…, compressed_output=true)
         Arena perfetto::Trace
         GzipOutputStream level=1 → std::string data_
  → content_type application/octet-stream
  → tool_data content_encoding=None
  → respond: gzip.compress(already-gzip body)   // DOUBLE GZIP
  → FE: response.blob() → arrayBuffer()
       postMessage({perfetto: {buffer}}, ui.perfetto.dev)  // no transfer list → clone
```

**Note:** There is no separate `megascale_perfetto` convert tool — Perfetto reuses `megascale_stats` with `perfetto=true` (`megascale_perfetto.ts` `buildPerfettoDataURL`).

### Materialization by stage

| # | Stage | Representation | Peak notes | Confidence |
|---|-------|----------------|------------|------------|
| P1 | GetXSpace | Arena full host `XSpace` | Dominates for large multi-chip hosts | observed |
| P2 | XSpaceLoader | `XprofTrace`: TPU lines (Steps/Modules/Ops/TraceMe) + all Megascale Trace events | **Full event materialization**; `Event.name` is `std::string` (not interned) | observed (`xprof_trace.h`, `xspace_loader.cc`) |
| P3 | Args | StringTable interns keys/string values; numeric args as variants | Better than raw XStats; still O(events × args) | observed |
| P4 | Megascale buffer | `flat_hash_map<device, list<Track>>` then move to fragments | Temporary list + map overhead | observed |
| P5 | TraceProcessor | In-place sort; tiny-event collapse; flow maps; counter deltas | Grouping reduces events (<1ns) when enabled | observed |
| P6 | Perfetto proto | Full `perfetto::protos::Trace` on Arena | ≈ expanded IR size | observed |
| P7 | Inner gzip | `data_` string | XSpace + IR + Trace + gzip coexist until function returns | observed |
| P8 | Outer gzip | `respond.py` | **Double compress** when encoding unset | observed |
| P9 | Browser | blob + ArrayBuffer (+ postMessage clone) + Perfetto iframe heap | Dominant client peak; empty buffer → "Trace data too big" | observed (`megascale_perfetto.ts`) |

### Option plumbing gap (memory-relevant)

| Concern | Detail |
|---------|--------|
| FE sets `group_tiny_events` query param | `megascale_perfetto.ts` |
| `build_common_params` | does **not** pass `group_tiny_events` |
| `raw_to_tool_data.py` options | only `host_name`, `perfetto` |
| C++ default | `group_tiny_events=true` always |

So FE cannot disable grouping; default true is good for size, but the URL contract is dead code today.

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| MP-1 | Set `content_encoding=gzip` (or skip outer gzip) for pre-gzipped Perfetto | **Critical** | high |
| MP-2 | Stream / chunk Perfetto write; avoid holding XSpace+IR+Trace+gzip simultaneously | **H** | medium |
| MP-3 | Intern `Event.name` (StringId) like args | **H** | high |
| MP-4 | Viewport / time-range / device subset export (not full host) | **H** | medium |
| MP-5 | FE: transfer ArrayBuffer in `postMessage` transfer list; drop blob after `arrayBuffer` | **H** | high |
| MP-6 | Plumb `group_tiny_events` end-to-end; optional aggressive thresholds | **M** | high |
| MP-7 | Drop non-essential lines/args earlier (already skips some StatTypes) | **M** | medium |
| MP-8 | Disk/LevelDB intermediate like `trace_viewer@` for re-open | **M** | hypothesized |
| MP-9 | Server size budget → explicit error vs empty body | **M** | medium |

**Top:** MP-1, MP-5, MP-3, MP-2/MP-4.

---

## 3. Frontend retention (both megascale UIs)

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| FE-MS-1 | `onPanelClosed` does not clear `dataInfo` / Dashboard `dataTable` | **L** | high |
| FE-MS-2 | Perfetto path keeps no explicit ref after postMessage, but clone without transfer keeps peak high | **H** | high |
| FE-MS-3 | Cross-origin Perfetto iframe — no control of viewer heap after handoff | **M** | observed constraint |
| FE-MS-4 | Host switch does not abort in-flight fetch | **L** | medium |

---

## 4. CLI: detect_layout_mismatch_copies_tool

**Path:** `plugin/xprof/cli/tools/detect_layout_mismatch_copies_tool.py`

### Data path

```text
detect_layout_mismatch_copies(session)
  → hlo_tools._fetch_debug_info(session)   // ALL modules HloProto (1P helper; not in OSS hlo_tools.py)
  → get_top_hlo_ops(session, limit=100)    // full hlo_op_profile convert + flatten
  → for each module:
       build instr_by_id, users_by_id, callers, roots   // O(N) maps  [PR #2914/#2923]
       for EVERY opcode=="copy":
         BFS upstream max_depth=5
         BFS downstream max_depth=5
         if both sides compute → emit bottleneck
  → json.dumps(indent=2)
```

**Important OSS note:** `_fetch_debug_info` is referenced and mocked in tests (`hlo_tools.hlo_proto_dump_pb2.DebugInfoCollection`) but **not implemented** in `plugin/xprof/cli/internal/oss/hlo_tools.py`. Production 1P path loads full multi-module HLO; OSS would need an equivalent full-proto load (or fails).

### Materialization

| # | What | Risk |
|---|------|------|
| L1 | All modules' `HloModule` protos resident | **Critical** for large multi-module sessions |
| L2 | Connectivity maps per module | O(N) memory; already optimized from O(N²) CPU |
| L3 | BFS per **every** copy (not top-K filtered) | CPU heavy; visited sets per search |
| L4 | Full `hlo_op_profile` via `get_top_hlo_ops` | Extra full-profile convert (cached 86400s) |
| L5 | Result list of all sandwiched copies | Usually small vs L1 |

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| LMC-1 | Do not load all modules at once — stream module-by-module from `*.hlo_proto.pb` | **Critical** | high |
| LMC-2 | Restrict candidates to top-K copies by bytes/time **before** BFS | **H** | high |
| LMC-3 | Share one HLO index / debug_info cache across detector tools | **H** | medium |
| LMC-4 | Early filter: only copies with layout mismatch / non-optimal minor dim | **M** | medium |
| LMC-5 | Avoid full op_profile when only enriching metrics for matches | **M** | medium |
| LMC-6 | OSS implement `_fetch_debug_info` via on-disk protos (lazy) not dump-all | **H** | high (gap) |

**Top:** LMC-1, LMC-2, LMC-3.

---

## 5. CLI: detect_unfused_reshapes_tool

**Path:** `plugin/xprof/cli/tools/detect_unfused_reshapes_tool.py`

### Data path

```text
detect_unfused_reshapes(session, limit=50)
  → get_top_hlo_ops → top_by_bytes_accessed
  → filter formatting categories (reshape/transpose/copy/…)
  → for each candidate:
       guess module from name path
       get_hlo_neighborhood(instr, radius=2, module)
         → graph_viewer long/short_txt for FULL module text
         → parse all lines into adjacency
         → BFS radius
       on miss: list_hlo_modules + try EVERY module neighborhood
  → json.dumps findings
```

### Materialization / amplification

| # | What | Risk |
|---|------|------|
| U1 | Full op_profile for top bytes | **H** first hit; then CLI SQLite cache |
| U2 | **Per-candidate full module HLO text** via `graph_viewer` | **Critical** N× full text convert |
| U3 | Module scan fallback: candidates × modules × full text | **Critical** worst case |
| U4 | `list_hlo_modules` / neighborhood cached 86400s | Mitigates repeat calls only |
| U5 | Text graph rebuild each neighborhood call (even cached return is full string) | **H** |

This is the classic **N+1 convert** anti-pattern (similar spirit to `get_peak_allocations_tool` in doc 03).

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| URS-1 | Binary neighborhood from HloProto (no full text dump) | **Critical** | high |
| URS-2 | One module load + multi-instruction queries | **Critical** | high |
| URS-3 | Cap module fallback search; require module in op name | **H** | high |
| URS-4 | Reuse layout-mismatch style in-memory index (shared) | **H** | medium |
| URS-5 | Lower default limit / only top formatting ops | **M** | high |

**Top:** URS-1, URS-2.

---

## 6. CLI: detect_unnecessary_convert_reduce_tool

**Path:** `plugin/xprof/cli/tools/detect_unnecessary_convert_reduce_tool.py`

### Data path

```text
detect_unnecessary_convert_reduce(session, limit=50)
  → get_top_hlo_ops (top_by_time + top_by_bytes)
  → _fetch_debug_info → ALL modules
  → for each top op:
       resolve module + instruction
       lazy _HloModuleTracer (indices: instructions, consumers, fusion_callers, …)
       find reduce / fusion-inner reduce
       trace_upcast (DFS) + trace_downcast (DFS)
  → dedupe → json.dumps
```

Better than layout-mismatch: **candidate-driven** from top ops, not full copy scan. Still pays **all-modules HLO load**.

### Materialization

| # | What | Risk |
|---|------|------|
| C1 | All-module HLO protos | **Critical** same as LMC |
| C2 | Tracer indices per touched module | O(N) per module; reused within run | observed |
| C3 | DFS upcast/downcast with visited (id, tuple_index) | Bounded by allowed opcodes | observed |
| C4 | Top ops + full profile convert | **H** shared | observed |
| C5 | Timing logs for fetch vs core | Useful for production sizing | observed |

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| UCR-1 | Lazy per-module HLO load only for modules in top-ops set | **Critical** | high |
| UCR-2 | Shared HLO session index with other detectors | **H** | medium |
| UCR-3 | Optional skip of top_ops dual lists (time+bytes doubles work) | **M** | medium |
| UCR-4 | Bound DFS visited size / depth hard limits | **L** | medium |

**Top:** UCR-1, UCR-2.

---

## 7. CLI: get_overview_tool

**Path:** `plugin/xprof/cli/tools/get_overview_tool.py`  
**Not covered in depth elsewhere** (overview appears only as warm-up / ALL_HOSTS in doc 02).

### Data path

```text
get_overview(session)  [@cached 86400s]
  → client.fetch(overview_page.json)
       OverviewPageProcessor:
         ConvertMultiXSpaceToCombinedOpStatsWithCache  // ALL hosts OpStats
         ConvertOpStatsToOverviewPage
         [inference] ConvertMultiXSpaceToInferenceStats again
         OverviewPageToJson  // full UI JSON sections
  → json.loads full overview
  → extract PERFORMANCE_SUMMARY_KEYS + RUN_ENVIRONMENT_KEYS (+ stat_/sc_/run_ prefixes)
  → discard rest → small summary JSON
```

### Materialization

| # | What | Waste |
|---|------|-------|
| O1 | Multi-host combined OpStats | Dominant; shared with UI overview / warm-up |
| O2 | Full OverviewPage JSON for CLI that needs ~30 scalars | **Dead weight on wire + parse** |
| O3 | Inference path re-walks multi-XSpace | Extra peak for non-training |
| O4 | CLI holds full parsed list of sections then drops | Transient Python heap |
| O5 | SQLite caches slim output (good) | First call still full cost |

### Opportunities

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| GOV-1 | `summary_only` / `cli` convert mode emitting scalars only | **Critical** | high |
| GOV-2 | Reuse warm OpStats disk cache without re-serializing full OverviewPage UI tables | **H** | high |
| GOV-3 | Skip inference latency block for CLI summary | **M** | medium |
| GOV-4 | Shared policy with GMP-1 / FAM-1 (view modes) | **H** | high |

**Top:** GOV-1 (mirror memory-family summary_only).

---

## 8. Cross-cutting: detectors + HLO + top ops

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| DET-1 | Shared `SessionHloIndex`: lazy module map, single connectivity build | **Critical** | high |
| DET-2 | Detectors must not force full `hlo_op_profile` UI proto — metrics-only extract | **H** | medium |
| DET-3 | Prefer on-disk `*.hlo_proto.pb` (`generate_hlo_protos`) over opaque `_fetch_debug_info` dump-all | **H** | high |
| DET-4 | Never fetch full graph_viewer text for structural graph queries | **Critical** | high (URS) |
| DET-5 | Single-flight convert when MCP tools fan-out in parallel | **H** | hypothesized |
| DET-6 | JSON `indent=2` for large inefficient_ops lists | **L** | high |

---

## Ranked priority (this family)

| Rank | ID | Description | Severity | Confidence | Effort |
|------|-----|-------------|----------|------------|--------|
| 1 | MP-1 | Stop double-gzip of Perfetto payload | **Critical** | high | S |
| 2 | URS-1/2 | Neighborhood without full HLO text × N | **Critical** | high | M–L |
| 3 | LMC-1 / UCR-1 | Lazy / streaming per-module HLO | **Critical** | high | M |
| 4 | GOV-1 | Overview summary_only for CLI | **Critical** | high | M |
| 5 | MP-5 / MP-3 | FE transfer + intern event names | **H** | high | S–M |
| 6 | DET-1 | Shared HLO index across detectors | **H** | medium | L |
| 7 | LMC-2 | Top-K copy candidates only | **H** | high | S |
| 8 | MP-2 / MP-4 | Subset / streaming Perfetto export | **H** | medium | L |
| 9 | MS-1 | Lazier DCN ALL_HOSTS cold path | **M** | medium | M |
| 10 | MP-6 | Plumb group_tiny_events | **M** | high | S |

---

## Severity × confidence matrix (summary)

| Tool | Dominant risk | Severity | Confidence |
|------|---------------|----------|------------|
| megascale_stats (table) | Cold multi-host DCN convert | **M** | high |
| megascale_perfetto | Full XSpace→IR→Trace + double gzip + FE clone | **Critical** | high |
| detect_layout_mismatch | All-module HLO + all copies BFS | **Critical** | high |
| detect_unfused_reshapes | N× full graph_viewer text | **Critical** | high |
| detect_unnecessary_convert_reduce | All-module HLO (candidate-limited logic) | **H–Critical** | high |
| get_overview_tool | Full multi-host OpStats/Overview for scalars | **Critical** (first hit) | high |

---

## Paths cited

### Megascale convert / FE
- `frontend/app/components/megascale_stats/megascale_stats.ts`
- `frontend/app/components/megascale_stats/megascale_stats.ng.html`
- `frontend/app/components/megascale_perfetto/megascale_perfetto.ts`
- `frontend/app/components/megascale_perfetto/megascale_perfetto.ng.html`
- `frontend/app/components/chart/dashboard/dashboard.ts`
- `plugin/xprof/convert/raw_to_tool_data.py`
- `plugin/xprof/profile_plugin/tools/options/base.py`
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/services/tool_data.py`
- `plugin/xprof/profile_plugin/services/hosts.py`
- `plugin/xprof/profile_plugin/http/respond.py`
- `xprof/convert/megascale_stats_processor.cc`
- `xprof/convert/process_megascale_dcn.cc`
- `xprof/convert/xplane_to_dcn_collective_stats.cc`
- `xprof/convert/xplane_to_tools_data.cc`
- `xprof/convert/megascale_perfetto/xprof_trace.h`
- `xprof/convert/megascale_perfetto/xspace_loader.cc`
- `xprof/convert/megascale_perfetto/trace_processor.cc`
- `xprof/convert/megascale_perfetto/perfetto_writer.cc`

### CLI detectors / overview
- `plugin/xprof/cli/tools/detect_layout_mismatch_copies_tool.py`
- `plugin/xprof/cli/tools/detect_unfused_reshapes_tool.py`
- `plugin/xprof/cli/tools/detect_unnecessary_convert_reduce_tool.py`
- `plugin/xprof/cli/tools/get_overview_tool.py`
- `plugin/xprof/cli/tools/get_top_hlo_ops_tool.py`
- `plugin/xprof/cli/internal/oss/hlo_tools.py`
- `plugin/xprof/cli/internal/decorators.py`
- `xprof/convert/overview_page_processor.cc`
- `xprof/convert/overview_page_processor.h`

### Docs cross-ref
- `docs/superpowers/specs/memory-optimization/01-trace-viewer-path.md` (gzip / FE transfer patterns)
- `docs/superpowers/specs/memory-optimization/02-convert-serve-cache-path.md` (ALL_HOSTS, overview warm-up)
- `docs/superpowers/specs/memory-optimization/03-memory-tools-family.md` (summary_only / N+1 convert patterns)

---

## Suggested implementation order (no code in this doc)

1. **MP-1** content_encoding for precompressed Perfetto (and share with zstd/pb traces).  
2. **URS-1/2** + **DET-4** kill full-text neighborhood for detectors.  
3. **LMC-1 / UCR-1 / DET-1 / DET-3** lazy modular HLO index.  
4. **GOV-1** overview summary mode for CLI / MCP.  
5. **MP-3 / MP-5** Perfetto IR + FE transfer polish.  
6. **LMC-2** top-K copies.  
7. **MS-1 / MP-4** scale multi-host / full-host exports.
