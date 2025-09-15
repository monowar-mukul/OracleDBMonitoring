# Oracle Database Monitoring Scripts

A comprehensive collection of Oracle database monitoring, maintenance, and utility scripts for DBAs.

> **Note**: These scripts are based on a lab setup. Please review and modify according to your environment and licensing requirements.

## Table of Contents

- [Database Connection](#database-connection)
- [Session Monitoring](#session-monitoring)
- [Extent and Space Monitoring](#extent-and-space-monitoring)
- [Resource and Performance Monitoring](#resource-and-performance-monitoring)
- [Tablespace Monitoring](#tablespace-monitoring)
- [Lock Monitoring](#lock-monitoring)
- [Long Running Sessions](#long-running-sessions)
- [DBMS Jobs Monitoring](#dbms-jobs-monitoring)
- [Data Guard Monitoring](#data-guard-monitoring)
- [Archive Log Monitoring](#archive-log-monitoring)
- [Housekeeping Scripts](#housekeeping-scripts)
- [ASM Monitoring](#asm-monitoring)
- [Backup Status Reports](#backup-status-reports)
- [CRS Resource Status](#crs-resource-status)
- [Memory and Performance](#memory-and-performance)

## Database Connection

### Connect without tnsnames.ora entry

```sql
-- Method 1: Direct connection
$ sqlplus username/password@hostname:port/SERVICENAME

-- Method 2: Interactive password
$ sqlplus username
Enter password: password@//hostname:port/SERVICENAME

-- Method 3: Using nolog
$ sqlplus /nolog
SQL> connect username/password@hostname:port/SERVICENAME
```

## Session Monitoring

### Basic Session Report (monitor_report_sessions.sql)

```sql
SET HEAD OFF
SET LINESIZE 200 TRIMSPOOL ON PAGESIZE 0

SELECT s.status ||','||s.serial#||','||s.TYPE||','||
       s.username||','||s.osuser||','||
       s.machine||','||s.module||','||s.client_info||','||
       s.terminal||','||s.program||','||s.action
FROM v$session s, v$process p, SYS.v_$sess_io si
WHERE s.paddr = p.addr(+) AND si.SID(+) = s.SID;
```

### Sessions with Service Name

```sql
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
FROM v$session s, v$process p
WHERE s.paddr = p.addr
AND service_name = 'TRMPS_RO_SVC'
ORDER BY s.username, s.osuser;
```

### Temp Space Usage by Sessions

```sql
COL size_mb FORMAT 999 HEAD "size_mb"
COMPUTE SUM OF size_mb ON REPORT

SELECT SUBSTR(s.sid || ',' || s.serial#,1,10) sid,
       SUBSTR(s.username,1,8) u_name,
       SUBSTR(SUM(ROUND(((u.blocks*p.value)/1024/1024),2)),1,7) size_mb,
       SUBSTR(s.osuser||','||s.machine||','||s.module||','||s.client_info||','||
         s.terminal||','||s.program||','||s.action,1,51) osUsr_Mach_mod_clinf_ter_prog
FROM v$sort_usage u,
     v$session s,
     v$sqlarea a,
     v$parameter p
WHERE s.saddr (+) = u.session_addr
  AND a.address (+) = s.sql_address
  AND a.hash_value (+) = s.sql_hash_value
  AND p.name = 'db_block_size'
GROUP BY s.sid || ',' || s.serial#,
         s.username,
         a.sql_text,
         u.tablespace,
         SUBSTR(ROUND(((u.blocks*p.value)/1024/1024),2),1,7),
         SUBSTR(s.osuser||','||s.machine||','||s.module||','||
         s.client_info||','||s.terminal||','||s.program||','||s.action,1,51);
```

### Kill Session Command Generator

```sql
SELECT 'ALTER SYSTEM KILL SESSION '''||sid||','||serial#||''' IMMEDIATE;' 
FROM gv$session 
WHERE INST_ID=1 
  AND status='INACTIVE' 
  AND LOGON_TIME BETWEEN '22-OCT-12' AND '27-NOV-12' 
GROUP BY inst_id,sid,serial#,username,osuser,machine,status,LOGON_TIME 
ORDER BY LOGON_TIME DESC;
```

## Extent and Space Monitoring

### Monitor Max Extent Issues (monitor_max_extent.sql)

```sql
-- Check segments that cannot extend due to insufficient free space
SET VERIFY OFF
SET TERMOUT OFF
SET LINESIZE 80
SET PAGESIZE 0

COLUMN db_ownr_seg_tblespace FORMAT A80

SELECT SUBSTR(SUBSTR(db.name,1,10)||' '|| SUBSTR(a.owner,1,12)||'.'||
       SUBSTR(DECODE(partition_name, NULL, segment_name, segment_name
       || ':' || partition_name),1,20)||' '||
       a.tablespace_name,1,80) db_ownr_seg_tblespace
FROM dba_segments a, v$database db,
     (SELECT df.tablespace_name, NVL(MAX(fs.bytes), 0) free,
             NVL(SUM(fs.bytes), 0) remain
      FROM dba_data_files df,
           dba_free_space fs
      WHERE df.file_id = fs.file_id (+)
      GROUP BY df.tablespace_name) b,
     (SELECT tablespace_name, MAX(maxbytes - bytes) morebytes,
             SUM(DECODE(AUTOEXTENSIBLE, 'YES', maxbytes - bytes, 0)) totalmorebytes,
             SUM(DECODE(AUTOEXTENSIBLE, 'YES', 1, 0)) autoextensible
      FROM dba_data_files
      GROUP BY tablespace_name) c
WHERE a.tablespace_name = b.tablespace_name
  AND a.tablespace_name = c.tablespace_name
  AND a.segment_type != 'ROLLBACK'
  AND a.next_extent * NVL('&1',1) > b.free;
```

### Monitor Max Extents Reached

```sql
-- Check segments close to max extents
COLUMN db_ownr_seg_tblespace FORMAT A80

SELECT SUBSTR(SUBSTR(db.name,1,10)||' '|| SUBSTR(ds.owner,1,12)||'.'||
       SUBSTR(DECODE(ds.partition_name, NULL, ds.segment_name, ds.segment_name ||
       ':' || ds.partition_name),1,20)||' '||
       ds.tablespace_name,1,80) db_ownr_seg_tblespace
FROM dba_segments ds, v$database db
WHERE ds.extents >= ds.max_extents - NVL('&1',1)
  AND ds.segment_type IN ('TABLE','INDEX','LOBINDEX','LOBSEGMENT',
                         'TABLE PARTITION','INDEX PARTITION','ROLLBACK','CLUSTER')
ORDER BY 1;
```

## Resource and Performance Monitoring

### Resource Limit Monitoring (monitor_resource_limit.sql)

```sql
-- Monitor resources within specified percentage of capacity
SET VERIFY OFF
SET TERMOUT OFF
SET LINESIZE 80
SET PAGESIZE 0

SELECT SUBSTR(name,1,10)||' '|| SUBSTR(resource_name,1,30)||' '||
       ROUND(CURRENT_UTILIZATION/LIMIT_VALUE*100,2)
FROM (SELECT * FROM v$resource_limit
      WHERE TRIM(limit_value) <> 'UNLIMITED'), v$database
WHERE TO_NUMBER(current_utilization) > TO_NUMBER(limit_value) * (NVL('&1',1)/100)
  AND resource_name <> 'max_rollback_segments'
  AND resource_name NOT LIKE 'gcs%'
UNION
SELECT SUBSTR(name,1,10)||' '|| SUBSTR(resource_name,1,30)||' '||
       ROUND(CURRENT_UTILIZATION/LIMIT_VALUE*100,2)
FROM (SELECT * FROM v$resource_limit
      WHERE TRIM(limit_value) <> 'UNLIMITED'), v$database
WHERE TO_NUMBER(current_utilization) > TO_NUMBER(limit_value) * (NVL('&2',1)/100)
  AND resource_name IN ('processes','sessions')
ORDER BY 1;
```

## Tablespace Monitoring

### Tablespace Usage (monitor_tablespace.sql)

```sql
-- Report tablespaces with low free space
SET VERIFY OFF
SET TERMOUT OFF
SET LINESIZE 80
SET PAGESIZE 0
SET FEEDBACK ON

COL tspace FORM A25 HEADING "Tablespace"
COL tot_ts_size FORM 99999999999999 HEADING "Size(Mb) "
COL free_ts_size FORM 99999999999999 HEADING "Free_(Mb)"
COL ts_pct FORM 999 HEADING "%Free "

SELECT df.tablespace_name tspace,
       df.bytes/(1024*1024) tot_ts_size,
       fs.free_space/(1024*1024) free_ts_size,
       ROUND(((df.bytes - (df.allocated - fs.free_space))/df.bytes) * 100) tc_pct
FROM (SELECT tablespace_name, SUM(bytes) free_space
      FROM dba_free_space
      GROUP BY tablespace_name) fs,
     (SELECT tablespace_name,
             SUM(DECODE(autoextensible,'NO',bytes,maxbytes)) bytes,
             SUM(user_bytes) allocated
      FROM dba_data_files
      GROUP BY tablespace_name) df
WHERE fs.tablespace_name = df.tablespace_name
  AND fs.tablespace_name NOT IN ('TEMP','ROLLBACK','UNDOTBS1','UNDOTBS2','UNDOTBS3')
  AND ((df.bytes - (df.allocated - fs.free_space))/df.bytes) * 100 < &1;
```

### Tablespace Status (monitor_tablespace_status.sql)

```sql
-- Report offline tablespaces and invalid files
SET VERIFY OFF
SET TERMOUT OFF
SET LINESIZE 80
SET PAGESIZE 0

COLUMN tblespace_filename_status FORMAT A80

SELECT vi.instance_name||' '||ddf.tablespace_name||' '||ddf.file_name||' '||ddf.status 
       instnce_tblspce_filnm_status
FROM dba_data_files ddf, v$instance vi
WHERE ddf.status <> 'AVAILABLE'
  AND tablespace_name NOT LIKE ('%RBS%')
UNION
SELECT vi.instance_name||' '||dt.tablespace_name||'        '||dt.status 
       instnce_tblspce_filnm_status
FROM dba_tablespaces dt, v$instance vi
WHERE dt.status <> 'ONLINE';
```

### Temp Tablespace Monitoring

```sql
-- Monitor temp tablespace usage
COLUMN db_ownr_seg_tblespace FORMAT A80

SELECT SUBSTR(u.tablespace,1,15) tablespace,
       SUM(ROUND(((u.blocks*p.value)/1024/1024),2)) ts_used_mb, 
       '(tablespace MbUsed)'
FROM v$sort_usage u, v$parameter p
WHERE p.name = 'db_block_size'
GROUP BY u.tablespace
HAVING SUM(u.blocks) > (SELECT (SUM(DECODE(autoextensible,'NO',blocks,maxblocks))*(NVL('&1',1)/100))
                        FROM dba_temp_files dtf
                        WHERE dtf.tablespace_name = u.tablespace);
```

## Lock Monitoring

### RAC Blocking Locks (monitor_locks.sql)

```sql
SET LINESIZE 145
SET PAGESIZE 9999

COLUMN locking_instance FORMAT A17 HEAD 'LOCKING|Instance - SID' JUST LEFT
COLUMN waiting_instance FORMAT A17 HEAD 'WAITING|Instance - SID' JUST LEFT
COLUMN waiter_lock_type HEAD 'Waiter Lock Type' JUST LEFT
COLUMN waiter_mode_req HEAD 'Waiter Mode Req.' JUST LEFT

SELECT ih.instance_name || ' - ' || lh.sid locking_instance,
       iw.instance_name || ' - ' || lw.sid waiting_instance,
       DECODE(lh.type,
              'CF', 'Control File',
              'DX', 'Distrted Transaction',
              'FS', 'File Set',
              'IR', 'Instance Recovery',
              'IS', 'Instance State',
              'IV', 'Libcache Invalidation',
              'LS', 'LogStartORswitch',
              'MR', 'Media Recovery',
              'RT', 'Redo Thread',
              'RW', 'Row Wait',
              'SQ', 'Sequence #',
              'ST', 'Diskspace Transaction',
              'TE', 'Extend Table',
              'TT', 'Temp Table',
              'TX', 'Transaction',
              'TM', 'Dml',
              'UL', 'PLSQL User_lock',
              'UN', 'User Name',
              'Nothing-') waiter_lock_type,
       DECODE(lw.request,
              0, 'None',
              1, 'NoLock',
              2, 'Row-Share',
              3, 'Row-Exclusive',
              4, 'Share-Table',
              5, 'Share-Row-Exclusive',
              6, 'Exclusive',
              'Nothing-') waiter_mode_req
FROM gv$lock lw, gv$lock lh, gv$instance iw, gv$instance ih
WHERE iw.inst_id = lw.inst_id
  AND ih.inst_id = lh.inst_id
  AND lh.id1 = lw.id1
  AND lh.id2 = lw.id2
  AND lh.request = 0
  AND lw.lmode = 0
  AND (lh.id1, lh.id2) IN (SELECT id1,id2 FROM gv$lock WHERE request = 0
                           INTERSECT
                           SELECT id1,id2 FROM gv$lock WHERE lmode = 0)
ORDER BY lh.sid;
```

## Long Running Sessions

### Monitor Long Sessions (monitor_long_sessions.sql)

```sql
-- Monitor sessions running longer than threshold (in seconds)
SET LINESIZE 132
SET PAGESIZE 64
SET FEEDBACK OFF
SET TRIMSPOOL ON

COL sid FORMAT 999 HEAD "Sid"
COL serial# FORMAT 99999 HEAD "Serial"
COL username FORMAT A8 HEAD "ORA User"
COL last_call_et FORMAT 9999999 HEAD "ExecSecs"
COL logon_time FORMAT A16 HEAD "Logged_On_Since"

SELECT s.sid, s.serial#, p.spid, s.username,
       RPAD(s.action,8) "OS User",
       RPAD(s.machine,8) "Source",
       s.process,
       s.last_call_et,
       TO_CHAR(s.logon_time,'DD/MM/YYYY HH24:MI') "Logged_On_Since",
       s.status, s.module
FROM v$session s, v$process p
WHERE status IN ('ACTIVE','SNIPED')
  AND s.type != 'BACKGROUND'
  AND s.last_call_et > '&1'
  AND s.paddr = p.addr
  AND s.lockwait IS NULL
ORDER BY s.last_call_et, s.sid;

-- SQL Text for long running sessions
SET HEADING OFF
SELECT t.sql_text
FROM v$session s, V$sqltext_with_newlines t
WHERE status IN ('ACTIVE','SNIPED')
  AND s.type != 'BACKGROUND'
  AND s.last_call_et > '&1'
  AND s.lockwait IS NULL
  AND s.sql_address = t.address
  AND s.sql_hash_value = t.hash_value
ORDER BY s.last_call_et, s.sid, t.piece;
```

## DBMS Jobs Monitoring

### Monitor Failed Jobs (monitor_dbms_jobs.sql)

```sql
-- Report failing or broken DBMS jobs
SELECT job, priv_user, failures,
       DECODE(TO_CHAR(next_date,'J'),'3182030','** Disabled/Broken **',
              TO_CHAR(next_date,'dd-Mon-yyyy hh24:mi:ss')) overdue_next_date
FROM dba_jobs dj
WHERE (dj.next_date < (SYSDATE - 1800/86400) -- allow 30 min for snp latency
   OR dj.failures > 0)
  AND NOT EXISTS (SELECT NULL FROM dba_jobs_running
                  WHERE job = dj.job);
```

## Data Guard Monitoring

### Data Guard Status (monitor_dataguard.sql)

```sql
-- Monitor Data Guard status
SET VERIFY OFF
SET TERMOUT OFF
SET LINESIZE 180
SET PAGESIZE 60
SET TRIMSPOOL ON

ALTER SESSION SET nls_date_format = 'DD-MON-YY hh24:mi:ss';

-- Database and instance info
SELECT d.name, i.status, d.database_role, i.host_name
FROM v$database d, v$instance i;

-- Check archive gaps
SELECT * FROM v$archive_gap;

-- Archive logs applied in last two days
SELECT sequence#, name, first_time, next_time, completion_time, applied, status
FROM v$archived_log
WHERE first_time > CASE TO_CHAR(SYSDATE,'DY')
                   WHEN 'MON' THEN TRUNC(SYSDATE - 3,'dd')
                   ELSE TRUNC(SYSDATE - 1,'dd')
                   END
ORDER BY sequence#;

-- Check for errors
SELECT timestamp, message, severity
FROM v$dataguard_status
WHERE UPPER(severity) IN ('ERROR','FATAL');

-- Apply process status
SELECT process, status, sequence#
FROM v$managed_standby
WHERE process LIKE 'MRP%';
```

## Archive Log Monitoring

### Archive Generation Scripts

**archive_gen.sh**
```bash
#!/bin/bash
ps -ef |grep pmon |grep -v +ASM |grep -v "grep pmon"|cut -f3 -d_ |while read dblist
do
    export ORACLE_SID=$dblist
    echo $ORACLE_SID
    export ORACLE_HOME=`cat /etc/oratab|grep ^$dblist |awk -F: {'print $2'}`
    export spoolf1="$dblist"_audit.lst
    $ORACLE_HOME/bin/sqlplus -s "/as sysdba" << EOF
spool $spoolf1
select name from v\$database;
@arch_gen.sql
@arch_gen2.sql
spool off
exit
EOF
done
```

**arch_gen.sql**
```sql
SELECT A.*, ROUND(A.Count#*B.AVG#/1024/1024) Daily_Avg_Mb
FROM (SELECT TO_CHAR(First_Time,'YYYY-MM-DD') DAY,
             COUNT(1) Count#,
             MIN(RECID) Min#,
             MAX(RECID) Max# 
      FROM v$log_history 
      GROUP BY TO_CHAR(First_Time,'YYYY-MM-DD') 
      ORDER BY 1 DESC) A,
     (SELECT AVG(BYTES) AVG#,COUNT(1) Count#,
             MAX(BYTES) Max_Bytes,MIN(BYTES) Min_Bytes
      FROM v$log) B;
```

**arch_gen2.sql**
```sql
SELECT SUM(GB_USED_PER_DAY)/COUNT(GB_USED_PER_DAY) 
FROM (SELECT TO_CHAR(completion_time,'YYYY-MM-DD') completion_date,
             ROUND(SUM(block_size*(blocks+1)) / 1024 / 1024 / 1024, 2) GB_USED_PER_DAY
      FROM v$archived_log
      WHERE TRUNC(completion_time) BETWEEN TRUNC(SYSDATE-30) AND TRUNC(SYSDATE)
      GROUP BY TO_CHAR(completion_time,'YYYY-MM-DD')
      ORDER BY 1 DESC);
```

## Housekeeping Scripts

### Oracle Database Home Cleanup (oracle_housekeeping.sh)

```bash
#!/bin/sh
# Oracle housekeeping script for cleanup of log, trace, audit and core files

if [ "$1" ] ; then
    ORACLE_SID=$1
    export ORACLE_SID
else
    echo "ERROR: Database name not passed as parameter... Aborting"
    exit 1
fi

# Local variables
OUTFILE=/tmp/oracle_housekeeping_${ORACLE_SID}.out
LOGDIR=/var/oracle/cron/${ORACLE_SID}
LOGFILE=${LOGDIR}/oracle_housekeeping_${ORACLE_SID}_`date +%Y%m%d_%H%M`.log
KEEPTIME=30

# Redirect output to logfile
exec > ${LOGFILE} 2>&1

echo "ORACLE HOUSEKEEPING for $1 Started `date`"

# Set Oracle environment
ORAENV_ASK=NO
export ORAENV_ASK
. /usr/local/bin/oraenv

# Check if database is running
if [ -z "`ps -ef | grep ora_pmon_${ORACLE_SID} | grep -v grep`" ] ; then
    echo "Database is not running, cleanup skipped"
    exit 1
fi

# Get directory locations from database
${ORACLE_HOME}/bin/sqlplus -s /nolog << EOF
connect / as sysdba
set echo off heading off pages 0 lines 200
spool ${OUTFILE}
select name,value
from v\$parameter
where name in ('audit_file_dest',
               'user_dump_dest',
               'background_dump_dest',
               'core_dump_dest');
spool off
EOF

if [ ! -s ${OUTFILE} ] ; then
    echo "Unable to determine directory names, cleanup skipped"
    exit 1
fi

# Cleanup audit files
echo "Cleaning up audit files for ${ORACLE_SID}"
AUDIT_DIR=`grep "^audit_file_dest" ${OUTFILE} | awk '{print $2}'`
AUDIT_DIR=`echo ${AUDIT_DIR} | sed s!\?!${ORACLE_HOME}!`
echo "  directory: " ${AUDIT_DIR}
find ${AUDIT_DIR} -name "*.aud" -mtime +${KEEPTIME} -ls -exec rm {} \;

# Cleanup core files
echo "Cleaning up core files for ${ORACLE_SID}"
CORE_DIR=`grep "^core_dump_dest" ${OUTFILE} | awk '{print $2}'`
CORE_DIR=`echo ${CORE_DIR} | sed s!\?!${ORACLE_HOME}!`
echo "  directory: " ${CORE_DIR}
find ${CORE_DIR} -name "core_*" -ls -exec rm -r {} \;

# Cleanup trace files
echo "Cleaning up trace files for ${ORACLE_SID}"
TRACE_DIR=`grep "^user_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`
echo "  directory: " ${TRACE_DIR}
find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;
find ${TRACE_DIR} -name "*.trm" -mtime +${KEEPTIME} -ls -exec rm {} \;

TRACE_DIR=`grep "^background_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`
find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;
find ${TRACE_DIR} -name "*.trm" -mtime +${KEEPTIME} -ls -exec rm {} \;

# Cleanup alert log files
echo "Cleaning up alert log files for ${ORACLE_SID}"
cd ${TRACE_DIR}
DESTFILE=alert_${ORACLE_SID}_`date '+%Y%m%d'`
cp alert_${ORACLE_SID}.log $DESTFILE
cat /dev/null > alert_${ORACLE_SID}.log
compress $DESTFILE
find ${TRACE_DIR} -name "alert*.Z" -mtime +365 -ls -exec rm {} \;

# Cleanup log files from this job
find ${LOGDIR} -name "oracle_housekeeping_${ORACLE_SID}*.log" -mtime +36 -ls -exec rm {} \;

echo "ORACLE HOUSEKEEPING Finished `date`"
exit
```

### Grid Infrastructure Cleanup (grid_housekeeping.sh)

```bash
#!/bin/sh
# Grid Infrastructure housekeeping script

if [ "$1" ] ; then
    ORACLE_SID=$1
    export ORACLE_SID
else
    echo "ERROR: Database name not passed as parameter... Aborting"
    exit 1
fi

# Local variables
OUTFILE=/tmp/grid_housekeeping_${ORACLE_SID}.out
LOGDIR=/var/oracle/cron/asm
LOGFILE=${LOGDIR}/grid_housekeeping_${ORACLE_SID}_`date +%Y%m%d_%H%M`.log
KEEPTIME=15

exec > ${LOGFILE} 2>&1

echo "ORACLE HOUSEKEEPING for $1 Started `date`"

# Set Oracle environment
ORAENV_ASK=NO
export ORAENV_ASK
. /usr/local/bin/oraenv

# Cleanup trace files
echo "Cleaning up trace files for ${ORACLE_SID}"
TRACE_DIR=`grep "^user_dump_dest" ${OUTFILE} | awk '{print $2}'`
TRACE_DIR=`echo ${TRACE_DIR} | sed s!\?!${ORACLE_HOME}!`
find ${TRACE_DIR} -name "*.trc" -mtime +${KEEPTIME} -ls -exec rm {} \;

# Cleanup listener log files
echo "Cleaning up listener log files"
cd ${TNS_ADMIN:-${ORACLE_HOME}/network/admin}
LISTENER_DIR=`grep LOG_DIRECTORY listener.ora | awk '{print $NF}'`

if [ -d ${LISTENER_DIR} ] ; then
    cd ${LISTENER_DIR}
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

echo "ORACLE HOUSEKEEPING Finished `date`"
exit
```

### Manual Trace File Cleanup

```bash
# Replace with correct paths from parameter file
find /<db_home>/admin/<SID>/bdump -name "*.trc" -a -mtime +30 -exec rm -rf {} \;
find /<db_home>/admin/<SID>/udump -name "*.trc" -a -mtime +30 -exec rm -rf {} \;
find /<db_home>/admin/<SID>/adump -name "*.aud" -a -mtime +30 -exec rm -rf {} \;

# Example for 11g+ diagnostic destination
find /app/wspsp/product/diag/rdbms/eaym/eaym2/trace -name "*.trc" -a -mtime +30 -exec rm -rf {} \;
find /app/wspsp/product/admin/eaym/adump -name "*.aud" -a -mtime +30 -exec rm -rf {} \;
```

## ASM Monitoring

### ASM Process Monitor (process_asm.sh)

```bash
#!/bin/bash
ps -ef | grep -i asm | wc -l >> /tmp/session.out
export proc=`ps -ef | grep -i asm | wc -l`
if [ "$proc" -gt 900 ]; then
    mailx -s "Process threshold has breached for ASM `hostname` at `date`" \
          "admin@example.com" < /tmp/session.out
fi
```

## Backup Status Reports

### RMAN Backup Status View

```sql
CREATE OR REPLACE FORCE VIEW "SYS"."RMAN_REPORT"
("DB_NAME","INPUT_TYPE", "STATUS", "START_TIME", "END_TIME", "HRS", 
 "SUM_BYTES_BACKED_IN_GB","SUM_BACKUP_PIECES_IN_GB", "OUTPUT_DEVICE_TYPE") AS
SELECT a.NAME,
       INPUT_TYPE,
       STATUS,
       TO_CHAR(START_TIME,'DD-MON-YYYY-HH24:mi:ss') start_time,
       TO_CHAR(END_TIME,'DD-MON-YYYY-HH24:mi:ss') end_time,
       ELAPSED_SECONDS/3600 hrs,
       INPUT_BYTES/1024/1024/1024 SUM_BYTES_BACKED_IN_GB,
       OUTPUT_BYTES/1024/1024/1024 SUM_BACKUP_PIECES_IN_GB,
       OUTPUT_DEVICE_TYPE
FROM V$RMAN_BACKUP_JOB_DETAILS, v$database a
ORDER BY a.name, SESSION_KEY;
```

### Daily Backup Status View

```sql
CREATE OR REPLACE FORCE VIEW "RMAN_REPORT_DAILY"
("DB_NAME", "INPUT_TYPE", "STATUS", "START_TIME", "END_TIME") AS
SELECT name,INPUT_TYPE,STATUS,
       TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM v$database,V$RMAN_BACKUP_JOB_DETAILS 
WHERE status IN ('COMPLETED','COMPLETED WITH WARNINGS') 
  AND INPUT_TYPE IN ('DB FULL','DB INCR') 
  AND OUTPUT_DEVICE_TYPE='SBT_TAPE' 
  AND END_TIME=(SELECT MAX(END_TIME) FROM V$RMAN_BACKUP_JOB_DETAILS 
                WHERE STATUS IN ('COMPLETED','COMPLETED WITH WARNINGS') 
                  AND INPUT_TYPE IN ('DB FULL','DB INCR') 
                  AND OUTPUT_DEVICE_TYPE='SBT_TAPE')
UNION
SELECT name,INPUT_TYPE,STATUS,
       TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM V$RMAN_BACKUP_JOB_DETAILS,v$database 
WHERE status IN ('COMPLETED','COMPLETED WITH WARNINGS') 
  AND INPUT_TYPE='ARCHIVELOG' 
  AND OUTPUT_DEVICE_TYPE='SBT_TAPE' 
  AND END_TIME=(SELECT MAX(END_TIME) FROM V$RMAN_BACKUP_JOB_DETAILS 
                WHERE STATUS IN ('COMPLETED','COMPLETED WITH WARNINGS') 
                  AND INPUT_TYPE='ARCHIVELOG' 
                  AND OUTPUT_DEVICE_TYPE='SBT_TAPE');

-- Grant access to specific user (replace &usr with actual username)
GRANT SELECT ON sys.RMAN_REPORT_DAILY TO &usr;
CREATE SYNONYM cscbkp.RMAN_REPORT_DAILY FOR sys.RMAN_REPORT_DAILY;
```

## CRS Resource Status

### CRS Status Script (crs_stat.sh)

```bash
#!/usr/bin/ksh
# 10g CRS resource status query script
# Usage: ./crs_stat.sh [resource_key]

RSC_KEY=$1
QSTAT=-u
AWK=/usr/bin/awk

# Table header
echo ""
$AWK 'BEGIN {printf "%-45s %-10s %-18s\n", "HA Resource", "Target", "State";
             printf "%-45s %-10s %-18s\n", "-----------", "------", "-----";}'

# Table body
/opt/oracle/11.2.0.2_grid/grid/bin/crs_stat $QSTAT | $AWK \
'BEGIN { FS="="; state = 0; }
$1~/NAME/ && $2~/'$RSC_KEY'/ {appname = $2; state=1};
state == 0 {next;}
$1~/TARGET/ && state == 1 {apptarget = $2; state=2;}
$1~/STATE/ && state == 2 {appstate = $2; state=3;}
state == 3 {printf "%-45s %-10s %-18s\n", appname, apptarget, appstate; state=0;}'
```

### Database Health Check Script (health.sql)

```sql
SET LINES 200 PAGES 2000
SELECT instance_name, status, archiver FROM v$instance;
SELECT * FROM v$recover_file;
ARCHIVE LOG LIST;
SELECT flashback_on FROM v$database;
```

### Run All Databases Script (run_all.sh)

```bash
#!/bin/sh
set -vx
cd /home/oracle/mm

cat /etc/oratab | while read LINE
do
    case $LINE in
    \#* ) ;;      # comment-line in oratab
    * )
        ORACLE_SID=`echo $LINE | awk -F: '{print $1}' -`
        if [ "$ORACLE_SID" = '*' ] ; then
            ORACLE_SID=""
        fi
        export ORACLE_SID
        echo $ORACLE_SID
        ORACLE_HOME=`echo $LINE | awk -F: '{print $2}' -`
        export ORACLE_HOME
        export PATH=$PATH:$ORACLE_HOME/bin:.
        export SID=$ORACLE_SID
        echo $SID |. oraenv 1>/dev/null 2>&1
        
        sqlplus -S "/ as sysdba" <<EOF
spool ${SID}_alert.log
@@/home/oracle/mm/show_alert.sql
spool off;
exit
EOF
    ;;
    esac
done
```

## Memory and Performance

### Memory Usage by Sessions

```sql
SET PAGES 500 LINES 110 TRIMS ON
CLEAR COL
COL NAME FORMAT A30
COL username FORMAT A20
BREAK ON username NODUP SKIP 1

SELECT vses.username||':'||vsst.SID||','||vses.serial# username, 
       vstt.NAME, 
       MAX(vsst.VALUE) VALUE
FROM v$sesstat vsst, v$statname vstt, v$session vses
WHERE vstt.statistic# = vsst.statistic# 
  AND vsst.SID = vses.SID 
  AND vstt.NAME IN ('session pga memory','session pga memory max',
                    'session uga memory','session uga memory max',
                    'session cursor cache count','session cursor cache hits',
                    'session stored procedure space','opened cursors current',
                    'opened cursors cumulative') 
  AND vses.username IS NOT NULL
GROUP BY vses.username, vsst.SID, vses.serial#, vstt.NAME
ORDER BY vses.username, vsst.SID, vses.serial#, vstt.NAME;
```

### Transaction and Rollback Information

```sql
COLUMN RBS_NAME FORMAT A10

SELECT '    Rollback Used                : '||t.used_ublk*8192/1024/1024 ||' M'          || CHR(10) ||
       '    Rollback Records             : '||t.used_urec        || CHR(10)||
       '    Rollback Segment Number      : '||t.xidusn           || CHR(10)||
       '    Rollback Segment Name        : '||r.name             || CHR(10)||
       '    Logical IOs                  : '||t.log_io           || CHR(10)||
       '    Physical IOs                 : '||t.phy_io           || CHR(10)||
       '    RBS Starting Extent ID       : '||t.start_uext       || CHR(10)||
       '    Transaction Start Time       : '||t.start_time       || CHR(10)||
       '    Transaction Status           : '||t.status
FROM v$transaction t, v$session s, v$rollname r
WHERE t.addr = s.taddr
  AND r.usn = t.xidusn
  AND s.sid = &sid_number;
```

### Sort Information

```sql
COLUMN username FORMAT A20
COLUMN user FORMAT A20
COLUMN tablespace FORMAT A20

SELECT '    Sort Space Used (8k blocks assumed)     : '||u.blocks/1024*8 ||' M'     || CHR(10) ||
       '    Sorting Tablespace                      : '||u.tablespace    || CHR(10)||
       '    Sort Tablespace Type                    : '||u.contents      || CHR(10)||
       '    Total Extents Used for Sorting          : '||u.extents
FROM v$session s, v$sort_usage u
WHERE s.saddr = u.session_addr
  AND s.sid = &sid_number;
```

### Performance Statistics (performance_stat.sql)

```sql
COL sid FORMAT 9999
COL username FORMAT A10
COL osuser FORMAT A10
COL program FORMAT A25
COL process FORMAT 9999999
COL spid FORMAT 999999
COL logon_time FORMAT A13
SET LINES 150
SET HEADING OFF
SET VERIFY OFF
SET FEEDBACK OFF

UNDEFINE sid_number
UNDEFINE spid_number
COL sid NEW_VALUE sid_number NOPRINT
COL spid NEW_VALUE spid_number NOPRINT

SELECT s.sid sid, p.spid spid
FROM v$session s, v$process p
WHERE s.sid LIKE NVL('&sid', '%')
  AND p.spid LIKE NVL ('&OS_ProcessID', '%')
  AND s.process LIKE NVL('&Client_Process', '%')
  AND s.paddr = p.addr;

-- Session and Process Information
SELECT '    SID                         : '||v.sid      || CHR(10)||
       '    Serial Number               : '||v.serial#  || CHR(10) ||
       '    Oracle User Name            : '||v.username || CHR(10) ||
       '    Client OS user name         : '||v.osuser   || CHR(10) ||
       '    Client Process ID           : '||v.process  || CHR(10) ||
       '    Client machine Name         : '||v.machine  || CHR(10) ||
       '    Oracle PID                  : '||p.pid      || CHR(10) ||
       '    OS Process ID(spid)         : '||p.spid     || CHR(10) ||
       '    Session Status              : '||v.status   || CHR(10) ||
       '    Logon Time                  : '||TO_CHAR(v.logon_time, 'MM/DD HH24:MIpm') || CHR(10) ||
       '    Program Name                : '||v.program  || CHR(10)
FROM v$session v, v$process p
WHERE v.paddr = p.addr
  AND v.serial# > 1
  AND p.background IS NULL
  AND p.username IS NOT NULL
  AND sid = &sid_number
ORDER BY logon_time, v.status, 1;

-- SQL Statement
SELECT sql_text
FROM v$sqltext, v$session
WHERE v$sqltext.address = v$session.sql_address
  AND sid = &sid_number
ORDER BY piece;

-- Event Wait Information
SELECT '   SID '|| &sid_number ||' is waiting on event  : ' || x.event || CHR(10) ||
       '   P1 Text                      : ' || x.p1text || CHR(10) ||
       '   P1 Value                     : ' || x.p1 || CHR(10) ||
       '   P2 Text                      : ' || x.p2text || CHR(10) ||
       '   P2 Value                     : ' || x.p2 || CHR(10) ||
       '   P3 Text                      : ' || x.p3text || CHR(10) ||
       '   P3 Value                     : ' || x.p3
FROM v$session_wait x
WHERE x.sid= &sid_number;

-- Session Statistics
SELECT '     '|| b.name ||'             : '||
       DECODE(b.name, 'redo size', ROUND(a.value/1024/1024,2)||' M', a.value)
FROM v$session s, v$sesstat a, v$statname b
WHERE a.statistic# = b.statistic#
  AND name IN ('redo size', 'parse count (total)', 'parse count (hard)', 'user commits')
  AND s.sid = &sid_number
  AND a.sid = &sid_number
ORDER BY DECODE(b.name, 'redo size', 1, 2), b.name;
```

### SQL Session History

```sql
SPOOL hist_sql_DEV.lst
UNDEFINE sql_id
PROMPT 'Please enter SQL_ID: (ie: 0sfvmtydgh4qan)'
DEFINE sql_id='&SQL_ID'

SET NULL NULL
SET LINES 320
SET PAGES 99
SET TRIMSPOOL ON

-- Column formatting
COL snap_beg FORMAT A12
COL iowait_delta FORMAT 99999999.99 HEADING io|wait|delta|(ms)
COL ELAPSED_TIME_DELTA FORMAT 99999999.99 HEADING elapsd|time|delta|(ms)
COL CPU_TIME_DELTA FORMAT 99999999.99 HEADING cpu|time|delta|(ms)
COL PLAN_HASH_VALUE HEADING plan_hash|value
COL executions_delta FORMAT 999999 HEADING exec|delta
COL buffer_gets_delta FORMAT 99999999999 HEADING buffer|gets|delta
COL disk_reads_delta FORMAT 9999999999 HEADING disk|reads|delta
COL rows_processed_delta FORMAT 999999999 HEADING rows|processed|delta

SELECT dba_hist_sqlstat.instance_number,
       sql_id,
       plan_hash_value,
       dba_hist_sqlstat.snap_id,
       TO_CHAR(dba_hist_snapshot.BEGIN_INTERVAL_TIME,'dd-mm hh24:mi') snap_beg,
       invalidations_delta,
       parse_calls_delta,
       executions_delta,
       elapsed_time_delta/1000 elapsed_time_delta,
       cpu_time_delta/1000 cpu_time_delta,
       iowait_delta/1000 iowait_delta,
       CASE WHEN executions_delta = 0 THEN NULL
            WHEN elapsed_time_delta = 0 THEN NULL
            ELSE (elapsed_time_delta/executions_delta)/1000
       END ela_ex,
       SUBSTR(SQL_PROFILE,1,32) sql_profile
FROM dba_hist_sqlstat, dba_hist_snapshot
WHERE sql_id='&&sql_id'
  AND dba_hist_sqlstat.snap_id=dba_hist_snapshot.snap_id
  AND dba_hist_sqlstat.instance_number=dba_hist_snapshot.instance_number
ORDER BY dba_hist_sqlstat.instance_number, plan_hash_value, dba_hist_sqlstat.snap_id;

-- Execution Plans
SELECT plan_table_output 
FROM TABLE(dbms_xplan.display_awr('&&sql_id',NULL, NULL, 'ADVANCED +PEEKED_BINDS'));

SELECT plan_table_output 
FROM TABLE(dbms_xplan.display_cursor('&&sql_id', NULL, 'ADVANCED +PEEKED_BINDS'));

SPOOL OFF
```

## Utility Scripts

### Default Profile Information

```sql
SPOOL profile_commands.sql
SELECT 'ALTER USER ' ||u.username|| ' PROFILE '||u.profile||';' 
FROM sys.dba_users u;
SPOOL OFF
```

### Password Backup Script

```sql
-- PasswordBackup_<db>_<date>.sql
SET LINES 120
SET PAGES 999
SET VER OFF HEAD OFF FEED OFF

SELECT 'ALTER USER '|| name ||' IDENTIFIED BY VALUES '''||
       DECODE(spare4,NULL,password,spare4)||''';' sql
FROM sys.user$;
```

## Usage Examples

### Execute Housekeeping Scripts

```bash
# Oracle Database Home cleanup
/var/oracle/cron/oracle_housekeeping.sh EAYM > /var/oracle/cron/logs/oracle_housekeeping.out 2>/dev/null

# Grid Infrastructure cleanup  
/var/oracle/cron/grid_housekeeping.sh +ASM1 > /var/oracle/cron/logs/grid_housekeeping.out 2>/dev/null

# ASM Process monitoring
/export/home/oracle/bin/monitor/process_asm.sh >> /export/home/oracle/bin/monitor/logs/monitor_ASM_process.log
```

### Query Backup Status

```sql
SET LINES 200 PAGES 2000
SELECT * FROM rman_report;
SELECT * FROM rman_report_daily;
```

## Script Categories Summary

| Category | Scripts | Purpose |
|----------|---------|---------|
| **Session Management** | monitor_report_sessions.sql, monitor_long_sessions.sql | Monitor and analyze database sessions |
| **Space Management** | monitor_max_extent.sql, monitor_tablespace.sql | Monitor storage and space issues |
| **Performance** | performance_stat.sql, SQL session history | Analyze performance metrics and SQL |
| **Maintenance** | oracle_housekeeping.sh, grid_housekeeping.sh | Automated cleanup and maintenance |
| **Monitoring** | monitor_locks.sql, monitor_dataguard.sql | Monitor locks, Data Guard status |
| **Backup** | RMAN report views | Monitor backup status and history |
| **RAC/Grid** | crs_stat.sh, process_asm.sh | Monitor cluster resources and ASM |

## Prerequisites

- Oracle Database 10g or higher
- Appropriate privileges (SYSDBA for most scripts)
- Shell access for housekeeping scripts
- Proper environment setup (ORACLE_HOME, ORACLE_SID)

## Important Notes

1. **Test First**: Always test scripts in a development environment before production use
2. **Modify Paths**: Update file paths, email addresses, and thresholds according to your environment
3. **Permissions**: Ensure proper file system and database permissions
4. **Scheduling**: Consider using cron for automated execution of housekeeping scripts
5. **Monitoring**: Set up alerting based on script outputs for proactive monitoring

## License and Disclaimer

These scripts are provided as-is for educational and operational purposes. Please review and modify according to your organization's requirements and Oracle licensing agreements.
