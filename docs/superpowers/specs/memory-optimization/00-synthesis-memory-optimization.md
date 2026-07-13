# Synthesis: XProf Memory Optimization Opportunities

**Date:** 2026-07-13  
**Status:** Analysis / design only — **no product code changes in this campaign**  
**Audience:** Engineers prioritizing RAM-sensitive work as memory costs rise  
**Companion docs:** `01`–`06` in this directory  
**Severity labels:** **H** | **M** | **L** only (never "Critical")  
**Confidence labels:** **observed** | **hypothesized** only

This document is the **portfolio synthesis**. It does not re-derive every tool path; it inventories surfaces, pulls top wins from the deep dives (especially Trace Viewer), ranks **cross-cutting themes**, and sequences a **measurement-first** roadmap with explicit dependencies. For code-level `path:line` evidence, open the linked design note.

---

## How to read this synthesis and the rest of the set

### Reading order (recommended)

| Order | Doc | Approx depth | When to open |
|-------|-----|--------------|--------------|
| 0 | **This file (`00`)** | ≥450 lines | Always first — portfolio, themes, phases |
| 1 | `01-trace-viewer-path.md` | **~1600 lines** | Browser + multi-host first-open + LevelDB + V2/PB |
| 2 | `02-convert-serve-cache-path.md` | **~1600 lines** | Shared `/data` → convert → gzip → cache for **all** tools |
| 3 | `03-memory-tools-family.md` | **~1200 lines** | memory_profile / memory_viewer / peak CLI |
| 4 | `04-overview-and-stats-tools.md` | family note | OpStats fan-out (overview, kernel, framework, …) |
| 5 | `05-graph-hlo-util-tools.md` | family note | graph, inference, util, smart, perf, pod |
| 6 | `06-megascale-and-cli-detectors.md` | **≥800 lines** | Perfetto double-gzip, detectors, get_overview CLI |

### Conventions shared by all notes

| Convention | Meaning |
|------------|---------|
| Severity **H** | High peak-RAM or multi-MiB wire / N× full convert for a small need |
| Severity **M** | Measurable waste; secondary path or bounded amplification |
| Severity **L** | Small absolute, polish, or optional path |
| Confidence **observed** | Confirmed in source with cites in the deep-dive note |
| Confidence **hypothesized** | Plausible from architecture; needs measurement before large bets |
| No product code | This campaign ships analysis docs only |

### How themes map to docs

```text
00 synthesis (this file)
  ├─ Theme: projection modes ──────────── 03, 05, 06 (CLI) + 04 (OpStats views)
  ├─ Theme: concurrent convert cap ────── 02
  ├─ Theme: encoding hygiene ──────────── 01, 02, 06
  ├─ Theme: ALL_HOSTS combine ─────────── 02, 04, 05
  ├─ Theme: disk cache / single-flight ── 02
  ├─ Theme: stream / viewport / top-K ─── 01, 04, 05
  ├─ Theme: double materialization ────── 02, 05
  ├─ Theme: FE virtualization ─────────── 01, 04
  ├─ Theme: dead fields / dual sorts ──── 03, 04
  └─ Theme: HLO amortization ──────────── 03, 05, 06
```

### What "synthesis" is not

- Not a substitute for `01` when changing Trace Viewer LevelDB or V2 SoA.  
- Not a substitute for `02` when changing `respond()` or cache policy.  
- Not an implementation PR checklist with patches — phases are an **analysis roadmap**.

---

## (a) Tool inventory

### Shared data path (all tools)

```text
On-disk XPlane / HLO
  → SessionResolver + HostSelector
  → convert (raw_to_tool_data / pywrap / C++ processors)
  → ToolDataService (+ result cache version policy)
  → HTTP respond (JSON/bytes + default gzip when content_encoding is None)
  → Angular tool component (or CLI client / MCP)
```

**Sources (registry spine):** `plugin/xprof/profile_plugin/tools/registry.py`, `services/tool_data.py`, `api/data_api.py`, `http/respond.py`, `convert/raw_to_tool_data.py`. Full buffer-ownership diagrams: **doc 02**.

### XPLANE_TOOLS (registry)

