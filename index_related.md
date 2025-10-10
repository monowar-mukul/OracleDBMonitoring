# Oracle Index Management - Complete Guide

A comprehensive collection of SQL scripts for managing, analyzing, and optimizing indexes in Oracle databases.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Index Information Scripts](#index-information-scripts)
- [Index Recreation Scripts](#index-recreation-scripts)
- [Index Rebuild Analysis](#index-rebuild-analysis)
- [Usage Examples](#usage-examples)
- [Best Practices](#best-practices)

## Overview

Proper index management is crucial for database performance. This repository provides tools for:

- Querying index information and structure
- Generating index creation scripts for backup/migration
- Identifying indexes that need rebuilding
- Monitoring index health and fragmentation
- Analyzing index utilization patterns

## Prerequisites

- Oracle Database 10g or higher
- DBA privileges or appropriate SELECT privileges on:
  - `DBA_INDEXES`
  - `DBA_IND_COLUMNS`
  - `DBA_SEGMENTS`
  - `DBA_TABLES`
  - `DBA_TAB_COLS`
  - `DBA_OBJECTS`
  - `DBA_TABLESPACES`

## Index Information Scripts

### Check Indexes for a Specific Table

Query all indexes and their columns for a given table, showing uniqueness and structure.

```sql
COLUMN iio FORMAT A10;
COLUMN iin FORMAT A25;
COLUMN iic FORMAT A25;
COLUMN uniqueness FORMAT A10;
BREAK ON iio SKIP 1 ON iin SKIP 1;

SELECT c.index_owner iio,
       i.index_name iin,
       c.column_name iic,
       uniqueness
FROM all_indexes i,
     all_ind_columns c
WHERE i.table_name = UPPER('&table_name')
  AND i.table_name = c.table_name
  AND i.table_owner = c.table_owner
  AND i.index_name = c.index_name
  AND i.owner = c.index_owner
ORDER BY c.index_owner, i.index_name, c.column_position;
```

**Usage:**
```sql
SQL> @check_indexes.sql
Enter table name: EMPLOYEES
```

**Output columns:**
- `IIO` - Index Owner
- `IIN` - Index Name
- `IIC` - Index Column Name
- `UNIQUENESS` - UNIQUE, NON-UNIQUE, or BITMAP

## Index Recreation Scripts

### Generate CREATE INDEX Statements

This sophisticated script generates a complete SQL file containing CREATE INDEX statements for all non-system indexes in the database. Perfect for database migrations, disaster recovery, or documentation.

**Features:**
- Excludes SYS schema indexes
- Supports UNIQUE, BITMAP, and regular indexes
- Includes all storage parameters (INITIAL, NEXT, FREELISTS, etc.)
- Option to use current segment size instead of original INITIAL_EXTENT
- Automatically calculates optimal extent sizes
- Handles tablespace placement
- Generates ready-to-execute SQL script

**Configuration:**

```sql
-- Set to 'Y' to use current segment size for INITIAL_EXTENT
-- Set to 'N' to use original INITIAL_EXTENT value
DEF usesegs='Y'
```

**Full Script:**

```sql
SET VERIFY OFF
SET FEEDBACK OFF
SET ECHO OFF
SET PAGESIZE 0
SET TERMOUT ON

-- Configuration: Use current segment size (Y) or original extent size (N)
DEF usesegs='Y'

SELECT 'Creating index build script...' FROM dual;

CREATE TABLE indx_temp (
    lineno NUMBER,
    text VARCHAR2(80)
);

DECLARE
    CURSOR ind_cursor IS 
        SELECT uniqueness,
               UPPER(owner),
               UPPER(index_name),
               UPPER(table_owner),
               UPPER(table_name),
               ini_trans,
               max_trans,
               tablespace_name,
               initial_extent,
               next_extent,
               min_extents,
               max_extents,
               freelists,
               freelist_groups,
               pct_increase,
               pct_free,
               table_type
        FROM sys.dba_indexes
        WHERE owner != 'SYS'
        ORDER BY owner, index_name;

    CURSOR segments_cursor (s_own VARCHAR2, s_ind VARCHAR2) IS 
        SELECT bytes
        FROM sys.dba_segments
        WHERE segment_name = s_ind 
          AND owner = s_own 
          AND segment_type = 'INDEX';

    CURSOR col_cursor (c_own VARCHAR2, c_ind VARCHAR2) IS 
        SELECT UPPER(column_name), column_position
        FROM sys.dba_ind_columns
        WHERE index_name = c_ind 
          AND index_owner = c_own
        ORDER BY column_position;

    lv_uniqueness           sys.dba_indexes.uniqueness%TYPE;
    lv_owner                sys.dba_indexes.owner%TYPE;
    lv_index_name           sys.dba_indexes.index_name%TYPE;
    lv_towner               sys.dba_indexes.table_owner%TYPE;
    lv_table_name           sys.dba_indexes.table_name%TYPE;
    lv_ini_trans            sys.dba_indexes.ini_trans%TYPE;
    lv_max_trans            sys.dba_indexes.max_trans%TYPE;
    lv_tablespace_name      sys.dba_indexes.tablespace_name%TYPE;
    lv_initial_extent       sys.dba_indexes.initial_extent%TYPE;
    lv_next_extent          sys.dba_indexes.next_extent%TYPE;
    lv_min_extents          sys.dba_indexes.min_extents%TYPE;
    lv_max_extents          sys.dba_indexes.max_extents%TYPE;
    lv_freelists            sys.dba_indexes.freelists%TYPE;
    lv_freelist_groups      sys.dba_indexes.freelist_groups%TYPE;
    lv_pct_increase         sys.dba_indexes.pct_increase%TYPE;
    lv_pct_free             sys.dba_indexes.pct_free%TYPE;
    lv_table_type           sys.dba_indexes.table_type%TYPE;
    segment_bytes           sys.dba_segments.bytes%TYPE;
    lv_column_name          sys.dba_ind_columns.column_name%TYPE;
    lv_column_position      sys.dba_ind_columns.column_position%TYPE;
    lv_lineno               NUMBER := 0;
    initial_extent_size     VARCHAR2(16);
    next_extent_size        VARCHAR2(16);
    a_lin                   VARCHAR2(80);

    FUNCTION wri(x_lin IN VARCHAR2, x_str IN VARCHAR2, x_force IN NUMBER) 
    RETURN VARCHAR2 IS
    BEGIN
        IF LENGTH(x_lin) + LENGTH(x_str) > 80 THEN
            lv_lineno := lv_lineno + 1;
            INSERT INTO indx_temp VALUES (lv_lineno, x_lin);
            IF x_force = 0 THEN
                RETURN x_str;
            ELSE
                lv_lineno := lv_lineno + 1;
                INSERT INTO indx_temp VALUES (lv_lineno, x_str);
                RETURN '';
            END IF;
        ELSE
            IF x_force = 0 THEN
                RETURN x_lin || x_str;
            ELSE
                lv_lineno := lv_lineno + 1;
                INSERT INTO indx_temp VALUES (lv_lineno, x_lin || x_str);
                RETURN '';
            END IF;
        END IF;
    END wri;

BEGIN
    a_lin := '';
    OPEN ind_cursor;
    LOOP
        FETCH ind_cursor INTO
            lv_uniqueness, lv_owner, lv_index_name, lv_towner,
            lv_table_name, lv_ini_trans, lv_max_trans,
            lv_tablespace_name, lv_initial_extent, lv_next_extent,
            lv_min_extents, lv_max_extents, lv_freelists,
            lv_freelist_groups, lv_pct_increase, lv_pct_free,
            lv_table_type;
        EXIT WHEN ind_cursor%NOTFOUND;

        -- Use current segment size if configured
        IF '&&usesegs' = 'Y' THEN
            lv_max_extents := 2147483645;
            lv_pct_increase := 0;
            OPEN segments_cursor (lv_owner, lv_index_name);
            FETCH segments_cursor INTO segment_bytes;
            IF segments_cursor%FOUND THEN
                -- Calculate optimal extent size based on current segment size
                IF segment_bytes <= 16384 THEN
                    lv_initial_extent := 16384;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 32768 THEN
                    lv_initial_extent := 32768;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 65536 THEN
                    lv_initial_extent := 65536;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 131072 THEN
                    lv_initial_extent := 131072;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 262144 THEN
                    lv_initial_extent := 262144;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 524288 THEN
                    lv_initial_extent := 524288;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 1048576 THEN
                    lv_initial_extent := 1048576;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 2097152 THEN
                    lv_initial_extent := 2097152;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 4194304 THEN
                    lv_initial_extent := 4194304;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 8388608 THEN
                    lv_initial_extent := 8388608;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 16777216 THEN
                    lv_initial_extent := 16777216;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 33554432 THEN
                    lv_initial_extent := 33554432;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 67108864 THEN
                    lv_initial_extent := 67108864;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '01';
                ELSIF segment_bytes <= 134217728 THEN
                    lv_initial_extent := 134217728;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '02';
                ELSE
                    lv_initial_extent := 268435456;
                    lv_tablespace_name := RTRIM(lv_tablespace_name,'0123456789') || '03';
                END IF;
                lv_next_extent := lv_initial_extent;
            END IF;
            CLOSE segments_cursor;
        END IF;

        -- Set default INITRANS and MAXTRANS
        IF TO_CHAR(lv_ini_trans) = '0' THEN
            lv_ini_trans := 1;
        END IF;
        IF TO_CHAR(lv_max_trans) = '0' THEN
            lv_max_trans := 1;
        END IF;

        -- Build CREATE INDEX statement
        IF lv_uniqueness = 'UNIQUE' THEN
            a_lin := wri(a_lin, 'CREATE UNIQUE INDEX ' || lv_owner, 0);
        ELSIF lv_uniqueness = 'BITMAP' THEN
            a_lin := wri(a_lin, 'CREATE BITMAP INDEX ' || lv_owner, 0);
        ELSE
            a_lin := wri(a_lin, 'CREATE INDEX ' || lv_owner, 0);
        END IF;

        a_lin := wri(a_lin, '.' || lv_index_name, 0);
        a_lin := wri(a_lin, ' ON ', 0);
        a_lin := wri(a_lin, lv_towner, 0);
        a_lin := wri(a_lin, '.' || lv_table_name, 0);

        -- Add column list for regular tables
        IF lv_table_type = 'TABLE' THEN
            a_lin := wri(a_lin, ' (', 0);
            OPEN col_cursor(lv_owner, lv_index_name);
            LOOP
                FETCH col_cursor INTO lv_column_name, lv_column_position;
                EXIT WHEN col_cursor%NOTFOUND;
                IF lv_column_position <> 1 THEN
                    a_lin := wri(a_lin, ',', 0);
                END IF;
                a_lin := wri(a_lin, CHR(34) || lv_column_name || CHR(34), 0);
            END LOOP;
            CLOSE col_cursor;
            a_lin := wri(a_lin, ')', 0);
        END IF;

        -- Add storage clauses
        a_lin := wri(a_lin, ' TABLESPACE ' || lv_tablespace_name, 0);
        a_lin := wri(a_lin, ' INITRANS ' || TO_CHAR(lv_ini_trans), 0);
        a_lin := wri(a_lin, ' MAXTRANS ' || TO_CHAR(lv_max_trans), 0);
        a_lin := wri(a_lin, ' PCTFREE ' || TO_CHAR(lv_pct_free), 1);

        -- Format extent sizes in M or K where possible
        IF MOD(lv_initial_extent, 1048576) = 0 THEN
            initial_extent_size := TO_CHAR(lv_initial_extent / 1048576) || 'M';
        ELSIF MOD(lv_initial_extent, 1024) = 0 THEN
            initial_extent_size := TO_CHAR(lv_initial_extent / 1024) || 'K';
        ELSE
            initial_extent_size := TO_CHAR(lv_initial_extent);
        END IF;

        IF MOD(lv_next_extent, 1048576) = 0 THEN
            next_extent_size := TO_CHAR(lv_next_extent / 1048576) || 'M';
        ELSIF MOD(lv_next_extent, 1024) = 0 THEN
            next_extent_size := TO_CHAR(lv_next_extent / 1024) || 'K';
        ELSE
            next_extent_size := TO_CHAR(lv_next_extent);
        END IF;

        a_lin := wri(a_lin, '    STORAGE (INITIAL ' || initial_extent_size, 0);
        a_lin := wri(a_lin, ' NEXT ' || next_extent_size, 0);
        a_lin := wri(a_lin, ' MINEXTENTS ' || TO_CHAR(lv_min_extents), 0);
        a_lin := wri(a_lin, ' MAXEXTENTS ' || TO_CHAR(lv_max_extents), 0);
        a_lin := wri(a_lin, ' PCTINCREASE ' || TO_CHAR(lv_pct_increase), 0);
        a_lin := wri(a_lin, ' FREELISTS ' || TO_CHAR(lv_freelists), 0);
        a_lin := wri(a_lin, ' FREELIST GROUPS ' || TO_CHAR(lv_freelist_groups), 0);
        a_lin := wri(a_lin, ');', 1);
    END LOOP;
    CLOSE ind_cursor;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20000,
            'Unexpected error on ' || lv_index_name || ', ' || 
            lv_column_name || ': ' || TO_CHAR(SQLCODE) || ' - Aborting...');
END;
/

-- Generate output file
SET TERMOUT OFF
SET HEADING OFF
SPOOL cr_index.sql
SELECT 'rem cr_index.sql' FROM dual;
SELECT 'rem' FROM dual;
SELECT 'rem ***** All indexes for database ' || name FROM v$database;
SELECT 'rem' FROM dual;
SELECT 'set feedback off' FROM dual;
SELECT text FROM indx_temp ORDER BY lineno;
SPOOL OFF

DROP TABLE indx_temp;
SET TERMOUT ON
SELECT 'Created cr_index.sql...' FROM dual;
EXIT
```

**Output:**
- Creates `cr_index.sql` containing all CREATE INDEX statements
- Can be executed directly to recreate all indexes

**Extent Size Mapping:**

| Segment Size | Initial Extent | Tablespace Suffix |
|--------------|----------------|-------------------|
| ≤ 16 KB      | 16 KB          | 01                |
| ≤ 32 KB      | 32 KB          | 01                |
| ≤ 64 KB      | 64 KB          | 01                |
| ≤ 128 KB     | 128 KB         | 01                |
| ≤ 256 KB     | 256 KB         | 01                |
| ≤ 512 KB     | 512 KB         | 01                |
| ≤ 1 MB       | 1 MB           | 01                |
| ≤ 2 MB       | 2 MB           | 01                |
| ≤ 4 MB       | 4 MB           | 01                |
| ≤ 8 MB       | 8 MB           | 01                |
| ≤ 16 MB      | 16 MB          | 01                |
| ≤ 32 MB      | 32 MB          | 01                |
| ≤ 64 MB      | 64 MB          | 01                |
| ≤ 128 MB     | 128 MB         | 02                |
| > 128 MB     | 256 MB         | 03                |

## Index Rebuild Analysis

### Overview

The Index Rebuild Candidate system identifies indexes that are fragmented and would benefit from rebuilding. It uses sophisticated algorithms to:

- Calculate optimal index size based on table statistics
- Compare actual vs. optimal leaf block counts
- Track index growth and fragmentation over time
- Store historical data for trend analysis

### Setup Tables

```sql
-- Main logging table for current index state
CREATE TABLE index_log (
    owner          VARCHAR2(30),
    index_name     VARCHAR2(30),
    last_inspected DATE,
    leaf_blocks    NUMBER,    
    target_size    NUMBER,
    idx_layout     CLOB
);

ALTER TABLE index_log ADD CONSTRAINT pk_index_log 
    PRIMARY KEY (owner, index_name);

-- Historical tracking table
CREATE TABLE index_hist (
    owner          VARCHAR2(30),
    index_name     VARCHAR2(30),
    inspected_date DATE,
    leaf_blocks    NUMBER,    
    target_size    NUMBER,
    idx_layout     VARCHAR2(4000)
);

ALTER TABLE index_hist ADD CONSTRAINT pk_index_hist 
    PRIMARY KEY (owner, index_name, inspected_date);
```

### Configuration Parameters

The package uses the following constants:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `vMinBlks` | 1000 | Minimum leaf blocks for analysis (smaller indexes skipped) |
| `vScaleFactor` | 0.6 | Rebuild threshold (60% of current size) |
| `vTargetUse` | 90% | Target block utilization (equates to PCTFREE 10) |
| `vHistRet` | 10 | Number of historical records to retain per index |

### Index Utility Package

```sql
CREATE OR REPLACE PACKAGE index_util AUTHID CURRENT_USER IS
    vMinBlks     CONSTANT POSITIVE := 1000;
    vScaleFactor CONSTANT NUMBER := 0.6;
    vTargetUse   CONSTANT POSITIVE := 90;
    vHistRet     CONSTANT POSITIVE := 10;
    
    PROCEDURE inspect_schema (aSchemaName IN VARCHAR2);
    PROCEDURE inspect_index (
        aIndexOwner IN VARCHAR2, 
        aIndexName IN VARCHAR2, 
        aTableOwner IN VARCHAR2, 
        aTableName IN VARCHAR2, 
        aLeafBlocks IN NUMBER
    );
END index_util;
/

CREATE OR REPLACE PACKAGE BODY index_util IS

-- Inspect all eligible indexes in a schema
PROCEDURE inspect_schema (aSchemaName IN VARCHAR2) IS
BEGIN
    FOR r IN (
        SELECT table_owner, table_name, owner index_owner, 
               index_name, leaf_blocks
        FROM dba_indexes  
        WHERE owner = UPPER(aSchemaname)
          AND index_type IN ('NORMAL', 'NORMAL/REV', 'FUNCTION-BASED NORMAL')
          AND partitioned = 'NO'  
          AND temporary = 'N'  
          AND dropped = 'NO'  
          AND status = 'VALID'  
          AND last_analyzed IS NOT NULL  
        ORDER BY owner, table_name, index_name
    ) LOOP
        IF r.leaf_blocks > vMinBlks THEN
            inspect_index(r.index_owner, r.index_name, r.table_owner, 
                         r.table_name, r.leaf_blocks);
        END IF;
    END LOOP;
    COMMIT;
END inspect_schema;

-- Analyze a specific index for rebuild candidacy
PROCEDURE inspect_index (
    aIndexOwner IN VARCHAR2, 
    aIndexName IN VARCHAR2, 
    aTableOwner IN VARCHAR2, 
    aTableName IN VARCHAR2, 
    aLeafBlocks IN NUMBER
) IS
    vLeafEstimate NUMBER;  
    vBlockSize    NUMBER;
    vOverhead     NUMBER := 192; -- Leaf block overhead in bytes
    vIdxObjID     NUMBER;
    vSqlStr       VARCHAR2(4000);
    vIndxLyt      CLOB;
    vCnt          NUMBER := 0;
    
    TYPE IdxRec IS RECORD (
        rows_per_block NUMBER, 
        cnt_blocks NUMBER
    );
    TYPE IdxTab IS TABLE OF IdxRec;
    l_data IdxTab;
    
BEGIN  
    -- Get block size for the index's tablespace
    SELECT a.block_size 
    INTO vBlockSize 
    FROM dba_tablespaces a, dba_indexes b 
    WHERE b.index_name = aIndexName 
      AND b.owner = aIndexOwner 
      AND a.tablespace_name = b.tablespace_name;
    
    -- Calculate optimal leaf block count
    SELECT ROUND(
        100 / vTargetUse * (
            ind.num_rows * (tab.rowid_length + ind.uniq_ind + 4) + 
            SUM((tc.avg_col_len) * (tab.num_rows))
        ) / (vBlockSize - vOverhead)
    ) index_leaf_estimate  
    INTO vLeafEstimate  
    FROM (
        SELECT /*+ no_merge */ 
               table_name, num_rows, 
               DECODE(partitioned, 'YES', 10, 6) rowid_length  
        FROM dba_tables
        WHERE table_name = aTableName  
          AND owner = aTableOwner
    ) tab,  
    (
        SELECT /*+ no_merge */ 
               index_name, index_type, num_rows, 
               DECODE(uniqueness, 'UNIQUE', 0, 1) uniq_ind  
        FROM dba_indexes  
        WHERE table_owner = aTableOwner  
          AND table_name = aTableName  
          AND owner = aIndexOwner  
          AND index_name = aIndexName
    ) ind,  
    (
        SELECT /*+ no_merge */ column_name  
        FROM dba_ind_columns  
        WHERE table_owner = aTableOwner  
          AND table_name = aTableName
          AND index_owner = aIndexOwner   
          AND index_name = aIndexName
    ) ic,  
    (
        SELECT /*+ no_merge */ column_name, avg_col_len  
        FROM dba_tab_cols  
        WHERE owner = aTableOwner  
          AND table_name = aTableName
    ) tc  
    WHERE tc.column_name = ic.column_name  
    GROUP BY ind.num_rows, ind.uniq_ind, tab.rowid_length;
    
    -- Check if index is a rebuild candidate
    IF vLeafEstimate < vScaleFactor * aLeafBlocks THEN
        -- Get index object ID
        SELECT object_id 
        INTO vIdxObjID
        FROM dba_objects  
        WHERE owner = aIndexOwner
          AND object_name = aIndexName;
        
        -- Build dynamic SQL to analyze index layout
        vSqlStr := 'SELECT rows_per_block, COUNT(*) blocks ' ||
                   'FROM (SELECT /*+ cursor_sharing_exact ' ||
                   'dynamic_sampling(0) no_monitoring no_expand index_ffs(' || 
                   aTableName || ',' || aIndexName || ') noparallel_index(' || 
                   aTableName || ',' || aIndexName || ') */ sys_op_lbid(' || 
                   vIdxObjID || ', ''L'', ' || aTableName || 
                   '.rowid) block_id, COUNT(*) rows_per_block FROM ' || 
                   aTableOwner || '.' || aTableName || 
                   ' GROUP BY sys_op_lbid(' || vIdxObjID || ', ''L'', ' || 
                   aTableName || '.rowid)) GROUP BY rows_per_block ' ||
                   'ORDER BY rows_per_block';
        
        EXECUTE IMMEDIATE vSqlStr BULK COLLECT INTO l_data;
        
        -- Build layout string
        vIndxLyt := '';
        FOR i IN l_data.FIRST..l_data.LAST LOOP
            vIndxLyt := vIndxLyt || l_data(i).rows_per_block || ' - ' || 
                       l_data(i).cnt_blocks || CHR(10);
        END LOOP;
        
        -- Check if index already exists in log
        SELECT COUNT(*) 
        INTO vCnt 
        FROM index_log 
        WHERE owner = aIndexOwner 
          AND index_name = aIndexName;
        
        IF vCnt = 0 THEN
            -- New entry
            INSERT INTO index_log 
            VALUES (aIndexOwner, aIndexName, SYSDATE, aLeafBlocks, 
                   ROUND(vLeafEstimate, 2), vIndxLyt);
        ELSE
            -- Update existing entry
            vCnt := 0;
            
            -- Check history count
            SELECT COUNT(*) 
            INTO vCnt 
            FROM index_hist 
            WHERE owner = aIndexOwner 
              AND index_name = aIndexName;
            
            -- Maintain history retention limit
            IF vCnt >= vHistRet THEN
                DELETE FROM index_hist
                WHERE owner = aIndexOwner
                  AND index_name = aIndexName
                  AND inspected_date = (
                      SELECT MIN(inspected_date)
                      FROM index_hist
                      WHERE owner = aIndexOwner
                        AND index_name = aIndexName
                  );
            END IF;
            
            -- Archive current entry to history
            INSERT INTO index_hist 
            SELECT * 
            FROM index_log 
            WHERE owner = aIndexOwner 
              AND index_name = aIndexName;
            
            -- Update current entry
            UPDATE index_log  
            SET last_inspected = SYSDATE,
                leaf_blocks = aLeafBlocks,
                target_size = ROUND(vLeafEstimate, 2),
                idx_layout = vIndxLyt
            WHERE owner = aIndexOwner 
              AND index_name = aIndexName;
        END IF;
    END IF;
END inspect_index;

END index_util;
/
```

### Query Rebuild Candidates

After running the analysis, query the results:

```sql
-- Show all rebuild candidates
SELECT owner,
       index_name,
       last_inspected,
       leaf_blocks AS current_blocks,
       target_size AS optimal_blocks,
       ROUND((leaf_blocks - target_size) / leaf_blocks * 100, 2) AS waste_pct
FROM index_log
WHERE leaf_blocks > target_size
ORDER BY (leaf_blocks - target_size) DESC;

-- Show index layout distribution
SELECT owner,
       index_name,
       idx_layout
FROM index_log
WHERE index_name = 'YOUR_INDEX_NAME';
```

### Understanding Index Layout

The `idx_layout` column shows the distribution of rows per block:

```
1 - 245      (245 blocks with 1 row each)
2 - 189      (189 blocks with 2 rows each)
3 - 156      (156 blocks with 3 rows each)
...
```

This helps identify:
- **Sparse blocks** (low rows per block) indicating fragmentation
- **Dense blocks** (high rows per block) indicating good utilization
- **Distribution patterns** showing index health

## Usage Examples

### Example 1: Check Indexes for a Table

```sql
SQL> @check_indexes.sql
Enter table name: EMPLOYEES

IIO        IIN                       IIC                       UNIQUENESS
---------- ------------------------- ------------------------- ----------
HR         EMP_EMAIL_UK              EMAIL                     UNIQUE

HR         EMP_EMP_ID_PK             EMPLOYEE_ID               UNIQUE

HR         EMP_DEPARTMENT_IX         DEPARTMENT_ID             NONUNIQUE

HR         EMP_JOB_IX                JOB_ID                    NONUNIQUE

HR         EMP_MANAGER_IX            MANAGER_ID                NONUNIQUE

HR         EMP_NAME_IX               LAST_NAME                 NONUNIQUE
                                     FIRST_NAME
```

### Example 2: Generate Index Creation Scripts

```sql
SQL> @create_index_script.sql
Creating index build script...
