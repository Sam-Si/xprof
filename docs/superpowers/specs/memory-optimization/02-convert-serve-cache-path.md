# Memory Optimization: Convert → Serve → Cache Path

**Date:** 2026-07-13  
**Status:** Research / design (no product code)  
**Scope:** Shared pipeline for all XPlane-backed UI tools — HTTP edge through convert to disk caches  
**Audience:** Engineers prioritizing RAM-sensitive work; companion to `00-synthesis`, `01-trace-viewer`, `03`–`06` tool family notes  
**Mode:** Analysis only — confidence labels use **observed** | **hypothesized** only

---

## 0. Why this path matters

Every XPlane-backed UI tool (and default warm-up) shares one Python orchestration spine:

```text
HTTP /data | /data_csv | /generate_cache
  → data_api / cache_api
  → ToolDataService (or cache task)
  → SessionResolver + HostSelector
  → result_cache_policy + build_tool_params
  → raw_to_tool_data → pywrap C++ (SessionSnapshot / processors)
  → ToolResult
  → respond() (JSON encode + default gzip)
```

Peak resident memory is usually **not** dominated by small Python objects (`ToolRequest`, path strings). It is dominated by:

1. **C++ convert peak** — full XSpace arenas, `OpStats` vectors, TraceEventsContainer, JSON `std::string`s.  
2. **HTTP double buffers** — raw body + `gzip.compress(body)` coexisting.  
3. **Concurrency** — foreground `/data` unbounded vs cache pool `max_workers=1`.  
4. **ALL_HOSTS amplification** — N hosts loaded/combined before any projection.  
5. **Disk caches without quota** — `*.op_stats_v2.pb`, SSTABLEs, smart_suggestion blobs.

Cross-cutting fixes here multiply across **all** tools in `XPLANE_TOOLS` (`registry.py`), not just one viewer.

---

## 1. End-to-end pipeline (named components)

### 1.1 Stage diagram (ownership + buffers)

```text
Browser / CLI
    │  GET /data?run=&tag=&host=  |  GET /data_csv  |  POST /generate_cache
    ▼
┌──────────────────────────────────────────────────────────────────────┐
│ data_api.py (DataApiMixin)         cache_api.py (CacheApiMixin)      │
│  data_route / data_csv_route        generate_cache_route             │
│  data_impl → ToolDataService        _generate_cache_impl → pool      │
└─────────────────┬───────────────────────────────┬────────────────────┘
                  │                               │
                  ▼                               ▼
         ToolDataService.get_tool_data    _generate_cache_task
                  │                               │
    ┌─────────────┼─────────────┐                 │
    ▼             ▼             ▼                 ▼
 SessionResolver  result_cache  HostSelector   all asset_paths
 resolve_run_dir  _policy       .select()      hosts_from_xplane_*
    │             use_saved     asset_paths[]  write_cache_version
    ▼             ▼             ▼                 │
 build_tool_params  →  ConvertPort.xspace_to_tool_data
                       (plugin.py _ConvertAdapter → raw_to_tool_data)
                                  │
                                  ▼
                    C++ SessionSnapshot (repository.cc)
                    processors (base_op_stats_processor, …)
                                  │
                                  ▼
                    ToolResult(data, content_type, content_encoding=None)
                                  │
                                  ▼
                         respond()  [http/respond.py]
                         optional json.dumps → utf-8 → gzip.compress
```

### 1.2 Buffer ownership table

| Stage | Module / symbol | Peak buffers | Lifetime | Confidence |
|-------|-----------------|--------------|----------|------------|
| Query parse | `parse_request.tool_request_from_args` | `ToolRequest` + `raw_args` dict | Request | observed |
| Session resolve | `SessionResolver.resolve_run_dir` | path strings; `_run_to_profile_run_dir` map entry | Process (map) / call | observed |
| Cache policy | `result_cache_policy.should_use_saved_result` | version string from `cache_version.txt` | Call | observed |
| Host selection | `HostSelector.select` | N path objects + host names | Call | observed |
| Convert Python | `raw_to_tool_data.xspace_to_tool_data` | options dict; for legacy trace: Trace proto + joined JSON | Convert | observed |
| Convert C++ | pywrap / `SessionSnapshot::GetXSpace` | Arena XSpace(s) + intermediates + output | Convert | observed |
| OpStats path | `base_op_stats_processor` / `multi_xplanes_to_op_stats` | vector of all host OpStats + combined | Convert | observed |
| Result | `ToolResult` | payload bytes/str; `content_encoding=None` always from service | Request | observed |
| HTTP | `respond` | body + gzip(body) coexist when encoding default | Response build | observed |
| CSV | `data_csv_route` + `csv_writer.json_to_csv_string` | JSON decode tree + CSV string + gzip | Request | observed |
| Cache gen pool | `plugin.py` `ThreadPoolExecutor(max_workers=1)` | Full convert peak for largest tool in list | Worker thread | observed |

### 1.3 Inventory strings (must-match index)

| Name | Primary path |
|------|----------------|
| `ToolDataService` / `tool_data_service` | `plugin/xprof/profile_plugin/services/tool_data.py` |
| `HostSelector` / `hosts_selector` | `plugin/xprof/profile_plugin/services/hosts.py` |
| `SessionResolver` / `session_resolver` | `plugin/xprof/profile_plugin/services/sessions.py` |
| `raw_to_tool_data` | `plugin/xprof/convert/raw_to_tool_data.py` |
| `respond` | `plugin/xprof/profile_plugin/http/respond.py` |
| `tools_cache` | `plugin/xprof/profile_plugin/cache/tools_cache.py` (`ToolsCache`) |
| `result_cache_policy` | `plugin/xprof/profile_plugin/cache/result_cache_policy.py` |
| `generate_cache` | `plugin/xprof/profile_plugin/api/cache_api.py` (`generate_cache_route`) |
| `data_api` | `plugin/xprof/profile_plugin/api/data_api.py` |