| Tool | Design note | Host policy highlight |
|------|-------------|------------------------|
| `trace_viewer` | **01** (~1600) | multi-host selection; legacy full JSON |
| `trace_viewer@` | **01** | multi-host; LevelDB streaming; in `DEFAULT_CACHE_TOOLS` |
| `overview_page` | **04** (+ CLI in **06**) | ALL_HOSTS only when multi-host; in `DEFAULT_CACHE_TOOLS` |
| `input_pipeline_analyzer` | **04** | ALL_HOSTS supported |
| `framework_op_stats` | **04** | ALL_HOSTS supported |
| `kernel_stats` | **04** | ALL_HOSTS supported |
| `memory_profile` | **03** (~1200) | single-host enforced |
| `pod_viewer` | **05** | ALL_HOSTS only |
| `op_profile` | **04** / CLI top-ops **05/06** | not in ALL_HOSTS UI sets |
| `hlo_stats` | **04** | not in ALL_HOSTS UI sets |
| `roofline_model` | **04** | not in ALL_HOSTS UI sets |
| `inference_profile` | **05** | multi-host parallel |
| `memory_viewer` | **03** | HLO (+ XPlane extract) |
| `graph_viewer` | **05** (+ neighborhood in **06**) | HLO_TOOLS |
| `megascale_stats` | **06** (≥800) | ALL_HOSTS supported; Perfetto via `perfetto=true` |
| `perf_counters` | **05** | multi-host events |
| `utilization_viewer` | **05** | single-host enforced |
| `smart_suggestion` | **05** | ALL_HOSTS only; multi-tool cold |

### Warm-up pair (always pay attention)

```text
DEFAULT_CACHE_TOOLS = ('overview_page', 'trace_viewer@')
```

Cold session open therefore tends to pay:

1. Multi-host **overview** OpStats combine + full Overview JSON.  
2. Multi-host **trace_viewer@** preprocess → LevelDB (and optional DCN).

Everything else is on-demand — but CLI/MCP agents can pull many tools without UI expand gates.

### Cross-cutting surfaces

| Surface | Design note |
|---------|-------------|
| convert → serve → cache | **02** (~1600) |
| tools_cache / generate_cache | **02** |
| megascale_perfetto FE | **06** |
| stack_trace_page / stack_trace_snippet | **01** (adjacent) |
| CLI / MCP tools | **03**, **05**, **06** |
| HTTP `respond()` gzip default | **02** (shared); special cases **01**, **06** |

### CLI / MCP tools

| CLI | Design note | Dominant waste (one line) |
|-----|-------------|---------------------------|
| get_memory_profile_tool | **03** | Full memory_profile UI for summary needs |
| get_peak_allocations_tool | **03** | N+1 module / maxHeap path |
| get_overview_tool | **06** | Full Overview JSON → ~30 scalars |
| get_graph_viewer_tool | **05** | Full graph payloads |
| get_utilization_viewer_tool | **05** | Full util UI convert |
| get_smart_suggestions_tool | **05** | Multi-tool cold + JSON round-trip |
| get_kpi_metrics_tool | **05** | Projection gap |
| get_top_hlo_ops_tool | **05** / **06** | Full hlo_op_profile UI proto |
| detect_layout_mismatch_copies_tool | **06** | All-module HLO + all copies BFS |
| detect_unfused_reshapes_tool | **06** | N× full graph_viewer text |
| detect_unnecessary_convert_reduce_tool | **06** | All-module HLO (candidate logic) |

### Inventory completeness check

| Category | Covered by |
|----------|------------|
| Interactive timeline | 01 |
| Shared HTTP/convert/cache | 02 |
| Runtime + static memory UIs | 03 |
| OpStats-backed stats pages | 04 |
| Graph / util / pod / smart / perf | 05 |
| Megascale + leftover detectors + overview CLI | 06 |
| Portfolio ranking + phases | **00 (here)** |

---

## (b) Trace-viewer path (deep dive summary)

Full detail: **`01-trace-viewer-path.md` (~1600 lines)** — deepest single-tool-family note. Doc 02 is deeper on the **shared** convert/serve spine (~1600) but is multi-tool.

### Critical path memory story

1. **First open multi-host:** Σ-aware host loop → full TraceEventsContainer → LevelDB write (large SSTABLE blocks) — highest **server** peak for interactive sessions.  
2. **Steady-state:** LevelDB filtered load by time span + resolution — good design, undermined if visibility floors / `resolution=0` / overfetch.  
3. **Wire:** Default JSON + **gzip always** in `respond()` when encoding unset; `format=pb` + zstd exists but production V2 URL wiring for PB remains a gap (see 01).  
4. **Browser V2:** JSON object graph + WASM copy + SoA with per-event `entry_args` (TODO to drop in 01).  
5. **Legacy Catapult:** merges tiles into unbounded model.  
6. **Adjacent:** optional `ProcessMegascaleDcn` on preprocess; stack_trace bridge holds separate caches (01).

### Top trace wins (ranked — updated for deepened 01)

IDs below follow the Trace Viewer note's opportunity catalog (T1…); severities are **H|M|L** only.

