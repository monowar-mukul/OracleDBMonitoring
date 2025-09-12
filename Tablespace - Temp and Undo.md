# Oracle UNDO and TEMP Management Guide

Comprehensive guide for monitoring and managing Oracle UNDO and TEMP tablespaces, including calculations, troubleshooting, and maintenance procedures.

## Table of Contents

- [UNDO Management](#undo-management)
  - [UNDO Retention Calculations](#undo-retention-calculations)
  - [UNDO Size Calculations](#undo-size-calculations)
  - [Active Transaction Monitoring](#active-transaction-monitoring)
  - [UNDO Usage Monitoring](#undo-usage-monitoring)
  - [UNDO Tablespace Recreation](#undo-tablespace-recreation)
- [TEMP Management](#temp-management)
  - [TEMP Usage Monitoring](#temp-usage-monitoring)
  - [TEMP Session Analysis](#temp-session-analysis)
  - [TEMP Tablespace Recreation](#temp-tablespace-recreation)
  - [Historical TEMP Usage](#historical-temp-usage)
- [Rollback Segment Management (Legacy)](#rollback-segment-management-legacy)
  - [Rollback Statistics](#rollback-statistics)
  - [Rollback Contention Analysis](#rollback-contention-analysis)
  - [Rollback Maintenance](#rollback-maintenance)

## UNDO Management

### UNDO Retention Calculations

Calculate optimal UNDO retention based on current activity:

```sql
-- Calculate UNDO_RETENTION for given UNDO Tablespace
SELECT 
    ROUND(d.undo_size/(1024*1024), 2) AS "ACTUAL UNDO SIZE [MByte]",
    SUBSTR(e.value, 1, 25) AS "UNDO RETENTION [Sec]",
    ROUND((d.undo_size / (TO_NUMBER(f.value) * g.undo_block_per_sec))) AS "OPTIMAL UNDO RETENTION [Sec]"
FROM 
    (SELECT SUM(a.bytes) undo_size
     FROM v$datafile a,
          v$tablespace b,
          dba_tablespaces c
     WHERE c.contents = 'UNDO'
       AND c.status = 'ONLINE'
       AND b.name = c.tablespace_name
       AND a.ts# = b.ts#) d,
    v$parameter e,
    v$parameter f,
    (SELECT MAX(undoblks/((end_time-begin_time)*3600*24)) undo_block_per_sec
     FROM v$undostat) g
WHERE e.name = 'undo_retention'
  AND f.name = 'db_block_size';
```

### UNDO Size Calculations

Calculate required UNDO size based on database activity:

```sql
-- Calculate Needed UNDO Size for given Database Activity
SELECT 
    ROUND(d.undo_size/(1024*1024), 2) AS "ACTUAL UNDO SIZE [MByte]",
    SUBSTR(e.value, 1, 25) AS "UNDO RETENTION [Sec]",
    ROUND((TO_NUMBER(e.value) * TO_NUMBER(f.value) * g.undo_block_per_sec) / (1024*1024), 2) AS "NEEDED UNDO SIZE [MByte]"
FROM 
    (SELECT SUM(a.bytes) undo_size
     FROM v$datafile a,
          v$tablespace b,
          dba_tablespaces c
     WHERE c.contents = 'UNDO'
       AND c.status = 'ONLINE'
       AND b.name = c.tablespace_name
       AND a.ts# = b.ts#) d,
    v$parameter e,
    v$parameter f,
    (SELECT MAX(undoblks/((end_time-begin_time)*3600*24)) undo_block_per_sec
     FROM v$undostat) g
WHERE e.name = 'undo_retention'
  AND f.name = 'db_block_size';
```

### Active Transaction Monitoring

Monitor active transactions and their UNDO usage:

```sql
-- Identify active transactions in UNDO segments
COLUMN o FORMAT A10 HEADING 'OS USER'
COLUMN u FORMAT A10 HEADING 'ORACLE USER'
COLUMN s FORMAT A15 HEADING 'SEGMENT'
COLUMN txt FORMAT A50 HEADING 'SQL TEXT'

SELECT 
    s.osuser AS o,
    s.username AS u,
    s.sid,
    r.segment_name AS s,
    SUBSTR(sa.sql_text, 1, 50) AS txt
FROM 
    v$session s,
    v$transaction t,
    dba_rollback_segs r,
    v$sqlarea sa
WHERE s.taddr = t.addr
  AND t.xidusn = r.segment_id(+)
  AND s.sql_address = sa.address(+)
  AND SUBSTR(sa.sql_text, 1, 50) IS NOT NULL
ORDER BY s.sid;
```

```sql
-- Active transactions with timing information
COLUMN name FORMAT A8 HEADING 'RBS NAME'
COLUMN username FORMAT A10 HEADING 'USERNAME'
COLUMN osuser FORMAT A10 HEADING 'OS USER'
COLUMN start_time FORMAT A17 HEADING 'START TIME'
COLUMN status FORMAT A12 HEADING 'STATUS'

SELECT 
    s.username,
    s.osuser,
    t.start_time,
    r.name,
    t.used_ublk AS "ROLLB BLKS",
    DECODE(t.space, 'YES', 'SPACE TX',
           DECODE(t.recursive, 'YES', 'RECURSIVE TX',
                  DECODE(t.noundo, 'YES', 'NO UNDO TX', t.status))) AS status
FROM 
    sys.v_$transaction t,
    sys.v_$rollname r,
    sys.v_$session s
WHERE t.xidusn = r.usn
  AND t.ses_addr = s.saddr;
```

### UNDO Usage Monitoring

Monitor UNDO space utilization:

```sql
-- UNDO tablespace extent status
SELECT 
    tablespace_name,
    status,
    ROUND(SUM(bytes)/1024/1024, 2) AS size_mb,
    COUNT(*) AS extent_count
FROM dba_undo_extents
GROUP BY tablespace_name, status
ORDER BY tablespace_name, status;
```

```sql
-- Current UNDO usage by session
SELECT 
    s.sid,
    s.username,
    ROUND(SUM(ss.value)/1024/1024, 2) AS undo_size_mb
FROM 
    v$sesstat ss
    JOIN v$session s ON s.sid = ss.sid
    JOIN v$statname stat ON stat.statistic# = ss.statistic#
WHERE stat.name = 'undo change vector size'
  AND s.type <> 'BACKGROUND'
  AND s.username IS NOT NULL
GROUP BY s.sid, s.username
HAVING SUM(ss.value) > 0
ORDER BY undo_size_mb DESC;
```

### UNDO Tablespace Recreation

Step-by-step procedure for recreating UNDO tablespace:

```sql
-- Step 1: Create new UNDO tablespace
CREATE UNDO TABLESPACE undotbs_new
DATAFILE '/path/to/undotbs_new01.dbf' SIZE 1500M
AUTOEXTEND ON NEXT 100M MAXSIZE 4G;

-- Step 2: Switch to new UNDO tablespace
ALTER SYSTEM SET undo_tablespace = undotbs_new;

-- Step 3: Wait for transactions to complete, then drop old tablespace
-- Check for active transactions first
SELECT COUNT(*) FROM dba_undo_extents WHERE status = 'ACTIVE' AND tablespace_name = 'UNDOTBS1';

-- If no active extents, drop old tablespace
DROP TABLESPACE undotbs1 INCLUDING CONTENTS AND DATAFILES;

-- Step 4: Recreate original tablespace with desired size
CREATE UNDO TABLESPACE undotbs1
DATAFILE '/path/to/undotbs01.dbf' SIZE 5500M
AUTOEXTEND ON NEXT 100M MAXSIZE 8G;

-- Step 5: Switch back to original tablespace
ALTER SYSTEM SET undo_tablespace = undotbs1;

-- Step 6: Drop temporary tablespace
DROP TABLESPACE undotbs_new INCLUDING CONTENTS AND DATAFILES;

-- Step 7: Set UNDO retention
ALTER SYSTEM SET undo_retention = 900;
```

## TEMP Management

### TEMP Usage Monitoring

Monitor temporary tablespace usage across instances:

```sql
-- Instance-wise TEMP usage (RAC and non-RAC)
COLUMN tablespace_name FORMAT A30 HEADING 'TABLESPACE'
COLUMN FreeSpaceGB FORMAT 999.999 HEADING 'FREE GB'
COLUMN UsedSpaceGB FORMAT 999.999 HEADING 'USED GB'
COLUMN TotalSpaceGB FORMAT 999.999 HEADING 'TOTAL GB'
COLUMN instance_name FORMAT A10 HEADING 'INSTANCE'
COLUMN host_name FORMAT A30 HEADING 'HOST NAME'

SELECT 
    ss.tablespace_name,
    ROUND((ss.free_blocks * 8)/1024/1024, 3) AS FreeSpaceGB,
    ROUND((ss.used_blocks * 8)/1024/1024, 3) AS UsedSpaceGB,
    ROUND((ss.total_blocks * 8)/1024/1024, 3) AS TotalSpaceGB,
    i.instance_name,
    i.host_name
FROM 
    gv$sort_segment ss,
    gv$instance i
WHERE ss.tablespace_name IN (
        SELECT tablespace_name 
        FROM dba_tablespaces 
        WHERE contents = 'TEMPORARY')
  AND i.inst_id = ss.inst_id;
```

```sql
-- TEMP tablespace detailed statistics
COLUMN current_users FORMAT 999 HEADING 'ACTIVE|USERS'
COLUMN extent_size FORMAT 99,999,999 HEADING 'EXTENT|SIZE'
COLUMN total_extents FORMAT 9,999 HEADING 'TOTAL|EXTENTS'
COLUMN used_extents FORMAT 9,999 HEADING 'USED|EXTENTS'
COLUMN free_extents FORMAT 9,999 HEADING 'FREE|EXTENTS'

SELECT
    tablespace_name,
    current_users,
    total_blocks,
    used_blocks,
    free_blocks,
    max_blocks,
    max_used_blocks,
    max_sort_blocks
FROM v$sort_segment
WHERE tablespace_name = 'TEMP'
ORDER BY tablespace_name;
```

### TEMP Session Analysis

Identify sessions using TEMP space:

```sql
-- Current TEMP usage by session
SELECT 
    s.sid,
    s.serial#,
    s.username,
    s.osuser,
    p.spid,
    s.module,
    p.program,
    ROUND(SUM(su.blocks) * ts.block_size/1024/1024, 2) AS mb_used,
    su.tablespace
FROM 
    v$sort_usage su,
    v$session s,
    dba_tablespaces ts,
    v$process p
WHERE su.session_addr = s.saddr
  AND su.tablespace = ts.tablespace_name
  AND s.paddr = p.addr
GROUP BY 
    s.sid, s.serial#, s.username, s.osuser, p.spid, s.module,
    p.program, ts.block_size, su.tablespace
ORDER BY mb_used DESC;
```

```sql
-- TEMP usage with SQL information
SELECT 
    SYSDATE AS time_stamp,
    vsu.username,
    vs.sid,
    vp.spid,
    vs.sql_id,
    SUBSTR(vst.sql_text, 1, 100) AS sql_text,
    vsu.tablespace,
    ROUND(sum_blocks * dt.block_size/1024/1024, 2) AS usage_mb
FROM 
    (SELECT 
         username, 
         sql_id, 
         tablespace, 
         session_addr,
         SUM(blocks) AS sum_blocks
     FROM v$sort_usage
     GROUP BY username, sql_id, tablespace, session_addr
     HAVING SUM(blocks) > 1000) vsu,
    v$sqltext vst,
    v$session vs,
    v$process vp,
    dba_tablespaces dt
WHERE vs.sql_id = vst.sql_id
  AND vsu.session_addr = vs.saddr
  AND vs.paddr = vp.addr
  AND vst.piece = 0
  AND dt.tablespace_name = vsu.tablespace
ORDER BY usage_mb DESC;
```

### TEMP Tablespace Recreation

Procedure for recreating TEMP tablespace:

```sql
-- Step 1: Check current default temporary tablespace
SELECT property_name, property_value 
FROM database_properties 
WHERE property_name = 'DEFAULT_TEMP_TABLESPACE';

-- Step 2: Create new temporary tablespace
CREATE TEMPORARY TABLESPACE temp_new
TEMPFILE '/path/to/temp_new01.dbf' SIZE 2G
AUTOEXTEND ON NEXT 100M MAXSIZE 4G
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- Step 3: Set as default temporary tablespace
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp_new;

-- Step 4: Check for active sessions using old TEMP
SELECT username, session_num, session_addr, tablespace
FROM v$sort_usage
WHERE tablespace = 'TEMP';

-- Step 5: Kill sessions if necessary (use with caution)
-- SELECT 'ALTER SYSTEM KILL SESSION ''' || sid ||','|| serial# || ''';'
-- FROM v$session WHERE saddr IN (select session_addr from v$sort_usage where tablespace='TEMP');

-- Step 6: Drop old temporary tablespace
DROP TABLESPACE temp INCLUDING CONTENTS AND DATAFILES;

-- Step 7: Recreate with desired size
CREATE TEMPORARY TABLESPACE temp
TEMPFILE '/path/to/temp01.dbf' SIZE 6G
AUTOEXTEND ON NEXT 100M MAXSIZE 8G
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- Step 8: Set back as default
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp;

-- Step 9: Drop temporary tablespace
DROP TABLESPACE temp_new INCLUDING CONTENTS AND DATAFILES;
```

### Historical TEMP Usage

Track TEMP usage over time using AWR:

```sql
-- TEMP space usage history from ASH
SELECT 
    sql_id,
    ROUND(MAX(temp_space_allocated)/(1024*1024*1024), 2) AS max_temp_gb
FROM dba_hist_active_sess_history
WHERE sample_time > SYSDATE - 2
  AND temp_space_allocated > (1*1024*1024*1024)  -- > 1GB
GROUP BY sql_id
ORDER BY max_temp_gb DESC;
```

```sql
-- Current sessions using significant TEMP (RAC)
SELECT 
    se.inst_id,
    se.sid,
    ROUND(SUM(tu.blocks) * 8192 / 1024 / 1024, 2) AS temp_mb
FROM 
    gv$tempseg_usage tu,
    gv$session se
WHERE tu.inst_id = se.inst_id
  AND tu.session_addr = se.saddr
  AND tu.session_num = se.serial#
GROUP BY se.inst_id, se.sid
HAVING SUM(tu.blocks) * 8192 > 100 * 1024 * 1024  -- > 100MB
ORDER BY temp_mb DESC;
```

## Rollback Segment Management (Legacy)

*Note: This section applies to Oracle versions using manual rollback segment management (pre-9i).*

### Rollback Statistics

```sql
-- Rollback segment statistics
COLUMN name FORMAT A15 HEADING 'RBS NAME'
COLUMN extends FORMAT 9999 HEADING 'EXTENDS'
COLUMN wraps FORMAT 9999 HEADING 'WRAPS'
COLUMN shrinks FORMAT 999 HEADING 'SHRINKS'

SELECT 
    r.name,
    ROUND(s.rssize/1048576, 2) AS size_mb,
    ROUND(s.hwmsize/1048576, 2) AS hwm_mb,
    s.extends,
    s.xacts AS transactions,
    s.wraps,
    s.shrinks,
    SUBSTR(s.status, 1, 7) AS status
FROM 
    v$rollstat s,
    v$rollname r
WHERE s.usn = r.usn
ORDER BY r.name;
```

### Rollback Contention Analysis

```sql
-- Rollback segment contention check
SELECT 
    'Total Requests' AS metric,
    SUM(value) AS value
FROM v$sysstat 
WHERE name IN ('db block gets', 'consistent gets')
UNION ALL
SELECT 
    'Undo Waits' AS metric,
    SUM(count) AS value
FROM v$waitstat
WHERE class IN ('system undo header', 'system undo block', 'undo header', 'undo block');

-- Rollback segment wait ratios
COLUMN name FORMAT A15 HEADING 'RBS NAME'
COLUMN ratio FORMAT 99.99999 HEADING 'WAIT RATIO'

SELECT 
    r.name,
    s.waits,
    s.gets,
    CASE 
        WHEN s.gets > 0 THEN ROUND(s.waits/s.gets, 5)
        ELSE 0 
    END AS ratio
FROM 
    v$rollstat s,
    v$rollname r
WHERE s.usn = r.usn
ORDER BY ratio DESC;
```

### Rollback Maintenance

```sql
-- Generate scripts for rollback segment management

-- Take rollback segments offline
SELECT 'ALTER ROLLBACK SEGMENT ' || segment_name || ' OFFLINE;'
FROM dba_rollback_segs
WHERE status = 'ONLINE'
  AND segment_name <> 'SYSTEM';

-- Drop offline rollback segments
SELECT 'DROP ROLLBACK SEGMENT ' || segment_name || ';'
FROM dba_rollback_segs
WHERE status = 'OFFLINE'
  AND segment_name <> 'SYSTEM';

-- Manually resize rollback segment
-- ALTER ROLLBACK SEGMENT rbs_name SHRINK TO 100M;

-- Bring rollback segment online
-- ALTER ROLLBACK SEGMENT rbs_name ONLINE;
```

## Best Practices

### UNDO Management
- Set `UNDO_RETENTION` based on longest-running query requirements
- Size UNDO tablespace for peak activity plus retention buffer
- Monitor for `ORA-01555` errors and adjust accordingly
- Use automatic undo management (AUM) in modern Oracle versions

### TEMP Management
- Size TEMP tablespace for largest sort operations
- Monitor sessions with high TEMP usage
- Consider multiple TEMP tablespaces for workload isolation
- Use locally managed tablespaces with uniform extents

### Monitoring
- Regular monitoring of space usage and growth trends
- Set up alerts for high utilization thresholds
- Review AWR reports for historical analysis
- Monitor wait events related to UNDO and TEMP

## Troubleshooting

### Common Issues
- **ORA-01555**: Snapshot too old - increase UNDO_RETENTION or UNDO tablespace size
- **ORA-01652**: Unable to extend temp segment - add space to TEMP tablespace
- **High UNDO waits**: Add more UNDO tablespace or check for long-running transactions
- **TEMP space exhaustion**: Identify queries causing excessive sorting

### Performance Tips
- Use TEMP tablespace groups for parallel operations
- Consider SSD storage for high-activity UNDO/TEMP tablespaces
- Monitor and tune queries that generate excessive UNDO/TEMP usage
- Implement proper indexing to reduce sort operations
