### JOB STATUS
```
set lines 200
set pages 60
col job format 99999
col priv_user format a12
col log_user format a12
col FAILURES format 99999
col what format a75
select JOB, LOG_USER, PRIV_USER, LAST_DATE, NEXT_DATE, BROKEN, FAILURES, WHAT 
      from dba_jobs
/

select  failures, log_user, last_date,this_date, job, what
from dba_jobs
where failures <> 0;

select what, broken, next_date, interval, job
from dba_jobs
where broken='Y';
```

###  scheduled tasks failed during execution, and why?
```
COL log_id              FORMAT 9999   HEADING 'Log#'
COL log_date            FORMAT A32    HEADING 'Log Date'
COL owner               FORMAT A06    HEADING 'Owner'
COL job_name            FORMAT A20    HEADING 'Job'
COL status              FORMAT A10    HEADING 'Status'
COL actual_start_date   FORMAT A32    HEADING 'Actual|Start|Date'
COL error#              FORMAT 999999 HEADING 'Error|Nbr'

TTITLE 'Scheduled Tasks That Failed:'
SELECT
     log_id
    ,log_date
    ,owner
    ,job_name
    ,status
    ,actual_start_date
    ,error#
  FROM dba_scheduler_job_run_details
 WHERE status <> 'SUCCEEDED'
 ORDER BY actual_start_date;

```
### ===============11g=======================
```
SELECT j.LOG_ID, j.LOG_DATE, e.OWNER,
DECODE(instr(e.NAME,'"'),0, e.NAME,substr(e.NAME,1,instr(e.NAME,'"')-1)),
DECODE(instr(e.NAME,'"'),0,NULL,substr(e.NAME,instr(e.NAME,'"')+1)),
e.STATUS, j.ERROR#, j.REQ_START_DATE, j.START_DATE, j.RUN_DURATION,
j.INSTANCE_ID, j.SESSION_ID, j.SLAVE_PID, j.CPU_USED, e.DESTINATION,
j.ADDITIONAL_INFO
FROM scheduler$_job_run_details j, scheduler$_event_log e
WHERE j.log_id = e.log_id and e.dbid is null
AND e.type# = 66
```
The SCHEDULER$_EVENT_LOG table is the underlying table for dba_scheduler_job_log and dba_scheduler_window_log views .

The SCHEDULER$_JOB_RUN_DETAILS table is the underlying table for *_scheduler_job_run_details views.

The job_log views give an overview of the job history (if logging is on). The job_run_details views give more detailed information about individual runs of a job (cpu_used etc) again if logging is on.

Before purge :
``
select * from all_scheduler_running_chains; --- should be zero
``
#### RUN THE JOB:
##########################################
```
exec dbms_ijob.run ( 7 );
```
#### Create The JOB
#################################################
```
Declare
job_id number;
begin
DBMS_JOB.SUBMIT (job_id,
what=> dbms_stats.gather_schema_stats(ownname => 'ELLIPSE' , estimate_percent => 10 , cascade => true);,
next_date=> to_date('21-Feb-2009 02:10:00 PM','dd-Mon-YYYY hh:mi:ss PM'),
interval=> 'sysdate + 7 '
);
end;
/
```

#### Viewing scheduled dbms_jobs
##################################################
scheduled_dbms_jobs.sql
```
set linesize 250
col log_user       for a10
col job            for 9999999  head 'Job'
col broken         for a1       head 'B'
col failures       for 99       head "fail"
col last_date      for a18      head 'Last|Date'
col this_date      for a18      head 'This|Date'
col next_date      for a18      head 'Next|Date'
col interval       for 9999.000 head 'Run|Interval'
col what           for a60

select j.log_user,
     j.job,
     j.broken,
     j.failures,
     j.last_date||':'||j.last_sec last_date,
     j.this_date||':'||j.this_sec this_date,
     j.next_date||':'||j.next_sec next_date,
     j.next_date - j.last_date interval,
     j.what
from (select dj.LOG_USER, dj.JOB, dj.BROKEN, dj.FAILURES, 
             dj.LAST_DATE, dj.LAST_SEC, dj.THIS_DATE, dj.THIS_SEC, 
             dj.NEXT_DATE, dj.NEXT_SEC, dj.INTERVAL, dj.WHAT
        from dba_jobs dj) j;

```
#### What Jobs are Actually Running
##################################################

