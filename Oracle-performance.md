# Oracle Performance SQL Repository

A unified, production-ready collection of Oracle database performance monitoring and tuning SQL scripts. All queries are optimized for DBA use and cover monitoring, diagnostics, and optimization across all major Oracle performance areas.

## Table of Contents

1. [SQL Performance Analysis](#1-sql-performance-analysis)
2. [Wait Events Analysis](#2-wait-events-analysis)
3. [Latch & Enqueue Contention](#3-latch--enqueue-contention)
4. [Library Cache & Shared Pool](#4-library-cache--shared-pool)
5. [Buffer Cache & SGA](#5-buffer-cache--sga)
6. [I/O Performance](#6-io-performance)
7. [AWR Analysis](#7-awr-analysis)
8. [ASH Analysis](#8-ash-analysis)
9. [SQL Tuning Advisor](#9-sql-tuning-advisor)
10. [Database Statistics](#10-database-statistics)
11. [Performance Spikes Detection](#11-performance-spikes-detection)

---
## 1. SQL Performance Analysis

Identify resource-intensive SQL statements and analyze execution patterns.

### Top SQL by CPU Time

```sql
SELECT sql_id,
       executions,
       ROUND(cpu_time/1e6, 2) cpu_s,
       ROUND(elapsed_time/1e6, 2) elapsed_s,
       buffer_gets,
       disk_reads,
       SUBSTR(sql_text, 1, 120) sql_text
FROM v$sqlarea
WHERE executions > 0
ORDER BY cpu_time DESC
FETCH FIRST 20 ROWS ONLY;
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

### Top SQL by CPU, Wait, IO (Last 5 Minutes)

```sql
SELECT ash.sql_id,
       SUM(DECODE(ash.session_state, 'ON CPU', 1, 0)) "CPU",
       SUM(DECODE(ash.session_state, 'WAITING', 1, 0)) - 
       SUM(DECODE(ash.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "WAIT",
       SUM(DECODE(ash.session_state, 'WAITING', DECODE(en.wait_class, 'User I/O', 1, 0), 0)) "IO",
       SUM(DECODE(ash.session_state, 'ON CPU', 1, 1)) "TOTAL"
FROM v$active_session_history ash,
     v$event_name en
WHERE sql_id IS NOT NULL
  AND sample_time > SYSDATE - (5/(24*60))
  AND en.event# = ash.event#
GROUP BY sql_id
ORDER BY SUM(DECODE(session_state, 'ON CPU', 1, 0)) DESC;
```

### SQL with High Disk I/O

```sql
SELECT *
FROM (
    SELECT SUBSTR(sql_text, 1, 500) sql,
           elapsed_time,
           cpu_time,
           disk_reads,
           executions,
           disk_reads/executions "Reads/Exec",
           hash_value,
           address
    FROM v$sqlarea
    WHERE (hash_value, address) IN (
        SELECT DISTINCT hash_value, address
        FROM v$sql_plan
        WHERE distribution IS NOT NULL
    )
      AND disk_reads > 100
      AND executions > 0
    ORDER BY elapsed_time DESC
)
WHERE ROWNUM <= 30;
```

### SQL with High Sharable Memory

```sql
SELECT sql_id,
       sharable_mem,
       SUBSTR(sql_text, 1, 120) sql_text
FROM v$sqlarea
ORDER BY sharable_mem DESC
FETCH FIRST 20 ROWS ONLY;
```

### SQL with Many Versions (Literal Issues)

```sql
SELECT SUBSTR(sql_text, 1, 100) sql_snippet,
       COUNT(*) versions,
       SUM(executions) execs
FROM v$sql
GROUP BY SUBSTR(sql_text, 1, 100)
HAVING COUNT(*) > 10
ORDER BY versions DESC;
```

### Get SQL Text by SQL_ID

```sql
SELECT sql_text
FROM v$sql
WHERE sql_id = '&sql_id';
```

### SQL Text by Hash Value

```sql
SELECT hash_value,
       sql_text
FROM v$sql
WHERE hash_value IN (&hash_value_list);
```

### SQL Text by Hash and Address

```sql
SELECT piece,
       sql_text
FROM v$sqltext
WHERE hash_value = &hash
  AND address = '&addr'
ORDER BY piece;
```

### Top SQL by Buffer Gets

```sql
SELECT hash_value,
       executions,
       buffer_gets,
       buffer_gets/executions get_exec
FROM v$sqlarea
WHERE buffer_gets > 0
  AND executions > 0
ORDER BY buffer_gets DESC;
```

### Historical Execution Plans with Nested Loops

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

### Cached Execution Plans

```sql
SELECT operation,
       object_owner,
       object_name
FROM v$sql_plan
WHERE hash_value = &hash_value;
```

---

## 2. Wait Events Analysis

Analyze database wait events and identify performance bottlenecks.

### All System Wait Events

```sql
SELECT event,
       time_waited,
       total_waits
FROM v$system_event
WHERE event LIKE '%latch%';
```

### Top System Wait Events

```sql
SELECT event,
       total_waits,
       time_waited/100 AS time_s,
       wait_class
FROM v$system_event
WHERE total_waits > 0
ORDER BY time_waited DESC
FETCH FIRST 15 ROWS ONLY;
```

### Wait Events by Count

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

### Complete Wait Events from AWR

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
WHERE event_name = '&event_name'
  AND pct > 15
ORDER BY m.instance_number, m.snap_id, event_time_waited DESC;
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
-- Count distinct objects
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

## 3. Latch & Enqueue Contention

Identify latch contention and enqueue bottlenecks.

### Latch Activity Between Snapshots

```sql
SELECT b.name,
       e.gets - b.gets AS gets,
       ROUND(DECODE(e.gets - b.gets, 0, 0, 
             (e.misses - b.misses) * 100 / (e.gets - b.gets)), 2) AS pct_miss,
       ROUND((e.wait_time - b.wait_time)/1e6, 2) AS wait_s
FROM stats$latch b,
     stats$latch e
WHERE b.snap_id = :bid
  AND e.snap_id = :eid
  AND b.dbid = e.dbid
  AND b.instance_number = e.instance_number
  AND b.name = e.name
ORDER BY pct_miss DESC;
```

### Current Latch Usage Summary

```sql
SELECT name,
       gets,
       misses,
       sleeps,
       spin_gets,
       wait_time
FROM v$latch
ORDER BY gets DESC
FETCH FIRST 10 ROWS ONLY;
```

### Latch Details Top 10

```sql
SELECT *
FROM (
    SELECT name,
           gets,
           misses,
           sleeps,
           spin_gets,
           wait_time
    FROM v$latch
    ORDER BY gets DESC
)
WHERE ROWNUM < 11;
```

### Enqueue Contention (Live View)

```sql
SELECT event,
       total_waits,
       time_waited/100 AS time_s
FROM v$system_event
WHERE event LIKE 'enq:%'
ORDER BY time_waited DESC;
```

### Enqueue Deadlocks from AWR

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

### ASH Enqueue Statistics

```sql
COLUMN begin_interval_time FORMAT A10
COLUMN req_reason FORMAT A25

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

---

## 4. Library Cache & Shared Pool

Evaluate library cache efficiency and shared pool usage.

### Shared Pool Component Usage

```sql
SELECT name,
       ROUND(bytes/1024/1024, 2) mb
FROM v$sgastat
WHERE pool = 'shared pool'
ORDER BY bytes DESC;
```

### Library Cache Statistics

```sql
SELECT namespace,
       gets,
       gethitratio,
       pins,
       pinhitratio,
       reloads,
       invalidations
FROM v$librarycache;
```

### Library Cache Hit Ratio

```sql
SELECT SUM(pins - reloads) * 100/SUM(pins) AS "Hit Ratio"
FROM v$librarycache;
```

### SGA Statistics Top 10

```sql
SELECT name,
       ROUND(bytes/(1024*1024), 0) mb
FROM v$sgastat
WHERE pool = 'shared pool'
ORDER BY bytes DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## 5. Buffer Cache & SGA

Assess SGA and buffer cache performance.

### Buffer Cache Hit Ratio

```sql
SELECT (1 - (phy.value - dir.value - lob.value) / ses.value) * 100 AS hit_ratio
FROM v$sysstat ses,
     v$sysstat phy,
     v$sysstat dir,
     v$sysstat lob
WHERE ses.name = 'session logical reads'
  AND phy.name = 'physical reads'
  AND dir.name = 'physical reads direct'
  AND lob.name = 'physical reads direct (lob)';
```

### Alternative Buffer Cache Hit Ratio

```sql
SELECT pr.value AS "phy. reads",
       prd.value AS "phy. reads direct",
       prdl.value AS "phy. reads direct (lob)",
       slr.value AS "session logical reads",
       1 - (pr.value - prd.value - prdl.value) / slr.value AS "hit ratio"
FROM v$sysstat pr,
     v$sysstat prd,
     v$sysstat prdl,
     v$sysstat slr
WHERE pr.name = 'physical reads'
  AND prd.name = 'physical reads direct'
  AND prdl.name = 'physical reads direct (lob)'
  AND slr.name = 'session logical reads';
```

### SGA Component Allocation

```sql
SELECT pool,
       name,
       ROUND(bytes/1024/1024, 1) mb
FROM v$sgastat
ORDER BY bytes DESC;
```

### Buffer Pool Statistics and Hit Ratios

```sql
SELECT name,
       physical_reads AS "physical reads",
       db_block_gets AS "DB block gets",
       consistent_gets AS "consistent gets",
       1 - (physical_reads / (db_block_gets + consistent_gets)) AS "hit ratio"
FROM v$buffer_pool_statistics
WHERE db_block_gets + consistent_gets > 0;
```

### Contents of Data Buffers

```sql
SET PAGES 999
SET LINES 128
COLUMN c0 HEADING "Owner" FORMAT A12
COLUMN c1 HEADING "Object|Name" FORMAT A30
COLUMN c2 HEADING "Object|Type" FORMAT A8
COLUMN c3 HEADING "Number of|Blocks in|Buffer|Cache" FORMAT 99,999,999

SELECT t1.owner c0,
       object_name c1,
       CASE
           WHEN object_type = 'TABLE PARTITION' THEN 'TAB PART'
           WHEN object_type = 'INDEX PARTITION' THEN 'IDX PART'
           ELSE object_type
       END c2,
       SUM(num_blocks) c3,
       (SUM(num_blocks)/GREATEST(SUM(blocks), .001)) * 100 pct_of_object,
       (SUM(num_blocks)/GREATEST(total_buffer_blocks, .001)) * 100 pct_of_pool,
       buffer_pool c5,
       SUM(bytes)/SUM(blocks) block_size
FROM (SELECT COUNT(*) total_buffer_blocks FROM v$bh) t2,
     (SELECT o.owner owner,
             o.object_name object_name,
             o.subobject_name subobject_name,
             o.object_type object_type,
             COUNT(DISTINCT file# || block#) num_blocks
      FROM dba_objects o,
           v$bh bh
      WHERE o.data_object_id = bh.objd
        AND o.owner NOT IN ('SYS', 'SYSTEM')
        AND bh.status != 'free'
      GROUP BY o.owner, o.object_name, o.subobject_name, o.object_type) t1,
     dba_segments s
WHERE s.segment_name = t1.object_name
  AND s.owner = t1.owner
  AND s.segment_type = t1.object_type
  AND NVL(s.partition_name, '-') = NVL(t1.subobject_name, '-')
GROUP BY t1.owner, object_name, object_type, buffer_pool, total_buffer_blocks
HAVING SUM(num_blocks) > 10
ORDER BY SUM(num_blocks) DESC;
```

### Enable KEEP and RECYCLE Buffer Cache

```sql
SHOW PARAMETER db_keep_cache_size;
SHOW PARAMETER db_recycle_cache_size;

ALTER SYSTEM SET db_keep_cache_size = 16M;
ALTER SYSTEM SET db_recycle_cache_size = 16M;
```

### Move Table to KEEP Buffer Cache

```sql
ALTER TABLE schema.table_name STORAGE (BUFFER_POOL KEEP);
```

---

## 6. I/O Performance

Identify datafiles, tablespaces, and segments generating heavy I/O.

### Datafiles with Highest I/O

```sql
SELECT *
FROM (
    SELECT name,
           phyrds,
           phywrts,
           readtim,
           writetim
    FROM v$filestat a,
         v$datafile b
    WHERE a.file# = b.file#
    ORDER BY readtim DESC
)
WHERE ROWNUM < 6;
```

### Datafiles by I/O

```sql
SELECT df.name,
       fs.phyrds reads,
       fs.phywrts writes,
       fs.readtim,
       fs.writetim
FROM v$filestat fs
JOIN v$datafile df ON fs.file# = df.file#
ORDER BY fs.phyrds DESC
FETCH FIRST 10 ROWS ONLY;
```

### I/O Spread per Tablespace

```sql
SELECT t.name,
       SUM(physical_reads) physical_reads,
       ROUND((RATIO_TO_REPORT(SUM(physical_reads)) OVER ()) * 100, 2) || '%' perc_reads,
       SUM(physical_writes) physical_writes,
       ROUND((RATIO_TO_REPORT(SUM(physical_writes)) OVER ()) * 100, 2) || '%' perc_writes,
       SUM(total) total,
       ROUND((RATIO_TO_REPORT(SUM(total)) OVER ()) * 100, 2) || '%' perc_total
FROM (
    SELECT ts#,
           name,
           phyrds physical_reads,
           phywrts physical_writes,
           phyrds + phywrts total
    FROM v$datafile df,
         v$filestat fs
    WHERE df.file# = fs.file#
    ORDER BY phyrds DESC
) a,
sys.ts$ t
WHERE a.ts# = t.ts#
GROUP BY t.name
ORDER BY physical_reads DESC;
```

### I/O by Tablespace

```sql
SELECT t.name tablespace,
       SUM(fs.phyrds) reads,
       SUM(fs.phywrts) writes
FROM v$datafile df
JOIN v$filestat fs ON df.file# = fs.file#
JOIN sys.ts$ t ON df.ts# = t.ts#
GROUP BY t.name
ORDER BY SUM(fs.phyrds) DESC;
```

### I/O Spread per Datafile

```sql
SELECT name,
       phyrds physical_reads,
       ROUND((RATIO_TO_REPORT(phyrds) OVER ()) * 100, 2) || '%' perc_reads,
       phywrts physical_writes,
       ROUND((RATIO_TO_REPORT(phywrts) OVER ()) * 100, 2) || '%' perc_writes,
       phyrds + phywrts total
FROM v$datafile df,
     v$filestat fs
WHERE df.file# = fs.file#
ORDER BY phyrds DESC;
```

### I/O Spread per Filesystem

```sql
SELECT filesystem,
       ROUND((RATIO_TO_REPORT(reads) OVER ()) * 100, 2) || '%' perc_reads,
       ROUND((RATIO_TO_REPORT(writes) OVER ()) * 100, 2) || '%' perc_writes,
       ROUND((RATIO_TO_REPORT(total) OVER ()) * 100, 2) || '%' perc_total
FROM (
    SELECT filesystem,
           SUM(physical_reads) reads,
           SUM(physical_writes) writes,
           SUM(total) total
    FROM (
        SELECT SUBSTR(name, 0, 25) filesystem,
               phyrds physical_reads,
               phywrts physical_writes,
               phyrds + phywrts total
        FROM v$datafile df,
             v$filestat fs
        WHERE df.file# = fs.file#
    ) a
    GROUP BY filesystem
) b
ORDER BY ROUND((RATIO_TO_REPORT(total) OVER ()) * 100, 2) DESC;
```

### I/O for Specific Tablespace Datafiles

```sql
SELECT df.name,
       phyrds physical_reads,
       ROUND((RATIO_TO_REPORT(phyrds) OVER ()) * 100, 2) || '%' perc_reads,
       phywrts physical_writes,
       ROUND((RATIO_TO_REPORT(phywrts) OVER ()) * 100, 2) || '%' perc_writes,
       phyrds + phywrts total
FROM v$datafile df,
     v$filestat fs,
     ts$ t
WHERE df.file# = fs.file#
  AND df.ts# = t.ts#
  AND t.name = '&tablespace_name'
ORDER BY phyrds DESC;
```

### Top Segments by Logical and Physical Reads

```sql
SELECT owner,
       object_name,
       object_type,
       statistic_name,
       value
FROM v$segstat s
JOIN dba_objects o ON s.obj# = o.object_id
WHERE statistic_name IN ('logical reads', 'physical reads', 'physical writes')
ORDER BY value DESC
FETCH FIRST 20 ROWS ONLY;
```

### Segments with High I/O (Top 10)

```sql
SELECT ROWNUM AS rank,
       st.owner,
       st.obj#,
       st.object_type,
       st.object_name,
       st.value,
       'LIO' AS unit
FROM v$segment_statistics st
WHERE st.statistic_name = 'logical reads'
ORDER BY st.value DESC
FETCH FIRST 10 ROWS ONLY;
```

### Session I/O by User

```sql
SELECT NVL(ses.username, 'ORACLE PROC') username,
       osuser os_user,
       process pid,
       ses.sid sid,
       serial#,
       physical_reads,
       block_gets,
       consistent_gets,
       block_changes,
       consistent_changes
FROM v$session ses,
     v$sess_io sio
WHERE ses.sid = sio.sid
ORDER BY physical_reads DESC, ses.username;
```

---

## 7. AWR Analysis

Use AWR to analyze long-term workload trends.

### AWR Top SQL by Elapsed Time

```sql
SELECT sql_id,
       plan_hash_value,
       executions_delta execs,
       ROUND(elapsed_time_delta/1e6, 2) elapsed_s
FROM dba_hist_sqlstat
WHERE executions_delta > 0
ORDER BY elapsed_time_delta DESC
FETCH FIRST 20 ROWS ONLY;
```

### AWR Logons Current

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

### AWR Wait Event Statistics

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
  AND b.event_name = &event_name
  AND a.begin_interval_time > SYSDATE - 5
  AND a.instance_number = 1
ORDER BY 2, 1;
```

### AWR Datafile Latency Histogram

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
    WHERE h.instance_number = &instance_number
      AND sn.instance_number = &instance_number
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

### Historical DML Operations (INSERT/UPDATE/DELETE)

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
      AND s.begin_interval_time BETWEEN TO_DATE('&start_date', 'dd-mon-yyyy hh24:mi:ss')
                                    AND TO_DATE('&end_date', 'dd-mon-yyyy hh24:mi:ss')
      AND sql.sql_id IN (
          SELECT DISTINCT stat.sql_id
          FROM dba_hist_sqlstat stat
          JOIN dba_hist_sqltext txt ON (stat.sql_id = txt.sql_id)
          JOIN dba_hist_snapshot snap ON (stat.snap_id = snap.snap_id)
          WHERE snap.begin_interval_time BETWEEN TO_DATE('&start_date', 'dd-mon-yyyy hh24:mi:ss')
                                             AND TO_DATE('&end_date', 'dd-mon-yyyy hh24:mi:ss')
            AND txt.command_type IN (2, 6, 7)
      )
) a
ORDER BY a.snap_id ASC;
```

### Generate AWR Report

```sql
@?/rdbms/admin/awrrpt.sql
```

---

## 8. ASH Analysis

Use ASH to analyze active session history for immediate issues.

### ASH Top Wait Events (Last 10 Minutes)

```sql
SELECT event,
       COUNT(*) samples,
       ROUND(COUNT(*) * 100/SUM(COUNT(*)) OVER (), 2) pct
FROM v$active_session_history
WHERE sample_time > SYSDATE - 10/1440
GROUP BY event
ORDER BY samples DESC
FETCH FIRST 15 ROWS ONLY;
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

### ASH by SID (Last 5 Minutes)

```sql
SELECT DISTINCT sql_id,
                session_serial#
FROM v$active_session_history
WHERE sample_time > SYSDATE - INTERVAL '5' MINUTE
  AND session_id = &sid;
```

### Last Running SQL from ASH

```sql
SELECT inst_id,
       sample_time,
       session_id,
       session_serial#,
       sql_id
FROM gv$active_session_history
WHERE sql_id IS NOT NULL
ORDER BY sample_time DESC
FETCH FIRST 100 ROWS ONLY;
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

### Generate ASH Report

```sql
@?/rdbms/admin/ashrpt.sql
```

---

## 9. SQL Tuning Advisor

Use SQL Tuning Advisor for SQL optimization recommendations.

### Find SQL_ID

```sql
SELECT sql_id,
       sql_text
FROM v$sql
WHERE sql_text LIKE '%&search_text%'
  AND sql_text NOT LIKE '%v$sql%';
```

### Create Tuning Task

```sql
SET SERVEROUTPUT ON
DECLARE
    mv_sql_tune_task_id VARCHAR2(100);
BEGIN
    mv_sql_tune_task_id := DBMS_SQLTUNE.create_tuning_task(
        sql_id      => '&sql_id',
        scope       => DBMS_SQLTUNE.scope_comprehensive,
        time_limit  => 60,
        task_name   => 'TUNING_TASK_&task_suffix',
        description => 'Tuning task for SQL_ID: &sql_id'
    );
    DBMS_OUTPUT.put_line('Task ID: ' || mv_sql_tune_task_id);
END;
/
```

### Execute Tuning Task

```sql
EXEC DBMS_SQLTUNE.execute_tuning_task(task_name => 'TUNING_TASK_&task_suffix');
```

### Check Task Status

```sql
SELECT task_name,
       status
FROM dba_advisor_log
WHERE task_name LIKE 'TUNING_TASK%'
ORDER BY execution_start DESC;
```

### View Recommendations

```sql
SET LONG 10000
SET PAGESIZE 1000
SET LINESIZE 200

SELECT DBMS_SQLTUNE.report_tuning_task('TUNING_TASK_&task_suffix') AS recommendations
FROM dual;
```

### Run SQL Tuning Report

```sql
@?/rdbms/admin/sqltrpt.sql
```

### SQL Profile Management

**Create SQL Profile:**
```sql
EXEC DBMS_SQLTUNE.accept_sql_profile(
    task_name   => '&task_name',
    task_owner  => 'SYS',
    replace     => TRUE,
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
EXEC DBMS_SQLTUNE.drop_sql_profile('&sql_profile_name');
```

**Disable SQL Profile:**
```sql
EXEC DBMS_SQLTUNE.alter_sql_profile(
    '&sql_profile_name',
    'STATUS',
    'DISABLED'
);
```

**Find SQL Profile by SQL Text:**
```sql
SELECT name,
       sql_text
FROM dba_sql_profiles
WHERE sql_text LIKE '%&search_text%';
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
       SUBSTR(sql_text, 1, 100) sql_text
FROM v$sql
WHERE sql_profile IS NOT NULL
ORDER BY elapsed_time DESC;
```

---

## 10. Database Statistics

Monitor database statistics collection and optimizer statistics.

### Statistics Status by Client

```sql
SELECT client_name,
       status
FROM dba_autotask_task
WHERE client_name LIKE 'auto optimizer %';
```

### Last Analyzed Tables by Owner

```sql
SELECT owner,
       MIN(last_analyzed)
FROM dba_tables
GROUP BY owner
ORDER BY 2 DESC;
```

### Table Modifications

```sql
SELECT table_owner,
       a.table_name,
       inserts,
       updates,
       deletes,
       num_rows,
       last_analyzed
FROM dba_tab_modifications a,
     dba_tables b
WHERE a.table_owner = b.owner
  AND a.table_name = b.table_name
  AND owner <> 'SYS'
ORDER BY 1, 2;
```

### Scheduler Jobs for Statistics

```sql
SELECT job_name,
       schedule_name,
       schedule_type,
       enabled,
       last_start_date
FROM dba_scheduler_jobs
WHERE program_name = 'GATHER_STATS_PROG';
```

### System Statistics

```sql
SELECT pname,
       pval1
FROM sys.aux_stats$
WHERE sname = 'SYSSTATS_MAIN';
```

### Autotask Job History

```sql
COLUMN client_name FORMAT A32
COLUMN job_status FORMAT A10
COLUMN job_start_time FORMAT A40

SELECT client_name,
       job_name,
       job_status,
       job_start_time,
       job_duration
FROM dba_autotask_job_history
WHERE job_start_time > SYSTIMESTAMP - 7
ORDER BY 1, 4;
```

### Auto Task Operations

```sql
SELECT client_name,
       operation_name
FROM dba_autotask_operation;
```

### Enable/Disable SQL Tuning Advisor

```sql
-- Enable
BEGIN
    DBMS_AUTO_TASK_ADMIN.enable(
        client_name  => 'sql tuning advisor',
        operation    => NULL,
        window_name  => NULL
    );
END;
/

-- Disable
BEGIN
    DBMS_AUTO_TASK_ADMIN.disable(
        client_name  => 'sql tuning advisor',
        operation    => NULL,
        window_name  => NULL
    );
END;
/
```

### Disable Auto Space Advisor

```sql
BEGIN
    DBMS_AUTO_TASK_ADMIN.disable(
        client_name  => 'auto space advisor',
        operation    => NULL,
        window_name  => NULL
    );
END;
/
```

---

## 11. Performance Spikes Detection

Detect and analyze performance spikes using historical data.

### Parallel Process Spikes

```sql
COLUMN program FORMAT A25
COLUMN machine FORMAT A10
COLUMN module FORMAT A25

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
    WHERE sample_time > TO_DATE('&start_time', 'DD-MON-YYYY HH24:MI:SS')
      AND sample_time < TO_DATE('&end_time', 'DD-MON-YYYY HH24:MI:SS')
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
           user_id,
           program,
           module,
           machine,
           sql_id
    FROM sys.dba_hist_active_sess_history
    WHERE sample_time > TO_DATE('&start_time', 'DD-MON-YYYY HH24:MI:SS')
      AND sample_time < TO_DATE('&end_time', 'DD-MON-YYYY HH24:MI:SS')
      AND event = '&event_name'
    GROUP BY user_id, program, module, machine, sql_id
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 20;
```

### Temp Tablespace Spikes

```sql
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
    WHERE sample_time > TO_DATE('&start_time', 'DD-MON-YYYY HH24:MI:SS')
      AND sample_time < TO_DATE('&end_time', 'DD-MON-YYYY HH24:MI:SS')
      AND temp_space_allocated > 1024 * 1024 * 1024
    GROUP BY user_id, program, module, machine, sql_id
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 20;
```

### PGA Spikes

```sql
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
    WHERE sample_time > TO_DATE('&start_time', 'DD-MON-YYYY HH24:MI:SS')
      AND sample_time < TO_DATE('&end_time', 'DD-MON-YYYY HH24:MI:SS')
      AND pga_allocated > 1024 * 1024 * 1024
    GROUP BY user_id, program, module, machine, sql_id
    ORDER BY COUNT(*) DESC
)
WHERE ROWNUM <= 20;
```

### Actual PGA Allocation Details

```sql
COLUMN pga_allocated FORMAT 999,999,999,999

SELECT /*+ PARALLEL */
       sql_id,
       TO_CHAR(sample_time, 'DD-MON-YYYY HH24:MI:SS') AS sample_time,
       pga_allocated
FROM sys.dba_hist_active_sess_history
WHERE sample_time > TO_DATE('&start_time', 'DD-MON-YYYY HH24:MI:SS')
  AND sample_time < TO_DATE('&end_time', 'DD-MON-YYYY HH24:MI:SS')
  AND sql_id = '&sql_id'
ORDER BY sample_time;
```

---

## Buffer Busy Waits Analysis

Detailed analysis of buffer busy waits and contention.

### Identify Sessions with Buffer Busy Waits

```sql
SELECT p1 "File #",
       p2 "Block #",
       p3 "Reason Code"
FROM v$session_wait
WHERE event = 'buffer busy waits';
```

### Find Segments Causing Buffer Busy Waits

```sql
SELECT owner,
       segment_name,
       segment_type
FROM dba_extents
WHERE file_id = &p1
  AND &p2 BETWEEN block_id AND block_id + blocks - 1;
```

### Detailed Buffer Busy Waits Analysis

```sql
-- Segment Header waits
SELECT 'Segment Header' class,
       a.segment_type,
       a.segment_name,
       a.partition_name
FROM dba_segments a,
     v$session_wait b
WHERE a.header_file = b.p1
  AND a.header_block = b.p2
  AND b.event = 'buffer busy waits'
UNION
-- Freelist Groups waits
SELECT 'Freelist Groups' class,
       a.segment_type,
       a.segment_name,
       a.partition_name
FROM dba_segments a,
     v$session_wait b
WHERE b.p2 BETWEEN a.header_block + 1 AND (a.header_block + a.freelist_groups)
  AND a.header_file = b.p1
  AND a.freelist_groups > 1
  AND b.event = 'buffer busy waits'
UNION
-- Data/Index block waits
SELECT a.segment_type || ' block' class,
       a.segment_type,
       a.segment_name,
       a.partition_name
FROM dba_extents a,
     v$session_wait b
WHERE b.p2 BETWEEN a.block_id AND a.block_id + a.blocks - 1
  AND a.file_id = b.p1
  AND b.event = 'buffer busy waits'
  AND NOT EXISTS (
      SELECT 1
      FROM dba_segments
      WHERE header_file = b.p1
        AND header_block = b.p2
  );
```

### Buffer Busy Waits from AWR

```sql
SELECT n.owner,
       n.tablespace_name,
       n.object_name,
       CASE
           WHEN LENGTH(n.subobject_name) < 11 THEN n.subobject_name
           ELSE SUBSTR(n.subobject_name, LENGTH(n.subobject_name) - 9)
       END subobject_name,
       n.object_type,
       r.buffer_busy_waits,
       SUBSTR(TO_CHAR(r.ratio * 100, '999.9MI'), 1, 5) ratio
FROM stats$seg_stat_obj n,
     (SELECT *
      FROM (
          SELECT e.dataobj#,
                 e.obj#,
                 e.ts#,
                 e.dbid,
                 e.buffer_busy_waits - NVL(b.buffer_busy_waits, 0) buffer_busy_waits,
                 RATIO_TO_REPORT(e.buffer_busy_waits - NVL(b.buffer_busy_waits, 0)) OVER () ratio
          FROM stats$seg_stat e,
               stats$seg_stat b
          WHERE b.snap_id(+) = :bid
            AND e.snap_id = :eid
            AND b.dbid(+) = :dbid
            AND e.dbid = :dbid
            AND b.instance_number(+) = :inst_num
            AND e.instance_number = :inst_num
            AND b.ts#(+) = e.ts#
            AND b.obj#(+) = e.obj#
            AND b.dataobj#(+) = e.dataobj#
            AND e.buffer_busy_waits - NVL(b.buffer_busy_waits, 0) > 0
          ORDER BY buffer_busy_waits DESC
      ) d
      WHERE ROWNUM <= &top_n_segstat
     ) r
WHERE n.dataobj# = r.dataobj#
  AND n.obj# = r.obj#
  AND n.ts# = r.ts#
  AND n.dbid = r.dbid
ORDER BY buffer_busy_waits DESC;
```

---

## Quick Reference Commands

Essential Oracle scripts for immediate diagnostics.

### Generate Reports

```sql
-- Generate AWR Report
@?/rdbms/admin/awrrpt.sql

-- Generate ASH Report
@?/rdbms/admin/ashrpt.sql

-- Generate SQL Tuning Report
@?/rdbms/admin/sqltrpt.sql

-- Generate ADDM Report
@?/rdbms/admin/addmrpt.sql
```

### Database Version and Status

```sql
-- Check Database Version
SELECT * FROM v$version;

-- Check Instance Status
SELECT instance_name,
       status,
       database_status
FROM v$instance;

-- Check Database Uptime
SELECT TO_CHAR(startup_time, 'DD-MON-YYYY HH24:MI:SS') startup_time,
       FLOOR(SYSDATE - startup_time) || ' days ' ||
       FLOOR(((SYSDATE - startup_time) - FLOOR(SYSDATE - startup_time)) * 24) || ' hours' uptime
FROM v$instance;
```

### SID to SPID Mapping

```sql
-- Find SPID from SID
SELECT spid
FROM v$process
WHERE addr = (
    SELECT paddr
    FROM v$session
    WHERE sid = &sid
);

-- Find SID from SPID
SELECT sid
FROM v$session
WHERE paddr = (
    SELECT addr
    FROM v$process
    WHERE spid = '&spid'
);
```

### SQL Address to Text

```sql
-- Get SQL address from session
SELECT sql_address,
       sql_hash_value
FROM v$session
WHERE sid = &sid;

-- Get SQL text from address
SELECT sql_text
FROM v$sqltext
WHERE address = '&sql_address'
  AND hash_value = &hash_value
ORDER BY piece;
```

---

## Best Practices

### Performance Analysis Workflow

1. **Identify Problem Period**
   - Use AWR snapshots to identify problematic time periods
   - Check for performance spikes in CPU, I/O, or memory

2. **Analyze Top Wait Events**
   - Query AWR for top wait events during problem period
   - Focus on events consuming >15% of DB time

3. **Identify Problem SQL**
   - Use ASH to find SQL causing wait events
   - Check for high CPU or I/O intensive queries

4. **Analyze Execution Plans**
   - Review execution plans for problem SQL
   - Look for full table scans, nested loops with high row counts

5. **Apply Tuning**
   - Use SQL Tuning Advisor for recommendations
   - Consider SQL profiles, index creation, or statistics gathering

6. **Monitor Results**
   - Track performance metrics after changes
   - Compare AWR reports before and after tuning

### Key Performance Metrics

| Metric | Good Range | Action Threshold |
|--------|-----------|------------------|
| Buffer Cache Hit Ratio | >95% | <90% |
| Library Cache Hit Ratio | >95% | <90% |
| PGA Cache Hit Ratio | >95% | <90% |
| Parse to Execute Ratio | <10% | >20% |
| Soft Parse Ratio | >95% | <80% |

### Common Wait Events and Solutions

| Wait Event | Common Cause | Solution |
|------------|--------------|----------|
| db file sequential read | Index scans | Review indexes, consider full table scan |
| db file scattered read | Full table scans | Add indexes, use partitioning |
| log file sync | Commit frequency | Batch commits, check I/O performance |
| buffer busy waits | Block contention | Use reverse key indexes, increase freelists |
| gc buffer busy | RAC contention | Review application partitioning |
| enqueue waits | Lock contention | Reduce transaction time, check for deadlocks |
| library cache latch | Hard parsing | Use bind variables, increase shared pool |
| latch free | Various latches | Identify specific latch, reduce contention |

### Monitoring Schedule

- **Real-time**: v$session, v$sql, v$active_session_history
- **5-minute intervals**: ASH queries for immediate issues
- **Hourly**: AWR snapshots (default)
- **Daily**: Review AWR reports, check trends
- **Weekly**: Analyze historical performance, capacity planning
- **Monthly**: Comprehensive performance review, statistics maintenance

### AWR Configuration

```sql
-- Modify AWR snapshot interval (default 60 minutes)
EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
    interval => 30
);

-- Modify AWR retention (default 8 days, specify in minutes)
EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
    retention => 14400  -- 10 days
);

-- Create manual snapshot
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot();
```

---

## Important Notes

⚠️ **Performance Impact**: Many queries access dynamic performance views and can impact production systems. Always test in non-production first.

⚠️ **AWR/ASH License**: AWR and ASH features require Oracle Diagnostics Pack license for production use.

⚠️ **Privileges Required**: Most queries require SELECT_CATALOG_ROLE or DBA role.

⚠️ **Statistics**: Ensure statistics are current for accurate execution plans:
```sql
EXEC DBMS_STATS.gather_schema_stats('&schema_name');
EXEC DBMS_STATS.gather_database_stats();
```

⚠️ **Bind Variables**: Always use bind variables in the queries where prompted (e.g., :bid, :eid, &sid).

---

## Troubleshooting Guide

### High CPU Usage

1. Identify top CPU consumers using ASH
2. Check for high parse rates (library cache misses)
3. Review SQL with high CPU time
4. Analyze execution plans for inefficiency
5. Consider SQL profiles or index optimization

### High I/O Wait

1. Identify top I/O intensive SQL
2. Check datafile I/O distribution
3. Review segments with high physical reads
4. Analyze wait events (db file sequential/scattered read)
5. Consider adding indexes or partitioning

---

## Contributing

Contributions are welcome! When adding new queries:
- Test thoroughly in non-production first
- Include clear comments explaining purpose
- Follow existing formatting standards
- Add use case examples where applicable
- Document any required privileges

---

## License

This collection is provided for educational and professional use. All queries are based on Oracle documentation and community best practices.

---

**Repository Version**: 1.0  
**Last Updated**: 2025  
**Oracle Versions**: Compatible with 10g, 11g, 12c, 18c, 19c, 21c, 23c  
**Maintained By**: Oracle DBA Community

---

## Additional Resources

- [Oracle Database Performance Tuning Guide](https://docs.oracle.com/en/database/)
- [Oracle Support - My Oracle Support](https://support.oracle.com)
- [Oracle Database Reference](https://docs.oracle.com/en/database/)
- [Oracle Wait Events Documentation](https://docs.oracle.com/en/database/)

---

**End of Document**
