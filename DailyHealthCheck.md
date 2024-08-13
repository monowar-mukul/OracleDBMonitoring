```diff
- NOTE
! Make sure any Licence requirements from your side. Please do modify based on your own setup. This is purely based on my own lab setup. 

```

## connect to a database using host, port, SID/Service name without having an entry in tnsnames.ora?
```
$ sqlplus username/password@hostname:port/SERVICENAME
OR
$ sqlplus username
Enter password: password@//hostname:port/SERVICENAME
OR
$ sqlplus /nolog
SQL> connect username/password@hostname:port/SERVICENAME
```
### monitor_report_sessions.sql
```
set head off
set linesize 200 trimspool on pagesize 0

SELECT   s.status ||','||s.serial#||','||s.TYPE||','||
         s.username||','||s.osuser||','||
         s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action
    FROM v$session s, v$process p, SYS.v_$sess_io si
   WHERE s.paddr = p.addr(+) AND si.SID(+) = s.SID;
```
```
SELECT 'Sid,Serial#,User,Temp(Mb),Client User,Machine,Module,Client Info, Terminal,Program,Action'  From Dual;
```
## Including Service name

```
SET LINESIZE 500
SET PAGESIZE 1000

COLUMN username FORMAT A30
COLUMN osuser FORMAT A20
COLUMN spid FORMAT A10
COLUMN service_name FORMAT A15
COLUMN module FORMAT A45
COLUMN machine FORMAT A30
COLUMN logon_time FORMAT A20

SELECT NVL(s.username, '(oracle)') AS username,
       s.osuser,
       s.sid,
       s.serial#,
       p.spid,
       s.lockwait,
       s.status,
       s.service_name,
       s.machine,
       s.program,
       TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time,
       s.last_call_et AS last_call_et_secs,
       s.module,
       s.action,
       s.client_info,
       s.client_identifier
FROM   v$session s,
       v$process p
WHERE  s.paddr = p.addr
and service_name='TRMPS_RO_SVC'
ORDER BY s.username, s.osuser;
```
```
col size_mb format 999 head "size_mb"
COMPUTE SUM of size_mb ON REPORT


select substr(s.sid || ',' || s.serial#,1,10) sid,
       substr(s.username,1,8) u_name,
       substr(sum(round(((u.blocks*p.value)/1024/1024),2)),1,7) size_mb,
       substr(s.osuser||','||s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action,1,51) osUsr_Mach_mod_clinf_ter_prog
from v$sort_usage u,
     v$session s,
     v$sqlarea a,
     v$parameter p
where s.saddr (+) = u.session_addr
  and a.address (+) = s.sql_address
  and a.hash_value (+) = s.sql_hash_value
  and p.name = 'db_block_size'
group by  s.sid || ',' || s.serial#,
          s.username,
          a.sql_text,
          u.tablespace,
          substr(round(((u.blocks*p.value)/1024/1024),2),1,7),
          substr(s.osuser||','||s.machine||','||s.module||','||
          s.client_info||','||s.terminal||','||s.program||','||s.action,1,51);
```
# monitor_max_extent.sql
```
rem             e.g. if you want to report on segments that cannot extend
rem             by 3 times in the available
rem             free space (including auto extend), the the multipler would
rem             be set to 3.
rem
set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_extend.sh script to work


prompt          ====DB OWNER SEGMENT TABLESPACE======

column db_ownr_seg_tblespace format a80
select  substr(substr(db.name,1,10)||' '|| substr(a.owner,1,12)||'.'||
        substr(decode(partition_name, null, segment_name, segment_name
        || ':' || partition_name),1,20)||' '||
        a.tablespace_name,1,80) db_ownr_seg_tblespace
 from   dba_segments a, v$database db,
                (select df.tablespace_name, nvl(max(fs.bytes), 0) free,
                        nvl(sum(fs.bytes), 0) remain
                   from dba_data_files df,
                        dba_free_space fs
                  where df.file_id = fs.file_id (+)
                        group by df.tablespace_name) b
                ,(select  tablespace_name, max(maxbytes - bytes) morebytes,
                  sum(decode(AUTOEXTENSIBLE, 'YES', maxbytes - bytes, 0)) totalmorebytes,
                  sum(decode(AUTOEXTENSIBLE, 'YES', 1, 0)) autoextensible
                   from dba_data_files
                  group by tablespace_name) c
 where  a.tablespace_name = b.tablespace_name
   and  a.tablespace_name = c.tablespace_name
   and  a.segment_type != 'ROLLBACK'
   and  a.next_extent * nvl('&1',1) > b.free;
```
```
set verify on
set termout on

# monitor_max_extent.sql
rem    monitor_max_exent.sql
rem
rem    Script to list segments unable to extend due to reaching max_extents
rem    USAGE:   unable_to_extend <no. of extents>
rem
rem      <not of extents> is a numeric value used as a threshold
rem      to test when an object is close to max extents
rem      e.g. if you want to report on segments that are within 10 extents
rem      of max extents, then the the threshold would be set to 3.
rem

set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_extend.sh script to work


prompt          ====DB OWNER SEGMENT TABLESPACE======


column db_ownr_seg_tblespace format a80
select substr(substr(db.name,1,10)||' '|| substr(ds.owner,1,12)||'.'||
       substr(decode(ds.partition_name, null, ds.segment_name, ds.segment_name ||
       ':' || ds.partition_name),1,20)||' '||
       ds.tablespace_name,1,80) db_ownr_seg_tblespace
FROM  dba_segments ds, v$database db
WHERE ds.extents >= ds.max_extents - nvl('&1',1)
  AND ds.segment_type IN ('TABLE','INDEX','LOBINDEX','LOBSEGMENT','TABLE PARTITION','INDEX PARTITION','ROLLBACK','CLU
STER')
ORDER BY 1;
```
```
set verify on
set termout on

more monitor_resource_limit.sql
rem    monitor_resource_limit.sql
rem
rem    Script to list any resource withing 5% of capacity
rem    USAGE:   monitor_resource_limit.sql limit1 limit2
rem
rem      Will detect any resource within <limit1> percent  of capacity
rem      or for processes within <limit2>% of capacity
set verify off
set termout off
set linesize 80
set pagesize 0

   Select /*+ rule */ substr(name,1,10)||' '|| substr(resource_name,1,30)||' '||round(CURRENT_UTILIZATION/LIMIT_VALUE
*100,2)
    FROM (select * from v$resource_limit
          WHERE trim(limit_value) <> 'UNLIMITED'), v$database
    WHERE  to_number(current_utilization) > to_number(limit_value) * (nvl('&1',1)/100)
      AND resource_name <> 'max_rollback_segments'
      AND resource_name not like 'gcs%'
UNION
 select substr(name,1,10)||' '|| substr(resource_name,1,30)||' '||round(CURRENT_UTILIZATION/LIMIT_VALUE*100,2)
    FROM (select * from v$resource_limit
          WHERE trim(limit_value) <> 'UNLIMITED'), v$database
    WHERE  to_number(current_utilization) > to_number(limit_value) * (nvl('&2',1)/100)
      AND resource_name in ('processes','sessions')
ORDER BY 1;
spool off
set verify on
/
```

