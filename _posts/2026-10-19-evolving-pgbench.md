---
layout: post
title: "Evolving pgbench into a Universal Database Benchmark Engine: OLTP, HTAP, Property Graphs, and Vector Search"
date: 2026-10-19 12:00:00 +0000
author: Hannu Krosing
render_with_liquid: false
---

* TOC
{:toc}

For over two decades, `pgbench` has been the standard tool for quick PostgreSQL performance testing. Its built-in banking transaction (`pgbench_accounts`, `pgbench_branches`, `pgbench_tellers`, `pgbench_history`) is simple, fast, and familiar to every PostgreSQL engineer.

However, modern database workloads look very different from a four-table banking loop. Today, PostgreSQL powers:

* **Complex multi-transaction OLTP** with skewed data distributions, keying/think times, and deliberate business-rule rollbacks;
* **Hybrid Transactional/Analytical Processing (HTAP)** where high-throughput operational writers run alongside multi-join analytical query streams;
* **Property Graph workloads** with power-law social neighborhoods, temporal financial fraud chains, and causally ordered event streams;
* **Vector & RAG workloads** where latency and QPS mean nothing without simultaneously measuring **Recall@K** and **index degradation under continuous write churn**.

Historically, testing any of these workloads required spinning up heavy external drivers in Java, Python, or Go—introducing client-side bottlenecks, dependency sprawl, and multi-gigabyte CSV staging steps.

What if `pgbench` itself could natively drive all of these workloads at C speed, generate realistic data directly inside PostgreSQL in parallel, and validate both performance and result quality SLAs out of the box?

Here is a look at how we refactored and extended `pgbench` into a modular, multi-paradigm benchmarking engine—complete with an in-database `contrib/pgbench` extension and **8 ready-to-run workload suites**.

---

## 1. From an 8,000-Line Monolith to a Modular Architecture

Before adding new capabilities, we needed a clean foundation. Originally, nearly all of `pgbench` lived inside a single monolithic file (`pgbench.c`). We decomposed the codebase into focused, testable subsystems and extracted shared deterministic PRNG and data-generation primitives into `src/common/pgbench_funcs.c`:

![pgbench_evolution_blog_post diagram 1](/assets/images/pgbench/pgbench_evolution_blog_post_diagram_1.svg)

By sharing `pgbench_funcs.c` between the `pgbench` client binary and the new `contrib/pgbench` server extension (`CREATE EXTENSION pgbench;`), the exact same deterministic distributions (`random_zipfian`, `nurand`, `c_last`, `random_vector`, `ldbc_correlated_edge`, `ldbc_flashmob_ts`) are available both in client-side `\set` expressions and inside parallel `INSERT INTO ... SELECT FROM generate_series(...)` server-side data loaders.

---

## 2. Turning the Scripting Language into a Full Workload DSL

Standard `pgbench` scripts only supported 64-bit integers, floating-point numbers, and simple `\if` branches. Real-world benchmarks require rich types, loops, error handling, and modular composition:

* **Rich Native Data Types (`variable.c`)**: Variables now natively hold `text` strings, `timestamp` values, `bytea` payloads, PostgreSQL 1-D arrays (`{1,2,3}`), and dense `float4` vectors (`[0.12,-0.34,...]`).
* **Control Flow & Exception Handling (`commands.c`)**:
  * `\for i = 1 TO :ol_cnt ... \endfor` and `\while <cond> ... \endwhile` for building multi-item order arrays or iterative graph traversals.
  * `\try ... \catch ... \endtry` and `\expect_error <sqlstate>` for gracefully handling expected business-rule rollbacks (such as invalid item IDs or financial fraud aborts) and tracking them separately as **expected rollbacks** rather than benchmark failures.