Additional closely coupled modules: `runs.py` (`RunDiscovery`), `plugin.py` (wiring + cache executor), `parse_request.py`, `csv_writer.py`, `repository.cc`, `base_op_stats_processor.cc`.

---

## 2. Component deep dives (code-grounded)

### 2.1 `data_api` — HTTP entry for tool payloads

**File:** `plugin/xprof/profile_plugin/api/data_api.py`  
**Class:** `DataApiMixin` (mixed into `ProfilePlugin`).

| Handler | Route constant | Behavior |
|---------|----------------|----------|
| `data_route` | `DATA_ROUTE` (`/data`) | `data_impl` → `respond(data, content_type, content_encoding)` |
| `data_csv_route` | `DATA_CSV_ROUTE` (`/data_csv`) | Same convert, then `json_to_csv_string`, then CSV + attachment headers |
| `hlo_module_list_route` | `HLO_MODULE_LIST_ROUTE` | Lists `*.hlo_proto.pb`; may call `xspace_to_tool_names` to materialize HLO |

**`data_impl` flow (observed):**

1. `tool_request_from_args(request.args)` → `ToolRequest` (`tag` → tool name, `host`/`hosts`, `use_saved_result` default **True**).  
2. `self._tool_data.get_tool_data(...)` with `session_path`, `run_path`, `logdir`, `run_dir_cache=self._run_to_profile_run_dir`, `cache_lock`.  
3. Returns `(result.data, result.content_type, result.content_encoding)` — encoding is **always** `None` from `ToolDataService` today.

**Memory notes for `data_api`:**

| Issue | Severity | Confidence |
|-------|----------|------------|
| No backpressure / semaphore around convert | **H** | hypothesized |
| 404 path still cheap (`data is None` → plain text) | L | observed |
| Exception path re-stringifies errors (tiny) | L | observed |
| `/data_csv` forces full convert + full JSON parse before CSV | **M** | observed |
| `hlo_module_list_impl` may convert all XPlanes just to list module names | **M** | observed |

**`/data_csv` double materialization (observed):**

```text
data_impl → (often bytes JSON from C++)
  → json_to_csv_string: decode utf-8 if bytes → json.loads → walk cols/rows → StringIO CSV
  → respond(csv_data, 'text/csv', content_encoding=None) → gzip.compress again
```

CSV is useful for export; for large tables (`kernel_stats`, `framework_op_stats`) peak ≈ JSON tree + CSV + gzip. A streaming or C++-native CSV path would drop the Python object graph (**M**, hypothesized ROI; **observed** double parse).

### 2.2 `parse_request` — edge adapter

**File:** `plugin/xprof/profile_plugin/http/parse_request.py`

- Builds frozen `ToolRequest` from Werkzeug args.  
- `hosts` from comma-separated `hosts=`; empty → `()`.  
- `tool` from `tag` (historical TensorBoard naming).  
- `use_saved_result` via `get_bool_arg(..., default=True)`.  
- Full `raw_args` retained for option builders (trace, graph, memory).

**Memory:** negligible vs convert. **Risk:** large query strings for multi-host lists are small; no payload body on GET.

### 2.3 `ToolDataService` / `tool_data_service`

**File:** `plugin/xprof/profile_plugin/services/tool_data.py`  
**Class:** `ToolDataService` — “data_impl algorithm without HTTP.”

#### Construction (from `plugin.py`)

```text
ToolDataService(
  convert=_ConvertAdapter(self._get_xspace_fn),  # → raw_to_tool_data.xspace_to_tool_data
  sessions=SessionResolver(...),
  hosts=HostSelector(),
  version=version_module,
  epath_module=epath,
  fs_factory=_FsFactoryAdapter(self._profile_fs),
)
```

#### `get_tool_data` steps (observed order)

1. **Fast path:** `try_counter_names_only(req)` — may return without convert.  
2. Default `run_dir_cache={}` / new lock if not injected (tests); production passes process maps.  
3. **`SessionResolver.run_map_from_params(session_path, run_path)`** then **`resolve_run_dir`**.  
4. **`should_use_saved_result(run_dir, req.use_saved_result, ...)`** → may force False.  
5. **`build_tool_params(req, use_saved_result=use_saved)`** — tool family options.  
6. Gate: tool must be in `TOOLS` or `use_xplane(tool)`; non-XPlane tools return empty `ToolResult`.  
7. **`HostSelector.select`** with `get_xplane_basenames(run_dir)`.  
8. Inject `params['hosts'] = list(selection.selected_hosts)`.  
9. **`convert.xspace_to_tool_data(asset_paths, tool, params)`**.  
10. If `not use_saved`: **`write_cache_version_file`** after successful convert.  
11. Return `ToolResult(data, content_type, content_encoding=None)`.

#### Memory-relevant properties (observed)

| Property | Implication |
|----------|-------------|
| `content_encoding=None` always | `respond` always gzip unless caller overrides |
| No single-flight / memo of payloads | Concurrent identical `/data` → N× convert |
| No size / host-count guard | ALL_HOSTS with large N fully accepted |
| Errors re-raised as AttributeError / ValueError / FileNotFoundError | HTTP 500; convert peak already spent |

