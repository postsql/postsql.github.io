---
layout: page
title: "Design Specification: Direct PostgreSQL Database Logging, Reporting & Live Benchmark Viewer for pgbench"
render_with_liquid: false
---

> [!NOTE]
> Companion specification to [Evolving `pgbench` into a Universal Database Benchmark Engine](/2026/10/19/evolving-pgbench.html).

* TOC
{:toc}

## 1. Overview & Goals

This design adds native support to `pgbench` (`20devel`) for logging transaction/interval telemetry and storing structured benchmark reports directly into a secondary (or primary) PostgreSQL database, with first-class support for **`UNLOGGED` tables**, **zero observer-effect buffered `COPY` streaming with periodic live commits**, and a **lightweight `bottle.py` + `Water.css` + WebAssembly (`WASM`) Benchmark Viewer**.

### Key Capabilities
1. **Three Modular Telemetry Tiers**:
   - **Tier A — Run Catalog & Structured End-of-Run Reports** (`LOGGED` by default): Stores run metadata, effective configuration (`jsonb`), full report (`jsonb`), and normalized summary tables for overall, per-group (`--group`), per-script, per-command (`-r`), named Recall@K (`\record_recall`), and `--sla` verifications.
   - **Tier B — Live Progress & Aggregated Interval Time-Series** (`UNLOGGED` by default): Streams periodic `-P` (`--progress`) snapshots and `--aggregate-interval` windows for real-time SQL dashboards.
   - **Tier C — High-Volume Per-Transaction Log Stream** (`UNLOGGED` by default): Streams `-l` (`--log`) transaction records (enriched with `run_id`, `thread_id`, `group_id`, `recall`, and `validation_ok`) via per-thread buffered `COPY ... FROM STDIN` or post-run `deferred` bulk load.
2. **Zero Observer-Effect Architecture with Periodic Live Commits**:
   - Each worker thread (`TState`) buffers formatted `COPY` rows in a 64 KB thread-local buffer and flushes self-contained `COPY ... FROM STDIN` batches over a dedicated `PGconn` (every 64 KB or 1 second for `pgbench_tx_log`, and every interval for `pgbench_agg_log`)—requiring zero cross-thread locks while making in-flight streams immediately visible to live viewers via MVCC.
   - High-volume stream tables (`pgbench_tx_log`, `pgbench_agg_log`, `pgbench_progress_log`) are created with **zero secondary indexes during ingestion**, with optional post-run index creation and `ANALYZE` (`--log-db-index`).
3. **Built-In SQL Views & Set-Returning Functions (SRFs)**:
   - Automatic schema bootstrap installs analytical views (`pgbench_runs_v`, `pgbench_live_progress_v`), 5 overloaded **Set-Returning Functions (SRFs)** for inspecting a single run (`pgbench_show_run(...)`), comparing a pair of runs (`pgbench_compare_runs(base_id, cand_id)`), or comparing an arbitrary set/array of runs (`pgbench_compare_runs(run_ids[])`), plus the single-JSON viewer helper (`pgbench_viewer_payload_json(...)`).
4. **Lightweight Benchmark Viewer (`bottle.py` + `Water.css` + WASM)**:
   - Single-JSON-from-DB query architecture rendering Phase 1 (Run Selector, Single/Pairwise/Multi-Run Views, Live Running Benchmark Monitor) and Phase 2 (SQL Data Explorer with dynamic Python metatemplates generating `bottle.SimpleTemplate` code + 439-byte WASM diagram projector).

---

## 2. Architecture & Data Flow

![pgbench_db_logging_design diagram 1](/assets/images/pgbench/pgbench_db_logging_design_diagram_1.svg)

---

## 3. CLI Flags & `.pgbench.json` Configuration Surface

