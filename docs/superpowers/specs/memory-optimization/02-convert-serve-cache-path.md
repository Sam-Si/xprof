# Memory Optimization: Convert → Serve → Cache Path

**Date:** 2026-07-13  
**Status:** Research / design (no product code)  
**Scope:** Shared pipeline for all XPlane-backed UI tools

## 1. Shared pipeline stages + buffer ownership

```text
HTTP /data | /data_csv | /generate_cache
  → api/data_api.py | api/cache_api.py
  → ToolDataService.get_tool_data  OR  cache_api._generate_cache_task
       sessions.resolve_run_dir
       result_cache_policy.should_use_saved_result
       build_tool_params
       HostSelector.select → asset_paths[]
       convert.xspace_to_tool_data → pywrap C++
  → ToolResult(data, content_type, content_encoding=None)
  → respond(): optional json.dumps → utf-8 → gzip.compress (default)
```

| Stage | Owner | Peak buffers | Lifetime |
|-------|-------|--------------|----------|
| Host selection | `services/hosts.py` | N path strings | Call |
| Convert C++ | pywrap / processors | XPlanes + intermediates + output string | Convert call |
| Legacy trace JSON | `process_raw_trace` | Trace proto + joined JSON str | Convert |
| HTTP | `http/respond.py` | body + gzip(body) coexist | Response build |
| Cache gen pool | `plugin.py` ThreadPoolExecutor(max_workers=**1**) | Full convert peak for largest tool | Worker |

## 2. Caching layers

1. **Result/convert cache (C++ disk)** — `cache_version.txt` gate; per-host `*.op_stats.pb`; `ClearCacheFiles` when `use_saved_result=false`. No size quota/eviction (**H** growth risk).
2. **Tools list cache** — `.cached_tools.json` small; miss may call `xspace_to_tool_names`.
3. **Process maps** — `_run_to_profile_run_dir`; no in-process LRU of payloads.

**Warm-up:** `DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')`; all session XPlanes; version file may write **before** tools finish (**M**).

## 3. Encoding / double materialization

- Main `/data` path: C++ already emits JSON **bytes** → skip dumps; still **gzip.compress** holds raw+gzip.
- `/data_csv`: full `json.loads` of payload → CSV → gzip again.
- Legacy `trace_viewer`: `''.join(TraceEventsJsonStream)` kills streaming benefit.
- Binary/pb still gzipped when `content_encoding is None`.

## 4. ALL_HOSTS amplification

- `host=ALL_HOSTS` or missing host → **all** XPlanes (`hosts.py`).
- `XPLANE_TOOLS_ALL_HOSTS_ONLY`: overview_page, pod_viewer, smart_suggestion.
- Multi-host selection only for trace_viewer / trace_viewer@.
- Warm-up always full asset_paths list.
- Peak ≈ Σ X_i + intermediates; trace-like T can ≫ max X_i.

## 5. Concurrency risks

- Cache pool max_workers=1 (**observed OOM mitigation**).
- **No shared semaphore** with foreground `/data` → concurrent peak = cache worker + N request converts (**H**, hypothesized).
- No single-flight lock per (session, tool, hosts).
- Unbounded POST queue of warm-ups (serialized but delayed peaks).

## 6. Ranked opportunities

| Rank | Opportunity | Severity | Confidence |
|------|-------------|----------|------------|
| 1 | Cap concurrent converts (/data + cache) | **H** | hypothesized |
| 2 | Stream gzip / free raw after compress | **H** | observed |
| 3 | Stream legacy trace JSON (no full join) | **H** | observed |
| 4 | Prefer trace_viewer@; deprecate full JSON path | **H** | observed |
| 5 | Batch ALL_HOSTS / lower resident set | **H** | hypothesized |
| 6 | Single-flight per (session, tool, hosts) | **M** | hypothesized |
| 7 | Don't gzip binary/precompressed | **M** | observed |
| 8 | CSV without full json.loads | **M** | observed |
| 9 | Write cache_version after successful tools | **M** | observed |
| 10 | Disk cache quota/cleanup | **M** | observed |

## 7. Cross-tool impact

| Opportunity | Highest impact | Lower |
|-------------|----------------|-------|
| Concurrent convert cap | trace_viewer@, multi-host overview/pod | graph_viewer, counter names |
| Stream JSON | legacy trace_viewer | small tools |
| ALL_HOSTS batching | overview, pod, megascale, stats tools | memory_profile (1 host) |
| Gzip policy | large JSON tools | tiny responses |
| CSV double parse | framework_op_stats, kernel_stats, tables | binary/html |

## 8. Real paths cited

- `plugin/xprof/convert/raw_to_tool_data.py`
- `plugin/xprof/convert/trace_events_json.py`
- `plugin/xprof/convert/csv_writer.py`
- `plugin/xprof/profile_plugin/services/tool_data.py`
- `plugin/xprof/profile_plugin/services/hosts.py`
- `plugin/xprof/profile_plugin/api/data_api.py`
- `plugin/xprof/profile_plugin/api/cache_api.py`
- `plugin/xprof/profile_plugin/http/respond.py`
- `plugin/xprof/profile_plugin/cache/tools_cache.py`
- `plugin/xprof/profile_plugin/cache/result_cache_policy.py`
- `plugin/xprof/profile_plugin/tools/registry.py`
- `plugin/xprof/profile_plugin/plugin.py`
- `xprof/pywrap/profiler_plugin_impl.cc`
- `xprof/convert/repository.cc`
- `xprof/convert/base_op_stats_processor.cc`
