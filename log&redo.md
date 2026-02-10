# Oracle Database log management

This wiki provides comprehensive guidance for Oracle database operations including redo log management, rollback segment monitoring, undo tablespace management, and archiving operations.

## Table of Contents
- [Redo Log Management](#redo-log-management)
- [Archive Log Operations](#archive-log-operations)
- [Rollback Segment Monitoring](#rollback-segment-monitoring)
- [Undo Tablespace Management](#undo-tablespace-management)
- [RAC Operations](#rac-operations)
- [Monitoring and Statistics](#monitoring-and-statistics)

## Alert Log [ MM - in progress] 
```sql
 
-- ===== SQL*Plus / SQLcl output formatting =====
set pagesize 200
set linesize 240
set trimout on
set trims on
set trimspool on
set tab off
set wrap off
set underline '='

-- Column headings + widths
column inst  heading 'Inst'          format a12
column host  heading 'Host'          format a18
column cid   heading 'Con|ID'        format 9999
column sev   heading 'Sev'           format 999
column typ   heading 'Type'          format 999
column ts    heading 'Timestamp'     format a19
column msg   heading 'Message'       format a160 word_wrapped

-- ===== RAC-wide query (all instances) =====
SELECT
  inst,
  host,
  cid,
  sev,
  typ,
  ts,
  msg
FROM TABLE(
  gv$(CURSOR(
    SELECT
      (SELECT instance_name FROM v$instance)                 AS inst,
      host_id                                                AS host,
      con_id                                                 AS cid,
      message_level                                          AS sev,
      message_type                                           AS typ,
      TO_CHAR(originating_timestamp,'YYYY-MM-DD HH24:MI:SS') AS ts,
      message_text                                           AS msg
    FROM v$diag_alert_ext
    WHERE originating_timestamp >= SYSTIMESTAMP - INTERVAL '60' MINUTE
      AND (
            message_text LIKE '%ORA-%'
         OR message_text LIKE '%TNS-%'
         OR message_text LIKE '%WARNING%'
         OR message_text LIKE '%ERROR%'
         OR message_text LIKE '%INCIDENT%'
          )
  ))
)
ORDER BY ts DESC, inst, host, cid;
```  
## Redo Log Management

### Display Redo Log Information

Comprehensive view of redo log groups, sequences, and status:

```sql
-- Configure display settings
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
SET VERIFY OFF

-- Format columns
COL group# HEA "Group" FOR 99
COL sequence# HEA "Sequence" FOR 9,999,999,999
COL bytes HEA "Datafile Size (bytes)" FOR 999,999,999,999,999
COL members HEA "Member" FOR 999
COL member HEA "Datafile Name" FOR A44
COL archived HEA "Arc?" FOR A4
COL status HEA "Status" FOR A16
COL first_changed# HEA "First Changed No." FOR 999,999,999,999
COL first_time HEA "First Time" FOR A20

-- Main query
SELECT l.group#, 
       l.members, 
       l.sequence#, 
       lf.member, 
       l.bytes,
       l.status, 
       l.archived, 
       l.first_change#, 
       TO_CHAR(l.first_time,'DD-Mon-YYYY HH24:MI:SS') AS first_time
FROM v$log l, v$logfile lf
WHERE l.group# = lf.group#
ORDER BY l.group#;
```

### Monitor Redo Log Capacity

Query to monitor redo log file usage across threads and groups:

```sql
SELECT THREAD#, 
       GROUP#, 
       STATUS, 
       BYTES / 1048576 AS "Size_MB", 
       ARCHIVED, 
       MEMBER
FROM v$log l, v$logfile lf
WHERE l.GROUP# = lf.GROUP#
ORDER BY THREAD#, GROUP#, lf.MEMBER;
```

### Quick Redo Log Status Check

Simple status overview of redo log groups:

```sql
SELECT group#, status, archived
FROM v$log
ORDER BY group#;
```

## Archive Log Operations

### Archive Generation Statistics

Track archive log generation by day with size metrics:

```sql
SELECT TO_CHAR(COMPLETION_TIME, 'DD/MON/YYYY') AS Day,
       ROUND(SUM(blocks * block_size) / 1024 / 1024 / 1024, 2) AS "Size_GB",
       COUNT(sequence#) AS "Total_Archives"
FROM v$archived_log
WHERE COMPLETION_TIME IS NOT NULL
GROUP BY TO_CHAR(COMPLETION_TIME, 'DD/MON/YYYY')
ORDER BY TO_DATE(Day, 'DD/MON/YYYY') DESC;
```

### Archive Log History by Hour

Detailed hourly breakdown of archive log generation:

```sql
SELECT TRUNC(first_time) AS "Date",
       TO_CHAR(first_time, 'Dy') AS "Day",
       COUNT(1) AS "Total",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'00',1,0)) AS "00h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'01',1,0)) AS "01h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'02',1,0)) AS "02h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'03',1,0)) AS "03h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'04',1,0)) AS "04h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'05',1,0)) AS "05h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'06',1,0)) AS "06h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'07',1,0)) AS "07h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'08',1,0)) AS "08h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'09',1,0)) AS "09h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'10',1,0)) AS "10h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'11',1,0)) AS "11h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'12',1,0)) AS "12h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'13',1,0)) AS "13h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'14',1,0)) AS "14h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'15',1,0)) AS "15h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'16',1,0)) AS "16h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'17',1,0)) AS "17h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'18',1,0)) AS "18h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'19',1,0)) AS "19h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'20',1,0)) AS "20h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'21',1,0)) AS "21h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'22',1,0)) AS "22h",
       SUM(DECODE(TO_CHAR(first_time, 'HH24'),'23',1,0)) AS "23h"
FROM v$log_history
GROUP BY TRUNC(first_time), TO_CHAR(first_time, 'Dy')
ORDER BY 1 DESC;
```

## Rollback Segment Monitoring

### Active Transaction Rollback Usage

Monitor rollback segment usage by active transactions:

```sql
SELECT s.username,
       s.sid,
       s.serial#,
       t.used_ublk AS "Used_Undo_Blocks",
       t.used_urec AS "Used_Undo_Records",
       rs.segment_name,
       ROUND(r.rssize / 1024 / 1024, 2) AS "Rollback_Size_MB",
       r.status
FROM v$transaction t,
     v$session s,
     v$rollstat r,
     dba_rollback_segs rs
WHERE s.saddr = t.ses_addr
  AND t.xidusn = r.usn
  AND rs.segment_id = t.xidusn
ORDER BY t.used_ublk DESC;
```

## Undo Tablespace Management

### Set Undo Retention

Configure undo retention policy (in seconds):

```sql
-- Set undo retention to 2800 seconds (46.7 minutes)
ALTER SYSTEM SET UNDO_RETENTION = 2800 SCOPE=BOTH SID='*';
```

### Monitor Maximum Query Length

Check the longest-running query requiring undo data:

```sql
SELECT MAX(maxquerylen) AS "Max_Query_Length_Seconds"
FROM v$undostat;
```

### Monitor Current Undo Statistics

Detailed undo tablespace statistics:

```sql
SELECT BEGIN_TIME,
       END_TIME,
       UNDOTSN,
       UNDOBLKS,
       TXNCOUNT,
       MAXQUERYLEN,
       MAXCONCURRENCY,
       UNXPSTEALCNT,
       UNXPBLKRELCNT,
       UNXPBLKREUCNT
FROM v$undostat
WHERE BEGIN_TIME >= SYSDATE - 1
ORDER BY BEGIN_TIME DESC;
```

### Expand Undo Tablespace

Add datafiles to increase undo tablespace capacity:

```sql
-- Add new datafile to undo tablespace
ALTER TABLESPACE undotbs1 
ADD DATAFILE '/oracle/oradata/undo02.dbf' SIZE 20G;

-- Add with autoextend enabled
ALTER TABLESPACE undotbs1 
ADD DATAFILE '/oracle/oradata/undo03.dbf' SIZE 10G 
AUTOEXTEND ON NEXT 1G MAXSIZE 32G;
```

### Monitor Undo Tablespace Usage

Check current undo tablespace space utilization:

```sql
SELECT tablespace_name,
       ROUND(SUM(bytes) / 1024 / 1024 / 1024, 2) AS "Total_GB",
       ROUND(SUM(DECODE(maxbytes, 0, bytes, maxbytes)) / 1024 / 1024 / 1024, 2) AS "Max_GB",
       COUNT(*) AS "Datafiles"
FROM dba_data_files
WHERE tablespace_name LIKE '%UNDO%'
GROUP BY tablespace_name;
```

## RAC Operations

### Check Thread Status

Monitor Oracle RAC thread status:

```sql
SELECT THREAD#, 
       STATUS, 
       ENABLED,
       GROUPS,
       INSTANCE,
       OPEN_TIME,
       CURRENT_GROUP#
FROM v$thread
ORDER BY THREAD#;
```

### Drop Logfile Groups

Remove logfile groups (use with caution):

```sql
-- Ensure logfile groups are not current or active before dropping
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;
```

### Add Logfile Groups for RAC

Add new logfile groups for RAC instances:

```sql
-- Add logfile group for thread 1
ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 4 
('/oracle/oradata/redo04a.log', '/oracle/oradata/redo04b.log') SIZE 1G;

-- Add logfile group for thread 2
ALTER DATABASE ADD LOGFILE THREAD 2 GROUP 5 
('/oracle/oradata/redo05a.log', '/oracle/oradata/redo05b.log') SIZE 1G;
```

## Monitoring and Statistics

### Database Performance Overview

Quick database health check focusing on redo and undo:

```sql
SELECT 'Redo Log Switches (Last 24h)' AS Metric,
       COUNT(*) AS Value
FROM v$log_history
WHERE first_time >= SYSDATE - 1
UNION ALL
SELECT 'Current Undo Retention (sec)' AS Metric,
       VALUE AS Value
FROM v$parameter
WHERE NAME = 'undo_retention'
UNION ALL
SELECT 'Active Transactions' AS Metric,
       COUNT(*) AS Value
FROM v$transaction;
```

### Log Switch Frequency Analysis

Analyze log switch patterns to optimize redo log sizing:

```sql
SELECT TO_CHAR(first_time, 'YYYY-MM-DD HH24') AS "Hour",
       COUNT(*) AS "Log_Switches",
       ROUND(AVG(COUNT(*)) OVER (ORDER BY TO_CHAR(first_time, 'YYYY-MM-DD HH24') 
                                 ROWS BETWEEN 23 PRECEDING AND CURRENT ROW), 1) AS "24h_Avg"
FROM v$log_history
WHERE first_time >= SYSDATE - 7
GROUP BY TO_CHAR(first_time, 'YYYY-MM-DD HH24')
ORDER BY 1 DESC;
```

## Best Practices and Notes

### Maintenance Guidelines

- **Redo Log Operations**: Always perform during maintenance windows as they can affect database availability
- **Undo Retention**: Balance retention time with storage consumption - monitor actual query requirements
- **Archive Log Management**: Implement regular archive log cleanup to prevent filesystem issues
- **RAC Considerations**: Ensure logfile groups are properly distributed across nodes for optimal performance

### Monitoring Recommendations

1. **Daily Checks**:
   - Monitor archive log generation rates
   - Check undo tablespace utilization
   - Verify log switch frequency

2. **Weekly Reviews**:
   - Analyze rollback segment usage patterns
   - Review undo retention effectiveness
   - Check for any dropped or corrupted logfiles

3. **Monthly Tasks**:
   - Evaluate redo log sizing based on switch frequency
   - Review archive log storage growth trends
   - Assess RAC thread distribution and performance

### Troubleshooting Tips

- **High Log Switch Frequency**: Consider increasing redo log size
- **ORA-01555 Errors**: Increase undo retention or undo tablespace size
- **Archive Log Destination Full**: Implement automated cleanup or increase storage
- **Inactive Threads in RAC**: Check instance status and thread enablement

### Security Considerations

- Ensure proper permissions for undo tablespace operations
- Monitor for unauthorized transaction rollbacks
- Implement audit trails for redo log modifications
- Secure archive log destinations from unauthorized access