### monitor_tablespace.sql
```
rem    monitor_tablespace.sql
rem
rem    Script to report when the free space for a tablespace breaches
rem    a nominated threshold
rem    USAGE:   monitor_tablespace <threshold>
rem         where threshold is a number between 1 and 99 inclusive
rem
rem             script is called by monitor_tablespace.sh
rem             threshold is a numeric value representing a percentage
rem             of tablespace freespace available.
rem             e.g. if you want to report when there is less than 20% of
rem             free space left in a tablespace
rem             the threshold would be set to 20.
set verify off
set termout off
set linesize 80
set pagesize 0
set feedback on

col tspace form a25 Heading "Tablespace"
col tot_ts_size form 99999999999999 Heading "Size(Mb) "
col free_ts_size form 99999999999999 Heading "Free_(Mb)"
col ts_pct form 999 Heading "%Free "

select df.tablespace_name tspace,
       df.bytes/(1024*1024) tot_ts_size,
       fs.free_space/(1024*1024) free_ts_size,
       round(((df.bytes - (df.allocated - fs.free_space))/df.bytes) * 100) tc_pct
from  (select tablespace_name, sum(bytes) free_space
         from dba_free_space
        group by tablespace_name ) fs ,
      (select tablespace_name,
              sum(decode(autoextensible,'NO',bytes,maxbytes)) bytes,
              sum(user_bytes) allocated
         from dba_data_files
        group by tablespace_name ) df
where fs.tablespace_name = df.tablespace_name
and   fs.tablespace_name not in ('TEMP','ROLLBACK','UNDOTBS1','UNDOTBS2','UNDOTBS3')
and ((df.bytes - (df.allocated - fs.free_space))/df.bytes) * 100 < &1
/
```

### monitor_tablespace_status.sql
```
rem    monitor_tablespace_status.sql
rem
rem    Script to list OFFLINE tablespaces and/or INVALID files
rem    USAGE:   monitor_tablespace_status
rem
rem    script is invoked by monitor_tablespace_status.sh to report
rem    on tablespaces that are offline and datafile that are unavailable
rem
set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_tablespace_status.sh script to work


prompt          ====INSTANCE TABLESPACE FILENAME STATUS======


column tblespace_filename_status format a80
select vi.instance_name||' '||ddf.tablespace_name||' '||ddf.file_name||' '||ddf.status instnce_tblspce_filnm_status
from dba_data_files ddf, v$instance vi
where ddf.status <> 'AVAILABLE'
and tablespace_name not like ('%RBS%')
union
select vi.instance_name||' '||dt.tablespace_name||'        '||dt.status instnce_tblspce_filnm_status
from dba_tablespaces dt, v$instance vi
where dt.status <> 'ONLINE';
```
```
set verify on
set termout on

TEMP Teblaspace
set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_extend.sh script to work


column db_ownr_seg_tblespace format a80

select   substr(u.tablespace,1,15) tablespace,
         sum(round(((u.blocks*p.value)/1024/1024),2)) ts_used_mb, '(tablespace MbUsed)'
from v$sort_usage u,  v$parameter p
where p.name = 'db_block_size'
group by  u.tablespace
having  sum(u.blocks)  > (select (sum(decode(autoextensible,'NO',blocks,maxblocks))*(nvl('&1',1)/100) )
                          from dba_temp_files dtf
                          where dtf.tablespace_name = u.tablespace);
spool off
set verify on
set termout on
```
### monitor_locks.sql

```
-- |----------------------------------------------------------------------------|
-- | DATABASE : Oracle                                                          |
-- | FILE     : rac_locks_blocking.sql                                          |
-- | CLASS    : Real Application Clusters                                       |
-- | PURPOSE  : Query all Blocking Locks in the databases. This query will      |
-- |            display both the user(s) holding the lock and the user(s)       |
-- |            waiting for the lock. This script is RAC enabled.               |
-- | NOTE     : As with any code, ensure to test this script in a development   |
-- |            environment before attempting to run it in production.          |
-- +----------------------------------------------------------------------------+


SET LINESIZE 145
SET PAGESIZE 9999


COLUMN locking_instance   FORMAT a17   HEAD 'LOCKING|Instance - SID'  JUST LEFT
COLUMN locking_sid        FORMAT a7    HEAD 'LOCKING|SID'             JUST LEFT
COLUMN waiting_instance   FORMAT a17   HEAD 'WAITING|Instance - SID'  JUST LEFT
COLUMN waiting_sid        FORMAT a7    HEAD 'WAITING|SID'             JUST LEFT
COLUMN waiter_lock_type                HEAD 'Waiter Lock Type'        JUST LEFT
COLUMN waiter_mode_req                 HEAD 'Waiter Mode Req.'        JUST LEFT

COLUMN serial_number      FORMAT a7    HEAD 'Serial|Number'           JUST LEFT
COLUMN session_status                  HEAD 'Status'                  JUST LEFT
COLUMN oracle_user        FORMAT a20   HEAD 'Oracle|Username'         JUST LEFT
rem           that have been running longer than the threshold ($1)
COLUMN object_owner       FORMAT a15   HEAD 'Object|Owner'            JUST LEFT
COLUMN object_name        FORMAT a20   HEAD 'Object|Name'             JUST LEFT
COLUMN object_type        FORMAT a15   HEAD 'Object|Type'             JUST LEFT


CLEAR BREAKS


prompt
prompt +----------------------------------------------------------------------------+
prompt | BLOCKING LOCKS                                                             |
prompt +----------------------------------------------------------------------------+
prompt


SELECT
    ih.instance_name || ' - ' ||  lh.sid        locking_instance
  , iw.instance_name || ' - ' ||  lw.sid        waiting_instance
  , DECODE (   lh.type
             , 'CF', 'Control File'
             , 'DX', 'Distrted Transaction'
             , 'FS', 'File Set'
             , 'IR', 'Instance Recovery'
             , 'IS', 'Instance State'
             , 'IV', 'Libcache Invalidation'
             , 'LS', 'LogStartORswitch'
             , 'MR', 'Media Recovery'
             , 'RT', 'Redo Thread'
             , 'RW', 'Row Wait'
             , 'SQ', 'Sequence #'
             , 'ST', 'Diskspace Transaction'
             , 'TE', 'Extend Table'
             , 'TT', 'Temp Table'
             , 'TX', 'Transaction'
             , 'TM', 'Dml'
             , 'UL', 'PLSQL User_lock'
             , 'UN', 'User Name'
             , 'Nothing-'
           )                                    waiter_lock_type
  , DECODE (   lw.request
             , 0, 'None'
             , 1, 'NoLock'
             , 2, 'Row-Share'
             , 3, 'Row-Exclusive'
             , 4, 'Share-Table'
             , 5, 'Share-Row-Exclusive'
             , 6, 'Exclusive'
             , 'Nothing-'
           )                                    waiter_mode_req
FROM
    gv$lock     lw
  , gv$lock     lh
  , gv$instance iw
  , gv$instance ih
WHERE
   iw.inst_id = lw.inst_id
  AND ih.inst_id = lh.inst_id
  AND lh.id1     = lw.id1
  AND lh.id2     = lw.id2
  AND lh.request = 0
  AND lw.lmode   = 0
  AND (lh.id1, lh.id2) IN ( SELECT id1,id2
                            FROM   gv$lock
                            WHERE  request = 0
                            INTERSECT
                            SELECT id1,id2
                            FROM   gv$lock
                            WHERE  lmode = 0
                          )
ORDER BY
    lh.sid
/
```
### monitor_long_sessions.sql
```
rem    monitor_long_sessions.sql
rem
rem    Script to list session details for active sessions
rem           that have been running longer than the threshold ($1) 3600Seconds
rem           seconds
rem    USAGE:   monitor_long_sessions.sql seconds_threshold
rem

set linesize 132
set pagesize 64
set feedback off
set trimspool on
col sid format 999 head "Sid"
col serial# format 99999 head "Serial"
col username format a8 head "ORA User"
col action format a8 head "OS User"
col machine format a8 head "Source"
col status format a8 head "Status"
col process format a9 head "SourcePID"
col spid format a8 head "LocalPID"
col last_call_et format 9999999 head "ExecSecs"
col logon_time format a16 head "Logged_On_Since"
col module format a32 head "Module"
set heading off
set heading on

select /*+ rule */ s.sid,s.serial#,p.spid,s.username,rpad(s.action,8) "OS User",
rpad(s.machine,8) "Source",s.process,
       s.last_call_et,to_char(s.logon_time,'DD/MM/YYYY HH24:MI') "Logged_On_Since",
       s.status,s.module
from v$session s, v$process p
where status in ( 'ACTIVE','SNIPED')
  and s.type != 'BACKGROUND'
  and s.last_call_et > '&1'
  and s.paddr = p.addr
  and s.lockwait is null
order by s.last_call_et, s.sid
/

set heading off
select t.sql_text
  from v$session s, V$sqltext_with_newlines t
where status in ( 'ACTIVE','SNIPED')
   and s.type != 'BACKGROUND'
   and s.last_call_et > '&1'
   and s.lockwait is null
   and s.sql_address = t.address
   and s.sql_hash_value = t.hash_value
 order by s.last_call_et, s.sid, t.piece
/
```

