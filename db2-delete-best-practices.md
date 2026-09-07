# Db2 LUW — Best Practices for Deleting Data from Large Tables

**Published:** Sep 07, 2025  
**Platform:** IBM Db2 LUW 11.5 / 12.x  
**Audience:** Database Administrators, Application Developers

---

## Table of Contents

1. [Why DELETE Is Different from SELECT](#why-delete-is-different-from-select)
2. [What Happens Inside Db2 When You DELETE](#what-happens-inside-db2-when-you-delete)
3. [Why Multi-Column IN Subqueries Are Slow](#why-multi-column-in-subqueries-are-slow)
4. [Why Primary Key DELETE Is Faster](#why-primary-key-delete-is-faster)
5. [Best Practices](#best-practices)
6. [Option 1 — Batch DELETE with Loop (Simple)](#option-1--batch-delete-with-loop-simple)
7. [Option 2 — Stage Keys, Then Batch Delete by Primary Key (Recommended)](#option-2--stage-keys-then-batch-delete-by-primary-key-recommended)
8. [Option 3 — Rewrite as EXISTS](#option-3--rewrite-as-exists)
9. [Option 4 — Stage, Truncate, Re-insert (Bulk Replacement)](#option-4--stage-truncate-re-insert-bulk-replacement)
10. [Add a Composite Index](#add-a-composite-index)
11. [Tuning Parameters to Review](#tuning-parameters-to-review)
12. [Quick Reference Decision Guide](#quick-reference-decision-guide)

---

## Why DELETE Is Different from SELECT

You may have noticed something puzzling: a `SELECT` that returns results in under a second can take hours when turned into a `DELETE` with the exact same `WHERE` clause. 

This is not a bug — it is how relational databases are designed to work.

A **SELECT** is read-only. Db2 reads pages, evaluates predicates, and returns rows. It touches nothing permanently.

A **DELETE** is a full transactional event. For every row it removes, Db2 must:

- Acquire an **exclusive (X) row lock** that is held until you COMMIT
- Write a **before-image** of the entire row to the transaction log
- Write a **delete marker** log record
- Remove the row's key from **every index** on the table — and log each of those index changes too

On a table with billions of rows and multiple indexes, this multiplies quickly.

---

## What Happens Inside Db2 When You DELETE

Consider a table `tab1` with 3 indexes (primary key on `c1`, indexes on `c5` and `c6`). For every row you delete, Db2 generates roughly **4 log records**:

| Event | Log Record |
|---|---|
| Row deleted from data page | ✅ Logged |
| Key removed from PK index (c1) | ✅ Logged |
| Key removed from index on c5 | ✅ Logged |
| Key removed from index on c6 | ✅ Logged |

If you delete 10 million rows in a single transaction, that is **40 million log records** before a single COMMIT. This is why:

- **Log space fills up** — you may hit `SQL0964N` (transaction log full)
- **Performance degrades** — disk I/O for logging becomes the bottleneck
- **Lock list overflows** — millions of row-level `X` locks accumulate, causing Db2 to **escalate to a table-level exclusive lock**, blocking all other users

---

## Why Multi-Column IN Subqueries Are Slow

```sql
-- Slow
DELETE FROM tab1 WHERE c5, c6 IN (SELECT c5, c6 FROM tab2);
```

There are three compounding problems here:

1. **The subquery returns billions of pairs** — Db2 must materialize or hash all `(c5, c6)` pairs from `tab2`, which is itself a billion-row table.

2. **No composite index exists** — you have separate indexes on `c5` and `c6` individually. For a two-column `IN` predicate, Db2 needs a **composite index on `(c5, c6)` together** to do an efficient key lookup. Without it, the optimizer may fall back to a full table scan.

3. **Duplicates are not eliminated** — if `tab2` has many rows sharing the same `(c5, c6)` values, those duplicate pairs cause repeated match attempts, wasting CPU and I/O.

---

## Why Primary Key DELETE Is Faster

```sql
-- Faster
DELETE FROM tab1 WHERE c1 IN (SELECT c1 FROM tab2);
```

- `c1` is the **primary key** — it has a unique clustered index that Db2 can use for direct, one-by-one row lookups.
- The optimizer chooses a **nested loop join**: for each `c1` value from the subquery, it does a single index probe into `tab1`. No full scan needed.
- Primary keys are unique — no duplicate key values, so no wasted work.

This is why **all high-volume deletes should be driven by the primary key**, even if the original filter condition is on other columns.

---

## Best Practices

| # | Practice | Why |
|---|---|---|
| 1 | **Always drive deletes by primary key** | Enables direct index lookup; avoids table scans |
| 2 | **Batch deletes with periodic COMMIT** | Releases locks and recycles log space between batches |
| 3 | **Rewrite IN to EXISTS** | Gives the optimizer more join strategy choices |
| 4 | **Add a composite index for multi-column filters** | One index covers both columns; avoids full scans |
| 5 | **Never delete billions of rows in one transaction** | Log fills up; lock list overflows; blocks all users |
| 6 | **Stage keys once, then delete in a loop** | Avoids re-evaluating the expensive join on every batch |
| 7 | **Run RUNSTATS after mass delete** | Optimizer statistics go stale; query plans degrade |
| 8 | **Use TRUNCATE when deleting almost everything** | Single log record regardless of row count |

---

## Option 1 — Batch DELETE with Loop (Simple)

### What it does
Deletes rows in chunks of N rows, committing after each chunk. The loop runs until no rows remain.

### When to use it
- You need a simple, quick solution
- The filter tables (`tab2`) are not too large
- You can tolerate the join to `tab2` being re-evaluated on every batch

### The Stored Procedure

```sql
CREATE OR REPLACE PROCEDURE delete_tab1_batches()
LANGUAGE SQL
BEGIN
    DECLARE v_rows_deleted INT     DEFAULT 1;
    DECLARE v_total        BIGINT  DEFAULT 0;

    -- Loop until no rows are left to delete
    WHILE v_rows_deleted > 0 DO

        DELETE FROM (
            SELECT t1.*
            FROM tab1 t1
            WHERE EXISTS (
                SELECT 1 FROM tab2 t2
                WHERE t2.c5 = t1.c5
                  AND t2.c6 = t1.c6
            )
            FETCH FIRST 10000 ROWS ONLY   -- adjust batch size as needed
        ) AS batch;

        GET DIAGNOSTICS v_rows_deleted = ROW_COUNT;
        SET v_total = v_total + v_rows_deleted;

        COMMIT;   -- releases all row locks; log space is recycled here

    END WHILE;

END@

-- Run it:
CALL delete_tab1_batches()@
```

### How the loop works

```
First run:   Deletes rows 1–10,000   → COMMIT → v_rows_deleted = 10000 → continue
Second run:  Deletes rows 10,001–20,000 → COMMIT → continue
...
Last run:    Deletes remaining rows → COMMIT → v_rows_deleted = 0 → EXIT loop
```

> **Tip:** Tune `FETCH FIRST N ROWS ONLY`. Start at 10,000 and monitor log usage. Increase to 50,000 if log space allows; decrease if you see lock contention.

---

## Option 2 — Stage Keys, Then Batch Delete by Primary Key (Recommended)

### What it does
Joins `tab1` and `tab2` **once** to collect all primary keys (`c1`) that need to be deleted, stores them in a temporary table, then deletes in batches using the fast primary key path.

### When to use it
- **Best choice for very large tables** (billions of rows)
- `tab2` data does not change during the delete operation
- You want maximum speed per batch (primary key lookup is the fastest possible access)

### The Stored Procedure

```sql
CREATE OR REPLACE PROCEDURE delete_tab1_by_pk_batches()
LANGUAGE SQL
BEGIN
    DECLARE v_rows INT DEFAULT 1;

    -- Step 1: Create temp table to hold PKs to delete
    DECLARE GLOBAL TEMPORARY TABLE session.del_keys (
        c1 BIGINT   -- match the actual data type of your primary key c1
    )
    ON COMMIT PRESERVE ROWS   -- temp table survives across COMMITs
    NOT LOGGED                -- no logging for the temp table itself
    WITH REPLACE;             -- drop and recreate if it already exists

    -- Step 2: Populate with matching PKs — expensive join runs ONCE
    INSERT INTO session.del_keys
        SELECT t1.c1
        FROM   tab1 t1
        WHERE  EXISTS (
                   SELECT 1
                   FROM   tab2 t2
                   WHERE  t2.c5 = t1.c5
                   AND    t2.c6 = t1.c6
               );

    -- Step 3: Loop — delete a batch by PK, shrink the key list, repeat
    WHILE v_rows > 0 DO

        -- Fast PK lookup — no join to tab2 needed anymore
        DELETE FROM tab1
        WHERE c1 IN (
            SELECT c1 FROM session.del_keys
            FETCH FIRST 10000 ROWS ONLY
        );

        GET DIAGNOSTICS v_rows = ROW_COUNT;

        -- Remove processed keys from the staging table
        DELETE FROM session.del_keys
        WHERE c1 IN (
            SELECT c1 FROM session.del_keys
            FETCH FIRST 10000 ROWS ONLY
        );

        COMMIT;   -- ON COMMIT PRESERVE ROWS keeps the temp table alive

    END WHILE;

END@

-- Run it:
CALL delete_tab1_by_pk_batches()@
```

### Why this is faster than Option 1

| | Option 1 | Option 2 |
|---|---|---|
| Join to tab2 | Every single batch | **Once only** |
| Access path per batch | EXISTS subquery scan | **Primary key index lookup** |
| Good when tab2 is huge | No (repeated cost) | **Yes** |
| Handles tab2 data changes during delete | Yes | No (snapshot at start) |

---

## Option 3 — Rewrite as EXISTS

### What it does
Replaces the `IN (subquery)` pattern with `EXISTS`, which gives the Db2 optimizer freedom to choose the best join strategy (hash join, merge join, or nested loop).

### When to use it
- As an improvement to any of the other options
- Always prefer `EXISTS` over `IN` for correlated subqueries on large tables

```sql
-- Instead of this:
DELETE FROM tab1
WHERE (c5, c6) IN (SELECT c5, c6 FROM tab2);

-- Use this:
DELETE FROM tab1 t1
WHERE EXISTS (
    SELECT 1
    FROM tab2 t2
    WHERE t2.c5 = t1.c5
      AND t2.c6 = t1.c6
);
```

**Combine this with batching for the full benefit:**

```sql
DELETE FROM (
    SELECT t1.*
    FROM tab1 t1
    WHERE EXISTS (
        SELECT 1 FROM tab2 t2
        WHERE t2.c5 = t1.c5
          AND t2.c6 = t1.c6
    )
    FETCH FIRST 10000 ROWS ONLY
) AS batch;
COMMIT;
```

---

## Option 4 — Stage, Truncate, Re-insert (Bulk Replacement)

### What it does
Instead of deleting unwanted rows, you keep the rows you **want** in a staging table, truncate the original (one log record, not millions), then repopulate.

### When to use it
- You are deleting **more than 50% of the table**
- You can tolerate a maintenance window (table is unavailable during the operation)
- The insert volume fits your log and temp space

```sql
-- Step 1: Create a staging table
CREATE TABLE tab1_keep LIKE tab1;

-- Step 2: Copy rows you want to KEEP
INSERT INTO tab1_keep
    SELECT * FROM tab1 t1
    WHERE NOT EXISTS (
        SELECT 1 FROM tab2 t2
        WHERE t2.c5 = t1.c5
          AND t2.c6 = t1.c6
    );

-- Step 3: Truncate the original — single log record, instant
TRUNCATE TABLE tab1 IMMEDIATE;

-- Step 4: Repopulate
INSERT INTO tab1 SELECT * FROM tab1_keep;

-- Step 5: Clean up
DROP TABLE tab1_keep;

-- Step 6: Rebuild statistics
RUNSTATS ON TABLE tab1 WITH DISTRIBUTION AND DETAILED INDEXES ALL;
```

> **Important:** Ensure all indexes and constraints are in place before repopulating. `TRUNCATE` removes rows but keeps the table structure and indexes.

---

## Add a Composite Index

If your delete condition is always on `(c5, c6)` together, a composite index on both columns is one of the highest-impact changes you can make.

```sql
-- Create composite indexes on both tables
CREATE INDEX idx_tab1_c5c6 ON tab1 (c5, c6);
CREATE INDEX idx_tab2_c5c6 ON tab2 (c5, c6);
```

### Why this helps

| Scenario | Without composite index | With composite index |
|---|---|---|
| Filter on c5=X AND c6=Y | Two separate index scans merged, or full table scan | Single precise range scan |
| EXISTS subquery resolution | Scans tab2 broadly | Targeted key lookup on tab2 |
| Nested loop join probe | No direct entry point | Direct (c5,c6) key probe |

> After creating the index, run `RUNSTATS` and `REORG` to let the optimizer use current statistics:
> ```sql
> RUNSTATS ON TABLE tab1 WITH DISTRIBUTION AND DETAILED INDEXES ALL;
> REORG INDEXES ALL FOR TABLE tab1;
> ```

---

## Tuning Parameters to Review

| Parameter | What to Change | Why |
|---|---|---|
| `LOGFILSIZ` | Increase | Larger individual log files reduce log switch overhead |
| `LOGPRIMARY` | Increase | More primary log files = more active log space before archiving |
| `LOGSECOND` | Increase | Secondary log files act as overflow buffer |
| `LOCKLIST` | Increase | Delays lock escalation from row locks to table-level X lock |
| `MAXLOCKS` | Increase (e.g. 80) | Higher % of lock list per application before escalation |
| `LOGBUFSZ` | Increase | Larger in-memory log buffer = fewer log disk writes per batch |

```sql
-- Example: increase lock list and log space
UPDATE DB CFG FOR yourdb USING LOCKLIST 65536;
UPDATE DB CFG FOR yourdb USING MAXLOCKS 80;
UPDATE DB CFG FOR yourdb USING LOGFILSIZ 65536;
UPDATE DB CFG FOR yourdb USING LOGPRIMARY 20;
UPDATE DB CFG FOR yourdb USING LOGSECOND 100;
```

> Always run `RUNSTATS` after any mass delete operation to keep optimizer statistics current:
> ```sql
> RUNSTATS ON TABLE tab1 WITH DISTRIBUTION AND DETAILED INDEXES ALL;
> ```

---

## Quick Reference Decision Guide

```
Is the table mostly empty after delete (>50% rows removed)?
  YES → Use Option 4 (Truncate + Re-insert)
  NO  → Continue...

Is tab2 stable (not changing during delete)?
  YES → Use Option 2 (Stage PKs once, batch delete by PK) ← Best
  NO  → Use Option 1 (Batch DELETE with EXISTS loop)

In all cases:
  - Rewrite IN to EXISTS (Option 3 style)
  - Add composite index on (c5, c6)
  - Run RUNSTATS after completion
```

---

*Published: Sep 07, 2026*
