#  Logical Standby Database: SKIP Rules & Log Retention

##  How to Skip DML, DDL, or Specific Transactions

You can control what DML or DDL operations are **ignored** on a Logical Standby database using the `DBMS_LOGSTDBY.SKIP` procedure.

Typical reasons to skip operations:
- Target a **specific table** or **schema**.
- Ignore DML/DDL based on **error conditions**.
- Customize DDL behavior with a **transformation procedure**.
- Completely exclude **problematic transactions**.

---

##  Basic SKIP Rule Example (DML on Table)

1. **Stop SQL Apply:**

```sql
ALTER DATABASE STOP LOGICAL STANDBY APPLY;
```

2. **Create a SKIP rule:**

```sql
EXECUTE DBMS_LOGSTDBY.SKIP(
    STMT => 'DML',
    SCHEMA_NAME => 'HR',
    OBJECT_NAME => 'DEPARTMENTS'
);
```

(Alternative full PL/SQL block with exception handling:)

```sql
SET SERVEROUTPUT ON
BEGIN
    DBMS_LOGSTDBY.SKIP(
         stmt => 'DML',
         schema_name => 'HR',
         object_name => 'DEPARTMENTS',
         proc_name => NULL
    );
EXCEPTION
    WHEN OTHERS THEN 
        DBMS_OUTPUT.PUT_LINE('Failure during setup of DML restrictions on HR.DEPARTMENTS');
END;
/
```

3. **Start SQL Apply:**

```sql
ALTER DATABASE START LOGICAL STANDBY APPLY IMMEDIATE;
```

After this, **DML** against `HR.DEPARTMENTS` on the **Primary** will **not** be applied to the Logical Standby.

---

##  Other Useful SKIP Examples

| Purpose | Command |
|:--------|:--------|
| Skip DML on `SCOTT.EMP` table | `EXECUTE DBMS_LOGSTDBY.SKIP('DML', 'SCOTT', 'EMP');` |
| Skip DDL on `SCOTT.EMP` table | `EXECUTE DBMS_LOGSTDBY.SKIP('SCHEMA_DDL', 'SCOTT', 'EMP');` |
| Skip **all DML** for HR schema | `EXECUTE DBMS_LOGSTDBY.SKIP('DML', 'HR', '%');` |
| Skip **CREATE/DROP DIRECTORY** operations | `EXECUTE DBMS_LOGSTDBY.SKIP('DIRECTORY');` |

---

## 🔧 Advanced: Modify DDL Before Apply (Path Replacement Example)

1. **Create a Transformation Procedure:**

```sql
CREATE OR REPLACE PROCEDURE sys.change_ts_ddl (
    old_stmt  IN VARCHAR2,
    stmt_typ  IN VARCHAR2,
    schema    IN VARCHAR2,
    name      IN VARCHAR2,
    xidusn    IN NUMBER,
    xidslt    IN NUMBER,
    xidsqn    IN NUMBER,
    action    OUT NUMBER,
    new_stmt  OUT VARCHAR2
) AS
BEGIN
    new_stmt := REPLACE(old_stmt,
                        '/u01/app/oracle2/datafile/ORCL',
                        '/datafile/ORCL');
    action := dbms_logstdby.skip_action_replace;
EXCEPTION
    WHEN OTHERS THEN
        action := dbms_logstdby.skip_action_error;
        new_stmt := NULL;
END change_ts_ddl;
/
```

2. **Register the Rule:**

```sql
EXECUTE DBMS_LOGSTDBY.SKIP(STMT => 'TABLESPACE', PROC_NAME => 'SYS.CHANGE_TS_DDL');
```

3. **Effect:**  
When DDL is applied (e.g., adding a datafile), the Logical Standby **rewrites** the file path automatically.

---

## 📜 Checking Existing SKIP Rules

```sql
SELECT OWNER, NAME, STATEMENT_OPT, PROC
FROM DBA_LOGSTDBY_SKIP
WHERE STATEMENT_OPT <> 'INTERNAL SCHEMA';
```

---

#  Remote Archived Log File Retention

Oracle manages old archived logs on Logical Standbys using:

| Parameter | Description |
|:----------|:------------|
| `LOG_AUTO_DELETE` (default `TRUE`) | Automatically deletes archived logs after apply is done |
| `LOG_AUTO_DEL_RETENTION_TARGET` (default `1440` minutes) | How long (in minutes) to retain applied logs before deletion |

Manage settings using:

- `DBMS_LOGSTDBY.APPLY_SET`
- `DBMS_LOGSTDBY.APPLY_UNSET`

---

#  Quick Flow Summary

```plaintext
1. STOP SQL Apply
2. Configure SKIP rules (DBMS_LOGSTDBY.SKIP)
3. START SQL Apply IMMEDIATE
4. Validate with DBA_LOGSTDBY_SKIP
```

---

Would you also like me to format this into an **ASCII-style table** or **cheat sheet** for quick reference (like you asked in other work)? 🚀  
It would make a nice, compact version too! 🎯  
Would you like that? 