| Rank | ID | Win | Severity | Confidence | Why still #1–N after 01 deepen |
|------|-----|-----|----------|------------|--------------------------------|
| 1 | T1 | Ship `format=pb` end-to-end for V2 | **H** | observed gap | Removes JSON object-graph explosion on wire + parse |
| 2 | T2 | Do not gzip already-zstd PB | **H** | observed | Same encoding class as Perfetto MP-1 (doc 06) |
| 3 | T3 | Slim `entry_args` / lazy args | **H** | observed TODO | Browser RSS after PB still dominated by args SoA |
| 4 | T4 | Bound multi-host first-open | **H** | observed | Cap/host batching; avoid multi-XSpace peak pileup |
| 5 | T5/T17 | Keep visibility on; reduce overfetch | **M** | observed | Steady-state overfetch → larger tiles |
| 6 | T8 / host policy | Require explicit host(s); no accidental all-hosts | **H** | observed | Silent multi-host fallback is a footgun |
| 7 | T9 | Bound / retire Catapult accumulation | **M**–**H** | observed | Legacy path unbounded merge |
| 8 | T10 | Abort in-flight viewport loads | **M** | hypothesized | Prevents stacked WASM heaps |
| 9 | T13/T36 | Resolution / ratio floors | **M** | observed | Prevents resolution=0 style full dumps |
| 10 | T14 | Feature-flag default flip plan for PB | **H** | hypothesized (rollout) | Without flag plan T1 stays dark |

### Trace vs rest of portfolio

| If your OOM is… | Prioritize |
|-----------------|------------|
| Multi-host session open / TV | 01 T4, T1, T2 + 02 concurrency |
| Browser tab after pan/zoom | 01 T3, T5, T9, T10 |
| CLI agent chain | Themes 1 + 10 (projection + HLO) in §c; docs 03/06 |
| Perfetto open | 06 MP-1, MP-5, MP-3 |
| Disk growth | 02 cache quota |

Trace Viewer should still be optimized **before** micro-optimizing stats DataTables when the goal is "large multi-host interactive sessions don't OOM."

### Trace dependencies into other docs

| TV item | Shared dependency |
|---------|-------------------|
| T2 gzip policy | 02 respond; 06 MP-1 same fix class |
| T4 multi-host | 02 concurrent convert cap; HostSelector |
| Warm-up `trace_viewer@` | 02 DEFAULT_CACHE_TOOLS / generate_cache |
| DCN optional preprocess | 06 ProcessMegascaleDcn adjacency |

---

## (c) Ranked cross-cutting opportunities (global themes)

| Rank | Theme | Why it multiplies across tools | Severity | Confidence | Primary docs |
|------|-------|--------------------------------|----------|------------|--------------|
| 1 | **Projection modes** (`full` / `ui` / `summary` / `peak_*`) so CLI/KPI never build UI protos | CLI detectors, overview, memory, util, top-HLO all pay full convert | **H** | observed | 03, 05, 06 |
| 2 | **Concurrent convert cap** (shared semaphore: `/data` + cache worker) | max_workers=1 for cache only; UI tabs unbounded | **H** | hypothesized | 02 |
| 3 | **Encoding hygiene** — no double compression; free raw after gzip; identity for precompressed | respond() always gzips when encoding unset; Perfetto + trace pb | **H** | observed | 01, 02, 06 |
| 4 | **ALL_HOSTS / multi-host combine without holding all intermediates** | OpStats vector of all hosts; TV multi-host preprocess | **H** | observed | 02, 04, 01, 05 |
| 5 | **Disk cache quota + single-flight** | Unbounded op_stats / SSTABLE growth; stampede on invalidation | **H**–**M** | observed / hypothesized | 02 |
| 6 | **Stream / viewport / top-K on wire** | Unbounded kernel reports, HLO rows, full-host Perfetto, request_details | **H** | observed | 01, 04, 05, 06 |
| 7 | **Avoid double materialization** (dict→json→gzip; json.loads for CSV; JSON round-trip for suggestions) | CSV path; smart_suggestion | **H**–**M** | observed | 02, 05 |
| 8 | **Frontend virtualization / no accumulation** | Catapult merge; full tables with pageSize draw-only | **M** | observed | 01, 04 |
| 9 | **Dead fields / dual sorts** | memory_viewer dual max_heap; dual idle stats | **M**–**H** | observed | 03, 04 |
| 10 | **HLO load amortization** (detectors / peak / graph) | N× module loads; N× graph_viewer text; dump-all debug_info | **H** | observed | 03, 05, 06 |

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
                               │
                    HLO SessionIndex (detectors + peak + graph)
