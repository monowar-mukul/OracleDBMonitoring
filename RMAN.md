# Oracle RMAN Scripts and Commands Reference

A comprehensive collection of Oracle Recovery Manager (RMAN) scripts, commands, and monitoring queries for database backup, recovery, and maintenance operations.

## Table of Contents
- [Configuration](#configuration)
- [Backup Operations](#backup-operations)
- [Monitoring and Reporting](#monitoring-and-reporting)
- [Archive Log Management](#archive-log-management)
- [Recovery Operations](#recovery-operations)
- [Maintenance and Troubleshooting](#maintenance-and-troubleshooting)
- [Block Recovery](#block-recovery)
- [Performance Monitoring](#performance-monitoring)

---

## Configuration

### Basic RMAN Configuration
```sql
RMAN> configure device type disk clear;
RMAN> configure device type disk backup type to backupset;
RMAN> configure device type disk backup type to compressed backupset;
RMAN> configure channel device type disk maxpiecesize 1g;
RMAN> configure retention policy to redundancy 2;
RMAN> show retention policy;
RMAN> configure default device type clear;
RMAN> configure channel device type sbt clear;
RMAN> configure channel 1 device type disk clear;
RMAN> configure backup optimization on;
```

### Data Guard Configuration
```sql
RMAN> CONFIGURE DB_UNIQUE_NAME PIIB1_01 CONNECT IDENTIFIER 'PIIB1_01';
RMAN> CONFIGURE DB_UNIQUE_NAME PIIB1_02 CONNECT IDENTIFIER 'PIIB1_02';
RMAN> RESYNC CATALOG FROM DB_UNIQUE_NAME PIIB1_02;
```

### Multiple Backup Copies
```sql
RMAN> configure datafile backup copies for device type disk to 2;
RMAN> configure datafile backup copies for device type sbt to 2;
RMAN> configure channel device type disk format '/save1/%U','/save2/%U','save3/%U';
```

### Encryption Configuration
```sql
-- With Oracle Wallet
SQL> alter system set encryption key identified by "welcome1";
RMAN> configure encryption for database on;

-- Password Encryption (without wallet)
RMAN> set encryption on identified by password only;
RMAN> configure encryption for database off;
```

---

## Backup Operations

### Basic Backup Commands
```sql
-- Tagged backup
RMAN> backup as copy tag users_bkp tablespace users;

-- Faster backup with parallelism and sections
RMAN> configure device type sbt parallelism 3;
RMAN> backup section size 150m tablespace system;

-- Incremental backups
RMAN> backup incremental level 0 database;
RMAN> backup incremental level 1 database;
```

### Archive Log Backups
```sql
-- Basic archive log backup
RMAN> backup archivelog all;
RMAN> backup archivelog all delete input;

-- Archive log backup with retention
RMAN> backup device type disk archivelog all delete all input;
RMAN> delete force noprompt archivelog until time "to_date(SYSDATE-(2/24))" backed up 1 times to disk;
```

### Long-term Retention Backups
```sql
run {
    backup database
    tag quarterly
    keep until time 'sysdate+180'
    restore point 2013Q1;
}
```

### Conditional Backups
```sql
RMAN> backup database not backed up;
RMAN> backup device type sbt archivelog all not backed up 2 times;
RMAN> backup database not backed up since time 'sysdate-31';
```

---

## Monitoring and Reporting

### Check Recent Backup Status (Non-Archive Log)
```sql
set linesize 500
col BACKUP_SIZE for a20
COL START_TIME FORMAT a20 heading "Start Time"
COL END_TIME FORMAT a20 heading "End Time"
COL STATUS FORMAT a14 heading "Status"

SELECT
    INPUT_TYPE "BACKUP_TYPE",
    STATUS,
    TO_CHAR(START_TIME,'MM/DD/YYYY:hh24:mi:ss') as START_TIME,
    TO_CHAR(END_TIME,'MM/DD/YYYY:hh24:mi:ss') as END_TIME,
    TRUNC((ELAPSED_SECONDS/60),2) "ELAPSED_TIME(Min)",
    OUTPUT_BYTES_DISPLAY "BACKUP_SIZE",
    OUTPUT_DEVICE_TYPE "OUTPUT_DEVICE"
FROM V$RMAN_BACKUP_JOB_DETAILS
WHERE start_time > SYSDATE -7
AND INPUT_TYPE != 'ARCHIVELOG'
ORDER BY END_TIME DESC;
```

### Check Archive Log Backup Status
```sql
SELECT 
    INPUT_TYPE "BACKUP_TYPE",
    STATUS,
    TO_CHAR(START_TIME,'MM/DD/YYYY:hh24:mi:ss') as START_TIME,
    TO_CHAR(END_TIME,'MM/DD/YYYY:hh24:mi:ss') as END_TIME,
    TRUNC((ELAPSED_SECONDS/60),2) "ELAPSED_TIME(Min)",
    OUTPUT_BYTES_DISPLAY "BACKUP_SIZE",
    OUTPUT_DEVICE_TYPE "OUTPUT_DEVICE"
FROM V$RMAN_BACKUP_JOB_DETAILS
WHERE start_time > SYSDATE -1/24
AND INPUT_TYPE = 'ARCHIVELOG'
ORDER BY END_TIME DESC;
```

### Monitor RMAN Job Progress
```sql
SELECT SID, SERIAL#, CONTEXT, SOFAR, TOTALWORK,
       ROUND(SOFAR/TOTALWORK*100,2) "%_COMPLETE"
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%'
  AND OPNAME NOT LIKE '%aggregate%'
  AND TOTALWORK != 0
  AND SOFAR <> TOTALWORK;
```

### Get RMAN Job Output
```sql
-- First get SESSION_RECID and SESSION_STAMP from backup job details
select SESSION_KEY,SESSION_RECID,SESSION_STAMP, INPUT_TYPE, STATUS, 
       to_char(START_TIME,'mm/dd/yy hh24:mi') start_time,
       to_char(END_TIME,'mm/dd/yy hh24:mi') end_time,
       elapsed_seconds/3600 hrs 
from V$RMAN_BACKUP_JOB_DETAILS
order by session_key;

-- Then get detailed output
set lines 200
set pages 1000
select output
from GV$RMAN_OUTPUT
where session_recid = &SESSION_RECID
and session_stamp = &SESSION_STAMP
order by recid;
```

---

## Archive Log Management

### List and Restore Archive Logs
```sql
-- List archive logs for specific time period
RMAN> sql "alter session set nls_date_format=''DD-MON-YYYY HH24:MI:SS''";
RMAN> LIST BACKUP OF ARCHIVELOG TIME between "TO_DATE('09/18/2015 22:00:00', 'MM/DD/YYYY hh24:mi:ss')" and "TO_DATE('09/18/2015 23:00:00','MM/DD/YYYY hh24:mi:ss')";
RMAN> LIST BACKUP OF ARCHIVELOG FROM TIME "SYSDATE-6/24";
```

### Restore Archive Logs from Disk
```sql
run {
    set archivelog destination to '/tmp/arch_restore';
    allocate channel c1 type disk;
    allocate channel c2 type disk;
    restore archivelog
    from time = "to_date('09-03-2016:09:00:00','DD-MM-YYYY:HH24:MI:SS')"
    until time = "to_date('09-03-2016:11:20:00','DD-MM-YYYY:HH24:MI:SS')";
    RELEASE CHANNEL c1;
    RELEASE CHANNEL c2;
}
```

### Restore Archive Logs from Tape
```sql
RUN {
    ALLOCATE CHANNEL ch1 TYPE 'SBT_TAPE'
    PARMS='ENV=(NB_ORA_SERVER=server,NB_ORA_CLIENT=client,NB_ORA_POLICY=policy)';
    restore archivelog from logseq=25318 until logseq=25324 thread=1;
    RELEASE CHANNEL ch1;
}
```

### Archive Log Cleanup
```sql
RMAN> crosscheck backup of archivelog all;
RMAN> delete expired backup;
RMAN> delete noprompt archivelog until time 'SYSDATE - 3';
RMAN> configure archivelog deletion policy to backed up 2 times to device type sbt;
```

---

## Recovery Operations

### Point-in-Time Recovery

#### Time-Based Recovery
```sql
connect target /
startup mount;
run {
    set until time '16-NOV-2015 11:00:00';  
    restore database;
    recover database;
}
alter database open resetlogs;
```

#### SCN-Based Recovery
```sql
connect target /
startup mount;
run {
    set until scn 950;
    restore database;
    recover database;
}
alter database open resetlogs;
```

#### Log Sequence Recovery
```sql
connect target /
startup mount;
restore database until sequence 50;
recover database until sequence 50;
alter database open resetlogs;
```

### Restore Preview Commands
```sql
RMAN> restore database preview;
RMAN> restore database from tag TAG20060927T183743 preview;
RMAN> restore datafile 1, 2, 3, 4 preview;
RMAN> restore archivelog all preview;
RMAN> restore database preview summary;
```

### Incarnation Management
```sql
RMAN> list incarnation of database;
RMAN> reset database to incarnation 1;
RMAN> restore database until time "to_date('03-sep-2006 00:00:00', 'dd-mon-rrrr hh24:mi:ss')";
```

---

## Block Recovery

### Identify and Recover Corrupt Blocks
```sql
-- Check for corrupt blocks
SQL> select * from v$database_block_corruption;

-- Recover specific blocks
RMAN> recover datafile 1 block 101760;
RMAN> recover datafile 5 block 130 to 131,142;
RMAN> BLOCKRECOVER corruption list;

-- Recover with specific options
RMAN> recover datafile 5 block 24 from tag=tues_backup;
RMAN> recover datafile 6 block 89 restore until sequence 546;
RMAN> recover datafile 5 block 32 restore until 'sysdate-1';
```

### Block Corruption Analysis
```sql
-- Identify affected segments
SELECT e.owner, e.segment_type, e.segment_name, e.partition_name, c.file#,
       greatest(e.block_id, c.block#) corr_start_block#,
       least(e.block_id+e.blocks-1, c.block#+c.blocks-1) corr_end_block#,
       least(e.block_id+e.blocks-1, c.block#+c.blocks-1) - greatest(e.block_id, c.block#) + 1 blocks_corrupted
FROM dba_extents e, v$database_block_corruption c
WHERE e.file_id = c.file#
  AND e.block_id <= c.block# + c.blocks - 1
  AND e.block_id + e.blocks - 1 >= c.block#
ORDER BY c.file#, corr_start_block#;
```

---

## Performance Monitoring

### Identify RMAN Sessions
```sql
SELECT s.SID, p.SPID, s.CLIENT_INFO
FROM V$PROCESS p, V$SESSION s
WHERE p.ADDR = s.PADDR
AND CLIENT_INFO LIKE 'rman%';
```

### Monitor Backup Performance
```sql
SELECT session_recid, input_bytes_per_sec_display,
       output_bytes_per_sec_display,
       time_taken_display, end_time
FROM v$rman_backup_job_details
ORDER BY end_time;
```

### Backup I/O Statistics
```sql
SELECT sid, inst_id, type, filename, status,
       to_char(open_time, 'mm/dd/yyyy hh24:mi:ss') open_time,
       to_char(close_time, 'mm/dd/yyyy hh24:mi:ss') close_time,
       round(effective_bytes_per_second/1024/1024) mbps
FROM gv$backup_async_io    
WHERE close_time > SYSDATE - 1
AND type='AGGREGATE'
ORDER BY close_time;
```

---

## Maintenance and Troubleshooting

### Debug and Logging
```bash
# Enable debug output
rman target / debug=all log=rman_output.log
rman target / debug=io

# Spool RMAN output
spool log to '/tmp/rman/backuplog.f';
backup datafile 1;
spool log off;
```

### Catalog Operations
```sql
-- Catalog files
RMAN> CATALOG ARCHIVELOG '/oracle/oradata/arju/arc001_223.arc';
RMAN> CATALOG DATAFILECOPY '/oradata/backup/users01.dbf' LEVEL 0;
RMAN> CATALOG START WITH '/tmp/backups' NOPROMPT;
RMAN> CATALOG RECOVERY AREA NOPROMPT;

-- Uncatalog operations
RMAN> CHANGE ARCHIVELOG ALL UNCATALOG;
RMAN> CHANGE BACKUP OF TABLESPACE USERS UNCATALOG;
```

### Crosscheck and Delete Operations
```sql
RMAN> crosscheck backup of database;
RMAN> delete expired backup;
RMAN> delete obsolete;
RMAN> delete backup tag='old_production';
RMAN> delete archivelog all backed up 3 times to sbt;
```

### Database Registration Management
```sql
-- Unregister database from catalog
RMAN> unregister database;

-- Or using procedure
SQL> execute dbms_rcvcat.unregisterdatabase(db_key, db_id);
```

---

## Useful Views and Queries

### Last Backup Information
```sql
SELECT dbf.file#, substr(dbf.name,1,55) name,
       to_char(least(nvl(sysdate-max(bdf.completion_time),999999),
                     nvl(sum(sysdate-dbf.creation_time),999999),
                     nvl(sum(sysdate-b.time),999999)),'999999.00') days
FROM v$backup_datafile bdf, v$datafile dbf, v$backup b
WHERE dbf.file# = bdf.file#(+)
  AND dbf.file# = b.file#
GROUP BY dbf.file#, substr(dbf.name,1,55)
ORDER BY days DESC;
```

### Recovery Requirements Check
```sql
SELECT a.name, a.checkpoint_change#, b.checkpoint_change#,
       CASE
           WHEN ((a.checkpoint_change# - b.checkpoint_change#) = 0) THEN 'Startup Normal'
           WHEN ((a.checkpoint_change# - b.checkpoint_change#) > 0) THEN 'Media Recovery'
           WHEN ((a.checkpoint_change# - b.checkpoint_change#) < 0) THEN 'Old Control File'
           ELSE 'Unknown'
       END STATUS
FROM v$datafile a, v$datafile_header b
WHERE a.file# = b.file#;
```

### Block Change Tracking
```sql
-- Check block change tracking status
SELECT filename, status, (bytes)/1024/1024 as "Size in MB"
FROM v$block_change_tracking;

-- Verify block change tracking usage
SELECT file#, avg(datafile_blocks), avg(blocks_read),  
       avg(blocks_read/datafile_blocks) * 100 as "% read for backup"  
FROM v$backup_datafile  
WHERE incremental_level > 0  
AND used_change_tracking = 'YES'  
GROUP BY file#   
ORDER BY file#;
```

---
# Oracle RMAN Troubleshooting Guide

A comprehensive guide for resolving common Oracle RMAN (Recovery Manager) issues and errors.

## Table of Contents

- [ASM File Access Issues](#asm-file-access-issues)
- [Debug Configuration](#debug-configuration)
- [Duplicate Database Issues](#duplicate-database-issues)
- [Flash Recovery Area (FRA) Issues](#flash-recovery-area-fra-issues)
- [Recovery Catalog Issues](#recovery-catalog-issues)
- [Archive Log Issues](#archive-log-issues)
- [Backup Failures](#backup-failures)
- [Memory and Resource Issues](#memory-and-resource-issues)
- [Standby Database Issues](#standby-database-issues)
- [Golden Gate and Streams Issues](#golden-gate-and-streams-issues)
- [Flashback Database Monitoring](#flashback-database-monitoring)

## ASM File Access Issues

### ORA-15028: ASM file not dropped; currently being accessed

**Problem**: ASM files cannot be dropped because they are currently being accessed by archive processes.

**Solution**: Kill and restart the archive processes (they will be automatically restarted by Oracle):

```bash
# 1. Find the process ID for arc processes
ps -ef | grep -i ora_arc*
# Example output: oracle 5607 1 1 19:02 ? 00:00:00 ora_arc9_prod1

# 2. Kill the running process
kill -9 5607

# 3. Check the process is started again
ps -ef | grep -i ora_arc9_prod1

# 4. Repeat for all arc processes running for your instance
```

## Debug Configuration

### Enabling RMAN Debug Mode

```bash
export NLS_DATE_FORMAT="DD-MON-RRRR HH24:MI:SS"
rman target sys/<password>@<tns_string> catalog <catalog_schema>/<password>@<tns_string> auxiliary sys/<password>@<tns_string> debug all trace=debug-duplicate.trc log=duplicate.log
```

**Files to check after debugging**:
- `debug-duplicate.trc`
- `duplicate.log`
- Alert log of target and auxiliary database

## Duplicate Database Issues

### ORA-19625, ORA-1517 During Active Duplicate

**Problem**: Required archives not backed up due to backup optimization feature.

**Solution**: Apply patch 17843104 or turn off backup optimization:

```sql
RMAN> CONFIGURE BACKUP OPTIMIZATION OFF;
```

**Bug Reference**: Bug 17843104 - "DUPLICATE FROM ACTIVE" FAILED BECAUSE ARCHIVELOG FILES WERE SKIPPED

### Temporary File Conflicts During Duplication

**Error**: 
```
RMAN-05517: temporary file conflicts with file used by target database
```

**Solution**: Set new names for temporary files in auxiliary database:

```sql
set newname for tempfile 1 to '/dbv6999/oracle/v6999/temp.dbf';
set newname for tempfile 2 to '/dbv6999/oracle/v6999/v6000_temp_02.dbf';
```

## Flash Recovery Area (FRA) Issues

### ORA-19809: Limit exceeded for recovery files

**Problem**: Flash Recovery Area is full and cannot reclaim space.

**Error Message**:
```
ORA-19809: limit exceeded for recovery files
ORA-19804: cannot reclaim 2144337920 bytes disk space from 32212254720 limit
```

**Solutions**:

1. **Check FRA usage**:
```sql
SELECT name,
       floor(space_limit/1024/1024) "Size_MB",
       ceil(space_used/1024/1024) "Used_MB"
FROM v$recovery_file_dest
ORDER BY name;
```

2. **Increase FRA size**:
```sql
ALTER SYSTEM SET db_recovery_file_dest_size=12G SCOPE=BOTH;
```

3. **Delete obsolete backups**:
```sql
RMAN> DELETE OBSOLETE;
```

### FRA Space Analysis

**Check detailed FRA usage**:
```sql
SELECT file_type,
       percent_space_used,
       percent_space_reclaimable,
       number_of_files
FROM v$flash_recovery_area_usage;
```

## Recovery Catalog Issues

### RMAN-20003: Target database incarnation not found

**Problem**: Database incarnation not registered in recovery catalog.

**Solution**:
```sql
RMAN> LIST INCARNATION OF DATABASE;
RMAN> REGISTER DATABASE;
```

### RMAN-20002: Database already registered

**Error**: Target database already registered in recovery catalog.

**Solution**:
```sql
RMAN> RESET DATABASE;
```

### RMAN-08040: Full resync skipped, control file not current

**Problem**: Control file is not current or backup.

**Solution**: Connect to production database (not DR) and register:
```sql
-- Connect to production database
RMAN> CONNECT TARGET;
RMAN> REGISTER DATABASE;
```

### RMAN-20004: Target database name does not match recovery catalog

**Problem**: Database name mismatch in recovery catalog.

**Investigation**:
```sql
-- Check database details
SELECT dbid, name FROM v$database;
SELECT * FROM v$database_incarnation;

-- Check recovery catalog
SELECT DISTINCT dbid FROM rc_database;
SELECT dbid, name, dbinc_key, resetlogs_change# FROM rc_database;
```

**Solution**: Unregister and re-register the database:
```sql
RMAN> UNREGISTER DATABASE;
RMAN> REGISTER DATABASE;
```

### Snapshot Control File Issues

**Error**: 
```
ORA-01580: error creating control backup file
ORA-27040: file create error, unable to create file
Linux-x86_64 Error: 13: Permission denied
```

**Solution**:
```sql
-- Option 1: Set correct path in database
EXECUTE SYS.DBMS_BACKUP_RESTORE.CFILESETSNAPSHOTNAME('/correct/path/snapcf_db1.f');

-- Option 2: Configure in RMAN
RMAN> CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/correct/path/snapshot.ctl';
```

## Archive Log Issues

### ORA-19625: Error identifying archive log file

**Problem**: Expected archived log not found, compromising recoverability.

**Error**:
```
RMAN-06059: expected archived log not found
ORA-19625: error identifying file /path/to/archivelog
ORA-27037: unable to obtain file status
```

**Solution**:
```sql
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> DELETE NOPROMPT EXPIRED ARCHIVELOG ALL;
```

**Check archive log status**:
```sql
SELECT sequence#, completion_time, backup_count, deleted, status 
FROM v$archived_log 
WHERE name = '&archivelogseqno';
```

### RMAN-08137: Archive log not deleted - needed for standby/streams

**Problem**: Archives cannot be deleted due to standby database or streams requirements.

**Check deletion policy**:
```sql
RMAN> SHOW ALL;
-- Look for: CONFIGURE ARCHIVELOG DELETION POLICY
```

**Verify standby status**:
```sql
SELECT sequence#, thread#, first_time, archived, applied, deleted, status 
FROM v$archived_log;
```

**Check for streams/Golden Gate**:
```sql
SELECT capture_name, queue_owner, capture_user, start_scn, status 
FROM dba_capture;

SELECT min_required_capture_change# FROM v$database;
```

## Backup Failures

### ORA-19573: Cannot obtain sub-shared enqueue for datafile

**Problem**: Managed Recovery Process holds a lock on newly added datafile.

**Solution**: Stop and restart MRP process:
```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```

**Bug Reference**: Bug 2749174 - Auto addition of standby datafile prevents RMAN backup

## Memory and Resource Issues

### ORA-04031: Unable to allocate shared memory

**Error**:
```
ORA-04031: unable to allocate bytes of shared memory
ORA-06508: PL/SQL: could not find program unit being called: "SYS.DBMS_RCVMAN"
```

**Solution**: Increase shared pool size:
```sql
-- Check current size
SHOW PARAMETER shared_pool_size;

-- Increase size
ALTER SYSTEM SET shared_pool_size=100M SCOPE=BOTH;
```

### Export Parameter Issues

**Error**: 
```
ORA-00068: invalid value for parameter sort_area_size
```

**Solution**: Restart the database to clear parameter corruption:
```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

## Golden Gate and Streams Issues

### Dropping Capture Processes

When Golden Gate extracts prevent archive deletion:

```sql
-- Check capture processes
SELECT capture_name, queue_owner, capture_user, start_scn, status 
FROM dba_capture;

-- Drop disabled captures
EXEC DBMS_CAPTURE_ADM.DROP_CAPTURE('capture_name');
```

## Flashback Database Monitoring

### Check FRA Configuration and Usage

```sql
-- Show FRA parameters
SHOW PARAMETER db_recovery_file_dest;
SHOW PARAMETER control_file_record_keep_time;
SHOW PARAMETER db_flashback_retention_target;
SHOW PARAMETER undo_retention;
```

### Monitor Flashback Database Status

```sql
-- Check oldest flashback SCN and time
SELECT oldest_flashback_scn, oldest_flashback_time 
FROM v$flashback_database_log;

-- Check flashback database statistics
SELECT * FROM v$flashback_database_stat;

-- Check reclaimable archives
SELECT applied, deleted, 
       decode(rectype,11,'YES','NO') reclaimable,
       count(*), min(sequence#), max(sequence#)
FROM v$archived_log 
LEFT OUTER JOIN sys.x$kccagf USING(recid) 
WHERE is_recovery_dest_file='YES' 
AND name IS NOT NULL 
GROUP BY applied, deleted, decode(rectype,11,'YES','NO') 
ORDER BY 5;
```

### Standby FRA Space Issue Workaround

**Problem**: Bug 14227959 - Standby did not release space in FRA when in mount mode.

**Workaround**: Schedule daily execution:
```sql
RMAN> SQL "BEGIN dbms_backup_restore.refreshagedfiles; END;";
```

## General Troubleshooting Tips

### Connection Issues

**RMAN-04004: Error from recovery catalog database**
- Check TNS entries for all Oracle homes
- Verify listener status
- Test connectivity with SQL*Plus

### File Permission Issues
- Verify Oracle user has proper permissions on backup directories
- Check disk space availability
- Ensure proper ownership of Oracle files

### Best Practices
- Regular monitoring of FRA usage
- Implement proper archive log deletion policies
- Schedule regular catalog synchronization
- Monitor standby database lag
- Keep recovery catalog up to date

## Useful Queries

### Check Database Information
```sql
-- Database details
SELECT dbid, name, log_mode, open_mode FROM v$database;

-- Archive log mode
SELECT log_mode FROM v$database;

-- Current SCN
SELECT current_scn FROM v$database;
```

### Monitor Backup Status
```sql
-- Recent backups
SELECT session_key, input_type, status, 
       start_time, end_time, elapsed_seconds
FROM v$rman_backup_job_details
WHERE start_time > SYSDATE - 7;
```

### Check Recovery File Destinations
```sql
SELECT dest_id, status, target, destination, error
FROM v$archive_dest
WHERE status != 'INACTIVE';
```

---

**Note**: Always test solutions in a non-production environment first. Ensure you have valid backups before making any changes to production systems.
---
## Notes

- Always test recovery procedures in a non-production environment
- Verify backup integrity regularly using `RESTORE DATABASE VALIDATE`
- Monitor backup performance and adjust parallelism as needed
- Keep recovery catalogs backed up and maintained
- Document your backup and recovery procedures
- Test point-in-time recovery scenarios periodically

---

## Contributing

Feel free to contribute additional RMAN scripts and improvements to this collection.

## License

This collection is provided as-is for educational and operational purposes.
