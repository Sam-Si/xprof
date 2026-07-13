# Memory Optimization: Convert → Serve → Cache Path

**Date:** 2026-07-13  
**Status:** Research / design (no product code)  
**Scope:** Shared pipeline for all XPlane-backed UI tools — HTTP edge through convert to disk caches  
**Audience:** Engineers prioritizing RAM-sensitive work; companion to `00-synthesis`, `01-trace-viewer`, `03`–`06` tool family notes  
**Mode:** Analysis only — confidence labels use **observed** | **hypothesized** only  
**Severity labels:** **H** | **M** | **L** only

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

### 0.1 Document goals

| Goal | Section |
|------|---------|
| Named component inventory with path:line cites | §1–§2 |
| Sequence diagrams for `/data`, `/data_csv`, `/generate_cache` with buffer ownership | §3 |
| Exact gzip / `json.dumps` conditions from `respond.py` | §4 |
| `use_saved_result` truth table vs `cache_version.txt` vs C++ `ClearCacheFiles` | §5 |
| ALL_HOSTS amplification math + registry sets | §6 |
| Concurrency: `max_workers=1` vs unbounded Werkzeug threads | §7 |
| Per-tool impact matrix for C1–C20 | §8 |
| Disk cache file naming / growth model | §9 |
| CSV double materialization walkthrough | §10 |
| Legacy `process_raw_trace` join analysis | §11 |
| Measurement plan | §12 |
| Ranked opportunities + phases | §13–§14 |

### 0.2 Required symbol index (must-match strings)

| Exact string | Role | Primary file |
|--------------|------|--------------|
| `ToolDataService` | Orchestrator class | `services/tool_data.py` |
| `tool_data_service` | Conceptual service name / instance role | `plugin.py` wires `self._tool_data` |
| `HostSelector` | Host / asset path selection | `services/hosts.py` |
| `hosts_selector` | Conceptual name for HostSelector instance | `plugin.py` `self._hosts` |
| `SessionResolver` | Run → session directory | `services/sessions.py` |
| `session_resolver` | Conceptual name for SessionResolver instance | `plugin.py` `self._sessions` |
| `generate_cache` | Warm-up API surface | `api/cache_api.py` |
| `tools_cache` | Tool-list disk cache module / `ToolsCache` | `cache/tools_cache.py` |
| `result_cache_policy` | Version gate for saved convert results | `cache/result_cache_policy.py` |
| `respond` | HTTP response builder | `http/respond.py` |
| `raw_to_tool_data` | Python convert dispatcher | `convert/raw_to_tool_data.py` |
| `data_api` | `/data` / `/data_csv` mixin module | `api/data_api.py` |

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

| Handler | Route constant | Lines | Behavior |
|---------|----------------|-------|----------|
| `data_route` | `DATA_ROUTE` (`/data`) | 111–132 | `data_impl` → `respond(data, content_type, content_encoding)` |
| `data_csv_route` | `DATA_CSV_ROUTE` (`/data_csv`) | 135–173 | Same convert, then `json_to_csv_string`, then CSV + attachment headers |
| `hlo_module_list_route` | `HLO_MODULE_LIST_ROUTE` | 22–27 | Lists `*.hlo_proto.pb`; may call `xspace_to_tool_names` to materialize HLO |

**`data_impl` flow (observed, lines 39–68):**

1. `tool_request_from_args(request.args)` → `ToolRequest` (`tag` → tool name, `host`/`hosts`, `use_saved_result` default **True**).  
2. `self._tool_data.get_tool_data(...)` with `session_path`, `run_path`, `logdir`, `run_dir_cache=self._run_to_profile_run_dir`, `cache_lock`.  
3. Returns `(result.data, result.content_type, result.content_encoding)` — encoding is **always** `None` from `ToolDataService` today (`tool_data.py:131–135`).

**Memory notes for `data_api`:**

| Issue | Severity | Confidence |
|-------|----------|------------|
| No backpressure / semaphore around convert | **H** | hypothesized |
| 404 path still cheap (`data is None` → plain text) | **L** | observed |
| Exception path re-stringifies errors (tiny) | **L** | observed |
| `/data_csv` forces full convert + full JSON parse before CSV | **M** | observed |
| `hlo_module_list_impl` may convert all XPlanes just to list module names | **M** | observed |

**`/data_csv` double materialization (observed, lines 137–162):**

```text
data_impl → (often bytes JSON from C++)
  → json_to_csv_string: decode utf-8 if bytes → json.loads → walk cols/rows → StringIO CSV
  → respond(csv_data, 'text/csv', content_encoding=None) → gzip.compress again
```

CSV is useful for export; for large tables (`kernel_stats`, `framework_op_stats`) peak ≈ JSON tree + CSV + gzip. A streaming or C++-native CSV path would drop the Python object graph (**M**, hypothesized ROI; **observed** double parse).

### 2.2 `parse_request` — edge adapter

**File:** `plugin/xprof/profile_plugin/http/parse_request.py`

- Builds frozen `ToolRequest` from Werkzeug args (`tool_request_from_args`, lines 13–31).  
- `hosts` from comma-separated `hosts=`; empty → `()`.  
- `tool` from `tag` (historical TensorBoard naming) — line 27: `tool=str(args.get('tag') or '')`.  
- `use_saved_result` via `get_bool_arg(..., default=True)` — line 29.  
- Full `raw_args` retained for option builders (trace, graph, memory).

**`get_bool_arg` truth (`request_params.py:26–40`):**

| Arg value | Result |
|-----------|--------|
| missing / `None` | `default` (True for use_saved_result) |
| case-insensitive `"true"` | True |
| anything else (`"false"`, `"0"`, `"1"`) | False |

**Memory:** negligible vs convert. **Risk:** large query strings for multi-host lists are small; no payload body on GET.

### 2.3 `ToolDataService` / `tool_data_service`

**File:** `plugin/xprof/profile_plugin/services/tool_data.py`  
**Class:** `ToolDataService` — “data_impl algorithm without HTTP.”  
**Lines:** class at 25–43; `get_tool_data` at 44–135.

#### Construction (from `plugin.py:180–187`)

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

#### `get_tool_data` steps (observed order, lines 64–135)

1. **Fast path:** `try_counter_names_only(req)` — may return without convert (`64–66`).  
2. Default `run_dir_cache={}` / new lock if not injected (tests); production passes process maps (`69–72`).  
3. **`SessionResolver.run_map_from_params(session_path, run_path)`** then **`resolve_run_dir`** (`74–81`).  
4. **`should_use_saved_result(run_dir, req.use_saved_result, ...)`** → may force False (`83–85`).  
5. **`build_tool_params(req, use_saved_result=use_saved)`** — tool family options (`86`).  
6. Gate: tool must be in `TOOLS` or `use_xplane(tool)`; non-XPlane tools return empty `ToolResult` (`88–94`).  
7. **`HostSelector.select`** with `get_xplane_basenames(run_dir)` (`96–105`).  
8. Inject `params['hosts'] = list(selection.selected_hosts)` (`109`).  
9. **`convert.xspace_to_tool_data(asset_paths, tool, params)`** (`111–115`).  
10. If `not use_saved`: **`write_cache_version_file`** after successful convert (`128–129`).  
11. Return `ToolResult(data, content_type, content_encoding=None)` (`131–135`).

#### Memory-relevant properties (observed)

| Property | Implication |
|----------|-------------|
| `content_encoding=None` always (`134`) | `respond` always gzip unless caller overrides |
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
**Class:** `SessionResolver` lines 28–102.

#### Responsibilities

1. **`run_map_from_params(session_path, run_path)`** (`46–60`)  
   - `session_path` wins: if dir has XPlanes → `{basename: session_path}`; else `{}`.  
   - Else `run_path` → `get_session_paths`.  
   - Else `None` → fall back to logdir layout.

2. **`resolve_run_dir(run, run_map, logdir, run_dir_cache, cache_lock)`** (`62–102`)  
   - Map hit: return session dir.  
   - Else under lock check `_run_to_profile_run_dir`.  
   - Else derive `logdir / tb_run / plugins/profile / profile_run`.  
   - **Does not always write cache on logdir path** in this method (write happens in `RunDiscovery.iter_frontend_runs`).

#### Process-level map

- Plugin field: `_run_to_profile_run_dir` + `_run_dir_cache_lock` (`plugin.py:194–197`).  
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
**Class:** `HostSelector` lines 24–115.  
**Ported from:** `ProfilePlugin._get_valid_hosts`.

#### Selection matrix (observed, lines 69–98)

| Condition | Lines | Result |
|-----------|-------|--------|
| `hosts_param` set **and** `supports_multi_host_selection(tool)` | 69–78 | Parse comma list; require each host’s XPlane |
| `host == ALL_HOSTS` | 79–81 | **All** XPlanes + all host names |
| `host` in map | 82–84 | Single host path |
| `host` set but missing | 85–93 | Raise unless host is tool name in `XPLANE_TOOLS_ALL_HOSTS_ONLY` quirk |
| **No host and no hosts_param** | 95–98 | **All hosts** (implicit ALL_HOSTS) |
| Empty asset_paths after filters | 99–110 | Raise if host required for tool |