* **Sub-Script Composition (`\call`)**: Complex workflows can chain reusable sub-scripts via `\call sub_script.sql param1=:val1`, with automatic lexical variable scoping and a 16-frame call-stack depth guard.
* **Array-Aware Query Capture (`\gset_col` & `\gset_meta`)**: While `\gset` captures columns from a single row, `\gset_col` collects an entire result-set column across multiple rows (e.g., `LIMIT 10`) into a single `PGBT_ARRAY` variable—essential for feeding multi-hop graph discoveries or top-$$K$$ vector neighbor IDs into downstream commands.
* **Realistic User Pacing**: `\keying_time` and `\think_time` implement truncated negative-exponential delays using the non-blocking client state machine, allowing thousands of simulated terminals to sleep without blocking OS worker threads.

---

## 3. Orchestrating Complex Multi-Client Topologies

Modern benchmarks rarely run a single homogeneous loop. We added four major orchestration primitives to `engine.c` and `dataset.c`:

1. **Heterogeneous Client Groups (`--group`)**:
   A single `pgbench` invocation can now partition clients into independent workload groups—for example, running 16 high-rate OLTP clients alongside 4 sequential OLAP reporting streams and 1 rate-limited refresh stream, each with its own `--sla` latency and throughput contracts.
2. **Lock-Free Cross-Client Dynamic Reservoirs (`--reservoir`)**:
   In read/write graph and vector benchmarks, readers need to query entities that were *just* inserted by concurrent writers, without running expensive `SELECT id FROM tbl ORDER BY random() LIMIT 1` queries. Writers push newly created IDs via `\reservoir_push`, and readers sample or pop them in $$O(1)$$ time via `\reservoir_sample` / `\reservoir_pop` using lock-free atomic Algorithm R reservoir sampling.
3. **Causal Dependency Watermarks (`\begin_dependent_write` / `\wait_for_watermark`)**:
   When multiple writer threads insert timestamped graph events out of wall-clock order, a lock-free **Multi-Writer Min-Watermark Gatekeeper** tracks the safe global lower-bound timestamp across all active writers so readers never observe half-written causal dependencies.
4. **Phased Execution Barriers (`\phase_barrier`)**:
   Analytical benchmarks often require strict phase transitions (e.g., *Power Run* $$\rightarrow$$ *Micro-Batch Refresh* $$\rightarrow$$ *Throughput Run*). `\phase_barrier <name>` synchronizes all worker threads at a generation-counted pthread barrier before advancing to the next phase.

---

## 4. First-Class Telemetry: HDR Latency, Recall@K, and Index Aging

Throughput alone can be deceptive—especially in approximate nearest-neighbor (ANN) vector search, where an index can achieve 10,000 QPS by returning completely wrong neighbors, or gradually lose accuracy as updates accumulate.

We upgraded `stats.c` with three telemetry engines:

* **HDR Latency & Schedule-Lag Percentiles**: Every script and client group tracks `p50`, `p90`, `p95`, `p99`, and `p99.9` latency, plus open-loop schedule lag percentiles (`--rate` / `--tcr`) and on-time compliance percentages.
* **In-Process Recall@K & Named Quality Metrics (`\record_recall`)**:
  Scripts compare retrieved neighbor arrays against precomputed exact ground-truth arrays directly in C (`\record_recall :retrieved :gt 10 r_prefilter`), recording mean recall, `p5`/`p50` tail recall, and a 10-bucket distribution histogram (`[0.0-0.1)` through `[0.9-1.0]`).
* **Zero-Allocation Sliding-Window Recall Degradation**:
  During continuous vector ingestion and churn, `pgbench` maintains fixed-size 256-sample circular ring buffers (`early_ring` and `recent_ring`) to compare **Initial Recall ($$Q_1$$)** against **Mature Recall ($$Q_4$$)**, quantifying index degradation (`delta` and `degradation_pct`) with zero heap allocations on the hot path.
* **CI/CD SLA Gating (`--sla`) & `--json-report`**:
  Any metric—latency percentiles, TPS, schedule lag, result validation failures, or named recall (`--sla="total:r_prefilter:recall>=0.90"`)—can be enforced as a pass/fail gate that exits with code `2` on violation and emits a structured JSON report.

