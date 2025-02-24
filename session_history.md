## Most active session in last one hour can be found using active session history
```
SELECT sql_id,COUNT(*),ROUND(COUNT(*)/SUM(COUNT(*)) OVER(), 2) PCTLOAD
FROM gv$active_session_history
WHERE sample_time > SYSDATE - 1/24
AND session_type = 'BACKGROUND'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
### Get the wait event for the above session
```
SELECT sample_time, event, wait_time
FROM gv$active_session_history
WHERE session_id = &1
AND session_serial# = &2
```
## Most I/O intensive sql in last 1 hour
```
SELECT sql_id, COUNT(*)
FROM gv$active_session_history ash, gv$event_name evt
WHERE ash.sample_time > SYSDATE - 1/24
AND ash.session_state = 'WAITING'
AND ash.event_id = evt.event_id
AND evt.wait_class = 'User I/O'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
## Top sqls spent more on cpu/wait/io
```
select
ash.SQL_ID ,
sum(decode(a.session_state,'ON CPU',1,0)) "CPU",
sum(decode(a.session_state,'WAITING',1,0)) -
sum(decode(a.session_state,'WAITING', decode(en.wait_class, 'User I/O',1,0),0)) "WAIT" ,
sum(decode(a.session_state,'WAITING', decode(en.wait_class, 'User I/O',1,0),0)) "IO" ,
sum(decode(a.session_state,'ON CPU',1,1)) "TOTAL"
from v$active_session_history a,v$event_name en
where SQL_ID is not NULL and en.event#=ash.event#
```
##  A particular session sql analysis
```
SELECT C.SQL_TEXT,
B.NAME,
COUNT(*),
SUM(TIME_WAITED)
FROM v$ACTIVE_SESSION_HISTORY A,
v$EVENT_NAME B,
v$SQLAREA C
WHERE A.SAMPLE_TIME BETWEEN '&starttime' AND
'&endtime' AND
A.EVENT# = B.EVENT# AND
A.SESSION_ID= &sid AND
A.SQL_ID = C.SQL_ID
GROUP BY C.SQL_TEXT, B.NAME
```
## Top session on CPU in last 15 minute
```
SELECT * FROM
(
SELECT s.username, s.module, s.sid, s.serial#, s.sql_id,count(*)
FROM v$active_session_history h, v$session s
WHERE h.session_id = s.sid
AND h.session_serial# = s.serial#
AND session_state= 'ON CPU' AND
sample_time > sysdate - interval '15' minute
GROUP BY s.username, s.module, s.sid, s.serial#,s.sql_id
ORDER BY count(*) desc
)
where rownum <= 10;
```
## Find queries executed in the last 30 days.
```
SELECT
   h.sample_time,
   u.username,
   h.program,
   h.module,
   s.sql_text
FROM
   DBA_HIST_ACTIVE_SESS_HISTORY h,
   DBA_USERS u,
   DBA_HIST_SQLTEXT s
WHERE  sample_time between to_date('01/10/2024 19:45:00','DD/MM/YYYY HH24:MI:SS') 
and to_date('01/10/2024 19:50:00','DD/MM/YYYY HH24:MI:SS')
   AND h.user_id=u.user_id
AND h.sql_id = s.sql_iD
ORDER BY h.sample_time;
```
```
SELECT
   h.sample_time,
   u.username,
   h.program,
   h.module,
   s.sql_text
FROM
   DBA_HIST_ACTIVE_SESS_HISTORY h,
   DBA_USERS u,
   DBA_HIST_SQLTEXT s
WHERE  WHERE sample_time >= SYSDATE - 30
AND h.user_id=u.user_id
AND h.sql_id = s.sql_iD
ORDER BY h.sample_time;
```
```
set feedback on
set long 10000
set pagesize 132
set linesize 110
col username form a10
col program form a30
col how_many forma 9999
col machine form a25
col module form a50
--
rem show time
select to_char(sysdate, 'DD-MON-RR HH24:MI:SS') START_TIME from dual
/
rem Instance identification
select *
from v$instance
/
REM Summary of database connections
select s.module, s.machine, s.username, count(*) how_many
from (select distinct PROGRAM, PADDR, machine, username, module, inst_id from gV$SESSION) s,
 gv$process p
where s.paddr = p.addr
and p.inst_id = s.inst_id
group by rollup(s.module,s.machine,s.username)
/
REM Connections by machine and instance
select s.machine, s.username, s.module, s.inst_id, count(*) how_many
from (select distinct PROGRAM, PADDR, machine, username, module, inst_id from gV$SESSION) s,
 gv$process p
where s.paddr = p.addr
and p.inst_id = s.inst_id
group by s.machine,s.username, s.module, s.inst_id
/
```