### monitor_dbms_jobs.sql
```
rem    monitor_dbms_jobs.sql
rem
rem    Script to list dbms jobs that are failing/broken
rem    USAGE:   unable_to_extend <no. of extents>
rem

set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_dbms_jobs.sh script to work


SELECT job, priv_user, failures,
       decode(to_char(next_date,'J'),'3182030','** Disabled/Broken **',
       to_char(next_date,'dd-Mon-yyyy hh24:mi:ss')) overdue_next_date
FROM   dba_jobs dj
WHERE  (dj.next_date < (sysdate - 1800/86400) -- allow 30 min for snp latency
   OR  dj.failures > 0)
  AND  NOT EXISTS (SELECT null FROM dba_jobs_running
       WHERE job = dj.job);

set verify on
set termout on
```
### monitor_dataguard.sql
```
rem    monitor_dataguard.sql
rem
rem    Script to report status of applied archive logs on standby database
rem
rem
rem    USAGE:   monitor_dataguard
rem
rem             script is called by monitor_dataguard.sh
rem

set verify off
set termout off
set linesize 180
set pagesize 60
set trimspool on

col name      format a15
col host_name format a15

alter session set nls_date_format = 'DD-MON-YY hh24:mi:ss';

select d.name, i.status, d.database_role, i.host_name
from v$database d,
     v$instance i;

col name      format a40
col message   format a90
col sequence# format 99999999

PROMPT Checking V$ARCHIVE_GAP
PROMPT ======================

select *
from v$archive_gap;

PROMPT Checking archive logs applied in last two days/weekend
PROMPT ======================================================

select sequence#, name, first_time, next_time, completion_time, applied, status
from v$archived_log
where first_time > case to_char(sysdate,'DY')
                   when 'MON' then trunc(sysdate - 3,'dd')
                   else            trunc(sysdate - 1,'dd')
                   end
order by sequence#;

PROMPT Checking messages
PROMPT =================

select timestamp, message, severity
from v$dataguard_status
where upper(severity) in ('ERROR','FATAL');

PROMPT Checking status of apply process
PROMPT ================================

select process, status, sequence#
from v$managed_standby
where process like 'MRP%';

spool off
set verify on
set termout on
```

### USER INFORMATION
```
SELECT   s.status ||','||s.serial#||','||s.TYPE||','||
         s.username||','||s.osuser||','||
         s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action
    FROM v$session s, v$process p, SYS.v_$sess_io si
   WHERE s.paddr = p.addr(+) AND si.SID(+) = s.SID;
```

### ToKillsession -- collect the session information and confirm from the respective team first before killing session
```
alter system kill session '''||sid||','||serial#||''' immediate;' 
from gv$session 
where INST_ID=1 and status='INACTIVE' 
and LOGON_TIME between '22-OCT-12' and '27-NOV-12' 
group by inst_id,sid,serial#,username,osuser,machine,status,LOGON_TIME 
order by LOGON_TIME desc;
```
```
rem    monitor_resource_limit.sql
rem
rem    Script to list any resource withing 5% of capacity
rem    USAGE:   monitor_resource_limit.sql limit1 limit2
rem
rem      Will detect any resource within <limit1> percent  of capacity
rem      or for processes within <limit2>% of capacity
set verify off
set termout off
set linesize 80
set pagesize 0

   Select /*+ rule */ substr(name,1,10)||' '|| substr(resource_name,1,30)||' '||round(CURRENT_UTILIZATION/LIMIT_VALUE*100,2)
    FROM (select * from v$resource_limit
          WHERE trim(limit_value) <> 'UNLIMITED'), v$database
    WHERE  to_number(current_utilization) > to_number(limit_value) * (nvl('&1',1)/100)
      AND resource_name <> 'max_rollback_segments'
      AND resource_name not like 'gcs%'
UNION
 select substr(name,1,10)||' '|| substr(resource_name,1,30)||' '||round(CURRENT_UTILIZATION/LIMIT_VALUE*100,2)
    FROM (select * from v$resource_limit
          WHERE trim(limit_value) <> 'UNLIMITED'), v$database
    WHERE  to_number(current_utilization) > to_number(limit_value) * (nvl('&2',1)/100)
      AND resource_name in ('processes','sessions')
ORDER BY 1;
spool off
set verify on
/
```

### monitor_max_extent.sql
```
rem    monitor_max_exent.sql
rem
rem    Script to list segments unable to extend due to reaching max_extents
rem    USAGE:   unable_to_extend <no. of extents>
rem
rem      <not of extents> is a numeric value used as a threshold
rem      to test when an object is close to max extents
rem      e.g. if you want to report on segments that are within 10 extents
rem      of max extents, then the the threshold would be set to 3.
rem

set verify off
set termout off
set linesize 80
set pagesize 0
rem set feedback off  feedback needed for monitor_extend.sh script to work


prompt          ====DB OWNER SEGMENT TABLESPACE======


column db_ownr_seg_tblespace format a80
select substr(substr(db.name,1,10)||' '|| substr(ds.owner,1,12)||'.'||
       substr(decode(ds.partition_name, null, ds.segment_name, ds.segment_name ||
       ':' || ds.partition_name),1,20)||' '||
       ds.tablespace_name,1,80) db_ownr_seg_tblespace
FROM  dba_segments ds, v$database db
WHERE ds.extents >= ds.max_extents - nvl('&1',1)
  AND ds.segment_type IN ('TABLE','INDEX','LOBINDEX','LOBSEGMENT','TABLE PARTITION','INDEX PARTITION','ROLLBACK','CLUSTER')
ORDER BY 1;

set verify on
set termout on
```
## Archive Information

