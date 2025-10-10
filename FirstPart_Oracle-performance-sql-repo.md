### IN PROGRESS
# Oracle Performance Tuning - SQL Scripts Collection

A comprehensive collection of production-ready SQL scripts for Oracle database performance analysis, monitoring, and tuning.

## Table of Contents

- [SQL History and Analysis](#sql-history-and-analysis)
- [Active Session Monitoring](#active-session-monitoring)
- [AWR (Automatic Workload Repository)](#awr-automatic-workload-repository)
- [ASH (Active Session History)](#ash-active-session-history)
- [SQL Tuning Advisor](#sql-tuning-advisor)
- [Wait Events Analysis](#wait-events-analysis)
- [Memory Management](#memory-management)
- [Buffer Cache Analysis](#buffer-cache-analysis)
- [Session Diagnostics](#session-diagnostics)
- [Performance Spikes Detection](#performance-spikes-detection)

---

## SQL History and Analysis

### Historical DML Operations (INSERT/UPDATE/DELETE)

Track DML operations from AWR snapshots for a specific time period:

```sql
SELECT DISTINCT a.*
FROM (
    SELECT sql.snap_id,
           sql.module,
           sql.sql_id,
           DECODE(t.command_type, 2, 'INSERT', 6, 'UPDATE', 7, 'DELETE') cmd_type,
           s.begin_interval_time,
           s.end_interval_time,
           sql.executions_total,
           sql.rows_processed_total,
           sql.elapsed_time_total/1000000 "elapsed(in secs)"
    FROM dba_hist_sqlstat sql,
         dba_hist_snapshot s,
         dba_hist_sqltext t
    WHERE s.snap_id = sql.snap_id
      AND sql.sql_id = t.sql_id
      AND s.begin_interval_time BETWEEN TO_DATE('13-nov-2015 13:00:00', 'dd-mon-yyyy hh24:mi:ss')
                                    AND TO_DATE('14-nov-2015 10:00:00', 'dd-mon-yyyy hh24:mi:ss')
      AND sql.sql_id IN (
          SELECT DISTINCT stat.sql_id
          FROM dba_hist_sqlstat stat
          JOIN dba_hist_sqltext txt ON (stat.sql_id = txt.sql_id)
          JOIN dba_hist_snapshot snap ON (stat.snap_id = snap.snap_id)
          WHERE snap.begin_interval_time BETWEEN TO_DATE('13-nov-2015 13:00:00', 'dd-mon-yyyy hh24:mi:ss')
                                             AND TO_DATE('14-nov-2015 12:00:00', 'dd-mon-yyyy hh24:mi:ss')
            AND txt.command_type IN (2, 6, 7)
      )
) a
ORDER BY a.snap_id ASC;
```

### Get SQL Text by SQL_ID

```sql
SELECT sql_text
FROM v$sql
WHERE sql_id = '&sql_id';
```

### Historical Execution Plans with Nested Loops

Analyze nested loop performance over time:

```sql
SELECT TO_CHAR(sn.begin_interval_time, 'yy-mm-dd hh24') snap_time,
       COUNT(*) ct,
       SUM(st.rows_processed_delta) row_ct,
       SUM(st.disk_reads_delta) disk,
       SUM(st.cpu_time_delta) cpu
FROM dba_hist_snapshot sn,
     dba_hist_sqlstat st,
     dba_hist_sql_plan sp
WHERE st.snap_id = sn.snap_id
  AND st.dbid = sn.dbid
  AND st.instance_number = sn.instance_number
  AND sp.sql_id = st.sql_id
  AND sp.dbid = st.dbid
  AND sp.plan_hash_value = st.plan_hash_value
  AND sp.operation = 'NESTED LOOPS'
GROUP BY TO_CHAR(sn.begin_interval_time, 'yy-mm-dd hh24')
HAVING COUNT(*) > 50;
```

---

## Active Session Monitoring

### Current Active Sessions

```sql
SET PAGESIZE 1000
SET LINESIZE 200
COLUMN username FORMAT A12
COLUMN lockwait FORMAT A10
COLUMN machine FORMAT A8
COLUMN status FORMAT A10
COLUMN program FORMAT A25
COLUMN sid FORMAT 9999

SELECT DECODE(a.username, NULL, 'Oracle RDBMS', a.username) username,
       a.sid sid,
       DECODE(a.lockwait, NULL, '-none-', a.lockwait) lockwait,
       e.consistent_gets query,
       e.physical_reads disk,
       a.last_call_et last_call,
       a.status,
       DECODE(a.process, NULL, d.spid, a.process) process,
       DECODE(SUBSTR(a.program, 1, 25), NULL,
              SUBSTR(d.program, 1, 25),
              SUBSTR(a.program, 1, 25)) program
FROM v$session a,
     v$dispatcher b,
     v$circuit c,
     v$process d,
     v$sess_io e
WHERE a.saddr = c.saddr(+)
  AND a.server IS NOT NULL
  AND b.paddr(+) = c.dispatcher
  AND a.paddr(+) = d.addr
  AND a.sid = e.sid
ORDER BY 2, 4, 5, 6;
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
    ORDER BY 4 DESC
)
WHERE ROWNUM < 11;
```

### Active Sessions with SQL Text

```sql
SELECT sesion.sid,
       sesion.serial#,
       sesion.username,
       optimizer_mode,
       hash_value,
       address,
       sql_text
FROM v$sqlarea sqlarea,
     v$session sesion
WHERE sesion.sql_hash_value = sqlarea.hash_value
  AND sesion.sql_address = sqlarea.address
  AND sesion.username IS NOT NULL;
```

### Current Running SQLs (Detailed)

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
  AND sql_text NOT LIKE 'select S.USERNAME,S.seconds_in_wait%';
```

### Sessions Running for More Than 1 Hour

```sql
SET PAGES 50000 LINES 32767
COLUMN username FORMAT A10
COLUMN machine FORMAT A15
COLUMN program FORMAT A40

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

### Session Details by SID

```sql
SET HEAD OFF
SET VERIFY OFF
SET PAGES 1500
SET LINESIZE 120

SELECT 'Session  Id.............................................: ' || s.sid,
       'Serial Num..............................................: ' || s.serial#,
       'User Name ..............................................: ' || s.username,
       'Session Status .........................................: ' || s.status,
       'Client Process Id on Client Machine ....................: ' || '*' || s.process || '*' Client,
       'Server Process ID ......................................: ' || p.spid Server,
       'Sql_Address ............................................: ' || s.sql_address,
       'Sql_hash_value .........................................: ' || s.sql_hash_value,
       'Schema Name ............................................: ' || s.schemaname,
       'Program  ...............................................: ' || s.program,
       'Module .................................................: ' || s.module,
       'Action .................................................: ' || s.action,
       'Terminal ...............................................: ' || s.terminal,
       'Client Machine .........................................: ' || s.machine,
       'LAST_CALL_ET ...........................................: ' || s.last_call_et,
       'S.LAST_CALL_ET/3600 ....................................: ' || s.last_call_et/3600
FROM v$session s,
     v$process p
WHERE p.addr = s.paddr
  AND s.sid = NVL('&sid', s.sid);
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

---

## AWR (Automatic Workload Repository)

### Current Logons Count

```sql
SELECT TO_CHAR(begin_interval_time, 'mm-dd-yy hh24:mi'),
       SUM(value) value
FROM dba_hist_sysstat s,
     dba_hist_snapshot h
WHERE s.snap_id = h.snap_id
  AND s.instance_number = h.instance_number
  AND stat_name LIKE '%logons current%'
  AND begin_interval_time > SYSDATE - 1
GROUP BY TO_CHAR(begin_interval_time, 'mm-dd-yy hh24:mi')
ORDER BY 2;
```

### Wait Event Statistics by Time Period

```sql
SELECT a.instance_number,
       TO_CHAR(a.begin_interval_time, 'mm-dd-yy hh24') begin_interval,
       SUBSTR(a.end_interval_time - a.begin_interval_time, 11, 9) snap_time_hms,
       b.total_waits_fg - LAG(b.total_waits_fg, 1) OVER (ORDER BY a.begin_interval_time) twg,
       b.total_timeouts_fg - LAG(b.total_timeouts_fg, 1) OVER (ORDER BY a.begin_interval_time) ttf,
       b.time_waited_micro_fg - LAG(b.time_waited_micro_fg, 1) OVER (ORDER BY a.begin_interval_time) twmf
FROM dba_hist_snapshot a,
     dba_hist_system_event b
WHERE a.snap_id = b.snap_id
  AND a.instance_number = b.instance_number
  AND a.dbid = b.dbid
  AND a.instance_number = b.instance_number
  AND b.event_name = &name
  AND a.begin_interval_time > SYSDATE - 5
  AND a.instance_number = 1
ORDER BY 2, 1;
```

### Enqueue Deadlocks

```sql
COLUMN enq_deadlocks FORMAT 999,999,999

SELECT TO_CHAR(a.end_interval_time, 'dd Mon HH24:mi') snap_date,
       s.value enq_deadlocks
FROM dba_hist_sysstat s,
     dba_hist_snapshot a
WHERE a.snap_id = (SELECT MAX(snap_id) FROM dba_hist_snapshot)
  AND s.snap_id = a.snap_id
  AND s.stat_name = 'enqueue deadlocks'
ORDER BY 1, 2;
```

### Datafile Latency Histogram

```sql
SET LINES 80
SET PAGES 80

WITH hist AS (
    SELECT sn.snap_id,
           sn.dbid,
           TO_CHAR(TRUNC(CAST(begin_interval_time AS DATE)) +
                   (ROUND((CAST(begin_interval_time AS DATE) -
                          TRUNC(CAST(begin_interval_time AS DATE))) * 24) / 24),
                   'YYYY/MM/DD HH24:MI') btime,
           h.event_name,
           h.wait_time_milli,
           h.wait_count
    FROM dba_hist_event_histogram h,
         dba_hist_snapshot sn
    WHERE h.instance_number = 2
      AND sn.instance_number = 2
      AND h.event_name LIKE 'db file seq%'
      AND sn.snap_id = h.snap_id
      AND sn.dbid = h.dbid
)
SELECT btime,
       SUM(milli_1) milli_1,
       SUM(milli_2) milli_2,
       SUM(milli_4) milli_4,
       SUM(milli_8) milli_8,
       SUM(milli_16) milli_16,
       SUM(milli_32) milli_32,
       SUM(milli_64) milli_64,
       SUM(milli_128) milli_128,
       SUM(milli_256) milli_256,
       SUM(milli_512) milli_512
FROM (
    SELECT btime,
           CASE wait_time_milli WHEN 1 THEN wait_count ELSE 0 END milli_1,
           CASE wait_time_milli WHEN 2 THEN wait_count ELSE 0 END milli_2,
           CASE wait_time_milli WHEN 4 THEN wait_count ELSE 0 END milli_4,
           CASE wait_time_milli WHEN 8 THEN wait_count ELSE 0 END milli_8,
           CASE wait_time_milli WHEN 16 THEN wait_count ELSE 0 END milli_16,
           CASE wait_time_milli WHEN 32 THEN wait_count ELSE 0 END milli_32,
           CASE wait_time_milli WHEN 64 THEN wait_count ELSE 0 END milli_64,
           CASE wait_time_milli WHEN 128 THEN wait_count ELSE 0 END milli_128,
           CASE wait_time_milli WHEN 256 THEN wait_count ELSE 0 END milli_256,
           CASE wait_time_milli WHEN 512 THEN wait_count ELSE 0 END milli_512
    FROM (
        SELECT a.btime,
               a.wait_time_milli,
               SUM(b.wait_count - a.wait_count) wait_count
        FROM hist a,
             hist b
        WHERE a.dbid = b.dbid
          AND a.snap_id = b.snap_id - 1
          AND a.wait_time_milli = b.wait_time_milli
        GROUP BY a.btime, a.wait_time_milli
        HAVING SUM(b.wait_count - a.wait_count) > 0
    )
)
GROUP BY btime
ORDER BY 1, 2;
```

### Complete Wait Events Information from AWR

```sql
SELECT s.begin_interval_time,
       m.*
FROM (
    SELECT ee.instance_number,
           ee.snap_id,
           ee.event_name,
           ROUND(ee.event_time_waited/1000000) event_time_waited,
           ee.total_waits,
           ROUND((ee.event_time_waited * 100)/et.total_time_waited, 1) pct,
           ROUND((ee.event_time_waited/ee.total_waits)/1000) avg_wait
    FROM (
        SELECT ee1.instance_number,
               ee1.snap_id,
               ee1.event_name,
               ee1.time_waited_micro - ee2.time_waited_micro event_time_waited,
               ee1.total_waits - ee2.total_waits total_waits
        FROM dba_hist_system_event ee1
        JOIN dba_hist_system_event ee2
          ON ee1.snap_id = ee2.snap_id + 1
         AND ee1.instance_number = ee2.instance_number
         AND ee1.event_id = ee2.event_id
         AND ee1.wait_class_id <> 2723168908
         AND ee1.time_waited_micro - ee2.time_waited_micro > 0
        UNION
        SELECT st1.instance_number,
               st1.snap_id,
               st1.stat_name event_name,
               st1.value - st2.value event_time_waited,
               1 total_waits
        FROM dba_hist_sys_time_model st1
        JOIN dba_hist_sys_time_model st2
          ON st1.instance_number = st2.instance_number
         AND st1.snap_id = st2.snap_id + 1
         AND st1.stat_id = st2.stat_id
         AND st1.stat_name = 'DB CPU'
         AND st1.value - st2.value > 0
    ) ee
    JOIN (
        SELECT et1.instance_number,
               et1.snap_id,
               et1.value - et2.value total_time_waited
        FROM dba_hist_sys_time_model et1
        JOIN dba_hist_sys_time_model et2
          ON et1.snap_id = et2.snap_id + 1
         AND et1.instance_number = et2.instance_number
         AND et1.stat_id = et2.stat_id
         AND et1.stat_name = 'DB time'
         AND et1.value - et2.value > 0
    ) et
      ON ee.instance_number = et.instance_number
     AND ee.snap_id = et.snap_id
) m
JOIN dba_hist_snapshot s
  ON m.snap_id = s.snap_id
WHERE event_name = 'log file sync'
  AND pct > 15
ORDER BY m.instance_number, m.snap_id, event_time_waited DESC;
```

---

## ASH (Active Session History)

### Generate ASH Report

```sql
@?/rdbms/admin/ashrpt.sql
```

### ASH Enqueue Statistics

```sql
COLUMN begin_interval_time FORMAT A10
COLUMN req_reason FORMAT A25
COLUMN cum_wait_time HEAD 'CUM|WAIT|TIME'
COLUMN total_req# HEAD 'TOTAL|REQ#'
COLUMN total_wait# HEAD 'TOTAL|WAIT#'
COLUMN failed_req# HEAD 'FAILED|REQ#'

SELECT begin_interval_time,
       eq_type,
       req_reason,
       total_req#,
       total_wait#,
       succ_req#,
       failed_req#,
       cum_wait_time
FROM dba_hist_enqueue_stat
NATURAL JOIN dba_hist_snapshot
WHERE cum_wait_time > 0
ORDER BY begin_interval_time, cum_wait_time;
```

### Last Running SQL from ASH

```sql
SET PAGES 50000 LINES 32767

SELECT inst_id,
       sample_time,
       session_id,
       session_serial#,
       sql_id
FROM gv$active_session_history
WHERE sql_id IS NOT NULL
ORDER BY 1 DESC;
```

### Top 10 CPU Consumers (Last 5 Minutes)

```sql
SELECT *
FROM (
    SELECT session_id,
           session_serial#,
           COUNT(*)
    FROM v$active_session_history
    WHERE session_state = 'ON CPU'
      AND sample_time > SYSDATE - INTERVAL '5' MINUTE
    GROUP BY session_id, session_serial#
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 10;
```

### Most CPU Intensive SQL (Past 1 Hour)

```sql
SELECT a.sql_id,
       a.sess_count,
       a.cpu_load,
       b.sql_text
FROM (
    SELECT sql_id,
           COUNT(*) sess_count,
           ROUND(COUNT(*)/SUM(COUNT(*)) OVER (), 2) cpu_load
    FROM v$active_session_history
    WHERE sample_time > SYSDATE - 1/24
      AND session_type <> 'BACKGROUND'
      AND session_state = 'ON CPU'
    GROUP BY sql_id
    ORDER BY COUNT(*) DESC
) a,
v$sqlarea b
WHERE a.sql_id = b.sql_id;
```

### Most IO Intensive SQL (Past 1 Hour)

```sql
SELECT a.sql_id,
       a.sess_count,
       b.sql_text
FROM (
    SELECT c.sql_id,
           COUNT(*) sess_count
    FROM v$active_session_history c,
         v$event_name d
    WHERE c.sample_time > SYSDATE - 1/24
      AND c.event_id = d.event_id
      AND c.wait_class = 'User I/O'
    GROUP BY c.sql_id
    ORDER BY COUNT(*) DESC
) a,
v$sqlarea b
WHERE a.sql_id = b.sql_id;
```

### Top 10 Waiting Sessions (Last 5 Minutes)

```sql
SELECT *
FROM (
    SELECT session_id,
           session_serial#,
           COUNT(*)
    FROM v$active_session_history
    WHERE session_state = 'WAITING'
      AND sample_time > SYSDATE - INTERVAL '5' MINUTE
    GROUP BY session_id, session_serial#
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 10;
```

### ASH Blocking Events

```sql
SELECT TO_CHAR(sample_time, 'mm-dd-yy hh24') sample_time,
       event,
       COUNT(blocking_session)
FROM dba_hist_active_sess_history
WHERE blocking_session IS NOT NULL
  AND sample_time > SYSDATE - 1
  AND event = 'gc buffer busy acquire'
GROUP BY TO_CHAR(sample_time, 'mm-dd-yy hh24'), event
ORDER BY 1;
```

---

## SQL Tuning Advisor

### Step 1: Find SQL_ID

```sql
SELECT sql_id,
       sql_text
FROM v$sql
WHERE sql_text LIKE '%<SQL PART>%';
```

### Step 2: Create Tuning Task

```sql
SET SERVEROUTPUT ON
DECLARE
    mv_sql_tune_task_id VARCHAR2(100);
BEGIN
    mv_sql_tune_task_id := DBMS_SQLTUNE.create_tuning_task(
        sql_id      => '&sql_id',
        scope       => DBMS_SQLTUNE.scope_comprehensive,
        time_limit  => 60,
        task_name   => 'TUNING_TASK_01',
        description => 'Tuning task for performance analysis'
    );
    DBMS_OUTPUT.put_line('Task ID: ' || mv_sql_tune_task_id);
END;
/
```

### Step 3: Execute Tuning Task

```sql
EXEC DBMS_SQLTUNE.execute_tuning_task(task_name => 'TUNING_TASK_01');
```

### Step 4: Check Task Status

```sql
SELECT task_name,
       status
FROM dba_advisor_log
WHERE task_name LIKE 'TUNING%';
```

### Step 5: View Recommendations

```sql
SET LONG 10000
SET PAGESIZE 1000
SET LINESIZE 200

SELECT DBMS_SQLTUNE.report_tuning_task('TUNING_TASK_01') AS recommendations
FROM dual;
```

### Run SQL Tuning Report (sqltrpt.sql)

```sql
@?/rdbms/admin/sqltrpt.sql
```

### SQL Profile Management

**Create SQL Profile:**
```sql
EXEC DBMS_SQLTUNE.accept_sql_profile(
    task_name  => '<TASK_NAME>',
    task_owner => 'SYS',
    replace    => TRUE,
    force_match => TRUE
);
```

**View SQL Profiles:**
```sql
SELECT name,
       created,
       last_modified
FROM dba_sql_profiles
ORDER BY created DESC;
```

**Drop SQL Profile:**
```sql
EXEC DBMS_SQLTUNE.drop_sql_profile('<SQL_PROFILE_NAME>');
```

**Disable SQL Profile:**
```sql
EXEC DBMS_SQLTUNE.alter_sql_profile(
    '<SQL_PROFILE_NAME>',
    'STATUS',
    'DISABLED'
);
```

**Find SQL Profile by SQL Text:**
```sql
SELECT name,
       sql_text
FROM dba_sql_profiles
WHERE sql_text LIKE '%SELECT%TABLE%NAME%';
```

**Check SQLs Using Profiles:**
```sql
SELECT sql_id,
       child_number,
       plan_hash_value plan_hash,
       sql_profile,
       executions execs,
       (elapsed_time/1000000)/DECODE(NVL(executions, 0), 0, 1, executions) avg_etime,
       buffer_gets/DECODE(NVL(executions, 0), 0, 1, executions) avg_lio,
       sql_text
FROM v$sql s
WHERE UPPER(sql_text) LIKE UPPER(NVL('&sql_text', sql_text))
  AND sql_text NOT LIKE '%from v$sql where sql_text like nvl(%'
  AND sql_id LIKE NVL('&sql_id', sql_id)
  AND sql_profile LIKE NVL('&sql_profile_name', sql_profile)
  AND sql_profile IS NOT NULL
ORDER BY 1, 2, 3;
```

---

## Wait Events Analysis

### All Wait Events by Count

```sql
COLUMN event FORMAT A30 TRUNC HEADING "Wait event"
COLUMN wait_ct FORMAT 9,999 HEADING "Sessions Waiting"

SELECT event,
       COUNT(*) wait_ct
FROM v$session_wait
WHERE event NOT IN ('Null event', 'rdbms ipc message', 'smon timer', 'pmon timer')
GROUP BY event
ORDER BY 2 DESC;
```

### Top 5 Wait Events from AWR

```sql
SELECT t.*,
       e.name
FROM (
    SELECT event_id,
           SUM(time_waited)
    FROM dba_hist_active_sess_history
    WHERE snap_id BETWEEN &begin_snap AND &end_snap
    GROUP BY event_id
    ORDER BY 2 DESC
) t,
v$event_name e
WHERE t.event_id = e.event_id
  AND ROWNUM <= 5;
```

### Identify SQL Causing Specific Wait Event

```sql
SELECT sql_id,
       event_id,
       COUNT(*) cnt
FROM dba_hist_active_sess_history
WHERE snap_id BETWEEN &begin_snap AND &end_snap
  AND event_id = &event_id
GROUP BY sql_id, event_id
HAVING COUNT(*) > 100
ORDER BY 3 DESC;
```

### Object Causing Wait Events

```sql
-- Get distinct objects
SELECT COUNT(DISTINCT(current_obj#))
FROM dba_hist_active_sess_history
WHERE snap_id BETWEEN &begin_snap AND &end_snap
  AND event_id = &event_id
  AND sql_id = '&sql_id';

-- Get object wait counts
SELECT current_obj#,
       COUNT(*) cnt
FROM dba_hist_active_sess_history
WHERE snap_id BETWEEN &begin_snap AND &end_snap
  AND event_id = &event_id
  AND sql_id = '&sql_id'
GROUP BY current_obj#
ORDER BY 2 DESC;

-- Get object details
SELECT object_id,
       owner,
       object_name,
       object_type
FROM dba_objects
WHERE object_id = &object_id;
```

### Find Blocks Causing Contention

```sql
SELECT current_file#,
       current_block#,
       COUNT(*) cnt
FROM dba_hist_active_sess_history
WHERE snap_id BETWEEN &begin_snap AND &end_snap
  AND event_id = &event_id
  AND sql_id = '&sql_id'
  AND current_obj# = &object_id
GROUP BY current_file#, current_block#
HAVING COUNT(*) > 0
ORDER BY 3 DESC;
```

### Check if Blocks are Header or Data Blocks

```sql
SELECT segment_name,
       header_file,
       header_block
FROM dba_segments
WHERE owner = '&owner'
  AND segment_name = '&segment_name';
```

---

## Performance Spikes Detection

### Parallel Process Spikes

```sql
COLUMN program FORMAT A25
COLUMN machine FORMAT A10
COLUMN module FORMAT A25
COLUMN user_id FORMAT 9999999
COLUMN count FORMAT 99999

SELECT *
FROM (
    SELECT /*+ PARALLEL */
           COUNT(*) AS count,
           user_id,
           program,
           module,
           machine,
           sql_id
    FROM sys.dba_hist_active_sess_history
    WHERE sample_time > TO_DATE('25-SEP-2012 08:00:00', 'DD-MON-YYYY HH24:MI:SS')
      AND sample_time < TO_DATE('25-SEP-2012 11:10:00', 'DD-MON-YYYY HH24:MI:SS')
      AND program LIKE 'oracle@%'
    GROUP BY user_id, program, module, machine, sql_id
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 20;
```

### Wait Event Spikes

```sql
SELECT *
FROM (
    SELECT /*+ PARALLEL */
           COUNT(*) AS count,
           user