```
set linesize 250
col sid            for 9999     head 'Session|ID'
col log_user       for a10
col job            for 9999999  head 'Job'
col broken         for a1       head 'B'
col failures       for 99       head "fail"
col last_date      for a18      head 'Last|Date'
col this_date      for a18      head 'This|Date'
col next_date      for a18      head 'Next|Date'
col interval       for 9999.000 head 'Run|Interval'
col what           for a60
select j.sid,
       j.log_user,
       j.job,
       j.broken,
       j.failures,
       j.last_date||':'||j.last_sec last_date,
       j.this_date||':'||j.this_sec this_date,
       j.next_date||':'||j.next_sec next_date,
       j.next_date - j.last_date interval,
       j.what
from (select djr.SID, 
             dj.LOG_USER, dj.JOB, dj.BROKEN, dj.FAILURES, 
             dj.LAST_DATE, dj.LAST_SEC, dj.THIS_DATE, dj.THIS_SEC, 
             dj.NEXT_DATE, dj.NEXT_SEC, dj.INTERVAL, dj.WHAT
        from dba_jobs dj, dba_jobs_running djr
       where dj.job = djr.job ) j;
```

#### What Sessions are Running the Jobs
##################################################
```
set linesize 250
col sid            for 9999     head 'Session|ID'
col spid                        head 'O/S|Process|ID'
col serial#        for 9999999  head 'Session|Serial#'
col log_user       for a10
col job            for 9999999  head 'Job'
col broken         for a1       head 'B'
col failures       for 99       head "fail"
col last_date      for a18      head 'Last|Date'
col this_date      for a18      head 'This|Date'
col next_date      for a18      head 'Next|Date'
col interval       for 9999.000 head 'Run|Interval'
col what           for a60
select j.sid,
s.spid,
s.serial#,
       j.log_user,
       j.job,
       j.broken,
       j.failures,
       j.last_date||':'||j.last_sec last_date,
       j.this_date||':'||j.this_sec this_date,
       j.next_date||':'||j.next_sec next_date,
       j.next_date - j.last_date interval,
       j.what
from (select djr.SID, 
             dj.LOG_USER, dj.JOB, dj.BROKEN, dj.FAILURES, 
             dj.LAST_DATE, dj.LAST_SEC, dj.THIS_DATE, dj.THIS_SEC, 
             dj.NEXT_DATE, dj.NEXT_SEC, dj.INTERVAL, dj.WHAT
        from dba_jobs dj, dba_jobs_running djr
       where dj.job = djr.job ) j,
     (select p.spid, s.sid, s.serial#
          from v$process p, v$session s
         where p.addr  = s.paddr ) s
 where j.sid = s.sid;
```
#### Change job schedule
```
exec dbms_job.change(42,'SUMM_REFRESH.PROC_REFRESH;',trunc(sysdate+1) + 5.5/24,'trunc(sysdate+1) +5.5/24');
```

### Check long running queries :
-----------------------------
```
col sql_text format a100
set linesize 400
SELECT l.sid, l.start_time, l.username, l.elapsed_seconds, a.sql_text, a.elapsed_time
FROM v$session_longops l, v$sqlarea a
WHERE a.elapsed_time = l. elapsed_seconds
AND l.elapsed_seconds > 1
/
```
### Check long running process for particular session  and sid :
```
select * from (  select opname, target, sofar, totalwork,units, elapsed_seconds, message  
		from v$session_longops  
		where sid = <sid> and serial# = <serial#>  
		order by start_time desc)
where rownum <=1;
```

### Check the time remaining :
```
SELECT opname, target,
ROUND((sofar/totalwork),4)*100 Percentage_Complete,
start_time,
CEIL(time_remaining/60) Max_Time_Remaining_In_Min,
FLOOR(elapsed_seconds/60) Time_Spent_In_Min
FROM v$session_longops
WHERE sofar != totalwork;
```
#### DEFAULT JOBS
```
SQL> select log_date,status
from dba_scheduler_job_run_details
where job_name='BSLN_MAINTAIN_STATS_JOB'
order by log_date desc;

SQL> select owner, job_name, enabled, LAST_START_DATE, NEXT_RUN_DATE from DBA_SCHEDULER_JOBS WHERE job_name like '%PURGE%';

SQL> exec dbms_scheduler.disable('CDC_STG_PUB.CDC$_DEFAULT_PURGE_JOB');
```