| CLI Flag | `.pgbench.json` Key | Default | Description |
| :--- | :--- | :---: | :--- |
| `--log-db=<conninfo>` | `"log_db"` | `NULL` | `libpq` connection string (`postgresql://...` or `host=... port=... dbname=...`) for the secondary database, or `"self"` to use the target benchmark DB. Always records Tier A (and Tier B when `-P` is active); also records Tier C when `-l` (`--log`) is set. |
| `--log-db-unlogged[=MODE]` | `"log_db_unlogged"` | `"stream"` | Controls `UNLOGGED` table creation: `"stream"` (or `true` in JSON) creates `pgbench_tx_log`, `pgbench_agg_log`, and `pgbench_progress_log` as `UNLOGGED` while keeping `pgbench_runs` and summary tables `LOGGED`; `"all"` creates all tables as `UNLOGGED`; `"off"` (`false`) creates all tables as `LOGGED`. |
| `--log-db-mode=MODE` | `"log_db_mode"` | `"stream"` | `"stream"`: live per-thread 64 KB / 1s buffered `COPY ... FROM STDIN`. `"deferred"`: spools to local files during the benchmark window and bulk-loads via `COPY` after the timed run finishes. |
| `--log-db-schema=SCHEMA` | `"log_db_schema"` | `"public"` | Target schema in the logging database (`CREATE SCHEMA IF NOT EXISTS <schema>`). |
| `--log-db-index` | `"log_db_index"` | `false` | Creates `(run_id, ts_usec)` indexes and runs `ANALYZE` on stream tables after the benchmark completes. |
| `--run-label=LABEL` | `"run_label"` | `NULL` | Human/agent label stored in `pgbench_runs.run_label` and exported in `--json-report`. |

### Interaction with Existing Flags
- **`-l` (`--log`) + `--log-db`**: In `stream` mode, writes transaction/interval logs **only** to the database unless `--log-prefix` was explicitly passed on the CLI/config (avoiding redundant local file I/O by default).
- **`--config` Replay**: `pgbench` can continue to replay any JSON config exported from `pgbench_runs.config` or `pgbench_runs.report`.

---

## 4. Relational Schema Specification

All objects are created in `<log_db_schema>` (default `public`).

### 4.1 Tier A: Run Catalog & Summary Tables (`LOGGED` by default)

