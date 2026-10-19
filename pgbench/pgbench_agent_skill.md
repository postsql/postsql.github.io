---
layout: page
title: "Agent Skill: Advanced pgbench Workload Execution"
render_with_liquid: false
---

> [!NOTE]
> Companion specification to [Evolving `pgbench` into a Universal Database Benchmark Engine](/2026/10/19/evolving-pgbench.html).

* TOC
{:toc}

You are an expert database performance agent. Use this concise reference to run `pgbench` benchmark suites located in `workloads/`.

## 1. Mandatory Legal & Trademark Guardrails

- **DO NOT** use official trademarked prefixes (`TPC-C`, `CH-benCHmark`, `LDBC`, `VecBench`) in standalone benchmark names.
- **DO NOT** report trademarked council metrics (`tpmC`, `QphH`, `LDBC Benchmark Results`).
- **DO** use project-scoped names: `pgbench-b`, `pgbench-c`, `pgbench-e`, `pgbench-h`, `pgbench-ds`, `pgbench-htap`, `pgbench-graph`, `pgbench-vec`.
- **DO** use generic metrics: `TPS`, `QPM`, `orders/sec`, `Recall@K`, `Degradation %`, `Validation Pass %`.
- **DISCLAIMER**: Always include in user summaries: *"Results represent un-audited internal engineering measurements and do not constitute official certified benchmark results."*

---

## 2. Invariant Execution Pipeline

All suites require a 3-step initialization sequence. Never run execution scripts before completing initialization.

```bash
# 1. DDL Schema (Run once, 1 client)
pgbench -n -d $DB -t 1 -f workloads/$SUITE/${SUITE}_schema.sql

# 2. Parallel In-Database Load (Scale N, partitioned across workers)
pgbench -n -d $DB -c $W -j $W -t 1 -D scale=$SCALE -D num_workers=$W \
        -f workloads/$SUITE/${SUITE}_init.sql

# 3. Secondary Indexes, FKs, & ANALYZE
pgbench -n -d $DB -t 1 -f workloads/$SUITE/${SUITE}_constraints.sql

# 3b. (MANDATORY for pgbench-vec only): Precompute Ground Truth
pgbench -n -d $DB -t 1 -f workloads/pgbench-vec/pgbench_vec_gt_gen.sql
```

---

## 3. Canonical Workload Invocations

### A. Order-Entry OLTP (`pgbench-c`)

```bash
pgbench -n -d $DB -c 16 -j 4 -T 60 -D scale=4 \
        -f workloads/pgbench-c/pgbench_c_new_order.sql@45 \
        -f workloads/pgbench-c/pgbench_c_payment.sql@43 \
        -f workloads/pgbench-c/pgbench_c_order_status.sql@4 \
        -f workloads/pgbench-c/pgbench_c_delivery.sql@4 \
        -f workloads/pgbench-c/pgbench_c_stock_level.sql@4 \
        --sla="total:p95<=5000ms" --json-report=out_c.json
```

### B. Mixed OLTP + OLAP (`pgbench-htap`)

```bash
pgbench -n -d $DB -T 60 -D scale=4 \
        --group="name=oltp,clients=8,threads=4,file=workloads/pgbench-htap/pgbench_htap_oltp.sql" \
        --group="name=olap,clients=2,threads=2,file=workloads/pgbench-htap/pgbench_htap_olap_q1.sql@20,file=workloads/pgbench-htap/pgbench_htap_olap_q3.sql@20,file=workloads/pgbench-htap/pgbench_htap_olap_q5.sql@20,file=workloads/pgbench-htap/pgbench_htap_olap_q6.sql@20,file=workloads/pgbench-htap/pgbench_htap_olap_q18.sql@20" \
        --sla="oltp:p95<=5000ms" --sla="olap:p95<=10000ms" --json-report=out_htap.json
```

### C. Graph Social Network & Fraud (`pgbench-graph`)

```bash
pgbench -n -d $DB -T 60 -s 1 --tcr=0.1 \
        --reservoir=persons:4096 --reservoir=messages:8192 \
        --group="name=snb_readers,clients=4,threads=2,file=workloads/pgbench-graph/pgbench_ldbc_snb_ic.sql@30,file=workloads/pgbench-graph/pgbench_ldbc_snb_is.sql@70" \
        --group="name=snb_writers,clients=2,threads=2,file=workloads/pgbench-graph/pgbench_ldbc_snb_iu.sql@75,file=workloads/pgbench-graph/pgbench_ldbc_snb_id.sql@25" \
        --sla="validation_failures<=0" --json-report=out_graph.json
```

### D. Vector Search & Aging Degradation (`pgbench-vec`)

```bash
# Mode 1, 2, 3, 5 Hybrid Search:
pgbench -n -d $DB -c 4 -j 2 -T 60 -D scale=1 --reservoir=vec_ids:1024 \
        -f workloads/pgbench-vec/pgbench_vec_search.sql@30 \
        -f workloads/pgbench-vec/pgbench_vec_filtered.sql@25 \
        -f workloads/pgbench-vec/pgbench_vec_postfiltered.sql@20 \
        -f workloads/pgbench-vec/pgbench_vec_multivec.sql@15 \
        -f workloads/pgbench-vec/pgbench_vec_insert.sql@10 \
        --sla="total:r_unfiltered:recall>=0.95" \
        --sla="total:r_prefilter:recall>=0.90" \
        --sla="total:r_postfilter:recall>=0.40" \
        --sla="total:r_multivec:recall>=0.85" \
        --sla="total:p95<=100ms" --json-report=out_vec_hybrid.json

# Mode 6 Continuous Aging and Index Degradation:
pgbench -n -d $DB -c 4 -j 2 -T 120 -D scale=1 --reservoir=vec_ids:1024 \
        -f workloads/pgbench-vec/pgbench_vec_aging.sql \
        --sla="total:r_aging:recall>=0.80" --sla="total:p95<=150ms" \
        --json-report=out_vec_aging.json
```

---

## 4. Operational Invariants & Rules

1. **Flag Exclusions**:
   - Never combine `-i` with custom workloads (`-f`).
   - If `--group` is used, do not specify top-level `-f`, `-c`, or `-t`.
2. **Reservoirs**:
   - Always supply `--reservoir=<name>:<size>` when running scripts using `\reservoir_push` or `\reservoir_pop`.
3. **Exit Code 2**:
   - Code `2` indicates an SLA contract failure, not an engine crash. Parse `--json-report` under `sla_verifications` to inspect which target failed.
