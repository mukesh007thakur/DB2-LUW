# DB2 LUW: Why `CREATE OR REPLACE VIEW` Fails While `DROP` + `CREATE VIEW` Works

**Product:** IBM DB2 LUW 12.1  
**Environment:** Single/Database Partitioning Feature (DPF) — multiple MLNs  
**Error:** `SQL0551N` — Scenario 5  

---

## The Problem

A user with `CREATEIN`, `DROPIN`, and `ALTERIN` privileges on a schema runs:

```sql
CREATE OR REPLACE VIEW myschema.myview AS
SELECT col1, col2 FROM myschema.mytable WHERE col3 = 'X';
```

DB2 returns:

```
SQL0551N  The statement failed because the authorization ID does not have
the required authorization or privilege to perform the operation.
Authorization ID: "USERID". Operation: "REPLACE". Object: "MYSCHEMA.MYVIEW".
SQLSTATE=42501
```

But when the same user runs:

```sql
DROP VIEW myschema.myview;
CREATE VIEW myschema.myview AS
SELECT col1, col2 FROM myschema.mytable WHERE col3 = 'X';
```

It works perfectly.

---

## Root Cause

### DB2 `CREATE OR REPLACE VIEW` is a two-phase operation

When the view **already exists**, `CREATE OR REPLACE VIEW` internally performs a **REPLACE** operation on the existing object — not just a CREATE. DB2 enforces a strict rule for this:

> **The user executing `CREATE OR REPLACE VIEW` must be the OWNER of the existing view object.**

Schema-level privileges (`CREATEIN`, `DROPIN`, `ALTERIN`) **do not grant** the right to replace an object you do not own.

### Official IBM Explanation — SQL0551N Scenario 5

> *"Replacing an existing object by using a CREATE OR REPLACE statement failed because the user is not the owner of the object."*

— IBM DB2 12.1 SQL0551N Message Reference

---

## Privilege Comparison Table

| Operation | Privilege Required | Schema Privilege Sufficient? |
|---|---|---|
| `CREATE VIEW` (new object) | `CREATEIN` on schema | ✅ Yes |
| `DROP VIEW` | `DROPIN` on schema or object ownership | ✅ Yes |
| `ALTER VIEW` | `ALTERIN` on schema or object ownership | ✅ Yes |
| `CREATE OR REPLACE VIEW` (existing object) | Must be **OWNER** of the object OR `SYSADM`/`DBADM` | ❌ No — ownership required |

---

## Why DROP + CREATE Works

The two-step approach works because each operation uses **different privilege checks**:

1. `DROP VIEW` → checks `DROPIN` on schema → ✅ user has it → succeeds
2. `CREATE VIEW` → checks `CREATEIN` on schema → ✅ user has it → succeeds
3. The **new view** is now owned by the user who ran `CREATE VIEW`

The key difference: `CREATE OR REPLACE VIEW` checks **object ownership** on the existing object, while `DROP` + `CREATE` checks **schema privileges** only.

---

## Diagnostic Steps

### Step 1 — Find the current owner of the view

```sql
SELECT TABNAME, TABSCHEMA, OWNER, OWNERTYPE
FROM SYSCAT.TABLES
WHERE TABNAME   = 'YOUR_VIEW_NAME'
  AND TABSCHEMA = 'YOUR_SCHEMA';
```

**OWNERTYPE values:**
- `U` = User
- `R` = Role

### Step 2 — Check schema-level privileges of the user

```sql
SELECT GRANTOR, GRANTEE, SCHEMANAME,
       CREATEINAUTH, DROPINAUTH, ALTERINAUTH, UPDATEINAUTH
FROM SYSCAT.SCHEMAAUTH
WHERE GRANTEE = 'YOUR_USERID';
```

### Step 3 — Check for role-based grants

```sql
SELECT R.ROLENAME
FROM SYSCAT.ROLEAUTH R
WHERE R.GRANTEE = 'YOUR_USERID';
```

### Step 4 — Check view-level privileges

