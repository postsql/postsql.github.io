---
layout: page
title: "Design Specification: pgbench Telemetry Viewer (bottle.py + Water.css + WASM)"
render_with_liquid: false
---

> [!NOTE]
> Companion specification to [Evolving `pgbench` into a Universal Database Benchmark Engine](/2026/10/19/evolving-pgbench.html).

* TOC
{:toc}

## 1. Architectural Principles

1. **Single-JSON-Payload Database Fetch**:
   - Every view in the web application retrieves its entire page state from the logging/telemetry PostgreSQL database in **one round-trip SQL query returning a single JSON document** (`SELECT pgbench_viewer_page_json(...)` or `SELECT jsonb_build_object(...)`).
   - The `bottle.py` server acts as a thin, stateless presenter: it fetches the single JSON document from PostgreSQL (or serves it directly at `/api/page.json`), passes it into `bottle.template()`, and renders clean semantic HTML styled by classless **`Water.css`**.
2. **Regular Telemetry Commits for Real-Time Visibility**:
   - In `logdb.c`, `pgbench_progress_log` (`-P`) commits every interval, and `stream` mode (`pgbench_agg_log` / `pgbench_tx_log`) commits each flushed `COPY ... FROM STDIN` batch (`PQputCopyEnd` + re-initiate `COPY` on next flush or interval tick).
   - Because stream tables are `UNLOGGED` (`--log-db-unlogged`), committing a batch does not trigger a WAL flush (`XLogFlush`) on the secondary database, keeping ingestion overhead near zero while making in-flight rows immediately visible under PostgreSQL `READ COMMITTED`.
3. **Zero-Build, Modular Runtime & Source Separation (`html/`, `css/`, `wasm/`, `src/`)**:
   - Built with standalone `bottle.py` (`SimpleTemplate`), `Water.css` (`css/water.css` + `css/viewer.css`), modular HTML templates (`html/*.html.stpl`), and a 439-byte WebAssembly (`wasm/pgbench_chart.wasm`) micro-plotter that computes SVG polyline/bar coordinates and scales directly from JSON arrays.
   - `PartsLoader` in `pgbench_viewer.py` loads templates, stylesheets, and the WASM binary on demand with `st_mtime` hot-reload caching, while non-runtime WebAssembly sources and the zero-dependency assembler (`src/pgbench_chart.wat`, `src/pgbench_chart.c`, `src/build_wasm.py`) are isolated in `src/`. See `src/bin/pgbench/viewer/README.md` for complete CLI usage and endpoint documentation.

---

## 2. End-to-End Architecture

![pgbench_viewer_design diagram 1](/assets/images/pgbench/pgbench_viewer_design_diagram_1.svg)

---

## 3. Phase 1: Run Selector, Benchmark Views & Live Monitor

### 3.1 Single-JSON Database Payload (`pgbench_viewer_json`)
A single SQL query (or helper function `pgbench_viewer_json(p_run_ids bigint[], p_live_run_id bigint)`) returns one unified JSON document:

```json
{
  "generated_at": "2026-10-19T08:55:00Z",
  "active_runs": [ ... ],
  "runs_catalog": [ ... ],
  "selected_run": {
    "overview": { ... },
    "sections": [ ... ],
    "groups": [ ... ],
    "scripts": [ ... ],
    "recalls": [ ... ],
    "commands": [ ... ],
    "slas": [ ... ],
    "timeseries": [ ... ]
  },
  "comparison": {
    "mode": "pair | multi",
    "run_ids": [1, 2],
    "rows": [ ... ]
  }
}
```

### 3.2 Phase 1 UI Components (`Water.css` Semantic Elements)
1. **Benchmark Run Selector**:
   - Top navigation bar and `<details>` / `<form>` selector listing all runs from `runs_catalog` with status pills (`COMPLETED`, `RUNNING`, `SLA_FAILED`, `ABORTED`), scale, clients, TPS, `p95` latency, and `run_label`.
   - Allows selecting:
     - **Single Run** (inspects via `pgbench_show_run(run_id)`)
     - **Pair of Runs** (baseline vs. candidate diff via `pgbench_compare_runs(base_id, cand_id)`)
     - **Set of Runs** (multi-run checkbox selection rendered via `pgbench_compare_runs(run_ids[])`)
2. **Benchmark Views**:
   - **Single-Run View**: Clean `Water.css` cards and tables for Overall Summary, Latency Percentiles (`p50`, `p90`, `p95`, `p99`, `p99.9`), Client Groups, Scripts, Vector Recall@K 10-bucket distribution & Q1-vs-Q4 Aging Degradation, Per-Statement latencies (`-r`), and SLA pass/fail badges.
   - **Comparison View**: Side-by-side delta table with automatic polarity coloring (`IMPROVED` in green, `REGRESSED` in red, `UNCHANGED` neutral).