### ENABLE:
---------------------------
```
BEGIN
  DBMS_AUTO_TASK_ADMIN.ENABLE(
    client_name => 'auto optimizer stats collection',
    operation => NULL,
    window_name => NULL);
END;
/
```
### DISABLE:
----------------------------------
```
BEGIN
  DBMS_AUTO_TASK_ADMIN.DISABLE(
    client_name => 'auto optimizer stats collection', 
    operation => NULL, 
    window_name => NULL);
END;
/
```
```
SELECT client_name, window_name, jobs_created, jobs_started, jobs_completed FROM dba_autotask_client_history WHERE client_name like '%stats%';

SELECT client_name, status FROM dba_autotask_operation;
```
The tasks that run for these autotask ‘clients’ are named as follows:

ORA$AT_SA_SPC_SY_nnn for Space advisor tasks
ORA$AT_OS_OPT_SY_nnn for Optimiser stats collection tasks
ORA$AT_SQ_SQL_SW_nnn for Space advisor tasks

```
set linesize 300
col JOB_DURATION format a13
col JOB_ERROR format 999
col JOB_INFO format a25 wrap

select JOB_NAME, JOB_STATUS, JOB_START_TIME, JOB_DURATION, JOB_ERROR, JOB_INFO
from DBA_AUTOTASK_JOB_HISTORY
where client_name='auto space advisor'
order by window_start_time desc;

select distinct client_name, window_name, job_status, job_info
from dba_autotask_job_history
where client_name ='auto space advisor '
order by 1,2;

SELECT * FROM dba_autotask_schedule;
```
disable all the automated tasks, issue the following command::

SQL> EXEC dbms_auto_task_admin.disable;
SQL> SELECT client_name, status FROM dba_autotask_operation;
SQL> EXEC dbms_auto_task_admin.disable( 'auto optimizer stats collection', NULL, NULL );

### BROKEN JOBS
```
select 'exec dbms_job.broken(job =>'||job||', next_date=>'||interval||', broken =>false)' 
from dba_jobs where schema_user ='PRODOWNER' and broken='Y';
```
```
select 'exec dbms_job.run(job =>'||job||')' from dba_jobs where schema_user ='REPORTS' and broken='Y';
```
```
set echo off verify off feedback off head off pagesize 0
select 'execute dbms_job.broken(job=>' || job || ', broken=>FALSE);' 
from dba_jobs
where broken = 'Y'
and schema_user = 'USERNAME'
/
```
> exec dbms_ijob.broken(job =>121, next_date=>SYSDATE + 1/24, broken =>false)

### Create an Oracle job to run a stored procedure every hour
If you need to run an Oracle Stored Procedure every hour you can either setup a job via the Cron tab or you can create a an Oracle Job to do this for you.
Here is a simple example, you can use the different parameters to cater for your needs:

```
BEGIN
SYS.DBMS_JOB.SUBMIT
( job => X
,what => 'MYDB.GET_BALANCES;'
,next_date => to_date('29/06/2011 14:19:08','dd/mm/yyyy hh24:mi:ss')
,interval => 'sysdate+1/24 '
,no_parse => FALSE
);
SYS.DBMS_OUTPUT.PUT_LINE('Job Number is: ' || to_char(x));
COMMIT;
END;
/
```

### Procedure:
```
CREATE OR REPLACE PROCEDURE Schema.proc_update_message_flow
IS
cursor flow_cur is
SELECT source,target,business_event,segment,service,last_24hour_count,last_hour_count FROM "Message_Volume_View1";
flow_rec flow_cur%rowtype;
BEGIN
for flow_rec in flow_cur
LOOP
update Message_data_flow set FLOW_LAST_HOUR = flow_rec.last_hour_count where source = flow_rec.source and target = flow_rec.target and business_event = flow_rec.business_event and segment = flow_rec.segment and service = flow_rec.service;
update Message_data_flow set FLOW_LAST_24HOUR = flow_rec.last_24hour_count where source = flow_rec.source and target = flow_rec.target and business_event = flow_rec.business_event and segment = flow_rec.segment and service = flow_rec.service;
commit;
end LOOP;
END proc_update_message_flow;
/
```

