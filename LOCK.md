
### Lists all locked objects for whole RAC.
``` 
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN owner FORMAT A20
COLUMN username FORMAT A20
COLUMN object_owner FORMAT A20
COLUMN object_name FORMAT A30
COLUMN locked_mode FORMAT A15
 
SELECT b.inst_id,
       b.session_id AS sid,
       NVL(b.oracle_username, '(oracle)') AS username,
       a.owner AS object_owner,
       a.object_name,
       Decode(b.locked_mode, 0, 'None',
                             1, 'Null (NULL)',
                             2, 'Row-S (SS)',
                             3, 'Row-X (SX)',
                             4, 'Share (S)',
                             5, 'S/Row-X (SSX)',
                             6, 'Exclusive (X)',
                             b.locked_mode) locked_mode,
       b.os_user_name
FROM   dba_objects a,
       gv$locked_object b
WHERE  a.object_id = b.object_id
ORDER BY 1, 2, 3, 4
/

```

### LOCKED OBJECTS
```
prompt 
prompt +----------------------------------------------------------------------------+
prompt | LOCKED OBJECTS                                                             |
prompt +----------------------------------------------------------------------------+
prompt 


SELECT
    i.instance_name           instance_name
  , RPAD(l.session_id,7)      sid
  , RPAD(s.serial#,7)         serial_number
  , s.status                  session_status
  , l.oracle_username         oracle_user
  , l.os_user_name            os_username
  , o.owner                   object_owner
  , o.object_name             object_name
  , o.object_type             object_type
FROM
    dba_objects       o
  , gv$session        s
  , gv$locked_object  l
  , gv$instance       i
WHERE
      i.inst_id    = l.inst_id
  AND l.inst_id    = s.inst_id
  AND l.session_id = s.sid
  AND o.object_id  = l.object_id
ORDER BY
    l.session_id
/

Select 
   s.sid SID,
   s.serial# Serial#,
   l.type type,
   ' ' object_name,
   lmode held,
   request request
from 
   v$lock l, 
   v$session s, 
   v$process p
where 
   s.sid = l.sid and
   s.username <> ' ' and
   s.paddr = p.addr and
   l.type <> 'TM' and
   (l.type <> 'TX' or l.type = 'TX' and l.lmode <> 6)
union
select 
   s.sid SID,
   s.serial# Serial#,
   l.type type,
   object_name object_name,
   lmode held,
   request request
from 
   v$lock l,
   v$session s,
   v$process p, 
   sys.dba_objects o
where 
   s.sid = l.sid and
   o.object_id = l.id1 and
   l.type = 'TM' and
   s.username <> ' ' and
   s.paddr = p.addr
union
select 
   s.sid SID,
   s.serial# Serial#,
   l.type type,
   '(Rollback='||rtrim(r.name)||')' object_name,
   lmode held,
   request request
from 
   v$lock l, 
   v$session s, 
   v$process p, 
   v$rollname r
where 
   s.sid = l.sid and
   l.type = 'TX' and
   l.lmode = 6 and
   trunc(l.id1/65536) = r.usn and
   s.username <> ' ' and
   s.paddr = p.addr
order by 5, 6;

REM Lock Types:
REM                'MR', 'Media Recov',
REM                'RT', 'Redo Thread',
REM                'UN', 'User Name',
REM                'TX', 'Txn',
REM                'TM', 'DML',
REM                'UL', 'PL/SQL User',
REM                'DX', 'Distrib Txn',
REM                'CF', 'Control File',
REM                'IS', 'Instance State',
REM                'FS', 'File Set',
REM                'IR', 'Instance Recov',
REM                'ST', 'Disk Space Txn',
REM                'TS', 'Temp Segment',
REM                'IV', 'Lib Cache',
REM                'LS', 'Log St/Sw',
REM                'RW', 'Row Wait',
REM                'SQ', 'Sequence #',
REM                'TE', 'Extend Table',
REM                'TT', 'Temp Table',
 
set pause off
set pagesize 9999
set linesize 150
set feedback off
set echo off
 
column "User"           format a12
column "Session"        heading "Session" format a14
column "Client Host"    format a15
column "Client PID"     heading "Client|PID" format a6
column "Server PID"     heading "Server|PID" format a6
column "Satus"          format a8
column "Lock Type"      heading "Lock|Type" format a4
column "Mode Held"      format a14
column "Mode Requested" format a14
column "Lock ID"        format a14
column "Object"         format a30
 
select /*+ ORDERED */
       s.USERNAME "User"
     , chr(39) || to_char(s.SID) || ',' ||
                  to_char(s.serial#) || chr(39) "Session"
     , s.MACHINE "Client Host"
     , s.PROCESS "Client PID"
     , to_char(p.spid)  "Server PID"
     , s.STATUS "Status"
     , l.type "Lock Type"
     , decode(l.lmode,
                0, 'None',           /* Mon Lock equivalent */
                1, 'Null',           /* N */
                2, 'Row-S (SS)',     /* L */
                3, 'Row-X (SX)',     /* R */
                4, 'Share',          /* S */
                5, 'S/Row-X (SSX)',  /* C */
                6, 'Exclusive',      /* X */
                to_char(l.lmode)) "Mode Held"
     , decode(l.request,
                0, 'None',           /* Mon Lock equivalent */
                1, 'Null',           /* N */
                2, 'Row-S (SS)',     /* L */
                3, 'Row-X (SX)',     /* R */
                4, 'Share',          /* S */
                5, 'S/Row-X (SSX)',  /* C */
                6, 'Exclusive',      /* X */
                to_char(l.request)) "Mode Requested"
     , to_char(l.id1) || '.' || to_char(l.id2) "Lock ID"
     , decode(l.type,'TX',r.name,o.name) "Object"
from  v$lock l
    , v$session s
    , v$process p
    , sys.obj$ o
    , v$rollname r
where p.addr = s.paddr
and   l.sid = s.sid
and   o.obj#(+) = decode(l.id2,0,l.id1,l.id2)
and   r.usn(+) = decode(l.type,'TX',trunc(l.id1/65536),9999999999)
and   s.type != 'BACKGROUND'
order by decode(l.id2,0,l.id1,l.id2)
    , 8
    , 9
/
```

