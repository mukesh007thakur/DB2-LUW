# A Calendar-Aware TIMESTAMPDIFF for IBM Db2 LUW

> **Platform:** IBM Db2 LUW 12.1  
> **Schema:** `DB2INST1`  
> **Function:** `DB2INST1.TIMESTAMPDIFF(INTEGER, TIMESTAMP, TIMESTAMP)`  
> **References:** [TIMESTAMPDIFF](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-timestampdiff) · [EXTRACT](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-extract) · [YMD_BETWEEN](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-ymd-between)  ·  
> **Author:** Mukesh Kumar · [mkt@ibm.com](mailto:mkt@ibm.com)

---

## About the Function Signature

`SYSIBM.TIMESTAMPDIFF` has the signature `(INTEGER, CHAR/VARCHAR)` — the second argument is a **pre-computed timestamp duration string**, produced by calling `CHAR(ts1 - ts2)` outside the function. Replicating that signature in `DB2INST1` is not possible and would defeat the purpose: if we accepted the same duration string, we would receive the same 30-day-month estimate that `SYSIBM.TIMESTAMPDIFF` already produces, with nothing to improve.

`DB2INST1.TIMESTAMPDIFF` therefore accepts **two raw TIMESTAMP values** as its second and third arguments, performs its own subtraction using `EXTRACT(EPOCH)` and `YMD_BETWEEN`, and returns accurate results.

| Function | Schema | Signature | Conflict? |
|---|---|---|---|
| `TIMESTAMPDIFF` | `SYSIBM` | `(INTEGER, VARCHAR)` | No — 2 args |
| `TIMESTAMPDIFF` | `DB2INST1` | `(INTEGER, TIMESTAMP, TIMESTAMP)` | No — 3 args |

Db2 resolves function calls by **both schema path and full argument signature**. An unqualified call with 3 arguments resolves to `DB2INST1.TIMESTAMPDIFF` only if `DB2INST1` is on `CURRENT PATH`. Using the fully qualified name `DB2INST1.TIMESTAMPDIFF(...)` in all calling code is always the safest approach.

---

## Scope and Accuracy

This function improves on `SYSIBM.TIMESTAMPDIFF` for **fixed-length intervals** (seconds, minutes, hours, days, weeks) by replacing fixed-assumption arithmetic with `EXTRACT(EPOCH)` division. For **calendar intervals** (months, quarters, years) it delegates to `SYSIBM.YMD_BETWEEN`, which uses IBM's full-precision timestamp arithmetic internally.

**Known limitations:**

| Interval | Limitation |
|---|---|
| Microseconds (`1`) | `EXTRACT(EPOCH)` returns `BIGINT` whole seconds. Multiplying by 1,000,000 yields microseconds at **whole-second resolution only**. Fractional microseconds within the last second are not included. |
| Unsupported interval codes | Returns `NULL`. Callers must validate `P_INTERVAL`. |

---

## 1. What Is Happening with SYSIBM.TIMESTAMPDIFF?

`SYSIBM.TIMESTAMPDIFF` is documented to return an **estimate**, not an exact value. From IBM documentation:

> *"The returned estimate may vary by a number of days. For example, if the number of days (interval 16) is requested for the difference between '1997-03-01-00.00.00' and '1997-02-01-00.00.00', the result is 30. This is because the difference between the timestamps is 1 month, and the assumption of 30 days in a month applies."*

The native function works by:
1. Subtracting two `TIMESTAMP` values to produce a **timestamp duration string** (format `YYYYMMDDHHMMSS.ffffff`)
2. Converting that to `CHAR` and passing it as the second argument
3. Applying **fixed calendar assumptions** to convert the duration components into the requested interval

```sql
-- Native (two-argument form) — returns an estimate
VALUES TIMESTAMPDIFF(16, CHAR(TIMESTAMP('1997-03-01 00:00:00')
                           - TIMESTAMP('1997-02-01 00:00:00')));
-- Returns: 30  (wrong — February 1997 has only 28 days)

-- DB2INST1 (three-argument form) — returns accurate result
VALUES DB2INST1.TIMESTAMPDIFF(16,
    TIMESTAMP('1997-03-01 00:00:00'),
    TIMESTAMP('1997-02-01 00:00:00'));
-- Returns: 28
```