```sql
CREATE TABLE IF NOT EXISTS pgbench_runs (
    run_id              bigserial PRIMARY KEY,
    run_label           text,
    status              text NOT NULL DEFAULT 'running', -- 'running', 'completed', 'sla_failed', 'aborted'
    exit_code           integer,
    started_at          timestamptz NOT NULL DEFAULT clock_timestamp(),
    ended_at            timestamptz,
    host                text,
    target_db           text,
    scale               integer,
    clients             integer,
    threads             integer,
    duration_sec        double precision,
    transactions_cnt    bigint,
    transactions_failed bigint,
    expected_rollbacks  bigint,
    transactions_skipped bigint,
    retries             bigint,
    retried             bigint,
    tps                 double precision,
    latency_avg_ms      double precision,
    latency_stddev_ms   double precision,
    latency_min_ms      double precision,
    latency_max_ms      double precision,
    latency_p50_ms      double precision,
    latency_p90_ms      double precision,
    latency_p95_ms      double precision,
    latency_p99_ms      double precision,
    latency_p99_9_ms    double precision,
    lag_avg_ms          double precision,
    lag_p50_ms          double precision,
    lag_p90_ms          double precision,
    lag_p95_ms          double precision,
    lag_p99_ms          double precision,
    ontime_pct          double precision,
    recall_samples      bigint,
    recall_avg          double precision,
    recall_p5           double precision,
    recall_p50          double precision,
    recall_p95          double precision,
    recall_initial_q1   double precision,
    recall_mature_q4    double precision,
    recall_deg_pct      double precision,
    validation_passed   bigint,
    validation_failed   bigint,
    validation_pct      double precision,
    sla_all_passed      boolean,
    log_db_dropped_rows bigint DEFAULT 0,
    config              jsonb,
    report              jsonb
);

CREATE TABLE IF NOT EXISTS pgbench_run_groups (
    run_id              bigint NOT NULL REFERENCES pgbench_runs(run_id) ON DELETE CASCADE,
    group_idx           integer NOT NULL,
    group_name          text NOT NULL,
    clients             integer NOT NULL,
    threads             integer NOT NULL,
    sequential          boolean NOT NULL,
    target_rate         double precision,
    transactions_cnt    bigint,
    transactions_failed bigint,
    expected_rollbacks  bigint,
    tps                 double precision,
    latency_avg_ms      double precision,
    latency_p50_ms      double precision,
    latency_p90_ms      double precision,
    latency_p95_ms      double precision,
    latency_p99_ms      double precision,
    latency_p99_9_ms    double precision,
    recall_samples      bigint,
    recall_avg          double precision,
    recall_p5           double precision,
    recall_p50          double precision,
    PRIMARY KEY (run_id, group_idx)
);

CREATE TABLE IF NOT EXISTS pgbench_run_scripts (
    run_id              bigint NOT NULL REFERENCES pgbench_runs(run_id) ON DELETE CASCADE,
    script_idx          integer NOT NULL,
    script_name         text NOT NULL,
    builtin_name        text,
    weight              integer NOT NULL,
    transactions_cnt    bigint,
    transactions_failed bigint,
    expected_rollbacks  bigint,
    tps                 double precision,
    latency_avg_ms      double precision,
    latency_p50_ms      double precision,
    latency_p90_ms      double precision,
    latency_p95_ms      double precision,
    latency_p99_ms      double precision,
    latency_p99_9_ms    double precision,
    recall_samples      bigint,
    recall_avg          double precision,
    recall_p5           double precision,
    recall_p50          double precision,
    PRIMARY KEY (run_id, script_idx)
);

CREATE TABLE IF NOT EXISTS pgbench_run_commands (
    run_id              bigint NOT NULL REFERENCES pgbench_runs(run_id) ON DELETE CASCADE,
    script_idx          integer NOT NULL,
    cmd_idx             integer NOT NULL,
    cmd_type            text NOT NULL,
    first_line          text,
    cnt                 bigint,
    failed              bigint,
    retries             bigint,
    latency_avg_ms      double precision,
    latency_stddev_ms   double precision,
    latency_min_ms      double precision,
    latency_max_ms      double precision,
    PRIMARY KEY (run_id, script_idx, cmd_idx)
);

CREATE TABLE IF NOT EXISTS pgbench_run_recalls (
    run_id              bigint NOT NULL REFERENCES pgbench_runs(run_id) ON DELETE CASCADE,
    scope               text NOT NULL, -- 'overall', 'group:<name>', 'script:<name>'
    metric_name         text NOT NULL, -- 'default' or named recall metric
    samples             bigint,
    avg_recall          double precision,
    stddev_recall       double precision,
    min_recall          double precision,
    max_recall          double precision,
    p5_recall           double precision,
    p50_recall          double precision,
    p95_recall          double precision,
    initial_q1          double precision,
    mature_q4           double precision,
    delta               double precision,
    degradation_pct     double precision,
    PRIMARY KEY (run_id, scope, metric_name)
);

CREATE TABLE IF NOT EXISTS pgbench_run_slas (
    run_id              bigint NOT NULL REFERENCES pgbench_runs(run_id) ON DELETE CASCADE,
    sla_idx             integer NOT NULL,
    raw_spec            text NOT NULL,
    target_scope        text NOT NULL,
    metric              text NOT NULL,
    op                  text NOT NULL,
    threshold_val       double precision,
    actual_val          double precision,
    passed              boolean NOT NULL,
    PRIMARY KEY (run_id, sla_idx)
);
```

### 4.2 Tier B & Tier C: High-Throughput Stream Tables (`UNLOGGED` when `--log-db-unlogged` is `stream` or `all`)

Created **without** foreign keys or secondary indexes during ingestion so `COPY ... FROM STDIN` writes directly to heap pages:

```sql
CREATE [UNLOGGED] TABLE IF NOT EXISTS pgbench_progress_log (
    run_id              bigint NOT NULL,
    ts_epoch_sec        double precision NOT NULL,
    elapsed_sec         double precision NOT NULL,
    tps                 double precision,
    latency_avg_ms      double precision,
    latency_stddev_ms   double precision,
    lag_avg_ms          double precision,
    failed_cnt          bigint
);

CREATE [UNLOGGED] TABLE IF NOT EXISTS pgbench_agg_log (
    run_id                  bigint NOT NULL,
    thread_id               integer NOT NULL,
    interval_start_sec      bigint NOT NULL,
    cnt                     bigint NOT NULL,
    latency_sum_us          double precision,
    latency_sum2_us         double precision,
    latency_min_us          double precision,
    latency_max_us          double precision,
    lag_sum_us              double precision,
    lag_sum2_us             double precision,
    lag_min_us              double precision,
    lag_max_us              double precision,
    skipped_cnt             bigint,
    retried_cnt             bigint,
    retries_cnt             bigint,
    serialization_failures  bigint,
    deadlock_failures       bigint,
    other_sql_failures      bigint,
    recall_samples          bigint,
    recall_sum              double precision,
    validation_passed       bigint,
    validation_failed       bigint
);

CREATE [UNLOGGED] TABLE IF NOT EXISTS pgbench_tx_log (
    run_id              bigint NOT NULL,
    thread_id           integer NOT NULL,
    client_id           integer NOT NULL,
    group_id            integer NOT NULL,
    script_idx          integer NOT NULL,
    tx_cnt              bigint NOT NULL,
    ts_usec             bigint NOT NULL,
    status              text NOT NULL, -- 'ok', 'skipped', 'serialization', 'deadlock', 'other', 'failed'
    latency_us          double precision,
    lag_us              double precision,
    retries             integer,
    recall              double precision,
    validation_ok       boolean
);
```

---

## 5. Built-In SQL Views & Set-Returning Functions (SRFs)

In addition to `pgbench_runs_v` and `pgbench_live_progress_v`, schema bootstrap creates **five Set-Returning Functions (SRFs)** plus a **Single-JSON Viewer Function**:

### 5.1 Single-Run Inspection SRFs
1. **`pgbench_show_run(p_run_id bigint DEFAULT NULL)`**:
   - When `p_run_id IS NULL`, automatically selects the most recent `run_id`.
   - Returns `TABLE (section text, scope text, metric text, value text, unit text, details text)` ordered by logical section (`1. Run Info`, `2. Overall Metrics`, `3. Client Groups`, `4. Scripts`, `5. Recall & Aging`, `6. Commands`, `7. SLA Verifications`).
2. **`pgbench_show_run(p_run_label text)`**:
   - Overload that resolves the latest `run_id` matching `run_label = p_run_label` and delegates to `pgbench_show_run(bigint)`.

### 5.2 Pairwise Run Comparison SRFs
3. **`pgbench_compare_runs(p_base_run_id bigint, p_cand_run_id bigint)`**:
   - Compares two runs across `overall`, matching `group:<name>`, matching `script:<name>`, and matching `recall:<name>` metrics.
   - Returns `TABLE (scope text, metric text, unit text, base_val double precision, cand_val double precision, abs_delta double precision, pct_change double precision, verdict text)`.
   - Automatically evaluates metric polarity (`tps`, `ontime_pct`, `validation_pct`, `recall_avg`, `recall_p5` $$\rightarrow$$ higher is better; `latency_*`, `lag_*`, `transactions_failed` $$\rightarrow$$ lower is better) to emit `'IMPROVED'`, `'REGRESSED'`, or `'UNCHANGED'` (within $$\pm 1\%$$).
4. **`pgbench_compare_runs(p_base_label text, p_cand_label text)`**:
   - Overload that compares the latest run matching `p_base_label` against the latest run matching `p_cand_label`.

### 5.3 Multi-Run Set Comparison SRF
5. **`pgbench_compare_runs(p_run_ids bigint[])`**:
   - Accepts an array of 2 or more `run_id`s (using `p_run_ids[1]` as the baseline reference).
   - Returns `TABLE (scope text, metric text, unit text, run_id bigint, run_label text, val double precision, delta_vs_base double precision, pct_vs_base double precision, sla_all_passed boolean)` ordered by `(scope, metric, array_position(p_run_ids, run_id))`.