```sql
SELECT GRANTOR, GRANTEE, TABNAME, TABSCHEMA, CONTROLAUTH, ALTERAUTH, DELETEAUTH, INDEXAUTH, INSERTAUTH, REFAUTH, SELECTAUTH, UPDATEAUTH 
FROM SYSCAT.TABAUTH
WHERE TABNAME  = 'YOUR_VIEW_NAME'
  AND TABSCHEMA = 'YOUR_SCHEMA';
```

---

## Resolution Options

IBM DB2 SQL0551N (Scenario 5) provides three official resolutions:

### Option 1 — Run as the Original View Owner *(Restricted environments)*

Identify the owner using the diagnostic query above, and have that user (or a process running under that ID) execute the `CREATE OR REPLACE VIEW` statement.

```sql
-- Confirm the owner first
SELECT OWNER FROM SYSCAT.TABLES
WHERE TABNAME = 'MYVIEW' AND TABSCHEMA = 'MYSCHEMA';

-- Then connect as that owner and run:
CREATE OR REPLACE VIEW myschema.myview AS
SELECT col1, col2 FROM myschema.mytable WHERE col3 = 'X';
```

---

### Option 2 — Transfer Ownership *(Requires SYSADM or DBADM)*

Transfer the view ownership to the user who needs to replace it:

```sql
TRANSFER OWNERSHIP OF VIEW myschema.myview
TO USER target_userid PRESERVE PRIVILEGES;
```

After transfer, the user can freely use `CREATE OR REPLACE VIEW` on that object.

> ⚠️ **Note:** `TRANSFER OWNERSHIP` requires `SYSADM` or `DBADM` authority. In restricted environments, this must be done by a DBA.

---

### Option 3 — DROP then CREATE *(Already working — pragmatic solution)*

```sql
DROP VIEW myschema.myview;

CREATE VIEW myschema.myview AS
SELECT col1, col2 FROM myschema.mytable WHERE col3 = 'X';
```

**Considerations for this approach:**
- The new view will be **owned by the user** who ran `CREATE VIEW`
- Any **dependent objects** (packages, other views) that referenced the old view will be **invalidated** and must be rebound/recreated
- In a DPF environment with multiple MLNs, this is a catalog-level operation propagated across all partitions — it is safe but ensure no active sessions are using the view

---

## Important Note for DPF / multiple MLN Environments

In a partitioned database (DPF) with multiple MLNs, all DDL operations including `DROP VIEW` and `CREATE VIEW` are **propagated across all partitions via the catalog partition**. This is expected behavior. The ownership rule applies identically regardless of the number of partitions — the error is **not** caused by partition count or DPF architecture.

---

## Summary

| Question | Answer |
|---|---|
| Why does `CREATE OR REPLACE VIEW` fail? | The user is not the **owner** of the existing view object |
| Does having `CREATEIN`, `DROPIN`, `ALTERIN` help? | No — schema privileges do not override ownership check for REPLACE |
| Why does `DROP` + `CREATE` work? | These operations use schema privileges, not object ownership |
| Is this a DPF/MLN-specific issue? | No — it applies to all DB2 LUW environments |
| What is the error code? | `SQL0551N` SQLSTATE `42501` — Scenario 5 |
| Quickest fix in restricted environment? | Continue using `DROP VIEW` then `CREATE VIEW` |

---

## IBM Reference

- **SQL0551N Message Reference:** https://www.ibm.com/docs/en/db2/12.1?topic=messages-sql0551n
- **TRANSFER OWNERSHIP Statement:** https://www.ibm.com/docs/en/db2/12.1?topic=statements-transfer-ownership
- **CREATE VIEW Statement:** https://www.ibm.com/docs/en/db2/12.1?topic=statements-create-view
- **SYSCAT.TABLES Catalog View:** https://www.ibm.com/docs/en/db2/12.1?topic=views-syscattables
- **SYSCAT.SCHEMAAUTH Catalog View:** https://www.ibm.com/docs/en/db2/12.1?topic=views-syscatschemaauth
- **DB2 LUW 12.1 Documentation Home:** https://www.ibm.com/docs/en/db2/12.1

---

*This document is based on IBM DB2 LUW 12.1 official SQL0551N message text and behavior observed in a DPF environment with multiple MLNs.*
