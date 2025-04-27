# Oracle Database Size and Space Monitoring

<details>
<summary><strong>Table of Contents</strong></summary>

- [Host Configuration](#host-configuration)
- [Container Database (CDB) Size](#container-database-cdb-size)
  - [Full CDB Size](#1-full-cdb-size-datafiles--tempfiles--redo-logs--control-files)
  - [CDB Size with PDB Breakdown](#2-cdb-size-with-pdb-breakdown-datafiles--tempfiles)
  - [PDB Sizes Only](#3-pdb-sizes-only)
- [Non-CDB Database Size (Used vs Free Space)](#non-cdb-database-size-used-vs-free-space)
- [Database Storage Breakdown](#database-storage-breakdown)
- [Schema and Segment Size Queries](#schema-and-segment-size-queries)
  - [Schema Size by Segment Type](#schema-size-by-segment-type)
  - [Single Segment Size](#single-segment-size-eg-table)
  - [Tablespaces Usage Status](#tablespaces-usage-status)
- [Flash Recovery Area (FRA) Usage](#flash-recovery-area-fra-usage)
- [Object Growth Tracking](#object-growth-tracking)
  - [Object Space Growth Over Time](#object-space-growth-over-time)
  - [Database Growth by Day](#database-growth-by-day)
- [Notes](#notes)

</details>

---

# Host Configuration

```bash
$ lscpu | egrep 'Model name|Socket|Thread|NUMA|CPU\(s\)'
```

---

# Container Database (CDB) Size

## 1. Full CDB Size (Datafiles + Tempfiles + Redo Logs + Control Files)

```sql
SELECT x.data_size + y.temp_size + z.redo_size + w.controlfile_size AS "CDB_SIZE_GB"
FROM ...;
```

## 2. CDB Size with PDB Breakdown (Datafiles + Tempfiles)

```sql
COLUMN con_id FORMAT 9999
SELECT con_id, pdb_name, SUM(bytes)/1024/1024/1024 AS size_gb
FROM ...;
```

## 3. PDB Sizes Only

```sql
COLUMN con_id FORMAT 9999
SELECT con_id, pdb_name, SUM(bytes)/1024/1024/1024 AS size_gb
FROM ...
WHERE container_name IS NOT NULL;
```

---

# Non-CDB Database Size (Used vs Free Space)

```sql
COL "Database Size" FORMAT A20
SELECT ... FROM ...;
```

---

# Database Storage Breakdown

```sql
SELECT 'Data Files' type, SUM(bytes)/1024/1024/1024 szgb, COUNT(*) cnt
FROM dba_data_files
UNION ALL
SELECT 'Temp Files', SUM(bytes)/1024/1024/1024, COUNT(*)
FROM dba_temp_files
UNION ALL
SELECT 'Redo Logs', SUM(bytes)/1024/1024/1024, COUNT(*)
FROM v$log;
```

---

# Schema and Segment Size Queries

## Schema Size by Segment Type

```sql
SET pagesize 10000
SELECT segment_type, SUM(bytes)/1024/1024 AS MB
FROM dba_segments
WHERE owner = UPPER('&owner')
GROUP BY segment_type
ORDER BY 2 DESC;
```

## Single Segment Size (e.g., Table)

```sql
SELECT SUM(bytes)/(1024*1024) "Table Size MB", segment_name
FROM dba_segments
WHERE segment_type='TABLE' AND segment_name=UPPER('&TABLE_NAME')
GROUP BY segment_name;
```

## Tablespaces Usage Status

```sql
SET linesize 300
SELECT tablespace_name,
       ROUND(used_space*8/1024,2) USED_MB,
       ROUND((tablespace_size-used_space)*8/1024,2) FREE_MB,
       ROUND(tablespace_size*8/1024,2) TOTAL_MB,
       ROUND((used_space/tablespace_size)*100,2) USED_PCT
FROM dba_tablespace_usage_metrics;
```

---

# Flash Recovery Area (FRA) Usage

```sql
SET linesize 500
SELECT name, SPACE_LIMIT/1024/1024/1024 SPACE_LIMIT_GB,
       SPACE_USED/1024/1024/1024 SPACE_USED_GB,
       SPACE_RECLAIMABLE/1024/1024/1024 SPACE_RECLAIMABLE_GB,
       NUMBER_OF_FILES
FROM v$recovery_file_dest;
```

---

# Object Growth Tracking

## Object Space Growth Over Time

```sql
SELECT *
FROM (
    SELECT TO_CHAR(end_interval_time, 'MM/DD/YY') mydate,
           ROUND((SUM(tsu.tablespace_usedsize)) * (SELECT value/1024 FROM v$parameter WHERE name='db_block_size')/1024/1024, 2) size_mb
    FROM dba_hist_tbspc_space_usage tsu
         JOIN dba_hist_snapshot snap ON tsu.snap_id = snap.snap_id
    GROUP BY TO_CHAR(end_interval_time, 'MM/DD/YY')
    ORDER BY 1 DESC
);
```

## Database Growth by Day

```sql
SELECT SUBSTR(b.end_interval_time,1,9) DAY,
       ROUND((a.value/1048576)) DB_SIZE_MB
FROM dba_hist_sysstat a, dba_hist_snapshot b
WHERE a.snap_id = b.snap_id
  AND a.stat_name = 'db size in bytes'
ORDER BY 1;
```

---

# Notes

- Replace `&owner`, `&TABLE_NAME`, `&segment_name`, and `&days_back` with actual values during execution.
- Ensure proper privileges to access `DBA_SEGMENTS`, `V$` views, and `DBA_HIST_*` views.

