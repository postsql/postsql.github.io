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
3. **Zero-Build, Single-File Simplicity**:
   - Built with standalone `bottle.py` + `Water.css` + an embedded WebAssembly (`pgbench_chart.wasm`) micro-plotter (439-byte binary that computes SVG polyline/bar coordinates and scales directly from JSON arrays). See `src/bin/pgbench/viewer/README.md` for complete CLI usage and endpoint documentation.

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

2. **Python Metatemplate Generator (`build_result_template(columns_meta)`)**:
   - A Python function inspects the inferred type and name of each column in the JSON result (`numeric`/`float` latency/TPS/pct, `boolean` SLA/validation, `verdict` string, `jsonb`, or timestamp) and **dynamically generates a specialized `bottle.SimpleTemplate` (`stpl`) source string** tailored to that exact result schema:
     - Columns ending in `_ms`, `_us`, `_sec`, `tps`, or `pct` get right-aligned numeric formatting (`{{"{:.3f}".format(row["..."])}}`) and inline proportional bar rules.
     - `passed` / `validation_ok` / `verdict` columns compile into semantic badge markup (`<mark>`).
     - If numeric x/y series columns are detected (e.g., `elapsed_sec` or `interval_start_sec` + `tps`/`latency_avg_ms`), the metatemplate automatically injects a `<canvas>` / `<svg>` bound to the WASM chart renderer!
   - The generated template string is also displayed in a collapsible `<details><summary>View Generated Bottle Metatemplate</summary><pre>...</pre></details>` panel so users can inspect how the template was synthesized.

### 4.2 Minimalistic WebAssembly (WASM) Diagram Framework
To keep the frontend dependency-free and fast while supporting rich charts:

- The viewer embeds a tiny, self-contained **WebAssembly layout & scaling module** (`chart_kernel.wasm`, ~1.5 KB base64-encoded directly in the page and compiled from hand-written WAT/C):
  - **Exported WASM functions**:
    - `compute_line_points(ptr, len, width, height, pad)` $$\rightarrow$$ normalizes min/max bounds in linear memory and computes screen $$(x_i, y_i)$$ coordinates for time-series curves (`tps`, `latency_avg_ms`, `lag_avg_ms`).
    - `compute_bar_rects(ptr, len, width, height, pad)` $$\rightarrow$$ computes bar heights and positions for HDR latency percentiles (`p50`..`p99.9`), 10-bucket Recall@K histograms, and multi-run TPS comparisons.
- A ~60-line vanilla JS wrapper feeds numeric arrays from the page's single JSON object into the WASM linear memory (`Float64Array`) and renders crisp, responsive `<svg>` charts that automatically adapt to `Water.css` light/dark mode (`currentColor` / CSS variables).