#### Opportunities on `ToolDataService`

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| TDS-1 | Shared convert semaphore with cache worker | **H** | hypothesized |
| TDS-2 | Single-flight key `(run_dir, tool, sorted hosts, options hash)` | **M** | hypothesized |
| TDS-3 | Propagate precompressed encoding (pb/zstd) via `content_encoding` | **H** | observed gap |
| TDS-4 | Reject or page ALL_HOSTS when host count > policy | **M** | hypothesized |
| TDS-5 | Early size estimate from XPlane basenames/mtimes before convert | **L** | hypothesized |

### 2.4 `SessionResolver` / `session_resolver`

**File:** `plugin/xprof/profile_plugin/services/sessions.py`

#### Responsibilities

1. **`run_map_from_params(session_path, run_path)`**  
   - `session_path` wins: if dir has XPlanes → `{basename: session_path}`; else `{}`.  
   - Else `run_path` → `get_session_paths`.  
   - Else `None` → fall back to logdir layout.

2. **`resolve_run_dir(run, run_map, logdir, run_dir_cache, cache_lock)`**  
   - Map hit: return session dir.  
   - Else under lock check `_run_to_profile_run_dir`.  
   - Else derive `logdir / tb_run / plugins/profile / profile_run`.  
   - **Does not always write cache on logdir path** in this method (write happens in `RunDiscovery.iter_frontend_runs`).

#### Process-level map

- Plugin field: `_run_to_profile_run_dir` + `_run_dir_cache_lock` (`plugin.py`).  
- Populated heavily by **`RunDiscovery`** (`services/runs.py`) when listing runs.  
- Values are **directory path strings only** — not tool payloads. Memory of map is O(number of sessions), not O(profile size).

#### Memory notes

| Issue | Severity | Confidence |
|-------|----------|------------|
| Map unbounded growth with many profile sessions in long-lived TB process | **L**–**M** | hypothesized |
| No LRU eviction of run_dir_cache | **L** | observed |
| Session resolve itself not a peak contributor | — | observed |

### 2.5 `HostSelector` / `hosts_selector`

**File:** `plugin/xprof/profile_plugin/services/hosts.py`  
**Ported from:** `ProfilePlugin._get_valid_hosts`.

#### Selection matrix (observed)

| Condition | Result |
|-----------|--------|
| `hosts_param` set **and** `supports_multi_host_selection(tool)` | Parse comma list; require each host’s XPlane |
| `host == ALL_HOSTS` | **All** XPlanes + all host names |
| `host` in map | Single host path |
| `host` set but missing | Raise unless host is tool name in `XPLANE_TOOLS_ALL_HOSTS_ONLY` quirk |
| **No host and no hosts_param** | **All hosts** (implicit ALL_HOSTS) |
| Empty asset_paths after filters | Raise if host required for tool |

`supports_multi_host_selection` is **only** `trace_viewer` and `trace_viewer@` (`registry.py`).

#### Amplification

```text
Missing host on overview_page / kernel_stats / …
  → HostSelector selects every *.xplane.pb
  → convert holds Σ X_i (or sequential with residual OpStats vector)
  → peak ∝ N hosts (and for trace-like tools, T ≫ max X_i)
```

| Issue | Severity | Confidence |
|-------|----------|------------|
| Implicit all-hosts when host omitted | **H** | observed |
| Explicit `ALL_HOSTS` for multi-host sessions | **H** | observed |
| Multi-host selection only for trace tools | design constraint | observed |
| Warm-up always passes full `asset_paths` list | **H** | observed (`cache_api`) |

### 2.6 `result_cache_policy`

**File:** `plugin/xprof/profile_plugin/cache/result_cache_policy.py`  
**On-disk gate:** `cache_version.txt` (`CACHE_VERSION_FILE` in `constants.py`).

#### `should_use_saved_result(run_dir, requested, version_module, epath)`

| Condition | Effect |
|-----------|--------|
| File missing | Force `False` (invalidate) |
| `cache_version < version_module.__version__` (string compare) | Force `False` |
| Unreadable (OSError) | Force `False` |
| Otherwise | Honor `requested` (HTTP default True) |

**Observed quirks:**

- String comparison of versions (not semver parse) — generally OK for monotonic version strings; edge cases hypothesized.  
- **Does not delete** cache files itself; invalidation is logical (`use_saved_result=False`).  
- C++ side may still **ClearCacheFiles** when forced recompute (see §4 / `repository.cc` / op_stats processors).

#### `write_cache_version_file`

- Writes current plugin version into session dir.  
- Called from `ToolDataService` when `not use_saved` **after** convert succeeds.  
- Called from **`generate_cache` task at start** — **before** tools finish (**M**, observed race/stampede risk).

| Opportunity | Severity | Confidence |
|-------------|----------|------------|
| Write version only after all warm-up tools succeed | **M** | observed |
| On invalidation, proactively call ClearCacheFiles / disk quota | **M** | hypothesized |
| Version compare via packaging.version | **L** | hypothesized |

### 2.7 `tools_cache`

**File:** `plugin/xprof/profile_plugin/cache/tools_cache.py`  
**Class:** `ToolsCache`  
**Disk file:** `.cached_tools.json` next to session  
**Format version:** `CACHE_VERSION = 1`

#### Behavior

- **load:** Read JSON; check format version; compare `files` map to current XPlane file states (hashes/mtimes via FS). Hit → list of tool names.  
- **save:** Persist tools + current file states.  
- **invalidate:** Delete cache file.

#### Used by