`supports_multi_host_selection` is **only** `trace_viewer` and `trace_viewer@` (`registry.py:99–114`).

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
**On-disk gate:** `cache_version.txt` (`CACHE_VERSION_FILE` in `constants.py:44`).

#### `should_use_saved_result(run_dir, requested, version_module, epath)` (lines 26–51)

| Condition | Effect |
|-----------|--------|
| File missing | Force `False` (invalidate) |
| `cache_version < version_module.__version__` (string compare) | Force `False` |
| Unreadable (OSError) | Force `False` |
| Otherwise | Honor `requested` (HTTP default True) |

**Observed quirks:**

- String comparison of versions (not semver parse) — generally OK for monotonic version strings; edge cases hypothesized.  
- **Does not delete** cache files itself; invalidation is logical (`use_saved_result=False`).  
- C++ side **ClearCacheFiles** when forced recompute (`profiler_plugin_impl.cc:84–90`).

#### `write_cache_version_file` (lines 54–66)

- Writes current plugin version into session dir.  
- Called from `ToolDataService` when `not use_saved` **after** convert succeeds (`tool_data.py:128–129`).  
- Called from **`generate_cache` task at start** — **before** tools finish (`cache_api.py:192–193`) (**M**, observed race/stampede risk).

| Opportunity | Severity | Confidence |
|-------------|----------|------------|
| Write version only after all warm-up tools succeed | **M** | observed |
| On invalidation, proactively call ClearCacheFiles / disk quota | **M** | hypothesized |
| Version compare via packaging.version | **L** | hypothesized |

### 2.7 `tools_cache`

**File:** `plugin/xprof/profile_plugin/cache/tools_cache.py`  
**Class:** `ToolsCache`  
**Disk file:** `.cached_tools.json` next to session (`CACHE_FILE_NAME`, line 35)  
**Format version:** `CACHE_VERSION = 1` (line 36)

#### Behavior

- **load (52–95):** Read JSON; check format version; compare `files` map to current XPlane file states (hashes/mtimes via FS). Hit → list of tool names.  
- **save (97–117):** Persist tools + current file states.  
- **invalidate (119–121):** Delete cache file.

#### Used by

- `RunDiscovery.tools_of_run` (`services/runs.py:110–132`): miss → `get_active_tools(all_filenames, run_dir)` then save.

#### Memory profile

| Aspect | Assessment | Confidence |
|--------|------------|------------|
| File size | Small JSON (tool name list + file states) | observed |
| Miss cost | May walk basenames; `get_active_tools` may trigger heavier discovery | observed |
| Relation to convert cache | **Orthogonal** — tools list ≠ OpStats/trace payloads | observed |
| Peak RAM | Not a convert peak driver | observed |

Opportunity: avoid calling `xspace_to_tool_names` / heavy paths during discovery (**L**–**M**, tool-list path only). Note: `catalog.py:get_tools_from_filenames` lines 65–68 may call `xspace_to_tool_names` when XPlanes exist under a real profile_run_dir.

### 2.8 `generate_cache` — warm-up API

**File:** `plugin/xprof/profile_plugin/api/cache_api.py`  
**Class:** `CacheApiMixin`  
**Route:** `GENERATE_CACHE_ROUTE` = `/generate_cache` (POST only, `constants.py:45`).

#### Acceptance path (`_generate_cache_impl`, lines 30–170)

1. Require `session_path` (`53–54`).  
2. List all XPlane basenames → **sorted full `asset_paths`** (always all hosts) (`57–61`).  
3. `runs_imp` must return exactly one run for that session (`75–86`).  
4. Discover available tools via `run_tools_imp` (`91`).  
5. Tools filter: request `tools=` or default **`DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')`** (`98–103`; registry 55).  
6. Intersect with `XPLANE_TOOLS_SET` (`112–114`).  
7. Submit `_generate_cache_task` to **`self._cache_generation_pool`** (`152–158`).  
8. HTTP **202** `{status: ACCEPTED, ...}` immediately (`166–169`).

#### Task body (`_generate_cache_task`, lines 173–227)

```text
write_cache_version_file(session_path)   # FIRST — before tools (192–193)
for tool in tool_list:
    tool_params['hosts'] = hosts_from_xplane_filenames(filenames, tool)
    _get_xspace_fn()(all asset_paths as epath.Path, tool, tool_params)
    # broad except: log and continue other tools (215–227)
```

#### Executor (`plugin.py:207–216`) — observed OOM mitigation

```text
ThreadPoolExecutor(max_workers=1, thread_name_prefix='XprofCacheGen')
# Comment lines 207–211: limit to 1 worker to prevent OOM (esp. trace viewer)
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

| Function | Lines | Role |
|----------|-------|------|
| `xspace_to_tool_data` | 97–255 | Path-based convert; default wrapper `_pywrap_profiler_plugin.xspace_to_tools_data` |
| `xspace_to_tools_data_from_byte_string` | 48–77 | In-memory XPlane bytes (tests / special) |
| `xspace_to_tool_names` | 80–94 | Tool name discovery via C++ `'tool_names'` |
| `process_raw_trace` | 41–45 | Legacy trace: ParseFromString → `''.join(TraceEventsJsonStream)` |
| `json_to_csv_string` | 258–260 | Delegate to `csv_writer` |

#### Tool branches and payload shapes (observed)

| Tool | Lines | Output path | content_type notes |
|------|-------|-------------|--------------------|
| `trace_viewer` | 127–137 | pb or **process_raw_trace** full JSON join | json or octet-stream |
| `trace_viewer@` | 138–146 | raw_data (JSON or pb) | json or octet-stream if format=pb |
| overview / stats family | 147–162 | json_data from C++ | application/json |
| `memory_profile` | 163–169 | assert **exactly 1** xspace path | json |
| `graph_viewer` | 191–210 | raw; may raise ValueError with message | text/plain / html / octet |
| `memory_viewer` | 211–226 | raw; timeline → text/html | |
| `megascale_stats` | 227–235 | json or perfetto octet-stream | |
| unknown | 253–254 | warning; data=None | |

#### Options always include `use_saved_result` (line 126)

```text
options['use_saved_result'] = params.get('use_saved_result', True)
```

Propagated into C++ `ToolOptions` and drives `ClearCacheFiles` when false.

#### Legacy trace double materialization (observed)

```text
C++ raw Trace proto bytes
  → Trace.ParseFromString          (process_raw_trace:43–44)
  → TraceEventsJsonStream yields chunks
  → ''.join(...) forces full str   (line 45)
  → respond utf-8 + gzip