---

## 5. Eight Built-In Workload Suites (`src/bin/pgbench/workloads/`)

Using these primitives, we built eight self-contained workload suites under `src/bin/pgbench/workloads/`—all using parallel in-database data generation with zero external data files required:

| Workload Suite | Inspiration / Domain | Highlights |
|---|---|---|
| **`pgbench-b`** | Stored-Procedure Banking OLTP | Server-side PL/pgSQL `CALL` variant of the classic banking benchmark to measure network round-trip reduction. |
| **`pgbench-c`** | Retail Order-Entry OLTP *(modeled after TPC-C)* | 9 tables, 5 weighted transactions (`new_order`, `payment`, `order_status`, `delivery`, `stock_level`), NURand skew, and 1% expected rollbacks. |
| **`pgbench-e`** | Financial Brokerage OLTP *(modeled after TPC-E)* | 16 tables, 10 stored procedures, customer positions, market watch, and trade-order/result execution. |
| **`pgbench-h`** | Decision Support OLAP *(modeled after TPC-H)* | 8 tables, 8 analytical queries (`Q1`, `Q3`, `Q5`, `Q6`, `Q10`, `Q12`, `Q14`, `Q19`), and concurrent `RF1`/`RF2` refresh streams. |
| **`pgbench-ds`** | Multi-Channel Retail OLAP *(modeled after TPC-DS)* | 13 star-schema fact/dimension tables, window-function cross-channel rollups, and fact-table maintenance. |
| **`pgbench-htap`** | Mixed OLTP + OLAP *(modeled after CH-benCHmark)* | Runs `pgbench-c` operational transactions concurrently against `pgbench-h`-style analytical queries over the same live tables. |
| **`pgbench-graph`** & **`pgbench-graph-pgq`** | Property Graphs *(inspired by LDBC SNB, FinBench, LSQB, Graphalytics)* | Covers LSQB (`Q1`–`Q9`), SNB Interactive v1/v2 (`IS`, `IC`, `IU`, `ID` cascading deletes), phased SNB BI, FinBench fraud detection, and Graphalytics (`BFS`, `WCC`, `SSSP`, `PageRank`, `LCC`)—available in both **relational SQL** (`pgbench-graph`) and **ISO SQL/PGQ `GRAPH_TABLE`** (`pgbench-graph-pgq`)! |
| **`pgbench-vec`** | Vector Similarity & RAG *(inspired by VecBench)* | Covers all 6 vector operational modes: Unfiltered Top-$$K$$, Pre-Filtered Hybrid, Post-Filtered Oversampling, Streaming Ingestion, Multi-Vector Score Fusion, and Continuous Aging & Degradation. |

> [!NOTE]
> **Fair-Use & Trademark Attribution**: All built-in suites are custom PostgreSQL workloads inspired by or modeled after published industry and academic specifications (TPC, LDBC, CH-benCHmark, VecBench). Metrics produced by these suites are for PostgreSQL engineering, regression testing, and comparative tuning, and are not official certified benchmark results.

---

## 6. Workload as Code: JSON/JSONC Configuration & Replay (`--config` & `--dump-config`)

While modular execution groups, dynamic reservoirs, and vector telemetry made `pgbench` extraordinarily capable, running an advanced workload initially required a 20-flag command line (`-c`, `-j`, `-t`, `-D`, `-f`, `--group`, `--reservoir`, `--sla`, `--dataset`) and a brittle 3-stage shell script (`schema.sql` $$\rightarrow$$ parallel `init.sql` $$\rightarrow$$ `constraints.sql`). Furthermore, relative SQL script paths resolved relative to the caller's working directory (`$PWD`), forcing users to `cd` into workload subdirectories.