### 5.4 Single-JSON Viewer Payload Function
6. **`pgbench_viewer_payload_json(p_mode text, p_run_id bigint, p_base_run_id bigint, p_cand_run_id bigint, p_run_ids bigint[], p_limit int)`**:
   - Returns a single `jsonb` document containing `runs`, `active_runs`, `selected_run`, `show_run_rows`, `groups`, `scripts`, `commands`, `recalls`, `slas`, `progress_series`, `agg_series`, `tx_series`, `live_tx_summary`, `compare_pair`, and `compare_multi`.

---

## 6. Fault Tolerance & Lifecycle Guarantees

1. **Startup Fail-Fast**:
   - Before threads start, the main thread connects to `--log-db`, runs the idempotent DDL bootstrap (`CREATE SCHEMA`, `CREATE [UNLOGGED] TABLE IF NOT EXISTS`, `CREATE OR REPLACE VIEW/FUNCTION`), and executes `INSERT INTO pgbench_runs (...) VALUES (...) RETURNING run_id`. Any connection or SQL error at this stage calls `pg_fatal()`.
2. **Per-Thread `COPY` Stream Resilience**:
   - Each thread `TState` holds a `PGconn *log_conn` and a `PQExpBufferData log_buf` (flushed whenever `log_buf.len >= 65536`, every 1 second in `stream` mode, or at thread teardown).
   - If `PQputCopyData()` or `PQputCopyEnd()` fails mid-run, `pgbench` logs a warning (`pg_log_warning`), increments `thread->log_db_dropped_rows`, closes `thread->log_conn`, and continues executing the benchmark without aborting.
3. **Aborted Run Recording & JSON Fallback**:
   - Whether the benchmark exits normally (`0`), fails an SLA (`1`), or aborts mid-run (`2`), the main thread updates `pgbench_runs` with `status` (`'completed'`, `'sla_failed'`, or `'aborted'`), `exit_code`, and the collected statistics.
   - If the secondary DB connection is down at end-of-run report time, `pgbench` automatically writes the full JSON report to `pgbench_fallback_report_<run_id>.json` in the current directory so no benchmark results are lost.

---

## 7. Periodic Stream Commits for Live View Visibility

To enable real-time monitoring of in-flight benchmarks (`status = 'running'`) without waiting for the benchmark to finish:

1. **`pgbench_progress_log` (`-P`)**:
   - Each periodic progress tick executes a parameterized `INSERT INTO pgbench_progress_log (...)` in autocommit mode on the main logging connection (`log_main_conn`), making every progress tick immediately visible via MVCC.
2. **`pgbench_agg_log` (`--aggregate-interval`) & `pgbench_tx_log` (`-l`) in `stream` Mode**:
   - Rather than holding a single `COPY ... FROM STDIN` statement open across the entire benchmark run (which would keep all streamed rows uncommitted until `threadRun()` finishes), each `logDbFlushThreadBuffer()` call in `stream` mode executes a self-contained `COPY ... FROM STDIN` $$\rightarrow$$ `PQputCopyData()` $$\rightarrow$$ `PQputCopyEnd()` $$\rightarrow$$ `PQgetResult()` batch transaction.
   - `logDbRecordAggInterval()` flushes and commits on every `--aggregate-interval` tick.
   - `logDbRecordTransaction()` flushes and commits whenever `thread->log_buf.len >= 65536` (64 KB) **or** at least once per second (`now_epoch_usec - thread->log_last_flush_us >= 1000000`).
   - Combined with `UNLOGGED` persistence (`--log-db-unlogged=stream`, the default), each periodic batch commit bypasses WAL generation while making live per-second throughput, tail latency percentiles, and Recall@K immediately visible to external readers.

---

## 8. Lightweight Benchmark Viewer (`bottle.py` + `Water.css` + WASM)

