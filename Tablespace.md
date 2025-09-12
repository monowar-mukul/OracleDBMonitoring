# Oracle Tablespace Management Scripts

A comprehensive collection of SQL scripts for Oracle database tablespace management, monitoring, and optimization.

## Table of Contents
- [Basic Tablespace Information](#basic-tablespace-information)
- [Tablespace Space Monitoring](#tablespace-space-monitoring)
- [Datafile Management](#datafile-management)
- [Segment Analysis](#segment-analysis)
- [Extent Management](#extent-management)
- [Tablespace Fragmentation](#tablespace-fragmentation)
- [Table Defragmentation](#table-defragmentation)
- [High Water Mark Analysis](#high-water-mark-analysis)
- [Tablespace Creation and Migration](#tablespace-creation-and-migration)
- [Troubleshooting](#troubleshooting)

## Basic Tablespace Information

### Default Tablespace Settings
```sql
SELECT * FROM database_properties 
WHERE property_name LIKE 'DEFAULT%TABLESPACE';
```

### Non-System Objects in SYSTEM Tablespace
```sql
SELECT OWNER, SEGMENT_NAME, SEGMENT_TYPE, TABLESPACE_NAME, BYTES
FROM dba_segments
WHERE TABLESPACE_NAME = 'SYSTEM'
AND OWNER NOT IN ('SYS','SYSTEM')
ORDER BY OWNER, SEGMENT_NAME;
```

### Check Autoextend Status
```sql
SELECT SUBSTR(file_name,1,50), AUTOEXTENSIBLE 
FROM dba_data_files;
```

## Tablespace Space Monitoring

### Comprehensive Tablespace Space Report
```sql
SET LINESIZE 150
COLUMN tablespace_name FORMAT a20 HEADING 'Tablespace'
COLUMN sumb FORMAT 999,999,999
COLUMN extents FORMAT 9999
COLUMN bytes FORMAT 999,999,999,999
COLUMN largest FORMAT 999,999,999,999
COLUMN Tot_Size FORMAT 999,999 HEADING 'Total| Size(Mb)'
COLUMN Tot_Free FORMAT 999,999,999 HEADING 'Total Free(MB)'
COLUMN Pct_Free FORMAT 999.99 HEADING '% Free'
COLUMN Chunks_Free FORMAT 9999 HEADING 'No Of Ext.'
COLUMN Max_Free FORMAT 999,999,999 HEADING 'Max Free(Kb)'

PROMPT FREE SPACE AVAILABLE IN TABLESPACES
SELECT a.tablespace_name,
       SUM(a.tots/1048576) Tot_Size,
       SUM(a.sumb/1048576) Tot_Free,
       SUM(a.sumb)*100/SUM(a.tots) Pct_Free,
       SUM(a.largest/1024) Max_Free,
       SUM(a.chunks) Chunks_Free
FROM (
    SELECT tablespace_name, 0 tots, SUM(bytes) sumb,
           MAX(bytes) largest, COUNT(*) chunks
    FROM dba_free_space 
    GROUP BY tablespace_name
    UNION
    SELECT tablespace_name, SUM(bytes) tots, 0, 0, 0 
    FROM dba_data_files
    GROUP BY tablespace_name
) a
GROUP BY a.tablespace_name
ORDER BY pct_free;
```

### Tablespace Usage with Alert (< 10% Free)
```sql
SELECT F.TABLESPACE_NAME,
       TO_CHAR((T.TOTAL_SPACE - F.FREE_SPACE),'999,999') "USED (MB)",
       TO_CHAR(F.FREE_SPACE, '999,999') "FREE (MB)",
       TO_CHAR(T.TOTAL_SPACE, '999,999') "TOTAL (MB)",
       TO_CHAR((ROUND((F.FREE_SPACE/T.TOTAL_SPACE)*100)),'999')||' %' PER_FREE
FROM (
    SELECT TABLESPACE_NAME, 
           ROUND(SUM(BLOCKS*(SELECT VALUE/1024 FROM V$PARAMETER 
                            WHERE NAME = 'db_block_size')/1024)) FREE_SPACE
    FROM DBA_FREE_SPACE
    GROUP BY TABLESPACE_NAME
) F,
(
    SELECT TABLESPACE_NAME,
           ROUND(SUM(BYTES/1048576)) TOTAL_SPACE
    FROM DBA_DATA_FILES
    GROUP BY TABLESPACE_NAME
) T
WHERE F.TABLESPACE_NAME = T.TABLESPACE_NAME
AND (ROUND((F.FREE_SPACE/T.TOTAL_SPACE)*100)) < 10;
```

### Tablespace with Autoextend Information
```sql
SELECT /* + RULE */ df.tablespace_name "Tablespace",
       df.bytes / (1024 * 1024) "Size (MB)",
       SUM(fs.bytes) / (1024 * 1024) "Free (MB)",
       NVL(ROUND(SUM(fs.bytes) * 100 / df.bytes),1) "% Free",
       ROUND((df.bytes - SUM(fs.bytes)) * 100 / df.bytes) "% Used"
FROM dba_free_space fs,
     (SELECT tablespace_name, SUM(bytes) bytes
      FROM dba_data_files
      GROUP BY tablespace_name) df
WHERE fs.tablespace_name (+) = df.tablespace_name
GROUP BY df.tablespace_name, df.bytes
ORDER BY 4;
```

## Datafile Management

### Datafile Size by Tablespace
```sql
SELECT TABLESPACE_NAME, FILE_NAME, SUM(bytes)/1024/1024 AS SIZE_MB
FROM dba_data_files
WHERE TABLESPACE_NAME = '&TBSPACE'
GROUP BY TABLESPACE_NAME, FILE_NAME;
```

### Find Datafile to Resize
```sql
SELECT file_name, SUM(bytes)/1024/1024 AS SIZE_MB
FROM dba_data_files
WHERE TABLESPACE_NAME = '&TSpace'
GROUP BY file_name;
```

### Resize Datafile
```sql
ALTER DATABASE DATAFILE '&dfile' RESIZE &size;
```

### Datafiles Near Maximum Size (90% of maxbytes)
```sql
SELECT file_name, 
       bytes/1024/1024 current_size_MB, 
       increment_by*8192/1024/1024 autoextend_size_MB, 
       maxbytes/1024/1024 max_size_MB
FROM dba_data_files
WHERE (bytes > (maxbytes * 0.90) AND autoextensible = 'YES')
   OR (bytes/1024/1024 + increment_by*8192/1024/1024 > maxbytes/1024/1024*0.9 
       AND autoextensible = 'YES');
```

## Segment Analysis

### Large Segments Report
```sql
COLUMN Object FORMAT a24;
COLUMN Type FORMAT a5;
SELECT segment_name "Object", 
       segment_type "Type",
       ROUND(bytes/1024/1024) "Mb", 
       ROUND(bytes/1024) "Kb",
       bytes "Bytes", 
       blocks "Blocks"
FROM dba_segments 
WHERE owner = '&OWNER'
AND segment_type IN ('TABLE','INDEX')
ORDER BY segment_name, segment_type DESC;
```

### Segments by Owner Size Summary
```sql
SELECT s.owner, SUM(s.BYTES) / (1024 * 1024 * 1024) SIZE_IN_GB
FROM dba_segments s
GROUP BY s.owner;
```

### Top 10 Largest Segments in Tablespace
```sql
COLUMN owner FORMAT a15
COLUMN segment_name FORMAT a35

SELECT * FROM (
    SELECT owner, segment_type, segment_name, bytes/1024 kbytes
    FROM dba_segments
    WHERE tablespace_name = '&ts_name'
    ORDER BY bytes/1024 DESC
)
WHERE rownum < 10;
```

## Extent Management

### Segments with High Extent Count (>20)
```sql
SELECT owner, segment_name, extents, bytes, max_extents, next_extent
FROM dba_segments
WHERE segment_type IN ('TABLE','INDEX') 
AND extents > 20
ORDER BY owner, segment_name;
```

### Segments Near Maximum Extents
```sql
SELECT owner "Owner",
       segment_name "Segment Name",
       segment_type "Type",
       tablespace_name "Tablespace",
       extents "Ext",
       max_extents "Max"
FROM dba_segments
WHERE ((max_extents - extents) <= 3) 
AND owner NOT IN ('SYS','SYSTEM')
ORDER BY owner, segment_name;
```

### Next Extent Cannot Fit
```sql
SELECT a.owner, a.segment_name, b.tablespace_name,
       DECODE(ext.extents,1,b.next_extent,
              a.bytes*(1+b.pct_increase/100)) nextext,
       freesp.largest
FROM dba_extents a,
     dba_segments b,
     (SELECT owner, segment_name, MAX(extent_id) extent_id,
             COUNT(*) extents
      FROM dba_extents
      GROUP BY owner, segment_name) ext,
     (SELECT tablespace_name, MAX(bytes) largest
      FROM dba_free_space
      GROUP BY tablespace_name) freesp
WHERE a.owner = b.owner 
AND a.segment_name = b.segment_name 
AND a.owner = ext.owner 
AND a.segment_name = ext.segment_name 
AND a.extent_id = ext.extent_id 
AND b.tablespace_name = freesp.tablespace_name 
AND DECODE(ext.extents,1,b.next_extent,
           a.bytes*(1+b.pct_increase/100)) > freesp.largest;
```

## Tablespace Fragmentation

### Fragmentation Analysis
```sql
SET PAGES 50000 LINES 32767
SELECT tablespace_name,
       COUNT(*) free_chunks,
       DECODE(ROUND((MAX(bytes) / 1024000),2),NULL,0,
              ROUND((MAX(bytes) / 1024000),2)) largest_chunk,
       NVL(ROUND(SQRT(MAX(blocks)/SUM(blocks))*(100/SQRT(SQRT(COUNT(blocks)))),2),0) fragmentation_index
FROM sys.dba_free_space 
GROUP BY tablespace_name 
ORDER BY 2 DESC, 1;
```

### Tablespace Block Map
```sql
SET LINES 132
SET PAGES 60
COLUMN file_id FORMAT 999
COLUMN block_id FORMAT 99999999
COLUMN end_block FORMAT 99999999
COLUMN owner FORMAT a15
COLUMN segment_name FORMAT a25
COLUMN partition_name FORMAT a25
COLUMN segment_type FORMAT a12

SELECT file_id, block_id, block_id + blocks - 1 end_block,
       owner, segment_name, partition_name, segment_type
FROM dba_extents
WHERE tablespace_name = '&&tablespace'
UNION ALL
SELECT file_id, block_id, block_id + blocks - 1 end_block,
       'free' owner, 'free' segment_name, 
       NULL partition_name, NULL segment_type
FROM dba_free_space
WHERE tablespace_name = '&&tablespace'
ORDER BY 1,2;
```

## Table Defragmentation

### Check Table Fragmentation
```sql
-- Step 1: Gather table stats
EXEC dbms_stats.gather_table_stats('&schema_name','&table_name');

-- Step 2: Check fragmentation percentage
SET PAGES 50000 LINES 32767
SELECT owner,
       table_name,
       ROUND((blocks*8),2)||'kb' "Fragmented size",
       ROUND((num_rows*avg_row_len/1024),2)||'kb' "Actual size",
       ROUND((blocks*8),2)-ROUND((num_rows*avg_row_len/1024),2)||'kb' "Waste",
       ((ROUND((blocks*8),2)-ROUND((num_rows*avg_row_len/1024),2))/ROUND((blocks*8),2))*100 - 10 "reclaimable space %"
FROM dba_tables 
WHERE table_name = '&table_Name' 
AND OWNER LIKE '&schema_name';
```

### Defragmentation Methods

#### Method 1: ALTER TABLE MOVE
```sql
-- Record index status
SELECT index_name, status FROM dba_indexes WHERE table_name LIKE '&table_name';

-- Move table (same or different tablespace)
ALTER TABLE &table_name MOVE;
-- OR
ALTER TABLE &table_name ENABLE ROW MOVEMENT;
ALTER TABLE &table_name MOVE TABLESPACE &new_tablespace_name;

-- Rebuild indexes
ALTER INDEX &INDEX_NAME REBUILD ONLINE;

-- Gather stats
EXEC dbms_stats.gather_table_stats('&owner_name','&table_name');
```

#### Method 2: SHRINK SPACE (Oracle 10g+)
```sql
-- Enable row movement
ALTER TABLE &table_name ENABLE ROW MOVEMENT;

-- Shrink in two steps
ALTER TABLE &table_name SHRINK SPACE COMPACT;  -- Rearrange rows
ALTER TABLE &table_name SHRINK SPACE;          -- Reset HWM

-- OR single step
ALTER TABLE &table_name SHRINK SPACE;
```

## High Water Mark Analysis

### Find High Water Mark for Datafiles
```sql
SET VERIFY OFF
COLUMN file_name FORMAT a50 WORD_WRAPPED
COLUMN smallest FORMAT 999,990 HEADING "Smallest|Size|Poss."
COLUMN currsize FORMAT 999,990 HEADING "Current|Size"
COLUMN savings FORMAT 999,990 HEADING "Poss.|Savings"
BREAK ON REPORT
COMPUTE SUM OF savings ON REPORT

SELECT file_name,
       CEIL((NVL(hwm,1)*8192)/1024/1024) smallest,
       CEIL(blocks*8192/1024/1024) currsize,
       CEIL(blocks*8192/1024/1024) - CEIL((NVL(hwm,1)*8192)/1024/1024) savings
FROM dba_data_files a,
     (SELECT file_id, MAX(block_id+blocks-1) hwm
      FROM dba_extents
      GROUP BY file_id) b
WHERE a.file_id = b.file_id(+)
ORDER BY savings DESC;
```

### Generate Datafile Resize Scripts
```sql
SELECT 'ALTER DATABASE DATAFILE ''' || FILE_NAME || ''' RESIZE ' || 
       CEIL((NVL(HWM,1)*8192)/1024/1024) || 'M;' SHRINK_DATAFILES,
       ROUND(bytes/1024/1024,2) curr_size_mb,
       ROUND(bytes/1024/1024,0) - CEIL((NVL(HWM,1)*8192)/1024/1024) saving_mb 
FROM dba_data_files dbadf,
     (SELECT file_id, MAX(block_id+blocks-1) hwm 
      FROM dba_extents 
      GROUP BY file_id) dbafs
WHERE dbadf.file_id = dbafs.file_id(+) 
AND CEIL(blocks*8192/1024/1024) - CEIL((NVL(hwm,1)*8192)/1024/1024) > 0;
```

## Tablespace Creation and Migration

### Generate CREATE TABLESPACE Scripts
```sql
SET PAGES 0;
SET HEAD OFF;
CREATE TABLE ts_gen_scripts(line INT, text VARCHAR2(2000));
TRUNCATE TABLE ts_gen_scripts;

DECLARE
    CURSOR cur_ts IS 
        SELECT * FROM dba_Tablespaces WHERE tablespace_name != 'SYSTEM';
    CURSOR cur_ts_files (ts_name VARCHAR2) IS 
        SELECT * FROM dba_data_files WHERE tablespace_name = ts_name 
        ORDER BY tablespace_name, file_id;
    
    cur_ts_files_rec cur_ts_files%ROWTYPE;
    create_ts_string VARCHAR2(2000);
    counter INT := 0; 
    count_files INT := 0;
BEGIN
    FOR cur_ts_rec IN cur_ts LOOP
        create_ts_string := 'CREATE TABLESPACE ' || cur_ts_rec.tablespace_name || CHR(10) || 'DATAFILE ';
        
        OPEN cur_ts_files(cur_ts_rec.tablespace_name);
        LOOP
            FETCH cur_ts_files INTO cur_ts_files_rec;  
            EXIT WHEN cur_ts_files%NOTFOUND; 
            
            IF count_files >= 1 THEN
                create_ts_string := create_ts_string || CHR(10) || '         ''' || 
                    cur_ts_files_rec.file_name || '''' || ' SIZE ' || 
                    cur_ts_files_rec.bytes/1024 || ' K ' || ' , ';
            ELSE 
                create_ts_string := create_ts_string || '''' || cur_ts_files_rec.file_name || 
                    '''' || ' SIZE ' || cur_ts_files_rec.bytes/1024 || ' K ' || ' , ';
            END IF;
            count_files := count_files + 1;
        END LOOP;
        count_files := 0;
        CLOSE cur_ts_files;
        
        Create_Ts_String := RTRIM(Create_Ts_String, ' , ');
        
        -- Add extent management parameters
        IF cur_ts_rec.extent_management = 'LOCAL' THEN
            IF cur_ts_rec.Allocation_Type = 'UNIFORM' THEN
                Create_Ts_String := Create_Ts_String || CHR(10) || 
                    'EXTENT MANAGEMENT LOCAL UNIFORM SIZE ' || cur_ts_rec.Initial_Extent || ' ;';
            ELSE
                Create_Ts_String := Create_Ts_String || CHR(10) || 
                    'EXTENT MANAGEMENT LOCAL AUTOALLOCATE ;';
            END IF;
        ELSE
            Create_Ts_String := Create_Ts_String || CHR(10) || ' DEFAULT STORAGE ( ' || CHR(10) ||
                ' INITIAL       ' || cur_ts_rec.Initial_Extent || CHR(10) ||
                ' NEXT          ' || cur_ts_rec.Next_Extent || CHR(10) ||
                ' MINEXTENTS    ' || cur_ts_rec.Min_Extents || CHR(10) ||
                ' PCTINCREASE   ' || cur_ts_rec.Pct_Increase || ' ) ;';
        END IF;
        
        IF cur_ts_rec.contents = 'TEMPORARY' THEN
            Create_Ts_String := RTRIM(Create_Ts_String, ';');
            Create_Ts_String := Create_Ts_String || ' TEMPORARY ; ';
        END IF;
        
        counter := counter + 1;
        INSERT INTO ts_gen_scripts VALUES(counter, Create_Ts_String);
    END LOOP;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        RAISE_APPLICATION_ERROR(-20999, 'Something Wrong..' || SQLERRM);
END;
/
```

### Bigfile Tablespace Creation for Migration
```sql
SET PAUSE OFF
SET PAGESIZE 9999
SET LINESIZE 132
SET FEEDBACK OFF
SET ECHO OFF
SET VERIFY OFF

COLUMN tsname FORMAT a30 HEAD "Tablespace Name"

SELECT 'CREATE BIGFILE TABLESPACE ' || ts.tablespace_name ||
       ' DATAFILE SIZE ' ||
       CEIL(DECODE(tf.tf_file_type,
                   'Temp', tf.tf_total - NVL(fs.free,0),
                   df.df_total - NVL(fs.free,0)) / 1048576) || 'M ' ||
       ' AUTOEXTEND ON NEXT ' ||
       CEIL(DECODE(tf.tf_file_type,
                   'Temp', tf.tf_total - NVL(fs.free,0),
                   df.df_total - NVL(fs.free,0)) * .10 / 1048576) || 'M;' AS "Tablespace Create Statements"
FROM (SELECT tablespace_name, 'Data' df_file_type, SUM(bytes) df_total
      FROM dba_data_files@sv_migrate
      GROUP BY tablespace_name) df,
     (SELECT tablespace_name, 'Temp' tf_file_type, SUM(bytes) tf_total
      FROM dba_temp_files@sv_migrate
      GROUP BY tablespace_name) tf,
     (SELECT tablespace_name, SUM(bytes) free
      FROM dba_free_space@sv_migrate
      GROUP BY tablespace_name) fs,
     (SELECT tablespace_name
      FROM dba_tablespaces@sv_migrate
      MINUS
      SELECT tablespace_name
      FROM dba_tablespaces) ts
WHERE ts.tablespace_name = df.tablespace_name(+)
AND ts.tablespace_name = tf.tablespace_name(+)
AND ts.tablespace_name = fs.tablespace_name(+);
```

## Troubleshooting

### Block Size Issues
```sql
-- Check current block size
SELECT * FROM V$PARAMETER WHERE NAME = 'db_block_size';

-- Set cache for different block sizes
ALTER SYSTEM SET DB_16K_CACHE_SIZE=64M SCOPE=BOTH;
ALTER SYSTEM SET db_32k_cache_size=64M SCOPE=BOTH;

-- Verify cache settings
SHOW PARAMETER cache_size;
```

### Tablespace Quota Management
```sql
SET LINES 100 NUMWIDTH 12
SELECT q.username, q.tablespace_name, q.bytes, q.max_bytes   
FROM dba_ts_quotas q, dba_users u  
WHERE q.username = u.username 
AND q.username IN ('&USER_NAME');
```

### Move Datafiles to ASM
```sql
-- Using RMAN
RMAN> SQL "ALTER TABLESPACE &TABLESPACE_NAME OFFLINE";
RMAN> COPY DATAFILE '&OLD_PATH' TO '+&DISKGROUP';
-- Then rename in SQL*Plus
SQL> ALTER DATABASE RENAME FILE '&OLD_PATH' TO '&NEW_ASM_PATH';
SQL> ALTER TABLESPACE &TABLESPACE_NAME ONLINE;
```

### Shrink Temporary Tablespace
```sql
ALTER TABLESPACE TEMP SHRINK SPACE;
```

## Utility Scripts

### Table Comparison (Column Differences)
```sql
SELECT 'TABLE_A has these columns that are not in TABLE_B', DIFF.*
FROM (
    SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
    FROM all_tab_columns
    WHERE table_name = 'TABLE_A'
    MINUS
    SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
    FROM all_tab_columns
    WHERE table_name = 'TABLE_B'
) DIFF
UNION
SELECT 'TABLE_B has these columns that are not in TABLE_A', DIFF.*
FROM (
    SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
    FROM all_tab_columns
    WHERE table_name = 'TABLE_B'
    MINUS
    SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
    FROM all_tab_columns
    WHERE table_name = 'TABLE_A'
) DIFF;
```

### Row Count Comparison
```sql
SELECT COUNT(*) FROM &TABLE_NAME
MINUS 
SELECT COUNT(*) FROM &TABLE_NAME@&DATABASE_LINK;
```

## Notes

- Always test scripts in a development environment first
- Ensure proper backups before performing any structural changes
- Monitor system performance during defragmentation operations
- Consider maintenance windows for operations that may impact availability
- Use `ONLINE` options where available to minimize downtime

## Version Compatibility

These scripts are compatible with Oracle 10g and later versions. Some features like `SHRINK SPACE` are only available from Oracle 10g onwards.
