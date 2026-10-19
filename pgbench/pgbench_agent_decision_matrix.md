---
layout: page
title: "Agent Guide: Benchmark Decision Matrix & Tool-Calling Protocol"
render_with_liquid: false
---

> [!NOTE]
> Companion specification to [Evolving `pgbench` into a Universal Database Benchmark Engine](/2026/10/19/evolving-pgbench.html).

* TOC
{:toc}

**Target Consumer**: Autonomous Tool-Calling Agents (ReAct, CodeAct, Function-Calling Agents)  
**Purpose**: Deterministic translation of natural language benchmark requests into valid `pgbench` CLI invocations, strict state-machine transitions, and error mitigation trees.

---

## 1. Intent-to-Workload Dispatch Matrix

Use this lookup table to map user requests directly to the correct workload suite, required scale factors, script weights, and mandatory SLA constraints:

| User Intent / Workload Request | Target Suite | Workload Path (`workloads/`) | Mandatory Options | Default Script Allocation | SLA Template |
|---|---|---|---|---|---|
| **Banking / Accounts OLTP** | `pgbench-b` | `pgbench-b/` | `-D scale=<N>` | `pgbench_b_tx.sql@100` | `--sla="total:p95<=100ms"` |
| **Order Entry / TPC-C-like OLTP** | `pgbench-c` | `pgbench-c/` | `-D scale=<warehouses>` | `pgbench_c_new_order.sql@45`<br>`pgbench_c_payment.sql@43`<br>`pgbench_c_order_status.sql@4`<br>`pgbench_c_delivery.sql@4`<br>`pgbench_c_stock_level.sql@4` | `--sla="total:p95<=5000ms"` |
| **Financial Brokerage / Trades** | `pgbench-e` | `pgbench-e/` | `-D scale=<N>` | `pgbench_e_trade_status.sql@44`<br>`pgbench_e_market_watch.sql@18`<br>`pgbench_e_customer_position.sql@13`<br>`pgbench_e_trade_order.sql@10`<br>`pgbench_e_trade_result.sql@10`<br>`pgbench_e_broker_volume.sql@5` | `--sla="total:p95<=3000ms"` |
| **Decision Support / Ad-Hoc OLAP** | `pgbench-h` | `pgbench-h/` | `-D scale=<N>` | `--group="name=queries,clients=4,sequential=true,file=pgbench_h_queries.sql"`<br>`--group="name=refresh,clients=1,rate=2,file=pgbench_h_refresh.sql"` | `--sla="queries:p95<=30000ms"` |
| **Star-Schema Retail Analytics** | `pgbench-ds` | `pgbench-ds/` | `-D scale=<N>` | `--group="name=reporting,clients=4,file=pgbench_ds_q_store_sales.sql@25,..."`<br>`--group="name=maint,clients=1,rate=2,file=pgbench_ds_maintenance.sql"` | `--sla="reporting:p95<=45000ms"` |
| **Mixed OLTP + Analytics (HTAP)** | `pgbench-htap` | `pgbench-htap/` | `-D scale=<warehouses>` | `--group="name=oltp,clients=8,file=pgbench_htap_oltp.sql"`<br>`--group="name=olap,clients=2,file=pgbench_htap_olap_q1.sql@20,..."` | `--sla="oltp:p95<=5000ms"`<br>`--sla="olap:p95<=15000ms"` |
| **Graph: Subgraph Patterns (LSQB)** | `pgbench-graph` | `pgbench-graph/` | `-s 1` | `pgbench_ldbc_lsqb.sql@100` | `--sla="validation_failures<=0"` |
| **Graph: SNB Interactive Readers/Writers** | `pgbench-graph` | `pgbench-graph/` | `--reservoir=persons:4096`<br>`--reservoir=messages:8192` | `--group="name=snb_readers,clients=4,file=pgbench_ldbc_snb_ic.sql@30,file=pgbench_ldbc_snb_is.sql@70"`<br>`--group="name=snb_writers,clients=2,file=pgbench_ldbc_snb_iu.sql@75,file=pgbench_ldbc_snb_id.sql@25"` | `--sla="validation_failures<=0"` |
| **Graph: SNB BI Phased Workload** | `pgbench-graph` | `pgbench-graph/` | `-s 1` | `pgbench_ldbc_snb_bi.sql@100` | `--sla="validation_failures<=0"` |
| **Graph: FinBench Fraud & Graphalytics** | `pgbench-graph` | `pgbench-graph/` | `-s 1` | `pgbench_ldbc_finbench.sql@100`<br>`pgbench_ldbc_graphalytics.sql@100` | `--sla="validation_failures<=0"` |
| **Graph (SQL/PGQ `GRAPH_TABLE`): LSQB, SNB, FinBench, Graphalytics** | `pgbench-graph-pgq` | `pgbench-graph-pgq/` | `-s 1` | `pgbench_ldbc_pgq_lsqb.sql`, `pgbench_ldbc_pgq_snb_*.sql`, `pgbench_ldbc_pgq_finbench.sql`, `pgbench_ldbc_pgq_graphalytics.sql` | `--sla="validation_failures<=0"` |
| **Vector: Hybrid Filtered & Multi-Vector** | `pgbench-vec` | `pgbench-vec/` | `--reservoir=vec_ids:1024` | `pgbench_vec_search.sql@30`<br>`pgbench_vec_filtered.sql@25`<br>`pgbench_vec_postfiltered.sql@20`<br>`pgbench_vec_multivec.sql@15`<br>`pgbench_vec_insert.sql@10` | `--sla="total:r_unfiltered:recall>=0.95"`<br>`--sla="total:r_prefilter:recall>=0.90"`<br>`--sla="total:r_postfilter:recall>=0.40"`<br>`--sla="total:r_multivec:recall>=0.85"` |
| **Vector: Continuous Aging & Degradation** | `pgbench-vec` | `pgbench-vec/` | `--reservoir=vec_ids:1024` | `pgbench_vec_aging.sql@100` | `--sla="total:r_aging:recall>=0.80"`<br>`--sla="total:p95<=150ms"` |