- `RunDiscovery.tools_of_run` (`services/runs.py`): miss → `get_active_tools(all_filenames, run_dir)` then save.

#### Memory profile

| Aspect | Assessment | Confidence |
|--------|------------|------------|
| File size | Small JSON (tool name list + file states) | observed |
| Miss cost | May walk basenames; `get_active_tools` may trigger heavier discovery | observed |
| Relation to convert cache | **Orthogonal** — tools list ≠ OpStats/trace payloads | observed |
| Peak RAM | Not a convert peak driver | observed |

Opportunity: avoid calling `xspace_to_tool_names` / heavy paths during discovery (**L**–**M**, tool-list path only).

### 2.8 `generate_cache` — warm-up API

**File:** `plugin/xprof/profile_plugin/api/cache_api.py`  
**Class:** `CacheApiMixin`  
**Route:** `GENERATE_CACHE_ROUTE` = `/generate_cache` (POST only).

#### Acceptance path (`_generate_cache_impl`)

1. Require `session_path`.  
2. List all XPlane basenames → **sorted full `asset_paths`** (always all hosts).  
3. `runs_imp` must return exactly one run for that session.  
4. Discover available tools via `run_tools_imp`.  
5. Tools filter: request `tools=` or default **`DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')`**.  
6. Intersect with `XPLANE_TOOLS_SET`.  
7. Submit `_generate_cache_task` to **`self._cache_generation_pool`**.  
8. HTTP **202** `{status: ACCEPTED, ...}` immediately.

#### Task body (`_generate_cache_task`)

```text
write_cache_version_file(session_path)   # FIRST — before tools
for tool in tool_list:
    tool_params['hosts'] = hosts_from_xplane_filenames(filenames, tool)
    _get_xspace_fn()(all asset_paths as epath.Path, tool, tool_params)
    # broad except: log and continue other tools
```

#### Executor (`plugin.py`) — observed OOM mitigation

```text
ThreadPoolExecutor(max_workers=1, thread_name_prefix='XprofCacheGen')
# Comment in source: limit to 1 worker to prevent OOM (esp. trace viewer)
```

| Issue | Severity | Confidence |
|-------|----------|------------|
| max_workers=1 serializes warm-ups only | design | observed |
| No shared limit with foreground `/data` | **H** | hypothesized |
| Unbounded queue of POST tasks (serialized execution) | **M** | hypothesized |
| Version file before success | **M** | observed |
| Always all hosts / all session XPlanes | **H** | observed |
| Default tools include heaviest pair (overview + streaming trace) | **H** | observed |
| Failures swallowed per-tool (good for liveness; may leave partial cache) | **M** | observed |

### 2.9 `raw_to_tool_data` — Python convert dispatcher

**File:** `plugin/xprof/convert/raw_to_tool_data.py`

#### Entry points

| Function | Role |
|----------|------|
| `xspace_to_tool_data` | Path-based convert; default wrapper `_pywrap_profiler_plugin.xspace_to_tools_data` |
| `xspace_to_tools_data_from_byte_string` | In-memory XPlane bytes (tests / special) |
| `xspace_to_tool_names` | Tool name discovery via C++ `'tool_names'` |
| `process_raw_trace` | Legacy trace: ParseFromString → `''.join(TraceEventsJsonStream)` |
| `json_to_csv_string` | Delegate to `csv_writer` |

#### Tool branches and payload shapes (observed)

| Tool | Output path | content_type notes |
|------|-------------|--------------------|
| `trace_viewer` | pb or **process_raw_trace** full JSON join | json or octet-stream |
| `trace_viewer@` | raw_data (JSON or pb) | json or octet-stream if format=pb |
| overview / stats family | json_data from C++ | application/json |
| `memory_profile` | assert **exactly 1** xspace path | json |
| `graph_viewer` | raw; may raise ValueError with message | text/plain / html / octet |
| `memory_viewer` | raw; timeline → text/html | |
| `megascale_stats` | json or perfetto octet-stream | |
| unknown | warning; data=None | |

#### Legacy trace double materialization (observed)

```text
C++ raw Trace proto bytes
  → Trace.ParseFromString
  → TraceEventsJsonStream yields chunks
  → ''.join(...) forces full str
  → respond utf-8 + gzip
```

Streaming generators exist but are killed by join (**H**, observed). Prefer `trace_viewer@` / format=pb (see doc 01).

### 2.10 `csv_writer`

**File:** `plugin/xprof/convert/csv_writer.py`

Two APIs:

1. `json_to_csv` — simple object → single row (older).  
2. `json_to_csv_string` — DataTable-style `{cols, rows}` with `QUOTE_ALL` (used by `/data_csv`).

**Observed costs:**

- Bytes → decode → `json.loads` → full Python structure.  
- Walk every row into `StringIO`.  
- No streaming writer from C++ protobuf.

| Opportunity | Severity | Confidence |
|-------------|----------|------------|
| CSV from C++ / without full JSON DOM | **M** | observed |
| Row limit / projection for export | **L** | hypothesized |

### 2.11 `respond` — encoding & double buffers

**File:** `plugin/xprof/profile_plugin/http/respond.py`

#### Algorithm (observed)

1. If `content_type == application/json` and body is dict/list/set/tuple → `json.dumps(..., sort_keys=True)`.  
2. If not bytes → `encode('utf-8')`.  
3. If `content_encoding` truthy → set header only.  
4. **Else** → set `Content-Encoding: gzip` and **`body = gzip.compress(body)`**.  
5. Attach CSP + nosniff headers.  
6. Return Werkzeug Response with **full body in memory**.

