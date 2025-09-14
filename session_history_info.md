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

## Usage Notes

- Replace `&1`, `&2`, `&sid`, `&starttime`, `&endtime` with actual values or use SQL*Plus substitution variables
- These queries require appropriate Oracle database privileges (typically DBA or specific system view access)
- Some queries use `gv$` views for RAC environments; use `v$` views for single-instance databases
- Historical queries (`DBA_HIST_*` views) require Oracle Diagnostics Pack license

## Contributing

Feel free to contribute additional useful Oracle monitoring queries by submitting a pull request.

## License

This collection is provided as-is for educational and operational purposes.