archive_gen.sh
```
#export PATH=$ORACLE_HOME/bin:$PATH:.
ps -ef |grep pmon |grep -v +ASM |grep -v "grep pmon"|cut -f3 -d_ |while read dblist
do
export ORACLE_SID=$dblist
echo $ORACLE_SID
export ORACLE_HOME=`cat /etc/oratab|grep ^$dblist |awk -F: {'print $2'}`
export spoolf1="$dblist"_audit.lst
$ORACLE_HOME/bin/sqlplus -s  "/as sysdba" << eof
spool $spoolfl
select name from v\$database;
@arch_gen.sql
@arch_gen2.sql
spool off
exit
eof
done
```
### arch_gen.sql
```
SELECT A.*,Round(A.Count#*B.AVG#/1024/1024) Daily_Avg_Mb
FROM (SELECT To_Char(First_Time,'YYYY-MM-DD') DAY,Count(1) Count#,
Min(RECID) Min#,Max(RECID) Max# FROM v$log_history GROUP
BY To_Char(First_Time,'YYYY-MM-DD') ORDER BY 1 DESC ) A,
(SELECT Avg(BYTES) AVG#,Count(1) Count#,Max(BYTES) Max_Bytes,Min(BYTES) Min_Bytes
FROM v$log) B;
```
### arch_gen2.sql
```
select sum(GB_USED_PER_DAY)/count(GB_USED_PER_DAY) from (SELECT
 TO_CHAR(completion_time,'YYYY-MM-DD') completion_date,
 round (SUM(block_size*(blocks+1)) / 1024 / 1024 / 1024 , 2) GB_USED_PER_DAY
 FROM v$archived_log
 WHERE TRUNC(completion_time) BETWEEN
 TRUNC(SYSDATE-30) AND TRUNC(SYSDATE)
GROUP BY TO_CHAR(completion_time,'YYYY-MM-DD')
 order by 1 desc);
```

## House Keeping Shell script for Oracle Database Home
 
oracle_housekeeping.sh
```
#!/bin/sh

#       -----------------------------------------------------------------------
#       Name            oracle_housekeeping.sh
#
#       Purpose         Cleanup of Oracle log, trace, audit and core files
#
#       Parameters      1. ORACLE_SID
#
#       Run As          oracle, from cron
#
#       Logic           For the database
#                               1.  Determine directory locations by
#                                   querying the database as user SNIFFER_DOG
#                               2.  Cleanup audit files
#                               3.  Cleanup core dumps
#                               4.  Cleanup trace files
#                               5.  Cleanup alert log files
#                       For ORACLE_HOME
#                               6.  Cleanup audit files
#                               7.  Cleanup network log files
#
#       Modification History
#
#       Date            Who     What
#       ==========      ======  ======================================
#              <>                  <>                  <>#
# -----------------------------------------------------------------------

#       ------------------------------------------------------------------------
#       Check input parameters
#       ------------------------------------------------------------------------

if [ "$1" ] ; then
        ORACLE_SID=$1
        export ORACLE_SID
else
        echo "ERROR: Database name not passed as parameter to oracle_housekeeping.sh ... Aborting"
        exit 1
fi

#       -----------------------------------------------------------------------
#       Local variables
#       -----------------------------------------------------------------------

OUTFILE=/tmp/oracle_housekeeping_${ORACLE_SID}.out
LOGDIR=/var/oracle/cron//${ORACLE_SID}
LOGFILE=${LOGDIR}/oracle_housekeeping_${ORACLE_SID}_`date +%Y%m%d_%H%M`.log
KEEPTIME=30

#       -----------------------------------------------------------------------
#       Redirect output to logfile
#       -----------------------------------------------------------------------

exec > ${LOGFILE} 2>&1

echo "***************************************************************"
echo
echo "ORACLE HOUSEKEEPING for $1 Started `date`"
echo

#       ------------------------------------------------------------------------
#       Ensure local bin directory is in PATH
#       ------------------------------------------------------------------------

case "$PATH" in
        */usr/local/bin*)       ;;
        *:)                     PATH=${PATH}/usr/local/bin   ;;
        "")                     PATH=/usr/local/bin          ;;
        *)                      PATH=${PATH}:/usr/local/bin  ;;
esac

export PATH

#       ------------------------------------------------------------------------
#       Set Oracle environment
#       ------------------------------------------------------------------------

ORACLE_SID=$1
ORAENV_ASK=NO
export ORACLE_SID
export ORAENV_ASK

. /usr/local/bin/oraenv

#       -----------------------------------------------------------------------
#       Check the database is running, if not exit
#       -----------------------------------------------------------------------

if [ -z "`ps -ef | grep ora_pmon_${ORACLE_SID} | grep -v grep`" ] ; then
        echo "Database is not running, cleanup skipped"
        echo
        exit 1
fi

#       -----------------------------------------------------------------------
#       Determine the location of the log, trace, audit and core dump
#       directories by querying the database.
#       -----------------------------------------------------------------------

rm -f ${OUTFILE}

${ORACLE_HOME}/bin/sqlplus -s /nolog << EOF
connect \/ as sysdba
set echo off
set heading off
set pages 0
set lines 200
col name  format a30
col value format a80

spool ${OUTFILE}

select name,value
from v\$parameter
where name in ('audit_file_dest',
                'user_dump_dest',
                'background_dump_dest',
                'core_dump_dest');

spool off
EOF

#       -----------------------------------------------------------------------
#       If no spool file was created, skip to the next database
#       -----------------------------------------------------------------------

if [ ! -s ${OUTFILE} ] ; then
        echo "Unable to determine directory names, cleanup skipped"
        echo
        exit 1
fi

#       -----------------------------------------------------------------------
#       Cleanup audit files older than specified keep time
#       -----------------------------------------------------------------------

echo "Cleaning up audit files for ${ORACLE_SID}"
echo

AUDIT_DIR=`grep "^audit_file_dest" ${OUTFILE} | awk '{print $2}'`
AUDIT_DIR=`echo ${AUDIT_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${AUDIT_DIR}
echo

find ${AUDIT_DIR} -name "*.aud" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo

#       -----------------------------------------------------------------------
#       Cleanup core files
#       -----------------------------------------------------------------------

echo "Cleaning up core files for ${ORACLE_SID}"
echo

CORE_DIR=`grep "^core_dump_dest" ${OUTFILE} | awk '{print $2}'`
CORE_DIR=`echo ${CORE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${CORE_DIR}
echo

find ${CORE_DIR} -name "core_*" -ls -exec rm -r {} \;

echo

#       -----------------------------------------------------------------------
#       Cleanup trace files older than specified keep time
#       -----------------------------------------------------------------------

echo "Cleaning up trace files for ${ORACLE_SID}"
echo

TRACE_DIR=`grep "^user_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${TRACE_DIR}
echo

find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;
find ${TRACE_DIR} -name "*.trm" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo

TRACE_DIR=`grep "^background_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${TRACE_DIR}
echo

find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;
find ${TRACE_DIR} -name "*.trm" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo

#       -----------------------------------------------------------------------
#       Cleanup alert log files
#       -----------------------------------------------------------------------

echo "Cleaning up alert log files for ${ORACLE_SID}"
echo

TRACE_DIR=`grep "^background_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${TRACE_DIR}
echo

cd ${TRACE_DIR}

DESTFILE=alert_${ORACLE_SID}_`date '+%Y%m%d'`

cp alert_${ORACLE_SID}.log $DESTFILE
cat /dev/null > alert_${ORACLE_SID}.log
compress $DESTFILE

find ${TRACE_DIR} -name "alert*.Z" -mtime +365 -ls -exec rm {} \;

echo

#       -----------------------------------------------------------------------
#       Cleanup log files from this job (keep logs from the last 5 weeks)
#       and exit
#       -----------------------------------------------------------------------

echo "***************************************************************"
echo
echo "Cleaning up log files from oracle_housekeeping.sh"
echo

find ${LOGDIR} -name "oracle_housekeeping_${ORACLE_SID}*.log" -mtime +36 -ls -exec rm {} \;

echo
echo "ORACLE HOUSEKEEPING Finished `date`"
echo
echo "********************* END OF JOB ******************************"

exit

```
### sample execute command
/var/oracle/cron/oracle_housekeeping.sh EAYM      > /var/oracle/cron/logs/oracle_housekeeping.out 2>>/dev/null


## House Keeping Shell script for Grid Infrastructure Home

grid_housekeping.sh
```
#!/bin/sh

#       -----------------------------------------------------------------------
#       Name            grid_housekeeping.sh
#
#       Purpose         Cleanup of Oracle log, trace, audit and core files
#
#       Parameters      1. ORACLE_SID
#
#       Run As          oracle, from cron
#
#       Logic           For the database
#                               1.  Cleanup audit files
#                               2.  Cleanup network log files
#
#       Modification History
#
#       Date            Who     What
#       ==========      ======  ======================================
#       <>                  <>                  <>
#       -----------------------------------------------------------------------

#       ------------------------------------------------------------------------
#       Check input parameters
#       ------------------------------------------------------------------------

if [ "$1" ] ; then
        ORACLE_SID=$1
        export ORACLE_SID
else
        echo "ERROR: Database name not passed as parameter to grid_housekeeping.sh ... Aborting"
        exit 1
fi

#       -----------------------------------------------------------------------
#       Local variables
#       -----------------------------------------------------------------------

OUTFILE=/tmp/grid_housekeeping_${ORACLE_SID}.out
LOGDIR=/var/oracle/cron/asm
LOGFILE=${LOGDIR}/grid_housekeeping_${ORACLE_SID}_`date +%Y%m%d_%H%M`.log
KEEPTIME=15

#       -----------------------------------------------------------------------
#       Redirect output to logfile
#       -----------------------------------------------------------------------

exec > ${LOGFILE} 2>&1

echo "***************************************************************"
echo
echo "ORACLE HOUSEKEEPING for $1 Started `date`"
echo

#       ------------------------------------------------------------------------
#       Ensure local bin directory is in PATH
#       ------------------------------------------------------------------------

case "$PATH" in
        */usr/local/bin*)       ;;
        *:)                     PATH=${PATH}/usr/local/bin   ;;
        "")                     PATH=/usr/local/bin          ;;
        *)                      PATH=${PATH}:/usr/local/bin  ;;