#### Critical properties

| Property | Effect | Confidence |
|----------|--------|------------|
| Default gzip always | Peak ≈ raw + gzip (~1.1–2×) | observed |
| ToolDataService never sets encoding | All main tool traffic gzipped | observed |
| Already-zstd `format=pb` still gzipped | Double compression + hold both | observed (see 01) |
| Binary / HTML / CSV all same path | No type-based skip | observed |
| No streaming Response / free raw after compress | Both buffers coexist until Response build completes | observed |
| `json.dumps` only for Python structures | Main C++ path already emits JSON **bytes** — skips dumps | observed |

| ID | Opportunity | Severity | Confidence |
|----|-------------|----------|------------|
| R-1 | Stream gzip; release raw after compress | **H** | observed |
| R-2 | Skip gzip when `content_encoding` or magic (zstd/gzip) | **H** | observed |
| R-3 | Identity for small bodies / precompressed | **M** | observed |
| R-4 | Optional brotli/zstd with client Accept-Encoding | **L** | hypothesized |

### 2.12 `plugin.py` — wiring, maps, cache executor

**File:** `plugin/xprof/profile_plugin/plugin.py`

Creates:

- `SessionResolver`, `HostSelector`, `ToolDataService`  
- `_run_to_profile_run_dir` + lock  
- `RunDiscovery`  
- `_cache_generation_pool`: default `ThreadPoolExecutor(max_workers=1, ...)`  
- Lazy `_get_xspace_fn` → `raw_to_tool_data.xspace_to_tool_data`  
- Route table including `DATA_ROUTE`, `DATA_CSV_ROUTE`, `GENERATE_CACHE_ROUTE`

**Injectability (observed, good for tests):** `xspace_to_tool_data_fn`, `cache_generation_executor`, `epath_module`, `version_module`.

**Concurrency model (observed):**

```text
Werkzeug/TB request threads  ──N──►  /data convert   (unbounded)
XprofCacheGen pool           ──1──►  warm-up convert (serialized)
No shared semaphore between the two.
```

Peak process RSS ≈ **foreground converts + one cache worker + residual arenas**.

### 2.13 C++ `repository.cc` — SessionSnapshot I/O & disk caches

**File:** `xprof/convert/repository.cc`

#### Roles

- Map XPlane paths ↔ hostnames (`GetHostnameByPath` strips `.xplane.pb` / `.xplane.riegeli`).  
- **`GetXSpace`**: arena allocate; read binary proto or **riegeli** (read full file string → record merge — intermediate full file buffer).  
- Host data suffixes (cache artifacts):

| StoredDataType | Suffix |
|----------------|--------|
| DCN_COLLECTIVE_STATS | `.dcn_collective_stats.pb` |
| OP_STATS | `.op_stats_v2.pb` |
| SMART_SUGGESTION | `.smart_suggestion.pb` |
| TRACE_EVENTS_* | `.metadata.SSTABLE`, `.trie.SSTABLE`, `.SSTABLE` |
| RIEGELI_XSPACE | `.xplane.riegeli` (source, not cleared) |

- **`ClearCacheFiles`**: deletes all known cache suffixes **except** riegeli XPlanes.  
- **`HasCacheFile` / `GetHostDataFilePath`**: presence probes; list dir children each time.

#### Memory notes

| Issue | Severity | Confidence |
|-------|----------|------------|
| Riegeli path: full file in `contents` then merge records | **H** large files | observed |
| No size quota on written cache files | **H** growth | observed |
| ClearCacheFiles is all-or-nothing per session | **M** stampede risk after invalidate | observed |
| Arena-scoped XSpace lifetime depends on caller | design | observed |

### 2.14 C++ `base_op_stats_processor` — multi-host OpStats peak

**File:** `xprof/convert/base_op_stats_processor.cc`

#### Map phase (per host)

- If `hostname.op_stats_v2.pb` exists → return path (cache hit).  
- Else: **copy** XSpace → preprocess → `ConvertXSpaceToOpStats` with **always**  
  `generate_op_metrics_db`, `generate_step_db`, `generate_kernel_stats_db` = true  
- Write binary OpStats per host.

#### Reduce phase

- Read **all** map OpStats into `std::vector<OpStats> all_op_stats`.  
- Combine → write `ALL_HOSTS` OpStats → `ProcessCombinedOpStats`.

#### `ProcessSession` path

- `ConvertMultiXSpaceToCombinedOpStatsWithCache` (same family as overview/stats).

#### Worker service gate

- Multi-host + (`!use_saved_result` OR not all hosts cached) → use worker service path.

#### Explicit TODO in `multi_xplanes_to_op_stats.cc` (observed)

```text
// TODO(profiler): Change the combiner to convert and combine one OpStats at
// a time, to reduce peak memory usage.
```

| Issue | Severity | Confidence |
|-------|----------|------------|
| Full vector of per-host OpStats before combine | **H** | observed |
| Always generate kernel/step/metrics DBs | **M** | observed |
| ALL_HOSTS combined + per-host files on disk | **M** disk | observed |

---

## 3. Caching layers (three kinds — do not confuse)

### 3.1 Layer summary

| Layer | Name | Scope | Payload | Eviction |
|-------|------|-------|---------|----------|
| A | `result_cache_policy` + C++ host data | Per session dir | OpStats, SSTABLE, smart_suggestion, … | Version gate + ClearCacheFiles; **no quota** |
| B | `tools_cache` / `.cached_tools.json` | Per session | Tool name list | File-state mismatch |
| C | Process maps `_run_to_profile_run_dir` | Process | Path strings | Never (until process exit) |