---

## 2. Why the Estimates Are Wrong

### Fixed Assumptions Used by SYSIBM.TIMESTAMPDIFF

| Assumption | Value |
|---|---|
| Days in a month | **30** (fixed) |
| Days in a year | **365** (fixed) |
| Hours in a day | 24 |
| Minutes in an hour | 60 |
| Seconds in a minute | 60 |

### Native Computation for Days

The subtraction of `'1997-03-01'` minus `'1997-02-01'` produces the duration string `0 years, 1 month, 0 days…`. Applying the Days formula:

```
days = 0 + (1 × 30) + (0 × 365) = 30  ← wrong
```

The real answer is **28** (February 1997 has 28 days).

---

## 3. The Math

### EXTRACT(EPOCH) path — exact for fixed-length intervals

```
EXTRACT(EPOCH FROM TIMESTAMP('1997-03-01 00:00:00')) = 857,174,400
EXTRACT(EPOCH FROM TIMESTAMP('1997-02-01 00:00:00')) = 854,755,200

Difference = 2,419,200 seconds

Days  = 2,419,200 / 86,400    = 28   ✓
Hours = 2,419,200 / 3,600     = 672  ✓
```

### YMD_BETWEEN path — exact for calendar intervals

`SYSIBM.YMD_BETWEEN` returns a packed `INTEGER` in the form `±YYYYMMDD` — the number of full years, full months, and full days between two timestamps, computed using IBM's own 12-digit precision arithmetic. IBM documents it as:

```sql
INTEGER( ( TIMESTAMP(expr1, 12) – TIMESTAMP(expr2, 12) ) / 1000000 )
```

Unpacking the packed value:

```sql
raw / 10000                        -- years component
MOD(raw / 100, 100)                -- months component
MOD(raw, 100)                      -- days component

-- Total months (used for intervals 64, 128, 256):
BIGINT(raw / 10000) * 12 + BIGINT(MOD(raw / 100, 100))
```

### All Intervals — Feb 1 → Mar 1, 1997 (28 actual days)

| Code | Interval | SYSIBM (estimate) | DB2INST1 (accurate) |
|---:|---|---:|---:|
| 1 | Microseconds | 2,592,000,000,000 | 2,419,200,000,000 ¹ |
| 2 | Seconds | 2,592,000 | 2,419,200 |
| 4 | Minutes | 43,200 | 40,320 |
| 8 | Hours | 720 | 672 |
| **16** | **Days** | **30** | **28** |
| 32 | Weeks | 4 | 4 |
| 64 | Months | 1 | 1 |
| 128 | Quarters | 0 | 0 |
| 256 | Years | 0 | 0 |

¹ Whole-second resolution only.

---

## 4. Why a Month Cannot Be a Fixed Number of Seconds

| Period | Seconds |
|---|---|
| February (non-leap year) | 28 × 86,400 = **2,419,200** |
| February (leap year) | 29 × 86,400 = **2,505,600** |
| 30-day month | 30 × 86,400 = **2,592,000** ← SYSIBM assumption |
| 31-day month | 31 × 86,400 = **2,678,400** |
| Non-leap year | 365 × 86,400 = **31,536,000** |
| Leap year | 366 × 86,400 = **31,622,400** |

Using a constant like `epoch_diff / 2592000` for months, or `epoch_diff / 31104000` for years (360-day year), reintroduces the same fixed-assumption bug. The EMPLOYEE table confirms this: 37 out of 42 employees get a wrong age (off by 1 year) when the 360-day shortcut is used.

---

## 5. Function DDL

