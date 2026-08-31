# Mastering IBM Db2 RUNSTATS: Deep Dive into Basic, Distribution & Column Group Statistics

A practical, production-tested guide for Database Administrators and Engineers to understand, execute, and audit `RUNSTATS` in IBM Db2.

---

## 1. Introduction: Why RUNSTATS Matters

The IBM Db2 cost-based optimizer is responsible for choosing the most efficient execution plan for every SQL statement (e.g., selecting index scan vs. table scan, hash join vs. nested-loop join, table access order). 

To do this accurately, the optimizer relies on statistics stored in system catalog tables. Without up-to-date and complete statistics:
* The optimizer falls back to default assumptions (like uniform data distribution and independent multi-column predicates).
* Cardinality estimates become inaccurate, often resulting in slow query performance and high I/O overhead.

Db2 gathers statistics across **three distinct tiers**:
1. **Basic Statistics**: Table and single-column metrics (`CARD`, `COLCARD`, `NUMNULLS`, `LOW2KEY`, `HIGH2KEY`).
2. **Distribution Statistics**: Data skew indicators—Most Frequent Values (MFVs) and Quantile buckets (`SYSCAT.COLDIST`).
3. **Column Group Statistics**: Multi-column correlation and joint cardinality (`SYSCAT.COLGROUPS`, `SYSCAT.COLGROUPCOLS`, `SYSCAT.COLGROUPDIST`).

---

## 2. Target Schema & Dataset

Our examples use the standard `DB2INST1.DEPARTMENT` table with 14 rows:

```sql
CREATE TABLE "DB2INST1"."DEPARTMENT" (
    "DEPTNO"   CHAR(3 OCTETS) NOT NULL,
    "DEPTNAME" VARCHAR(36 OCTETS) NOT NULL,
    "MGRNO"    CHAR(6 OCTETS),
    "ADMRDEPT" CHAR(3 OCTETS) NOT NULL,
    "LOCATION" CHAR(16 OCTETS)
)
DISTRIBUTE BY HASH("DEPTNO")
IN "USERSPACE1"
ORGANIZE BY ROW;
```

### Table Contents (`SELECT * FROM DB2INST1.DEPARTMENT`)

| DEPTNO | DEPTNAME | MGRNO | ADMRDEPT | LOCATION |
| :--- | :--- | :--- | :--- | :--- |
| `E01` | SUPPORT SERVICES | `000050` | `A00` | *NULL* |
| `F22` | BRANCH OFFICE F2 | *NULL* | `E01` | *NULL* |
| `A00` | SPIFFY COMPUTER SERVICE DIV. | `000010` | `A00` | *NULL* |
| `D01` | DEVELOPMENT CENTER | *NULL* | `A00` | *NULL* |
| `E11` | OPERATIONS | `000090` | `E01` | *NULL* |
| `I22` | BRANCH OFFICE I2 | *NULL* | `E01` | *NULL* |
| `J22` | BRANCH OFFICE J2 | *NULL* | `E01` | *NULL* |
| `B01` | PLANNING | `000020` | `A00` | *NULL* |
| `D21` | ADMINISTRATION SYSTEMS | `000070` | `D01` | *NULL* |
| `G22` | BRANCH OFFICE G2 | *NULL* | `E01` | *NULL* |
| `C01` | INFORMATION CENTER | `000030` | `A00` | *NULL* |
| `D11` | MANUFACTURING SYSTEMS | `000060` | `D01` | *NULL* |
| `E21` | SOFTWARE SUPPORT | `000100` | `E01` | *NULL* |
| `H22` | BRANCH OFFICE H2 | *NULL* | `E01` | *NULL* |

---

## 3. Catalog Architecture: Where Statistics Live

```
                        ┌─────────────────────────────────────────┐
                        │              SYSCAT.TABLES              │
                        │ (CARD, NPAGES, FPAGES, STATS_TIME)      │
                        └───────────────────┬─────────────────────┘
                                            │
                    ┌───────────────────────┴────────────────────────┐
                    │                                                │
┌───────────────────▼───────────────────┐        ┌───────────────────▼───────────────────┐
│            SYSCAT.COLUMNS             │        │           SYSCAT.COLGROUPS            │
│ (COLCARD, NUMNULLS, LOW2KEY, HIGH2KEY)│        │ (COLGROUPID, COLGROUPCARD)            │
└───────────────────┬───────────────────┘        └───────────────────┬───────────────────┘
                    │                                                │
┌───────────────────▼───────────────────┐        ┌───────────────────▼───────────────────┐
│            SYSCAT.COLDIST             │        │          SYSCAT.COLGROUPCOLS          │
│ (TYPE: 'F'=Freq, 'Q'=Quantile,        │        │ (COLGROUPID, TABNAME, COLNAME,        │
│  VALCOUNT, COLVALUE)                  │        │  ORDINAL)                             │
└───────────────────────────────────────┘        └───────────────────┬───────────────────┘
                                                                     │
                                                 ┌───────────────────▼───────────────────┐
                                                 │          SYSCAT.COLGROUPDIST          │
                                                 │ (Group Frequent Value Combinations)   │
                                                 └───────────────────────────────────────┘
```

