# Oracle ASM Diagnostic and Operational Guide

## Table of Contents
- [Diagnostic SQL Queries](#diagnostic-sql-queries)
- [ASM Administration Scripts](#asm-administration-scripts)
- [Disk Management Operations](#disk-management-operations)
- [Performance Monitoring](#performance-monitoring)
- [Maintenance Operations](#maintenance-operations)
- [Troubleshooting](#troubleshooting)

## Diagnostic SQL Queries

### 1. Disk Group Space Usage

Monitor disk group capacity and utilization:

```sql
SELECT
    name AS disk_group,
    total_mb,
    free_mb,
    ROUND((total_mb - free_mb) / total_mb * 100, 2) AS pct_used,
    ROUND(free_mb / total_mb * 100, 2) AS pct_free,
    usable_file_mb,
    offline_disks,
    type AS redundancy,
    state
FROM v$asm_diskgroup;
```
```
--
-- display information on ASM
--
set wrap on
set lines 256
set pages 999
col "Group Name"   form a25
col "Disk Name"    form a15
col "State"  form a9
col "Type"   form a7
col "Total GB" form 999,999
col "Used GB"  form 999,999
col "Free GB"  form 999,999
col "Pct Used"  form 999.00

compute sum of "Total GB" on report
break on report

prompt
prompt ASM Disk Groups
prompt ===============
select group_number  "Group"
,      name          "Group Name"
,      state         "State"
,      type          "Type"
,      total_mb/1024 "Total GB"
,      (total_mb - free_mb)/1024 "Used GB"
,      free_mb/1024  "Free GB"
,      (total_mb - free_mb)/total_mb * 100 "Pct Used"
from   v$asm_diskgroup
/
```
```
--
-- display information on ASM
--
set wrap on
set lines 256
set pages 999
col "Group Name"   form a25
col "Disk Name"    form a15
col "State"  form a9
col "Type"   form a7
col "Total GB" form 999,999
col "Used GB"  form 999,999
col "Free GB"  form 999,999
col "Pct Used"  form 999.00
col "Instance"  form a10

prompt
prompt ASM Disks
prompt =========

col "Group"          form 999
col "Disk"           form 999
col "Header"         form a11
col "Mode"           form a8
col "Redundancy"     form a10
col "Failure Group"  form a14
col "Path"           form a30

select group_number  "Group"
,      disk_number   "Disk"
,      header_status "Header"
,      mode_status   "Mode"
,      state         "State"
,      redundancy    "Redundancy"
,      total_mb      "Total MB"
,      free_mb       "Free MB"
,      name          "Disk Name"
,      failgroup     "Failure Group"
,      path          "Path"
from   v$asm_disk
order by group_number
,        disk_number
/

prompt
prompt free ASM disks and their paths
prompt ==============================
select header_status , mode_status, path from V$asm_disk
where header_status in ('FORMER','CANDIDATE')
/

prompt
prompt Instances currently accessing these diskgroups
prompt ==============================================
col "Instance" form a8
select c.group_number  "Group"
,      g.name          "Group Name"
,      c.instance_name "Instance"
from   v$asm_client c
,      v$asm_diskgroup g
where  g.group_number=c.group_number
/

prompt
prompt Current ASM disk operations
prompt ===========================
select *
from   v$asm_operation
/

prompt
prompt ASM Disk Groups
prompt ===============
select group_number  "Group"
,      name          "Group Name"
,      state         "State"
,      type          "Type"
,      total_mb/1024 "Total GB"
,      (total_mb - free_mb)/1024 "Used GB"
,      free_mb/1024  "Free GB"
,      (total_mb - free_mb)/total_mb * 100 "Pct Used"
from   v$asm_diskgroup
/

```

### 2. Disk Group Layout and IO Statistics

View detailed disk information and I/O metrics:

```sql
SELECT
    d.group_number,
    g.name AS disk_group_name,
    d.disk_number,
    d.name AS disk_name,
    d.path,
    d.mount_status,
    d.state,
    d.total_mb,
    d.free_mb,
    d.reads,
    d.writes,
    d.read_errs,
    d.write_errs
FROM v$asm_disk d
JOIN v$asm_diskgroup g ON d.group_number = g.group_number
ORDER BY g.name, d.disk_number;
```

### 3. ASM File Details Per Disk Group

List all files within disk groups:

```sql
SELECT
    f.group_number,
    g.name AS disk_group_name,
    f.file_number,
    f.file_type,
    f.bytes / 1024 / 1024 AS size_mb,
    f.blocks,
    f.block_size,
    f.space,
    f.redundancy,
    f.database,
    f.name
FROM v$asm_file f
JOIN v$asm_diskgroup g ON f.group_number = g.group_number
ORDER BY g.name, f.file_type;
```
```
--
-- list v$asm_files by size - should run against ASM instance for speed. (db instance can be slow)
--
set echo off
set verify off
set heading on
set feedback off
set pagesize 1000
set linesize 600
set trimspool on

col gname format a8 heading "Group"
col mb_used format 99999999999999 heading "Mb Used"
col full_path format a80 heading Path
col creation_date format a9 heading Created
col modification_date format a9 heading Modified

break on gname skip 1 on report

compute sum of MB_USED on gname
compute sum of MB_USED on report

select p.gname
      ,p.full_path
      ,f.CREATION_DATE
      ,f.MODIFICATION_DATE
      ,f.space / 1048576 MB_USED
from v$asm_file f
,(select gname, group_number, file_number, concat ('+'||gname , sys_connect_by_path(aname,'/')) full_path
        from (select g.name gname
                   , a.group_number
                   , a.file_number
                   , a.parent_index pindex
                   , a.name aname
                   , a.reference_index rindex
                   , a.alias_directory adir
                from v$asm_alias a
                   , v$asm_diskgroup g
               where a.group_number = g.group_number)
       where adir='N'
       start with (MOD(pindex, power(2,24))) = 0
       connect by prior rindex = pindex
       order by full_path desc) p
where f.group_number = p.group_number
and f.file_number = p.file_number
order by p.gname,f.space
.
spool asm_files
/
spool off
```

### 4. Disk Mapping and Host Visibility

Check physical disk mapping:

```sql
SELECT
    d.name AS disk_name,
    d.path,
    d.mount_status,
    d.label,
    d.failgroup,
    d.library,
    d.os_mb,
    d.total_mb,
    d.free_mb,
    d.state
FROM v$asm_disk d
ORDER BY d.path;
```

### 5. ASM Performance Metrics

Monitor I/O performance:

```sql
SELECT
    name AS disk_name,
    reads,
    writes,
    read_errs,
    write_errs,
    read_time,
    write_time,
    bytes_read,
    bytes_written
FROM v$asm_disk
WHERE mount_status = 'CACHED'
ORDER BY bytes_written DESC;
```

### 6. Capacity Summary by Redundancy Type

Aggregate statistics by redundancy:

```sql
SELECT
    type AS redundancy,
    COUNT(*) AS diskgroup_count,
    SUM(total_mb) AS total_mb,
    SUM(free_mb) AS free_mb,
    ROUND(SUM((total_mb - free_mb)) / SUM(total_mb) * 100, 2) AS pct_used
FROM v$asm_diskgroup
GROUP BY type;
```

## ASM Administration Scripts

### ASM Directory Usage Script (asmdu.sh)

Monitor space usage within ASM directories:

```bash
#!/bin/bash
#
# ASM Directory Usage - Shows space usage of subdirectories
# Usage: ./asmdu.sh <ASM_DIRECTORY_PATH>
#
D=$1
 
if [[ -z $D ]]; then
    echo "Please provide a directory!"
    echo "Usage: $0 <ASM_DIRECTORY_PATH>"
    echo "Example: $0 +DATA/MYDB/DATAFILE"
    exit 1
fi
 
(for DIR in $(asmcmd ls ${D}); do
    echo ${DIR} $(asmcmd du ${D}/${DIR} | tail -1)
done) | awk -v D="$D" '
BEGIN {
    printf("\n\t\t%40s\n\n", D " subdirectories size");
    printf("%25s%16s%16s\n", "Subdir", "Used MB", "Mirror MB");
    printf("%25s%16s%16s\n", "------", "-------", "---------");
}
{
    printf("%25s%16s%16s\n", $1, $2, $3);
    use += $2;
    mir += $3;
}
END {
    printf("\n\n%25s%16s%16s\n", "------", "-------", "---------");
    printf("%25s%16s%16s\n\n", "Total", use, mir);
}'
```

**Usage Examples:**
```bash
# Set Oracle environment
. oraenv
+ASM1

# Check DATA disk group usage
./asmdu.sh +DATA

# Check archivelog directory usage
./asmdu.sh +FRA/MYDB/ARCHIVELOG/
```

## Disk Management Operations

### Adding Disks to ASM Disk Groups

#### Step 1: Check Current Configuration

```sql
SET LINES 255 PAGESIZE 3000
COL path FOR A35
COL Diskgroup FOR A15
COL DiskName FOR A20
COL disk# FOR 999
COL total_mb FOR 999,999,999
COL free_mb FOR 999,999,999

SELECT 
    a.name DiskGroup, 
    b.disk_number Disk#, 
    b.name DiskName, 
    b.total_mb, 
    b.free_mb, 
    b.path, 
    b.header_status
FROM v$asm_disk b, v$asm_diskgroup a
WHERE a.group_number(+) = b.group_number
  AND a.name LIKE '%DATA%'
ORDER BY b.group_number, b.disk_number, b.name;
```

#### Step 2: Prepare Physical Disks

Check if disks are already configured:
```bash
# Check disk status
sudo /usr/sbin/oracleasm querydisk /dev/sdc1
sudo /usr/sbin/oracleasm querydisk /dev/sdd1

# Create ASM disks if not configured
sudo /usr/sbin/oracleasm createdisk DISK01 /dev/sdc1
sudo /usr/sbin/oracleasm createdisk DISK02 /dev/sdd1

# Scan for new disks on all cluster nodes
sudo /usr/sbin/oracleasm scandisks

# Verify disk creation
sudo /usr/sbin/oracleasm listdisks
```

#### Step 3: Add Disks to Disk Group

```sql
-- Add disks with rebalancing
ALTER DISKGROUP DATA ADD DISK 'ORCL:DISK01' REBALANCE POWER 4;
ALTER DISKGROUP DATA ADD DISK 'ORCL:DISK02' REBALANCE POWER 4;

-- Check rebalance progress
SELECT group_number, operation, state, power, est_minutes 
FROM v$asm_operation;
```

### Disk Online Operations

```sql
-- Bring all disks online
ALTER DISKGROUP DATA ONLINE ALL;

-- Bring specific disk online
ALTER DISKGROUP DATA ONLINE DISK 'ORCL:DISK01';
```

## Performance Monitoring

### Database Throughput by ASM

```sql
SELECT * FROM (
    SELECT 
        a.DBNAME as "DB Name",
        SUM(a.READS) as "Read Request",
        SUM(a.WRITES) as "Write Requests",
        SUM(a.BYTES_READ/1024/1024) as "Total Read MB",
        SUM(a.BYTES_WRITTEN/1024/1024) "Total Write MB",
        DECODE(SUM(a.BYTES_WRITTEN/1024/1024), 0, 0,
               SUM(a.BYTES_READ/1024/1024)/SUM(a.BYTES_WRITTEN/1024/1024)) "Throughput"
    FROM v$asm_disk_iostat a, v$asm_disk b
    WHERE a.GROUP_NUMBER = b.GROUP_NUMBER
      AND a.DISK_NUMBER = b.DISK_NUMBER
    GROUP BY a.DBNAME
    ORDER BY 4 DESC
)
WHERE ROWNUM <= 10;
```

### Archivelog Growth Analysis

```sql
-- Daily archivelog generation
SELECT 
    TRUNC(COMPLETION_TIME) TIME, 
    SUM(BLOCKS * BLOCK_SIZE)/1024/1024 SIZE_MB 
FROM V$ARCHIVED_LOG 
GROUP BY TRUNC(COMPLETION_TIME) 
ORDER BY 1;

-- Hourly archivelog generation
ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24';

SELECT 
    TRUNC(COMPLETION_TIME,'HH24') TIME, 
    SUM(BLOCKS * BLOCK_SIZE)/1024/1024 SIZE_MB 
FROM V$ARCHIVED_LOG 
GROUP BY TRUNC(COMPLETION_TIME,'HH24') 
ORDER BY 1;
```

## Maintenance Operations

### ASM File Path Resolution

Get full paths for ASM files:

```sql
SELECT 
    CONCAT('+' || db_files.disk_group_name, SYS_CONNECT_BY_PATH(db_files.alias_name, '/')) full_path,
    db_files.bytes,
    db_files.space,
    NVL(LPAD(db_files.type, 18), '<DIRECTORY>') type,
    db_files.creation_date,
    db_files.disk_group_name,
    LPAD(db_files.system_created, 4) system_created
FROM (
    SELECT 
        g.name disk_group_name,
        a.parent_index pindex,
        a.name alias_name,
        a.reference_index rindex,
        a.system_created system_created,
        f.bytes bytes,
        f.space space,
        f.type type,
        TO_CHAR(f.creation_date, 'DD-MON-YYYY HH24:MI:SS') creation_date
    FROM v$asm_file f 
    RIGHT OUTER JOIN v$asm_alias a USING (group_number, file_number)
    JOIN v$asm_diskgroup g USING (group_number)
) db_files
WHERE db_files.type IS NOT NULL
START WITH (MOD(db_files.pindex, POWER(2, 24))) = 0
CONNECT BY PRIOR db_files.rindex = db_files.pindex;
```

### Rebalance Operations

```sql
-- Check current operations
SELECT group_number, operation, state, power, est_minutes 
FROM v$asm_operation;

-- Adjust rebalance power
ALTER DISKGROUP DATA REBALANCE POWER 8;

-- Check rebalance tolerance
ALTER SYSTEM SET "_asm_imbalance_tolerance"=0;
```

### Disk Group Status Checks

```sql
-- Comprehensive disk group information
SET LINESIZE 145 PAGESIZE 9999 VERIFY OFF
COLUMN group_name FORMAT A20 HEAD 'Disk Group|Name'
COLUMN state FORMAT A11 HEAD 'State'
COLUMN type FORMAT A6 HEAD 'Type'
COLUMN total_mb FORMAT 999,999,999 HEAD 'Total Size (MB)'
COLUMN used_mb FORMAT 999,999,999 HEAD 'Used Size (MB)'
COLUMN pct_used FORMAT 999.99 HEAD 'Pct. Used'

SELECT
    name group_name,
    state,
    type,
    total_mb,
    (total_mb - free_mb) used_mb,
    ROUND((1- (free_mb / total_mb))*100, 2) pct_used
FROM v$asm_diskgroup
ORDER BY name;
```

## Troubleshooting

### Orphaned Files Detection and Cleanup

Find orphaned files in ASM disk groups:

```sql
-- Replace &ASMGROUP with your disk group name
DEFINE ASMGROUP="DATA"

SELECT 'rm '|| files files FROM (
    SELECT '+&ASMGROUP'||files files, type 
    FROM (
        SELECT 
            UPPER(SYS_CONNECT_BY_PATH(aa.name,'/')) files, 
            aa.reference_index, 
            b.type
        FROM (
            SELECT file_number, alias_directory, name, reference_index, parent_index 
            FROM v$asm_alias
        ) aa,
        (SELECT parent_index 
         FROM v$asm_alias 
         WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name='&ASMGROUP') 
           AND alias_index=0
        ) a,
        (SELECT file_number, type 
         FROM v$asm_file 
         WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name='&ASMGROUP')
        ) b
        WHERE aa.file_number=b.file_number(+)
          AND aa.alias_directory='N'
          AND b.type IN ('DATAFILE','ONLINELOG','CONTROLFILE','TEMPFILE')
        START WITH aa.PARENT_INDEX=a.parent_index
        CONNECT BY PRIOR aa.reference_index=aa.parent_index
    )
    WHERE SUBSTR(files,INSTR(files,'/',1,1),INSTR(files,'/',1,2)-INSTR(files,'/',1,1)+1) = 
          (SELECT '/'||UPPER(db_unique_name)||'/' FROM v$database)
    MINUS (
        SELECT UPPER(name) files, 'DATAFILE' type FROM v$datafile
        UNION ALL 
        SELECT UPPER(name) files, 'TEMPFILE' type FROM v$tempfile
        UNION ALL
        SELECT UPPER(name) files, 'CONTROLFILE' type FROM v$controlfile WHERE name LIKE '+&ASMGROUP%'
        UNION ALL
        SELECT UPPER(member) files, 'ONLINELOG' type FROM v$logfile WHERE member LIKE '+&ASMGROUP%'
    )
);
```

### ASM Client Information

Check which databases are using ASM disk groups:

```sql
SELECT 
    c.group_number "Group",
    g.name "Group Name",
    c.instance_name "Instance"
FROM v$asm_client c, v$asm_diskgroup g
WHERE g.group_number = c.group_number;
```

### Disk Mapping Verification

Script to map ASM disks to physical devices:

```bash
#!/bin/bash
# chk_asm_mapping.sh - Map ASM disks to physical devices

/etc/init.d/oracleasm querydisk -d $(/etc/init.d/oracleasm listdisks -d) |
cut -f2,10,11 -d" " | perl -pe 's/"(.*)".*\[(.*), *(.*)\]/$1 $2 $3/g;' |
while read v_asmdisk v_minor v_major; do
    v_device=$(ls -la /dev | grep " $v_minor, *$v_major " | awk '{print $10}')
    echo "ASM disk $v_asmdisk based on /dev/$v_device [$v_minor, $v_major]"
done
```

### Hidden ASM Parameters

View ASM-specific hidden parameters:

```sql
SELECT 
    a.ksppinm "Parameter Name", 
    c.ksppstvl "Value"
FROM x$ksppi a, x$ksppcv b, x$ksppsv c
WHERE a.indx = b.indx 
  AND a.indx = c.indx
  AND ksppinm LIKE '_asm%'
ORDER BY a.ksppinm;
```

---

## Notes

- Always test scripts in a non-production environment first
- Ensure proper backups before performing disk operations
- Monitor rebalance operations to avoid performance impact
- Use appropriate rebalance power settings based on system load
- Regular monitoring of disk group space usage is recommended

## Contributing

When contributing to this documentation:
1. Test all SQL queries and scripts
2. Include usage examples where appropriate
3. Document any prerequisites or restrictions
4. Follow the existing formatting conventions