To eliminate this operational friction, `config.c` introduces native declarative workload configuration (`--config=<file>`) and configuration export (`--dump-config=<file|->`):

### 6.1 Zero-Dependency JSON/JSONC via `libpgcommon`
PostgreSQL frontend tools avoid heavy external dependencies like `libyaml`. We built a lightweight DOM parser on top of PostgreSQL's core `common/jsonapi.h` engine in `libpgcommon`. Before tokenization, a single-pass JSONC preprocessor strips `//`, `#`, and `/* ... */` comments as well as trailing commas in arrays and objects, while preserving every newline character (`\n`) verbatim. As a result, syntax or semantic errors cite the exact 1-based line number in the user's `.pgbench.json` file.

### 6.2 Multi-Stage Initialization Orchestrator (`init_steps[]`)
Workload schemas, parallel in-database data loaders, and index/constraint builds can now be codified directly in the configuration file:

```json
"init_steps": [
  { "name": "schema", "file": "pgbench_c_schema.sql" },
  {
    "name": "data",
    "file": "pgbench_c_init.sql",
    "clients": 4, "threads": 4, "transactions": 1,
    "variables": { "num_workers": 4 }
  },
  { "name": "constraints", "file": "pgbench_c_constraints.sql" }
]
```

Running `pgbench --config=workloads/pgbench-c/pgbench_c.pgbench.json -i` executes each initialization phase with its own concurrency and variable overrides in a single invocation, with fail-fast validation.

### 6.3 Relocatable Paths & Two-Pass Deterministic Precedence
* **Relocatable Configurations**: All relative script files (`.sql`) and binary vector datasets (`.fvecs`, `.ivecs`) automatically resolve relative to the directory containing the configuration file (`dirname(config)`). Configs can be invoked from anywhere in the filesystem.
* **Deterministic Precedence**: `pgbench` implements a two-pass resolution model where built-in defaults are overridden by `--config` files in appearance order, which in turn are **always overridden by explicit CLI flags** regardless of flag positioning:

  $$\text{Built-in Defaults} \quad<\quad \text{--config file(s)} \quad<\quad \text{Explicit CLI Flags } (-c, -T, -D, \text{--sla})$$

* **Input/Output Symmetry**: Any `--json-report` file can be fed straight back into `pgbench --config=out.json` to reproduce the exact benchmark run. Passing `--dump-config=<file>` (or `--dump-config=-` for `stdout`) serializes the fully merged effective configuration as clean JSON and exits immediately without requiring a database connection.
* **28 Bundled Workload Configs**: Ready-to-run `.pgbench.json` files are included for all built-in workloads under `src/bin/pgbench/workloads/` across relational (`pgbench-b` through `pgbench-ds`), HTAP (`pgbench-htap`), graph (`pgbench-graph`, `pgbench-graph-sql-ud`, `pgbench-graph-pgq`, `pgbench-graph-plpgq`), and vector (`pgbench-vec`) suites.

---

## 7. Seeing It in Action

With `--config`, running full multi-group, multi-phase benchmarks collapses from sprawling shell scripts into single, clean commands.

### 7.1 Single-Command Multi-Group Property Graph Benchmark

Instead of running four separate shell commands, initializing and driving the concurrent reader/writer LDBC-inspired SNB Interactive graph benchmark is now a one-liner:

```bash
# 1. Multi-Stage In-Database Graph Initialization (Schema -> 4-Worker Parallel Gen -> Indexes)
pgbench --config=workloads/pgbench-graph/snb_interactive.pgbench.json -i

# 2. Run Concurrent Reader & Writer Execution Groups with Shared Reservoirs & SLAs
pgbench --config=workloads/pgbench-graph/snb_interactive.pgbench.json

# 3. On-the-fly CLI Overrides (scale clients or duration without modifying the JSON file)
pgbench --config=workloads/pgbench-graph/snb_interactive.pgbench.json -T 120 --sla="total:p95<=25ms"
```

