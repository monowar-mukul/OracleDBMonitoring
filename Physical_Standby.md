# Physical Standby Database Monitoring and Maintenance

## Table of Contents
1. [Checking the Database Mode](#checking-the-database-mode)
2. [Checking Dataguard Transport Lag](#checking-dataguard-transport-lag)
    - [Current Transport Lag](#current-transport-lag)
    - [Dataguard Metrics for RAC Databases](#dataguard-metrics-for-rac-databases)
    - [Raw Stats for Transport Lag](#raw-stats-for-transport-lag)
    - [1-Hour Rollup Stats for Transport Lag](#1-hour-rollup-stats-for-transport-lag)
3. [Monitoring Dataguard Status](#monitoring-dataguard-status)
    - [Dataguard Errors](#dataguard-errors)
    - [Listener Verification on Primary DB](#listener-verification-on-primary-db)
    - [Finding Archive Gaps](#finding-archive-gaps)
    - [DR Database Alert Log Error Check](#dr-database-alert-log-error-check)
    - [Checking Log Transfer and Apply Status](#checking-log-transfer-and-apply-status)
    - [Calculating Log Gaps](#calculating-log-gaps)
4. [Common Monitoring Tasks (Both Physical and Logical Standby)](#common-monitoring-tasks-both-physical-and-logical-standby)
    - [Check the Log File on Standby Server](#check-the-log-file-on-standby-server)
    - [Check the Type of Standby Database](#check-the-type-of-standby-database)
    - [Sequence and Applied Logs Status](#sequence-and-applied-logs-status)
    - [Last Sequence Received and Applied](#last-sequence-received-and-applied)
    - [Check Current Log Sequence](#check-current-log-sequence)
5. [Checking Standby Log Usage](#checking-standby-log-usage)
6. [Checking Recovery Mode of Standby](#checking-recovery-mode-of-standby)
    - [Real-Time Apply Standby](#real-time-apply-standby)
    - [Non-Realtime Apply Standby](#non-realtime-apply-standby)
7. [Gap Checking Script](#gap-checking)
8. [Checking Log Archive Differences Between Threads](#checking-log-archive-differences-between-threads)
9. [Monitoring Standby Database Time Difference](#monitoring-standby-database-time-difference)

---

## Checking the Database Mode

### Data Guard Configuration

```sql
SELECT open_mode,
       database_role,
       protection_mode,
       protection_level,
       standby_became_primary_scn
FROM v$database;
```
---

## Checking Dataguard Transport Lag

### Current Transport Lag
Run the following SQL query to check the transport lag:

```sql
select name, value from v$dataguard_stats
where name = 'transport lag';
```
### Data Guard Transport and Apply Lag

```sql
SELECT name,
       value,
       unit
FROM v$dataguard_stats
WHERE name IN ('transport lag', 'apply lag');
```
### Managed Recovery Status

```sql
SELECT process,
       status,
       client_process,
       sequence#,
       block#,
       blocks,
       delay_mins
FROM v$managed_standby;
```

### Dataguard Metrics for RAC Databases
To find Dataguard-related metrics for RAC databases:

```sql
select distinct column_label from mgmt_metrics where target_type = 'rac_database' and metric_name like 'dataguard%';
```

### Raw Stats for Transport Lag
To find the raw stats for transport lag:

```sql
select 
t.target_name,
m.column_label,
mmr.collection_timestamp,
mmr.value
from 
mgmt_targets t,
mgmt_metrics m,
mgmt_metrics_raw mmr
where t.target_guid = mmr.target_guid
and m.metric_guid = mmr.metric_guid
and m.column_label = 'Transport Lag (seconds)'
and t.target_name = '&DB';
```

### 1-Hour Rollup Stats for Transport Lag
For 1-hour rollup stats for transport lag:

```sql
select 
t.target_name,
m.column_label,
mm1.rollup_timestamp,
mm1.value_average,
mm1.value_minimum,
mm1.value_maximum
from 
mgmt_targets t,
mgmt_metrics m,
mgmt_metrics_1hour mm1
where t.target_guid = mm1.target_guid
and m.metric_guid = mm1.metric_guid
and m.column_label = 'Transport Lag (seconds)'
and t.target_name = '&DB';
```

---

## Monitoring Dataguard Status

### Dataguard Errors
Check for any errors in the Dataguard status:

```sql
set pages 999 lines 999
col MESSAGE for a100
select to_char(timestamp,'YYYY-MON-DD HH24:MI:SS')||' '||message||severity 
from gv$dataguard_status 
where severity in ('Error','Fatal') 
order by timestamp;
```

### Listener Verification on Primary DB
Check the listener status on the primary database:

```sql
select dest_id,status,error from v$archive_dest where dest_name='LOG_ARCHIVE_DEST_2';
```

### Finding Archive Gaps
To find gaps in the archive logs:

```sql
select thread#, low_sequence#, high_sequence# from gv$archive_log;
```

### DR Database Alert Log Error Check
Check errors in the DR database alert log:

```sql
set pages 999 lines 999
col MESSAGE for a100
select to_char(timestamp,'YYYY-MON-DD HH24:MI:SS')||' '||message||severity 
from gv$dataguard_status 
where severity in ('Error','Fatal') 
order by timestamp;
```

### Checking Log Transfer and Apply Status
To check the log transfer and apply status:

```sql
SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME, APPLIED FROM gV$ARCHIVED_LOG ORDER BY SEQUENCE#;
select count(*) from GV$ARCHIVED_LOG where applied='NO';
```
```sql
select max(al.sequence#) "Last Seq Recieved", max(lh.sequence#) "Last Seq Applied"
      from v$archived_log al, v$log_history lh;
```

### Calculating Log Gaps
To calculate the total log gap:

```sql
select sum(local.sequence#-target.sequence#) Total_gap
from 
(select thread#, max(sequence#) sequence# from gv$archived_log 
where standby_dest='YES' and applied='YES' 
and first_time between sysdate-7 and sysdate 
group by thread#) target,
(select thread#, max(sequence#) sequence# from gv$log group by thread#) local
where target.thread#=local.thread#;
```

---

## Common Monitoring Tasks (Both Physical and Logical Standby)

### Check the Log File on Standby Server
If the standby database is falling behind, check the log file:

```shell
tail -f /log/oracle/log/dataguard_check.<instance>.log
```

### Check the Type of Standby Database
Run the following query to check if the standby database is physical or logical:

```sql
select * from (
SELECT SEQUENCE#, APPLIED, completion_time
FROM V$ARCHIVED_LOG
ORDER BY SEQUENCE# desc)
Where rownum <= 10;
```

### Sequence and Applied Logs Status
Check the latest sequence number applied and completed:

```sql
select * from (
select SEQUENCE#, first_time, To_char(first_time, 'dd-mon-yy hh24:mi') as Completed 
from V$log_history
order by sequence# desc)
Where rownum <= 10;
```

### Last Sequence Received and Applied
To check the last sequence received and applied:

```sql
SQL> SELECT al.thrd "Thread", almax "Last Seq Received", lhmax "Last Seq Applied" 
FROM
  (select thread# thrd, MAX(sequence#) almax FROM v$archived_log 
   WHERE resetlogs_change#=(SELECT resetlogs_change# FROM v$database) 
   GROUP BY thread#) al, 
  (SELECT thread# thrd, MAX(sequence#) lhmax FROM v$log_history 
   WHERE resetlogs_change#=(SELECT resetlogs_change# FROM v$database) 
   GROUP BY thread#) lh 
WHERE al.thrd = lh.thrd;
```

### Check Current Log Sequence
To check the current log sequence:

```sql
SQL> select thread#, sequence# from v$log where status='CURRENT';
```

---

## Checking Standby Log Usage
Check which standby logs are being used:

```sql
set lines 155 pages 9999
col thread# for 9999990 
col sequence# for 999999990 
col grp for 990 
col fnm for a50 head "File Name" 
col "Fisrt SCN Number" for 999999999999990 
break on thread 
# skip 1 
select a.thread#, a.sequence#, a.group# grp, a.bytes/1024/1024 Size_MB,
       a.status, a.archived, a.first_change# "First SCN Number",
       to_char(FIRST_TIME,'DD-Mon-RR HH24:MI:SS') "First SCN Time", 
       to_char(LAST_TIME,'DD-Mon-RR HH24:MI:SS') "Last SCN Time" 
from v$standby_log a 
order by 1, 2, 3, 4;
```

---

## Checking Recovery Mode of Standby

### Real-Time Apply Standby
To check if the standby is using real-time apply mode:

```sql
SQLPLUS> select recovery_mode from V$ARCHIVE_DEST_STATUS;
```

### Non-Realtime Apply Standby
If not using real-time apply, run the following to apply logs manually:

```sql
SQLPLUS> alter database recover managed standby database disconnect from session;
```

---

## Gap Checking Script

```sql
column applied_time for a30
column instance_name form a10
set linesize 140
select to_char(sysdate,'dd-mm-yyyy hh24:mi:ss') "Current Time" from dual;

SELECT INSTANCE_NAME, APPLIED_TIME, LOG_ARCHIVED-LOG_APPLIED LOG_GAP, 
(case when ((APPLIED_TIME is not null and (LOG_ARCHIVED-LOG_APPLIED) is null) or
            (APPLIED_TIME is null and (LOG_ARCHIVED-LOG_APPLIED) is not null) or
            ((LOG_ARCHIVED-LOG_APPLIED) > 5))
      then 'Error! Log Gap is '
      else 'OK!'
 end) Status
FROM
(
SELECT INSTANCE_NAME INSTANCE_NAME
FROM GV$INSTANCE where INST_ID = 1
),
(
SELECT MAX(SEQUENCE#) LOG_ARCHIVED
FROM V$ARCHIVED_LOG WHERE DEST_ID=1 AND ARCHIVED='YES' and THREAD#=1
),
(
SELECT MAX(SEQUENCE#) LOG_APPLIED
FROM V$ARCHIVED_LOG WHERE DEST_ID=2 AND APPLIED='YES' and THREAD#=1
),
(
SELECT TO_CHAR(MAX(COMPLETION_TIME),'DD-MON/HH24:MI') APPLIED_TIME
FROM V$ARCHIVED_LOG WHERE DEST_ID=2 AND APPLIED='YES' and THREAD#=1
)
```

---

## Checking Log Archive Differences Between Threads

```sql
SELECT a.thread#, b.last_seq, a.applied_seq, a.last_app_timestamp, b.last_seq-a.applied_seq ARC_DIFF
FROM (SELECT thread#, MAX(sequence#) applied_seq, MAX(next_time) last_app_timestamp
      FROM gv$archived_log WHERE applied = 'YES' GROUP BY thread#) a,
     (SELECT thread#, MAX(sequence#) last_seq FROM gv$archived_log GROUP BY thread#) b
WHERE a.thread# = b.thread#;
```

---

## Monitoring Standby Database Time Difference

Run the following PL/SQL block to check the time difference between primary and standby:

```plsql
DECLARE
   v_diff NUMBER := 0;
   v_hrs NUMBER := 0;
   v_min NUMBER := 0;
   v_sec NUMBER := 0;
   p_dte1 DATE;
   p_dte2 DATE;
   date1 long;
   date2 long;
BEGIN
   date1 := 'select sysdate from dual';
   date2 := 'select max(TIMESTAMP) from v$recovery_progress';
   execute immediate date1 into p_dte1;
   execute immediate date2 into p_dte2;
   v_diff := ABS(p_dte2 - p_dte1);
   v_hrs := TRUNC(v_diff, 0)*24;
   v_diff := (v_diff - TRUNC(v_diff, 0))*24;
   v_hrs := v_hrs + TRUNC(v_diff, 0);
   v_diff := (v_diff - TRUNC(v_diff, 0))*60;
   v_min := TRUNC(v_diff, 0);
   v_sec := TRUNC((v_diff - TRUNC(v_diff, 0))*60, 0);
   DBMS_OUTPUT.put_line(CHR(10)||'The time difference between primary and standby database is '||
   TO_CHAR(v_hrs,'00') ||':'|| TO_CHAR(v_min,'00') ||':'|| TO_CHAR(v_sec,'00') ||CHR(10));
END;
```

---

## Quick Reference Cheat Sheet

| Task Description                              | SQL Query or Command                                                 |
|------------------------------------------------|----------------------------------------------------------------------|
| **Check database mode**                       | `select open_mode, database_role from v$database;`                    |
| **Check transport lag**                       | `select name, value from v$dataguard_stats where name = 'transport lag';` |
| **Dataguard error check**                     | `select to_char(timestamp,'YYYY-MON-DD HH24:MI:SS')||' '||message||severity from gv$dataguard_status where severity in ('Error','Fatal')` |
| **Find archive gaps**                         | `select thread#, low_sequence#, high_sequence# from gv$archive_log;`  |
| **Check recovery mode**                       | `select recovery_mode from V$ARCHIVE_DEST_STATUS;`                    |
| **Gap checking script**                       | Execute `Chk_gap.sql` script                                           |
| **Log sequence checking**                     | `select thread#, sequence# from v$log where status='CURRENT';`        |

---