esac

export PATH

#       ------------------------------------------------------------------------
#       Set Oracle environment
#       ------------------------------------------------------------------------

ORACLE_SID=$1
ORAENV_ASK=NO
export ORACLE_SID
export ORAENV_ASK

. /usr/local/bin/oraenv

#       -----------------------------------------------------------------------
#       Determine the location of the log, trace, audit and core dump
#       directories by querying the database.
#       -----------------------------------------------------------------------

rm -f ${OUTFILE}

${ORACLE_HOME}/bin/sqlplus -s /nolog << EOF
connect \/ as sysdba
set echo off
set heading off
set pages 0
set lines 200
col name  format a30
col value format a80

spool ${OUTFILE}

select name,value
from v\$parameter
where name in ('audit_file_dest',
                'user_dump_dest',
                'background_dump_dest',
                'core_dump_dest');

spool off
EOF

#       -----------------------------------------------------------------------
#       If no spool file was created, skip to the next database
#       -----------------------------------------------------------------------

if [ ! -s ${OUTFILE} ] ; then
        echo "Unable to determine directory names, cleanup skipped"
        echo
        exit 1
fi

#       -----------------------------------------------------------------------
#       Cleanup trace files older than specified keep time
#       -----------------------------------------------------------------------

echo "Cleaning up trace files for ${ORACLE_SID}"
echo

TRACE_DIR=`grep "^user_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${TRACE_DIR}
echo

find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo

TRACE_DIR=`grep "^background_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`

echo "  directory: " ${TRACE_DIR}
echo

find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo
#       -----------------------------------------------------------------------
#       Cleanup network log files
#       TNS log files are cycled through a four week cycle (old logs are zipped)
#       -----------------------------------------------------------------------

echo "Cleaning up listener log files"
echo

cd ${TNS_ADMIN:-${ORACLE_HOME}/network/admin}

LISTENER_DIR=`grep LOG_DIRECTORY listener.ora | awk '{print $NF}'`
#LISTENER_DIR=${LISTENER_DIR:-${ORACLE_HOME}/network/log}

if [ -d ${LISTENER_DIR} ] ; then
        cd ${LISTENER_DIR}

        echo "  directory: " ${LISTENER_DIR}
        echo

        for TNSLOG in `ls -1 listener.log`
        do
                echo "  log: ${TNSLOG}"
                test -f ${TNSLOG}.5.gz && rm ${TNSLOG}.5.gz
                test -f ${TNSLOG}.4.gz && mv ${TNSLOG}.4.gz ${TNSLOG}.5.gz
                test -f ${TNSLOG}.3.gz && mv ${TNSLOG}.3.gz ${TNSLOG}.4.gz
                test -f ${TNSLOG}.2.gz && mv ${TNSLOG}.2.gz ${TNSLOG}.3.gz
                test -f ${TNSLOG}.1.gz && mv ${TNSLOG}.1.gz ${TNSLOG}.2.gz
                test -f ${TNSLOG}.0.gz && mv ${TNSLOG}.0.gz ${TNSLOG}.1.gz
                cp -p ${TNSLOG} ${TNSLOG}.0
                cat /dev/null > ${TNSLOG}
                gzip ${TNSLOG}.0
        done
fi

echo


#       -----------------------------------------------------------------------
#       Cleanup network trace files
#       -----------------------------------------------------------------------

echo "Cleaning up listener trace files"
echo

cd ${TNS_ADMIN:-${ORACLE_HOME}/network/admin}

TRACE_DIR=`grep TRACE_DIRECTORY listener.ora | awk '{print $NF}'`
#TRACE_DIR=${TRACE_DIR:-${ORACLE_HOME}/network/trace}

if [ -d ${TRACE_DIR} ] ; then

        echo "  directory: " ${TRACE_DIR}
        echo

        find ${TRACE_DIR} -name "*.xml" -ls -exec rm {} \;
        find ${TRACE_DIR} -name "*.xml.gz" -ls -exec rm {} \;
fi

echo


#       -----------------------------------------------------------------------
#       Cleanup audit files older than specified keep time
#
#       Audit files may exist under ORACLE_HOME, even if the database
#       parameter is to write to another directory.
#       -----------------------------------------------------------------------

echo "Cleaning up audit files under ${ORACLE_HOME}"
echo

AUDIT_DIR=$ORACLE_HOME/rdbms/audit

echo "  directory: " ${AUDIT_DIR}
echo

find ${AUDIT_DIR} -name "*.aud" -mtime +${KEEPTIME} -ls -exec rm {} \;

echo