Under the hood, `snb_interactive.pgbench.json` declares the multi-stage pipeline, shared key reservoirs (`persons:4096`, `messages:8192`), the `snb_writers` and `snb_readers` groups, and SLA rules. When executed manually or inspecting the raw CLI equivalence, it maps to:

```bash
pgbench -n -T 60 -s 1 \
        --reservoir=persons:4096 --reservoir=messages:8192 \
        --group="name=snb_writers,clients=2,threads=2,rate=100,file=workloads/pgbench-graph/pgbench_ldbc_snb_iu.sql@80,file=workloads/pgbench-graph/pgbench_ldbc_snb_id.sql@20" \
        --group="name=snb_readers,clients=8,threads=4,file=workloads/pgbench-graph/pgbench_ldbc_snb_ic.sql@30,file=workloads/pgbench-graph/pgbench_ldbc_snb_is.sql@70" \
        --sla="snb_writers:p95<=15ms" \
        --sla="snb_readers:p95<=50ms" \
        --sla="total:validation_failures<=0" \
        --json-report=graph_snb_interactive.json
```

### 7.2 Single-Command Vector Search with Named Recall SLAs

Similarly, initializing and executing the hybrid vector benchmark with named Recall@K contracts is completely streamlined:

```bash
# 1. Multi-Stage Vector Init (Schema -> Clustered Vector Gen -> Constraints -> Ground Truth)
pgbench --config=workloads/pgbench-vec/vec_hybrid.pgbench.json -i

# 2. Execute Hybrid Filtered + Multi-Vector + Streaming Ingestion with Named Recall SLAs
pgbench --config=workloads/pgbench-vec/vec_hybrid.pgbench.json

# 3. Export Effective Merged Configuration to stdout or file
pgbench --config=workloads/pgbench-vec/vec_hybrid.pgbench.json -c 8 -T 120 --dump-config=-
```

Or executed via the equivalent direct CLI flags:

```bash
pgbench -n -c 4 -j 2 -T 60 -D scale=1 --reservoir=vec_ids:1024 \
        -f workloads/pgbench-vec/pgbench_vec_search.sql@30 \
        -f workloads/pgbench-vec/pgbench_vec_filtered.sql@25 \
        -f workloads/pgbench-vec/pgbench_vec_postfiltered.sql@20 \
        -f workloads/pgbench-vec/pgbench_vec_multivec.sql@15 \
        -f workloads/pgbench-vec/pgbench_vec_insert.sql@10 \
        --sla="total:r_unfiltered:recall>=0.95" \
        --sla="total:r_prefilter:recall>=0.90" \
        --sla="total:r_multivec:recall>=0.85" \
        --sla="total:p95<=50ms" \
        --json-report=vec_benchmark.json
```

---

## 8. Closing the Telemetry Loop: Direct PostgreSQL Database Logging (`--log-db`) & Live WASM Benchmark Viewer

Even with `--json-report` and `-l` per-transaction log files, managing historical benchmark runs across parameter sweeps or CI/CD branches remained painful: teams still had to scrape `pgbench_log.<pid>.<tid>` flat files, stitch together shell scripts to load `COPY` streams into external tables, and maintain ad-hoc Python notebooks just to compare two runs.

With `logdb.c`, `pgbench` now supports **first-class direct PostgreSQL database logging and live telemetry** (`--log-db=<conninfo|self>`, detailed in [`pgbench_db_logging_design.md`](/pgbench/pgbench_db_logging_design.html) and [`pgbench_viewer_design.md`](/pgbench/pgbench_viewer_design.html)):

### 8.1 Three-Tier Relational Telemetry & `UNLOGGED` Stream Tables
Passing `--log-db=self` (to log into the benchmarked database itself) or `--log-db="host=metrics-db dbname=perf"` automatically provisions a 9-table, 3-tier schema (defaulting to `public`, or customized via `--log-db-schema=<schema>`):