-----------------------------------------------------------------------------------------
### Find out which sessions are waiting for each other
-------------------------------------------------------------------------------------------
```
SELECT DECODE(request,0,'Holder: ','Waiter: ')||sid sess, 
        id1, id2, lmode, request, type
   FROM V$LOCK
  WHERE (id1, id2, type) IN
            (SELECT id1, id2, type FROM V$LOCK WHERE request>0)
  ORDER BY id1, request;
```
### Get the holder Session ID -- find out SID and serial
```
select OSUSER, USERNAME, sid, serial#  from v$session where USERNAME='&usr';
```
```
 SELECT s.sid, s.serial#, s.username, s.osuser, p.spid, s.machine, p.terminal, s.program
    FROM v$session s, v$process p
    WHERE s.paddr = p.addr
    and s.sid=&SID ;
```
or
```
SELECT s.sid, s.serial#, s.status, p.spid
FROM v$session s, v$process p
WHERE s.username = '&usr' --<<<--
AND p.addr(+) = s.paddr
/
```
```
SELECT USERNAME,
           TERMINAL,
           PROGRAM,
           SQL_ID,
           LOGON_TIME,
           ROUND((SYSDATE-LOGON_TIME)*(24*60),1) as MINUTES_LOGGED_ON,
           ROUND(LAST_CALL_ET/60,1) as Minutes_FOR_CURRENT_SQL
      From v$session
     WHERE STATUS='ACTIVE'
      AND USERNAME in ('MSGIS','MSHIST')
   ORDER BY MINUTES_LOGGED_ON DESC;
```
### IF YOU NEED TO KILL THE SESSION AFTER CONFIRMING FROM THE RESPECTIVE TEAM / USER - 
```
select sid, serial#  from v$session where SID=&SID;
```
then run the following:[Grep SID and Serial from above output]
```
alter system kill session '&SID,&SERIAL' immediate;
```
If it takes time -- 2minutes then Check the process ID 
======================
TO GET PROCESS ID
======================
```
SELECT s.sid, s.serial#, s.username, s.osuser, p.spid, s.machine, p.terminal, s.program
FROM v$session s, v$process p
WHERE s.paddr = p.addr
and s.sid=&SID ;
```
```
SELECT s.sid, s.serial#, s.username, s.osuser, p.spid, s.machine, p.terminal, s.program
FROM v$session s, v$process p
WHERE s.paddr = p.addr
and p.spid=&SPID ;
```
```
select s.sid, s.serial#, s.status, p.spid 
from v$session s, v$process p 
where s.username = '&usr' 
and p.addr (+) = s.paddr; 
```
### Then kill from Unix prompt
```
kill -9 <PID>
```
-----------------------------------------------------------------------------------------------------------------------------------------
### You need to find out which user is waiting and which user is holding and then find out which SQL is being run. Do the following:
-----------------------------------------------------------------------------------------------------------------------------------------
Run the "wait.sql" script and determine who is waiting for     which holding process.