#       -----------------------------------------------------------------------
#       Cleanup log files from this job (keep logs from the last 5 weeks)
#       and exit
#       -----------------------------------------------------------------------

echo "***************************************************************"
echo
echo "Cleaning up log files from grid_housekeeping.sh"
echo

find ${LOGDIR} -name "grid_housekeeping_${ORACLE_SID}*.log" -mtime +36 -ls -exec rm {} \;

echo
echo "ORACLE HOUSEKEEPING Finished `date`"
echo
echo "********************* END OF JOB ******************************"

exit
```

### sample execute command
/var/oracle/cron/grid_housekeping.sh +ASM1      > /var/oracle/cron/logs/grid_housekeping.out 2>>/dev/null


# Monitor ASM Process
more process_asm.sh
```
ps -ef | grep -i asm | wc -l >> /tmp/session.out
export proc=`ps -ef | grep -i asm | wc -l`
if [ "$proc" -gt 900 ]; then
mailx -s " Process threshold has breached for ASM `hostname` at `date`" "monowar.mukul@gmail.com" < /tmp/session.out
fi
```
### sample execute command
/export/home/oracle/bin/monitor/process_asm.sh >> /export/home/oracle/bin/monitor/logs/monitor_ASM_process.log

### monitor_report_sessions.sql
```
set head off
set linesize 200 trimspool on pagesize 0
select '  monitor_tempspaces.sh run on '||to_char(sysdate,'DD/MM/YYYY HH24:MI:SS
') from dual;

SELECT 'Status,Serial#,Type,DB User,Client User,Machine,Module,Client Info, Term
inal,Program,Action'  From Dual;

SELECT   s.status ||','||s.serial#||','||s.TYPE||','||
         s.username||','||s.osuser||','||
         s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action
    FROM v$session s, v$process p, SYS.v_$sess_io si
   WHERE s.paddr = p.addr(+) AND si.SID(+) = s.SID;

SELECT 'Sid,Serial#,User,Temp(Mb),Client User,Machine,Module,Client Info, Terminal,Program,Action'  From Dual;
```
```
col size_mb format 999 head "size_mb"
COMPUTE SUM of size_mb ON REPORT


select substr(s.sid || ',' || s.serial#,1,10) sid,
       substr(s.username,1,8) u_name,
       substr(sum(round(((u.blocks*p.value)/1024/1024),2)),1,7) size_mb,
       substr(s.osuser||','||s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action,1,51) osUsr_Mach_mod_clinf_ter_prog
from v$sort_usage u,
     v$session s,
     v$sqlarea a,
     v$parameter p
where s.saddr (+) = u.session_addr
  and a.address (+) = s.sql_address
  and a.hash_value (+) = s.sql_hash_value
  and p.name = 'db_block_size'
group by  s.sid || ',' || s.serial#,
          s.username,
          a.sql_text,
          u.tablespace,
          substr(round(((u.blocks*p.value)/1024/1024),2),1,7),
          substr(s.osuser||','||s.machine||','||s.module||','||
          s.client_info||','||s.terminal||','||s.program||','||s.action,1,51);
```

## Manually Clean Trace files [ replace the right db_home and sid / get the path from the parameter file]
```
find /<db_home>/admin/<SID>/bdump -name "*.trc" -a -mtime +30 -exec rm  -rf {} \;
find /<db_home>/admin/<SID>/udump -name "*.trc" -a -mtime +30 -exec rm -rf {} \;
find /<db_home>/admin/<SID>/adump -name "*.aud" -a -mtime +30 -exec rm -rf {} \;

find /app/wspsp/product/diag/rdbms/eaym/eaym2/trace  -name "*.trc" -a -mtime +30 -exec rm  -rf {} \;
find /app/wspsp/product/admin/eaym/adump  -name "*.aud" -a -mtime +30 -exec rm  -rf {} \;
```

## Get the sql session history
```
spool hist_sql_DEV.lst
undefine sql_id
prompt 'Please enter SQL_ID: (ie: 0sfvmtydgh4qan)'
define sql_id='&SQL_ID'

set null null
set lines 320
set pages 99
set trimspool on
col snap_beg format a12
col iowait_delta format 99999999.99 heading io|wait|delta|(ms)
col iowait_total format 99999999.99 heading io|wait|total|(ms)
col ELAPSED_TIME_DELTA format 99999999.99 heading elapsd|time|delta|(ms)
col CPU_TIME_DELTA format 99999999.99 heading cpu|time|delta|(ms)
col PLAN_HASH_VALUE heading plan_hash|value
col CONCURRENCY_WAIT_delta format 99999999.99 heading conc|wait|delta|(ms)
col CLUSTER_WAIT_DELTA format 99999999.99 heading clust|wait|delta|(ms)
col PX_SERVERS_EXECS_DELTA format 99999 heading PXServ|Exec|delta
col APWAIT_DELTA format 99999 heading appl|wait|time|delta(micro)
col PLSEXEC_TIME_DELTA format 99999 heading plsql|exec|time|delta(micro)
col JAVAEXEC_TIME_DELTA format 99999 heading java|exec|time|delta(micro)
col optimizer_cost format 9999 heading opt|cost
col optimizer_mode format a10 heading optim|mode
col kept_versions format 999 heading kept|vers
col invalidations_delta format 999 heading inv|alid|dlt
col parse_calls_delta format 99999 heading parse|calls|delta
col executions_delta format 999999 heading exec|delta
col fetches_delta format 9999999 heading fetches|delta
col end_of_fetch_count_delta format 99999 heading end|of|fetch|call|delta
col buffer_gets_delta format 99999999999 heading buffer|gets|delta
col disk_reads_delta format 9999999999 heading disk|reads|delta
col DIRECT_WRITES_DELTA format 99999999 heading direct|writes|delta
col rows_processed_delta format 999999999 heading rows|processed|delta
col rows_ex format 99999999 heading rows|exec
col snap_id format 99999 heading snap|id
col ela_ex format 99999999.99 heading elapsed|per|execution
col cwt_ex format 99999999.99 heading cwt|per|execution
col cc_ex format 99999999.99 heading cc|per|execution
col io_ex format 99999999.99 heading io|per|execution
col instance_number format 99 heading in|ID

select dba_hist_sqlstat.instance_number, sql_id, plan_hash_value,
dba_hist_sqlstat.snap_id,
to_char(dba_hist_snapshot.BEGIN_INTERVAL_TIME,'dd-mm hh24:mi') snap_beg,
invalidations_delta,
parse_calls_delta,
executions_delta,
px_servers_execs_delta,
fetches_delta,
buffer_gets_delta,
disk_reads_delta,
direct_writes_delta,
rows_processed_delta,
elapsed_time_delta/1000 elapsed_time_delta,
cpu_time_delta/1000 cpu_time_delta,
iowait_delta/1000 iowait_delta,
clwait_delta/1000 cluster_wait_delta,
ccwait_delta/1000 concurrency_wait_delta,
substr(optimizer_mode,1,3) opt,
case when executions_delta = 0 then NULL
when cpu_time_delta = 0 then NULL
else
(cpu_time_delta/executions_delta)/1000
end cpu_ex,
case when executions_delta = 0 then NULL
when elapsed_time_delta = 0 then NULL
else
(elapsed_time_delta/executions_delta)/1000
end ela_ex
,substr(SQL_PROFILE,1,32) sql_profile
from dba_hist_sqlstat, dba_hist_snapshot
where sql_id='&&sql_id'
and dba_hist_sqlstat.snap_id=dba_hist_snapshot.snap_id
and dba_hist_sqlstat.instance_number=dba_hist_snapshot.instance_number
order by dba_hist_sqlstat.instance_number, plan_hash_value, dba_hist_sqlstat.snap_id
/
select dba_hist_sqlstat.instance_number, sql_id, plan_hash_value,
dba_hist_sqlstat.snap_id,
to_char(dba_hist_snapshot.BEGIN_INTERVAL_TIME,'dd-mm hh24:mi') snap_beg,
invalidations_delta,
parse_calls_delta,
executions_delta,
elapsed_time_delta/1000 elapsed_time_delta,
cpu_time_delta/1000 cpu_time_delta,
iowait_delta/1000 iowait_delta,
clwait_delta/1000 cluster_wait_delta,
ccwait_delta/1000 concurrency_wait_delta,
substr(optimizer_mode,1,3) opt,
case when executions_delta = 0 then NULL
when rows_processed_delta = 0 then NULL
else
(rows_processed_delta/executions_delta)
end rows_ex,
case when executions_delta = 0 then NULL
when iowait_delta = 0 then NULL
else
(iowait_delta/executions_delta)/1000
end io_ex,
case when executions_delta = 0 then NULL
when clwait_delta = 0 then NULL
else
(clwait_delta/executions_delta)/1000
end cwt_ex,
case when executions_delta = 0 then NULL
when ccwait_delta = 0 then NULL
else
(ccwait_delta/executions_delta)/1000
end cc_ex,
case when executions_delta = 0 then NULL
when cpu_time_delta = 0 then NULL
else
(cpu_time_delta/executions_delta)/1000
end cpu_ex,
case when executions_delta = 0 then NULL
when elapsed_time_delta = 0 then NULL
else
(elapsed_time_delta/executions_delta)/1000
end ela_ex
from dba_hist_sqlstat, dba_hist_snapshot
where sql_id='&&sql_id'
and dba_hist_sqlstat.snap_id=dba_hist_snapshot.snap_id
and dba_hist_sqlstat.instance_number=dba_hist_snapshot.instance_number
order by dba_hist_sqlstat.instance_number, plan_hash_value, dba_hist_sqlstat.snap_id
/

select plan_table_output from table (dbms_xplan.display_awr('&&sql_id',null, null, 'ADVANCED +PEEKED_BINDS'));

select plan_table_output from table (dbms_xplan.display_cursor('&&sql_id', null, 'ADVANCED +PEEKED_BINDS'));

spool off
```

### Get the rollback segment information

```
column RBS_NAME format a10


PROMPT
PROMPT Transaction and Rollback Information
PROMPT ------------------------------------


select        '    Rollback Used                : '||t.used_ublk*8192/1024/1024 ||' M'          || chr(10) ||
              '    Rollback Records             : '||t.used_urec        || chr(10)||
              '    Rollback Segment Number      : '||t.xidusn           || chr(10)||
              '    Rollback Segment Name        : '||r.name             || chr(10)||
              '    Logical IOs                  : '||t.log_io           || chr(10)||
              '    Physical IOs                 : '||t.phy_io           || chr(10)||
              '    RBS Startng Extent ID        : '||t.start_uext       || chr(10)||
              '    Transaction Start Time       : '||t.start_time       || chr(10)||
              '    Transaction_Status           : '||t.status
FROM v$transaction t, v$session s, v$rollname r
WHERE t.addr = s.taddr
and r.usn = t.xidusn
and s.sid = &sid_number
/

```
### SORT information
```
PROMPT
PROMPT Sort Information
PROMPT ----------------


column username format a20
column user format a20
column tablespace format a20


SELECT        '    Sort Space Used(8k block size is asssumed    : '||u.blocks/1024*8 ||' M'             || chr(10) ||
              '    Sorting Tablespace                           : '||u.tablespace       || chr(10)||
              '    Sort Tablespace Type                 : '||u.contents || chr(10)||
              '    Total Extents Used for Sorting               : '||u.extents
FROM v$session s, v$sort_usage u
WHERE s.saddr = u.session_addr
AND s.sid = &sid_number
/
```
### Performance Statistics

more performance_stat.sql
```
col sid format 9999
col username format a10
col osuser format a10
col program format a25
col process format 9999999
col spid format 999999
col logon_time format a13
set lines 150
set heading off
set verify off
set feedback off
undefine sid_number
undefine spid_number
rem accept sid_number number prompt "pl_enter_sid:"
col sid NEW_VALUE sid_number noprint
col spid NEW_VALUE spid_number noprint

select  s.sid   sid,
                p.spid  spid,
         FROM v$session s,
              v$process p
         WHERE s.sid LIKE NVL('&sid', '%')
         AND p.spid LIKE NVL ('&OS_ProcessID', '%')
         AND s.process LIKE NVL('&Client_Process', '%')
         AND s.paddr = p.addr ;


PROMPT Session and Process Information
PROMPT -------------------------------


col event for a30


select '    SID                         : '||v.sid      || chr(10)||
       '    Serial Number               : '||v.serial#  || chr(10) ||
       '    Oracle User Name            : '||v.username         || chr(10) ||
       '    Client OS user name         : '||v.osuser   || chr(10) ||
       '    Client Process ID           : '||v.process  || chr(10) ||
       '    Client machine Name         : '||v.machine  || chr(10) ||
       '    Oracle PID                  : '||p.pid      || chr(10) ||
       '    OS Process ID(spid)         : '||p.spid     || chr(10) ||
       '    Session''s Status           : '||v.status   || chr(10) ||
       '    Logon Time                  : '||to_char(v.logon_time, 'MM/DD HH24:MIpm')   || chr(10) ||
       '    Program Name                : '||v.program  || chr(10)
from v$session v, v$process p
where v.paddr = p.addr
and v.serial# > 1
and p.background is null
and p.username is not null
and sid = &sid_number
order by logon_time, v.status, 1
/

Enter value for sid_number:

PROMPT Sql Statement
PROMPT --------------


select sql_text
from v$sqltext , v$session
where v$sqltext.address = v$session.sql_address
and sid = &sid_number
order by piece
/


PROMPT
PROMPT Event Wait Information
PROMPT ----------------------


select '   SID '|| &sid_number ||' is waiting on event  : ' || x.event || chr(10) ||
       '   P1 Text                      : ' || x.p1text || chr(10) ||
       '   P1 Value                     : ' || x.p1 || chr(10) ||
       '   P2 Text                      : ' || x.p2text || chr(10) ||
       '   P2 Value                     : ' || x.p2 || chr(10) ||
       '   P3 Text                      : ' || x.p3text || chr(10) ||
       '   P3 Value                     : ' || x.p3
from v$session_wait x
where x.sid= &sid_number
/


PROMPT
PROMPT Session Statistics
PROMPT ------------------


select        '     '|| b.name  ||'             : '||decode(b.name, 'redo size', round(a.value/1024/1024,2)||' M', a.value)
from v$session s, v$sesstat a, v$statname b
where a.statistic# = b.statistic#
and name in ('redo size', 'parse count (total)', 'parse count (hard)', 'user commits')
and s.sid = &sid_number
and a.sid = &sid_number
--order by b.name
order by decode(b.name, 'redo size', 1, 2), b.name
/


COLUMN USERNAME FORMAT a10
COLUMN status FORMAT a8
column RBS_NAME format a10


PROMPT

set heading on
set verify on


clear column

text.sql

spool latest1_pre.log

set pages 1000
col object_name format a40
col object_type format a20
col comp_name format a30
column library_name format a8
column file_spec format a60 wrap
spool text_install_verification.log

-- check on setup
select comp_name, status, substr(version,1,10) as version from dba_registry where comp_id = 'CONTEXT';
select * from ctxsys.ctx_version;
select substr(ctxsys.dri_version,1,10) VER_CODE from dual;

select count(*) from dba_objects where owner='CTXSYS';

-- Get a summary count
select object_type, count(*) from dba_objects where owner='CTXSYS' group by object_type;

-- Any invalid objects
select object_name, object_type, status from dba_objects where owner='CTXSYS' and status != 'VALID' order by object_name;


spool off
```

### Backup Status Report - Grant permission only to the right user / group
more bkp.sql
```
CREATE OR REPLACE FORCE VIEW "SYS"."RMAN_REPORT"
("DB_NAME","INPUT_TYPE", "STATUS", "START_TIME", "END_TIME", "HRS", "SUM_BYTES_BACKED_IN_GB","SUM_BACKUP_PIECES_IN_GB", "OUTPUT_DEVICE_TYPE") AS
select a.NAME,
INPUT_TYPE,
STATUS,
TO_CHAR(START_TIME,'DD-MON-YYYY-HH24:mi:ss') start_time,
TO_CHAR(END_TIME,'DD-MON-YYYY-HH24:mi:ss') end_time,
ELAPSED_SECONDS/3600 hrs,
INPUT_BYTES/1024/1024/1024 SUM_BYTES_BACKED_IN_GB,
OUTPUT_BYTES/1024/1024/1024 SUM_BACKUP_PIECES_IN_GB,
OUTPUT_DEVICE_TYPE
FROM V_$RMAN_BACKUP_JOB_DETAILS,v$database a
order by a.name,SESSION_KEY;
```
```
set lines 200 pages 2000
select * from rman_report;
```
```
CREATE OR REPLACE FORCE VIEW "RMAN_REPORT_DAILY"
("DB_NAME", "INPUT_TYPE", "STATUS", "START_TIME", "END_TIME")
AS
select name,INPUT_TYPE,STATUS,TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM v$database,V$RMAN_BACKUP_JOB_DETAILS where status in ('COMPLETED','COMPLETED WITH WARNINGS') and INPUT_TYPE in ('DB FULL','DB INCR') and OUTPUT_DEVICE_TYPE='SBT_TAPE' AND END_TIME=(SELECT MAX(END_TIME) FROM
V$RMAN_BACKUP_JOB_DETAILS WHERE STATUS in ('COMPLETED','COMPLETED WITH WARNINGS') AND INPUT_TYPE in ('DB FULL','DB INCR') and OUTPUT_DEVICE_TYPE='SBT_TAPE')
union
select name,INPUT_TYPE,STATUS,TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM V$RMAN_BACKUP_JOB_DETAILS,v$database where status in ('COMPLETED','COMPLETED WITH WARNINGS') and INPUT_TYPE='ARCHIVELOG' and OUTPUT_DEVICE_TYPE='SBT_TAPE' AND END_TIME=(SELECT MAX(END_TIME) FROM
V$RMAN_BACKUP_JOB_DETAILS WHERE STATUS in ('COMPLETED','COMPLETED WITH WARNINGS') AND INPUT_TYPE='ARCHIVELOG' and OUTPUT_DEVICE_TYPE='SBT_TAPE');

create synonym cscbkp.RMAN_REPORT_DAILY for sys.RMAN_REPORT_DAILY;
grant select on sys.RMAN_REPORT_DAILY to &usr;
```

## CRS resource status query script

cat crs_stat.sh
```
#!/usr/bin/ksh
#
# 10g CRS resource status query script
#
# Description:
#
# - The argument, $RSC_KEY, is optional and if passed to the script, will
# limit the output to HA resources whose names match $RSC_KEY.


RSC_KEY=$1
QSTAT=-u
AWK=/usr/bin/awk # if not available use /usr/bin/awk
# Table header:echo ""
$AWK \
'BEGIN {printf "%-45s %-10s %-18s\n", "HA Resource", "Target", "State";
printf "%-45s %-10s %-18s\n", "-----------", "------", "-----";}'

# Table body:
/opt/oracle/11.2.0.2_grid/grid/bin/crs_stat $QSTAT | $AWK \
'BEGIN { FS="="; state = 0; }
$1~/NAME/ && $2~/'$RSC_KEY'/ {appname = $2; state=1};
state == 0 {next;}
$1~/TARGET/ && state == 1 {apptarget = $2; state=2;}
$1~/STATE/ && state == 2 {appstate = $2; state=3;}
state == 3 {printf "%-45s %-10s %-18s\n", appname, apptarget, appstate; state=0;}'
health.sql
set lines 200 pages 2000
select instance_name,status,archiver from v$instance;
select * from v$recover_file;
archive log list
select flashback_on from v$database;
~
run_all.sh
#!/usr/bin/sh
set -vx
cd /home/oracle/sthati
cat /etc/oratab | while read LINE
do
  case $LINE in
  \#* ) ;;      #comment-line in oratab
  * )
                 ORACLE_SID=`echo $LINE | awk -F: '{print $1}' -`
                        if [ "$ORACLE_SID" = '*' ] ; then
                        ORACLE_SID=""
                                fi
                  export ORACLE_SID
                  echo $ORACLE_SID
        ORACLE_HOME=`echo $LINE | awk -F: '{print $2}' -`
        export ORACLE_HOME
        export PATH=$PATH:$ORACLE_HOME\bin:.
