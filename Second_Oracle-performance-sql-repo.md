# IN PROGRESS
# Oracle Performance SQL Repository

---

## Table of Contents

1. [Latch Analysis](#1-latch-analysis)
2. [Shared Pool & Library Cache](#2-shared-pool--library-cache)
3. [Top SQLs & Reports](#3-top-sqls--reports)
4. [Session & CPU Stats](#4-session--cpu-stats)
5. [I/O Analysis](#5-io-analysis)
6. [PGA & Memory Advice](#6-pga--memory-advice)

---

## 1. Latch Analysis

```sql
-- Latch Activity summary (compare two snapshots)
SELECT b.name name,
       e.gets - b.gets gets,
       to_number(decode(e.gets, b.gets, null,(e.misses - b.misses) * 100/(e.gets - b.gets))) missed,
       to_number(decode(e.misses, b.misses, null,(e.sleeps - b.sleeps)/(e.misses - b.misses))) sleeps,
       (e.wait_time - b.wait_time)/1000000 wt,
       e.immediate_gets - b.immediate_gets nowai,
       to_number(decode(e.immediate_gets,b.immediate_gets, null,(e.immediate_misses - b.immediate_misses) * 100 /(e.immediate_gets - b.immediate_gets))) imiss
FROM stats$latch b, stats$latch e
WHERE b.snap_id = :bid
  AND e.snap_id = :eid
  AND b.dbid = :dbid
  AND e.dbid = :dbid
  AND b.instance_number = :inst_num
  AND e.instance_number = :inst_num
  AND b.name = e.name
  AND (e.gets - b.gets + e.immediate_gets - b.immediate_gets) > 0
ORDER BY b.name;

-- Latch Sleep breakdown
SELECT b.name name,
       e.gets - b.gets gets,
       e.misses - b.misses misses,
       e.sleeps - b.sleeps sleeps,
       e.spin_gets - b.spin_gets spin_gets
FROM stats$latch b, stats$latch e
WHERE b.snap_id = :bid
  AND e.snap_id = :eid
  AND b.dbid = :dbid
  AND e.dbid = :dbid
  AND b.instance_number = :inst_num
  AND e.instance_number = :inst_num
  AND b.name = e.name
  AND e.sleeps - b.sleeps > 0
ORDER BY misses DESC;

-- Live latch view
SELECT NAME, GETS, MISSES, SLEEPS, SPIN_GETS, WAIT_TIME
FROM V$LATCH
ORDER BY GETS DESC FETCH FIRST 10 ROWS ONLY;
```

---

## 2. Shared Pool & Library Cache

```sql
-- Shared Pool usage
SELECT NAME, round(BYTES/(1024*1024),0) MB
FROM V$SGASTAT
WHERE POOL = 'shared pool'
ORDER BY BYTES DESC FETCH FIRST 20 ROWS ONLY;

-- Library cache hit ratios
SELECT NAMESPACE, GETS, GETHITRATIO, PINS, PINHITRATIO, RELOADS, INVALIDATIONS
FROM V$LIBRARYCACHE;

SELECT SUM(PINS - RELOADS) * 100 / NULLIF(SUM(PINS),0) AS library_cache_hit_pct
FROM V$LIBRARYCACHE;

-- Heavy sharable memory consumers
SELECT substr(sql_text,1,120) sql_snippet,
       count(*) occurrences,
       sum(sharable_mem) total_sharable_mem,
       sum(executions) total_executions
FROM v$sql
GROUP BY substr(sql_text,1,120)
HAVING sum(sharable_mem) > 0
ORDER BY total_sharable_mem DESC FETCH FIRST 20 ROWS ONLY;

-- SQLs with excessive versions
SELECT substr(sql_text,1,80) stmt,
       count(*) versions,
       sum(executions) executions
FROM v$sql
GROUP BY substr(sql_text,1,80)
HAVING count(*) > 10
ORDER BY versions DESC;
```

---

## 3. Top SQLs & Reports

```sql
-- Top SQL by CPU
SELECT sql_id, executions, cpu_time/1000000 cpu_sec, elapsed_time/1000000 elapsed_sec, buffer_gets, disk_reads, substr(sql_text,1,120) sql_text
FROM v$sqlarea
WHERE executions > 0
ORDER BY cpu_time DESC FETCH FIRST 20 ROWS ONLY;

-- Top SQL by Buffer Gets
SELECT sql_id, executions, buffer_gets, buffer_gets/executions gets_per_exec, substr(sql_text,1,120) sql_text
FROM v$sqlarea
WHERE executions > 0
ORDER BY buffer_gets DESC FETCH FIRST 20 ROWS ONLY;

-- Consolidated Top-N Report
DROP TABLE tmp_top10 PURGE;
CREATE TABLE tmp_top10 AS SELECT * FROM V$SQLAREA WHERE 1=2;

INSERT INTO tmp_top10 (hash_value, executions, buffer_gets, disk_reads, rows_processed, sorts, parse_calls, sharable_mem, version_count)
SELECT hash_value, executions, buffer_gets, disk_reads, rows_processed, sorts, parse_calls, sharable_mem, version_count
FROM (SELECT * FROM v$sqlarea WHERE executions > 0 ORDER BY buffer_gets DESC) WHERE ROWNUM <= 10;

MERGE INTO tmp_top10 t
USING (SELECT * FROM (SELECT * FROM v$sqlarea WHERE executions > 0 ORDER BY disk_reads DESC) WHERE ROWNUM <= 10) s
ON (t.hash_value = s.hash_value)
WHEN NOT MATCHED THEN INSERT (hash_value, executions, buffer_gets, disk_reads, rows_processed, sorts, parse_calls, sharable_mem, version_count)
VALUES (s.hash_value, s.executions, s.buffer_gets, s.disk_reads, s.rows_processed, s.sorts, s.parse_calls, s.sharable_mem, s.version_count);

SELECT count_hash, hash_value, MAX(executions) executions, MAX(buffer_gets) buffer_gets, MAX(disk_reads) disk_reads
FROM (SELECT COUNT(hash_value) count_hash, hash_value, executions, buffer_gets, disk_reads FROM tmp_top10 GROUP BY hash_value)
GROUP BY count_hash, hash_value
ORDER BY count_hash DESC FETCH FIRST 5 ROWS ONLY;
```

---

## 4. Session & CPU Stats

```sql
-- Active sessions
SELECT SID, SERIAL#, USERNAME, STATUS, SCHEMANAME, LOGON_TIME
FROM V$SESSION
WHERE STATUS = 'ACTIVE' AND USERNAME IS NOT NULL
ORDER BY LOGON_TIME DESC;

-- Top sessions by CPU usage
SELECT nvl(ss.USERNAME,'ORACLE PROC') username, se.SID, VALUE/100 cpu_seconds
FROM v$session ss, v$sesstat se, v$statname sn
WHERE se.STATISTIC# = sn.STATISTIC#
  AND NAME like '%CPU used by this session%'
  AND se.SID = ss.SID
ORDER BY VALUE DESC;

-- Sessions with highest DB time, CPU, IO (last 30 min)
SELECT s.sid, s.serial#, p.spid AS os_pid, s.username, s.module,
       st.value/100 "DB_Time_sec",
       stcpu.value/100 "CPU_sec",
       round(stcpu.value / NULLIF(st.value,0) * 100,2) as pct_cpu
FROM v$sesstat st
JOIN v$statname sn ON st.statistic# = sn.statistic# AND sn.name = 'DB time'
JOIN v$session s ON st.sid = s.sid
LEFT JOIN v$sesstat stcpu ON st.sid = stcpu.sid
LEFT JOIN v$statname sncpu ON stcpu.statistic# = sncpu.statistic# AND sncpu.name = 'CPU used by this session'
LEFT JOIN v$process p ON s.paddr = p.addr
WHERE s.last_call_et < 1800
ORDER BY st.value DESC;
```

---

## 5. I/O Analysis

```sql
-- Datafiles with highest I/O activity
SELECT name, phyrds PHYSICAL_READS, phywrts PHYSICAL_WRITES, readtim, writetim
FROM (SELECT name, phyrds, phywrts, readtim, writetim FROM v$filestat a JOIN v$datafile b ON a.file# = b.file# ORDER BY phyrds DESC)
WHERE ROWNUM <= 10;

-- I/O per tablespace
SELECT T.NAME, SUM(Physical_READS) Physical_READS,
       ROUND((RATIO_TO_REPORT(SUM(Physical_READS)) OVER ())*100,2) || '%' PERC_READS,
       SUM(Physical_WRITES) Physical_WRITES,
       ROUND((RATIO_TO_REPORT(SUM(Physical_WRITES)) OVER ())*100,2) || '%' PERC_WRITES
FROM (
  SELECT df.ts# ts#, df.name, fs.phyrds Physical_READS, fs.phywrts Physical_WRITES
  FROM v$datafile df JOIN v$filestat fs ON df.file# = fs.file#
) A
JOIN sys.ts$ T ON A.ts# = T.ts#
GROUP BY T.NAME
ORDER BY SUM(Physical_READS) DESC;

-- Top segments by logical and physical reads/writes
SELECT * FROM (
  SELECT owner, object_name, object_type, statistic_name, value
  FROM dba_objects o
  JOIN (SELECT obj#, statistic_name, value FROM v$segstat WHERE statistic_name IN ('logical reads','physical reads','physical writes')) s
    ON o.object_id = s.obj#
  ORDER BY value DESC
) WHERE ROWNUM <= 20;
```

---

## 6. PGA & Memory Advice

```sql
-- PGA target advice
SELECT round(PGA_TARGET_FOR_ESTIMATE / 1024 / 1024) AS PGA_Target_MB,
       ESTD_PGA_CACHE_HIT_PERCENTAGE,
       ESTD_OVERALLOC_COUNT
FROM V$PGA_TARGET_ADVICE
ORDER BY 1;

-- Session PGA allocations
SELECT NVL(a.username,'(oracle)') AS username,
       a.module,
       a.program,
       Trunc(b.value/1024) AS memory_kb
FROM v$session a
JOIN v$sesstat b ON a.sid = b.sid
JOIN v$statname c ON b.statistic# = c.statistic#
WHERE c.name = 'session pga memory'
  AND a.program IS NOT NULL
ORDER BY b.value DESC FETCH FIRST 50 ROWS ONLY;
```

---

