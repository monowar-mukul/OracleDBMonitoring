# Oracle Database Audit - Complete Guide

A comprehensive collection of SQL scripts for implementing and monitoring database auditing in Oracle databases.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Audit Configuration](#audit-configuration)
- [Monitoring Scripts](#monitoring-scripts)
- [Security Checks](#security-checks)
- [Best Practices](#best-practices)

## Overview

Database auditing is a critical security feature that tracks and records database activities. This repository provides ready-to-use SQL scripts for:

- Configuring audit settings
- Monitoring login/logout attempts
- Detecting suspicious activities
- Tracking schema changes
- Reviewing audit trails

## Prerequisites

- Oracle Database (tested on 10g and higher)
- DBA privileges or AUDIT SYSTEM privilege
- Access to audit-related data dictionary views

## Audit Configuration

### Check Current Audit Settings

```sql
-- Verify if auditing is enabled and check output destination
SELECT name, value 
FROM v$parameter
WHERE name LIKE 'audit%';
```

### View Active Audit Options

```sql
-- Check statement-level audit options
SELECT * FROM dba_stmt_audit_opts
UNION
SELECT * FROM dba_priv_audit_opts;
```

### Enable Basic Auditing

```sql
-- Audit session creation (login/logout)
AUDIT CREATE SESSION;

-- Audit all privileges by session
AUDIT ALL PRIVILEGES BY SESSION;

-- Audit key roles
AUDIT CONNECT BY SESSION;
AUDIT RESOURCE BY SESSION;
AUDIT DBA BY SESSION;

-- Audit dictionary access
AUDIT SELECT ANY DICTIONARY BY SESSION;
AUDIT ANALYZE ANY DICTIONARY BY SESSION;
```

### Check Audit Privileges

```sql
-- View which users/roles have audit privileges
SELECT *
FROM dba_sys_privs
WHERE privilege LIKE '%AUDIT%';
```

### Enable Schema Change Auditing

Generate audit commands for DDL operations:

```sql
SET HEAD OFF
SET FEED OFF
SET PAGES 0
SPOOL aud.lis

-- Audit CREATE operations
SELECT 'audit ' || name || ';'
FROM system_privilege_map
WHERE (name LIKE 'CREATE%TABLE%'
    OR name LIKE 'CREATE%INDEX%'
    OR name LIKE 'CREATE%CLUSTER%'
    OR name LIKE 'CREATE%SEQUENCE%'
    OR name LIKE 'CREATE%PROCEDURE%'
    OR name LIKE 'CREATE%TRIGGER%'
    OR name LIKE 'CREATE%LIBRARY%')
UNION
-- Audit ALTER operations
SELECT 'audit ' || name || ';'
FROM system_privilege_map
WHERE (name LIKE 'ALTER%TABLE%'
    OR name LIKE 'ALTER%INDEX%'
    OR name LIKE 'ALTER%CLUSTER%'
    OR name LIKE 'ALTER%SEQUENCE%'
    OR name LIKE 'ALTER%PROCEDURE%'
    OR name LIKE 'ALTER%TRIGGER%'
    OR name LIKE 'ALTER%LIBRARY%')
UNION
-- Audit DROP operations
SELECT 'audit ' || name || ';'
FROM system_privilege_map
WHERE (name LIKE 'DROP%TABLE%'
    OR name LIKE 'DROP%INDEX%'
    OR name LIKE 'DROP%CLUSTER%'
    OR name LIKE 'DROP%SEQUENCE%'
    OR name LIKE 'DROP%PROCEDURE%'
    OR name LIKE 'DROP%TRIGGER%'
    OR name LIKE 'DROP%LIBRARY%')
UNION
-- Audit EXECUTE operations
SELECT 'audit ' || name || ';'
FROM system_privilege_map
WHERE (name LIKE 'EXECUTE%INDEX%'
    OR name LIKE 'EXECUTE%PROCEDURE%'
    OR name LIKE 'EXECUTE%LIBRARY%');

SPOOL OFF
```
### Verify Enabled Audit Actions

```sql
SELECT audit_option, success, failure
FROM dba_stmt_audit_opts
UNION
SELECT privilege, success, failure
FROM dba_priv_audit_opts;
```

## Monitoring Scripts

### Session Audit Report

```sql
SET SERVEROUTPUT ON SIZE 1000000
COL username FOR A15
COL terminal FOR A6
COL timestamp FOR A15
COL logoff_time FOR A15
COL action_name FOR A8
COL returncode FOR 9999

SELECT username,
       terminal,
       action_name,
       TO_CHAR(timestamp, 'DDMMYYYY:HHMISS') timestamp,
       TO_CHAR(logoff_time, 'DDMMYYYY:HHMISS') logoff_time,
       returncode
FROM dba_audit_session
ORDER BY timestamp DESC;
```

### Failed Login Attempts (Daily Summary)

```sql
-- Group failed logins by user, terminal, and day
SELECT COUNT(*), 
       username, 
       terminal, 
       TO_CHAR(timestamp, 'DD-MON-YYYY')
FROM dba_audit_session
WHERE returncode <> 0
GROUP BY username, terminal, TO_CHAR(timestamp, 'DD-MON-YYYY')
ORDER BY 1 DESC;
```

### Failed Login Attempts (Detailed)

```sql
-- Include return codes for detailed analysis
SELECT COUNT(*), 
       username, 
       terminal, 
       TO_CHAR(timestamp, 'DD-MON-YYYY'), 
       returncode
FROM dba_audit_session
GROUP BY username, terminal, TO_CHAR(timestamp, 'DD-MON-YYYY'), returncode
HAVING COUNT(*) > 1
ORDER BY 1 DESC;
```

## Security Checks

### Detect Invalid User Access Attempts

Identifies login attempts with non-existent usernames (potential brute force attacks):

```sql
SELECT username, 
       terminal, 
       TO_CHAR(timestamp, 'DD-MON-YYYY HH24:MI:SS')
FROM dba_audit_session
WHERE returncode <> 0
  AND NOT EXISTS (
      SELECT 'x'
      FROM dba_users
      WHERE dba_users.username = dba_audit_session.username
  )
ORDER BY timestamp DESC;
```

### Unusual Access Hours

Detect database access outside normal business hours:

```sql
SELECT username,
       terminal,
       action_name,
       returncode,
       TO_CHAR(timestamp, 'DD-MON-YYYY HH24:MI:SS'),
       TO_CHAR(logoff_time, 'DD-MON-YYYY HH24:MI:SS')
FROM dba_audit_session
WHERE TO_DATE(TO_CHAR(timestamp, 'HH24:MI:SS'), 'HH24:MI:SS') < TO_DATE('08:00:00', 'HH24:MI:SS')
   OR TO_DATE(TO_CHAR(timestamp, 'HH24:MI:SS'), 'HH24:MI:SS') > TO_DATE('19:30:00', 'HH24:MI:SS')
ORDER BY timestamp DESC;
```

**Note:** Adjust the time thresholds (08:00:00 and 19:30:00) based on your organization's business hours.

### Shared Account Detection

Identify accounts accessed from multiple terminals (possible account sharing):

```sql
SELECT COUNT(DISTINCT(terminal)), username
FROM dba_audit_session
GROUP BY username
HAVING COUNT(DISTINCT(terminal)) > 1
ORDER BY 1 DESC;
```

### Multiple Account Usage from Single Terminal

Detect terminals using multiple accounts (potential security risk):

```sql
SELECT COUNT(DISTINCT(username)), terminal
FROM dba_audit_session
GROUP BY terminal
HAVING COUNT(DISTINCT(username)) > 1
ORDER BY 1 DESC;
```

### Schema Change Audit Trail

Monitor all structural changes to the database:

```sql
COL username FOR A8
COL priv_used FOR A16
COL obj_name FOR A22
COL timestamp FOR A17
COL returncode FOR 9999

SELECT username,
       priv_used,
       obj_name,
       TO_CHAR(timestamp, 'DD-MON-YYYY HH24:MI') timestamp,
       returncode
FROM dba_audit_trail
WHERE priv_used IS NOT NULL
  AND priv_used <> 'CREATE SESSION'
ORDER BY timestamp DESC;
```

## Comprehensive Audit Status Check

A complete audit configuration report:

```sql
SET LINES 132
SET PAGES 60
COL name FORMAT A30
COL value FORMAT A40

SPOOL check_audit.lis

PROMPT Database Parameter Settings
PROMPT ===================================
SELECT name, value
FROM v$parameter
WHERE name LIKE 'audit%';

PROMPT 
PROMPT Active Statement Level Audit Options
PROMPT ===================================
SELECT audit_option, 
       success, 
       failure, 
       DECODE(user_name, NULL, '', ' BY ' || user_name) "USER"
FROM dba_stmt_audit_opts;

PROMPT 
PROMPT Active Privilege Level Audit Options
PROMPT ====================================
SELECT privilege, 
       success, 
       failure, 
       DECODE(user_name, NULL, '', ' BY ' || user_name) "USER"
FROM dba_priv_audit_opts;

PROMPT 
PROMPT Active Object Level Audit Options
PROMPT ====================================
SELECT * FROM dba_obj_audit_opts;

PROMPT 
PROMPT Default Object Audit Options
PROMPT ====================================
SELECT * FROM all_def_audit_opts;

SPOOL OFF
```

### Understanding Audit Option Characters

- **`-`** : Audit option is not set
- **`S`** : Audit option set BY SESSION
- **`A`** : Audit option set BY ACCESS
- Format: `SUCCESS/FAILURE` (e.g., `S/S` means audit both successful and failed attempts by session)

## Best Practices

1. **Regular Review**: Schedule regular reviews of audit trails to detect anomalies
2. **Archive Old Audit Data**: Implement a retention policy to manage audit table growth
3. **Secure Audit Tables**: Restrict access to `SYS.AUD$` and related audit tables
4. **Monitor Disk Space**: Audit trails can grow rapidly; ensure adequate storage
5. **Selective Auditing**: Audit only what's necessary to minimize performance impact
6. **Alert on Critical Events**: Set up automated alerts for suspicious activities
7. **Compliance**: Ensure audit configuration meets regulatory requirements (SOX, HIPAA, PCI-DSS, etc.)

## Common Return Codes

| Code | Description |
|------|-------------|
| 0    | Successful operation |
| 1017 | Invalid username/password |
| 1045 | User lacks CREATE SESSION privilege |
| 28000 | Account is locked |
| 28001 | Password has expired |
| 28002 | Password will expire soon |

## Performance Considerations

- Use `BY SESSION` instead of `BY ACCESS` when possible to reduce audit records
- Regularly purge old audit records from `SYS.AUD$`
- Consider using Fine-Grained Auditing (FGA) for specific data access tracking
- Monitor the size of the SYSTEM tablespace where audit records are stored

## Resources

- [Oracle Database Security Guide](https://docs.oracle.com/en/database/)
- [Oracle Auditing Documentation](https://docs.oracle.com/en/database/)

## License

This project is provided as-is for educational and professional use.

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues for improvements.

---

**⚠️ Important:** Always test these scripts in a development environment before applying to production systems.