wait.sql
```
SELECT substr(s1.username,1,12)    "WAITING User",
       substr(s1.osuser,1,8)            "OS User",
       substr(to_char(w.session_id),1,5)    "Sid",
       P1.spid                              "PID",
       substr(s2.username,1,12)    "HOLDING User",
       substr(s2.osuser,1,8)            "OS User",
       substr(to_char(h.session_id),1,5)    "Sid",
       P2.spid                              "PID"
FROM   sys.v_$process P1,   sys.v_$process P2,
       sys.v_$session S1,   sys.v_$session S2,
       sys.dba_locks w,     sys.dba_locks h
WHERE
  (((h.mode_held != 'None') and (h.mode_held != 'Null')
     and ((h.mode_requested = 'None') or (h.mode_requested = 'Null')))
   and  (((w.mode_held = 'None') or (w.mode_held = 'Null'))
     and ((w.mode_requested != 'None') and (w.mode_requested != 'Null'))))
  and  w.lock_type  =  h.lock_type
  and  w.lock_id1  =  h.lock_id1
  and  w.lock_id2        =  h.lock_id2
  and  w.session_id     !=  h.session_id
  and w.session_id       = S1.sid
  and h.session_id       = S2.sid
  AND    S1.paddr           = P1.addr
  AND    S2.paddr           = P2.addr;
```

### Run the "whatsql.sql" script to determine which SQL is being run by the holding process.

whatsql.sql
```
SELECT /*+ ORDERED */
       s.sid, s.username, s.osuser, 
       nvl(s.machine, '?') machine, 
       nvl(s.program, '?') program,
       s.process F_Ground, p.spid B_Ground, 
       X.sql_text
FROM   sys.v_$session S,
       sys.v_$process P, 
       sys.v_$sqlarea X
WHERE  s.osuser      like lower(nvl('&OS_User','%'))
AND    s.username    like upper(nvl('&Oracle_User','%'))
AND    s.sid         like nvl('&SID','%')
AND    s.paddr          = p.addr 
AND    s.type          != 'BACKGROUND' 
AND    s.sql_address    = x.address
AND    s.sql_hash_value = x.hash_value
ORDER
    BY S.sid
```
```
select a.object_name,logon_time,
		decode (o.locked_mode,0,'NONE', 1,'NULL', 2,'RS', 3,'RX', 4,'S', 5,'SRX', 6,'X') TYPE, 
		o.oracle_username username, o.process, o.session_id,s.serial#, 
		s.lockwait, s.machine ,s.program
		from v$locked_object o,all_objects a, v$session s 
		where o.object_id = a.object_id and 
		o.process = s.process and 
		o.session_id = s.sid; 
```
```
select s1.username as blocker,s1.machine b_machine,
s2.username h_username,s2.machine h_machine,s3.sql_text
from v$lock l1, v$session s1, v$lock l2, v$session s2,v$sqltext s3
where s1.sid=l1.sid and s2.sid=l2.sid
and l1.BLOCK=1 and l2.request > 0
and l1.id1 = l2.id1
and l2.id2 = l2.id2
AND s1.PREV_SQL_ADDR =s3.address
and s1.PREV_HASH_VALUE=s3.HASH_VALUE;
```
### ------------------ DeadLock ---------------------------
ORA-000060: Deadlock detected. More info in file /oracle/admin/<>/udump/<>_ora_24409.trc.

If you are, one session automatically gets rolled back - so there is nothing to check and you will have a trace file with deadlock detected in it.
For staying long period deadlock you can check the following:

To find out which sessions are waiting for each other, you can execute :
```
SELECT DECODE(request,0,'Holder: ','Waiter: ')||sid sess, 
        id1, id2, lmode, request, type
   FROM V$LOCK
  WHERE (id1, id2, type) IN
            (SELECT id1, id2, type FROM V$LOCK WHERE request>0)
  ORDER BY id1, request;
```

```
SELECT sid, sql_hash_value
FROM V$SESSION
WHERE SID =&HolderSID;
```
and then use the hash value
```
select sql_text from V$SQLTEXT a  where hash_value = &HASHVALUE;
```