```

### Theme × example opportunity IDs

| Theme | Example IDs from deep dives |
|-------|----------------------------|
| 1 Projection | GOV-1, GMP/FAM summary_only (03), util/KPI summary (05), DET-2 |
| 2 Concurrency | C-sem from 02; DET-5 single-flight |
| 3 Encoding | T2, MP-1, MS-3, respond free-raw |
| 4 ALL_HOSTS | OpStats combine streaming; TV T4; MS-1 lazy DCN |
| 5 Cache | quota, stampede single-flight keys |
| 6 Stream/top-K | T1 viewport; kernel top-K; MP-4; LMC-2 |
| 7 Double materialization | CSV path 02; smart_suggestion 05 |
| 8 FE virtualization | T9 Catapult; stats tables 04 |
| 9 Dead fields | memory_viewer dual heap 03 |
| 10 HLO amortize | URS-1/2, LMC-1, UCR-1, DET-1, peak N+1 03 |

### Severity discipline (portfolio rule)

- Use **H / M / L** only.  
- Do **not** use Critical / Blocker / P0-as-severity.  
- Effort (S/M/L) is separate from severity.  
- Confidence is **observed** or **hypothesized** only — never "likely" / "certain".

### Global ranking notes

1. **Theme 1 beats tool-local polish** when agents and UIs share converts.  
2. **Theme 3 is the best S-effort H win** (encoding) — do early; unblocks PB + Perfetto.  
3. **Theme 10 is the CLI/MCP tax** — after projection, still need binary HLO neighborhood.  
4. **Theme 2 is hypothesized on magnitude** until Phase 0 measures concurrent `/data` peaks — still design it before enabling more parallel warm-up.

---

## (d) Phased recommendations (analysis roadmap — not an implementation PR)

Phases are ordered by **dependency**, not by which file is longest. Each phase lists: goals, concrete work items, dependencies, exit criteria, and primary docs.

### Phase 0 — Measure first (1–2 weeks) — **detail**

Phase 0 is mandatory before large L-effort architecture. It produces numbers that turn **hypothesized** themes into funded work — or demote them.

#### 0.1 Workloads (minimum set)

| ID | Workload | Why |
|----|----------|-----|
| W1 | Multi-host `trace_viewer@` cold open (N∈{2,4,8,10} hosts) | Portfolio #1 interactive server peak |
| W2 | Same session steady-state pan/zoom (3 viewport moves) | Browser + LevelDB filter quality |
| W3 | ALL_HOSTS `overview_page` cold + warm | Warm-up default; CLI get_overview parent |
| W4 | `get_peak_allocations` on multi-module HLO session | Doc 03 N+1 |
| W5 | `megascale_stats&perfetto=true` single large host | Double gzip + full IR (06) |
| W6 | One detector: prefer `detect_unfused_reshapes` then layout mismatch | N× HLO text vs all-module load |
| W7 | `get_overview` CLI after cold overview vs after warm-up | Projection value |
| W8 | Parallel: 4× `/data` different tools same session | Theme 2 concurrency |

#### 0.2 Metrics (collect for each W*)

| Metric | Where | Notes |
|--------|-------|-------|
| Peak RSS (plugin process) | cgroup / `/proc` / sample | Separate Python vs during C++ convert if possible |
| Convert wall time | existing LOG(INFO) processors | overview_page_processor already times |
| Wire bytes (pre/post gzip) | proxy or respond instrumentation | Essential metric for T2/MP-1 |
| Browser JS heap / WASM heap | Chrome memory, performance.memory | TV + Perfetto |
| Disk growth | session dir size before/after | SSTABLE + op_stats + dcn pb |
| Convert count | log tool tag + host | Detector N× confirmation |
| Slim/full ratio | CLI | sizeof(get_overview out) / overview JSON |

#### 0.3 Hypotheses to confirm or kill

| Hypothesis | Theme | Kill criterion |
|------------|-------|----------------|
| Concurrent `/data` causes multi-XSpace RSS pileup | 2 | Peak(W8) ≤ 1.2× max single if false |
| Double gzip costs ≥20% CPU on Perfetto | 3 | Profile MP-1 path |
| PB path unused in prod V2 | 1 (TV) | Traffic sample of `format=` |
| Unfused miss-path multiplies modules | 10 | Log miss rate |
| Overview slim ratio ≪ 0.1 | 1 | Measure W7 |

#### 0.4 Deliverables of Phase 0

1. One spreadsheet / memo: W1–W8 × metrics.  
2. Updated confidence tags on themes 2 and parts of 6/10 if data warrants.  
3. Go / no-go for Phase 1 item ordering (encoding first still expected).  
4. **No product code required** — instrumentation can be temporary scripts outside the campaign rule if needed; prefer existing logs.

#### 0.5 Phase 0 dependencies

| Needs | From |
|-------|------|
| Representative multi-host sessions | Lab / prod-like captures (not committed here) |
| Ability to run plugin + CLI against session | Local xprof install |
| Browser with TV V2 + Perfetto | Standard FE |

---

### Phase 1 — High ROI, low design risk (S/M effort)

**Goal:** Cut obvious double buffers and CLI dead weight without redesigning convert architecture.

| # | Work item | Theme | Severity | Confidence | Depends on | Docs |
|---|-----------|-------|----------|------------|------------|------|
| 1.1 | Encoding: skip gzip when payload already compressed (trace pb, Perfetto) | 3 | **H** | observed | Phase 0 wire samples optional | 01, 02, 06 |
| 1.2 | Wire dead weight: drop dual max_heap sorts; slim memory_profile samples | 9 | **M**–**H** | observed | — | 03 |
| 1.3 | CLI projection flags: overview, memory_profile, utilization, peak | 1 | **H** | observed | API design only | 03, 05, 06 |
| 1.4 | Feature flag: wire `use_pb` → `format=pb` for V2 data URL | 6 / TV | **H** | observed gap | 1.1 preferred first | 01 |
| 1.5 | FE Perfetto transfer list | 3 / FE | **H** | observed | — | 06 |
| 1.6 | LMC-2 top-K copies before BFS | 10 | **H** | observed | — | 06 |
| 1.7 | URS-3 cap module fallback | 10 | **H** | observed | — | 06 |

**Exit criteria Phase 1:**

- No double Content-Encoding path for known precompressed tools (golden tests).  
- At least one CLI tool ships summary/peak mode behind flag.  
- V2 can request PB in a dogfood flag build.  
- Detector fallback cannot scan unbounded modules without cap.

**Dependencies:** 1.4 should follow or pair with 1.1 so PB is not double-gzipped on day one.

---

### Phase 2 — Convert pipeline architecture (M/L)

**Goal:** Make multi-host and multi-tool convert safe under memory limits.

| # | Work item | Theme | Severity | Confidence | Depends on | Docs |
|---|-----------|-------|----------|------------|------------|------|
| 2.1 | Shared convert concurrency limit (`/data` + generate_cache) | 2 | **H** | hypothesized | Phase 0 W8 | 02 |
| 2.2 | OpStats combine streaming / free hosts early | 4 | **H** | observed | — | 02, 04 |
| 2.3 | Projection API on convert for OpStats-backed tools | 1 | **H** | observed | 1.3 learnings | 04, 02 |
| 2.4 | LevelDB/visibility tuning; bound multi-host first open | 6 / TV | **H** | observed | 2.1 helpful | 01 |
| 2.5 | MS-1 lazier DCN cold path | 4 | **M** | hypothesized | — | 06 |
| 2.6 | Single-flight convert keys for cache stampede | 5 | **H** | hypothesized | 2.1 | 02 |

**Exit criteria Phase 2:**

- Documented max concurrent converts; stress test W8 under cap.  
- Multi-host overview RSS scales sub-linearly vs naive Σ (measure).  
- TV first-open has explicit host batch policy.

**Dependencies:** 2.3 builds on Phase 1 projection flags; 2.4 should not enable more parallel hosts without 2.1.

---

### Phase 3 — Product/UX memory + HLO amortization (M/L)

**Goal:** Browser long-session health and CLI detector cost collapse.

| # | Work item | Theme | Severity | Confidence | Depends on | Docs |
|---|-----------|-------|----------|------------|------------|------|
| 3.1 | FE: no Catapult accumulation; slim V2 SoA (T3) | 8 | **H** | observed | T1 PB preferred | 01 |
| 3.2 | Detectors: lazy HLO load; neighborhood from proto not N× graph text | 10 | **H** | observed | DET-1 design | 06, 05, 03 |
| 3.3 | Disk cache quotas + eviction policy | 5 | **H**–**M** | observed | — | 02 |
| 3.4 | Deprecate legacy `trace_viewer` full JSON path | 6 | **M**–**H** | observed | V2 PB default plan | 01 |
| 3.5 | MP-3 intern Event.name; MP-4 subset Perfetto | 6 | **H** | observed / hypothesized | MP-1 done | 06 |
| 3.6 | Shared SessionHloIndex across peak + detectors + graph | 10 | **H** | hypothesized | 3.2 | 03, 06 |

**Exit criteria Phase 3:**

- Unfused reshape path does zero full-text graph_viewer calls on binary path.  
- Peak RSS for detectors independent of untouched modules.  
- Cache dir growth bounded by quota config.  
- Legacy TV JSON behind explicit opt-in only.

---

### Phase 4 — Optional / opportunistic (after 1–3)

| # | Work item | Severity | Notes |
|---|-----------|----------|-------|
| 4.1 | Stream JSON for remaining non-PB clients | **M** | Only if PB insufficient |
| 4.2 | Perfetto LevelDB intermediate | **M** | If re-open cost dominates |
| 4.3 | FE clear tables on panel close (megascale stats, etc.) | **L** | Polish |
| 4.4 | indent=2 removal on large CLI JSON | **L** | Polish |

---

### Explicit non-goals of this document set

- No C++/Python/TS patches shipped in this campaign.  
- No proprietary large-trace benchmarks committed to the repo.  
- Severities labeled **hypothesized** where unmeasured.  
- No rename of tools solely for memory (e.g. no second `megascale_perfetto` processor name without need).

---

### Phase dependency diagram

```text
Phase 0 measure
    │
    ▼