**There is no in-process LRU of tool JSON/bytes** (observed). Every cold `/data` pays convert (or C++ disk cache hit inside convert).

### 3.2 Result / convert cache lifecycle

```text
HTTP use_saved_result=true (default)
  → should_use_saved_result may still force false if version missing/stale
  → options['use_saved_result'] into C++
  → C++ may read *.op_stats_v2.pb / LevelDB / etc.
  → on forced recompute: ClearCacheFiles / rewrite artifacts

generate_cache warm-up
  → write cache_version.txt immediately
  → convert DEFAULT_CACHE_TOOLS on all hosts
  → side effect: populates OpStats + trace LevelDB
```

### 3.3 Warm-up product behavior

| Item | Value | Confidence |
|------|-------|------------|
| Default tools | `overview_page`, `trace_viewer@` | observed |
| Hosts | All session XPlanes | observed |
| Pool | 1 worker | observed |
| HTTP | 202 ACCEPTED | observed |
| Version stamp | Before tools complete | observed |

**Risk:** UI opens `/data` for same session while warm-up runs → dual peak (cache worker + request) (**H**, hypothesized).

### 3.4 Disk growth risk

Unbounded per-host:

- `host.op_stats_v2.pb`  
- `host.SSTABLE` + metadata + trie (trace)  
- `host.smart_suggestion.pb`  
- `ALL_HOSTS.*` aggregates  

No TTL, max-bytes, or LRU (**H**, observed absence).

---

## 4. Encoding / double materialization matrix

| Path | Stages co-resident | Severity | Confidence |
|------|--------------------|----------|------------|
| Main `/data` JSON bytes | C++ string → Python bytes → gzip | **H** | observed |
| `/data` Python dict (rare) | dumps + utf-8 + gzip | **M** | observed |
| Legacy `trace_viewer` | proto + joined JSON + gzip | **H** | observed |
| `trace_viewer@` format=pb | zstd pb + gzip again | **H** | observed |
| `/data_csv` | JSON DOM + CSV + gzip | **M** | observed |
| graph HTML | large text + gzip | **M** | observed |
| Counter names_only | small text + gzip | **L** | observed |

**Rule of thumb:** any tool with multi-10MB+ JSON is dominated by **encode path + convert**, not by `HostSelector`/`SessionResolver`.

---

## 5. ALL_HOSTS amplification

### 5.1 Policy sources

| Registry set | Tools |
|--------------|-------|
| `XPLANE_TOOLS_ALL_HOSTS_SUPPORTED` | input_pipeline_analyzer, framework_op_stats, kernel_stats, overview_page, pod_viewer, megascale_stats |
| `XPLANE_TOOLS_ALL_HOSTS_ONLY` | overview_page, pod_viewer, smart_suggestion |
| Multi-host selection | trace_viewer, trace_viewer@ only |
| Single-host assert | memory_profile (`len(xspace_paths)==1`) |

### 5.2 Amplification formula (conceptual)

```text
Peak_convert ≈
  max over stages of:
    Σ selected XSpace decode  (or sequential max + residual)
  + intermediates (OpStats_i vector or Trace containers)
  + output payload
  + (HTTP) gzip(output)

For OpStats family multi-host:
  Peak ≈ Σ OpStats_i + combined  (TODO stream-combine)

For trace first open multi-host:
  Peak ≈ Σ X_i + TraceEvents  (or sequential ProcessSession)
```

### 5.3 Who pays

| Tool class | ALL_HOSTS default? | Notes |
|------------|--------------------|-------|
| overview, pod, smart_suggestion | Often only ALL_HOSTS in UI | Warm-up overview always |
| kernel/framework stats | UI can pick host or ALL | ALL is heaviest |
| memory_profile | Single host enforced | Lower host amp |
| graph/memory_viewer | HLO-centric | Different bottleneck |
| generate_cache | Always all files | Always |

---

## 6. Concurrency & single-flight

### 6.1 Observed controls

| Control | Scope | Limit |
|---------|-------|-------|
| Cache ThreadPoolExecutor | generate_cache tasks | **1** worker |
| TB/Werkzeug threads | /data, /data_csv, discovery | process/server default |
| Shared convert semaphore | — | **none** |
| Per-key single-flight | — | **none** |
| POST queue bound | generate_cache submit | unbounded (serialized run) |

### 6.2 Failure modes

| Scenario | Peak | Severity | Confidence |
|----------|------|----------|------------|
| Warm-up + user opens overview + trace | 2–3 large converts | **H** | hypothesized |
| User opens many tabs/tools | N converts | **H** | hypothesized |
| Stampede after version bump (`use_saved=false`) | N hosts clear + rebuild | **H** | hypothesized |
| Multiple generate_cache POSTs | Queue of heavy jobs | **M** | hypothesized |
| Cache max_workers raised without /data cap | Historical OOM reason | **H** | observed comment |

### 6.3 Recommended control plane (design only)

```text
global ConvertGate(max_in_flight=K)  # K=1 or 2 for RAM-constrained
  ├─ /data acquire
  ├─ /data_csv acquire
  └─ generate_cache worker acquire

single_flight(key=hash(run_dir, tool, hosts, options))
  → waiters share one Future[ToolResult]

optional: fair queue priority UI > warm-up
```

---

## 7. Cross-tool impact map

