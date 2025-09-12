# Oracle Database Management Scripts

## Table of Contents
- [Sequences](#sequences)
- [Synonyms](#synonyms)
- [Views](#views)
- [Permissions and Grants](#permissions-and-grants)
- [Database Links](#database-links)

## Sequences

### Creating Auto-Sequence for Table Column
For auto sequence value of a table column (e.g., SEQID):

```sql
CREATE SEQUENCE HRI.BATCHDATA_SEQID
  START WITH 205139
  MAXVALUE 999999999
  MINVALUE 1
  CYCLE
  CACHE 20
  NOORDER;
```

### Trigger for Auto-Generation
```sql
CREATE OR REPLACE TRIGGER "HRI"."GENERATEBATCHDATAID" 
BEFORE INSERT ON HRI.Tbl_BatchData FOR EACH ROW
BEGIN
  SELECT batchData_SeqId.NEXTVAL
  INTO   :new.SEQID
  FROM   dual;
  
  SELECT CURRENT_TIMESTAMP
  INTO :new.CREATION_TIME
  FROM dual;
END;
/
```

### Query All Sequences
```sql
SELECT SEQUENCE_OWNER,
       SEQUENCE_NAME,
       MIN_VALUE,
       MAX_VALUE,
       INCREMENT_BY,
       CYCLE_FLAG,
       ORDER_FLAG,
       CACHE_SIZE,
       LAST_NUMBER
FROM dba_sequences
WHERE SEQUENCE_OWNER NOT IN ('SYS','SYSTEM')
ORDER BY SEQUENCE_OWNER, SEQUENCE_NAME;
```

## Synonyms

### Query All User-Defined Synonyms
```sql
SELECT OWNER,
       SYNONYM_NAME,
       TABLE_OWNER,
       TABLE_NAME,
       DB_LINK
FROM dba_synonyms
WHERE owner NOT IN ('SYS','SYSTEM','PUBLIC','DBSNMP')
ORDER BY OWNER, SYNONYM_NAME;
```

### Generate Synonym Creation Script
Script to capture all currently defined synonyms for a schema and create a recreation script:

```sql
SELECT 'CREATE OR REPLACE SYNONYM ' || owner || '.' || synonym_name || 
       ' FOR ' || table_owner || '.' || table_name || ';' 
FROM all_synonyms 
WHERE owner LIKE '%' 
ORDER BY owner, synonym_name;
```

### Create Synonyms for Specific Schema
Generate synonyms for read-only access:

```sql
SET heading OFF
SET echo OFF
SET feedback OFF
SPOOL D008_RO.sql

SELECT 'CREATE SYNONYM D008_RO.' || table_name || ' FOR D008.' || table_name || ';'  
FROM dba_tables 
WHERE OWNER = 'D008';

SPOOL OFF
```

Generate synonyms for read-write access:

```sql
SET heading OFF
SET echo OFF
SET feedback OFF
SPOOL D008_RW.sql

SELECT 'CREATE SYNONYM D008_RW.' || table_name || ' FOR D008.' || table_name || ';'  
FROM dba_tables 
WHERE OWNER = 'D008';

SPOOL OFF
```

### Synonym Syntax and Prerequisites

**Syntax:**
```sql
CREATE [PUBLIC] SYNONYM [SCHEMA.]synonym FOR [SCHEMA.]object[@dblink]
```

**Prerequisites:**
- To create a private synonym in your own schema: `CREATE SYNONYM` system privilege
- To create a private synonym in another user's schema: `CREATE ANY SYNONYM` system privilege

### Public Synonym Management
```sql
-- Drop and recreate public synonym
DROP PUBLIC SYNONYM court_type_code_cot;
CREATE PUBLIC SYNONYM court_type_code_cot FOR prodowner.court_type_code_cot;
GRANT SELECT ON prodowner.court_type_code_cot TO WIMANGX;
```

## Views

### Query All User Views
```sql
SELECT OWNER,
       OBJECT_NAME,
       TO_CHAR(CREATED,'MM/DD/YYYY HH24:MI:SS') AS created,
       status
FROM dba_objects
WHERE OWNER NOT IN ('SYS','SYSTEM')
  AND OBJECT_TYPE = 'VIEW'
ORDER BY OWNER, OBJECT_NAME;
```

### Sample View Creation
```sql
CREATE VIEW view_ctype_code_cot AS
SELECT *
FROM prodowner.court_type_code_cot
WHERE cot_court_type_id <> 'CORCT';
```

## Permissions and Grants

### User Permission Management
```sql
-- Revoke and grant permissions
REVOKE create procedure, create sequence, create table, create trigger, create view FROM &user;
GRANT create session, create any synonym TO &user;

REVOKE create procedure, create sequence, create table, create trigger, create view FROM &user_ro;
GRANT create procedure, create sequence, create table, create trigger, create view TO &user_ro;
```

### Generate Grant Statements
Capture and run output from these commands for bulk permission grants:

```sql
SET heading OFF
SET echo OFF
SET feedback OFF

SELECT 'GRANT delete, insert, update, select ON ' || table_name || ' TO USER_UPD;' 
FROM dba_tables 
WHERE owner = 'schema_owner';
```

### Public Synonym with Grants
```sql
CREATE PUBLIC SYNONYM JJV_MA_RPT_CTL FOR ELLIPSE.JJV_MA_RPT_CTL;
GRANT SELECT ON ELLIPSE.JJV_MA_RPT_CTL TO corvu_read;
```

## Database Links

### Query Database Links
```sql
COLUMN OWNER HEADING 'Owner' FORMAT A15
COLUMN DB_LINK HEADING 'Database Link' FORMAT A30
COLUMN USERNAME HEADING 'Username' FORMAT A20
COLUMN HOST HEADING 'Host' FORMAT A25

SELECT OWNER, DB_LINK, USERNAME, HOST
FROM dba_db_links;
```

### Get Database Link DDL
```sql
SET pages 999
SET long 90000
SET linesize 120

SELECT 'SELECT dbms_metadata.get_ddl(''DB_LINK'',''' || db_link || 
       ''',''' || owner || ''') FROM dual;' AS "Execute Query for DDL"
FROM dba_db_links 
WHERE db_link = '&DBLINKNAME';
```

**Sample DDL Output:**
```sql
CREATE DATABASE LINK "TEST.TRNSPRT"
   CONNECT TO "<USER>" IDENTIFIED BY VALUES '0XXXXXXXXXXXXXXXXXX0890FD'
   USING 'TNSALIAS/CONNECTSTRING';
```

### Query Specific Database Link Details
```sql
SET linesize 200
COLUMN OWNER FORMAT A15
COLUMN DB_LINK FORMAT A30
COLUMN USERNAME FORMAT A15
COLUMN HOST FORMAT A30

SELECT OWNER, DB_LINK, USERNAME, HOST
FROM DBA_DB_LINKS
WHERE db_link = '&linkname';
```

### Monitor Database Link Usage
Script to identify which sessions are using database links by matching GTXID:

```sql
SELECT /*+ ORDERED */
       SUBSTR(s.ksusemnm,1,10) || '-' || SUBSTR(s.ksusepid,1,10) AS "ORIGIN",
       SUBSTR(g.K2GTITID_ORA,1,35) AS "GTXID",
       SUBSTR(s.indx,1,4) || '.' || SUBSTR(s.ksuseser,1,5) AS "LSESSION",
       s2.username,
       SUBSTR(
           DECODE(BITAND(ksuseidl,11),
               1, 'ACTIVE',
               0, DECODE(BITAND(ksuseflg,4096), 0, 'INACTIVE', 'CACHED'),
               2, 'SNIPED',
               3, 'SNIPED',
               'KILLED'
           ), 1, 1
       ) AS "S",
       SUBSTR(w.event,1,10) AS "WAITING"
FROM x$k2gte g, 
     x$ktcxb t, 
     x$ksuse s, 
     v$session_wait w, 
     v$session s2
WHERE g.K2GTDXCB = t.ktcxbxba
  AND g.K2GTDSES = t.ktcxbses
  AND s.addr = g.K2GTDSES
  AND w.sid = s.indx
  AND s2.sid = w.sid;
```

## Notes

- Run scripts on both ends of database links to match sessions using GTXID
- Use `SET heading OFF`, `SET echo OFF`, `SET feedback OFF` for clean script output
- Always test permission changes in development environment first