### JOB
```
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
   job_name        =>  'UpdateFlow',
   job_type        =>  'STORED_PROCEDURE',
   job_action      =>  'ICCSYSDB.proc_update_message_flow',
   start_date      => to_date('18/03/2016 13:30:00', 'dd/mm/yyyy hh24:mi:ss'),
   repeat_interval => 'FREQ=MINUTELY;INTERVAL=5',
   enabled         => true);
END;
/
```
```
BEGIN 
--remove job
SYS.DBMS_SCHEDULER.DROP_JOB (job_name => 'BusinessSum');
END; 
/
```

### PARTITION:

```
CREATE OR REPLACE PROCEDURE ICCSYSDB."DELETE_OLD_PARTITION" (
    TABLE_NAME IN VARCHAR,
    DAYS_OLD IN NUMBER,
    STATUS OUT VARCHAR2)
IS

v_sql              varchar2(500);
psql              varchar2(500);
v_date            date;
v_partition_name  user_tab_partitions.partition_name%TYPE;
v_high_value      user_tab_partitions.high_value%TYPE;
c1 sys_refcursor;

BEGIN

--DBMS_OUTPUT.PUT_LINE ('TABLE_NAME : ' || TABLE_NAME ); 
--DBMS_OUTPUT.PUT_LINE ('DAYS_OLD : ' || DAYS_OLD );  
psql  := 'select PARTITION_NAME, HIGH_VALUE  from user_tab_partitions where table_name = ''' ||  TABLE_NAME || ''' and  PARTITION_NAME != ''P0'' ' ;
 
OPEN  c1 for psql;
  LOOP
    FETCH c1 INTO v_partition_name, v_high_value;
   -- DBMS_OUTPUT.PUT_LINE ('v_partition_name  : ' || v_partition_name );
    EXIT WHEN c1%NOTFOUND;
    v_date := to_date(SUBSTR(v_high_value,11,19),'YYYY-MM-DD HH24:MI:SS');
   IF v_date < (sysdate - DAYS_OLD ) then
      v_sql := 'alter table ' || TABLE_NAME || ' drop  partition ' || v_partition_name || ' update indexes' ;
      execute immediate v_sql;
      --DBMS_OUTPUT.PUT_LINE ('v_sql  : ' || v_sql );
    END IF;
  END LOOP;
  CLOSE c1;
  STATUS := 'Success' ; 
  --DBMS_OUTPUT.PUT_LINE ('executed  : ' || STATUS );
END;
/
```

### ---> another procedure to call the DELETE_OLD_PARTITION
```
CREATE OR REPLACE PROCEDURE ICCSYSDB."DELETE_OLD_EVENT_DATA" 
IS
status_msg              varchar(10);
status_payload          varchar(10);
BEGIN
 DELETE_OLD_PARTITION ('WMB_MSGS',6,status_msg);
 DELETE_OLD_PARTITION ('WMB_BINARY_DATA',6,status_payload);
 DBMS_OUTPUT.PUT_LINE ('executed STATUS_MSG : ' || status_msg );
 DBMS_OUTPUT.PUT_LINE ('executed STATUS_BINARY_DATA : ' || status_payload );
END;
/
```
```
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
   job_name        =>  'Drop_Old_Partition',
   job_type        =>  'STORED_PROCEDURE',
   job_action      =>  'ICCSYSDB.DELETE_OLD_EVENT_DATA',
   start_date      => to_date('27/05/2016 13:30:00', 'dd/mm/yyyy hh24:mi:ss'),
   repeat_interval => 'FREQ=WEEKLY;',
   enabled         => true);
END;
/

```
### Long Running Jobs