* **Tier A — Run Metadata & Final Summary Tables (`LOGGED` by default)**:
  * `pgbench_runs`: Complete run catalog (`run_id`, `run_label`, `status`, `config_jsonb`, `report_jsonb`, overall TPS, HDR latency percentiles `p50`/`p90`/`p95`/`p99`/`p99.9`, schedule lag, `sla_all_passed`, and `validation_passed`/`failed`).
  * `pgbench_run_groups`, `pgbench_run_scripts`, `pgbench_run_commands`, `pgbench_run_recalls`, and `pgbench_run_slas`: Normalized breakdowns for multi-group workloads, per-script percentiles, per-command latencies (`-r`), Recall@K histograms + `Q1`-vs-`Q4` aging degradation, and SLA contract evaluations.
* **Tier B — Live Time-Series Progress & Interval Aggregation (`UNLOGGED` by default)**:
  * `pgbench_progress_log` (`-P`) and `pgbench_agg_log` (`--aggregate-interval`), streamed over dedicated `COPY ... FROM STDIN` connections and committed after every progress/aggregation tick so external dashboards see live time-series curves while the benchmark is still running.
* **Tier C — Per-Transaction & Per-Query Microsecond Trace (`UNLOGGED` by default)**:
  * `pgbench_tx_log` (`-l`), streamed via buffered `COPY ... FROM STDIN` (`--log-db-mode=stream`, periodically committing every 64 KB or 1 second for live visibility) or spooled to local disk and bulk-loaded at teardown (`--log-db-mode=deferred` for zero mid-run network/CPU perturbation).
* **Configurable WAL Persistence (`--log-db-unlogged=stream|all|off`)**:
  By default (`stream`), high-rate Tier B/C tables are created as `UNLOGGED` to avoid polluting WAL write amplification during the benchmark, while Tier A summary catalogs remain crash-safe `LOGGED` tables.

### 8.2 Built-In SQL Run Inspection & Regression Comparison SRFs
Alongside the telemetry tables, `logdb.c` installs **2 views** (`pgbench_runs_v`, `pgbench_live_progress_v`) and **6 SQL functions** directly in the logging schema—allowing engineers to inspect or compare runs from `psql` without any external tooling:

```sql
-- 1. Pretty-print the most recent run (or pass a specific run_id or run_label)
SELECT section, item, metric, value FROM pgbench_show_run();

-- 2. Pairwise baseline-vs-candidate regression diff (by run_id or run_label)
--    Automatically computes delta, delta_pct, and IMPROVEMENT / REGRESSION verdicts!
SELECT category, target, metric, baseline_val, candidate_val, delta_pct, status
  FROM pgbench_compare_runs('baseline-hnsw-m16', 'candidate-hnsw-m32');

-- 3. N-way side-by-side comparison matrix across an array of run_ids
SELECT category, target, metric, values_by_run, pct_vs_first
  FROM pgbench_compare_runs(ARRAY[101, 102, 103]::bigint[]);
```

### 8.3 Zero-Framework Web Viewer (`bottle.py` + `Water.css` + 439-Byte WebAssembly Projector + Metatemplate SQL Explorer)
To visualize both completed and **currently running** benchmarks without pulling in React, Node.js, or heavy charting bundles, `src/bin/pgbench/viewer/pgbench_viewer.py` (documented in `src/bin/pgbench/viewer/README.md`) provides a self-contained web UI built on four minimalist pillars:

1. **Single-JSON-from-Database Architecture (`pgbench_viewer_payload_json`)**:
   Every Phase 1 view (`single`, `compare`, `multi`, and `live`) executes a single `SELECT pgbench_viewer_payload_json(...)` query. For active `status = 'running'` runs, PostgreSQL computes real-time `p50`/`p90`/`p95`/`p99`/`p99.9` latency percentiles and live Recall@K directly from committed `pgbench_tx_log` and `pgbench_progress_log` batches.
