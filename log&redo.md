````markdown
# Oracle Database Operations Wiki

This wiki provides an overview of key Oracle database operations related to redo log management, rollback segment monitoring, and undo tablespace management.

## Table of Contents
- [Redo Log Information](#redo-log-information)
- [Redo Log Archiving and Status](#redo-log-archiving-and-status)
- [Log History and Rollback Segment Monitoring](#log-history-and-rollback-segment-monitoring)
- [Undo Tablespace Management](#undo-tablespace-management)
- [Archive Generation Statistics](#archive-generation-statistics)
- [RAC Logfile Group Operations](#rac-logfile-group-operations)

---

## Redo Log Information

The following SQL commands display details about the redo log groups, sequences, and their respective statuses:

```sql
-- Display REDO Log Information
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
SET VERIFY OFF

COL group# HEA "Group" FOR 99
COL sequence# HEA "Sequence" FOR 9,999,999,999
COL bytes HEA "Datafile Size (bytes)" FOR 999,999,999,999,999
COL members HEA "Member" FOR 999
COL member HEA "Datafile Name" FOR a44
COL archived HEA "Arc?" FOR a4
COL status HEA "Status" FOR a16
COL first_changed# HEA "First Changed No." FOR 999,999,999,999
COL first_time HEA "First Time" FOR a20

SELECT l.group#, l.members, l.sequence#, lf.member, l.bytes,
       l.status, l.archived, l.first_change#, To_Char(l.first_time,'DD-Mon-YYYY HH24:MI:SS') as first_time
FROM v$log l, v$logfile lf
WHERE l.group# = lf.group#;
````

---

## Redo Log Archiving and Status

Displays the status and archive status of each redo log group.

```sql
SELECT group#, status, archived
FROM v$log;
```

---

## Log History and Rollback Segment Monitoring

### Log History Statistics

This query generates a report on the `v$log_history` table, showing a breakdown of log usage by hour.

```sql
SELECT trunc(first_time) "Date",
       to_char(first_time, 'Dy') "Day",
       count(1) "Total",
       SUM(DECODE(to_char(first_time, 'hh24'),'00',1,0)) "h0",
       ...
FROM v$log_history
GROUP BY trunc(first_time), to_char(first_time, 'Dy')
ORDER BY 1;
```

### Rollback Segment Usage

This query shows detailed information about the rollback segments in use, including the session ID, username, and the amount of space utilized.

```sql
SELECT s.username,
       s.sid,
       s.serial#,
       t.used_ublk,
       t.used_urec,
       rs.segment_name,
       r.rssize,
       r.status
FROM   v$transaction t,
       v$session s,
       v$rollstat r,
       dba_rollback_segs rs
WHERE  s.saddr = t.ses_addr
AND    t.xidusn = r.usn
AND    rs.segment_id = t.xidusn
ORDER BY t.used_ublk DESC;
```

---

## Undo Tablespace Management

The following actions help manage and monitor the undo tablespace:

1. **Set Undo Retention:**
   Update the undo retention parameter to ensure that undo data is kept for a longer period.

   ```sql
   ALTER SYSTEM SET UNDO_RETENTION = 2800 scope=both sid='*';
   ```

2. **Monitor Undo Tablespace Usage:**
   Query to see statistics about undo tablespace usage.

   ```sql
   SELECT max(maxquerylen) 
   FROM v$undostat;
   ```

3. **Increase Undo Tablespace Size:**
   Add two additional datafiles to the active UNDO tablespace.

   ```sql
   ALTER TABLESPACE undotbs1 ADD DATAFILE '/oracle/oradata/undo02.dbf' SIZE 20G;
   ```

---

## Archive Generation Statistics

This section tracks the total archive logs generated, grouped by day, and their total size.

```sql
SELECT to_char(COMPLETION_TIME, 'DD/MON/YYYY') Day,
       SUM(blocks * block_size) / 1048576 / 1024 AS "Size(GB)",
       COUNT(sequence#) AS "Total Archives"
FROM v$archived_log
GROUP BY to_char(COMPLETION_TIME, 'DD/MON/YYYY')
ORDER BY Day;
```

---

## RAC Logfile Group Operations

Perform operations on Oracle RAC log file groups, such as disabling threads, dropping logfile groups, and checking thread statuses.

```sql
-- Drop logfiles
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;

-- Check thread status
SELECT THREAD#, STATUS, ENABLED 
FROM v$thread;
```

---

### Additional Information

* **Monitoring Redo Log Capacity**:
  Queries to monitor redo log file usage and the current size of redo log groups.

```sql
SELECT THREAD#, GROUP#, STATUS, BYTES / 1048576 "MBytes", ARCHIVED, MEMBER
FROM v$log l, v$logfile lf
WHERE l.GROUP# = lf.GROUP#
ORDER BY THREAD#, GROUP#, lf.MEMBER;
```

---

## Notes

* Ensure that any changes made to redo log files or rollback segments are done during maintenance windows, as it can affect database availability.
* The undo retention settings and undo tablespace size adjustments should be carefully managed to avoid excessive space consumption or retention issues.