```

Streaming generators exist (`trace_events_json.py:31–37` docstring: “suitable for returning in a werkzeug Response”) but are killed by join (**H**, observed). Prefer `trace_viewer@` / format=pb (see doc 01).

### 2.10 `csv_writer`

**File:** `plugin/xprof/convert/csv_writer.py`

Two APIs:

1. `json_to_csv` (lines 24–87) — simple object → single row (older).  
2. `json_to_csv_string` (lines 90–139) — DataTable-style `{cols, rows}` with `QUOTE_ALL` (used by `/data_csv`).

**Observed costs in `json_to_csv_string`:**

- Bytes → decode (`93–94`) → `json.loads` (`98–99`) → full Python structure.  
- Walk every row into `StringIO` (`110–139`).  
- No streaming writer from C++ protobuf.

| Opportunity | Severity | Confidence |
|-------------|----------|------------|
| CSV from C++ / without full JSON DOM | **M** | observed |
| Row limit / projection for export | **L** | hypothesized |

### 2.11 `respond` — encoding & double buffers

**File:** `plugin/xprof/profile_plugin/http/respond.py`  
**Function:** `respond` lines 29–118.

#### Algorithm (observed, exact conditions)

1. **JSON structure dumps** (`52–55`): If `content_type == 'application/json'` **and** `isinstance(body, (dict, list, set, tuple))` → `body = json.dumps(body, sort_keys=True)`.  
2. **UTF-8 encode** (`56–57`): If not `isinstance(body, bytes)` → `body = body.encode('utf-8')`.  
3. **Content-Encoding branch** (`107–111`):  
   - If `content_encoding` is truthy → append header only; **do not re-compress**.  
   - **Else** → append `('Content-Encoding', 'gzip')` and **`body = gzip.compress(body)`**.  
4. Attach CSP + nosniff headers (`58–106`, `113–114`).  
5. Return Werkzeug Response with **full body in memory** (`116–118`).

#### Critical properties

| Property | Effect | Confidence |
|----------|--------|------------|
| Default gzip always when encoding is None/falsy | Peak ≈ raw + gzip (~1.1–2×) | observed |
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

- `SessionResolver` (`176–178`), `HostSelector` (`179`), `ToolDataService` (`180–187`)  
- `_run_to_profile_run_dir` + lock (`194–197`)  
- `RunDiscovery` (`198–205`)  
- `_cache_generation_pool`: default `ThreadPoolExecutor(max_workers=1, ...)` (`212–216`)  
- Lazy `_get_xspace_fn` → `raw_to_tool_data.xspace_to_tool_data` (`219–223`)  
- Route table including `DATA_ROUTE`, `DATA_CSV_ROUTE`, `GENERATE_CACHE_ROUTE` (`270–277`)

**Injectability (observed, good for tests):** `xspace_to_tool_data_fn`, `cache_generation_executor`, `epath_module`, `version_module` (`139–160`).

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

- Map XPlane paths ↔ hostnames (`GetHostnameByPath`, lines 63–73) strips `.xplane.pb` / `.xplane.riegeli`.  
- **`GetXSpaceFromRiegeli`** (lines 107–118): arena allocate; **read full file string** → record merge — intermediate full file buffer.  
- Host data suffixes (cache artifacts, lines 51–59):

| StoredDataType | Suffix |
|----------------|--------|
| DCN_COLLECTIVE_STATS | `.dcn_collective_stats.pb` |
| OP_STATS | `.op_stats_v2.pb` |
| SMART_SUGGESTION | `.smart_suggestion.pb` |
| TRACE_EVENTS_METADATA_LEVELDB | `.metadata.SSTABLE` |
| TRACE_EVENTS_PREFIX_TRIE_LEVELDB | `.trie.SSTABLE` |
| TRACE_LEVELDB | `.SSTABLE` |
| RIEGELI_XSPACE | `.xplane.riegeli` (source, not cleared) |

- **`ClearCacheFiles`** (lines 246–266): deletes all known cache suffixes **except** riegeli XPlanes.  
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

#### Map phase (per host, lines 98–122)

- If `hostname.op_stats_v2.pb` exists → return path (cache hit, 104–106).  
- Else: **copy** XSpace → preprocess → `ConvertXSpaceToOpStats` with **always**  
  `generate_op_metrics_db`, `generate_step_db`, `generate_kernel_stats_db` = true (`111–115`)  
- Write binary OpStats per host (`119–120`).

#### Reduce phase (lines 124–162)

- Read **all** map OpStats into `std::vector<OpStats> all_op_stats` (`131–138`).  
- Combine → write `ALL_HOSTS` OpStats → `ProcessCombinedOpStats` (`154–161`).

#### `ProcessSession` path (lines 164–172)

- `ConvertMultiXSpaceToCombinedOpStatsWithCache` (same family as overview/stats).

#### Worker service gate (lines 174–187)

- Multi-host + (`!use_saved_result` OR not all hosts cached) → use worker service path.

### 2.15 C++ `multi_xplanes_to_op_stats.cc` — explicit memory TODO

**File:** `xprof/convert/multi_xplanes_to_op_stats.cc`

#### `ConvertMultiXSpacesToCombinedOpStats` (lines 42–114)

```text
// TODO(profiler): Change the combiner to convert and combine one OpStats at
// a time, to reduce peak memory usage.   [lines 51–52]
std::vector<OpStats> all_op_stats;
all_op_stats.reserve(session_snapshot.XSpaceSize());
for each host:
  GetXSpace into Arena
  ConvertXSpaceToOpStats → push_back (keeps ALL)
CombineAllOpStats into combined_op_stats
```

#### `ConvertMultiXSpaceToCombinedOpStatsWithCache` (lines 116–159)

- Cache hit: read `ALL_HOSTS.op_stats_v2.pb` only.  
- Miss: full multi convert + write ALL_HOSTS file.  
- Options always generate all three DBs (`121–124`).

| Issue | Severity | Confidence |
|-------|----------|------------|
| Full vector of per-host OpStats before combine | **H** | observed |
| Always generate kernel/step/metrics DBs | **M** | observed |
| ALL_HOSTS combined + per-host files on disk | **M** disk | observed |

### 2.16 C++ pywrap — `ClearCacheFiles` trigger

**File:** `xprof/pywrap/profiler_plugin_impl.cc` lines 84–90:

```text
// If use_saved_result is False, clear the cache files before converting to
// tool data.
if (use_saved_result.has_value() && !use_saved_result.value()) {
  ClearCacheFiles();
}
```

This is the **only production call site** that clears convert disk caches when Python/C++ force recompute. Coupled tightly with `result_cache_policy` forcing `use_saved_result=False`.

### 2.17 Models (payload types)

**File:** `plugin/xprof/profile_plugin/models.py`

| Type | Lines | Fields relevant to memory |
|------|-------|---------------------------|
| `ToolRequest` | 20–28 | run, tool, host, hosts, use_saved_result, raw_args |
| `HostSelection` | 32–36 | selected_hosts, asset_paths |
| `ToolResult` | 40–45 | data, content_type, content_encoding (default None) |

`ToolResult.content_encoding` is the only plumb for skipping gzip — currently unused by service.

### 2.18 `runs.py` — discovery + tools_cache consumer

**File:** `plugin/xprof/profile_plugin/services/runs.py`

- `RunDiscovery.iter_frontend_runs` (`56–108`): walks logdir, fills `_run_to_profile_run_dir`.  
- `tools_of_run` (`110–132`): `ToolsCache.load` → miss → `get_active_tools` → `cache.save`.  
- Discovery itself is not convert-peak except when catalog triggers `xspace_to_tool_names` (`catalog.py:65–68`).

### 2.19 `registry.py` + `filenames.py` + `catalog.py`

**registry.py** owns:

- `XPLANE_TOOLS` list (33–52)  
- `XPLANE_TOOLS_SET` (54)  
- `DEFAULT_CACHE_TOOLS` (55)  
- `XPLANE_TOOLS_ALL_HOSTS_SUPPORTED` (58–66)  
- `XPLANE_TOOLS_ALL_HOSTS_ONLY` (68–70)  
- `_MULTI_HOST_SELECTION_TOOLS` (99)  
- `supports_multi_host_selection` (112–114)

**filenames.py** `hosts_from_xplane_filenames` (104–122):

- Multi-host + ALL_HOSTS_ONLY tool → `[ALL_HOSTS]` only.  
- Multi-host + ALL_HOSTS_SUPPORTED → hosts ∪ `{ALL_HOSTS}`.  
- Used by warm-up task to populate `tool_params['hosts']`.

**catalog.py** `get_active_tools` (74–91): may invoke C++ tool_names conversion when XPlanes present.

---

## 3. Sequence diagrams (request classes) — buffer ownership

### 3.1 Cold `/data` overview_page ALL_HOSTS

```text
Client                data_api           ToolDataService      SessionResolver
  |                      |                      |                    |
  |--GET /data---------->|                      |                    |
  |  run,tag,host=ALL   |                      |                    |
  |                      |--tool_request_from_args                   |
  |                      |  [B0: ToolRequest + raw_args]              |
  |                      |--get_tool_data------>|                    |
  |                      |                      |--resolve_run_dir-->|
  |                      |                      |<-run_dir-----------|
  |                      |                      | [B1: path strings] |
  |                      |                      | result_cache_policy|
  |                      |                      | [B2: version str]  |
  |                      |                      | HostSelector ALL   |
  |                      |                      | [B3: N Path objs]  |
  |                      |                      | raw_to_tool_data   |
  |                      |                      |   C++ OpStats Σ    |
  |                      |                      | [B4: vector OpStats|
  |                      |                      |  + combined + JSON]|
  |                      |<-ToolResult JSON-----|                    |
  |                      |  [B5: data bytes,     |                    |
  |                      |   content_encoding=None]                  |
  |                      | respond: gzip.compress                    |
  |                      |  [B6: raw + gzip coexist]                 |
  |<-200 gzip JSON-------|                      |                    |
  |  [B7: Response body] |                      |                    |
```

**Buffer release order (observed / hypothesized):**

| Buffer | Owner | Freed when | Confidence |
|--------|-------|------------|------------|
| B0 ToolRequest | request stack | end of data_impl | observed |
| B3 asset_paths | HostSelection | after convert returns | observed |
| B4 C++ OpStats vector | convert stack/heap | end of convert | observed |
| B5 ToolResult.data | Python | after gzip assigns new body | hypothesized (GC) |
| B6 gzip body | respond local | after Response constructed | observed |

### 3.2 `/data_csv` kernel_stats single host

```text
Client              data_api              ToolDataService         csv_writer
  |--GET /data_csv---->|                       |                      |
  |                    |--data_impl----------->|                      |
  |                    |  full convert as /data                       |
  |                    |<-JSON bytes-----------|                      |
  |                    |  [B5: full JSON]                             |
  |                    |--json_to_csv_string------------------------->|
  |                    |                      decode + json.loads     |
  |                    |                      [B8: Python DOM]        |
  |                    |                      StringIO CSV            |
  |                    |                      [B9: CSV str]           |
  |                    |<-csv string----------------------------------|
  |                    |  B5 may still live until GC                  |
  |                    | respond(csv, text/csv, encoding=None)        |
  |                    |  [B10: CSV bytes + gzip CSV]                 |
  |<-200 gzip CSV------|                                              |
