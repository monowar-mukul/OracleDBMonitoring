# Oracle DataPump Export/Import Complete Guide

## Table of Contents
- [Initial Setup and Grants](#initial-setup-and-grants)
- [Common Issues and Solutions](#common-issues-and-solutions)
- [Monitoring DataPump Jobs](#monitoring-datapump-jobs)
- [Performance Tuning](#performance-tuning)
- [Job Management](#job-management)
- [Export Examples](#export-examples)
- [Import Examples](#import-examples)
- [Troubleshooting](#troubleshooting)
- [Scripts and Utilities](#scripts-and-utilities)

## Initial Setup and Grants

### Required Grants
```sql
GRANT imp_full_database TO system;
GRANT exp_full_database TO system;
CREATE DATABASE LINK EXA_MIG CONNECT TO SYSTEM IDENTIFIED BY <password> USING '<connect string/Service>';
```

### Directory Setup
```sql
CREATE OR REPLACE DIRECTORY exp_dir AS '/path/to/export/directory';
GRANT ALL ON DIRECTORY exp_dir TO system;
```

## Common Issues and Solutions

### ORA-39083: Object Type Failed to Create

**Cause:** Oracle Text was installed in the source database but not in the target database.

**Solutions:**
1. Ignore the errors and check data in target database after import
2. Install Oracle Text in target database first, then perform import

### ORA-39083 and ORA-02304 During Import

**Error:** Invalid object identifier literal with OID conflicts

**Cause:** Oracle trying to import object with existing OID

**Solution:** Add `TRANSFORM=OID:N` to impdp command
```bash
impdp USERID="/ as sysdba" CONTENT=ALL DIRECTORY=UP_MIG \
JOB_NAME=IMPDIR_JOB_01 LOGFILE=imp_all_2.log \
DUMPFILE=exp_source_%U.dmp PARALLEL=3 \
TRACE=480300 METRICS=y TRANSFORM=oid:n
```

### ORA-00054: Resource Busy

**Find the locking session:**
```sql
SELECT c.object_name, b.sid, b.serial#, b.machine
FROM v$locked_object a, v$session b, dba_objects c
WHERE b.sid = a.session_id
AND a.object_id = c.object_id;
```

**Kill the session:**
```sql
ALTER SYSTEM KILL SESSION '<SID>,<Serial>';
```

### IMP-00032: SQL Statement Exceeded Buffer Length

**Solution:** Use BUFFER parameter with maximum value
```bash
imp BUFFER=1000000 ...
```

### Corrupted Export Dump

**Error:** ORA-39002, ORA-31694, ORA-31640, ORA-19505, ORA-27046

**Cause:** File size not multiple of logical block size

**Solution:** Check file integrity and re-export if necessary

### STREAMS_POOL_SIZE Issue

**Error:** UDE-00008, ORA-39078 unable to dequeue message

**Solution:**
```sql
ALTER SYSTEM SET STREAMS_POOL_SIZE=100M SCOPE=spfile;
SHUTDOWN IMMEDIATE;
STARTUP;
```

## Monitoring DataPump Jobs

### Check Active Jobs
```sql
SELECT owner_name, job_name, operation, job_mode, state, attached_sessions
FROM dba_datapump_jobs
WHERE job_name NOT LIKE 'BIN$%'
ORDER BY 1,2;
```

### Monitor Progress
```sql
SELECT sid, serial#, sofar, totalwork, dp.owner_name, dp.state, dp.job_mode
FROM gv$session_longops sl, gv$datapump_job dp
WHERE sl.opname = dp.job_name AND sofar != totalwork;
```

### Detailed Progress with Time Remaining
```sql
SELECT x.job_name, b.state, b.job_mode, b.degree,
       x.owner_name, z.sql_text, p.message,
       p.totalwork, p.sofar,
       ROUND((p.sofar/p.totalwork)*100,2) AS done,
       p.time_remaining
FROM dba_datapump_jobs b 
LEFT JOIN dba_datapump_sessions x ON (x.job_name = b.job_name)
LEFT JOIN v$session y ON (y.saddr = x.saddr)
LEFT JOIN v$sql z ON (y.sql_id = z.sql_id)
LEFT JOIN v$session_longops p ON (p.sql_id = y.sql_id)
WHERE y.module='Data Pump Worker'
AND p.time_remaining > 0;
```

### Monitor Table Import Progress
```sql
SELECT SUBSTR(sql_text, INSTR(sql_text,'"')+1, 
              INSTR(sql_text,'"', 1, 2)-INSTR(sql_text,'"')-1) AS table_name,
       rows_processed,
       ROUND((SYSDATE - TO_DATE(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60, 1) AS minutes,
       TRUNC(rows_processed / 
             ((SYSDATE-TO_DATE(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60)) AS rows_per_min
FROM v$sqlarea 
WHERE UPPER(sql_text) LIKE 'INSERT % INTO "%'
AND command_type = 2 
AND open_versions > 0;
```

## Performance Tuning

### Key Parameters for Performance
```bash
# Initialization parameters
DISK_ASYNCH_IO=TRUE
DB_BLOCK_CHECKING=FALSE
DB_BLOCK_CHECKSUM=FALSE
PROCESSES=<high_value>
SESSIONS=<high_value>
PARALLEL_MAX_SERVERS=<high_value>
PGA_AGGREGATE_TARGET=<large_value>
```

### Export with Compression
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=compressed_export.dmp \
LOGFILE=compressed_export.log \
COMPRESSION=ALL \
PARALLEL=4 \
SCHEMAS=schema_name
```

### Estimate Export Size
```bash
# Using BLOCKS method (default)
expdp system/password DIRECTORY=exp_dir SCHEMAS=schema_name ESTIMATE=BLOCKS ESTIMATE_ONLY=y

# Using STATISTICS method (more accurate if tables are analyzed)
expdp system/password DIRECTORY=exp_dir SCHEMAS=schema_name ESTIMATE=STATISTICS ESTIMATE_ONLY=y
```

## Job Management

### Attach to Running Job
```bash
expdp system attach=job_name
impdp system attach=job_name
```

### Interactive Commands
- `STATUS` - Check job status
- `CONTINUE_CLIENT` - Resume the job
- `KILL_JOB` - Terminate the job
- `START_JOB` - Restart a stopped job
- `STOP_JOB` - Stop the current job
- `PARALLEL=n` - Change parallelism

### Clean Up Orphaned Jobs
```sql
-- 1. Check orphaned jobs
SELECT owner_name, job_name, operation, job_mode, state, attached_sessions 
FROM dba_datapump_jobs;

-- 2. Drop master tables for NOT RUNNING jobs
DROP TABLE system.SYS_EXPORT_SCHEMA_01;
DROP TABLE system.SYS_EXPORT_SCHEMA_02;

-- 3. Purge from recyclebin
PURGE TABLE system.SYS_EXPORT_SCHEMA_01;
PURGE TABLE system.SYS_EXPORT_SCHEMA_02;
```

## Export Examples

### Full Database Export
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=full_db_%U.dmp \
LOGFILE=full_db.log \
FULL=y \
PARALLEL=4 \
COMPRESSION=ALL \
REUSE_DUMPFILES=y
```

### Schema Export with Flashback
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=schema_export.dmp \
LOGFILE=schema_export.log \
SCHEMAS=hr,scott \
FLASHBACK_SCN=1386808 \
REUSE_DUMPFILES=y
```

### Table Export with Query
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=table_export.dmp \
LOGFILE=table_export.log \
TABLES=scott.emp,scott.dept \
QUERY=emp:"WHERE job='ANALYST' OR sal >= 3000"
```

### Metadata Only Export
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=metadata_only.dmp \
LOGFILE=metadata_only.log \
SCHEMAS=hr \
CONTENT=METADATA_ONLY
```

### Export with Exclusions
```bash
expdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=export_no_indexes.dmp \
LOGFILE=export_no_indexes.log \
SCHEMAS=hr \
EXCLUDE=INDEX,CONSTRAINT
```

## Import Examples

### Full Database Import
```bash
impdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=full_db_%U.dmp \
LOGFILE=full_import.log \
FULL=y \
PARALLEL=4
```

### Schema Remap
```bash
impdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=schema_export.dmp \
LOGFILE=schema_import.log \
REMAP_SCHEMA=old_schema:new_schema \
REMAP_TABLESPACE=old_tbs:new_tbs
```

### Table Import with Different Actions
```bash
# Skip existing tables
impdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=table_export.dmp \
TABLES=hr.employees \
TABLE_EXISTS_ACTION=SKIP

# Append to existing tables
impdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=table_export.dmp \
TABLES=hr.employees \
TABLE_EXISTS_ACTION=APPEND

# Replace existing tables
impdp system/password \
DIRECTORY=exp_dir \
DUMPFILE=table_export.dmp \
TABLES=hr.employees \
TABLE_EXISTS_ACTION=REPLACE
```

### Network Import (Database Link)
```bash
impdp system/password \
DIRECTORY=exp_dir \
LOGFILE=network_import.log \
NETWORK_LINK=source_db_link \
SCHEMAS=hr \
REMAP_SCHEMA=hr:hr_copy
```

## Troubleshooting

### Check DataPump Sessions
```sql
SELECT owner_name, job_name, osuser
FROM dba_datapump_sessions DPS, v$session S
WHERE S.saddr = DPS.saddr;
```

### Find Session IDs
```sql
SELECT sid, serial#
FROM v$session s, dba_datapump_sessions d
WHERE s.saddr = d.saddr;
```

### Check Wait Events
```sql
SELECT * FROM v$session_event WHERE sid = <session_id>;
```

### Trace DataPump Operations
Add `TRACE=480300` parameter to expdp/impdp commands for detailed tracing.

## Scripts and Utilities

### Export with GZIP Compression
```bash
# Create named pipe
mknod gzip_pipe p

# Start compression in background
gzip -c < gzip_pipe > ./export.dmp.gz &

# Run export
nohup exp parfile=./exp.par &
```

### Generate Export Commands for All Tables
```sql
SELECT 'expdp system/password tables='|| owner||'.'||segment_name ||
       ' dumpfile='||segment_name||'.dmp logfile='||segment_name||'.log directory=exp_dir;'
FROM dba_segments
WHERE owner = 'SCHEMA_NAME'
AND segment_type = 'TABLE'
ORDER BY bytes;
```

### Extract DDL Using DBMS_METADATA
```sql
SET LINESIZE 132 PAGESIZE 0 FEEDBACK OFF VERIFY OFF TRIMSPOOL ON LONG 1000000
EXECUTE DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM,'PRETTY',true);
EXECUTE DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM,'SQLTERMINATOR',true);

SELECT DBMS_METADATA.GET_DDL('PACKAGE BODY', object_name)
FROM user_objects 
WHERE object_type = 'PACKAGE BODY';
```

### Create Synonyms Script
```sql
SPOOL create_synonyms.sql
SELECT 'CREATE OR REPLACE '|| DECODE(owner,'PUBLIC','PUBLIC ',null) ||  
       'SYNONYM ' || DECODE(owner,'PUBLIC',null, LOWER(owner) || '.') ||  
       LOWER(synonym_name) || ' FOR ' || LOWER(table_owner) || '.' || 
       LOWER(table_name) || DECODE(db_link,null,null,'@'||db_link) || ';'
FROM sys.dba_synonyms 
WHERE table_owner NOT IN ('SI_INFORMTN_SCHEMA','SYS','SYSTEM','ORDSYS','XDB',
                          'CTXSYS','DMSYS','EXFSYS','MDSYS','SYSMAN','WKSYS','WMSYS')
ORDER BY owner, table_name;
SPOOL OFF;
```

## Parameter Comparison: EXP/IMP vs DataPump

| EXP/IMP Parameter | DataPump Equivalent | Description |
|-------------------|-------------------|-------------|
| owner | schemas | Schema specification |
| file | dumpfile | Dump file name |
| log | logfile/nologfile | Log file specification |
| fromuser,touser | remap_schema | Schema remapping |
| indexfile | sqlfile | SQL file generation |
| consistent=y | flashback_scn/flashback_time | Consistent export |

## Best Practices

1. **Use PARALLEL** for large exports/imports to improve performance
2. **Set appropriate REUSE_DUMPFILES=y** to overwrite existing dump files
3. **Use COMPRESSION=ALL** for network transfers or storage savings
4. **Monitor jobs** using the queries provided in this guide
5. **Use FLASHBACK_SCN or FLASHBACK_TIME** for consistent exports of active databases
6. **Clean up orphaned jobs** regularly to avoid resource issues
7. **Use ESTIMATE_ONLY=y** first to check export size before actual export
8. **Set TABLE_EXISTS_ACTION** appropriately for imports
9. **Use TRANSFORM parameters** to modify object attributes during import
10. **Keep master tables** with KEEP_MASTER=y for job analysis

## Additional Notes

- DataPump is server-based, unlike traditional EXP/IMP which are client-based
- Directory objects must exist and have proper permissions
- Parallel processing requires multiple dump files (use %U in filename)
- Network imports can be performed without intermediate dump files
- HCC (Hybrid Columnar Compression) tables require special handling on non-Exadata systems
- Always test imports on non-production systems first
