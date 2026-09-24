# IBM Db2 LUW (DPF with multiple MLNs) | Deep Dive | Performance Tuning | SELECT DISTINCT vs. GROUP BY

## Summary

A common question in SQL development is whether `SELECT DISTINCT id FROM table_name` can be replaced with `SELECT id FROM table_name GROUP BY id`. 

Logically and mathematically, **both queries produce identical result sets and identical row counts**, including identical handling of `NULL` values. However, in real-world IBM Db2 environments—especially involving **multi-table joins, massive datasets (millions/billions of rows), and Database Partitioning Feature (DPF / MPP) architectures with multiple Multiple Logical Nodes (MLNs)**—the physical execution mechanics diverge drastically.

While a query with multiple joins and `DISTINCT` may take **10–15 minutes**, rewriting it with `GROUP BY` often finishes in **a few seconds**. This post explains the architectural reasons, examines actual Db2 Explain plans, and breaks down why `GROUP BY` scales significantly better in large distributed database topologies.

---

## 1. Logical Equivalence vs. Physical Reality

### Logical Equivalence
* **Deduplication:** Both `DISTINCT` and `GROUP BY` ensure each key value appears exactly once in the final result.
* **NULL Handling:** In standard SQL and IBM Db2, `NULL` values are treated as distinct-equivalent / grouped together into a single `NULL` row.

```sql
-- Query A: DISTINCT
SELECT DISTINCT id 
FROM orders o
JOIN order_items i ON o.order_id = i.order_id
JOIN customers c   ON o.customer_id = c.customer_id;

-- Query B: GROUP BY
SELECT id 
FROM orders o
JOIN order_items i ON o.order_id = i.order_id
JOIN customers c   ON o.customer_id = c.customer_id
GROUP BY id;
```

Both queries output the exact same rows. But how Db2 evaluates them under the hood is fundamentally different when queries grow in complexity.

---

## 2. Why `DISTINCT` Takes 15 Minutes While `GROUP BY` Takes Seconds with Joins

When you execute a query with multiple one-to-many (`1:N`) or many-to-many (`N:M`) joins:

### The `DISTINCT` Pitfall: Late Deduplication
1. **Intermediate Row Explosion:** Each joined table can duplicate rows exponentially (Cartesian expansion in intermediate steps).
2. **Carrying the Bloat:** Db2 often joins the full, inflated datasets first before performing deduplication at the very end of the plan.
3. **Sort Heap Exhaustion & Disk Spills:** A dataset of 100M+ intermediate rows easily exceeds memory thresholds (`SORTHEAP` / `SHEAPTHRES_SHR`). Db2 is forced to spill temporary datasets to physical disk (`TEMPSPACE`), causing heavy I/O, latch contention, and multi-minute runtimes.

### The `GROUP BY` Advantage: Early Aggregation Pushdown
1. **Operation Movement / Pushdown:** The Db2 query rewrite engine recognizes grouping semantics and can push aggregation below join operators.
2. **Early Volume Reduction:** Db2 aggregates/groups rows *before* joining downstream tables, drastically cutting intermediate data flow.
3. **Index & Hash Exploitation:** Grouping columns allow the optimizer to choose index scans that deliver pre-sorted streams, eliminating sort phases altogether.

```
--- DISTINCT Execution Flow ---
[Table A] ─── Join ───> [Table B] ─── Join ───> [Table C] ───> (100M Rows Spilled to Disk) ───> SORT UNIQUE ───> (10K Rows)

--- GROUP BY Execution Flow ---
[Table A] ─── GROUP BY (Early Pushdown) ───> (10K Rows) ─── Join ───> [Table B] ─── Join ───> [Table C] ───> (10K Rows)
```

---

## 3. Explain Plan Analysis: What Happens Under the Hood

When comparing how the Db2 optimizer builds physical access plans, understanding the operators and their execution placement reveals why performance differs:

### In a Single-Table / Base Query Case
* **Distinct Execution:** Db2 performs a scan and sorts the data using an aggregating or unique sort operator (`SORT` with uniqueness enabled) to eliminate duplicate rows locally, then routes the sorted unique stream to the coordinator partition.
* **GROUP BY Execution:** Db2 applies the same local grouping and unique sort logic, then passes the stream through an explicit grouping aggregation operator (`GRPBY`) at the coordinator to enforce group semantics.
* **Result:** On simple single-table queries, both strategies yield nearly identical operational costs and execution times because both rely on the same underlying unique sorting mechanism.

### In a Multi-Table Join Scenario
* **What happens in the Join with DISTINCT case:**
  * **Late Deduplication:** The optimizer prioritizes resolving join predicates across all participating tables first.
  * **Volume Expansion Across Operators:** Join operators (Nested Loop `NLJOIN`, Hash Join `HSJOIN`, or Merge Join `MSJOIN`) process and multiply intermediate row streams across all join legs.
  * **High Memory & I/O Overhead:** Deduplication only occurs at the root/final step after all joins are complete. If intermediate result sets exceed available sort memory (`SORTHEAP`), Db2 spills sorted runs to temporary tablespaces on disk, resulting in severe I/O bottlenecks.