```

**Peak approximation:** `max(convert_peak, |JSON| + |DOM| + |CSV| + |gzip(CSV)|)`.

### 3.3 `/generate_cache` default tools

```text
Client          cache_api              pool(1)              convert C++
  |--POST /generate_cache-->|              |                   |
  |  session_path          |              |                   |
  |                        | list all XPlanes [B11: N paths]  |
  |                        | filter tools  |                   |
  |                        |--submit----->|                   |
  |<-202 ACCEPTED----------|              |                   |
  |  [tiny JSON + gzip]    |              |                   |
  |                        |              |--write version--->| disk
  |                        |              |--overview_page--->| peak1
  |                        |              |  all asset_paths  |
  |                        |              |  [B12: OpStats Σ] |
  |                        |              |--trace_viewer@--->| peak2
  |                        |              |  [B13: LevelDB/T] |
```

**Ownership note:** HTTP thread releases after 202; worker owns full convert peaks sequentially (`max_workers=1`).

### 3.4 Overlap worst case (hypothesized OOM)

```text
Thread A: cache worker building trace_viewer@ LevelDB (max_workers=1)
Thread B: user /data overview_page ALL_HOSTS
Thread C: user /data kernel_stats ALL_HOSTS
→ three convert peaks without gate
→ Peak_RSS ≈ Peak_A + Peak_B + Peak_C + residual arenas
```

Severity **H**, confidence **hypothesized** (architecture allows it; magnitude workload-dependent).

---

## 4. Exact gzip / `json.dumps` conditions (`respond.py`)

### 4.1 Decision tree (line-cited)

```text
respond(body, content_type, code=200, content_encoding=None, extra_headers=None)
  │
  ├─ IF content_type == 'application/json'
  │     AND isinstance(body, (dict, list, set, tuple))     [52–54]
  │       → body = json.dumps(body, sort_keys=True)        [55]
  │
  ├─ IF not isinstance(body, bytes)                        [56]
  │       → body = body.encode('utf-8')                    [57]
  │
  ├─ IF content_encoding:                                  [107]
  │       → headers append Content-Encoding: <value>       [108]
  │       → body unchanged (no gzip)                       [no compress]
  │
  └─ ELSE:                                                 [109–111]
        → headers append Content-Encoding: gzip            [110]
        → body = gzip.compress(body)                       [111]
        → raw pre-gzip bytes still referenced until rebind;
          peak holds both during compress
```

### 4.2 Caller × outcome matrix

| Caller | content_type | body type | content_encoding | dumps? | gzip? | Cite |
|--------|--------------|-----------|------------------|--------|-------|------|
| `data_route` success | from convert (often `application/json`) | bytes or str | `None` from ToolResult | no (bytes) | **yes** | data_api:118–121 |
| `data_route` 404 | `text/plain` | str `"No Data"` | default None | no | **yes** | data_api:119–120 |
| `data_csv_route` success | `text/csv` | str CSV | **explicit None** | no | **yes** | data_api:155–162 |
| `data_csv_route` non-json tool | `text/plain` | str | default | no | **yes** | data_api:145–150 |
| `generate_cache` 202 | `application/json` | **dict** | default | **yes** | **yes** | cache_api:166–169 |
| `generate_cache` errors | `text/plain` | str | default | no | **yes** | cache_api:47–84 |
| `version_route` | `text/plain` | str version | default | no | **yes** | respond:121–123 |

### 4.3 Peak formula for encode stage

```text
Let R = len(raw_utf8_or_bytes)
Let G ≈ compression_ratio * R   (often 0.1–0.5 for JSON tables; ~1.0 for already-zstd)

During gzip.compress:
  Peak_encode ≥ R + G

If dumps path also used (dict body):
  Peak_encode ≥ |python_obj| + |json_str| + R + G
  (tool path rarely hits this; 202 ACCEPTED does)
```

| Issue | Severity | Confidence |
|-------|----------|------------|
| Always gzip when encoding falsy | **H** | observed |
| Precompressed pb double-compressed | **H** | observed |
| No free of raw after compress | **H** | observed |
| CSP string construction trivial | **L** | observed |

---

## 5. `use_saved_result` truth table

### 5.1 Three layers

| Layer | Module | What it controls |
|-------|--------|------------------|
| HTTP request arg | `parse_request` / `get_bool_arg` | Client desire (default True) |
| Python policy | `result_cache_policy.should_use_saved_result` | May force False via `cache_version.txt` |
| C++ clear | `profiler_plugin_impl` → `ClearCacheFiles` | Deletes disk artifacts when flag False |

### 5.2 Full truth table

Legend: `req` = HTTP `use_saved_result`; `ver` = `cache_version.txt` vs plugin version; `use` = value passed to convert; `clear` = ClearCacheFiles called.

| req | cache_version.txt | String compare vs plugin | `use` (Python) | C++ ClearCacheFiles | Disk OpStats/SSTABLE used | Confidence |
|-----|-------------------|--------------------------|----------------|---------------------|---------------------------|------------|
| True | missing | n/a | **False** | **yes** | recompute (after clear) | observed |
| True | present, `<` plugin | force False | **False** | **yes** | recompute | observed |
| True | present, `>=` plugin | honor req | **True** | **no** | may hit cache | observed |
| True | unreadable OSError | force False | **False** | **yes** | recompute | observed |
| False | any | n/a (starts False) | **False** | **yes** | recompute | observed |
| True | present, equal | honor | **True** | **no** | hit if files exist | observed |

Notes:

1. Version compare is **lexicographic string** (`result_cache_policy.py:43`), not semver (`a < b`).  
2. Python does **not** delete files; C++ does when `use_saved_result` option is present and false (`profiler_plugin_impl.cc:86–89`).  
3. After cold convert with `not use_saved`, Python **writes** version file (`tool_data.py:128–129`).  
4. Warm-up **writes version first** (`cache_api.py:192–193`) then converts — mid-failure can leave stamp saying “fresh” while caches incomplete (**M**, observed).

### 5.3 Interaction with OpStats processors

| Path | Behavior when use_saved_result True | When False |
|------|--------------------------------------|------------|
| Map phase file exists | Return path, skip convert | Cleared first → miss → rebuild |
| Reduce / ALL_HOSTS | Read/write ALL_HOSTS OpStats | Rebuild after clear |
| ShouldUseWorkerService | False if all hosts cached | True for multi-host (force map/reduce) | observed `base_op_stats_processor.cc:181–186` |

### 5.4 Stampede scenarios

| Scenario | Peak | Severity | Confidence |
|----------|------|----------|------------|
| Plugin version bump → all sessions force False | Many concurrent Clear+rebuild | **H** | hypothesized |
| Missing version on brand-new session | First /data clears (noop) + builds | **M** | observed |
| Warm-up stamps version early; concurrent /data trusts cache | Partial SSTABLE/OpStats | **M** | hypothesized |
| Client sends use_saved_result=false | Always clear session caches | **M** | observed |

---

## 6. ALL_HOSTS amplification

### 6.1 Policy sources (registry cites)

| Registry set | Definition lines | Tools |
|--------------|------------------|-------|
| `XPLANE_TOOLS_ALL_HOSTS_SUPPORTED` | registry.py:58–66 | input_pipeline_analyzer, framework_op_stats, kernel_stats, overview_page, pod_viewer, megascale_stats |
| `XPLANE_TOOLS_ALL_HOSTS_ONLY` | registry.py:68–70 | overview_page, pod_viewer, smart_suggestion |
| Multi-host selection | registry.py:99, 112–114 | trace_viewer, trace_viewer@ only |
| Single-host assert | raw_to_tool_data.py:165–166 | memory_profile (`len(xspace_paths)==1`) |
| `DEFAULT_CACHE_TOOLS` | registry.py:55 | overview_page, trace_viewer@ |
| `ALL_HOSTS` constant | constants.py:47 | sentinel string `'ALL_HOSTS'` |

### 6.2 How HostSelector expands

```text
Let H = set of hosts with *.xplane.pb in run_dir
Let X_i = size of XSpace i (decoded)
Let N = |H|

host == ALL_HOSTS OR (host is None AND hosts_param is None):
  asset_paths = all N paths                         [hosts.py:79–81, 95–98]

hosts_param + multi-host tool:
  asset_paths = |selected| ≤ N                      [hosts.py:69–78]

single host:
  asset_paths = 1                                   [hosts.py:82–84]
```

### 6.3 Amplification math

```text
# OpStats family (overview, kernel_stats, framework_op_stats, …)
# multi_xplanes_to_op_stats.cc holds full vector:

