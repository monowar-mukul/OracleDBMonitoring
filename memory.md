# Memory and PGA Monitoring

This wiki provides comprehensive SQL queries for monitoring memory usage in Oracle databases, including session-level monitoring, process memory analysis, and performance optimization queries.

## Table of Contents
- [Database / Session Memory Monitoring](#session-memory-monitoring)
- [Process Memory Analysis](#process-memory-analysis)
- [User-Specific Memory Queries](#user-specific-memory-queries)
- [Memory Statistics and Trends](#memory-statistics-and-trends)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting Queries](#troubleshooting-queries)

## Session Memory Monitoring
### Database Memory Configuration

```sql
SET HEADING OFF
SET FEEDBACK OFF
SET PAGESIZE 0

-- Database Name
SELECT SUBSTR(a.name, 1, 30) "PARAM_NAME",
       DECODE(a.value, '', 'ASM', a.value) "VALUE_MB"
FROM v$parameter a
WHERE UPPER(a.name) = 'DB_NAME';

-- Memory Parameters
SELECT SUBSTR(a.name, 1, 30) "PARAM_NAME",
       ROUND(a.value/(1024 * 1024), 0) "VALUE_MB"
FROM v$parameter a
WHERE UPPER(a.name) IN (
    'SGA_MAX_SIZE',
    'SGA_TARGET',
    'SHARED_POOL_SIZE',
    'LARGE_POOL_SIZE',
    'JAVA_POOL_SIZE',
    'STREAMS_POOL_SIZE',
    'LOG_BUFFER',
    'DB_CACHE_SIZE',
    'DB_KEEP_CACHE_SIZE',
    'DB_RECYCLE_CACHE_SIZE',
    'PGA_AGGREGATE_TARGET'
)
ORDER BY 1;

-- Total PGA Allocated
SELECT 'total_PGA_allocated' "PARAM_NAME",
       ROUND(a.value/(1024 * 1024), 0) "VALUE_MB"
FROM v$pgastat a
WHERE name = 'total PGA allocated';

-- Maximum PGA Allocated
SELECT 'max_PGA_allocated' "PARAM_NAME",
       ROUND(a.value/(1024 * 1024), 0) "VALUE_MB"
FROM v$pgastat a
WHERE name = 'maximum PGA allocated';
```
### PGA Target Advice

```sql
SELECT ROUND(pga_target_for_estimate/1024/1024) pga_target_mb,
       estd_pga_cache_hit_percentage,
       estd_overalloc_count
FROM v$pga_target_advice
ORDER BY 1;
```

### Top Sessions by PGA Usage

```sql
SELECT NVL(s.username, '(oracle)') username,
       s.program,
       TRUNC(v.value/1024) memory_kb
FROM v$session s
JOIN v$sesstat v ON s.sid = v.sid
JOIN v$statname n ON v.statistic# = n.statistic#
WHERE n.name = 'session pga memory'
ORDER BY v.value DESC
FETCH FIRST 20 ROWS ONLY;
```
### Sort Statistics

```sql
SELECT *
FROM v$sysstat
WHERE name LIKE '%sorts%';
```

### Current Session Memory Overview

Display memory usage for all active sessions with PID and thread information:

```sql
SELECT 
    TO_CHAR(ssn.sid, '9999') || ' - ' || NVL(ssn.username, NVL(bgp.name, 'background')) ||
    ' @ ' || NVL(LOWER(ssn.machine), ins.host_name) AS "SESSION_INFO",
    TO_CHAR(prc.spid, '999999999') AS "PID_THREAD",
    TO_CHAR((se1.value/1024)/1024, '999G999G990D00') || ' MB' AS "CURRENT_PGA",
    TO_CHAR((se2.value/1024)/1024, '999G999G990D00') || ' MB' AS "MAX_PGA",
    ssn.status,
    ssn.program
FROM 
    v$sesstat se1, 
    v$sesstat se2, 
    v$session ssn, 
    v$bgprocess bgp, 
    v$process prc,
    v$instance ins, 
    v$statname stat1, 
    v$statname stat2
WHERE 
    se1.statistic# = stat1.statistic# AND stat1.name = 'session pga memory'
    AND se2.statistic# = stat2.statistic# AND stat2.name = 'session pga memory max'
    AND se1.sid = ssn.sid
    AND se2.sid = ssn.sid
    AND ssn.paddr = bgp.paddr (+)
    AND ssn.paddr = prc.addr (+)
    AND se1.value > 0
ORDER BY se1.value DESC;
```
### Session Memory Allocations

```sql
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

### Session CPU Usage

```sql
SELECT SUBSTR(name, 1, 30) parameter,
       ss.username || '(' || se.sid || ')' user_process,
       value
FROM v$session ss,
     v$sesstat se,
     v$statname sn
WHERE se.statistic# = sn.statistic#
  AND name LIKE '%CPU used by this session%'
  AND se.sid = ss.sid
ORDER BY SUBSTR(name, 1, 25), value DESC;
```

### Comprehensive Session Memory Details

Complete PGA and UGA memory information for all sessions:

```sql
SELECT
    s.sid,
    s.serial#,
    LPAD(s.username, 15) AS oracle_user,
    LPAD(s.osuser, 12) AS os_user,
    SUBSTR(s.program, 1, 20) AS program,
    LPAD(s.machine, 12) AS machine,
    s.status,
    -- PGA Memory Statistics
    (SELECT ROUND(ss.value/1024/1024, 2) 
     FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session pga memory') AS pga_current_mb,
    (SELECT ROUND(ss.value/1024/1024, 2) 
     FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session pga memory max') AS pga_max_mb,
    -- UGA Memory Statistics
    (SELECT ROUND(ss.value/1024/1024, 2) 
     FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session uga memory') AS uga_current_mb,
    (SELECT ROUND(ss.value/1024/1024, 2) 
     FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session uga memory max') AS uga_max_mb
FROM 
    v$session s
WHERE 
    s.username IS NOT NULL
    AND EXISTS (SELECT 1 FROM v$sesstat ss, v$statname sn
                WHERE ss.sid = s.sid 
                AND sn.statistic# = ss.statistic# 
                AND sn.name = 'session pga memory'
                AND ss.value > 1024*1024) -- Only sessions using > 1MB
ORDER BY pga_current_mb DESC NULLS LAST;
```
### Top CPU-Consuming Sessions

```sql
SELECT s.sid,
       s.serial#,
       s.username,
       p.spid,
       v.value/100 AS cpu_seconds
FROM v$sesstat v,
     v$statname n,
     v$session s,
     v$process p
WHERE n.statistic# = v.statistic#
  AND n.name LIKE '%CPU used by this session%'
  AND v.sid = s.sid
  AND s.paddr = p.addr
ORDER BY v.value DESC
FETCH FIRST 10 ROWS ONLY;
```
### Top Memory Consuming Sessions

Identify sessions consuming the most memory:

```sql
SELECT 
    s.sid,
    s.username,
    s.program,
    s.machine,
    s.status,
    ROUND(pga.value/1024/1024, 2) AS pga_mb,
    ROUND(uga.value/1024/1024, 2) AS uga_mb,
    ROUND((pga.value + uga.value)/1024/1024, 2) AS total_mb,
    s.sql_id,
    s.last_call_et AS idle_seconds
FROM 
    v$session s,
    v$sesstat pga,
    v$sesstat uga,
    v$statname pga_name,
    v$statname uga_name
WHERE 
    s.sid = pga.sid
    AND s.sid = uga.sid
    AND pga.statistic# = pga_name.statistic#
    AND uga.statistic# = uga_name.statistic#
    AND pga_name.name = 'session pga memory'
    AND uga_name.name = 'session uga memory'
    AND s.username IS NOT NULL
    AND (pga.value + uga.value) > 50*1024*1024 -- Sessions using > 50MB
ORDER BY total_mb DESC;
```
### List memory allocations RAC Sessions.
```
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 9
COLUMN username FORMAT A20
COLUMN module FORMAT A40
COLUMN program FORMAT A40
 
SELECT a.inst_id,
       NVL(a.username,'(oracle)') AS username,
       a.module,
       a.program,
       Trunc(b.value/1024) AS memory_kb
FROM   gv$session a,
       gv$sesstat b,
       gv$statname c
WHERE  a.sid = b.sid
AND    a.inst_id = b.inst_id
AND    b.statistic# = c.statistic#
AND    b.inst_id = c.inst_id
AND    c.name = 'session pga memory'
AND    a.program IS NOT NULL
ORDER BY b.value DESC;
```
```
SET HEADING OFF
SET FEEDBACK OFF
SET ECHO OFF
SET TERMOUT OFF
SET VERIFY OFF
SET PAGESIZE 0
SET LINESIZE 80
SET WRAP OFF

SELECT ssn.osuser,to_char((se1.value/1024)/1024, '999G999G990D00') || ' MB' " CURRENT SIZE", 
                                    to_char((se2.value/1024)/1024, '999G999G990D00') || ' MB' " MAXIMUM SIZE", 
                                    ssn.sid||','||ssn.serial#|| ' - ' || nvl(ssn.username, nvl(bgp.name, 'background')) || ': ' 
                                            || nvl(lower(ssn.machine), ins.host_name) "Sid,Serial - User", 
                                                ssn.process,
                                                prc.spid Spid,
                                                prc.pid Pid,
                                                ssn.action,ssn.logon_time,
                                                ssn.module,
                                                ssn.program,
                                                q.sql_text ,
                                                q.optimizer_mode
                                    FROM v$sesstat se1, v$sesstat se2, v$session ssn, v$bgprocess bgp, v$process prc, v$instance ins 
                                    , (select distinct hash_value,sql_text,optimizer_mode from v$sql ) q 
                                    WHERE se1.statistic# = 20 
                                    AND se2.statistic# = 21 
                                    AND se1.sid = ssn.sid 
                                    AND se2.sid = ssn.sid 
                                    AND ssn.paddr = bgp.paddr (+) 
                                    AND ssn.paddr = prc.addr (+) 
                                    AND ssn.sql_hash_value = q.hash_value(+) 
                                    ORDER BY se1.value DESC ;
```
```
SET HEADING OFF
SET FEEDBACK OFF
SET ECHO OFF
SET TERMOUT OFF
SET VERIFY OFF
SET PAGESIZE 0
SET LINESIZE 80
SET WRAP OFF
SELECT ssn.osuser,to_char((se1.value/1024)/1024, '999G999G990D00') || ' MB' " CURRENT SIZE", 
                                    to_char((se2.value/1024)/1024, '999G999G990D00') || ' MB' " MAXIMUM SIZE", 
                                    ssn.sid||','||ssn.serial#|| ' - ' || nvl(ssn.username, nvl(bgp.name, 'background')) || ': ' 
                                            || nvl(lower(ssn.machine), ins.host_name) "Sid,Serial - User", 
                                                ssn.process,
                                                prc.spid Spid,
                                                prc.pid Pid,
                                                ssn.action,ssn.logon_time,
                                                ssn.module,
                                                ssn.program,
                                                q.sql_text ,
                                                q.optimizer_mode
                                    FROM v$sesstat se1, v$sesstat se2, v$session ssn, v$bgprocess bgp, v$process prc, v$instance ins 
                                    , (select distinct hash_value,sql_text,optimizer_mode from v$sql ) q 
                                    WHERE se1.statistic# = 20 
                                    AND se2.statistic# = 21 
                                    AND se1.sid = ssn.sid 
                                    AND se2.sid = ssn.sid 
                                    AND ssn.paddr = bgp.paddr (+) 
                                    AND ssn.paddr = prc.addr (+) 
                                    AND ssn.sql_hash_value = q.hash_value(+) 
                                    ORDER BY ssn.osuser DESC ;
```

## Process Memory Analysis

### Memory Usage by Category

Analyze memory allocation by category (SQL, PL/SQL, etc.) for a specific session:

```sql
SELECT
    pm.category,
    ROUND(pm.allocated/1024/1024, 2) AS allocated_mb,
    ROUND(pm.used/1024/1024, 2) AS used_mb,
    ROUND(pm.max_allocated/1024/1024, 2) AS max_allocated_mb,
    ROUND((pm.used/pm.allocated)*100, 2) AS utilization_pct
FROM 
    v$process_memory pm
WHERE 
    pm.pid = (
        SELECT p.pid
        FROM v$process p, v$session s
        WHERE p.addr = s.paddr
        AND s.sid = &SID
    )
ORDER BY allocated_mb DESC;
```

### Process Memory Summary

Overall process memory statistics:

```sql
SELECT 
    p.pid,
    p.spid,
    p.program,
    ROUND(SUM(pm.allocated)/1024/1024, 2) AS total_allocated_mb,
    ROUND(SUM(pm.used)/1024/1024, 2) AS total_used_mb,
    ROUND(SUM(pm.max_allocated)/1024/1024, 2) AS total_max_mb,
    COUNT(*) AS memory_categories
FROM 
    v$process p,
    v$process_memory pm
WHERE 
    p.pid = pm.pid
GROUP BY p.pid, p.spid, p.program
HAVING SUM(pm.allocated) > 10*1024*1024 -- Processes using > 10MB
ORDER BY total_allocated_mb DESC;
```

## User-Specific Memory Queries

### Memory Usage for Specific User

Monitor memory consumption for a particular database user:

```sql
SELECT 
    sess.sid,
    sess.serial#,
    sess.program,
    sess.machine,
    sess.status,
    sess.logon_time,
    ROUND(pga_mem.value/1024/1024, 2) AS current_pga_mb,
    ROUND(pga_max.value/1024/1024, 2) AS max_pga_mb,
    ROUND(uga_mem.value/1024/1024, 2) AS current_uga_mb,
    ROUND(uga_max.value/1024/1024, 2) AS max_uga_mb,
    sess.sql_id,
    sess.last_call_et AS idle_seconds
FROM 
    v$session sess,
    v$sesstat pga_mem,
    v$sesstat pga_max,
    v$sesstat uga_mem,
    v$sesstat uga_max,
    v$statname pga_mem_name,
    v$statname pga_max_name,
    v$statname uga_mem_name,
    v$statname uga_max_name
WHERE 
    sess.sid = pga_mem.sid
    AND sess.sid = pga_max.sid
    AND sess.sid = uga_mem.sid
    AND sess.sid = uga_max.sid
    AND pga_mem.statistic# = pga_mem_name.statistic#
    AND pga_max.statistic# = pga_max_name.statistic#
    AND uga_mem.statistic# = uga_mem_name.statistic#
    AND uga_max.statistic# = uga_max_name.statistic#
    AND pga_mem_name.name = 'session pga memory'
    AND pga_max_name.name = 'session pga memory max'
    AND uga_mem_name.name = 'session uga memory'
    AND uga_max_name.name = 'session uga memory max'
    AND sess.username = UPPER('&USERNAME')
ORDER BY current_pga_mb DESC;
```

### User Memory Aggregation

Total memory usage by database user:

```sql
SELECT 
    s.username,
    COUNT(*) AS session_count,
    ROUND(SUM(pga.value)/1024/1024, 2) AS total_pga_mb,
    ROUND(AVG(pga.value)/1024/1024, 2) AS avg_pga_mb,
    ROUND(SUM(uga.value)/1024/1024, 2) AS total_uga_mb,
    ROUND(AVG(uga.value)/1024/1024, 2) AS avg_uga_mb,
    ROUND(SUM(pga.value + uga.value)/1024/1024, 2) AS total_memory_mb
FROM 
    v$session s,
    v$sesstat pga,
    v$sesstat uga,
    v$statname pga_name,
    v$statname uga_name
WHERE 
    s.sid = pga.sid
    AND s.sid = uga.sid
    AND pga.statistic# = pga_name.statistic#
    AND uga.statistic# = uga_name.statistic#
    AND pga_name.name = 'session pga memory'
    AND uga_name.name = 'session uga memory'
    AND s.username IS NOT NULL
GROUP BY s.username
ORDER BY total_memory_mb DESC;
```

## Memory Statistics and Trends

### Database-Wide Memory Statistics

Overall memory usage across the database:

```sql
SELECT 
    'Total Sessions' AS metric,
    COUNT(*) AS value,
    'Count' AS unit
FROM v$session 
WHERE username IS NOT NULL
UNION ALL
SELECT 
    'Total PGA Memory' AS metric,
    ROUND(SUM(pga.value)/1024/1024/1024, 2) AS value,
    'GB' AS unit
FROM v$sesstat pga, v$statname pga_name
WHERE pga.statistic# = pga_name.statistic#
AND pga_name.name = 'session pga memory'
UNION ALL
SELECT 
    'Total UGA Memory' AS metric,
    ROUND(SUM(uga.value)/1024/1024/1024, 2) AS value,
    'GB' AS unit
FROM v$sesstat uga, v$statname uga_name
WHERE uga.statistic# = uga_name.statistic#
AND uga_name.name = 'session uga memory'
ORDER BY metric;
```

### Memory Usage Histogram

Distribution of memory usage across sessions:

```sql
SELECT 
    memory_range,
    COUNT(*) AS session_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage
FROM (
    SELECT 
        s.sid,
        CASE 
            WHEN pga.value < 1024*1024 THEN '< 1 MB'
            WHEN pga.value < 10*1024*1024 THEN '1-10 MB'
            WHEN pga.value < 50*1024*1024 THEN '10-50 MB'
            WHEN pga.value < 100*1024*1024 THEN '50-100 MB'
            WHEN pga.value < 500*1024*1024 THEN '100-500 MB'
            ELSE '> 500 MB'
        END AS memory_range
    FROM 
        v$session s,
        v$sesstat pga,
        v$statname pga_name
    WHERE 
        s.sid = pga.sid
        AND pga.statistic# = pga_name.statistic#
        AND pga_name.name = 'session pga memory'
        AND s.username IS NOT NULL
)
GROUP BY memory_range
ORDER BY 
    CASE memory_range
        WHEN '< 1 MB' THEN 1
        WHEN '1-10 MB' THEN 2
        WHEN '10-50 MB' THEN 3
        WHEN '50-100 MB' THEN 4
        WHEN '100-500 MB' THEN 5
        WHEN '> 500 MB' THEN 6
    END;
```

## Performance Optimization

### Identify Memory-Intensive SQL

Find SQL statements consuming significant memory:

```sql
SELECT 
    s.sql_id,
    s.sql_child_number,
    ROUND(pga.value/1024/1024, 2) AS pga_mb,
    s.username,
    s.program,
    s.status,
    t.sql_text
FROM 
    v$session s,
    v$sesstat pga,
    v$statname pga_name,
    v$sqltext t
WHERE 
    s.sid = pga.sid
    AND pga.statistic# = pga_name.statistic#
    AND pga_name.name = 'session pga memory'
    AND s.sql_id = t.sql_id (+)
    AND t.piece = 0
    AND pga.value > 100*1024*1024 -- Sessions using > 100MB
ORDER BY pga_mb DESC;
```

### Memory Usage by Program Type

Analyze memory consumption by client program:

```sql
SELECT 
    SUBSTR(s.program, 1, 30) AS program,
    COUNT(*) AS session_count,
    ROUND(SUM(pga.value)/1024/1024, 2) AS total_pga_mb,
    ROUND(AVG(pga.value)/1024/1024, 2) AS avg_pga_mb,
    ROUND(MAX(pga.value)/1024/1024, 2) AS max_pga_mb
FROM 
    v$session s,
    v$sesstat pga,
    v$statname pga_name
WHERE 
    s.sid = pga.sid
    AND pga.statistic# = pga_name.statistic#
    AND pga_name.name = 'session pga memory'
    AND s.username IS NOT NULL
    AND s.program IS NOT NULL
GROUP BY SUBSTR(s.program, 1, 30)
HAVING COUNT(*) > 1
ORDER BY total_pga_mb DESC;
```
### Memory Allocation Recommendations

```sql
-- Set SGA and PGA
ALTER SYSTEM SET sga_max_size = 12G SCOPE=SPFILE;
ALTER SYSTEM SET sga_target = 12G SCOPE=SPFILE;
ALTER SYSTEM SET pga_aggregate_target = 4G SCOPE=SPFILE;

-- Set Block-Specific Cache Sizes
ALTER SYSTEM SET db_8k_cache_size = 6G SCOPE=SPFILE;
ALTER SYSTEM SET db_16k_cache_size = 4500M SCOPE=SPFILE;
ALTER SYSTEM SET db_32k_cache_size = 1500M SCOPE=SPFILE;
```
### Database Buffer Cache Advice
```
SET HEADING OFF
SET FEEDBACK OFF
SET ECHO OFF
SET TERMOUT OFF
SET VERIFY OFF
SET PAGESIZE 0
SET LINESIZE 80
SET WRAP OFF
SELECT ssn.osuser,to_char((se1.value/1024)/1024, '999G999G990D00') || ' MB' " CURRENT SIZE", 
                                    to_char((se2.value/1024)/1024, '999G999G990D00') || ' MB' " MAXIMUM SIZE", 
                                    ssn.sid||','||ssn.serial#|| ' - ' || nvl(ssn.username, nvl(bgp.name, 'background')) || ': ' 
                                            || nvl(lower(ssn.machine), ins.host_name) "Sid,Serial - User", 
                                                ssn.process,
                                                prc.spid Spid,
                                                prc.pid Pid,
                                                ssn.action,ssn.logon_time,
                                                ssn.module,
                                                ssn.program,
                                                q.sql_text ,
                                                q.optimizer_mode
                                    FROM v$sesstat se1, v$sesstat se2, v$session ssn, v$bgprocess bgp, v$process prc, v$instance ins 
                                    , (select distinct hash_value,sql_text,optimizer_mode from v$sql ) q 
                                    WHERE se1.statistic# = 20 
                                    AND se2.statistic# = 21 
                                    AND se1.sid = ssn.sid 
                                    AND se2.sid = ssn.sid 
                                    AND ssn.paddr = bgp.paddr (+) 
                                    AND ssn.paddr = prc.addr (+) 
                                    AND ssn.sql_hash_value = q.hash_value(+) 
                                    ORDER BY ssn.osuser DESC ;
```

### Database Shared Pool Advice
```
SELECT (SELECT ROUND(value/1024/1024,0) FROM v$parameter
      WHERE name = 'shared_pool_size') "Current Mb"
, shared_pool_size_for_estimate "Projected Mb"
, ROUND(shared_pool_size_factor*100) "%"
, ESTD_LC_SIZE "Library Mb"
, ESTD_LC_TIME_SAVED "Parse Savings"
,ESTD_LC_MEMORY_OBJECT_HITS "Hits"
FROM v$shared_pool_advice
ORDER BY 1;
```
### Database Shared Pool Advice
```
SELECT lc_namespace "Library"
,LC_INUSE_MEMORY_OBJECTS "Objects"
,LC_INUSE_MEMORY_SIZE "Objects Mb"
,LC_FREEABLE_MEMORY_OBJECTS "Freeable Objects"
,LC_FREEABLE_MEMORY_SIZE "Freeable Mb"
FROM v$library_cache_memory;
```
## Troubleshooting Queries

### Sessions Exceeding Memory Thresholds

Identify sessions that may be causing memory pressure:

```sql
SELECT 
    s.sid,
    s.username,
    s.program,
    s.machine,
    s.status,
    s.sql_id,
    ROUND(pga_current.value/1024/1024, 2) AS current_pga_mb,
    ROUND(pga_max.value/1024/1024, 2) AS max_pga_mb,
    ROUND(uga_current.value/1024/1024, 2) AS current_uga_mb,
    s.last_call_et AS idle_seconds,
    CASE 
        WHEN pga_current.value > 500*1024*1024 THEN 'HIGH'
        WHEN pga_current.value > 100*1024*1024 THEN 'MEDIUM'
        ELSE 'NORMAL'
    END AS memory_level
FROM 
    v$session s,
    v$sesstat pga_current,
    v$sesstat pga_max,
    v$sesstat uga_current,
    v$statname pga_current_name,
    v$statname pga_max_name,
    v$statname uga_current_name
WHERE 
    s.sid = pga_current.sid
    AND s.sid = pga_max.sid
    AND s.sid = uga_current.sid
    AND pga_current.statistic# = pga_current_name.statistic#
    AND pga_max.statistic# = pga_max_name.statistic#
    AND uga_current.statistic# = uga_current_name.statistic#
    AND pga_current_name.name = 'session pga memory'
    AND pga_max_name.name = 'session pga memory max'
    AND uga_current_name.name = 'session uga memory'
    AND s.username IS NOT NULL
    AND pga_current.value > 50*1024*1024 -- Sessions using > 50MB
ORDER BY current_pga_mb DESC;
```

### Memory Growth Analysis

Track memory growth over time for active sessions:

```sql
SELECT 
    s.sid,
    s.username,
    s.program,
    ROUND(pga_current.value/1024/1024, 2) AS current_pga_mb,
    ROUND(pga_max.value/1024/1024, 2) AS max_pga_mb,
    ROUND(((pga_current.value/pga_max.value) * 100), 2) AS memory_utilization_pct,
    s.logon_time,
    ROUND((SYSDATE - s.logon_time) * 24, 2) AS hours_connected,
    s.status,
    s.sql_id
FROM 
    v$session s,
    v$sesstat pga_current,
    v$sesstat pga_max,
    v$statname pga_current_name,
    v$statname pga_max_name
WHERE 
    s.sid = pga_current.sid
    AND s.sid = pga_max.sid
    AND pga_current.statistic# = pga_current_name.statistic#
    AND pga_max.statistic# = pga_max_name.statistic#
    AND pga_current_name.name = 'session pga memory'
    AND pga_max_name.name = 'session pga memory max'
    AND s.username IS NOT NULL
    AND pga_max.value > pga_current.value * 1.5 -- Sessions where max is 50% higher than current
ORDER BY memory_utilization_pct DESC;
```

### Dead/Idle Session Cleanup Candidates

Identify idle sessions consuming memory that could be terminated:

```sql
SELECT 
    s.sid,
    s.serial#,
    s.username,
    s.program,
    s.machine,
    s.status,
    ROUND(pga.value/1024/1024, 2) AS pga_mb,
    ROUND(s.last_call_et/3600, 2) AS idle_hours,
    s.logon_time,
    'ALTER SYSTEM KILL SESSION ''' || s.sid || ',' || s.serial# || ''';' AS kill_command
FROM 
    v$session s,
    v$sesstat pga,
    v$statname pga_name
WHERE 
    s.sid = pga.sid
    AND pga.statistic# = pga_name.statistic#
    AND pga_name.name = 'session pga memory'
    AND s.username IS NOT NULL
    AND s.status = 'INACTIVE'
    AND s.last_call_et > 3600 -- Idle for more than 1 hour
    AND pga.value > 10*1024*1024 -- Using more than 10MB
ORDER BY pga_mb DESC, idle_hours DESC;
```

## Best Practices and Recommendations

### Monitoring Guidelines

1. **Regular Monitoring**:
   - Run memory overview queries daily during peak hours
   - Set up alerts for sessions exceeding memory thresholds
   - Monitor memory growth trends weekly

2. **Threshold Settings**:
   - **Normal**: < 50MB per session
   - **Warning**: 50-200MB per session
   - **Critical**: > 200MB per session

3. **Investigation Priorities**:
   - Sessions with high PGA usage and long-running SQL
   - Users with multiple high-memory sessions
   - Processes showing continuous memory growth

### Performance Optimization Tips

- **PGA Management**: Configure `PGA_AGGREGATE_TARGET` appropriately
- **Sort Area**: Monitor sort operations in temporary tablespace
- **Connection Pooling**: Implement to reduce overall session memory
- **Query Optimization**: Identify and tune memory-intensive SQL statements

### Troubleshooting Steps

1. **High Memory Usage**:
   - Identify top consumers using session memory queries
   - Check for long-running or inefficient SQL statements
   - Review application connection patterns

2. **Memory Leaks**:
   - Compare current vs. maximum memory usage
   - Look for sessions with continuously growing memory
   - Check for uncommitted transactions holding resources

3. **System Performance Issues**:
   - Use memory distribution queries to identify bottlenecks
   - Check for sessions exceeding configured limits
   - Consider terminating idle sessions consuming significant memory
  
### Memory Issues

1. Check PGA and SGA allocation
2. Review memory advisories
3. Identify sessions with high memory usage
4. Check for memory-intensive operations (sorts, hash joins)
5. Adjust memory parameters if needed

### Security and Maintenance

- **Access Control**: Ensure only authorized users can view memory statistics
- **Regular Cleanup**: Implement procedures to identify and terminate unused sessions
- **Documentation**: Maintain records of memory usage patterns and optimization efforts
- **Automation**: Consider automated monitoring and alerting for memory thresholds