```sql
CREATE OR REPLACE FUNCTION DB2INST1.TIMESTAMPDIFF (
    P_INTERVAL   INTEGER,
    P_TS1        TIMESTAMP,
    P_TS2        TIMESTAMP
)
RETURNS BIGINT
LANGUAGE SQL
DETERMINISTIC
NO EXTERNAL ACTION
READS SQL DATA
BEGIN
    DECLARE V_EPOCH_DIFF   BIGINT;
    DECLARE V_YMD          INTEGER;
    DECLARE V_TOTAL_MONTHS BIGINT;
    DECLARE V_RESULT       BIGINT;

    -- Fixed-length intervals: EXTRACT(EPOCH) division.
    -- Returns BIGINT whole seconds; positive when P_TS1 > P_TS2.
    -- Note: sub-second precision is truncated.
    SET V_EPOCH_DIFF = EXTRACT(EPOCH FROM P_TS1) - EXTRACT(EPOCH FROM P_TS2);

    -- Calendar intervals: delegate to YMD_BETWEEN.
    -- Returns packed INTEGER as YYYYMMDD.
    -- Handles time-of-day, nanoseconds, negative spans, leap days,
    -- and month-end transitions natively via IBM internal arithmetic.
    SET V_YMD = YMD_BETWEEN(P_TS1, P_TS2);

    -- Total months = (years component × 12) + months component
    SET V_TOTAL_MONTHS =
          BIGINT(V_YMD / 10000) * 12
        + BIGINT(MOD(V_YMD / 100, 100));

    SET V_RESULT =
        CASE P_INTERVAL
            -- Fixed-length: EXTRACT(EPOCH) — no calendar assumptions
            WHEN 1   THEN V_EPOCH_DIFF * 1000000   -- Microseconds (whole-second resolution)
            WHEN 2   THEN V_EPOCH_DIFF              -- Seconds
            WHEN 4   THEN V_EPOCH_DIFF / 60         -- Minutes
            WHEN 8   THEN V_EPOCH_DIFF / 3600       -- Hours
            WHEN 16  THEN V_EPOCH_DIFF / 86400      -- Days
            WHEN 32  THEN V_EPOCH_DIFF / 604800     -- Weeks

            -- Calendar: YMD_BETWEEN — IBM-defined, all edge cases handled
            WHEN 64  THEN V_TOTAL_MONTHS            -- Months
            WHEN 128 THEN V_TOTAL_MONTHS / 3        -- Quarters
            WHEN 256 THEN V_TOTAL_MONTHS / 12       -- Years

            -- Unsupported code: returns NULL. Caller must validate P_INTERVAL.
            ELSE NULL
        END;

    RETURN V_RESULT;
END@
```

---

## 6. Deployment

### Standard deployment (function already exists)

```sql
db2 -td@ -vf new_timestampdiff.sql -z new_timestampdiff.sql.out
```


---

## 7. Schema-Path Resolution

```sql
-- Check current path
VALUES CURRENT PATH;

-- Add DB2INST1 for a session
SET PATH = SYSTEM PATH, DB2INST1;
```

Always use the fully qualified name `DB2INST1.TIMESTAMPDIFF(...)` in all calling code.

---

## 8. Usage

```sql
-- SYSIBM native (estimate, 2-argument form):
SELECT TIMESTAMPDIFF(16, CHAR(END_TS - START_TS)) FROM MY_TABLE;

-- DB2INST1 accurate (3-argument form):
SELECT DB2INST1.TIMESTAMPDIFF(16, END_TS, START_TS) FROM MY_TABLE;
```

---

## 9. What Cannot Be Done

| Limitation | Detail |
|---|---|
| Sub-second microsecond precision | `EXTRACT(EPOCH)` truncates to whole seconds. To include fractional microseconds, add `EXTRACT(MICROSECONDS FROM P_TS1) − EXTRACT(MICROSECONDS FROM P_TS2)` with second-boundary handling. |
| Unsupported interval codes | Passing any value other than 1, 2, 4, 8, 16, 32, 64, 128, 256 returns `NULL`. |

---

## 10. Application-Local Time & DST

> Validate behaviour where application timestamps represent local civil time across daylight-saving transitions. The expected result depends on the timestamp data type, database configuration, and how the application stores time-zone context. If your Db2 instance records timestamps in local time, a clock-change day may have 23 or 25 hours, which will affect hour, minute, second, day, and week intervals by ±1 hour.

---

## Example Group 1 — Raw YMD_BETWEEN unpacking