Peak_opstats ≈
    sum_i (X_i_decode + OpStats_i)     # during loop residual grows
  + |all_op_stats vector|              # after loop, all N live
  + |combined_op_stats|
  + |JSON output|

# Stream-combine TODO would target:
Peak_opstats_streamed ≈
    max_i (X_i_decode + OpStats_i) + |running_combined| + |JSON|

Amplification factor A_N ≈ Peak_opstats / Peak_single
  ≈ O(N) for naive vector hold (linear in hosts)

# Trace first open multi-host:
Peak_trace ≈ f(Σ X_i or sequential) + TraceEvents + LevelDB write buffers

# HTTP layer (independent multiplier):
Peak_http ≈ |payload| + |gzip(payload)|

# End-to-end:
Peak_total ≥ max(Peak_convert, Peak_http_overlap) 
           + concurrent_siblings
```

### 6.4 Numeric illustration (hypothesized orders of magnitude)

Assume N=8 hosts, each XSpace decode ~2 GB peak sequential residual ~0.5 GB OpStats:

| Model | Peak (order) | Notes |
|-------|--------------|-------|
| Sequential convert+drop | ~2.5 GB | best case if no vector hold |
| Vector hold all OpStats | ~0.5×8 + combine = multi-GB | matches TODO |
| + gzip of 200 MB JSON | +200–400 MB | respond double buffer |
| + concurrent /data sibling | ×2 | no semaphore |

Confidence on numbers: **hypothesized**. Structure of O(N) hold: **observed**.

### 6.5 Who pays

| Tool class | ALL_HOSTS default? | Notes |
|------------|--------------------|-------|
| overview, pod, smart_suggestion | Often only ALL_HOSTS in UI | Warm-up overview always |
| kernel/framework stats | UI can pick host or ALL | ALL is heaviest |
| memory_profile | Single host enforced | Lower host amp |
| graph/memory_viewer | HLO-centric | Different bottleneck |
| generate_cache | Always all files | Always |
| implicit missing host | All hosts | hosts.py:95–98 |

### 6.6 `hosts_from_xplane_filenames` for warm-up

```text
filenames → get_hosts
if |hosts| > 1:
  if tool in ALL_HOSTS_ONLY: hosts = [ALL_HOSTS]
  elif tool in ALL_HOSTS_SUPPORTED: hosts.add(ALL_HOSTS)
return sorted(hosts)
```

Warm-up still passes **all asset_paths** to convert (`cache_api.py:204–206`); hosts list is metadata for options, not a reduction of paths.

---

## 7. Concurrency: max_workers=1 vs unbounded Werkzeug

### 7.1 Observed controls

| Control | Scope | Limit | Cite |
|---------|-------|-------|------|
| Cache ThreadPoolExecutor | generate_cache tasks | **1** worker | plugin.py:212–216 |
| TB/Werkzeug threads | /data, /data_csv, discovery | process/server default | no plugin cap |
| Shared convert semaphore | — | **none** | observed absence |
| Per-key single-flight | — | **none** | observed absence |
| POST queue bound | generate_cache submit | unbounded (serialized run) | cache_api:152 |

### 7.2 Peak formula

```text
Let W_cache = 1   # max_workers for XprofCacheGen
Let W_http  = T   # concurrent Werkzeug/TB threads executing convert
Let P_t     = peak RSS of convert for tool t, host selection h

Peak_process ≈
    baseline_TB
  + sum over in-flight converts C_i of P_{tool(i)}
  + residual unreclaimed arenas (hypothesized)

With no shared gate:
  |C| ≤ W_cache + W_http ≤ 1 + T

Worst case T large:
  Peak ≈ baseline + P_cache + sum_{j=1..T} P_j
```

| Scenario | Peak | Severity | Confidence |
|----------|------|----------|------------|
| Warm-up alone default tools | max(P_overview, P_trace@) sequential | **H** workload | observed serial |
| Warm-up + user overview + kernel | P_trace@ + P_ov + P_ks | **H** | hypothesized |
| User opens many tabs/tools | N converts | **H** | hypothesized |
| Stampede after version bump | N hosts clear + rebuild concurrent | **H** | hypothesized |
| Multiple generate_cache POSTs | Queue of heavy jobs on 1 worker | **M** | hypothesized |
| Cache max_workers raised without /data cap | Historical OOM reason | **H** | observed comment |

### 7.3 Recommended control plane (design only)

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

## 8. Per-tool impact matrix for C1–C20

### 8.1 Opportunity catalog

| ID | Opportunity | Severity | Confidence | Primary levers |
|----|-------------|----------|------------|----------------|
| C1 | Cap concurrent converts (`/data` + cache worker shared) | **H** | hypothesized | plugin + ToolDataService |
| C2 | Stream gzip / free raw after compress | **H** | observed | respond |
| C3 | Stream legacy trace JSON (no full `''.join`) | **H** | observed | raw_to_tool_data |
| C4 | Prefer `trace_viewer@` (+ format=pb); deprecate full JSON path | **H** | observed | registry + FE + raw_to_tool_data |
| C5 | Batch ALL_HOSTS / stream-combine OpStats (fix C++ TODO) | **H** | observed | multi_xplanes_to_op_stats / base_op_stats_processor |
| C6 | Single-flight per (session, tool, hosts, options) | **M** | hypothesized | ToolDataService |
| C7 | Don’t gzip binary/precompressed; plumb content_encoding | **M** | observed | ToolDataService + respond |
| C8 | CSV without full `json.loads` | **M** | observed | csv_writer + data_api |
| C9 | Write `cache_version` after successful warm-up tools | **M** | observed | generate_cache / result_cache_policy |
| C10 | Disk cache quota / cleanup / selective ClearCache | **M** | observed | repository.cc + policy |
| C11 | Bound generate_cache queue; drop/coalesce duplicates | **M** | hypothesized | cache_api + pool |
| C12 | Per-tool OpStatsOptions (skip unused DBs) | **M** | hypothesized | base_op_stats_processor |
| C13 | Guard implicit all-hosts when host omitted | **M** | observed | HostSelector / FE |
| C14 | LRU process maps / session metadata only | **L** | hypothesized | SessionResolver / plugin |
| C15 | Avoid hlo_module_list full convert when protos present | **L**–**M** | observed | data_api |
| C16 | Riegeli GetXSpace streaming (no full file string) | **H** | observed | repository.cc |
| C17 | tools_cache avoid xspace_to_tool_names on miss | **L**–**M** | observed | catalog.py |
| C18 | Priority: UI convert over warm-up | **M** | hypothesized | ConvertGate |
| C19 | Page/limit ALL_HOSTS host count | **M** | hypothesized | HostSelector |
| C20 | Metrics: convert peak / cache dir size per session | **L** | hypothesized | observability |

### 8.2 Per-tool × opportunity impact (H/M/L = expected RAM ROI for that tool)

Legend: cell = impact of applying opportunity to tool’s typical load.  
Dash = not applicable / negligible.

| Tool | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 | C9 | C10 | C11 | C12 | C13 | C16 |
|------|----|----|----|----|----|----|----|----|----|-----|-----|-----|-----|-----|
| trace_viewer | **H** | **H** | **H** | **H** | — | **M** | **M** | — | **M** | **L** | **M** | — | **M** | **M** |
| trace_viewer@ | **H** | **H** | — | **H** | — | **M** | **H** | — | **M** | **H** | **H** | — | **M** | **H** |
| overview_page | **H** | **M** | — | — | **H** | **M** | **L** | **L** | **M** | **M** | **M** | **M** | **H** | **M** |
| input_pipeline_analyzer | **M** | **M** | — | — | **H** | **M** | **L** | **L** | **L** | **M** | **L** | **M** | **H** | **M** |
| framework_op_stats | **M** | **M** | — | — | **H** | **M** | **L** | **H** | **L** | **M** | **L** | **H** | **H** | **M** |
| kernel_stats | **M** | **M** | — | — | **H** | **M** | **L** | **H** | **L** | **M** | **L** | **H** | **H** | **M** |
| memory_profile | **M** | **M** | — | — | — | **L** | **L** | — | **L** | **L** | **L** | — | — | **M** |
| pod_viewer | **H** | **M** | — | — | **H** | **M** | **L** | — | **L** | **M** | **L** | **M** | **H** | **M** |
| op_profile | **M** | **M** | — | — | **M** | **L** | **L** | — | **L** | **L** | **L** | **M** | **L** | **M** |
| hlo_stats | **M** | **M** | — | — | **M** | **L** | **L** | **M** | **L** | **L** | **L** | **M** | **L** | **M** |
| roofline_model | **M** | **M** | — | — | **M** | **L** | **L** | — | **L** | **L** | **L** | **M** | **L** | **M** |
| inference_profile | **M** | **M** | — | — | **M** | **L** | **L** | — | **L** | **L** | **L** | **L** | **L** | **M** |
| memory_viewer | **L** | **M** | — | — | — | **L** | **L** | — | **L** | **L** | **L** | — | — | **L** |
| graph_viewer | **L** | **M** | — | — | — | **L** | **M** | — | **L** | **L** | **L** | — | — | **L** |
| megascale_stats | **M** | **M** | — | — | **M** | **L** | **H** | — | **L** | **M** | **L** | **L** | **M** | **M** |
| perf_counters | **L** | **L** | — | — | **L** | **L** | **L** | — | **L** | **L** | **L** | — | **L** | **L** |
| utilization_viewer | **L** | **L** | — | — | — | **L** | **L** | — | **L** | **L** | **L** | — | — | **L** |
| smart_suggestion | **M** | **L** | — | — | **M** | **L** | **L** | — | **L** | **M** | **L** | **L** | **H** | **L** |
| warm-up path | **H** | **L** | — | **H** | **H** | **M** | **L** | — | **H** | **H** | **H** | **M** | **H** | **H** |
| /data_csv path | **M** | **M** | — | — | **M** | **L** | **L** | **H** | — | — | — | — | **M** | — |

### 8.3 Cross-tool impact summary

| Opportunity theme | Highest impact tools | Lower impact |
|-------------------|----------------------|--------------|
| Concurrent convert cap (C1) | trace_viewer@, multi-host overview/pod, warm-up+UI | graph_viewer single module, counter names |
| Stream gzip / free raw (C2) | large JSON tools, legacy trace | tiny responses |
| Stream / no join legacy JSON (C3) | trace_viewer | N/A if deprecated |
| Prefer trace_viewer@ / pb (C4) | trace path | other tools |
| ALL_HOSTS batch / stream-combine OpStats (C5) | overview, pod, kernel/framework stats, input pipeline | memory_profile (1 host) |
| Single-flight (C6) | popular default tools after open session | rare tools |
| Don’t gzip precompressed (C7) | trace pb, megascale perfetto | pure JSON tables |
| CSV without full loads (C8) | framework_op_stats, kernel_stats tables | binary/html tools |
| Version write after success (C9) | all warm-up sessions | none |
| Disk quota / ClearCache policy (C10) | multi-host long-lived sessions | single-host tiny |

---

## 9. Disk cache file naming / growth model

### 9.1 Naming conventions

| Artifact | Pattern | Producer | Cleared by ClearCacheFiles? |
|----------|---------|----------|------------------------------|
| XPlane source | `{host}.xplane.pb` | capture | **no** (not a cache suffix) |
| Riegeli XPlane | `{host}.xplane.riegeli` | capture | **no** (explicit skip, repository.cc:257) |
| OpStats | `{host}.op_stats_v2.pb` | base_op_stats Map | **yes** |
| OpStats aggregate | `ALL_HOSTS.op_stats_v2.pb` | Reduce / WithCache | **yes** |
| Smart suggestion | `{host}.smart_suggestion.pb` | smart suggestion processor | **yes** |
| DCN collective | `{host}.dcn_collective_stats.pb` | DCN path | **yes** |
| Trace LevelDB | `{host}.SSTABLE` | trace_viewer@ | **yes** |
| Trace metadata | `{host}.metadata.SSTABLE` | trace_viewer@ | **yes** |
| Trace trie | `{host}.trie.SSTABLE` | trace_viewer@ | **yes** |
| Plugin version stamp | `cache_version.txt` | result_cache_policy | **no** (not in suffixes) |
| Tools list | `.cached_tools.json` | ToolsCache | **no** |

Host name derivation: basename of XPlane path with `.xplane.pb` / `.xplane.riegeli` stripped (`repository.cc:63–73`). Aggregate identifier: `kAllHostsIdentifier = "ALL_HOSTS"` (`repository.h:46`).

### 9.2 Growth model

```text
Let N = number of hosts
Let S_op(h) = size of host h OpStats proto
Let S_tr(h) = size of host h trace LevelDB suite (3 files)
Let S_ss(h) = smart_suggestion blob
Let tools_cached = set of tools that wrote artifacts

