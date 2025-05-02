````markdown
# Oracle Session Monitoring Queries

These SQL queries and scripts are designed to provide detailed information about active Oracle database sessions, CPU usage, memory usage, I/O operations, and other session details. Below is a breakdown of the provided queries and their use cases:

## Active Session History (ASH) Queries:

### 1. Most Active Session in the Last One Hour
This query shows the most active sessions over the last hour based on `gv$active_session_history`. It uses `sql_id` to identify sessions and calculates the percentage load for each SQL ID.

```sql
SELECT sql_id, COUNT(*), ROUND(COUNT(*) / SUM(COUNT(*)) OVER(), 2) PCTLOAD
FROM gv$active_session_history
WHERE sample_time > SYSDATE - 1/24
AND session_type = 'BACKGROUND'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
````

### 2. Get the Wait Event for the Above Session

Once a session is identified from the previous query, this query will retrieve the specific wait events that the session is currently waiting on, along with wait times.

```sql
SELECT sample_time, event, wait_time
FROM gv$active_session_history
WHERE session_id = &1
AND session_serial# = &2;
```

### 3. Most I/O Intensive SQL in the Last 1 Hour

This query identifies SQL queries with the most I/O waits in the last hour. It focuses on sessions in the `WAITING` state, with an event class of `User I/O`.

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

### 4. Top SQLs Based on CPU, Wait, and I/O

This query aggregates the `v$active_session_history` and `v$event_name` to show which SQLs are consuming the most CPU, waiting time, or I/O resources.

```sql
SELECT
  ash.SQL_ID,
  SUM(DECODE(a.session_state, 'ON CPU', 1, 0)) "CPU",
  SUM(DECODE(a.session_state, 'WAITING', 1, 0)) -
    SUM(DECODE(a.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "WAIT",
  SUM(DECODE(a.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "IO",
  SUM(DECODE(a.session_state, 'ON CPU', 1, 1)) "TOTAL"
FROM v$active_session_history a, v$event_name en
WHERE SQL_ID IS NOT NULL
AND en.event# = ash.event#
```

### 5. SQL Analysis for a Particular Session

This query looks at the active session history for a particular session ID (`sid`) and provides details about the SQLs being executed, events, and the associated wait times.

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

### 6. Top Sessions on CPU in the Last 15 Minutes

This query returns the top sessions that have consumed the most CPU time in the last 15 minutes.

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

### 7. Find Queries Executed in the Last 30 Days

This query retrieves the SQL queries executed by users in the last 30 days, providing the sample time, username, program, module, and the actual SQL text.

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

## Running SQL Queries:

### 1. Current Running SQLs

This query displays details of SQL queries currently being executed, including the SQL ID, program, machine, event, and session wait times. It excludes certain internal queries (e.g., `S.USERNAME`).

```sql
SELECT sid, serial#, a.sql_id, a.SQL_TEXT, S.USERNAME, i.host_name, machine, S.event, S.seconds_in_wait sec_wait,
       TO_CHAR(logon_time, 'DD-MON-RR HH24:MI') login
FROM gv$session S, gv$SQLAREA A, gv$instance i
WHERE S.username IS NOT NULL
AND S.sql_address = A.address
AND s.inst_id = a.inst_id
AND i.inst_id = a.inst_id
AND sql_text NOT LIKE 'select S.USERNAME, S.seconds_in_wait%'
```

### 2. Active Sessions Running for More Than 1 Hour

Displays active sessions that have been running for more than an hour. It provides session login details, including session ID, machine, program, and login time.

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

### 3. Active Sessions Running for More Than 30 Minutes

This query shows active sessions running for more than 30 minutes and their associated details.

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

## Session Details Associated with Oracle SID:

### 1. Session Details

This script provides detailed session information for a particular `SID`, including client and server process IDs, SQL address, schema name, program, module, and more. It’s useful for diagnosing issues associated with specific sessions.

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