* **What happens in the Join with GROUP BY case:**
  * **Early Aggregation Pushdown:** The optimizer's rewrite engine can push grouping down through the join graph (*Operation Movement*).
  * **Upstream Data Reduction:** Rows from driving tables are grouped and collapsed down to unique keys *before* being fed into subsequent join operators.
  * **Streamlined Joins:** Downstream joins operate on significantly smaller, distinct row sets, fitting easily within memory buffers and eliminating temporary table disk spills.

---

## 4. Why `GROUP BY` Wins in a multiple MLN DPF Environment (Scale: Millions to Billions of Rows)

In a **Db2 Database Partitioning Feature (DPF / MPP)** cluster with **multiple Multiple Logical Nodes (MLNs)**:

```
                      ┌──────────────────────────────────────┐
                      │    Coordinator Partition (COOR)      │
                      └──────────────────▲───────────────────┘
                                         │
                        Merging Table Queue (MDTQ / DTQ)
                                         │
     ┌───────────────────────────────────┼───────────────────────────────────┐
     │ (MLN 1)                           │ (MLN 2)                           │ (MLN multiple)
┌────┴────────────┐                 ┌────┴────────────┐                 ┌────┴────────────┐
│ Local Partition │                 │ Local Partition │                 │ Local Partition │
│  Early Grouping │                 │  Early Grouping │                 │  Early Grouping │
│  (pGRPBY / HASH)│                 │  (pGRPBY / HASH)│                 │  (pGRPBY / HASH)│
└─────────────────┘                 └─────────────────┘                 └─────────────────┘
```

### 1. Inter-Partition Table Queue (TQ) Traffic
* **With `DISTINCT`:** Unaggregated joined intermediate rows must often be broadcast (`BTQ`) or redistributed (`DTQ`) across the interconnect. Shipping billions of rows saturates network bandwidth and causes table queue buffer waits.
* **With `GROUP BY`:** Db2 utilizes **two-phase partial/final aggregation (`PARTIAL_FINAL_REPART` / `pGRPBY`)**. Each of the multiple MLNs aggregates raw data locally in parallel, collapsing billions of rows down to a tiny fraction of distinct keys before any network transmission occurs.

### 2. Coordinator Node Bottleneck Elimination
* In multi-partition environments, merging unreduced data onto the coordinator node (`COOR`) creates a severe single-node CPU and memory bottleneck.
* `GROUP BY` distributes the CPU overhead of deduplication across all multiple MLNs simultaneously.

### 3. Collocation and Distribution Key Exploitation
* If the `GROUP BY` column matches or contains the table's **Distribution Key (Hash Key)**, Db2 can perform **`COMPLETE_LOCAL`** aggregation.
* **Zero network shipping is required:** each node completes grouping independently within its local memory partition.

---

## 5. Architectural Comparison Matrix

| Architectural Dimension | `SELECT DISTINCT` with Joins | `SELECT ... GROUP BY` with Joins |
| :--- | :--- | :--- |
| **Deduplication Timing** | Typically late (post-join, root level) | Early pushdown before or between joins |
| **DPF Strategy (multiple MLNs)** | Coordinator-heavy merge queues (`MDTQ`) | Distributed Two-Phase (`pGRPBY` $\rightarrow$ `DTQ` $\rightarrow$ `GRPBY`) |
| **Interconnect Network Usage** | High (transfers duplicate-inflated rows) | Low (transfers only pre-aggregated keys) |
| **Sort Heap / Memory Impact** | High risk of `SORTHEAP` exhaustion & disk spill | Minimal; local partitions handle manageable chunks |
| **Join Efficiency** | Joins millions of Cartesian rows | Joins pre-collapsed, small cardinality streams |
| **Distribution Key Alignment** | Rarely eliminates cross-node TQ movement | Enables zero-network `COMPLETE_LOCAL` aggregation |

---

## Conclusion & Practical Recommendations

1. **For Simple Single-Table Lookups:** Both `DISTINCT` and `GROUP BY` yield identical execution plans and performance. Use `DISTINCT` for semantic clarity.
2. **For Multi-Table Joins & High Volume:** Prefer `GROUP BY` or pre-aggregated Common Table Expressions (CTEs) / derived tables to enable early aggregation pushdown.
3. **In Db2 DPF / Massive MPP Environments (multiple MLNs):** Always ensure aggregation strategies leverage local parallel processing (`pGRPBY`) and align grouping columns with distribution keys wherever possible to eliminate network bottlenecks.