Current Running SQLs
--------------------
```
set pages 50000 lines 32767
col HOST_NAME for a20
col EVENT for a40
col MACHINE for a30
col SQL_TEXT for a50
col USERNAME for a15

select sid,serial#,a.sql_id,a.SQL_TEXT,S.USERNAME,i.host_name,machine,S.event,S.seconds_in_wait sec_wait,
to_char(logon_time,'DD-MON-RR HH24:MI') login
from gv$session S,gV$SQLAREA A,gv$instance i
where S.username is not null
-- and S.status='ACTIVE'
AND S.sql_address=A.address
and s.inst_id=a.inst_id and i.inst_id = a.inst_id
and sql_text not like 'select S.USERNAME,S.seconds_in_wait%'
```

Active Sessions running for more than 1 hour
---------------------------------------------
```
set pages 50000 lines 32767
col USERNAME for a10
col MACHINE for a15
col PROGRAM for a40

SELECT USERNAME,machine,inst_id,sid,serial#,PROGRAM,
to_char(logon_time,'dd-mm-yy hh:mi:ss AM')"Logon Time",
ROUND((SYSDATE-LOGON_TIME)*(24*60),1) as MINUTES_LOGGED_ON,
ROUND(LAST_CALL_ET/60,1) as Minutes_FOR_CURRENT_SQL
From gv$session
WHERE STATUS='ACTIVE'
AND USERNAME IS NOT NULL and ROUND((SYSDATE-LOGON_TIME)*(24*60),1) > 60
ORDER BY MINUTES_LOGGED_ON DESC;

set pages 50000 lines 32767
col USERNAME for a10
col MACHINE for a15
col PROGRAM for a40

SELECT USERNAME,machine,PROGRAM, TERMINAL, 
to_char(logon_time,'dd-mm-yy hh:mi:ss AM')"Logon Time",
ROUND((SYSDATE-LOGON_TIME)*(24*60),1) as MINUTES_LOGGED_ON,
ROUND(LAST_CALL_ET/60,1) as Minutes_FOR_CURRENT_SQL
From v$session
WHERE STATUS='ACTIVE'
AND USERNAME IS NOT NULL and ROUND((SYSDATE-LOGON_TIME)*(24*60),1) > 30
ORDER BY MINUTES_LOGGED_ON DESC;
```
Session details associated with Oracle SID
-------------------------------------------

```
set head off
set verify off
set echo off
set pages 1500
set linesize 100
set lines 120
prompt
prompt Details of SID / SPID / Client PID
prompt ==================================
select /*+ CHOOSE*/
'Session Id.............................................: '||s.sid,
'Serial Num..............................................: '||s.serial#,
'User Name ..............................................: '||s.username,
'Session Status .........................................: '||s.status,
'Client Process Id on Client Machine ....................: '||'*'||s.process||'*' Client,
'Server Process ID ......................................: '||p.spid Server,
'Sql_Address ............................................: '||s.sql_address,
'Sql_hash_value .........................................: '||s.sql_hash_value,
'Schema Name ..... ......................................: '||s.SCHEMANAME,
'Program ...............................................: '||s.program,
'Module .................................................: '|| s.module,
'Action .................................................: '||s.action,
'Terminal ...............................................: '||s.terminal,
'Client Machine .........................................: '||s.machine,
'LAST_CALL_ET ...........................................: '||s.last_call_et,
'S.LAST_CALL_ET/3600 ....................................: '||s.last_call_et/3600
from v$session s, v$process p
where p.addr=s.paddr and
s.sid=nvl('&sid',s.sid) 
/
```