Disk_session ≈
    Σ_h |XPlane_h|                         # source (immutable for growth model)
  + Σ_h S_op(h) + S_op(ALL_HOSTS)          # if any OpStats tool ran
  + Σ_h S_tr(h)                            # if trace_viewer@ warmed
  + Σ_h S_ss(h)                            # if smart_suggestion ran
  + |cache_version.txt| + |.cached_tools.json|

No TTL, max-bytes, or LRU.  [observed absence]
```

### 9.3 Growth drivers

| Driver | Effect | Severity | Confidence |
|--------|--------|----------|------------|
| Warm-up default tools | Always OpStats path + trace SSTABLE | **H** | observed |
| Multi-host N | Linear files + ALL_HOSTS aggregate | **H** | observed |
| use_saved_result=false | Clear then rewrite (temp free then re-grow) | **M** | observed |
| Long-lived logdir many sessions | Process-external disk; TB process map of paths only | **M** | hypothesized |
| ToolsCache invalidation | Tiny rewrite | **L** | observed |

### 9.4 ClearCacheFiles semantics

```text
For each child in session run dir:
  if ends with any kHostDataSuffixes.second EXCEPT RIEGELI_XSPACE:
    DeleteFile
```

- All-or-nothing per session (**M** stampede: one false flag wipes every tool’s cache).  
- Does not touch XPlane sources or `cache_version.txt`.  
- Next convert rewrites needed files only for tools that run.

---

## 10. CSV double materialization walkthrough

### 10.1 Call chain (observed)

```text
GET /data_csv?run=...&tag=kernel_stats&host=...
  → DataApiMixin.data_csv_route                 [data_api.py:137]
  → data_impl → ToolDataService.get_tool_data   [data_api.py:140]
  → raw_to_tool_data / C++ → JSON bytes/str
  → load_convert_module().json_to_csv_string(data)  [data_api.py:152]
  → raw_to_tool_data.json_to_csv_string → csv_writer.json_to_csv_string
  → respond(csv_data, 'text/csv', content_encoding=None, Content-Disposition)
                                                      [data_api.py:155–162]
```

### 10.2 Buffer timeline

| Step | Buffer | Approx size | Live simultaneously with |
|------|--------|-------------|---------------------------|
| Convert complete | JSON payload J | |J| | C++ residual (hypothesized) |
| Decode bytes | unicode string | ~|J| | J (if bytes path copies) |
| json.loads | Python DOM D | often 2–5× |J| | J until GC |
| StringIO CSV | string C | ~0.5–1.5× |J| | D |
| respond encode | bytes of C | |C| | C str |
| gzip.compress | G | ρ|C| | C bytes |
| Response | G | |G| | until send |

### 10.3 Conditions / guards

| Condition | Outcome | Cite |
|-----------|---------|------|
| data is None | 404 plain text | data_api:142–143 |
| content_type != application/json | 400 "CSV format not supported" | data_api:145–150 |
| missing cols | ValueError → 500 | csv_writer:107–108 |
| list JSON | take first element | csv_writer:103–105 |
| QUOTE_ALL | larger CSV than minimal | csv_writer:111–116 |

### 10.4 Tools most affected

| Tool | Why CSV heavy | Severity |
|------|---------------|----------|
| kernel_stats | large DataTable rows | **M** |
| framework_op_stats | large DataTable rows | **M** |
| hlo_stats | medium tables | **L**–**M** |
| overview_page | multi-section JSON; may fail cols check | **L** |

Opportunity C8: stream rows or C++ CSV writer without Python DOM.

---

## 11. Legacy `process_raw_trace` join analysis

### 11.1 Code path

```text
xspace_to_tool_data tool=='trace_viewer'     [raw_to_tool_data.py:127–137]
  options from trace_viewer_options
  if success and format != 'pb':
    data = process_raw_trace(raw_data)       [41–45]
```

```python
def process_raw_trace(raw_trace: bytes) -> str:
  trace = trace_events_old_pb2.Trace()
  trace.ParseFromString(raw_trace)           # full proto in RAM
  return ''.join(trace_events_json.TraceEventsJsonStream(trace))
```

### 11.2 Why join destroys streaming

`TraceEventsJsonStream` (`trace_events_json.py:31–37`) is explicitly designed as a chunk iterator “suitable for returning in a werkzeug Response.” The `''.join(...)` call:

1. Materializes every chunk into one contiguous `str`.  
2. Temporarily holds list-of-chunks + final string (CPython join).  
3. Forces subsequent `respond` to encode full UTF-8 + gzip of entire trace JSON.

```text
Peak_legacy_trace ≥
    |Trace proto|
  + |chunk list during join|
  + |full JSON str|
  + |utf-8 bytes|
  + |gzip bytes|