```
--------------------------------
--
--  List long operations for RAC.
--
 
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 99
COLUMN sid FORMAT 9999
COLUMN serial# FORMAT 99999
COLUMN username FORMAT A24
COLUMN module FORMAT A40
COLUMN progress_pct FORMAT 999
COLUMN elapsed FORMAT A10
COLUMN remaining FORMAT A10
 
SELECT s.inst_id,
       s.sid,
       s.serial#,
       s.sql_address,
       s.username,
       s.module,
       ROUND(sl.elapsed_seconds/60) || ':' || MOD(sl.elapsed_seconds,60) elapsed,
       ROUND(sl.time_remaining/60) || ':' || MOD(sl.time_remaining,60) remaining,
       ROUND(sl.sofar/(sl.totalwork+1)*100, 2) progress_pct
FROM   gv$session s,
       gv$session_longops sl
WHERE  s.sid     = sl.sid
AND    s.inst_id = sl.inst_id
AND    s.serial# = sl.serial#
ORDER BY progress_pct
/
```
```
set lines 132
set pages 60
col opname format a35
SELECT opname
, sid
, serial#
, context
, sofar
, totalwork
, decode(totalwork,0,0
         ,round(sofar/totalwork*100,2)) "% Complete"
, time_remaining / 60 "Minutes Remaining"
FROM v$session_longops
WHERE 
  totalwork != 0
  and sofar <> totalwork
/
```
```
set lines 200
col OPNAME for a25
Select
a.sid,
a.serial#,
b.status,
a.opname,
to_char(a.START_TIME,' dd-Mon-YYYY HH24:mi:ss') START_TIME,
to_char(a.LAST_UPDATE_TIME,' dd-Mon-YYYY HH24:mi:ss') LAST_UPDATE_TIME,
a.time_remaining as "Time Remaining Sec" ,
a.time_remaining/60 as "Time Remaining Min",
a.time_remaining/60/60 as "Time Remaining HR"
From v$session_longops a, v$session b
where a.sid = b.sid
and a.sid =&sid
And time_remaining > 0;
```
OR
```
set pages 50000 lines 32767
col OPNAME for a10
col SID form 9999
col SERIAL form 9999999
col PROGRAM for a10
col USERNAME for a10
col SQL_TEXT for a40
col START_TIME for a10
col LAST_UPDATE_TIME for a10
col TARGET for a25
col MESSAGE for a25

alter session set nls_date_format = 'DD-MM-YYYY HH24:MI:SS';

SELECT l.inst_id,l.sid, l.serial#, l.sql_id, l.opname, l.username, l.target, l.sofar, l.totalwork, l.start_time,l.last_update_time,round(l.time_remaining/60,2) "REMAIN MINS", round(l.elapsed_seconds/60,2) "ELAPSED MINS", round((l.time_remaining+l.elapsed_seconds)/60,2) "TOTAL MINS", ROUND(l.SOFAR/l.TOTALWORK*100,2) "%_COMPLETE", l.message,s.sql_text 
FROM gv$session_longops l 
LEFT OUTER JOIN v$sql s on s.hash_value=l.sql_hash_value and s.address=l.sql_address and s.child_number=0
WHERE l.OPNAME NOT LIKE 'RMAN%' AND l.OPNAME NOT LIKE '%aggregate%' AND l.TOTALWORK != 0 AND l.sofar<>l.totalwork AND l.time_remaining > 0
/
```
```
column  sid         format   a10    heading 'Sid/Serial'
column  operation   format   a11    heading 'Acao'
column  object      format   a35    heading 'Objeto'
column  executado   format   99999999 heading 'BLK|Lidos'
column  total       format   99999999 heading 'BLK|Total'
column  pct         format   990.90   heading 'PCT(%)'
column  rate        format   a08      heading 'Rate|Mb/Min'
column  l_update    format   a11    heading 'Dt Start'
column  t_elap      format   a11 heading 'Elapsed|DD:HH:MI:SS'
column  t_remain    format   a11 heading 'Remaining|DD:HH:MI:SS'

column  unidade     format   a3     heading 'Uni'

accept SesID prompt 'Informe Sid: '

select to_char(sid) || ',' || ltrim(to_char(serial#)) sid,
       decode(opname,'Hash Join',    'Hash Join',
              'Index Fast Full Scan','Index Scan',
              'Sort Output',         'Sort Output',
              'Sort/Merge',          'Sort Merge',
              'Table Scan',          'Table Scan',
              '-') operation,
       target object,
       sofar executado,
       totalwork total,
       trunc((sofar/totalwork)*100,2) pct,
       to_char(60*sofar*8192/(24*60*(last_update_time - start_time))/1024/1024/60, '9990.90') Rate,
--       decode(units,'Blocks','Blk','-') unidade,
       to_char(start_time, 'DD/MM HH24:MI') l_update,
       trunc(elapsed_seconds/86400)|| ':'
            || to_char(to_date(mod(elapsed_seconds,86400), 'SSSSS'), 'HH24:MI:SS') t_elap,
       trunc(time_remaining/86400)|| ':'
            || to_char(to_date(mod(time_remaining,86400), 'SSSSS'), 'HH24:MI:SS') t_remain
from v$session_longops
where time_remaining > 0
  and sid like nvl('&sesid','%')
;
```
