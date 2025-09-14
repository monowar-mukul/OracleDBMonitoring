# Oracle User Management and Security Scripts

A comprehensive collection of Oracle database user management, security monitoring, and administrative scripts for DBAs.

## Table of Contents

- [User Creation and Management](#user-creation-and-management)
- [User Cloning and Privilege Copying](#user-cloning-and-privilege-copying)
- [Password Management](#password-management)
- [User Security and Auditing](#user-security-and-auditing)
- [Schema Analysis and Management](#schema-analysis-and-management)
- [Import/Export Operations](#importexport-operations)
- [Tablespace Operations](#tablespace-operations)
- [Network ACL Management](#network-acl-management)
- [Database Security Review](#database-security-review)

## User Creation and Management

### Basic User Creation

```sql
CREATE USER PI_READONLY
  IDENTIFIED BY PI_READONLY
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  PROFILE DEFAULT
  ACCOUNT UNLOCK;
```

### Extract User Password Hashes

```sql
SET LONG 150
SET LINESIZE 150
SET LONGCHUNKSIZE 150

SELECT 'ALTER USER ' || username || ' IDENTIFIED BY VALUES ' || 
       REGEXP_SUBSTR(DBMS_METADATA.get_ddl('USER', username), '''[^'']+''') || ';' 
FROM dba_users;
```

### Check User Account Status

```sql
ALTER SESSION SET nls_date_format = 'DY DD-MON-RRRR';

SELECT username, account_status 
FROM dba_users 
WHERE username LIKE '%PATTERN%';
```

### Get User Password Information

```sql
SELECT spare4 
FROM sys.user$ 
WHERE name = 'USERNAME';
```

### User Expiration Details

```sql
SELECT username, account_status, expiry_date, sysdate
FROM dba_users
WHERE username = '&usr';
```

## User Cloning and Privilege Copying

### Interactive User Clone Script

```sql
SET PAGES 0 FEED OFF VERIFY OFF LINES 500

ACCEPT oldname PROMPT "Enter user to model new user to: "
ACCEPT newname PROMPT "Enter new user name: "
ACCEPT psw PROMPT "Enter new user's password: "

-- Create user
SELECT 'CREATE USER &&newname IDENTIFIED BY &&psw' ||
       ' DEFAULT TABLESPACE ' || default_tablespace ||
       ' TEMPORARY TABLESPACE ' || temporary_tablespace || 
       ' PROFILE ' || profile || ';'
FROM sys.dba_users
WHERE username = UPPER('&&oldname');

-- Grant Roles
SELECT 'GRANT ' || granted_role || ' TO &&newname' ||
       DECODE(admin_option, 'YES', ' WITH ADMIN OPTION') || ';'
FROM sys.dba_role_privs
WHERE grantee = UPPER('&&oldname');

-- Grant System Privileges
SELECT 'GRANT ' || privilege || ' TO &&newname' ||
       DECODE(admin_option, 'YES', ' WITH ADMIN OPTION') || ';'
FROM sys.dba_sys_privs
WHERE grantee = UPPER('&&oldname');

-- Grant Table Privileges
SELECT 'GRANT ' || privilege || ' ON ' || owner || '.' || table_name || ' TO &&newname;'
FROM sys.dba_tab_privs
WHERE grantee = UPPER('&&oldname');

-- Grant Column Privileges
SELECT 'GRANT ' || privilege || ' ON ' || owner || '.' || table_name ||
       '(' || column_name || ') TO &&newname;'
FROM sys.dba_col_privs
WHERE grantee = UPPER('&&oldname');

-- Set Default Role
SELECT 'ALTER USER &&newname DEFAULT ROLE ' || granted_role || ';'
FROM sys.dba_role_privs
WHERE grantee = UPPER('&&oldname')
AND default_role = 'YES';
```

### Complete User DDL Export

```sql
SET HEAD OFF
SET PAGES 0
SET LONG 9999999

SPOOL user_script.sql

SELECT DBMS_METADATA.get_ddl('USER', '&user') FROM dual
UNION ALL
SELECT DBMS_METADATA.get_granted_ddl('SYSTEM_GRANT', '&user') FROM dual
UNION ALL
SELECT DBMS_METADATA.get_granted_ddl('OBJECT_GRANT', '&user') FROM dual
UNION ALL
SELECT DBMS_METADATA.get_granted_ddl('ROLE_GRANT', '&user') FROM dual
UNION ALL
SELECT DBMS_METADATA.get_granted_ddl('TABLESPACE_QUOTA', '&user') FROM dual;

SPOOL OFF;
```

## Password Management

### Update Profile Password Settings

```sql
ALTER PROFILE SPER_PROFILE_TEMPORARY LIMIT
  FAILED_LOGIN_ATTEMPTS 3
  PASSWORD_LIFE_TIME UNLIMITED;
```

### Reset Multiple User Passwords

```sql
SELECT 'ALTER USER "' || username || '" IDENTIFIED BY Password01 ACCOUNT UNLOCK;' 
FROM dba_users 
WHERE username NOT IN (
  'SYS', 'SYSTEM', 'SYSMAN', 'OUTLN', 'SCOTT', 'ADAMS', 'JONES', 'CLARK', 'BLAKE',
  'HR', 'OE', 'SH', 'DEMO', 'ANONYMOUS', 'AURORA$ORB$UNAUTHENTICATED',
  'AWR_STAGE', 'CSMIG', 'CTXSYS', 'DBSNMP', 'DIP', 'DMSYS', 'EXFSYS', 'LBACSYS',
  'MDSYS', 'ORACLE_OCM', 'ORDPLUGINS', 'ORDSYS', 'PERFSTAT', 'TRACESVR',
  'TSMSYS', 'XDB', 'OLAPSYS', 'WMSYS', 'OWBSYS', 'OWBSYS_AUDIT', 'XS$NULL'
);
```

### Password History Analysis

```sql
-- Check if password has ever changed
SELECT name,
       TO_CHAR(ctime, 'DD-MON-YY HH24:MI:SS') created,
       TO_CHAR(ptime, 'DD-MON-YY HH24:MI:SS') password_changed,
       LENGTH(password) pwd_length
FROM user$
WHERE password IS NOT NULL
AND password NOT IN ('GLOBAL', 'EXTERNAL')
AND ctime = ptime;
```

### Password Expiration Report

```sql
SET PAGESIZE 500
SET LINESIZE 200
SET TRIMSPOOL ON
SET FEEDBACK OFF
SET HEADING OFF

COLUMN "EXPIRE DATE" FORMAT A20

SELECT TO_CHAR(STARTUP_TIME, 'Month DD, YYYY HH24:MI:SS') "StartUp" 
FROM v$instance;

SELECT username AS "USER NAME", 
       TO_CHAR(expiry_date, 'DD-MON-YYYY') AS "EXPIRE DATE",
       account_status
FROM dba_users
WHERE expiry_date < SYSDATE + 30
AND account_status IN ('OPEN', 'EXPIRED(GRACE)');
```

## User Security and Auditing

### Last Login Information

```sql
COL USERNAME FORMAT A15
COL OS_USERNAME FORMAT A15
COL USERHOST FORMAT A15
COL ACTION_NAME FORMAT A15

SET PAGES 200
SET LINES 100

ALTER SESSION SET nls_date_format = 'DD-MON-YY HH24:MI:SS';

SELECT USERNAME, OS_USERNAME, USERHOST, TIMESTAMP, ACTION_NAME, LOGOFF_TIME
FROM dba_audit_trail 
WHERE username = '&USERNAME' 
ORDER BY USERHOST;
```

### Detailed Login History

```sql
SELECT TO_CHAR(TIMESTAMP#, 'MM/DD/YY HH:MI:SS') TIMESTAMP,
       USERID, 
       AA.NAME ACTION 
FROM SYS.AUD$ AT, SYS.AUDIT_ACTIONS AA
WHERE AT.ACTION# = AA.ACTION
AND AA.name = 'LOGON'
AND userid = '&User_id'
ORDER BY TIMESTAMP# DESC;
```

### Current Active Sessions

```sql
SELECT username, osuser, terminal,
       UTL_INADDR.get_host_address(terminal) IP_ADDRESS
FROM v$session
WHERE username IS NOT NULL
ORDER BY username, osuser;
```

## Schema Analysis and Management

### Schema Size Calculation

```sql
SELECT SUM(bytes) / 1024 / 1024 total_size_mb
FROM dba_segments
WHERE owner = '&SCHEMA_NAME';
```

### Schema Objects Summary

```sql
SET PAUSE OFF
SET PAGESIZE 9999
SET LINESIZE 132
SET FEEDBACK OFF
SET ECHO OFF
SET VERIFY OFF

COLUMN OWNER FORMAT A20 HEAD "Owner Name"
COLUMN OBJECT_NAME FORMAT A30 HEAD "Object Name"
COLUMN OBJECT_TYPE FORMAT A12 HEAD "Object Type"
COLUMN CREATED FORMAT A20 HEAD "Date Created"
COLUMN LAST_DDL_TIME FORMAT A20 HEAD "Date Last Defined"
COLUMN STATUS FORMAT A7 HEAD "Status"

ACCEPT own PROMPT 'Owner Name [*] : '
ACCEPT obj PROMPT 'Object Name [*] : '
ACCEPT typ PROMPT 'Object Type [*] : '

SELECT OWNER, OBJECT_NAME, OBJECT_TYPE,
       TO_CHAR(CREATED, 'DD-MON-YYYY HH24:MI:SS') created,
       TO_CHAR(LAST_DDL_TIME, 'DD-MON-YYYY HH24:MI:SS') last_ddl_time,
       STATUS
FROM dba_objects
WHERE OWNER LIKE DECODE('&own', '', '%', UPPER('&own'))
AND OBJECT_NAME LIKE DECODE('&obj', '', '%', UPPER('&obj'))
AND OBJECT_TYPE LIKE DECODE('&typ', '', '%', UPPER('&typ'))
ORDER BY LAST_DDL_TIME;
```

### Table Row Count Analysis

```sql
-- Dynamic row count generation
ACCEPT owner PROMPT "Enter table owner: "

SET ECHO OFF
SET FEED OFF
SET VERIFY OFF
SET PAGES 0
SET TERMOUT OFF

SPOOL count_tables.sql

SELECT 'SELECT RPAD(''' || table_name || ''', 39, ''.'') || COUNT(*) FROM &owner..' || table_name || ';'
FROM dba_tables
WHERE owner = UPPER('&owner')
ORDER BY 1;

SPOOL OFF

SET TERMOUT ON
@count_tables
```

### Comprehensive User Privileges Report

```sql
SET VERIFY OFF
SET PAGESIZE 80
SET LINESIZE 120
SET ECHO OFF
SET FEEDBACK OFF

COLUMN grantee FORMAT A10
COLUMN privilege FORMAT A70
COLUMN type FORMAT A15

BREAK ON grantee SKIP 1 ON type

ACCEPT grantee PROMPT "Enter grantee or blank for all: "

-- Roles
SELECT grantee, 'Roles...' type,
       granted_role || ' ' || DECODE(admin_option, 'YES', ' WITH ADMIN OPTION') privilege
FROM sys.dba_role_privs 
WHERE grantee = DECODE('&grantee', NULL, grantee, UPPER('&grantee'))
UNION
-- System Privileges
SELECT grantee, 'System Privs...' type,
       privilege || ' ' || DECODE(admin_option, 'YES', ' WITH ADMIN OPTION') privilege
FROM sys.dba_sys_privs
WHERE grantee = DECODE('&grantee', NULL, grantee, UPPER('&grantee'))
UNION
-- Object Privileges
SELECT grantee, 'Object Privs...' type,
       privilege || ' ' || owner || '.' || table_name ||
       DECODE(grantable, 'YES', ' WITH GRANT OPTION') privilege
FROM sys.dba_tab_privs
WHERE grantee = DECODE('&grantee', NULL, grantee, UPPER('&grantee'));
```

## Import/Export Operations

### Generate Schema List for Export

```sql
SET LINES 2000
SET HEADING OFF

SPOOL exp_accounts.txt

SELECT LISTAGG(username, ',') WITHIN GROUP (ORDER BY username)
FROM dba_users 
WHERE username NOT IN (
  'SYS', 'SYSMAN', 'SYSTEM', 'OUTLN', 'SCOTT', 'ADAMS', 'JONES', 'CLARK', 'BLAKE',
  'BI', 'HR', 'OE', 'PM', 'IX', 'SH', 'DEMO', 'ANONYMOUS', 'APEX_PUBLIC_USER',
  'AWR_STAGE', 'CSMIG', 'CTXSYS', 'DBSNMP', 'DIP', 'DMSYS', 'DSSYS', 'EXFSYS',
  'FLOWS_040100', 'FLOWS_FILES', 'LBACSYS', 'MGMT_VIEW', 'MDDATA', 'MDSYS',
  'OLAPSYS', 'ORACLE_OCM', 'ORDDATA', 'ORDPLUGINS', 'ORDSYS', 'OWBSYS',
  'PERFSTAT', 'SI_INFORMTN_SCHEMA', 'SPATIAL_CSW_ADMIN_USR', 'SPATIAL_WFS_ADMIN_USR',
  'TRACESVR', 'TSMSYS', 'WK_TEST', 'WKSYS', 'WMSYS', 'XDB', 'XS$NULL',
  'FNMPAUDIT', 'OWBSYS_AUDIT', 'APEX_030200', 'APPQOSSYS'
)
AND username NOT LIKE 'APAC%'
AND profile NOT IN ('&profile');

SPOOL OFF
```

## Tablespace Operations

### Move Schema to New Tablespace

#### Step 1: Create New Tablespace

```sql
CREATE BIGFILE TABLESPACE SCHEMA_TS DATAFILE 
  '+DATA' SIZE 43456000K AUTOEXTEND ON NEXT 100M MAXSIZE 43456000K
LOGGING
ONLINE
EXTENT MANAGEMENT LOCAL AUTOALLOCATE
BLOCKSIZE 8K
SEGMENT SPACE MANAGEMENT AUTO
FLASHBACK ON;
```

#### Step 2: Set Default Tablespace and Quota

```sql
ALTER USER SCHEMA_NAME DEFAULT TABLESPACE SCHEMA_TS;
ALTER USER SCHEMA_NAME QUOTA UNLIMITED ON SCHEMA_TS;
```

#### Step 3: Generate Move Scripts

```sql
SPOOL move_objects.sql

-- Move tables
SELECT 'ALTER TABLE ' || owner || '.' || table_name || ' MOVE TABLESPACE SCHEMA_TS;' 
FROM DBA_TABLES 
WHERE OWNER = 'SCHEMA_NAME';

-- Rebuild indexes
SELECT 'ALTER INDEX ' || owner || '.' || index_name || ' REBUILD TABLESPACE SCHEMA_TS;' 
FROM DBA_INDEXES 
WHERE OWNER = 'SCHEMA_NAME';

SPOOL OFF

@move_objects.sql
```

#### Step 4: Recompile Objects

```sql
@?/rdbms/admin/utlrp.sql
```

## Network ACL Management

### View Current ACL Privileges

```sql
COLUMN ACL FORMAT A50
COLUMN PRINCIPAL FORMAT A20
COLUMN ACL_OWNER FORMAT A10
COLUMN PRIVILEGE FORMAT A10

SELECT * FROM dba_network_acl_privileges 
ORDER BY principal;
```

### Grant Network Privileges

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.add_privilege (
    acl => 'NETWORK_ACL_ID',
    principal => 'USERNAME',
    is_grant => TRUE,
    privilege => 'connect'
  );
  COMMIT;
END;
/

BEGIN
  DBMS_NETWORK_ACL_ADMIN.add_privilege (
    acl => 'NETWORK_ACL_ID',
    principal => 'USERNAME',
    is_grant => TRUE,
    privilege => 'resolve'
  );
  COMMIT;
END;
/

BEGIN
  DBMS_NETWORK_ACL_ADMIN.add_privilege (
    acl => 'NETWORK_ACL_ID',
    principal => 'USERNAME',
    is_grant => TRUE,
    privilege => 'smtp'
  );
  COMMIT;
END;
/
```

## Database Security Review

### Comprehensive Security Audit Script

```sql
-- Database Version Information
SELECT * FROM PRODUCT_COMPONENT_VERSION;

-- SYSDBA and SYSOPER Users
SELECT * FROM V$PWFILE_USERS;

-- User Accounts
SELECT * FROM sys.dba_users ORDER BY username;

-- Password Parameters
SELECT profile, resource_name, limit
FROM sys.dba_profiles
WHERE resource_type = 'PASSWORD'
ORDER BY profile;

-- Critical Database Parameters
SELECT name, value, isdefault, description
FROM v$parameter
WHERE name IN (
  'audit_trail', 'db_name', 'audit_sys_operations', 'remote_os_authent',
  'remote_os_roles', 'os_roles', 'remote_login_passwordfile',
  'O7_DICTIONARY_ACCESSIBILITY', 'sql92_security'
)
ORDER BY name;

-- Database Links
SELECT * FROM sys.dba_db_links;

-- Public Privileges (Security Risk Assessment)
SELECT * FROM sys.dba_sys_privs WHERE grantee = 'PUBLIC';
SELECT * FROM sys.dba_tab_privs WHERE grantee = 'PUBLIC' ORDER BY table_name, privilege;

-- Audit Options
SELECT * FROM sys.dba_priv_audit_opts;
SELECT * FROM sys.dba_obj_audit_opts;
```

### Find Users with Restricted Session Access

```sql
SELECT b.grantee, a.grantee || ' (Role)' AS granted
FROM dba_sys_privs a, dba_role_privs b
WHERE a.privilege = 'RESTRICTED SESSION'
AND a.grantee = b.granted_role
UNION
SELECT b.username, 'User (Direct)'
FROM dba_sys_privs a, dba_users b
WHERE a.privilege = 'RESTRICTED SESSION'
AND a.grantee = b.username;
```

## Usage Notes

- Replace `&variable_name` placeholders with actual values when running scripts
- Many scripts require DBA privileges to execute
- Always test scripts in a development environment first
- Some queries may take time to execute on large databases
- Consider performance impact when running comprehensive audit scripts

## Contributing

Feel free to contribute additional Oracle user management and security scripts by submitting a pull request.

## License

This collection is provided as-is for educational and operational purposes.