### GENERATE SQL TO DISCONNECT BLOCKING USERS.
```
select 'alter system disconnect session '''||sid||','||serial#||'''immediate;' from V$SESSION where sid in (select sid from V$LOCK where block=1);
```

### SHOW LOCKS.
```
select b.sid,b.username,d.id1,a.sql_text from V$SESSION b,V$LOCK d,V$SQLTEXT a
where b.lockwait = d.kaddr and   a.address = b.sql_address and   a.hash_value = b.sql_hash_value;
```
### SHOW LOCKING USERS.
```
select a.sid,a.username,b.id1,c.sql_text from V$SESSION a,V$LOCK b,V$SQLTEXT c
where b.id1 in (select distinct e.id1 from V$SESSION d, V$LOCK e where d.lockwait = e.kaddr) 
and a.sid = b.sid and c.hash_value = a.sql_hash_value and b.request = 0;
```
===============================================
### SHOW LONG OPERATIONS STILL IN PROGRESS  ###
===============================================
```
select b.sid,b.username,d.id1,a.sql_text from V$SESSION b,V$LOCK d,V$SQLTEXT a where b.lockwait = d.kaddr
and a.address = b.sql_address and a.hash_value = b.sql_hash_value;
```
```
select 
   sql_text 
from 
  sys.v_$sqltext 
where 
  hash_value = 1605085473
order by 
   piece;
```
```
select a.sql_text
from v$SQLAREA a , v$SESSION b
where a.ADDRESS=b.SADDR
and b.SID='&SID'
```
or
```
select s.sid, q.sql_text from v$sqltext q, v$session s
where q.address = s.sql_address
and s.sid = &sid
order by piece;
```
```
SPOOL v_lock.log;
set linesize 200
column SID format 9999 
column EVENT format a30
column P1 format a30
column P2 format a30
column P3 format a30 
column WAIT_T format 9999 
select SID,EVENT,P1TEXT||' '||P1 P1,P2TEXT||' '||P2 P2,P3TEXT||' '||P3 P3,SECONDS_IN_WAIT SEC_IN_WAIT 
from v$session_wait 
where sid in (&SIDLIST);
--example sidlist entry: 100,118,125

select s.username, s.sid, s.module, t.sql_text
from v$session s, v$sql t 
where s.sql_address =t.address and s.sql_hash_value =t.hash_value 
and s.sid in (&SIDLIST);
--example sidlist entry: 100,118,125

SPOOL OFF; 
```
######################################################
### Find SQL being executed by a OS Process ID (PID) #
######################################################
```
prompt "Please Enter The UNIX Process ID"
set pagesize 50000
set linesize 30000
set long 500000
set head off
select
s.username su,
substr(sa.sql_text,1,540) txt
from v$process p,
v$session s,
v$sqlarea sa
where p.addr=s.paddr
and s.username is not null
and s.sql_address=sa.address(+)
and s.sql_hash_value=sa.hash_value(+)
and spid=&SPID;
```
###############################
### List SQL being executed by a particular SID
###############################
```
col sql_text format a100 heading "Current SQL"
select q.sql_text
from v$session s
, v$sql q
WHERE s.sql_address = q.address
and s.sql_hash_value + DECODE
(SIGN(s.sql_hash_value), -1, POWER( 2, 32), 0) = q.hash_value
AND s.sid=&1;
```
============================================
### SQL text for locked session
=============================================
```
set pagesize 60 
set linesize 132 
select s.username username,  
       a.sid sid,  
       a.owner||'.'||a.object object,  
       s.lockwait,  
       t.sql_text SQL 
from   v$sqltext t,  
       v$session s,  
       v$access a 
where  t.address=s.sql_address  
and    t.hash_value=s.sql_hash_value  
and    s.sid = a.sid  
and    a.owner != 'SYS' 
and    upper(substr(a.object,1,2)) != 'V$' 
/ 
```
```
select s1.username as blocker,s1.machine b_machine,
s2.username h_username,s2.machine h_machine,s3.sql_text
from v$lock l1, v$session s1, v$lock l2, v$session s2,v$sqltext s3
where s1.sid=l1.sid and s2.sid=l2.sid
and l1.BLOCK=1 and l2.request > 0
and l1.id1 = l2.id1
and l2.id2 = l2.id2
AND s1.PREV_SQL_ADDR =s3.address
and s1.PREV_HASH_VALUE=s3.HASH_VALUE;
```

