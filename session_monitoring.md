# Oracle Session Monitoring Queries

A comprehensive collection of SQL queries and scripts for monitoring Oracle database sessions, analyzing performance bottlenecks, and identifying resource-intensive operations.

## Table of Contents

- [Active Session History (ASH) Queries](#active-session-history-ash-queries)
- [Current Running SQL Queries](#current-running-sql-queries)
- [Long-Running Session Monitoring](#long-running-session-monitoring)
- [Detailed Session Information](#detailed-session-information)

## Active Session History (ASH) Queries

### Most Active Sessions in the Last Hour

Identifies the most active sessions based on SQL ID and calculates percentage load for each.

```sql
SELECT sql_id, COUNT(*), ROUND(COUNT(*) / SUM(COUNT(*)) OVER(), 2) PCTLOAD
FROM gv$active_session_history
WHERE sample_time > SYSDATE - 1/24
AND session_type = 'BACKGROUND'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
### Active Sessions by Status

```sql
SELECT sid,
       SUBSTR(username, 1, 20) usrname,
       SUBSTR(osuser, 1, 20) osuser,
       SUBSTR(program, 1, 20) pgram,
       server,
       status
FROM v$session
WHERE username IS NOT NULL
  AND status = 'ACTIVE'
ORDER BY server, status, sid;
```

### Top 10 Longest-Active User Sessions

```sql
SELECT *
FROM (
    SELECT TO_CHAR(s.logon_time, 'mm/dd hh:mi:ssAM') loggedon,
           s.sid,
           s.serial#,
           s.status,
           FLOOR(last_call_et/60) "LastCallET",
           s.username,
           s.osuser,
           p.spid,
           s.module || '_' || s.program uprogram,
           s.machine,
           s.sql_hash_value
    FROM v$session s,
         v$process p
    WHERE p.addr = s.paddr
      AND s.type = 'USER'
      AND module IS NOT NULL
      AND s.status = 'ACTIVE'
    ORDER BY last_call_et DESC
)
WHERE ROWNUM < 11;
```

### Sessions Running for More Than 1 Hour

```sql
SELECT username,
       machine,
       inst_id,
       sid,
       serial#,
       program,
       TO_CHAR(logon_time, 'dd-mm-yy hh:mi:ss AM') "Logon Time",
       ROUND((SYSDATE - logon_time) * (24 * 60), 1) AS minutes_logged_on,
       ROUND(last_call_et/60, 1) AS minutes_for_current_sql
FROM gv$session
WHERE status = 'ACTIVE'
  AND username IS NOT NULL
  AND ROUND((SYSDATE - logon_time) * (24 * 60), 1) > 60
ORDER BY minutes_logged_on DESC;
```
### Wait Events for Specific Session

Retrieves wait events and wait times for a specific session (use session ID and serial number from previous query).

```sql
SELECT sample_time, event, wait_time
FROM gv$active_session_history
WHERE session_id = &1
AND session_serial# = &2;
```

### Most I/O Intensive SQL (Last Hour)

Identifies SQL queries with the highest I/O waits in the last hour.

```sql
SELECT sql_id, COUNT(*)
FROM gv$active_session_history ash, gv$event_name evt
WHERE ash.sample_time > SYSDATE - 1/24
AND ash.session_state = 'WAITING'
AND ash.event_id = evt.event_id
AND evt.wait_class = 'User I/O'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
### Detailed Session Information by SID

```sql
SELECT 'Session Id: ' || s.sid,
       'Serial Num: ' || s.serial#,
       'User Name: ' || s.username,
       'Session Status: ' || s.status,
       'Client Process Id: ' || s.process,
       'Server Process ID: ' || p.spid,
       'Sql_Address: ' || s.sql_address,
       'Sql_hash_value: ' || s.sql_hash_value,
       'Schema Name: ' || s.schemaname,
       'Program: ' || s.program,
       'Module: ' || s.module,
       'Action: ' || s.action,
       'Terminal: ' || s.terminal,
       'Client Machine: ' || s.machine,
       'LAST_CALL_ET: ' || s.last_call_et,
       'LAST_CALL_ET Hours: ' || s.last_call_et/3600
FROM v$session s,
     v$process p
WHERE p.addr = s.paddr
  AND s.sid = NVL('&sid', s.sid);
```
### Top SQLs by Resource Consumption

Aggregates SQL performance metrics showing CPU usage, wait time, and I/O operations.

```sql
SELECT
  ash.SQL_ID,
  SUM(DECODE(ash.session_state, 'ON CPU', 1, 0)) "CPU",
  SUM(DECODE(ash.session_state, 'WAITING', 1, 0)) -
    SUM(DECODE(ash.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "WAIT",
  SUM(DECODE(ash.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "IO",
  SUM(DECODE(ash.session_state, 'ON CPU', 1, 1)) "TOTAL"
FROM v$active_session_history ash, v$event_name en
WHERE ash.SQL_ID IS NOT NULL
AND en.event# = ash.event#
GROUP BY ash.SQL_ID
ORDER BY "TOTAL" DESC;
```

### Session Analysis for Specific Time Period

Analyzes active session history for a particular session within a specified time range.

```sql
SELECT C.SQL_TEXT,
       B.NAME,
       COUNT(*),
       SUM(TIME_WAITED)
FROM v$ACTIVE_SESSION_HISTORY A,
     v$EVENT_NAME B,
     v$SQLAREA C
WHERE A.SAMPLE_TIME BETWEEN '&starttime' AND '&endtime'
AND A.EVENT# = B.EVENT#
AND A.SESSION_ID = &sid
AND A.SQL_ID = C.SQL_ID
GROUP BY C.SQL_TEXT, B.NAME;
```

### Top CPU Consuming Sessions (Last 15 Minutes)

Returns the top 10 sessions with highest CPU consumption in the last 15 minutes.

```sql
SELECT * FROM (
  SELECT s.username, s.module, s.sid, s.serial#, s.sql_id, COUNT(*)
  FROM v$active_session_history h, v$session s
  WHERE h.session_id = s.sid
  AND h.session_serial# = s.serial#
  AND session_state = 'ON CPU'
  AND sample_time > sysdate - interval '15' minute
  GROUP BY s.username, s.module, s.sid, s.serial#, s.sql_id
  ORDER BY COUNT(*) DESC
)
WHERE rownum <= 10;
```
### Session Details with CPU and Wait Events

```sql
SELECT a.sid,
       a.value session_cpu,
       SUBSTR(d.event, 1, 30) evento,
       d.seconds_in_wait
FROM v$sesstat a,
     v$statname b,
     v$sess_io c,
     v$session_wait d
WHERE b.name = 'CPU used by this session'
  AND a.statistic# = b.statistic#
  AND a.sid = c.sid
  AND a.sid = d.sid
  AND a.sid IN (&sid_list);
```
### Current Running SQLs with Session Details

```sql
SET PAGES 50000 LINES 32767
COLUMN host_name FORMAT A20
COLUMN event FORMAT A40
COLUMN machine FORMAT A30
COLUMN sql_text FORMAT A50
COLUMN username FORMAT A15

SELECT sid,
       serial#,
       a.sql_id,
       a.sql_text,
       s.username,
       i.host_name,
       machine,
       s.event,
       s.seconds_in_wait sec_wait,
       TO_CHAR(logon_time, 'DD-MON-RR HH24:MI') login
FROM gv$session s,
     gv$sqlarea a,
     gv$instance i
WHERE s.username IS NOT NULL
  AND s.sql_address = a.address
  AND s.inst_id = a.inst_id
  AND i.inst_id = a.inst_id
  AND sql_text NOT LIKE '%gv$session%';
```

### Last/Latest Running SQL

```sql
SELECT t.inst_id,
       s.username,
       s.sid,
       s.serial#,
       t.sql_id,
       t.sql_text "Last SQL"
FROM gv$session s,
     gv$sqlarea t
WHERE s.sql_address = t.address
  AND s.sql_hash_value = t.hash_value;
```
### Historical Query Analysis (Last 30 Days)

Retrieves SQL queries executed in the last 30 days from historical data.

```sql
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
WHERE sample_time >= SYSDATE - 30
AND h.user_id = u.user_id
AND h.sql_id = s.sql_id
ORDER BY h.sample_time;
```

### Historical Query Analysis (Specific Time Range)

Retrieves SQL queries executed within a specific date/time range.

```sql
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
WHERE sample_time BETWEEN TO_DATE('01/10/2024 19:45:00', 'DD/MM/YYYY HH24:MI:SS')
AND TO_DATE('01/10/2024 19:50:00', 'DD/MM/YYYY HH24:MI:SS')
AND h.user_id = u.user_id
AND h.sql_id = s.sql_id
ORDER BY h.sample_time;
```

## Current Running SQL Queries

### Currently Executing SQL Statements

Displays details of SQL queries currently being executed across all instances.

```sql
SELECT sid, serial#, a.sql_id, a.SQL_TEXT, S.USERNAME, i.host_name, machine, S.event, S.seconds_in_wait sec_wait,
       TO_CHAR(logon_time, 'DD-MON-RR HH24:MI') login
FROM gv$session S, gv$SQLAREA A, gv$instance i
WHERE S.username IS NOT NULL
AND S.sql_address = A.address
AND s.inst_id = a.inst_id
AND i.inst_id = a.inst_id
AND sql_text NOT LIKE 'select S.USERNAME, S.seconds_in_wait%';
```

## Long-Running Session Monitoring

### Active Sessions Running Over 1 Hour

Identifies sessions that have been active for more than 60 minutes.

```sql
SELECT USERNAME, machine, inst_id, sid, serial#, PROGRAM,
       TO_CHAR(logon_time, 'dd-mm-yy hh:mi:ss AM') "Logon Time",
       ROUND((SYSDATE - LOGON_TIME) * (24 * 60), 1) as MINUTES_LOGGED_ON,
       ROUND(LAST_CALL_ET / 60, 1) as Minutes_FOR_CURRENT_SQL
FROM gv$session
WHERE STATUS = 'ACTIVE'
AND USERNAME IS NOT NULL
AND ROUND((SYSDATE - LOGON_TIME) * (24 * 60), 1) > 60
ORDER BY MINUTES_LOGGED_ON DESC;
```

### Active Sessions Running Over 30 Minutes

Identifies sessions that have been active for more than 30 minutes.

```sql
SELECT USERNAME, machine, PROGRAM, TERMINAL,
       TO_CHAR(logon_time, 'dd-mm-yy hh:mi:ss AM') "Logon Time",
       ROUND((SYSDATE - LOGON_TIME) * (24 * 60), 1) as MINUTES_LOGGED_ON,
       ROUND(LAST_CALL_ET / 60, 1) as Minutes_FOR_CURRENT_SQL
FROM v$session
WHERE STATUS = 'ACTIVE'
AND USERNAME IS NOT NULL
AND ROUND((SYSDATE - LOGON_TIME) * (24 * 60), 1) > 30
ORDER BY MINUTES_LOGGED_ON DESC;
```

## Detailed Session Information
### Sessions by Machine

```sql
SELECT COUNT(1),
       machine
FROM gv$session
WHERE inst_id = '&inst_id'
GROUP BY machine;
```

### Session and Process Counts

```sql
SELECT (SELECT COUNT(*) FROM v$session) sessions,
       (SELECT COUNT(*) FROM v$process) processes
FROM dual;
```

### Find Session by SPID

```sql
SELECT sid,
       serial#,
       username,
       status,
       osuser,
       process,
       machine,
       module,
       action,
       TO_CHAR(logon_time, 'yyyy-mm-dd hh24:mi:ss')
FROM v$session
WHERE paddr IN (
    SELECT addr
    FROM v$process
    WHERE spid = '&spid'
);
```
### Comprehensive Session Details

Provides detailed information for a specific session ID (SID), including process IDs, SQL details, and session metadata.

```sql
SET HEAD OFF
SET VERIFY OFF
SET ECHO OFF
SET PAGES 1500
SET LINESIZE 100
SET LINES 120

PROMPT
PROMPT Details of SID / SPID / Client PID
PROMPT ==================================

SELECT /*+ CHOOSE */
  'Session Id.............................................: ' || s.sid,
  'Serial Num..............................................: ' || s.serial#,
  'User Name ..............................................: ' || s.username,
  'Session Status .........................................: ' || s.status,
  'Client Process Id on Client Machine ....................: ' || '*' || s.process || '*' Client,
  'Server Process ID ......................................: ' || p.spid Server,
  'Sql_Address ............................................: ' || s.sql_address,
  'Sql_hash_value .........................................: ' || s.sql_hash_value,
  'Schema Name ..... ......................................: ' || s.SCHEMANAME,
  'Program ...............................................: ' || s.program,
  'Module .................................................: ' || s.module,
  'Action .................................................: ' || s.action,
  'Terminal ...............................................: ' || s.terminal,
  'Client Machine .........................................: ' || s.machine,
  'LAST_CALL_ET ...........................................: ' || s.last_call_et,
  'S.LAST_CALL_ET/3600 ....................................: ' || s.last_call_et / 3600
FROM v$session s, v$process p
WHERE p.addr = s.paddr
AND s.sid = NVL('&sid', s.sid);
```
### Who is doing what - based on session id
```
set scan on
set linesize 80
set pagesize 30
set head on
set feed off
set ver off

define piece = 0
col c1 noprint
col c2 noprint
col c3 new_value piece noprint
col sid format 999990
col sql_text format a64
col state format a7


break on sid nodup on state skip 1 nodup

select sess.sid, text.sql_text, 'CURRENT' state,
  sess.sql_address c1, sess.sql_hash_value c2, text.piece c3
   from v$session sess, v$sqltext text
   where sess.sid in ('&&1')
     and sess.sql_address = text.address
      and sess.sql_hash_value = text.hash_value
       and text.hash_value != 0
union
select sess.sid, text.sql_text, 'PREVID' state,
    sess.prev_sql_addr c1, sess.prev_hash_value c2, text.piece c3
    from v$session sess, v$sqltext text
    where sess.sid in ('&&1')
        and sess.prev_sql_addr = text.address
       and sess.prev_hash_value = text.hash_value
       and text.hash_value != 0
order by 3, 1, 4, 5, 6
/
```
### Inactive Sessions by Username

```sql
SELECT username,
       COUNT(*) num_inv_sess
FROM v$session
WHERE last_call_et > 3600
  AND username IS NOT NULL
  AND status = 'INACTIVE'
GROUP BY username
ORDER BY num_inv_sess DESC;
```

### Long Running Operations

```sql
ALTER SESSION SET nls_date_format = 'dd/mm/yyyy hh24:mi';

SELECT sid,
       opname,
       target,
       ROUND(sofar/totalwork * 100, 2) AS percent_done,
       start_time,
       last_update_time,
       time_remaining
FROM v$session_longops;
```

### Session Longops Details (RAC)

```sql
SELECT inst_id,
       sid,
       serial#,
       opname,
       sofar,
       totalwork,
       start_time,
       last_update_time,
       username
FROM gv$session_longops;
```

### Undo Generated by Session

```sql
SELECT username,
       t.used_ublk,
       t.used_urec
FROM gv$transaction t,
     gv$session s
WHERE t.addr = s.taddr
  AND s.sid = '&sid';
```

### DB Time by Session

```sql
SET LINES 250
COLUMN inst_id FORMAT 9999
COLUMN username FORMAT A25
COLUMN module FORMAT A20
COLUMN stat_name FORMAT A40

SELECT b.inst_id,
       b.username,
       SUBSTR(module, 1, 15) module,
       a.stat_name,
       ROUND(AVG(a.value)/1000000, 0) avg_time_secs
FROM gv$sess_time_model a,
     gv$session b
WHERE a.inst_id = b.inst_id
  AND a.sid = b.sid
  AND ROUND(a.value/1000000, 0) > 1000
  AND a.stat_name = 'DB time'
GROUP BY b.inst_id, b.username, SUBSTR(module, 1, 15), a.stat_name
ORDER BY avg_time_secs DESC;
```
## **SESSION CONNECTION TO RAC DATABASE LEVEL**

These queries report on current sessions and historical logon activity across all RAC instances,
using both dynamic performance views and AWR (`DBA_HIST_*`) views.

> Run the AWR-based queries from `CDB$ROOT`, not from a PDB.

---

### 1. Current User Sessions per RAC Instance (No AWR)

```sql
-- Total current USER sessions per RAC instance (no AWR required)

SELECT
    s.inst_id,
    i.instance_name,
    s.service_name,
    COUNT(*) AS session_count
FROM   gv$session s
JOIN   gv$instance i
       ON s.inst_id = i.inst_id
WHERE  s.type = 'USER'
GROUP  BY
       s.inst_id,
       i.instance_name,
       s.service_name
ORDER  BY
       s.service_name,
       session_count DESC;
```

---

### 2. Recent Logon Rate per Instance (No AWR)

```sql
-- Recent logon rate per RAC instance (no AWR, uses gv$sysmetric_history)

SELECT
    h.inst_id,
    i.instance_name,
    MIN(h.begin_time)      AS first_sample,
    MAX(h.end_time)        AS last_sample,
    ROUND(AVG(h.value), 2) AS avg_logons_per_sec
FROM   gv$sysmetric_history h
JOIN   gv$instance          i
       ON i.inst_id = h.inst_id
WHERE  h.metric_name = 'Logons Per Sec'
GROUP  BY
       h.inst_id,
       i.instance_name
ORDER  BY
       avg_logons_per_sec DESC;
```

---

### 3. Total Logons Since Startup per RAC Instance (No AWR)

```sql
-- Total logons since startup per RAC instance (cumulative since instance startup)

SELECT
    s.inst_id,
    i.instance_name,
    i.host_name,
    MAX(s.value) AS logons_since_startup
FROM   gv$sysstat s
JOIN   gv$instance i
       ON i.inst_id = s.inst_id
WHERE  s.name = 'logons cumulative'
GROUP  BY
       s.inst_id,
       i.instance_name,
       i.host_name
ORDER  BY
       logons_since_startup DESC;
```

---

### 4. AWR: Total Logons per Instance – Full AWR History

> Run from `CDB$ROOT`. This aggregates deltas of `logons cumulative` across all AWR snapshots.

```sql
-- Total logons per instance across entire AWR history

WITH stat_snap AS (
  SELECT
      sn.snap_id,
      sn.dbid,
      sn.instance_number,
      sn.end_interval_time,
      MAX(CASE
            WHEN st.stat_name = 'logons cumulative'
            THEN st.value
          END) AS logons_cum
  FROM   dba_hist_sysstat  st
  JOIN   dba_hist_snapshot sn
         ON st.snap_id         = sn.snap_id
        AND st.dbid            = sn.dbid
        AND st.instance_number = sn.instance_number
  -- Optional time filter, e.g. last 7 days:
  -- WHERE sn.end_interval_time >= SYSDATE - 7
  GROUP  BY
         sn.snap_id,
         sn.dbid,
         sn.instance_number,
         sn.end_interval_time
),
deltas AS (
  SELECT
      instance_number AS inst_id,
      end_interval_time,
      logons_cum
        - LAG(logons_cum) OVER (
            PARTITION BY instance_number
            ORDER BY end_interval_time
          ) AS logons_delta
  FROM stat_snap
)
SELECT
    d.inst_id,
    i.instance_name,
    SUM(d.logons_delta) AS total_logons
FROM   deltas    d
JOIN   gv$instance i
       ON i.inst_id = d.inst_id
WHERE  d.logons_delta IS NOT NULL
GROUP  BY
       d.inst_id,
       i.instance_name
ORDER  BY
       total_logons DESC;
```

---

### 5. AWR: Total Logons per Instance – Last 7 Days

```sql
-- Total logons per instance in the last 7 days (AWR)

WITH stat_snap AS (
  SELECT
      sn.snap_id,
      sn.instance_number,
      sn.end_interval_time,
      MAX(CASE
            WHEN st.stat_name = 'logons cumulative'
            THEN st.value
          END) AS logons_cum
  FROM   dba_hist_sysstat  st
  JOIN   dba_hist_snapshot sn
         ON st.snap_id         = sn.snap_id
        AND st.dbid            = sn.dbid
        AND st.instance_number = sn.instance_number
  WHERE  sn.end_interval_time >= SYSDATE - 7
  GROUP  BY
         sn.snap_id,
         sn.instance_number,
         sn.end_interval_time
),
deltas AS (
  SELECT
      instance_number AS inst_id,
      end_interval_time,
      logons_cum
        - LAG(logons_cum) OVER (
            PARTITION BY instance_number
            ORDER BY end_interval_time
          ) AS logons_delta
  FROM   stat_snap
)
SELECT
    d.inst_id,
    i.instance_name,
    SUM(d.logons_delta) AS total_logons
FROM   deltas    d
JOIN   gv$instance i
       ON i.inst_id = d.inst_id
WHERE  d.logons_delta IS NOT NULL
GROUP  BY
       d.inst_id,
       i.instance_name
ORDER  BY
       total_logons DESC;
```
```

## Usage Notes

- Replace `&1`, `&2`, `&sid`, `&starttime`, `&endtime` with actual values or use SQL*Plus substitution variables
- These queries require appropriate Oracle database privileges (typically DBA or specific system view access)
- Some queries use `gv$` views for RAC environments; use `v$` views for single-instance databases
- Historical queries (`DBA_HIST_*` views) require Oracle Diagnostics Pack license

## Contributing

Feel free to contribute additional useful Oracle monitoring queries by submitting a pull request.

## License

This collection is provided as-is for educational and operational purposes.