Phase 1 encoding + projection flags + small detector caps
    │
    ├──────────────► Phase 2 convert concurrency + multi-host combine
    │                     │
    │                     ▼
    └──────────────► Phase 3 FE + HLO index + cache quota
                          │
                          ▼
                     Phase 4 polish / optional streams
```

Cross-links:

- **1.1 encoding** unlocks safe **1.4 PB** and **MP-1 class** fixes.  
- **1.3 projection** feeds **2.3** convert API.  
- **2.1 concurrency** gates aggressive **2.4** multi-host parallelism.  
- **3.2 binary neighborhood** is the hard dependency under theme 10; **3.6** index is the shared library form.

---

## (e) How each deep dive feeds prioritization

### Doc 01 — Trace Viewer (~1600 lines)

| Contribution to portfolio | Impact |
|---------------------------|--------|
| Establishes TV as top interactive peak | Themes 3, 4, 6, 8 |
| Catalog of T-ids with path:line | Phase 1–3 TV rows |
| PB + gzip double-compress story | Unifies with 06 Perfetto |
| Catapult vs V2 split | Deprecation sequencing |

### Doc 02 — Convert / serve / cache (~1600 lines)

| Contribution | Impact |
|--------------|--------|
| Single respond/gzip truth | Theme 3 |
| Cache policy + DEFAULT_CACHE_TOOLS | Phase 0 W3; theme 5 |
| Concurrency model | Theme 2 |
| ALL_HOSTS amplification math | Theme 4 |
| CSV double materialization | Theme 7 |

### Doc 03 — Memory family (~1200 lines)

| Contribution | Impact |
|--------------|--------|
| summary_only / peak_only pattern language | Theme 1 template |
| N+1 peak allocations | Theme 10 sibling to detectors |
| Dual max_heap dead weight | Theme 9 quick win |
| Single-host enforcement contrast | Host policy matrix |

### Doc 04 — Overview / stats

| Contribution | Impact |
|--------------|--------|
| OpStats fan-out to many UIs | Theme 1 + 4 leverage |
| Unbounded table tools | Theme 6 top-K |
| Warm-up overview cost | Phase 0 W3 |

### Doc 05 — Graph / HLO / util / smart / perf / pod

| Contribution | Impact |
|--------------|--------|
| graph_viewer cost as dependency | Theme 10 (consumers in 06) |
| smart_suggestion multi-tool cold | Themes 2, 7 |
| util/KPI CLI projection | Theme 1 |

### Doc 06 — Megascale + detectors + get_overview (≥800 lines)

| Contribution | Impact |
|--------------|--------|
| Double gzip Perfetto (path:line) | Theme 3 star example |
| Full host IR | Theme 6 |
| N× HLO text | Theme 10 star example |
| Overview full JSON for scalars | Theme 1 star example |

---

## (f) Cross-tool opportunity matrix (condensed)

Severity only **H|M|L**. Confidence only **observed|hypothesized**.

| Opportunity class | TV | Overview/stats | Memory | Graph/util | Megascale | CLI detectors |
|-------------------|----|----------------|--------|------------|-----------|---------------|
| Projection / summary | M (viewport is different) | **H** | **H** | **H** | L table / **H** overview CLI | **H** |
| Encoding hygiene | **H** (PB) | M (JSON gzip) | M | M | **H** (Perfetto) | L |
| Multi-host bound | **H** | **H** | L (single) | M–H | M (DCN) | L |
| HLO amortize | L (adjacent only) | L | **H** peak | **H** graph | L | **H** |
| FE retention | **H** | M | M | M | **H** Perfetto clone | N/A |
| Cache quota | M (SSTABLE) | **H** op_stats | M | M | L dcn pb | L SQLite CLI |

---

## (g) Campaign method (how this analysis was produced)

| Wave | Concurrent | Focus | Docs (approx lines at synthesis time) |
|------|------------|-------|----------------------------------------|
| 1 (initial) | 3 | Trace, convert/serve, memory family | 01–03 (later deepened) |
| 2 (initial) | 3 | Overview/stats; graph/HLO/util; megascale+CLI | 04–06 |
| Depth re-audit wave 1 | 3 | **Deepen** 01/02/03 from code | **01≈1600 · 02≈1600 · 03≈1200** |
| Depth re-audit wave 2 | ≤3 | Deepen 04–06 | **06≥800**; 04/05 as available |

**Constraint:** max **3** concurrent research subagents; controller monitors each to DONE|FAILED.

**Depth rule:**

- `01-trace-viewer-path.md` is the deepest **single-tool-family** note (~1600 lines).  
- `02-convert-serve-cache-path.md` is the deepest **cross-cutting spine** note (~1600 lines).  
- `03-memory-tools-family.md` is the deepest **memory UI family** note (~1200 lines).  
- `06-megascale-and-cli-detectors.md` is the deepest **megascale + remaining CLI** note (≥800 lines).  
- This `00` synthesis stays shorter by design: inventory, ranking, phases — not a sixth re-derivation.

**Log:** campaign controller log path as used by the orchestration session (not product code).

---

## (h) Index of design docs

| File | Content | Approx depth |
|------|---------|--------------|
| `00-synthesis-memory-optimization.md` | This synthesis | ≥450 |
| `01-trace-viewer-path.md` | Full trace viewer path deep dive | ~1600 |
| `02-convert-serve-cache-path.md` | Shared convert/serve/cache | ~1600 |
| `03-memory-tools-family.md` | memory_profile/viewer + peak CLI | ~1200 |
| `04-overview-and-stats-tools.md` | OpStats fan-out tools | family |
| `05-graph-hlo-util-tools.md` | graph, inference, util, smart, perf, pod | family |
| `06-megascale-and-cli-detectors.md` | megascale/Perfetto + detector CLIs + get_overview | ≥800 |

---

## (i) Confidence legend

- **observed:** Confirmed in source during this campaign (see deep-dive `path:line` cites).  
- **hypothesized:** Plausible from architecture; needs measurement before implementation bets.

---

## (j) Severity legend

- **H:** High — multi-MiB class peaks, double compression of large bodies, N× full converts, or multi-host first-open without bounds.  
- **M:** Medium — real waste, secondary paths, or costs amortized by cache after first hit.  
- **L:** Low — polish, FE retention after close, formatting (indent), optional paths.  

**Never label severity Critical** in this document set.

---

## (k) One-page executive ranking (for steering)

If you can only fund **five** workstreams after Phase 0:

| # | Workstream | Theme | Severity | Confidence | Anchor docs |
|---|------------|-------|----------|------------|-------------|
| 1 | Encoding hygiene (PB + Perfetto + respond) | 3 | **H** | observed | 01, 02, 06 |
| 2 | Projection modes for CLI/overview/memory | 1 | **H** | observed | 03, 04, 06 |
| 3 | Trace PB e2e + multi-host first-open bounds | 6, 4 | **H** | observed | 01, 02 |
| 4 | HLO amortization (binary neighborhood + lazy modules) | 10 | **H** | observed | 03, 06, 05 |
| 5 | Convert concurrency cap + cache single-flight | 2, 5 | **H** | hypothesized | 02 |

Everything else (table virtualization polish, indent, panel-close frees) waits.

---

## (l) Anti-patterns catalog (portfolio)

| Anti-pattern | Where seen | Theme |
|--------------|------------|-------|
| Build UI proto then drop 95% fields in CLI | get_overview, memory CLI, util | 1 |
| Gzip already-compressed bytes | Trace PB, Perfetto | 3 |
| Load all modules to answer one instruction | detectors, peak | 10 |
| Full module text to compute radius-2 BFS | unfused reshapes / hlo_tools | 10 |
| Unbounded concurrent convert | multi-tab UI + cache | 2 |
| ALL_HOSTS combine holding all host structs | OpStats, DCN cold | 4 |
| postMessage clone of large ArrayBuffer | megascale_perfetto | 3 / FE |
| Cache without quota | op_stats, LevelDB, CLI sqlite growth | 5 |
| Dual sorted copies on wire unused by FE | memory_viewer | 9 |
| Accidental multi-host when host omitted | TV host policy | 4 |

---

## (m) Measurement-first Phase 0 checklist (copy/paste)

```text
[ ] Capture or locate multi-host session (W1/W3)
[ ] Capture large single-host megascale session (W5)
[ ] Capture multi-module HLO session (W4/W6)
[ ] Record plugin RSS for W1 cold open (N hosts)
[ ] Record plugin RSS for W3 overview cold vs warm
[ ] Record wire size Perfetto response; verify gzip layers
[ ] Record browser heap TV after 3 pans (W2)
[ ] Record convert counts for detect_unfused_reshapes (W6)
[ ] Record get_overview slim/full ratio (W7)
[ ] Run 4 parallel /data (W8); compare peak RSS
[ ] Note whether V2 requests format=pb in dogfood
[ ] Write Phase 0 memo; re-rank themes 2 and 10 if needed
[ ] Only then schedule Phase 1 implementation epic
```

---

## (n) Relationship between warm-up and CLI

```text
UI session start
  → generate_cache / warm-up
       overview_page (ALL_HOSTS OpStats + full JSON)
       trace_viewer@ (per-host LevelDB)
  → user explores tools on demand