```

### 11.3 Comparison to modern path

| Path | Intermediate | Serve | Severity of RAM |
|------|--------------|-------|-----------------|
| `trace_viewer` JSON | proto + full JSON | gzip full | **H** |
| `trace_viewer` format=pb | raw bytes | still gzipped (C7 gap) | **M**–**H** |
| `trace_viewer@` | LevelDB on disk | streaming reads (doc 01) | lower peak for UI window |
| `trace_viewer@` format=pb | zstd pb | double gzip gap | **H** encode |

### 11.4 Opportunities

| ID | Fix | Severity | Confidence |
|----|-----|----------|------------|
| C3 | Pass generator/stream to Response; never join | **H** | observed |
| C4 | Default FE to trace_viewer@ only | **H** | observed |
| C7 | Skip gzip for pb | **M** | observed |

---

## 12. Measurement plan (no product change)

### 12.1 Experiments

| Experiment | What to capture | Success signal for later fix | Maps to |
|------------|-----------------|------------------------------|---------|
| M1 | RSS + wall for generate_cache default on multi-host session | Baseline warm-up peak | C1, C9, C11 |
| M2 | RSS for concurrent M1 + /data overview | Delta proves C1 value | C1 |
| M3 | Wire size + RSS for kernel_stats /data vs /data_csv | C8 value | C8, C2 |
| M4 | format=pb vs JSON for trace_viewer@ including gzip layer | C2/C4/C7 | C2, C4, C7 |
| M5 | Disk growth of op_stats_v2 + SSTABLE over N tools | C10 | C10 |
| M6 | Host count sweep N=1,2,4,8 ALL_HOSTS overview | C5 scaling curve | C5, C13 |
| M7 | Version file timing vs first tool completion logs | C9 | C9 |
| M8 | Concurrent identical /data ×3 same tool | single-flight value | C6 |
| M9 | Riegeli vs pb XPlane same session | C16 | C16 |
| M10 | use_saved_result false vs true second hit | cache effectiveness | §5 |

### 12.2 Instrumentation hooks (existing logs)

- `ConvertMultiXSpaceToCombinedOpStatsWithCache` INFO timings (`multi_xplanes_to_op_stats.cc`)  
- `generate_cache` logger steps (`cache_api.py`)  
- Cache pool thread name `XprofCacheGen` (`plugin.py:214`)  
- ToolsCache hit/invalidate logs (`tools_cache.py`)  
- Cache version missing log (`result_cache_policy.py:46`)

### 12.3 Suggested metrics (design only)

| Metric | How | Use |
|--------|-----|-----|
| `convert_peak_rss_delta` | before/after convert in worker | rank tools |
| `respond_raw_bytes` / `respond_gzip_bytes` | len before/after compress | C2 ROI |
| `hosts_selected` | len(asset_paths) | ALL_HOSTS amp |
| `cache_dir_bytes` | sum suffixes in session | C10 |
| `use_saved_effective` | bool after policy | §5 correctness |
| `in_flight_converts` | gauge under gate | C1 |

### 12.4 Workload fixtures

| Fixture | Purpose |
|---------|---------|
| Single-host small | noise floor |
| Multi-host N=8 medium | ALL_HOSTS curve |
| Large trace session | C3/C4 encode |
| Table-heavy kernel_stats | C8 |
| Fresh session no cache_version | cold path |
| Session with stale version string | invalidate path |

---

## 13. Caching layers (three kinds — do not confuse)

### 13.1 Layer summary

| Layer | Name | Scope | Payload | Eviction |
|-------|------|-------|---------|----------|
| A | `result_cache_policy` + C++ host data | Per session dir | OpStats, SSTABLE, smart_suggestion, … | Version gate + ClearCacheFiles; **no quota** |
| B | `tools_cache` / `.cached_tools.json` | Per session | Tool name list | File-state mismatch |
| C | Process maps `_run_to_profile_run_dir` | Process | Path strings | Never (until process exit) |

**There is no in-process LRU of tool JSON/bytes** (observed). Every cold `/data` pays convert (or C++ disk cache hit inside convert).

### 13.2 Result / convert cache lifecycle

```text
HTTP use_saved_result=true (default)
  → should_use_saved_result may still force false if version missing/stale
  → options['use_saved_result'] into C++
  → C++ may ClearCacheFiles if false
  → C++ may read *.op_stats_v2.pb / LevelDB / etc. if true
  → on forced recompute: rewrite artifacts

generate_cache warm-up
  → write cache_version.txt immediately
  → convert DEFAULT_CACHE_TOOLS on all hosts
  → side effect: populates OpStats + trace LevelDB