```sql
SELECT
    P_TS1,
    P_TS2,
    YMD_BETWEEN(P_TS1, P_TS2)                           AS RAW_YMD,
    YMD_BETWEEN(P_TS1, P_TS2) / 10000                   AS YRS,
    MOD(YMD_BETWEEN(P_TS1, P_TS2) / 100, 100)           AS MOS,
    MOD(YMD_BETWEEN(P_TS1, P_TS2), 100)                 AS DYS,
    BIGINT(YMD_BETWEEN(P_TS1, P_TS2) / 10000) * 12
  + BIGINT(MOD(YMD_BETWEEN(P_TS1, P_TS2) / 100, 100))  AS TOTAL_MOS,
    LABEL
FROM (
    VALUES
      (TIMESTAMP('2016-03-01 23:59:59'), TIMESTAMP('2013-09-23 23:59:59'), 'IBM doc: 2yr 5mo 8d'),
      (TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-29 00:00:00'), '1 day, leap'),
      (TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-01 00:00:00'), '1 month exact'),
      (TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00'), 'Feb97: 28 days'),
      (TIMESTAMP('2026-03-15 09:00:00'), TIMESTAMP('2026-02-15 10:00:00'), 'Same day, time before'),
      (TIMESTAMP('2026-03-15 11:00:00'), TIMESTAMP('2026-02-15 10:00:00'), 'Same day, time after'),
      (TIMESTAMP('2025-02-28 00:00:00'), TIMESTAMP('2025-01-31 00:00:00'), 'Jan31 -> Feb28'),
      (TIMESTAMP('2025-04-30 00:00:00'), TIMESTAMP('2025-03-31 00:00:00'), 'Mar31 -> Apr30'),
      (TIMESTAMP('2025-05-31 00:00:00'), TIMESTAMP('2025-04-30 00:00:00'), 'Apr30 -> May31'),
      (TIMESTAMP('2025-06-15 00:00:00'), TIMESTAMP('2025-01-15 00:00:00'), 'Negative: -5 months'),
      (TIMESTAMP('2000-01-01 00:00:00'), TIMESTAMP('1969-12-31 00:00:00'), 'Pre-1970 span'),
      (TIMESTAMP('2025-06-15 12:00:00'), TIMESTAMP('2025-06-15 12:00:00'), 'Identical timestamps')
) AS T(P_TS1, P_TS2, LABEL)
ORDER BY LABEL;
```

**Output:**

```
P_TS1                    P_TS2                    RAW_YMD   YRS  MOS  DYS  TOTAL_MOS  LABEL
-----------------------  -----------------------  --------  ---  ---  ---  ---------  ----------------------
2000-03-01 00:00:00      2000-02-29 00:00:00             1    0    0    1          0  1 day, leap
2000-03-01 00:00:00      2000-02-01 00:00:00           100    0    1    0          1  1 month exact
2025-05-31 00:00:00      2025-04-30 00:00:00           101    0    1    1          1  Apr30 -> May31
1997-03-01 00:00:00      1997-02-01 00:00:00           100    0    1    0          1  Feb97: 28 days
2016-03-01 23:59:59      2013-09-23 23:59:59         20508    2    5    8         29  IBM doc: 2yr 5mo 8d
2025-06-15 12:00:00      2025-06-15 12:00:00             0    0    0    0          0  Identical timestamps
2025-02-28 00:00:00      2025-01-31 00:00:00            28    0    0   28          0  Jan31 -> Feb28
2025-04-30 00:00:00      2025-03-31 00:00:00            30    0    0   30          0  Mar31 -> Apr30
2025-06-15 00:00:00      2025-01-15 00:00:00           500    0    5    0          5  Negative: -5 months
2000-01-01 00:00:00      1969-12-31 00:00:00        300001   30    0    1        360  Pre-1970 span
2026-03-15 11:00:00      2026-02-15 10:00:00           100    0    1    0          1  Same day, time after
2026-03-15 09:00:00      2026-02-15 10:00:00            27    0    0   27          0  Same day, time before
```

---

## Example Group 2 — All 9 interval codes