3. **Live View of Currently Running Benchmark**:
   - Auto-detects any run in `pgbench_runs` with `status = 'running'` (or lets the user pin the live tab).
   - Polls `/api/live.json` (which executes a single SQL query joining `pgbench_runs`, `pgbench_progress_log`, and committed batches in `pgbench_agg_log` / `pgbench_tx_log`) every 1–2 seconds and updates the live KPIs and time-series charts without page flicker.

---

## 4. Phase 2: Metatemplate Data Explorer & WASM Diagram Framework

### 4.1 SQL Data Explorer with Python Metatemplates
Instead of a generic unformatted loop over cells, the Data Explorer uses a **two-stage Python Metatemplate pipeline**:

1. **Single-JSON Query Execution + Introspection**:
   - When the user submits a `SELECT` query (or selects a preset query such as *"Top Slowest Statements"*, *"Per-Thread Throughput Skew"*, *"Recall Aging Curve"*, or *"Latency Tail Anomalies"*), the server wraps the query in a read-only transaction and extracts both column metadata and rows as a single JSON payload:

     ```sql
     WITH q AS (<user_query> LIMIT 500)
     SELECT jsonb_build_object(
       'row_count', count(*),
       'rows', COALESCE(jsonb_agg(to_jsonb(q)), '[]'::jsonb)
     ) FROM q;
     ```

2. **Python Metatemplate Generator (`generate_explorer_stpl_metatemplate` + `html/explorer_table_meta.html.stpl`)**:
   - `infer_column_Descriptors()` inspects the inferred type and name of each column in the JSON result (`run_link`, `badge`, `meter_pct`, `meter_ratio`, `metric_num`, `num`, `json`, or `text`) and expands the Stage-1 metatemplate `html/explorer_table_meta.html.stpl` (using custom delimiter syntax `<@ @> @ {| |}`) into a **specialized Stage-2 `bottle.SimpleTemplate` (`stpl`) source string** tailored to that exact result schema:
     - Columns ending in `_ms`, `_us`, `_sec`, `tps`, or `pct` get right-aligned numeric formatting and inline proportional meter bars (`meter_pct` / `meter_ratio`).
     - `status` / `passed` / `validation_ok` / `verdict` / `sla_all_passed` columns compile into semantic badge markup.
     - When rows are returned, `html/explorer_body.html.stpl` includes `html/wasm_chart_box.html.stpl` bound to the WASM chart renderer (`renderExplorerChart()`).
   - The generated Stage-2 template string is also displayed in a collapsible `<details><summary>Inspect Generated stpl Metatemplate Source</summary><pre>...</pre></details>` panel so users can inspect how the template was synthesized.

### 4.2 Minimalistic WebAssembly (WASM) Diagram Framework (`wasm/pgbench_chart.wasm` & `src/`)
To keep the frontend dependency-free and fast while supporting rich charts:

- The viewer loads a 439-byte **WebAssembly statistical & 1D projection module** (`wasm/pgbench_chart.wasm`, expanded inline as Base64 via `html/wasm_chart_module.html.stpl` with `/static/pgbench_chart.wasm` streaming fallback, and built reproducibly from `src/pgbench_chart.wat`, `src/pgbench_chart.c`, and `src/build_wasm.py`):
  - **Exported WASM interface**:
    - `memory` (2 pages = 128 KiB linear memory for zero-copy `Float64Array` views).
    - `series_min(ptr, n) -> f64`, `series_max(ptr, n) -> f64`, `series_mean(ptr, n) -> f64` $$\rightarrow$$ compute `f64` series bounds and mean in linear memory.
    - `project_1d(in_ptr, n, min_v, max_v, out_start, out_span, out_ptr)` $$\rightarrow$$ normalizes and projects data arrays into SVG viewport coordinates for time-series curves (`tps`, `latency_avg_ms`), HDR latency percentiles (`p50`..`p99.9`), multi-run TPS comparisons, and Explorer bar charts.
- The JS bridge in `html/wasm_chart_module.html.stpl` feeds numeric arrays from the page's single JSON object into WASM linear memory (`Float64Array`) and renders crisp, responsive `<svg>` charts (`html/wasm_chart_box.html.stpl`) that automatically adapt to `Water.css` light/dark mode (`currentColor` / CSS variables).