---

## 2. Invariant State Machine for Database Initialization

Agents MUST execute database initialization sequentially. Never skip states:

![pgbench_agent_decision_matrix diagram 1](/assets/images/pgbench/pgbench_agent_decision_matrix_diagram_1.svg)

### Invariant Execution Templates

```bash
# STATE 1: Schema creation (Single transaction, single client)
pgbench -n -d $DB_NAME -t 1 -f workloads/$SUITE/${SUITE}_schema.sql

# STATE 2: Parallel data population (N workers, partitioned by :client_id)
pgbench -n -d $DB_NAME -c $WORKERS -j $WORKERS -t 1 \
        -D scale=$SCALE -D num_workers=$WORKERS \
        -f workloads/$SUITE/${SUITE}_init.sql

# STATE 3: Constraint, foreign key, secondary index, and ANALYZE
pgbench -n -d $DB_NAME -t 1 -f workloads/$SUITE/${SUITE}_constraints.sql

# STATE 3b (MANDATORY FOR pgbench-vec ONLY): In-database ground truth generation
pgbench -n -d $DB_NAME -t 1 -f workloads/pgbench-vec/pgbench_vec_gt_gen.sql
```

---

## 3. Tool-Calling CLI Flag Invariants & Exclusions

When generating `pgbench` command lines, enforce these boolean and relational constraints:

1. **`-i` (Builtin Init) vs `-f` (Custom Workload Script)**:
   - ⚠️ **MUTUALLY EXCLUSIVE**: Never pass `-i` when executing custom workload suites. Always use `-n -f <path>`.
2. **`--group` vs Top-Level `-f` / `-c`**:
   - ⚠️ **MUTUALLY EXCLUSIVE**: If `--group` is present, do NOT pass top-level `-f`, `-c`, or `-t`. All client counts and script files must reside inside `--group="name=...,clients=...,file=..."`.
3. **`--reservoir` Dependency**:
   - ⚠️ **MANDATORY**: Any script invoking `\reservoir_push <name>` or `\reservoir_pop <name>` (such as `pgbench_graph_add_edge.sql`, `pgbench_ldbc_snb_iu.sql`, `pgbench_vec_insert.sql`, `pgbench_vec_aging.sql`) **MUST** have `--reservoir=<name>:<capacity>` defined in the CLI flags.
4. **SLA Gating & Exit Code Guarantee**:
   - When `--sla` is passed, `pgbench` returns `exit code 2` if any SLA contract is breached.
   - Always pass `--json-report=<path>` to preserve machine-parseable details on SLA pass/fail status.

---

## 4. Deterministic Triage & Error Mitigation Tree

When a subshell or tool invocation returns non-zero, parse `stderr` and apply this decision logic:

![pgbench_agent_decision_matrix diagram 2](/assets/images/pgbench/pgbench_agent_decision_matrix_diagram_2.svg)

| Exit Code | `stderr` / Output Pattern | Root Cause | Automated Remediation Action |
|---|---|---|---|
| **`2`** | `SLA ... [FAIL]` | Performance target or quality gate missed | Read `--json-report`. If `p95` exceeded, reduce client concurrency `-c` or increase duration. If `recall` failed on postfilter, increase `k_oversample`. |
| **`1`** | `reservoir "..." does not exist` | Script used `\reservoir_push` without CLI allocation | Append `--reservoir=<name>:2048` to CLI options. |
| **`1`** | `relation "..." does not exist` | Invariant state skipped; schema or tables not created | Re-run State 1 (`schema.sql`) and State 2 (`init.sql`). |
| **`1`** | `One or more client groups failed to synchronize` | Barrier mismatch in phased run | Ensure all clients execute the same number of `\phase_barrier` calls; do not mix phased and unphased scripts in the same client group. |
| **`1`** | `database "..." does not exist` | Target database not created | Execute `createdb <dbname>` via psql before running benchmark. |