```sql
SELECT CODE, LABEL, NATIVE_APPROX,
    DB2INST1.TIMESTAMPDIFF(CODE,
        TIMESTAMP('1997-03-01 00:00:00'),
        TIMESTAMP('1997-02-01 00:00:00')) AS ACCURATE
FROM (
    SELECT  2 AS CODE, 'Seconds'  AS LABEL, TIMESTAMPDIFF(2,  CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) AS NATIVE_APPROX FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT  4, 'Minutes',  TIMESTAMPDIFF( 4, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT  8, 'Hours',    TIMESTAMPDIFF( 8, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT 16, 'Days',     TIMESTAMPDIFF(16, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT 32, 'Weeks',    TIMESTAMPDIFF(32, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT 64, 'Months',   TIMESTAMPDIFF(64, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT 128,'Quarters', TIMESTAMPDIFF(128,CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
    UNION ALL SELECT 256,'Years',    TIMESTAMPDIFF(256,CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) FROM SYSIBM.SYSDUMMY1
) T
ORDER BY CODE;
```

**Output:**

```
CODE  LABEL      NATIVE_APPROX  ACCURATE
----  ---------  -------------  --------
   2  Seconds        2,592,000  2,419,200
   4  Minutes           43,200     40,320
   8  Hours                720        672
  16  Days                  30         28
  32  Weeks                  4          4
  64  Months                 1          1
 128  Quarters               0          0
 256  Years                  0          0
```

---

## Example Group 3 — Calendar edge cases

```sql
SELECT
    TIMESTAMP('2000-03-01 00:00:00') AS P_TS1, TIMESTAMP('2000-02-29 00:00:00') AS P_TS2,
    'Leap day 1d'             AS LABEL,
    DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-29 00:00:00')) AS RESULT
FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-01 00:00:00'), '1 month exact',             DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2026-03-15 09:00:00'), TIMESTAMP('2026-02-15 10:00:00'), 'Time BEFORE (0 months)',    DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2026-03-15 09:00:00'), TIMESTAMP('2026-02-15 10:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2026-03-15 10:00:00'), TIMESTAMP('2026-02-15 10:00:00'), 'Time EXACT  (1 month)',     DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2026-03-15 10:00:00'), TIMESTAMP('2026-02-15 10:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2026-03-15 11:00:00'), TIMESTAMP('2026-02-15 10:00:00'), 'Time AFTER  (1 month)',     DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2026-03-15 11:00:00'), TIMESTAMP('2026-02-15 10:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-02-28 00:00:00'), TIMESTAMP('2025-01-31 00:00:00'), 'Jan31->Feb28 (0 months)',   DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2025-02-28 00:00:00'), TIMESTAMP('2025-01-31 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-04-30 00:00:00'), TIMESTAMP('2025-03-31 00:00:00'), 'Mar31->Apr30 (0 months)',   DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2025-04-30 00:00:00'), TIMESTAMP('2025-03-31 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-05-31 00:00:00'), TIMESTAMP('2025-04-30 00:00:00'), 'Apr30->May31 (1 month)',    DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2025-05-31 00:00:00'), TIMESTAMP('2025-04-30 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-01-15 00:00:00'), TIMESTAMP('2025-06-15 00:00:00'), 'Negative: -5 months',       DB2INST1.TIMESTAMPDIFF(64,  TIMESTAMP('2025-01-15 00:00:00'), TIMESTAMP('2025-06-15 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2001-03-01 00:00:00'), TIMESTAMP('2000-03-01 00:00:00'), '1 full year (256)',          DB2INST1.TIMESTAMPDIFF(256, TIMESTAMP('2001-03-01 00:00:00'), TIMESTAMP('2000-03-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-29 00:00:00'), '1 day = 0 years (256)',      DB2INST1.TIMESTAMPDIFF(256, TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-29 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2001-02-28 00:00:00'), TIMESTAMP('2000-03-01 00:00:00'), '364d = 0 years (256)',       DB2INST1.TIMESTAMPDIFF(256, TIMESTAMP('2001-02-28 00:00:00'), TIMESTAMP('2000-03-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('1970-01-01 00:00:00'), TIMESTAMP('1960-01-01 00:00:00'), 'Pre-1970: 10 years (256)',   DB2INST1.TIMESTAMPDIFF(256, TIMESTAMP('1970-01-01 00:00:00'), TIMESTAMP('1960-01-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-10-01 00:00:00'), TIMESTAMP('2025-01-01 00:00:00'), '3 quarters (128)',            DB2INST1.TIMESTAMPDIFF(128, TIMESTAMP('2025-10-01 00:00:00'), TIMESTAMP('2025-01-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
UNION ALL SELECT TIMESTAMP('2025-06-15 00:00:00'), TIMESTAMP('2025-01-01 00:00:00'), 'Bad code=99 -> NULL',         DB2INST1.TIMESTAMPDIFF(99,  TIMESTAMP('2025-06-15 00:00:00'), TIMESTAMP('2025-01-01 00:00:00')) FROM SYSIBM.SYSDUMMY1
ORDER BY LABEL;
```