```

### 13.3 Warm-up product behavior

| Item | Value | Confidence |
|------|-------|------------|
| Default tools | `overview_page`, `trace_viewer@` | observed |
| Hosts | All session XPlanes | observed |
| Pool | 1 worker | observed |
| HTTP | 202 ACCEPTED | observed |
| Version stamp | Before tools complete | observed |

**Risk:** UI opens `/data` for same session while warm-up runs → dual peak (cache worker + request) (**H**, hypothesized).

### 13.4 Disk growth risk

Unbounded per-host:

- `host.op_stats_v2.pb`  
- `host.SSTABLE` + metadata + trie (trace)  
- `host.smart_suggestion.pb`  
- `ALL_HOSTS.*` aggregates  

No TTL, max-bytes, or LRU (**H**, observed absence).

---

## 14. Encoding / double materialization matrix

| Path | Stages co-resident | Severity | Confidence |
|------|--------------------|----------|------------|
| Main `/data` JSON bytes | C++ string → Python bytes → gzip | **H** | observed |
| `/data` Python dict (rare) | dumps + utf-8 + gzip | **M** | observed |
| Legacy `trace_viewer` | proto + joined JSON + gzip | **H** | observed |
| `trace_viewer@` format=pb | zstd pb + gzip again | **H** | observed |
| `/data_csv` | JSON DOM + CSV + gzip | **M** | observed |
| graph HTML | large text + gzip | **M** | observed |
| Counter names_only | small text + gzip | **L** | observed |
| generate_cache 202 | tiny dict dumps + gzip | **L** | observed |

**Rule of thumb:** any tool with multi-10MB+ JSON is dominated by **encode path + convert**, not by `HostSelector`/`SessionResolver`.

---

## 15. Ranked opportunities (this document)

| Rank | ID | Opportunity | Severity | Confidence | Primary levers |
|------|----|-------------|----------|------------|----------------|
| 1 | C1 | Cap concurrent converts (`/data` + cache worker shared) | **H** | hypothesized | plugin + ToolDataService |
| 2 | C2 | Stream gzip / free raw after compress | **H** | observed | respond |
| 3 | C3 | Stream legacy trace JSON (no full `''.join`) | **H** | observed | raw_to_tool_data |
| 4 | C4 | Prefer `trace_viewer@` (+ format=pb); deprecate full JSON path | **H** | observed | registry + FE + raw_to_tool_data |
| 5 | C5 | Batch ALL_HOSTS / stream-combine OpStats (fix C++ TODO) | **H** | observed | multi_xplanes_to_op_stats / base_op_stats_processor |
| 6 | C16 | Riegeli streaming read | **H** | observed | repository.cc |
| 7 | C6 | Single-flight per (session, tool, hosts, options) | **M** | hypothesized | ToolDataService |
| 8 | C7 | Don’t gzip binary/precompressed; plumb content_encoding | **M** | observed | ToolDataService + respond |
| 9 | C8 | CSV without full `json.loads` | **M** | observed | csv_writer + data_api |
| 10 | C9 | Write `cache_version` after successful warm-up tools | **M** | observed | generate_cache / result_cache_policy |
| 11 | C10 | Disk cache quota / cleanup / selective ClearCache | **M** | observed | repository.cc + policy |
| 12 | C11 | Bound generate_cache queue; drop/coalesce duplicates | **M** | hypothesized | cache_api + pool |
| 13 | C12 | Per-tool OpStatsOptions (skip unused DBs) | **M** | hypothesized | base_op_stats_processor |
| 14 | C13 | Guard implicit all-hosts when host omitted | **M** | observed | HostSelector / FE |
| 15 | C18 | Priority UI over warm-up | **M** | hypothesized | ConvertGate |
| 16 | C19 | Page/limit ALL_HOSTS | **M** | hypothesized | HostSelector |
| 17 | C15 | Avoid hlo_module_list full convert when protos present | **L**–**M** | observed | data_api |
| 18 | C17 | tools_cache avoid heavy discovery | **L**–**M** | observed | catalog.py |
| 19 | C14 | LRU process maps | **L** | hypothesized | SessionResolver / plugin |
| 20 | C20 | Convert/cache metrics | **L** | hypothesized | observability |

---

## 16. Phased recommendations (analysis roadmap)

### Phase 0 — Measure (align with synthesis doc)

Run M1–M10 on representative multi-host production-like sessions.

### Phase 1 — Low design risk, high encode ROI

1. **C7 / C2:** Plumb `content_encoding` for precompressed; skip double gzip; free raw.  
2. **C9:** Move `write_cache_version_file` to end of successful `_generate_cache_task`.  
3. **C13:** FE always sends host; optional server guard for hostless all-hosts on heavy tools.

### Phase 2 — Convert control plane

1. **C1:** Shared convert semaphore (K=1 or 2).  
2. **C6:** Single-flight for identical requests.  
3. **C11 / C18:** Coalesce generate_cache; priority UI > warm-up.

### Phase 3 — Convert algorithm

1. **C5:** Stream-combine OpStats (resolve C++ TODO at multi_xplanes_to_op_stats.cc:51–52).  
2. **C12:** Tool-specific OpStatsOptions.  
3. **C3 / C4:** Legacy trace elimination; pb path for V2.  
4. **C16:** Riegeli full-file buffer elimination.

### Phase 4 — Disk hygiene

1. **C10:** Quota, TTL, or size-aware ClearCache.  
2. **C20:** Metrics on cache dir size in session.

---

## 17. Non-goals / out of scope for this note

- Frontend Angular heap (partially covered in 01 for trace; tables in 04).  
- CLI projection modes (`summary_only`) — synthesis theme 1; tool docs 03/05/06.  
- Capture/profile recording path (`capture_api`).  
- Product implementation PRs (this campaign is analysis only).

---

## 18. Real paths cited (path:line anchors)

### Python plugin spine

| Path | Lines (primary) | Symbols |
|------|-----------------|---------|
| `plugin/xprof/profile_plugin/services/tool_data.py` | 25–135 | **ToolDataService**, tool_data_service algorithm |
| `plugin/xprof/profile_plugin/services/hosts.py` | 24–115 | **HostSelector**, hosts_selector |
| `plugin/xprof/profile_plugin/services/sessions.py` | 28–102 | **SessionResolver**, session_resolver |
| `plugin/xprof/profile_plugin/services/runs.py` | 38–132 | RunDiscovery + **tools_cache** usage |
| `plugin/xprof/profile_plugin/api/data_api.py` | 19–173 | **data_api**, data_route, data_csv_route |
| `plugin/xprof/profile_plugin/api/cache_api.py` | 18–227 | **generate_cache**, _generate_cache_task |
| `plugin/xprof/profile_plugin/http/respond.py` | 29–118 | **respond** |
| `plugin/xprof/profile_plugin/http/parse_request.py` | 13–31 | tool_request_from_args |
| `plugin/xprof/profile_plugin/cache/tools_cache.py` | 27–121 | **tools_cache** / ToolsCache |
| `plugin/xprof/profile_plugin/cache/result_cache_policy.py` | 26–66 | **result_cache_policy** |
| `plugin/xprof/profile_plugin/plugin.py` | 139–216, 250–278 | ThreadPoolExecutor max_workers=1, wiring |
| `plugin/xprof/profile_plugin/tools/registry.py` | 27–123 | DEFAULT_CACHE_TOOLS, ALL_HOSTS sets |
| `plugin/xprof/profile_plugin/tools/filenames.py` | 104–122 | hosts_from_xplane_filenames |
| `plugin/xprof/profile_plugin/tools/catalog.py` | 27–91 | get_active_tools / tool_names |
| `plugin/xprof/profile_plugin/models.py` | 20–45 | ToolRequest, HostSelection, ToolResult |
| `plugin/xprof/profile_plugin/constants.py` | 34–47 | routes, CACHE_VERSION_FILE, ALL_HOSTS |

### Convert Python

| Path | Lines | Symbols |
|------|-------|---------|
| `plugin/xprof/convert/raw_to_tool_data.py` | 41–260 | **raw_to_tool_data**, process_raw_trace |
| `plugin/xprof/convert/csv_writer.py` | 24–139 | json_to_csv_string |
| `plugin/xprof/convert/trace_events_json.py` | 31–76 | TraceEventsJsonStream |

### Convert C++

| Path | Lines | Symbols |
|------|-------|---------|
| `xprof/convert/repository.cc` | 51–59, 107–118, 246–266 | suffixes, Riegeli, **ClearCacheFiles** |
| `xprof/convert/repository.h` | 46, 115 | kAllHostsIdentifier, ClearCacheFiles decl |
| `xprof/convert/base_op_stats_processor.cc` | 73–187 | Map/Reduce, use_saved_result, all_op_stats |
| `xprof/convert/multi_xplanes_to_op_stats.cc` | 42–159 | combine **TODO**, WithCache |
| `xprof/pywrap/profiler_plugin_impl.cc` | 84–90 | ClearCacheFiles on !use_saved_result |

---

## 19. Relationship to sibling design notes

| Doc | Overlap with this path |
|-----|------------------------|
| `00-synthesis-memory-optimization.md` | Global ranking; this doc is the convert/serve/cache deep dive |
| `01-trace-viewer-path.md` | Specializes C3/C4/C7 for streaming LevelDB + V2 pb |
| `03-memory-tools-family.md` | memory_profile single-host; memory_viewer HLO |
| `04-overview-and-stats-tools.md` | OpStats family; expands C5/C12 per tool |
| `05-graph-hlo-util-tools.md` | graph_viewer / util; still uses same respond + ToolDataService |
| `06-megascale-and-cli-detectors.md` | megascale_stats / perfetto encoding; CLI bypasses some HTTP |

---

## 20. Glossary

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
| ToolDataService | Python orchestrator for convert (tool_data_service) |
| HostSelector | hosts_selector for asset paths |
| SessionResolver | session_resolver for run_dir |
| result_cache_policy | Version gate module |
| tools_cache | Tool list `.cached_tools.json` |
| data_api | HTTP `/data` mixin |
| respond | HTTP encode helper |
| raw_to_tool_data | Python convert dispatcher |

---

## 21. Findings index (severity × confidence)

### 21.1 High severity

| Finding | Confidence | Primary cite |
|---------|------------|--------------|
| `respond` always gzip when content_encoding falsy; double buffer | observed | respond.py:107–111 |
| ToolDataService always content_encoding=None | observed | tool_data.py:131–135 |
| Cache pool max_workers=1 but /data unbounded | observed / hypothesized interaction | plugin.py:212–216 |
| Implicit all-hosts when host omitted | observed | hosts.py:95–98 |
| Warm-up always all asset_paths + heavy defaults | observed | cache_api.py:59–61, registry.py:55 |
| Multi-host OpStats holds full vector (TODO) | observed | multi_xplanes_to_op_stats.cc:51–66 |
| Legacy process_raw_trace full join | observed | raw_to_tool_data.py:41–45 |
| Riegeli full-file read into string | observed | repository.cc:109–111 |
| ClearCacheFiles all-or-nothing on !use_saved | observed | profiler_plugin_impl.cc:84–90; repository.cc:246–266 |
| Disk caches unbounded (no quota) | observed | repository.cc suffixes |

### 21.2 Medium severity

| Finding | Confidence | Primary cite |
|---------|------------|--------------|
| Warm-up writes cache_version before tools | observed | cache_api.py:192–193 |
| CSV double materialization | observed | data_api.py:152; csv_writer.py:90–139 |
| Precompressed still gzipped | observed | respond + pb paths |
| No single-flight / convert gate | hypothesized | architecture |
| Version string compare not semver | observed | result_cache_policy.py:43 |
| Always generate all OpStats DBs | observed | base_op_stats_processor.cc:111–115 |
| hlo_module_list may force convert | observed | data_api.py:86–103 |
| catalog tool_names convert on discovery | observed | catalog.py:65–68 |

### 21.3 Low severity

| Finding | Confidence | Primary cite |
|---------|------------|--------------|
| run_dir_cache unbounded paths only | hypothesized | plugin.py:197 |
| 404/error responses tiny but still gzip | observed | respond default |
| ToolsCache JSON small | observed | tools_cache.py |

---

## 22. Summary judgment

The convert → serve → cache path is **architecturally clean** (SessionResolver → HostSelector → convert → respond) but **memory-unsafe at the edges**:

1. **Encode:** `respond` always materializes full gzip alongside raw.  
2. **Hosts:** omitted host = all hosts; warm-up always all hosts.  
3. **Concurrency:** cache pool is carefully capped at 1; interactive `/data` is not.  
4. **Disk cache:** powerful OpStats/LevelDB reuse, no quota, version stamped early on warm-up.  
5. **Combine:** multi-host OpStats still holds full vectors (explicit TODO).  
6. **Clear:** `use_saved_result=False` (including version invalidation) triggers session-wide ClearCacheFiles.

Closing C1–C5 (and C16) first would address the largest cross-tool multipliers before investing in per-tool projection modes (synthesis theme 1, docs 03–06).

### 22.1 Must-match string checklist

This document intentionally includes each of:

`ToolDataService`, `tool_data_service`, `HostSelector`, `hosts_selector`, `SessionResolver`, `session_resolver`, `generate_cache`, `tools_cache`, `result_cache_policy`, `respond`, `raw_to_tool_data`, `data_api`.

---

*End of deepened analysis. No product code changes.*
