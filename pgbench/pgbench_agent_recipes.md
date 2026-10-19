---
layout: page
title: "pgbench Advanced Workloads: Declarative Agent Execution Recipes"
render_with_liquid: false
---

> [!NOTE]
> Companion specification to [Evolving `pgbench` into a Universal Database Benchmark Engine](/2026/10/19/evolving-pgbench.html).

```yaml
# pgbench Advanced Workloads: Declarative Agent Execution Recipes
# Schema Version: 1.0.0
# Target Consumer: Autonomous Orchestrator Agents, Pipeline Drivers, and CI/CD Runners

benchmarks:
  # ---------------------------------------------------------------------------
  # 1. TPC-C-Inspired OLTP Workload (pgbench-c)
  # ---------------------------------------------------------------------------
  - id: pgbench-c
    domain: OLTP / Order Entry
    lineage_disclaimer: "Modeled after TPC-C specification; unaudited internal metric"
    database: bench_oltp_c
    scale_factor: 4
    init_workers: 4
    pre_flight:
      - sql: "SELECT extname FROM pg_extension WHERE extname = 'pgbench';"
        expected_rows: 1
        remediation: "CREATE EXTENSION IF NOT EXISTS pgbench;"
    initialization:
      - step: schema
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-c/pgbench_c_schema.sql"
        expected_exit_code: 0
      - step: data_load
        cmd: "pgbench -n -d {database} -c {init_workers} -j {init_workers} -t 1 -D scale={scale_factor} -D num_workers={init_workers} -f workloads/pgbench-c/pgbench_c_init.sql"
        expected_exit_code: 0
      - step: constraints
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-c/pgbench_c_constraints.sql"
        expected_exit_code: 0
    execution:
      duration_seconds: 60
      clients: 16
      threads: 4
      scripts:
        - path: workloads/pgbench-c/pgbench_c_new_order.sql
          weight: 45
        - path: workloads/pgbench-c/pgbench_c_payment.sql
          weight: 43
        - path: workloads/pgbench-c/pgbench_c_order_status.sql
          weight: 4
        - path: workloads/pgbench-c/pgbench_c_delivery.sql
          weight: 4
        - path: workloads/pgbench-c/pgbench_c_stock_level.sql
          weight: 4
      sla_rules:
        - "total:p95<=5000ms"
      output_json: "report_pgbench_c.json"
    post_flight_assertions:
      - json_path: "$.sla_all_passed"
        operator: "=="
        value: true
      - json_path: "$.transactions_failed"
        operator: "=="
        value: 0

  # ---------------------------------------------------------------------------
  # 2. HTAP Mixed OLTP/OLAP Workload (pgbench-htap)
  # ---------------------------------------------------------------------------
  - id: pgbench-htap
    domain: Hybrid Transactional / Analytical Processing
    lineage_disclaimer: "Modeled after CH-benCHmark community specifications"
    database: bench_htap
    scale_factor: 4
    init_workers: 4
    initialization:
      - step: schema
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-htap/pgbench_htap_schema.sql"
        expected_exit_code: 0
      - step: data_load
        cmd: "pgbench -n -d {database} -c {init_workers} -j {init_workers} -t 1 -D scale={scale_factor} -D num_workers={init_workers} -f workloads/pgbench-htap/pgbench_htap_init.sql"
        expected_exit_code: 0
      - step: constraints
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-htap/pgbench_htap_constraints.sql"
        expected_exit_code: 0
    execution:
      duration_seconds: 60
      groups:
        - name: oltp
          clients: 8
          threads: 4
          scripts:
            - path: workloads/pgbench-htap/pgbench_htap_oltp.sql
              weight: 100
        - name: olap
          clients: 2
          threads: 2
          scripts:
            - path: workloads/pgbench-htap/pgbench_htap_olap_q1.sql
              weight: 20
            - path: workloads/pgbench-htap/pgbench_htap_olap_q3.sql
              weight: 20
            - path: workloads/pgbench-htap/pgbench_htap_olap_q5.sql
              weight: 20
            - path: workloads/pgbench-htap/pgbench_htap_olap_q6.sql
              weight: 20
            - path: workloads/pgbench-htap/pgbench_htap_olap_q18.sql
              weight: 20
      sla_rules:
        - "oltp:p95<=5000ms"
        - "olap:p95<=10000ms"
      output_json: "report_pgbench_htap.json"
    post_flight_assertions:
      - json_path: "$.sla_all_passed"
        operator: "=="
        value: true

  # ---------------------------------------------------------------------------
  # 3. LDBC-Inspired Graph Workload (pgbench-graph)
  # ---------------------------------------------------------------------------
  - id: pgbench-graph
    domain: Property Graph & Financial Fraud Traversal
    lineage_disclaimer: "Modeled after LDBC SNB, FinBench, and Graphalytics; unaudited custom results"
    database: bench_graph
    scale_factor: 1
    init_workers: 4
    initialization:
      - step: schema
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-graph/pgbench_graph_schema.sql"
        expected_exit_code: 0
      - step: data_load
        cmd: "pgbench -n -d {database} -c {init_workers} -j {init_workers} -t 1 -D scale={scale_factor} -D num_workers={init_workers} -f workloads/pgbench-graph/pgbench_graph_init.sql"
        expected_exit_code: 0
      - step: constraints
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-graph/pgbench_graph_constraints.sql"
        expected_exit_code: 0
    execution:
      duration_seconds: 60
      reservoirs:
        - name: persons
          capacity: 4096
        - name: messages
          capacity: 8192
      groups:
        - name: snb_readers
          clients: 4
          threads: 2
          scripts:
            - path: workloads/pgbench-graph/pgbench_ldbc_snb_ic.sql
              weight: 30
            - path: workloads/pgbench-graph/pgbench_ldbc_snb_is.sql
              weight: 70
        - name: snb_writers
          clients: 2
          threads: 2
          scripts:
            - path: workloads/pgbench-graph/pgbench_ldbc_snb_iu.sql
              weight: 75
            - path: workloads/pgbench-graph/pgbench_ldbc_snb_id.sql
              weight: 25
      sla_rules:
        - "validation_failures<=0"
        - "validation_pct>=100.0"
      output_json: "report_pgbench_graph.json"
    post_flight_assertions:
      - json_path: "$.validation_failures"
        operator: "=="
        value: 0
      - json_path: "$.sla_all_passed"
        operator: "=="
        value: true

  # ---------------------------------------------------------------------------
  # 4. VecBench-Inspired Vector Database Workload (pgbench-vec)
  # ---------------------------------------------------------------------------
  - id: pgbench-vec-aging
    domain: Vector Similarity Search & Aging Degradation (Mode 6)
    lineage_disclaimer: "Modeled after VecBench filtered vector search methodology"
    database: bench_vector
    scale_factor: 1
    init_workers: 4
    initialization:
      - step: schema
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-vec/pgbench_vec_schema.sql"
        expected_exit_code: 0
      - step: data_load
        cmd: "pgbench -n -d {database} -c {init_workers} -j {init_workers} -t 1 -D scale={scale_factor} -D num_workers={init_workers} -f workloads/pgbench-vec/pgbench_vec_init.sql"
        expected_exit_code: 0
      - step: constraints
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-vec/pgbench_vec_constraints.sql"
        expected_exit_code: 0
      - step: ground_truth
        cmd: "pgbench -n -d {database} -t 1 -f workloads/pgbench-vec/pgbench_vec_gt_gen.sql"
        expected_exit_code: 0
    execution:
      duration_seconds: 120
      clients: 4
      threads: 2
      reservoirs:
        - name: vec_ids
          capacity: 1024
      scripts:
        - path: workloads/pgbench-vec/pgbench_vec_aging.sql
          weight: 100
      sla_rules:
        - "total:r_aging:recall>=0.80"
        - "total:p95<=150ms"
      output_json: "report_pgbench_vec_aging.json"
    post_flight_assertions:
      - json_path: "$.recall.aging_degradation.initial_recall"
        operator: ">="
        value: 0.80
      - json_path: "$.named_recall.r_aging.avg"
        operator: ">="
        value: 0.80
      - json_path: "$.sla_all_passed"
        operator: "=="
        value: true
```
