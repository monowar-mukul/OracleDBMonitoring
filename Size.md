# Oracle Database Size and Space Monitoring

A comprehensive guide for monitoring Oracle database storage, including Container Database (CDB), Pluggable Database (PDB), and traditional non-CDB configurations.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Host Configuration](#host-configuration)
- [Container Database (CDB) Monitoring](#container-database-cdb-monitoring)
  - [Complete CDB Size](#complete-cdb-size)
  - [CDB with PDB Breakdown](#cdb-with-pdb-breakdown)
  - [PDB Sizes Only](#pdb-sizes-only)
- [Non-CDB Database Monitoring](#non-cdb-database-monitoring)
- [Storage Breakdown Analysis](#storage-breakdown-analysis)
- [Schema and Segment Analysis](#schema-and-segment-analysis)
  - [Schema Size by Segment Type](#schema-size-by-segment-type)
  - [Individual Segment Size](#individual-segment-size)
  - [Tablespace Usage](#tablespace-usage)
- [Recovery Area Management](#recovery-area-management)
- [Growth Tracking](#growth-tracking)
  - [Historical Space Growth](#historical-space-growth)
  - [Daily Database Growth](#daily-database-growth)

## Prerequisites

- Oracle Database with appropriate privileges
- Access to `DBA_SEGMENTS`, `V$` views, and `DBA_HIST_*` views
- Replace placeholder variables (`&owner`, `&TABLE_NAME`, etc.) with actual values during execution

## Host Configuration

Check system resources and CPU configuration:

```bash
lscpu | egrep 'Model name|Socket|Thread|NUMA|CPU\(s\)'
```

## Container Database (CDB) Monitoring

### Complete CDB Size

Get the total size including all database components:

```sql
-- Full CDB Size (Datafiles + Tempfiles + Redo Logs + Control Files)
SELECT 
    ROUND((x.data_size + y.temp_size + z.redo_size + w.controlfile_size)/1024/1024/1024, 2) AS cdb_total_size_gb
FROM 
    (SELECT SUM(bytes) data_size FROM dba_data_files) x,
    (SELECT SUM(bytes) temp_size FROM dba_temp_files) y,
    (SELECT SUM(bytes) redo_size FROM v$log) z,
    (SELECT SUM(block_size * file_size_blks) controlfile_size FROM v$controlfile) w;
```

### CDB with PDB Breakdown

View storage allocation across all PDBs:

```sql
-- CDB Size with PDB Breakdown (Datafiles + Tempfiles)
COLUMN con_id FORMAT 9999
COLUMN pdb_name FORMAT A20
COLUMN size_gb FORMAT 999,999.99

SELECT 
    c.con_id,
    c.name AS pdb_name,
    ROUND(NVL(d.data_size, 0)/1024/1024/1024, 2) AS datafiles_gb,
    ROUND(NVL(t.temp_size, 0)/1024/1024/1024, 2) AS tempfiles_gb,
    ROUND((NVL(d.data_size, 0) + NVL(t.temp_size, 0))/1024/1024/1024, 2) AS total_size_gb
FROM 
    v$containers c
    LEFT JOIN (SELECT con_id, SUM(bytes) data_size FROM cdb_data_files GROUP BY con_id) d ON c.con_id = d.con_id
    LEFT JOIN (SELECT con_id, SUM(bytes) temp_size FROM cdb_temp_files GROUP BY con_id) t ON c.con_id = t.con_id
WHERE c.open_mode = 'READ WRITE'
ORDER BY total_size_gb DESC;
```

### PDB Sizes Only

Focus on Pluggable Databases only:

```sql
-- PDB Sizes Only (excluding CDB$ROOT and PDB$SEED)
COLUMN con_id FORMAT 9999
COLUMN pdb_name FORMAT A20
COLUMN size_gb FORMAT 999,999.99

SELECT 
    c.con_id,
    c.name AS pdb_name,
    ROUND((NVL(d.data_size, 0) + NVL(t.temp_size, 0))/1024/1024/1024, 2) AS size_gb
FROM 
    v$containers c
    LEFT JOIN (SELECT con_id, SUM(bytes) data_size FROM cdb_data_files GROUP BY con_id) d ON c.con_id = d.con_id
    LEFT JOIN (SELECT con_id, SUM(bytes) temp_size FROM cdb_temp_files GROUP BY con_id) t ON c.con_id = t.con_id
WHERE c.name NOT IN ('CDB$ROOT', 'PDB$SEED')
    AND c.open_mode = 'READ WRITE'
ORDER BY size_gb DESC;
```

## Non-CDB Database Monitoring

Monitor traditional non-CDB database space usage:

```sql
-- Non-CDB Database Size (Used vs Free Space)
COLUMN "Database Size" FORMAT A15
COLUMN "Used Space" FORMAT A15
COLUMN "Free Space" FORMAT A15
COLUMN "% Used" FORMAT A10

SELECT 
    ROUND(SUM(used.bytes)/1024/1024/1024, 2) || ' GB' AS "Database Size",
    ROUND(SUM(used.bytes)/1024/1024/1024 - SUM(free.bytes)/1024/1024/1024, 2) || ' GB' AS "Used Space",
    ROUND(SUM(free.bytes)/1024/1024/1024, 2) || ' GB' AS "Free Space",
    ROUND((1 - SUM(free.bytes)/SUM(used.bytes)) * 100, 2) || '%' AS "% Used"
FROM 
    (SELECT tablespace_name, SUM(bytes) bytes FROM dba_data_files GROUP BY tablespace_name) used,
    (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space GROUP BY tablespace_name) free
WHERE used.tablespace_name = free.tablespace_name(+);
```

## Storage Breakdown Analysis

Analyze different types of database files:

```sql
-- Database Storage Breakdown
COLUMN type FORMAT A15
COLUMN size_gb FORMAT 999,999.99
COLUMN file_count FORMAT 9999

SELECT 'Data Files' AS type, 
       ROUND(SUM(bytes)/1024/1024/1024, 2) AS size_gb, 
       COUNT(*) AS file_count
FROM dba_data_files
UNION ALL
SELECT 'Temp Files', 
       ROUND(SUM(bytes)/1024/1024/1024, 2), 
       COUNT(*)
FROM dba_temp_files
UNION ALL
SELECT 'Redo Logs', 
       ROUND(SUM(bytes)/1024/1024/1024, 2), 
       COUNT(*)
FROM v$log
ORDER BY size_gb DESC;
```

## Schema and Segment Analysis

### Schema Size by Segment Type

Analyze storage usage by schema and segment type:

```sql
-- Schema Size by Segment Type
col owner format a15
select owner, segment_type, round(sum(bytes)/1024/1024,2) "Seg Size, Mb" from dba_segments where owner in ('DW','DI_EDW_COG','SSBI','SSBI_HR') group by owner, segment_type order by 1,3 desc;
```

```sql
-- Schema Size by Segment Type
SET pagesize 10000
COLUMN segment_type FORMAT A20
COLUMN size_mb FORMAT 999,999.99

SELECT 
    segment_type,
    ROUND(SUM(bytes)/1024/1024, 2) AS size_mb,
    COUNT(*) AS segment_count
FROM dba_segments
WHERE owner = UPPER('&owner')
GROUP BY segment_type
ORDER BY size_mb DESC;
```

### Individual Segment Size

Check the size of a specific table or segment:

```sql
-- Single Segment Size
COLUMN segment_name FORMAT A30
COLUMN size_mb FORMAT 999,999.99

SELECT 
    segment_name,
    segment_type,
    ROUND(SUM(bytes)/1024/1024, 2) AS size_mb
FROM dba_segments
WHERE segment_type = 'TABLE' 
    AND segment_name = UPPER('&TABLE_NAME')
GROUP BY segment_name, segment_type;
```

### Tablespace Usage

Monitor tablespace utilization:

```sql
-- Tablespace Usage Status
SET linesize 300
COLUMN tablespace_name FORMAT A25
COLUMN used_mb FORMAT 999,999.99
COLUMN free_mb FORMAT 999,999.99
COLUMN total_mb FORMAT 999,999.99
COLUMN used_pct FORMAT 999.99

SELECT 
    tablespace_name,
    ROUND(used_space * 8/1024, 2) AS used_mb,
    ROUND((tablespace_size - used_space) * 8/1024, 2) AS free_mb,
    ROUND(tablespace_size * 8/1024, 2) AS total_mb,
    ROUND((used_space/tablespace_size) * 100, 2) AS used_pct
FROM dba_tablespace_usage_metrics
ORDER BY used_pct DESC;
```

## Recovery Area Management

Monitor Flash Recovery Area (FRA) usage:

```sql
-- Flash Recovery Area (FRA) Usage
SET linesize 500
COLUMN name FORMAT A30
COLUMN space_limit_gb FORMAT 999,999.99
COLUMN space_used_gb FORMAT 999,999.99
COLUMN space_reclaimable_gb FORMAT 999,999.99
COLUMN number_of_files FORMAT 9999

SELECT 
    name,
    ROUND(space_limit/1024/1024/1024, 2) AS space_limit_gb,
    ROUND(space_used/1024/1024/1024, 2) AS space_used_gb,
    ROUND(space_reclaimable/1024/1024/1024, 2) AS space_reclaimable_gb,
    number_of_files
FROM v$recovery_file_dest;
```

## Growth Tracking

### Historical Space Growth

Track tablespace growth over time using AWR data:

```sql
-- Object Space Growth Over Time
COLUMN mydate FORMAT A10
COLUMN size_mb FORMAT 999,999.99

SELECT *
FROM (
    SELECT 
        TO_CHAR(snap.end_interval_time, 'MM/DD/YY') AS mydate,
        ROUND(SUM(tsu.tablespace_usedsize * db_block_size.value)/1024/1024, 2) AS size_mb
    FROM 
        dba_hist_tbspc_space_usage tsu
        JOIN dba_hist_snapshot snap ON tsu.snap_id = snap.snap_id
        CROSS JOIN (SELECT value FROM v$parameter WHERE name = 'db_block_size') db_block_size
    WHERE snap.end_interval_time >= SYSDATE - &days_back
    GROUP BY TO_CHAR(snap.end_interval_time, 'MM/DD/YY')
    ORDER BY TO_DATE(TO_CHAR(snap.end_interval_time, 'MM/DD/YY'), 'MM/DD/YY') DESC
);
```

### Daily Database Growth

Track overall database size changes:

```sql
-- Database Growth by Day
COLUMN day FORMAT A12
COLUMN db_size_mb FORMAT 999,999.99

SELECT 
    TO_CHAR(b.end_interval_time, 'YYYY-MM-DD') AS day,
    ROUND(a.value/1048576, 2) AS db_size_mb
FROM 
    dba_hist_sysstat a
    JOIN dba_hist_snapshot b ON a.snap_id = b.snap_id
WHERE 
    a.stat_name = 'db size in bytes'
    AND b.end_interval_time >= SYSDATE - &days_back
ORDER BY day DESC;
```

## Usage Notes

- **Parameter Substitution**: Replace variables with actual values:
  - `&owner`: Target schema name
  - `&TABLE_NAME`: Specific table name
  - `&days_back`: Number of days for historical queries
- **Privileges Required**: Ensure access to:
  - `DBA_SEGMENTS`
  - `V$` views (V$LOG, V$CONTAINERS, etc.)
  - `DBA_HIST_*` views for AWR data
- **Performance**: Historical queries may take time on large systems
- **CDB vs Non-CDB**: Use appropriate sections based on your database architecture

## Contributing

Feel free to submit issues and enhancement requests!