### TO SEE IF STILL PENDING TRANSACTIONS 
```
SQL> Select a.username,a.Machine,a.Process,a.Logon_time,a.sid,a.status,b.used_ublk
from v$session a , v$transaction b
                        where username = '&User'
                        and a.saddr=b.ses_addr;
```


In 11g R2, the blocking session can be found directly using the following sql: 
SQL> select sid,serial#,SQL_ID,BLOCKING_SESSION,BLOCKING_SESSION_STATUS,EVENT 
from v$session where event ='cursor: pin S wait on X' 

### Hanged Session
```
select p.spid,s.sid,s.serial# serial_num,s.username user_name,
a.type object_type,s.osuser os_user_name,a.owner,a.object object_name,decode(sign(48 - command),1,
to_char(command), 'Action Code #' || to_char(command) ) action,
p.program oracle_process,s.terminal terminal,s.program program,s.status session_status 
from v$session s, v$access a, v$process p 
where s.paddr = p.addr and s.type = 'USER' 
and a.sid = s.sid 
and a.object='&table_name' 
order by s.username, s.osuser;
```
Suppose session hangs during execute the procedure.
To pass the procedure name in below query.
```
select
V$S.LOGON_TIME,
V$S.SID,
V$S.SERIAL#,
V$S.USERNAME,
V$S.PROCESS,
V$P.SPID,
V$S.STATUS,
V$S.MACHINE,
V$S.CLIENT_INFO
from v$session V$S,
v$process V$P
where V$S.PADDR = V$P.ADDR
and V$S.SID in
(select distinct sid from v$access where object like ('%&obj%'))
order by
V$S.PROCESS,
V$S.SID;
```
```
Column host     format a6;
Column username format a10;
Column os_user  format a8;
Column program  format a30;
Column tsname   format a12;
 
select
   b.machine         host,
   b.username        username,
   b.server,
   b.osuser          os_user,
   b.program         program,
   a.tablespace_name ts_name,
   row_wait_file#    file_nbr,
   row_wait_block#   block_nbr,
   c.owner,
   c.segment_name,
   c.segment_type
from
   dba_data_files a,
   v$session      b,
   dba_extents    c
where b.row_wait_file# = a.file_id
and c.file_id = row_wait_file#
and row_wait_block#  between c.block_id and c.block_id + c.blocks - 1
and row_wait_file# <> 0
and type='USER'
;
```
The username will tell you the user who is waiting for a lock to be released at the block number. 

### To find out the what is blocking use the code snippet below, 
```
Select blocking_session, sid, serial#, wait_class,seconds_in_wait From v$session  
where blocking_session is not NULL
order by blocking_session;
```
### HOW LONG INACTIVE
```
SQL> set lines 100 pages 999
select username
,      floor(last_call_et / 60) "Minutes"
,      status
from   v$session;
```
```
SQL> 
select username, floor(last_call_et / 60) "Minutes", status
from   v$session
where SID=393
order by last_call_et;
```
```
SQL> alter session set nls_date_format = 'YYYY-MM-DD:HH24:MI:SS';
SQL> select to_char(sysdate - last_call_et / 86400,'HH24:MI') from v$session where SID=393;
```
The column last_call_et in v$session tells how long the inactive sessions were inactive.
LAST_CALL_ET is the amount of time in seconds since the session's last statement execution. 

### Get the locked row ROWID
```
select do.object_name, row_wait_obj#, row_wait_file#, row_wait_block#, row_wait_row#, dbms_rowid.rowid_create ( 1, ROW_WAIT_OBJ#, ROW_WAIT_FILE#, ROW_WAIT_BLOCK#, ROW_WAIT_ROW# )
from v$session s, dba_objects do
where sid=543
and s.ROW_WAIT_OBJ# = do.OBJECT_ID ;
```

### ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired

The most common reason for this are either 'SELECT FOR UPDATE ' or some uncommitted INSERT statements.