2. **Classless `Water.css` (`css/`) + Modular `bottle.py` (`html/*.html.stpl`)**:
   Semantic HTML tables (`<table>`, `<details>`, `<kbd>`, `<mark>`) are styled automatically by `css/water.css` and `css/viewer.css` and rendered from modular `SimpleTemplate` views.
3. **439-Byte Hand-Crafted WebAssembly Diagram Projector (`wasm/pgbench_chart.wasm` & `src/`)**:
   Time-series TPS, latency percentiles, Recall@K histograms, and `Q1`-vs-`Q4` aging bars are projected into SVG `<polyline>` and `<rect>` coordinates in browser memory via a 439-byte WASM binary (`wasm/pgbench_chart.wasm`, built from `src/pgbench_chart.wat`, `src/pgbench_chart.c`, and `src/build_wasm.py`, exporting `series_min`, `series_max`, `series_mean`, `project_1d` over `Float64Array`).
4. **Phase 2 SQL Data Explorer with Python Metatemplate Generator**:
   The `/explorer` tab lets users run 6 curated diagnostic presets (or custom read-only `SELECT`/`WITH` queries over `pgbench_*` tables). A Python metatemplate generator inspects the PostgreSQL result-set columns at runtime, synthesizes a custom `bottle.SimpleTemplate` (`stpl`) template with clickable `run_id` badges, verdict pills, and right-aligned numeric formatting, and auto-renders a WASM time-series or bar chart whenever numeric series columns are detected.

---

## 9. Built for Autonomous AI Agents: Experimental Agent-Oriented Documentation & Skill

As database infrastructure becomes increasingly automated, benchmarking is no longer performed exclusively by human engineers. Autonomous coding agents, LLM assistants, and automated CI/CD pipelines regularly evaluate PostgreSQL performance, validate optimizer improvements, and investigate latency regressions. However, traditional human-oriented documentation (verbose man pages and narrative prose) is notorious for tripping up AI agents—causing option hallucinations, skipped multi-stage initialization steps, unparsed terminal logs, and unintentional trademark violations.

To solve this, the new `pgbench` ecosystem introduces **experimental agent-oriented documentation and an integrated agent skill**:

* **Integrated Agent Skill (`pgbench-new`)**: Packaged as a specialized agent skill ([`pgbench_agent_skill.md`](/pgbench/pgbench_agent_skill.html)), it equips autonomous agents with explicit trigger definitions, invariant execution pipelines (`schema` $$\rightarrow$$ `init` $$\rightarrow$$ `constraints`), unambiguous exit code semantics (`0` = pass, `1` = fatal/execution error, `2` = SLA contract breach), and strict trademark/fair-use guardrails.
* **Deterministic Agent Decision Matrix & Recipes**: Accompanied by an agent decision matrix ([`pgbench_agent_decision_matrix.md`](/pgbench/pgbench_agent_decision_matrix.html)) and structured YAML recipes ([`pgbench_agent_recipes.yaml`](/pgbench/pgbench_agent_recipes.html)), agents can programmatically select the ideal benchmark suite, calculate appropriate scale factors, and enforce SLA contracts based on natural-language user requests without trial-and-error.
* **Seamless Synergy with `--config`, `--json-report` & `--log-db`**: Instead of synthesizing brittle 25-flag command-line invocations, agents can read, generate, or mutate `.pgbench.json` files, validate them offline via `pgbench --config=... --dump-config=-`, stream runs directly into PostgreSQL via `--log-db=self --run-label=...`, and query `pgbench_compare_runs()` or `--json-report` for reliable closed-loop benchmarking and automated parameter optimization.

---

With these additions, `pgbench` bridges the gap between micro-benchmarks and heavyweight external test harnesses—giving PostgreSQL developers, DBAs, and AI agents a unified, zero-dependency, configuration-driven engine to test, store, compare, and visualize transactional, analytical, graph, and vector workloads alike.