---

## 4. The Master Catalog Audit Query

This SQL query performs `LEFT OUTER JOIN`s across all catalog views using Common Table Expressions (CTEs) and clean `TRIM` logic:

```sql
WITH tbl_summary AS (
    SELECT TABSCHEMA, TABNAME, CARD AS TBL_CARD, NPAGES, FPAGES, STATS_TIME 
    FROM SYSCAT.TABLES 
    WHERE TABSCHEMA = 'DB2INST1' AND TABNAME = 'DEPARTMENT'
), 
dist_summary AS (
    SELECT TABSCHEMA, TABNAME, COLNAME, 
           COUNT(CASE WHEN TYPE = 'F' AND VALCOUNT >= 0 THEN 1 END) AS NUM_FREQ_FOUND, 
           COUNT(CASE WHEN TYPE = 'Q' AND VALCOUNT >= 0 THEN 1 END) AS NUM_QUANT_FOUND 
    FROM SYSCAT.COLDIST 
    WHERE TABSCHEMA = 'DB2INST1' AND TABNAME = 'DEPARTMENT' 
    GROUP BY TABSCHEMA, TABNAME, COLNAME
), 
grp_summary AS (
    SELECT cg.TABSCHEMA, cg.TABNAME, cg.COLNAME, 
           LISTAGG('Grp ' || TRIM(CHAR(cg.COLGROUPID)) || ' (Card=' || TRIM(CHAR(g.COLGROUPCARD)) || ', Ord=' || TRIM(CHAR(cg.ORDINAL)) || ')', '; ') 
               WITHIN GROUP (ORDER BY cg.COLGROUPID) AS GROUP_DETAILS 
    FROM SYSCAT.COLGROUPCOLS cg 
    JOIN SYSCAT.COLGROUPS g ON cg.COLGROUPID = g.COLGROUPID 
    WHERE cg.TABSCHEMA = 'DB2INST1' AND cg.TABNAME = 'DEPARTMENT' 
    GROUP BY cg.TABSCHEMA, cg.TABNAME, cg.COLNAME
) 
SELECT 
    t.STATS_TIME, 
    c.COLNAME, 
    c.COLCARD, 
    c.NUMNULLS, 
    c.LOW2KEY  AS LOW2KEY, 
    c.HIGH2KEY AS HIGH2KEY, 
    d.NUM_FREQ_FOUND AS NMOSTFREQ, 
    d.NUM_QUANT_FOUND AS NQUANTILES, 
    VARCHAR(CASE WHEN c.COLCARD = -1 THEN '[NO] None' ELSE '[YES] Yes' END, 10) AS BASIC_STATS, 
    VARCHAR(CASE 
        WHEN d.COLNAME IS NULL THEN '[NO] None' 
        WHEN COALESCE(d.NUM_FREQ_FOUND, 0) = 0 AND COALESCE(d.NUM_QUANT_FOUND, 0) = 0 THEN '[YES] Checked (no skew)' 
        ELSE '[YES] ' || TRIM(CHAR(COALESCE(d.NUM_FREQ_FOUND, 0))) || 'F, ' || TRIM(CHAR(COALESCE(d.NUM_QUANT_FOUND, 0))) || 'Q' 
    END, 25) AS DIST_STATS, 
    VARCHAR(CASE WHEN g.COLNAME IS NULL THEN '[NO] None' ELSE '[YES] ' || g.GROUP_DETAILS END, 75) AS GROUP_STATS 
FROM tbl_summary t 
JOIN SYSCAT.COLUMNS c ON t.TABSCHEMA = c.TABSCHEMA AND t.TABNAME = c.TABNAME 
LEFT JOIN dist_summary d ON c.TABSCHEMA = d.TABSCHEMA AND c.TABNAME = d.TABNAME AND c.COLNAME = d.COLNAME 
LEFT JOIN grp_summary g ON c.TABSCHEMA = g.TABSCHEMA AND c.TABNAME = g.TABNAME AND c.COLNAME = g.COLNAME 
ORDER BY c.COLNO;
```