MCP agent (may skip UI entirely)
  → get_overview        # may re-hit overview convert if cold
  → get_top_hlo_ops     # full op_profile
  → detect_*            # HLO dump / N× graph
  → get_peak_allocations
```

Synthesis implication: **warm-up helps UI**, not necessarily agents. Projection modes + HLO index are the agent-side equivalent of warm-up.

---

## (o) Risk register (analysis)

| Risk | Severity if true | Confidence | Mitigation phase |
|------|------------------|------------|------------------|
| PB dogfood never defaulted | **H** long-term RSS | hypothesized | 1.4 + rollout plan |
| Concurrency cap increases latency more than RSS savings | **M** UX | hypothesized | Phase 0 W8 + tunable limit |
| Binary HLO neighborhood incomplete vs text pretty-print | **M** detector quality | hypothesized | 3.2 parity tests |
| Encoding change breaks a client that double-inflates | **M** | hypothesized | golden + canary |
| Quota eviction drops warm LevelDB mid-session | **M** | hypothesized | pin in-use sessions |

---

## (p) Glossary (memory portfolio)

| Term | Meaning |
|------|---------|
| Projection mode | Convert/view flag that emits subset of UI proto |
| Double gzip | Compress already-compressed payload again |
| Full host IR | XprofTrace (or similar) for entire host without viewport |
| N× HLO load | Per-candidate or per-module full proto/text convert |
| Single-flight | Coalesce identical in-flight converts to one |
| ALL_HOSTS | Aggregate host selection token |
| DEFAULT_CACHE_TOOLS | Warm-up tool pair |
| SSTABLE / LevelDB | Trace streaming intermediate |
| summary_only | CLI-oriented projection name used across notes |
| observed / hypothesized | Confidence labels only |

---

## (q) What changed vs earlier short synthesis

| Area | Earlier short 00 | This expanded 00 |
|------|------------------|------------------|
| Doc depths | 01=762, 02=930, 03=521 cited | **01/02/03 ~1600/1600/1200**; **06≥800** |
| Trace top wins | 5 rows | 10 rows + dependency notes from deep 01 |
| Themes | table only | table + ID mapping + executive 5 |
| Phases | short bullets | Phase 0 detailed + 1–4 with deps/exits |
| How to read | index only | reading order + theme map + anti-patterns |
| Severity | H–M mixed wording | **H\|M\|L only**; never Critical |

---

## (r) Suggested owners (roles, not people)

| Stream | Likely skill mix |
|--------|------------------|
| Encoding / respond | Python plugin + golden tests |
| Trace PB / LevelDB | C++ convert + FE V2 |
| Projection modes | C++ processors + CLI Python |
| HLO index | CLI Python + HLO proto + maybe C++ |
| Concurrency / cache | Plugin services + cache_api |
| Perfetto IR | C++ megascale_perfetto + FE postMessage |

---

## (s) Final synthesis statement

XProf memory cost rises from a **small number of structural patterns**, not from dozens of unrelated bugs:

1. **Full UI materialization** for partial consumers (CLI, summary, neighborhood).  
2. **Encoding and buffer lifetime** mistakes (double gzip, coexisting raw+gzip, FE clones).  
3. **Multi-host and multi-module amplification** without streaming or lazy loads.  
4. **Unbounded concurrency and disk cache** on a shared convert spine.

The deepened notes (**01 ~1600**, **02 ~1600**, **03 ~1200**, **06 ≥800**) provide the path:line evidence. This synthesis orders the work: **measure (Phase 0) → encode + project (Phase 1) → convert architecture (Phase 2) → FE + HLO amortization (Phase 3)**. Severity stays **H|M|L**; confidence stays **observed|hypothesized**; **no product code** ships with this campaign.

---

## (t) Document maintenance

| Item | Value |
|------|-------|
| Last synthesis date | 2026-07-13 |
| Mode | analysis only |
| Depth target | ≥450 lines |
| Severity vocabulary | H \| M \| L only |
| Confidence vocabulary | observed \| hypothesized only |
| Companion depths | 01~1600, 02~1600, 03~1200, 06≥800 |

When deep dives grow further, update §(b) top wins and §(c) theme examples — do not duplicate full path:line catalogs here.