**Output:**

```
P_TS1                    P_TS2                    LABEL                     RESULT
-----------------------  -----------------------  ------------------------  ------
2000-03-01 00:00:00      2000-02-29 00:00:00      1 day = 0 years (256)          0
2000-03-01 00:00:00      2000-02-01 00:00:00      1 month exact                  1
2026-03-15 09:00:00      2026-02-15 10:00:00      Time BEFORE (0 months)         0
2026-03-15 10:00:00      2026-02-15 10:00:00      Time EXACT  (1 month)          1
2026-03-15 11:00:00      2026-02-15 10:00:00      Time AFTER  (1 month)          1
2025-02-28 00:00:00      2025-01-31 00:00:00      Jan31->Feb28 (0 months)        0
2025-04-30 00:00:00      2025-03-31 00:00:00      Mar31->Apr30 (0 months)        0
2025-05-31 00:00:00      2025-04-30 00:00:00      Apr30->May31 (1 month)         1
2025-01-15 00:00:00      2025-06-15 00:00:00      Negative: -5 months           -5
2001-03-01 00:00:00      2000-03-01 00:00:00      1 full year (256)              1
2000-03-01 00:00:00      2000-02-29 00:00:00      Leap day 1d                    0
2001-02-28 00:00:00      2000-03-01 00:00:00      364d = 0 years (256)           0
1970-01-01 00:00:00      1960-01-01 00:00:00      Pre-1970: 10 years (256)      10
2025-10-01 00:00:00      2025-01-01 00:00:00      3 quarters (128)               3
2025-06-15 00:00:00      2025-01-01 00:00:00      Bad code=99 -> NULL            -
```

---

## Example Group 4 — EMPLOYEE tenure breakdown

```sql
SELECT
    EMPNO,
    FIRSTNME || ' ' || LASTNAME                          AS NAME,
    HIREDATE,
    CURRENT DATE                                         AS TODAY,

    -- Components direct from YMD_BETWEEN
    YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) / 10000
                                                         AS SVC_YEARS,
    MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) / 100, 100)
                                                         AS SVC_MONTHS,
    MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)), 100)
                                                         AS SVC_DAYS,

    -- Human-readable: "49y 5m 10d"
    TRIM(CHAR(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) / 10000))
        || 'y '
    || TRIM(CHAR(MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) / 100, 100)))
        || 'm '
    || TRIM(CHAR(MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)), 100)))
        || 'd'                                           AS TENURE_YMD,

    DB2INST1.TIMESTAMPDIFF(256, CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) AS YRS_SERVICE,
    DB2INST1.TIMESTAMPDIFF(64,  CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) AS TOT_MONTHS,
    DB2INST1.TIMESTAMPDIFF(16,  CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) AS TOT_DAYS,
    DB2INST1.TIMESTAMPDIFF(8,   CURRENT TIMESTAMP, TIMESTAMP(HIREDATE)) AS TOT_HOURS

FROM EMPLOYEE
ORDER BY HIREDATE
FETCH FIRST 15 ROWS ONLY;
```
**Output (today = 2026-10-19, FETCH FIRST 15 ROWS):**