| Opportunity theme | Highest impact tools | Lower impact |
|-------------------|----------------------|--------------|
| Concurrent convert cap | trace_viewer@, multi-host overview/pod, warm-up+UI | graph_viewer single module, counter names |
| Stream gzip / free raw | large JSON tools, legacy trace | tiny responses |
| Stream / no join legacy JSON | trace_viewer | N/A if deprecated |
| Prefer trace_viewer@ / pb | trace path | other tools |
| ALL_HOSTS batch / stream-combine OpStats | overview, pod, kernel/framework stats, input pipeline | memory_profile (1 host) |
| Single-flight | popular default tools after open session | rare tools |
| Don’t gzip precompressed | trace pb, megascale perfetto | pure JSON tables |
| CSV without full loads | framework_op_stats, kernel_stats tables | binary/html tools |
| Version write after success | all warm-up sessions | none |
| Disk quota / ClearCache policy | multi-host long-lived sessions | single-host tiny |

---

## 8. Ranked opportunities (this document)

| Rank | ID | Opportunity | Severity | Confidence | Primary levers |
|------|----|-------------|----------|------------|----------------|
| 1 | C1 | Cap concurrent converts (`/data` + cache worker shared) | **H** | hypothesized | plugin + ToolDataService |
| 2 | C2 | Stream gzip / free raw after compress | **H** | observed | respond |
| 3 | C3 | Stream legacy trace JSON (no full `''.join`) | **H** | observed | raw_to_tool_data |
| 4 | C4 | Prefer `trace_viewer@` (+ format=pb); deprecate full JSON path | **H** | observed | registry + FE + raw_to_tool_data |
| 5 | C5 | Batch ALL_HOSTS / stream-combine OpStats (fix C++ TODO) | **H** | observed | multi_xplanes_to_op_stats / base_op_stats_processor |
| 6 | C6 | Single-flight per (session, tool, hosts, options) | **M** | hypothesized | ToolDataService |
| 7 | C7 | Don’t gzip binary/precompressed; plumb content_encoding | **M** | observed | ToolDataService + respond |
| 8 | C8 | CSV without full `json.loads` | **M** | observed | csv_writer + data_api |
| 9 | C9 | Write `cache_version` after successful warm-up tools | **M** | observed | generate_cache / result_cache_policy |
| 10 | C10 | Disk cache quota / cleanup / selective ClearCache | **M** | observed | repository.cc + policy |
| 11 | C11 | Bound generate_cache queue; drop/coalesce duplicates | **M** | hypothesized | cache_api + pool |
| 12 | C12 | Per-tool OpStatsOptions (skip unused DBs) | **M** | hypothesized | base_op_stats_processor |
| 13 | C13 | Guard implicit all-hosts when host omitted | **M** | observed | HostSelector / FE |
| 14 | C14 | LRU process maps / session metadata only | **L** | hypothesized | SessionResolver / plugin |
| 15 | C15 | Avoid hlo_module_list full convert when protos present | **L**–**M** | observed | data_api |

---

## 9. Sequence diagrams (request classes)

### 9.1 Cold `/data` overview_page ALL_HOSTS

```text
Client                data_api           ToolDataService      SessionResolver
  |                      |                      |                    |
  |--GET /data---------->|                      |                    |
  |                      |--get_tool_data------>|                    |
  |                      |                      |--resolve_run_dir-->|
  |                      |                      |<-run_dir-----------|
  |                      |                      | result_cache_policy|
  |                      |                      | HostSelector ALL   |
  |                      |                      | raw_to_tool_data   |
  |                      |                      |   C++ OpStats Σ    |
  |                      |<-ToolResult JSON-----|                    |
  |                      | respond gzip         |                    |
  |<-200 gzip JSON-------|                      |                    |
```

### 9.2 `/generate_cache` default tools

```text
Client          cache_api              pool(1)              convert
  |--POST /generate_cache-->|              |                   |
  |                        |--submit----->|                   |
  |<-202 ACCEPTED----------|              |                   |
  |                        |              |--write version--->|
  |                        |              |--overview_page--->| peak1
  |                        |              |--trace_viewer@--->| peak2
```

### 9.3 Overlap (worst case)

```text
Thread A: cache worker building trace_viewer@ LevelDB (max_workers=1)
Thread B: user /data overview_page ALL_HOSTS
Thread C: user /data kernel_stats ALL_HOSTS
→ three convert peaks without gate (hypothesized OOM)
```

---

## 10. Measurement plan (no product change)

| Experiment | What to capture | Success signal for later fix |
|------------|-----------------|------------------------------|
| M1 | RSS + wall for generate_cache default on multi-host session | Baseline warm-up peak |
| M2 | RSS for concurrent M1 + /data overview | Delta proves C1 value |
| M3 | Wire size + RSS for kernel_stats /data vs /data_csv | C8 value |
| M4 | format=pb vs JSON for trace_viewer@ including gzip layer | C2/C4/C7 |
| M5 | Disk growth of op_stats_v2 + SSTABLE over N tools | C10 |
| M6 | Host count sweep N=1,2,4,8 ALL_HOSTS overview | C5 scaling curve |
| M7 | Version file timing vs first tool completion logs | C9 |

Instrumentation hooks (existing logs):

- `ConvertMultiXSpaceToCombinedOpStatsWithCache` INFO timings  
- `generate_cache` logger steps  
- Cache pool thread name `XprofCacheGen`

---

## 11. Phased recommendations (analysis roadmap)

### Phase 0 — Measure (align with synthesis doc)

Run M1–M7 on representative multi-host production-like sessions.

### Phase 1 — Low design risk, high encode ROI

