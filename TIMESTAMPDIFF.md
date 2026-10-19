# A Calendar-Aware TIMESTAMPDIFF for IBM Db2 LUW

> Platform: IBM Db2 LUW 12.1  
> Schema: `DB2INST1`  
> Function: `DB2INST1.TIMESTAMPDIFF(INTEGER, TIMESTAMP, TIMESTAMP)`  
> References: [TIMESTAMPDIFF](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-timestampdiff) · [EXTRACT](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-extract) · [YMD_BETWEEN](https://www.ibm.com/docs/en/db2/12.1.x?topic=functions-ymd-between)  
> Author: Mukesh Kumar · [mkt@ibm.com](mailto:mkt@ibm.com)

---

## TL;DR

The built-in `SYSIBM.TIMESTAMPDIFF` is not calendar-accurate. It converts the timestamp difference into a duration string and then applies fixed assumptions such as 30 days per month and 365 days per year. That makes results like this possible:

```sql
-- Native function: approximate
VALUES TIMESTAMPDIFF(16,
    CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00')));
-- Returns: 30

-- Calendar-aware wrapper: correct
VALUES DB2INST1.TIMESTAMPDIFF(16,
    TIMESTAMP('1997-03-01 00:00:00'),
    TIMESTAMP('1997-02-01 00:00:00'));
-- Returns: 28
```

The fix is simple:

- Use `EXTRACT(EPOCH)` for fixed-length intervals: seconds, minutes, hours, days, weeks
- Use `YMD_BETWEEN` for calendar intervals: months, quarters, years

---

## Why the native function is wrong

`SYSIBM.TIMESTAMPDIFF` is documented to return an estimate, not an exact calendar-based result. The problem is that it effectively treats a month as 30 days and a year as 365 days:

| Assumption | Value |
|---|---:|
| Days in a month | 30 |
| Days in a year | 365 |
| Hours in a day | 24 |
| Minutes in an hour | 60 |
| Seconds in a minute | 60 |

For the interval between `1997-02-01` and `1997-03-01`, the native logic effectively does:

```text
days = 0 + (1 × 30) + (0 × 365) = 30
```

But February 1997 has only 28 days, so the correct answer is 28.

This is not a quirk. It is a consequence of the fixed-assumption arithmetic used by the native function.

---

## Function signature and resolution

`SYSIBM.TIMESTAMPDIFF` takes a duration string, usually created as `CHAR(ts1 - ts2)`, and its signature is effectively `(INTEGER, VARCHAR)`.

The custom function here accepts two raw `TIMESTAMP` values instead:

```sql
DB2INST1.TIMESTAMPDIFF(INTEGER, TIMESTAMP, TIMESTAMP)
```

The important part is that Db2 resolves calls by both schema and argument signature. An unqualified call with three arguments resolves to `DB2INST1.TIMESTAMPDIFF` only when `DB2INST1` is on `CURRENT PATH`.

| Function | Schema | Signature | Conflict? |
|---|---|---|---|
| `TIMESTAMPDIFF` | `SYSIBM` | `(INTEGER, VARCHAR)` | No — 2 args |
| `TIMESTAMPDIFF` | `DB2INST1` | `(INTEGER, TIMESTAMP, TIMESTAMP)` | No — 3 args |

---

## The correct math

### 1) Fixed-length intervals: `EXTRACT(EPOCH)`

For intervals such as seconds, minutes, hours, days, and weeks, the difference should be based on absolute elapsed seconds, not assumed month lengths.

```sql
EXTRACT(EPOCH FROM TIMESTAMP('1997-03-01 00:00:00'))
-- 857,174,400

EXTRACT(EPOCH FROM TIMESTAMP('1997-02-01 00:00:00'))
-- 854,755,200

Difference = 2,419,200 seconds
```

That is exactly:

```text
Days  = 2,419,200 / 86,400 = 28
Hours = 2,419,200 / 3,600 = 672
```

This is exact for fixed-length intervals.

### 2) Calendar intervals: `YMD_BETWEEN`

For month-based intervals, the correct logic is to use IBM's native calendar arithmetic via `YMD_BETWEEN`.

`YMD_BETWEEN` returns a packed integer in the form `±YYYYMMDD`, representing full years, months, and days between two timestamps.

```sql
raw / 10000                        -- years component
MOD(raw / 100, 100)                -- months component
MOD(raw, 100)                      -- days component

-- total months used for months/quarters/years
BIGINT(raw / 10000) * 12 + BIGINT(MOD(raw / 100, 100))
```

This is the correct basis for:

- months (`64`)
- quarters (`128`)
- years (`256`)

---

## Function DDL

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

    -- Fixed-length intervals: use absolute elapsed time in seconds.
    -- EXTRACT(EPOCH) returns whole seconds; sub-second precision is truncated.
    SET V_EPOCH_DIFF = EXTRACT(EPOCH FROM P_TS1) - EXTRACT(EPOCH FROM P_TS2);

    -- Calendar intervals: use IBM's native calendar arithmetic.
    SET V_YMD = YMD_BETWEEN(P_TS1, P_TS2);

    -- Total months = years * 12 + months
    SET V_TOTAL_MONTHS =
          BIGINT(V_YMD / 10000) * 12
        + BIGINT(MOD(V_YMD / 100, 100));

    SET V_RESULT =
        CASE P_INTERVAL
            WHEN 1   THEN V_EPOCH_DIFF * 1000000   -- Microseconds (whole-second resolution)
            WHEN 2   THEN V_EPOCH_DIFF              -- Seconds
            WHEN 4   THEN V_EPOCH_DIFF / 60         -- Minutes
            WHEN 8   THEN V_EPOCH_DIFF / 3600       -- Hours
            WHEN 16  THEN V_EPOCH_DIFF / 86400     -- Days
            WHEN 32  THEN V_EPOCH_DIFF / 604800    -- Weeks

            WHEN 64  THEN V_TOTAL_MONTHS            -- Months
            WHEN 128 THEN V_TOTAL_MONTHS / 3        -- Quarters
            WHEN 256 THEN V_TOTAL_MONTHS / 12       -- Years

            ELSE NULL
        END;

    RETURN V_RESULT;
END@
```

---

## Interval reference

| Code | Interval | Logic | Notes |
|---:|---|---|---|
| `1` | Microseconds | `EXTRACT(EPOCH) * 1,000,000` | Whole-second resolution only |
| `2` | Seconds | `EXTRACT(EPOCH)` | Exact |
| `4` | Minutes | `EXTRACT(EPOCH) / 60` | Exact |
| `8` | Hours | `EXTRACT(EPOCH) / 3600` | Exact |
| `16` | Days | `EXTRACT(EPOCH) / 86400` | Exact |
| `32` | Weeks | `EXTRACT(EPOCH) / 604800` | Exact |
| `64` | Months | `YMD_BETWEEN -> total months` | IBM calendar semantics |
| `128` | Quarters | `total months / 3` | IBM calendar semantics |
| `256` | Years | `total months / 12` | IBM calendar semantics |

---

## Why month and year cannot be fixed-second values

A month is not always 30 days, and a year is not always 365 days.

| Period | Seconds |
|---|---:|
| February (non-leap year) | 28 × 86,400 = 2,419,200 |
| February (leap year) | 29 × 86,400 = 2,505,600 |
| 30-day month | 30 × 86,400 = 2,592,000 |
| 31-day month | 31 × 86,400 = 2,678,400 |
| Non-leap year | 365 × 86,400 = 31,536,000 |
| Leap year | 366 × 86,400 = 31,622,400 |

Using a fixed divisor such as `epoch_diff / 2592000` for months or `epoch_diff / 31104000` for years reintroduces the same bug. Calendar intervals must be computed using the calendar, not a constant seconds-per-month assumption.

---

## Examples

### Example 1: February 1997 is 28 days, not 30

```sql
SELECT
    TIMESTAMPDIFF(16, CHAR(TIMESTAMP('1997-03-01 00:00:00') - TIMESTAMP('1997-02-01 00:00:00'))) AS NATIVE_APPROX,
    DB2INST1.TIMESTAMPDIFF(16, TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00')) AS ACCURATE
FROM SYSIBM.SYSDUMMY1;
```

Result:

```text
NATIVE_APPROX  ACCURATE
-------------  --------
30             28
```

### Example 2: month-end and leap-day edge cases

```sql
SELECT
    DB2INST1.TIMESTAMPDIFF(64, TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-29 00:00:00')) AS leap_day,
    DB2INST1.TIMESTAMPDIFF(64, TIMESTAMP('2000-03-01 00:00:00'), TIMESTAMP('2000-02-01 00:00:00')) AS one_month,
    DB2INST1.TIMESTAMPDIFF(64, TIMESTAMP('2025-05-31 00:00:00'), TIMESTAMP('2025-04-30 00:00:00')) AS apr30_to_may31,
    DB2INST1.TIMESTAMPDIFF(64, TIMESTAMP('2026-03-15 09:00:00'), TIMESTAMP('2026-02-15 10:00:00')) AS before_10am
FROM SYSIBM.SYSDUMMY1;
```

Result:

```text
LEAP_DAY  ONE_MONTH  APR30_TO_MAY31  BEFORE_10AM
--------  ---------  --------------  -----------
0         1          1               0
```

### Example 3: fixed-length intervals remain exact

```sql
SELECT
    DB2INST1.TIMESTAMPDIFF(2,  TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00')) AS seconds,
    DB2INST1.TIMESTAMPDIFF(4,  TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00')) AS minutes,
    DB2INST1.TIMESTAMPDIFF(8,  TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00')) AS hours,
    DB2INST1.TIMESTAMPDIFF(16, TIMESTAMP('1997-03-01 00:00:00'), TIMESTAMP('1997-02-01 00:00:00')) AS days
FROM SYSIBM.SYSDUMMY1;
```

Result:

```text
SECONDS  MINUTES  HOURS  DAYS
-------- -------- -----  -----
2419200  40320    672    28
```

---

## Deployment and usage

### Standard deployment

```sql
db2 -td@ -vf new_timestampdiff.sql -z new_timestampdiff.sql.out
```

### Schema path

```sql
VALUES CURRENT PATH;
SET PATH = SYSTEM PATH, DB2INST1;
```

Always call the function using the fully qualified name:

```sql
SELECT DB2INST1.TIMESTAMPDIFF(16, END_TS, START_TS)
FROM MY_TABLE;
```

---

## Known limitations

| Limitation | Detail |
|---|---|
| Sub-second precision | `EXTRACT(EPOCH)` returns whole seconds, so microseconds lose fractional precision |
| Unsupported interval codes | Any code other than `1, 2, 4, 8, 16, 32, 64, 128, 256` returns `NULL` |
| Application-local DST handling | Depends on the timestamp type, database settings, and how local time is stored |

If you need sub-second precision, you must handle the fractional part explicitly in the caller.

---

## Final takeaway

The key design rule is:

- Fixed-length intervals use elapsed seconds
- Calendar intervals use IBM calendar logic

That gives you a function that behaves like a true calendar-aware `TIMESTAMPDIFF` instead of a fixed-assumption approximation.