---

## 5. Step-by-Step Scenario Walkthrough

### Scenario 1: One Column Group `(DEPTNO, DEPTNAME)`

**Command Executed:**
```bash
db2 "RUNSTATS ON TABLE DB2INST1.DEPARTMENT ON ALL COLUMNS AND COLUMNS (DEPTNO, MGRNO, (DEPTNO, DEPTNAME)) ALLOW WRITE ACCESS"
```

**Audit Query Output:**

| STATS_TIME | COLNAME | COLCARD | NUMNULLS | LOW2KEY | HIGH2KEY | BASIC_STATS | DIST_STATS | GROUP_STATS |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `2026-08-31-02.52.43.931141` | **DEPTNO** | 8 | 0 | `'E01'` | `'F22'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/Grp_2063735285-(Card=8,_Ord=1)-2ea44f?style=flat-square) |
| `2026-08-31-02.52.43.931141` | **DEPTNAME** | 2 | 0 | `'BRANCH OFFICE F2'` | `'SUPPORT SERVICES'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/Grp_2063735285-(Card=8,_Ord=2)-2ea44f?style=flat-square) |
| `2026-08-31-02.52.43.931141` | **MGRNO** | 2 | 4 | `'000050'` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |
| `2026-08-31-02.52.43.931141` | **ADMRDEPT** | 2 | 0 | `'A00'` | `'E01'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |
| `2026-08-31-02.52.43.931141` | **LOCATION** | 1 | 8 | `''` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |

**Takeaways:**
* All columns received basic statistics (`[YES] Yes`).
* `DEPTNO` and `DEPTNAME` were combined into Group `2063735285` with joint cardinality = 8.
* `MGRNO` was analyzed as a single column only, so it has no group association (`[NO] None`).

---

### Scenario 2: Two Column Groups `(DEPTNO, DEPTNAME)` and `(DEPTNO, MGRNO)`

**Command Executed:**
```bash
db2 "RUNSTATS ON TABLE DB2INST1.DEPARTMENT ON ALL COLUMNS AND COLUMNS (DEPTNO, MGRNO, (DEPTNO, DEPTNAME), (DEPTNO, MGRNO)) ALLOW WRITE ACCESS"
```

**Audit Query Output:**

| STATS_TIME | COLNAME | COLCARD | NUMNULLS | LOW2KEY | HIGH2KEY | BASIC_STATS | DIST_STATS | GROUP_STATS |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `2026-08-31-02.52.53.108594` | **DEPTNO** | 8 | 0 | `'E01'` | `'F22'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/Grp_758674350-(Card=8,_Ord=1)-2ea44f?style=flat-square) <br> ![](https://img.shields.io/badge/Grp_924005857-(Card=8,_Ord=1)-2ea44f?style=flat-square) |
| `2026-08-31-02.52.53.108594` | **DEPTNAME** | 2 | 0 | `'BRANCH OFFICE F2'` | `'SUPPORT SERVICES'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/Grp_758674350-(Card=8,_Ord=2)-2ea44f?style=flat-square) |
| `2026-08-31-02.52.53.108594` | **MGRNO** | 2 | 4 | `'000050'` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/Grp_924005857-(Card=8,_Ord=2)-2ea44f?style=flat-square) |
| `2026-08-31-02.52.53.108594` | **ADMRDEPT** | 2 | 0 | `'A00'` | `'E01'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |
| `2026-08-31-02.52.53.108594` | **LOCATION** | 1 | 8 | `''` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |

**Takeaways:**
* `DEPTNO` participates as Ordinal 1 in **both** groups (`Grp 758674350` and `Grp 924005857`).
* `MGRNO` is now registered as Ordinal 2 in `Grp 924005857`.
* Notice that whenever RUNSTATS runs, Db2 removes old group IDs and generates fresh ones.

---

### Scenario 3: Adding Distribution Statistics (`WITH DISTRIBUTION`)

**Command Executed:**
```bash
db2 "RUNSTATS ON TABLE DB2INST1.DEPARTMENT ON ALL COLUMNS AND COLUMNS (DEPTNO, MGRNO, (DEPTNO, DEPTNAME), (DEPTNO, MGRNO)) WITH DISTRIBUTION ALLOW WRITE ACCESS"
```

**Audit Query Output:**

| STATS_TIME | COLNAME | COLCARD | NUMNULLS | LOW2KEY | HIGH2KEY | BASIC_STATS | DIST_STATS | GROUP_STATS |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `2026-08-31-02.18.56.785903` | **DEPTNO** | 8 | 0 | `'E01'` | `'F22'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[YES]-0F,_2Q-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/Grp_1339720353-(Card=8,_Ord=1)-2ea44f?style=flat-square) <br> ![](https://img.shields.io/badge/Grp_1820071294-(Card=8,_Ord=1)-2ea44f?style=flat-square) |
| `2026-08-31-02.18.56.785903` | **DEPTNAME** | 2 | 0 | `'BRANCH OFFICE F2'` | `'SUPPORT SERVICES'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[YES]-0F,_2Q-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/Grp_1339720353-(Card=8,_Ord=2)-2ea44f?style=flat-square) |
| `2026-08-31-02.18.56.785903` | **MGRNO** | 2 | 4 | `'000050'` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[YES]-Checked_(no_skew)-8a3ffc?style=flat-square) | ![](https://img.shields.io/badge/Grp_1820071294-(Card=8,_Ord=2)-2ea44f?style=flat-square) |
| `2026-08-31-02.18.56.785903` | **ADMRDEPT** | 2 | 0 | `'A00'` | `'E01'` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[YES]-0F,_2Q-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |
| `2026-08-31-02.18.56.785903` | **LOCATION** | 1 | 8 | `''` | `''` | ![](https://img.shields.io/badge/[YES]-Yes-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[YES]-1F,_0Q-2ea44f?style=flat-square) | ![](https://img.shields.io/badge/[NO]-None-8c959f?style=flat-square) |

---

## 6. Understanding Distribution Terms & Data Skew

### What "Checked (no skew)" Means for `MGRNO`
1. Db2 evaluated the distribution of `MGRNO` during RUNSTATS.
2. The table has 4 non-null manager entries spread across 2 distinct manager IDs (each ID appears ~2 times).
3. Because the values are **flat and evenly distributed**, no single value crossed the statistical threshold to be marked as a frequent value. Db2 recorded `VALCOUNT = -1` in `SYSCAT.COLDIST`.
4. The optimizer uses a standard uniform mathematical formula:
   ```text
   Expected Rows = (Total Rows - NUMNULLS) / COLCARD
   ```

### Why `LOCATION` Shows `1F, 0Q`
All 8 non-null rows have the identical blank/null value. This represents **extreme data skew**, so Db2 records it as 1 Frequent Value (`VALCOUNT = 8`).

---

## 7. Syntax Gotchas & Best Practices

| Do's | Don'ts |
| :--- | :--- |
| **DO** use `ON ALL COLUMNS AND COLUMNS (...)` to collect both global and group stats. | **DON'T** place `WITH DISTRIBUTION` directly on nested column groups (causes `SQL0270N rc=43`). |
| **DO** use `ALLOW WRITE ACCESS` for online operations. | **DON'T** forget to flush the package cache after `RUNSTATS`. |
| **DO** audit with the unified SQL query before and after major loads. | **DON'T** rely on single-column statistics for heavily correlated queries (e.g. `CITY` + `ZIPCODE`). |

### Recommended Post-RUNSTATS Step
Flush the dynamic SQL cache so currently executing statements pick up the latest access paths:
```sql
db2 "FLUSH PACKAGE CACHE DYNAMIC"
```

---

## 8. Official IBM References & Documentation

* [IBM Db2 RUNSTATS Command Reference](https://www.ibm.com/docs/en/db2/11.5?topic=commands-runstats)
* [Collecting Column Group Statistics in Db2](https://www.ibm.com/docs/en/db2/11.5?topic=statistics-collecting-column-group)
* [SYSCAT.TABLES Catalog View Reference](https://www.ibm.com/docs/en/db2/11.5?topic=views-tables)
* [SYSCAT.COLUMNS Catalog View Reference](https://www.ibm.com/docs/en/db2/11.5?topic=views-columns)
* [SYSCAT.COLDIST Catalog View Reference](https://www.ibm.com/docs/en/db2/11.5?topic=views-coldist)
* [SYSCAT.COLGROUPS & SYSCAT.COLGROUPCOLS](https://www.ibm.com/docs/en/db2/11.5?topic=views-colgroups)
* [SQL0270N Reason Code 43 Resolution](https://www.ibm.com/docs/en/db2/11.5?topic=messages-sql0200-sql0299#sql0270n)