```
EMPNO   NAME                  HIREDATE    SVC_YRS  SVC_MOS  SVC_DYS  TENURE_YMD    YRS_SVC  TOT_MOS  TOT_DAYS  TOT_HRS
------  --------------------  ----------  -------  -------  -------  ------------  -------  -------  --------  -------
000340  JASON GOUNOT          05/05/1977       49        5       14  49y 5m 14d         49      593     18064   433548
000050  JOHN GEYER            08/17/1979       47        2        2  47y 2m 2d          47      566     17230   413532
000110  VINCENZO LUCCHESSI    05/16/1988       38        5        3  38y 5m 3d          38      461     14035   336852
000120  SEAN O'CONNELL        12/05/1993       32       10       14  32y 10m 14d        32      394     12006   288156
200310  MICHELLE SPRINGER     09/12/1994       32        1        7  32y 1m 7d          32      385     11725   281412
000310  MAUDE SETRIGHT        09/12/1994       32        1        7  32y 1m 7d          32      385     11725   281412
000010  CHRISTINE HAAS        01/01/1995       31        9       18  31y 9m 18d         31      381     11614   278748
200010  DIAN HEMMINGER        01/01/1995       31        9       18  31y 9m 18d         31      381     11614   278748
000320  RAMLAL MEHTA          07/07/1995       31        3       12  31y 3m 12d         31      375     11427   274260
000230  JAMES JEFFERSON       11/21/1996       29       10       28  29y 10m 28d        29      358     10924   262188
200280  EILEEN SCHWARTZ       03/24/1997       29        6       26  29y 6m 26d         29      354     10801   259236
000280  ETHEL SCHNEIDER       03/24/1997       29        6       26  29y 6m 26d         29      354     10801   259236
200340  ROY ALONZO            07/05/1997       29        3       14  29y 3m 14d         29      351     10698   256764
000210  WILLIAM JONES         04/11/1998       28        6        8  28y 6m 8d          28      342     10418   250044
000220  JENNIFER LUTZ         08/29/1998       28        1       21  28y 1m 21d         28      337     10278   246684

```


---

## Example Group 5 — EMPLOYEE age breakdown

```sql
SELECT
    EMPNO,
    FIRSTNME || ' ' || LASTNAME                          AS NAME,
    BIRTHDATE,
    CURRENT DATE                                         AS TODAY,

    YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)) / 10000
                                                         AS AGE_YEARS,
    MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)) / 100, 100)
                                                         AS AGE_MONTHS,
    MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)), 100)
                                                         AS AGE_DAYS,

    TRIM(CHAR(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)) / 10000))
        || 'y '
    || TRIM(CHAR(MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)) / 100, 100)))
        || 'm '
    || TRIM(CHAR(MOD(YMD_BETWEEN(CURRENT TIMESTAMP, TIMESTAMP(BIRTHDATE)), 100)))
        || 'd'                                           AS AGE_YMD

FROM EMPLOYEE
WHERE BIRTHDATE IS NOT NULL
ORDER BY BIRTHDATE;
```

**Output (today = 2026-10-19):**