``` SQL> SELECT O.OBJECT_NAME, S.SID, S.SERIAL#, P.SPID, S.PROGRAM,SQ.SQL_FULLTEXT, S.LOGON_TIME FROM V$LOCKED_OBJECT L, DBA_OBJECTS O, V$SESSION S, V$PROCESS P, V$SQL SQ WHERE L.OBJECT_ID = O.OBJECT_ID AND L.SESSION_ID = S.SID AND S.PADDR = P.ADDR AND S.SQL_ADDRESS = SQ.ADDRESS;
```
```Sample Output
OBJECT_NAME
------------------------------------------------------------------------------------------------------------------------------------------------------
       SID    SERIAL# SPID
---------- ---------- ------------------------------------------------------------------------
PROGRAM
------------------------------------------------------------------------------------------------------------------------------------------------
SQL_FULLTEXT                                                                     LOGON_TIME
-------------------------------------------------------------------------------- -------------------
PNIA_STAGING_MVMNT_ORE_GRADE
        44      18919 93768
oracle@iorsdb02-adm.apac.ent.bhpbilliton.net (TN
INSERT  INTO "PNIA_STAGING_MVMNT_CONTRIB" "A1" ("STAGING_RECORD_ID","STAGING_R   18-02-2016:15:10:22
EC

PNIA_STAGING_MVMNT_QUANTITY
        44      18919 93768
oracle@iorsdb02-adm.apac.ent.bhpbilliton.net (TN
INSERT  INTO "PNIA_STAGING_MVMNT_CONTRIB" "A1" ("STAGING_RECORD_ID","STAGING_R   18-02-2016:15:10:22
EC

PNIA_STAGING_MVMNT
        44      18919 93768
oracle@iorsdb02-adm.apac.ent.bhpbilliton.net (TN
INSERT  INTO "PNIA_STAGING_MVMNT_CONTRIB" "A1" ("STAGING_RECORD_ID","STAGING_R   18-02-2016:15:10:22
EC

PNIA_STAGING_MVMNT_PHYS_PROP
        44      18919 93768
oracle@iorsdb02-adm.apac.ent.bhpbilliton.net (TN
INSERT  INTO "PNIA_STAGING_MVMNT_CONTRIB" "A1" ("STAGING_RECORD_ID","STAGING_R   18-02-2016:15:10:22
EC

PNIA_STAGING_MVMNT_CONTRIB
        44      18919 93768
oracle@iorsdb02-adm.apac.ent.bhpbilliton.net (TN
INSERT  INTO "PNIA_STAGING_MVMNT_CONTRIB" "A1" ("STAGING_RECORD_ID","STAGING_R   18-02-2016:15:10:22
EC
```

### SOLUTION:
```
We have 3 options to fix this error
      1.  Kill the DB session and get the tables unlocked
      2. Kill the application which holds this particular session(sql connection)
      3. The ideal solution is to get to the actual process(application) to debug/fix the issue

1. ==> Kill session 44 from client connection
cleared
  alter system kill session '953,40807'

2. Get the column value of MACHINE from the table V$SESSION and search for the running processes in that machine. 

SELECT O.OBJECT_NAME, S.SID, S.SERIAL#, P.SPID, S.PROGRAM,S.USERNAME,S.MACHINE,S.PORT ,S.LOGON_TIME,SQ.SQL_FULLTEXT FROM V$LOCKED_OBJECT L, DBA_OBJECTS O, V$SESSION S, V$PROCESS P, V$SQL SQ WHERE L.OBJECT_ID = O.OBJECT_ID AND L.SESSION_ID = S.SID AND S.PADDR = P.ADDR AND S.SQL_ADDRESS = SQ.ADDRESS; 

LOGON_TIME 
[bash]$ ps aux | grep java
kill -15 <PID>

3. Getting to the Actual Process on the application server

The better way to find the process is to get the V$SESSION.PORT from the above sql  and find the process listening on that port.

SELECT O.OBJECT_NAME, S.SID, S.SERIAL#, P.SPID, S.PROGRAM,S.USERNAME,S.MACHINE,S.PORT ,S.LOGON_TIME,SQ.SQL_FULLTEXT FROM V$LOCKED_OBJECT L, DBA_OBJECTS O, V$SESSION S, V$PROCESS P, V$SQL SQ WHERE L.OBJECT_ID = O.OBJECT_ID AND L.SESSION_ID = S.SID AND S.PADDR = P.ADDR AND S.SQL_ADDRESS = SQ.ADDRESS;

To get to the process listening on the port, execute the below command.For me, the port number was  xxxxx 

netstat -ap | grep xxxxx

which gave me

tcp        0      0 machine.name:34465  oracle.db.com:ncube-lm ESTABLISHED 4030/java

We got the process id xxxx. Debug or Kill; 
```