1. **C7 / C2:** Plumb `content_encoding` for precompressed; skip double gzip; free raw.  
2. **C9:** Move `write_cache_version_file` to end of successful `_generate_cache_task`.  
3. **C13:** FE always sends host; optional server guard for hostless all-hosts on heavy tools.

### Phase 2 — Convert control plane

1. **C1:** Shared convert semaphore (K=1 or 2).  
2. **C6:** Single-flight for identical requests.  
3. **C11:** Coalesce generate_cache for same session_path.

### Phase 3 — Convert algorithm

1. **C5:** Stream-combine OpStats (resolve C++ TODO).  
2. **C12:** Tool-specific OpStatsOptions.  
3. **C3 / C4:** Legacy trace elimination; pb path for V2.

### Phase 4 — Disk hygiene

1. **C10:** Quota, TTL, or size-aware ClearCache.  
2. Metrics on cache dir size in session.

---

## 12. Non-goals / out of scope for this note

- Frontend Angular heap (partially covered in 01 for trace; tables in 04).  
- CLI projection modes (`summary_only`) — synthesis theme 1; tool docs 03/05/06.  
- Capture/profile recording path (`capture_api`).  
- Product implementation PRs (this campaign is analysis only).

---

## 13. Real paths cited

### Python plugin spine

- `plugin/xprof/profile_plugin/services/tool_data.py` — **ToolDataService** / tool_data_service  
- `plugin/xprof/profile_plugin/services/hosts.py` — **HostSelector** / hosts_selector  
- `plugin/xprof/profile_plugin/services/sessions.py` — **SessionResolver** / session_resolver  
- `plugin/xprof/profile_plugin/services/runs.py` — RunDiscovery + **tools_cache** usage  
- `plugin/xprof/profile_plugin/api/data_api.py` — **data_api**  
- `plugin/xprof/profile_plugin/api/cache_api.py` — **generate_cache**  
- `plugin/xprof/profile_plugin/http/respond.py` — **respond**  
- `plugin/xprof/profile_plugin/http/parse_request.py`  
- `plugin/xprof/profile_plugin/cache/tools_cache.py` — **tools_cache**  
- `plugin/xprof/profile_plugin/cache/result_cache_policy.py` — **result_cache_policy**  
- `plugin/xprof/profile_plugin/plugin.py` — cache executor, wiring  
- `plugin/xprof/profile_plugin/tools/registry.py` — DEFAULT_CACHE_TOOLS, ALL_HOSTS sets  
- `plugin/xprof/profile_plugin/models.py` — ToolRequest, HostSelection, ToolResult  
- `plugin/xprof/profile_plugin/constants.py` — CACHE_VERSION_FILE, routes, ALL_HOSTS  

### Convert Python

- `plugin/xprof/convert/raw_to_tool_data.py` — **raw_to_tool_data**  
- `plugin/xprof/convert/csv_writer.py`  
- `plugin/xprof/convert/trace_events_json.py` (legacy stream join)

### Convert C++

- `xprof/convert/repository.cc` / `repository.h` — SessionSnapshot, ClearCacheFiles, GetXSpace  
- `xprof/convert/base_op_stats_processor.cc` — Map/Reduce OpStats, use_saved_result  
- `xprof/convert/multi_xplanes_to_op_stats.cc` — combine TODO, WithCache  
- `xprof/convert/xplane_to_tools_data.cc` — tool dispatch  
- `xprof/pywrap/profiler_plugin_impl.cc` (pywrap surface)

---

## 14. Relationship to sibling design notes

| Doc | Overlap with this path |
|-----|------------------------|
| `00-synthesis-memory-optimization.md` | Global ranking; this doc is the convert/serve/cache deep dive |
| `01-trace-viewer-path.md` | Specializes C3/C4/C7 for streaming LevelDB + V2 pb |
| `03-memory-tools-family.md` | memory_profile single-host; memory_viewer HLO |
| `04-overview-and-stats-tools.md` | OpStats family; expands C5/C12 per tool |
| `05-graph-hlo-util-tools.md` | graph_viewer / util; still uses same respond + ToolDataService |
| `06-megascale-and-cli-detectors.md` | megascale_stats / perfetto encoding; CLI bypasses some HTTP |

---

## 15. Glossary

| Term | Meaning in this codebase |
|------|--------------------------|
| Frontend run | TB UI run name (`tb_run/profile_run` or bare profile run) |
| Session / run_dir | Directory with `*.xplane.pb` (+ caches) |
| ALL_HOSTS | Sentinel host name selecting every XPlane |
| use_saved_result | Python/C++ flag to prefer on-disk convert artifacts |
| cache_version.txt | Plugin version stamp gating saved results |
| Warm-up | POST **generate_cache** background convert |
| Convert peak | Max RSS attributable to one xspace_to_tool_data call |
| Single-flight | Collapse duplicate in-flight converts to one execution |

---

## 16. Summary judgment

The convert → serve → cache path is **architecturally clean** (SessionResolver → HostSelector → convert → respond) but **memory-unsafe at the edges**:

1. **Encode:** `respond` always materializes full gzip alongside raw.  
2. **Hosts:** omitted host = all hosts; warm-up always all hosts.  
3. **Concurrency:** cache pool is carefully capped at 1; interactive `/data` is not.  
4. **Disk cache:** powerful OpStats/LevelDB reuse, no quota, version stamped early on warm-up.  
5. **Combine:** multi-host OpStats still holds full vectors (explicit TODO).

Closing C1–C5 first would address the largest cross-tool multipliers before investing in per-tool projection modes (synthesis theme 1, docs 03–06).

---

*End of deepened analysis. No product code changes.*
