## SEQUENCE

For auto sequence value of a table column (Ex: SEQID)
```
CREATE SEQUENCE HRI.BATCHDATA_SEQID
  START WITH 205139
  MAXVALUE 999999999
  MINVALUE 1
  CYCLE
  CACHE 20
  NOORDER;
```
```
CREATE OR REPLACE TRIGGER "HRI"."GENERATEBATCHDATAID" BEFORE INSERT ON HRI.Tbl_BatchData FOR EACH ROW
BEGIN
  SELECT batchData_SeqId.NEXTVAL
  INTO   :new.SEQID
  FROM   dual;
  SELECT CURRENT_TIMESTAMP
  INTO :new.CREATION_TIME
   from dual;
END;
/
```

### Synonym
==========================
```
select	OWNER,
	SYNONYM_NAME,
	TABLE_OWNER,
	TABLE_NAME,
	DB_LINK
from  	dba_synonyms
where	owner not in ('SYS','SYSTEM','PUBLIC','DBSNMP')
order 	by OWNER,SYNONYM_NAME
```
script : take all the currently defined synonyms for a certain schema and create a script that can be used to recreate it 
```
select 'create or replace synonym '||owner||'.'||synonym_name||' for '||table_owner||'.'||table_name||';' 
from all_synonyms 
where owner like '%' 
order by owner, synonym_name; 
```
```
select 	SEQUENCE_OWNER,
	SEQUENCE_NAME,
	MIN_VALUE,
	MAX_VALUE,
	INCREMENT_BY,
	CYCLE_FLAG,
	ORDER_FLAG,
	CACHE_SIZE,
	LAST_NUMBER
from  	dba_sequences
where	SEQUENCE_OWNER not in ('SYS','SYSTEM')
order 	by SEQUENCE_OWNER,SEQUENCE_NAME
/
```
## View
===============
```
select 	OWNER,
	OBJECT_NAME,
	to_char(CREATED,'MM/DD/YYYY HH24:MI:SS') created,
	status
from  	dba_objects
where	OWNER not in ('SYS','SYSTEM')
and	OBJECT_TYPE='VIEW'
order	by OWNER,OBJECT_NAME
```
Sample Create command
```
CREATE VIEW view_ctype_code_cot AS
SELECT *
FROM prodowner.court_type_code_cot
WHERE cot_court_type_id <> 'CORCT';
```
The syntax for create synonym is: create [PUBLIC] synonym [SCHEMA.]synonym FOR [SCHEMA.]object[@dblink] 

Prerequisites 
To create a private synonym in your own schema, you must have CREATE SYNONYM system privilege. 
To create a private synonym in another user?s schema, you must have CREATE ANY SYNONYM system privilege. 

```
set heading off
set echo off
set feedback off
spool D008_RO.sql
select 'create synonym ' || 'D008_RO'|| '.' || table_name || ' for D008.' || table_name || ';'  from dba_tables where OWNER = 'D008' ;
Spool off
```
```
set heading off
set echo off
set feedback off
spool D008_RW.sql
select 'create synonym ' || 'D008_RW'|| '.' || table_name || ' for D008.' || table_name || ';'  from dba_tables where OWNER = 'D008' ;
spool off
```
```
revoke create procedure, create sequence, create table, create trigger, create view from
grant create session, create any synonym to &user;
revoke create procedure, create sequence, create table, create trigger, create view from &user_ro;
grant create procedure, create sequence, create table, create trigger, create view to &user_ro;
create public synonym JJV_MA_RPT_CTL for ELLIPSE.JJV_MA_RPT_CTL;
grant select on ELLIPSE.JJV_MA_RPT_CTL to corvu_read;
```

.e. capture & run the output from these commands

```
set heading off
set echo off
set feedback off
```
```
select 'grant delete, insert, update, select on ' || table_name || ' to
USER_UPD;' 
from dba_tables where owner = 'schema owner' 
/
```

```
drop public synonym court_type_code_cot ;
create public synonym court_type_code_cot for prodowner.court_type_code_cot;
grant select on prodowner.court_type_code_cot to WIMANGX;
```

LINK
Who is using the DB Link?  
```
-- this script can be used at both ends of the database link
-- to match up which session on the remote database started
-- the local transaction
-- the GTXID will match for those sessions
-- just run the script on both databases

Select /*+ ORDERED */
substr(s.ksusemnm,1,10)||'-'|| substr(s.ksusepid,1,10)      "ORIGIN",
substr(g.K2GTITID_ORA,1,35) "GTXID",
substr(s.indx,1,4)||'.'|| substr(s.ksuseser,1,5) "LSESSION" ,
s2.username,
substr(
   decode(bitand(ksuseidl,11),
      1,'ACTIVE',
      0, decode( bitand(ksuseflg,4096) , 0,'INACTIVE','CACHED'),
      2,'SNIPED',
      3,'SNIPED',
      'KILLED'
   ),1,1
) "S",
substr(w.event,1,10) "WAITING"
from  x$k2gte g, x$ktcxb t, x$ksuse s, v$session_wait w, v$session s2
where  g.K2GTDXCB =t.ktcxbxba
and   g.K2GTDSES=t.ktcxbses
and  s.addr=g.K2GTDSES
and  w.sid=s.indx
and s2.sid = w.sid;
```
```
COLUMN OWNER HEADING 'Owner' FORMAT A15
COLUMN DB_LINK HEADING 'Database Link' FORMAT A30
COLUMN USERNAME HEADING 'Username' FORMAT A20
COLUMN HOST HEADING 'Host' FORMAT A25

SELECT OWNER, DB_LINK, USERNAME, HOST
FROM dba_db_links;

```
```
set pages 999
set long 90000
set lin 120

SELECT 'SELECT dbms_metadata.get_ddl(''DB_LINK'',''' || db_link || ''',''' || owner || ''') as "test.trns" FROM dual;' AS "Execute below Query for DDL"
FROM dba_db_links WHERE db_link IN ('&DBlINKNAME'); <<-- 
```
Sample Output:
```
CREATE DATABASE LINK "TEST.TRNSPRT"
   CONNECT TO "<USER>" IDENTIFIED BY VALUES '0XXXXXXXXXXXXXXXXXX0890FD'
   USING 'TNSALIAS/CONNECTSTRING';
```
Now execute the command