```
EMPNO   NAME                    BIRTHDATE   TODAY       AGE_YEARS  AGE_MONTHS  AGE_DAYS  AGE_YMD
------  ----------------------  ----------  ----------  ---------  ----------  --------  -----------
000130  DELORES QUINTANA        09/15/1955  10/19/2026         71           1         4  71y 1m 4d
000050  JOHN GEYER              09/15/1955  10/19/2026         71           1         4  71y 1m 4d
200340  ROY ALONZO              05/17/1956  10/19/2026         70           5         2  70y 5m 2d
000340  JASON GOUNOT            05/17/1956  10/19/2026         70           5         2  70y 5m 2d
000110  VINCENZO LUCCHESSI      11/05/1959  10/19/2026         66          11        14  66y 11m 14d
000310  MAUDE SETRIGHT          04/21/1961  10/19/2026         65           5        28  65y 5m 28d
200310  MICHELLE SPRINGER       04/21/1961  10/19/2026         65           5        28  65y 5m 28d
000320  RAMLAL MEHTA            08/11/1962  10/19/2026         64           2         8  64y 2m 8d
000010  CHRISTINE HAAS          08/24/1963  10/19/2026         63           1        26  63y 1m 26d
200280  EILEEN SCHWARTZ         03/28/1966  10/19/2026         60           6        22  60y 6m 22d
000250  DANIEL SMITH            11/12/1969  10/19/2026         56          11         7  56y 11m 7d
000030  SALLY KWAN              05/11/1971  10/19/2026         55           5         8  55y 5m 8d
000090  EILEEN HENDERSON        05/15/1971  10/19/2026         55           5         4  55y 5m 4d
000200  DAVID BROWN             05/29/1971  10/19/2026         55           4        21  55y 4m 21d
200330  HELENA WONG             07/18/1971  10/19/2026         55           3         1  55y 3m 1d
000330  WING LEE                07/18/1971  10/19/2026         55           3         1  55y 3m 1d
200120  GREG ORLANDO            10/18/1972  10/19/2026         54           0         1  54y 0m 1d
000120  SEAN O'CONNELL          10/18/1972  10/19/2026         54           0         1  54y 0m 1d
200010  DIAN HEMMINGER          08/14/1973  10/19/2026         53           2         5  53y 2m 5d
000060  IRVING STERN            07/07/1975  10/19/2026         51           3        12  51y 3m 12d
000140  HEATHER NICHOLLS        01/19/1976  10/19/2026         50           9         0  50y 9m 0d
200140  KIM NATZ                01/19/1976  10/19/2026         50           9         0  50y 9m 0d
000280  ETHEL SCHNEIDER         03/28/1976  10/19/2026         50           6        22  50y 6m 22d
000260  SYBIL JOHNSON           10/05/1976  10/19/2026         50           0        14  50y 0m 14d
000300  PHILIP SMITH            10/27/1976  10/19/2026         49          11        23  49y 11m 23d
000150  BRUCE ADAMSON           05/17/1977  10/19/2026         49           5         2  49y 5m 2d
000020  MICHAEL THOMPSON        02/02/1978  10/19/2026         48           8        17  48y 8m 17d
200220  REBA JOHN               03/19/1978  10/19/2026         48           7         0  48y 7m 0d
000220  JENNIFER LUTZ           03/19/1978  10/19/2026         48           7         0  48y 7m 0d
000180  MARILYN SCOUTTEN        02/21/1979  10/19/2026         47           7        26  47y 7m 26d
000160  ELIZABETH PIANKA        04/12/1980  10/19/2026         46           6         7  46y 6m 7d
000230  JAMES JEFFERSON         05/30/1980  10/19/2026         46           4        20  46y 4m 20d
000100  THEODORE SPENSER        12/18/1980  10/19/2026         45          10         1  45y 10m 1d
000170  MASATOSHI YOSHIMURA     01/05/1981  10/19/2026         45           9        14  45y 9m 14d
200170  KIYOSHI YAMAMOTO        01/05/1981  10/19/2026         45           9        14  45y 9m 14d
000190  JAMES WALKER            06/25/1982  10/19/2026         44           3        24  44y 3m 24d
200240  ROBERT MONTEVERDE       03/31/1984  10/19/2026         42           6        19  42y 6m 19d
000290  JOHN PARKER             07/09/1985  10/19/2026         41           3        10  41y 3m 10d
000240  SALVATORE MARINO        03/31/2002  10/19/2026         24           6        19  24y 6m 19d
000210  WILLIAM JONES           02/23/2003  10/19/2026         23           7        24  23y 7m 24d
000070  EVA PULASKI             05/26/2003  10/19/2026         23           4        24  23y 4m 24d
000270  MARIA PEREZ             05/26/2003  10/19/2026         23           4        24  23y 4m 24d

  42 record(s) selected.
```

---

## Interval Reference Card

| Code | Interval | Method | Resolution |
|---:|---|---|---|
| `1` | Microseconds | `EXTRACT(EPOCH) × 1,000,000` | Whole-second only ¹ |
| `2` | Seconds | `EXTRACT(EPOCH)` | Exact |
| `4` | Minutes | `EXTRACT(EPOCH) / 60` | Exact |
| `8` | Hours | `EXTRACT(EPOCH) / 3,600` | Exact |
| `16` | Days | `EXTRACT(EPOCH) / 86,400` | Exact |
| `32` | Weeks | `EXTRACT(EPOCH) / 604,800` | Exact |
| `64` | Months | `YMD_BETWEEN → total months` | IBM-defined |
| `128` | Quarters | `total months / 3` | IBM-defined |
| `256` | Years | `total months / 12` | IBM-defined |

¹ To include fractional microseconds: `V_EPOCH_DIFF * 1000000 + EXTRACT(MICROSECONDS FROM P_TS1) − EXTRACT(MICROSECONDS FROM P_TS2)` with second-boundary handling.