Located in `src/bin/pgbench/viewer/pgbench_viewer.py` (documented in `src/bin/pgbench/viewer/README.md`), the benchmark viewer provides an interactive web UI built on three core architectural principles:

![pgbench_db_logging_design diagram 2](/assets/images/pgbench/pgbench_db_logging_design_diagram_2.svg)

### 8.1 Single-JSON-from-DB Query Architecture
- Every HTTP route executes **one** PostgreSQL query returning **one** JSONB document:
  - **Phase 1 Views (`/`, `/live`, `/run/<id>`, `/api/payload`)**: Execute `SELECT pgbench_viewer_payload_json(p_mode, p_run_id, p_base_run_id, p_cand_run_id, p_run_ids, p_limit)`, which packages `runs`, `active_runs`, `selected_run`, `show_run_rows`, `groups`, `scripts`, `commands`, `recalls`, `slas`, `progress_series`, `agg_series`, `tx_series`, `live_tx_summary`, `compare_pair`, and `compare_multi` in a single round-trip.
  - **Phase 2 Data Explorer (`/explorer`)**: Wraps any read-only `SELECT` / `WITH` query into a single JSONB envelope (`SELECT jsonb_build_object('schema', current_schema(), 'row_count', ..., 'rows', COALESCE(jsonb_agg(to_jsonb(q)), '[]'::jsonb))`).

### 8.2 Phase 1 Capabilities
1. **Benchmark Run Selector**:
   - Filterable table of `pgbench_runs_v` with single-click run selection, dropdowns for pairwise Base vs. Candidate comparison, and checkboxes for arbitrary multi-run matrix selection.
2. **Benchmark Views**:
   - **Single-Run Breakdown**: Renders all 7 sections of `pgbench_show_run(run_id)`, per-command statement latency tables (`pgbench_run_commands`), and interactive charts.
   - **Pairwise Comparison**: Renders `pgbench_compare_runs(base_id, cand_id)` with `IMPROVED` / `REGRESSED` / `UNCHANGED` badges and dual-series Base vs. Candidate throughput overlay charts.
   - **Multi-Run Matrix**: Renders `pgbench_compare_runs(bigint[])` across $$N$$ runs with relative deltas and multi-run throughput bar charts.
3. **Live View of Currently Running Benchmarks (`/live`)**:
   - Displays all in-flight runs (`status = 'running'`), live elapsed time, latest progress tick, live HDR-equivalent percentiles (`p50`, `p90`, `p95`, `p99`, `p99.9`) computed from committed `pgbench_tx_log` batches, and configurable 1s/2s/5s single-JSON polling that updates KPI cards and WASM charts in place.

### 8.3 Phase 2 Capabilities
1. **SQL Data Explorer with Python Metatemplates**:
   - Includes 6 curated analytical presets (Runs Overview, Latest Run Breakdown, Top Slowest Commands, Per-Second Throughput & Tail Latency from `pgbench_tx_log`, Vector Recall@K & Index Aging, Live Progress Stream) plus custom read-only SQL execution.
   - **Python Metatemplate Generator (`generate_explorer_stpl_metatemplate`)**: Inspects the JSON result's column names and runtime types to classify each column (`run_link`, `badge`, `meter_pct`, `meter_ratio`, `metric_num`, `num`, `json`, `text`) and dynamically synthesizes a custom `bottle.SimpleTemplate` (`stpl`) template string with hyperlinks, status badges, inline visual percentage/recall meter bars, right-aligned monospace numbers, and collapsible JSON trees. The synthesized `stpl` source is inspectable directly in the UI.
2. **Minimalistic Diagram Framework (`wasm/pgbench_chart.wasm` & `src/`)**:
   - A self-contained 439-byte WebAssembly binary (`wasm/pgbench_chart.wasm`, expanded inline as Base64 via `html/wasm_chart_module.html.stpl` and assembled from `src/`) exporting `memory`, `series_min`, `series_max`, `series_mean`, and `project_1d`.
   - Computes series statistics and projects data arrays into SVG viewport coordinates directly in WebAssembly `Float64Array` linear memory with zero external JavaScript charting dependencies.