export SID=$ORACLE_SID
echo $SID |. oraenv 1>/dev/null 2>&1
sqlplus -S "/ as sysdba" <<!EOF
spool `echo $SID`_alert.log
@@/home/oracle/sthati/show_alert.sql
 spool off;
exit
!EOF

  ;;
esac
done
```

## Get default profile information
```
SQL> spool oon
SQL> select 'alter user ' ||u.username|| ' profile '||u.profile||';' from   sys.dba_users u;
SQL> spool off
```
### Passsword information 
cat PasswordBackup_<db>_<date>.sql
```
set lines 120
set pages 999
set ver off head off feed off
select 'alter user '|| name ||' identified by values '''||
decode(spare4,null,password,spare4)||''';' sql
from sys.user$;
```
## Check Memory Usage by Sessions in Oracle Database
```
SET pages500 lines110 trims ON
CLEAR col
col NAME format a30
col username format a20
break ON username nodup SKIP 1

SELECT vses.username||':'||vsst.SID||','||vses.serial# username, vstt.NAME, MAX(vsst.VALUE) VALUE
FROM v$sesstat vsst, v$statname vstt, v$session vses
WHERE vstt.statistic# = vsst.statistic# AND vsst.SID = vses.SID AND vstt.NAME IN
('session pga memory','session pga memory max','session uga memory','session uga memory max',
'session cursor cache count','session cursor cache hits','session stored procedure space',
'opened cursors current','opened cursors cumulative') AND vses.username IS NOT NULL
GROUP BY vses.username, vsst.SID, vses.serial#, vstt.NAME
ORDER BY vses.username, vsst.SID, vses.serial#, vstt.NAME;
```
