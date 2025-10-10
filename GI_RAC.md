# Oracle Grid Infrastructure & RAC Administration Guide

A comprehensive operational guide for DBAs and Grid Infrastructure administrators managing Oracle Real Application Clusters (RAC) and Automatic Storage Management (ASM) environments.

**Target Audience:** Database Administrators, System Administrators, Grid Infrastructure Administrators  
**Oracle Versions:** 11gR2, 12c, 19c, 21c

---

## Table of Contents

1. [Housekeeping & Log Management](#1-housekeeping--log-management)
2. [Cluster Management](#2-cluster-management)
3. [ASM Administration](#3-asm-administration)
4. [ACFS Management](#4-acfs-management)
5. [RAC Monitoring](#5-rac-monitoring)
6. [Database Operations](#6-database-operations)
7. [Network & Services](#7-network--services)
8. [Server Pools & Policy Management](#8-server-pools--policy-management)
9. [RAC One Node](#9-rac-one-node)
10. [Backup & Recovery](#10-backup--recovery)
11. [Troubleshooting](#11-troubleshooting)
12. [Quick Reference](#12-quick-reference)

---

## 1. Housekeeping & Log Management

### Clean Old Logs

```bash
# Check disk usage
du -a | sort -n | tail -15

# Clean archive logs (older than 3 days)
find /opt/oraarch/PLANNING/1_3*.dbf -mtime +3 -exec rm {} \;

# Clean Grid audit files (older than 130 days)
find $GRID_HOME/rdbms/audit/*.aud* -mtime +130 -exec rm {} \;

# Clean database trace files (older than 30 days)
find $ORACLE_HOME/admin/pprod25/bdump -name "*.trc" -mtime +30 -exec rm -rf {} \;
find $ORACLE_HOME/admin/pprod25/udump -name "*.trc" -mtime +30 -exec rm -rf {} \;
find $ORACLE_HOME/admin/pprod25/adump -name "*.aud" -mtime +30 -exec rm -rf {} \;

# Clean diagnostic trace files
find $ORACLE_BASE/diag/rdbms/wspsp/wspsp3/trace -name "*.trc" -mtime +30 -exec rm -rf {} \;
```

### Listener Logs

```bash
# Check listener log sizes
cd $GRID_HOME/log/diag/tnslsnr/<clustername>
du -sh *

# Example output:
# 332K    listener_scan1
# 1.6G    listener_scan2
# 332K    listener_scan3

# Clean listener logs (older than 20 days)
find /opt/oracle/product/diag/tnslsnr/<clustername>/listener/alert/*.xml* -mtime +20 -exec rm {} \;
```

---

## 2. Cluster Management

### Cluster Verification

```bash
# List cluster nodes
olsnodes
olsnodes -n -p -i    # Public, private, and virtual names

# Check cluster status
crsctl check cluster -all
crsctl status resource -t

# Check node applications
srvctl status nodeapps
crs_stat -t | grep vip
```

### Voting Disk & OCR

```bash
# Query voting disks
crsctl query css votedisk

# Check OCR
ocrcheck
ocrconfig -showbackup
ocrconfig -manualbackup
ocrconfig -export /home/oracle/ocr.backup
```

### Add Node to RAC Cluster

#### Pre-Checks

```bash
# Verify SSH equivalence
ssh rac1 date
ssh rac3 date

# Run cluster verification
cluvfy stage -pre nodeadd -n rac3 -verbose
```

#### Add Grid Infrastructure

```bash
# From existing node
cd $GRID_HOME/oui/bin
./addNode.sh -silent \
  "CLUSTER_NEW_NODES={rac3}" \
  "CLUSTER_NEW_VIRTUAL_HOSTNAMES={rac3-vip}"

# Run root script on new node
ssh root@rac3 /u01/app/11.2.0/grid/root.sh

# Verify
cluvfy stage -post nodeadd -n rac3 -verbose
crsctl stat res -t
olsnodes -n
```

#### Add Database Home

```bash
# Add Oracle Home
$ORACLE_HOME/oui/bin/addNode.sh -silent "CLUSTER_NEW_NODES={rac3}"

# Run root script
$ORACLE_HOME/root.sh

# Add database instance
dbca -silent -addInstance \
  -nodeList rac3 \
  -gdbName ORCL \
  -instanceName ORCL3 \
  -sysDBAUserName sys \
  -sysDBAPassword <password>
```

### Remove Node from RAC Cluster

```bash
# Pre-checks
olsnodes -s -t
srvctl status nodeapps -n rac3

# Deconfigure node
ssh rac3
$GRID_HOME/crs/install/rootcrs.pl -deconfig -force

# Remove VIP (if needed)
srvctl stop vip -i rac3-vip -f
srvctl remove vip -i rac3-vip -f

# Delete node from cluster
crsctl delete node -n rac3

# Update inventory
cd $GRID_HOME/oui/bin
./runInstaller -updateNodeList ORACLE_HOME=$GRID_HOME \
  "CLUSTER_NODES={rac3}" CRS=TRUE -local

# Verify removal
olsnodes -t -s
crsctl stat res -t
```

### Network Configuration

```bash
# Show network interfaces
oifcfg iflist
oifcfg getif

# Configure SCAN listeners
srvctl config scan_listener
srvctl status scan
srvctl relocate scan -i 3 -n <node>
```

### Timeout Parameters

```bash
# Check current values
crsctl get css disktimeout   # default: 200
crsctl get css misscount     # default: 60 (Linux)

# Modify (example)
crsctl set css misscount 100
```

---

## 3. ASM Administration

### Diskgroup Status

```sql
-- Diskgroup usage summary
SELECT name, 
       total_mb/1024 total_gb,
       (total_mb - free_mb)/1024 used_gb,
       free_mb/1024 free_gb,
       ROUND((total_mb - free_mb)/total_mb * 100, 2) pct_used
FROM v$asm_diskgroup
ORDER BY name;

-- Disk details
SELECT group_number, disk_number, 
       header_status, mode_status, state,
       total_mb, free_mb, name, path
FROM v$asm_disk
ORDER BY group_number, disk_number;

-- Free/candidate disks
SELECT header_status, mode_status, path
FROM v$asm_disk
WHERE header_status IN ('FORMER','CANDIDATE');
```

### Add Disks to Diskgroup

```sql
-- Add disk with rebalance
ALTER DISKGROUP PPROD24_DATA_01 
ADD DISK 'ORCL:ASMPPROD24DATA36' 
REBALANCE POWER 1;

-- Check rebalance progress
SELECT group_number, operation, state, power, est_minutes
FROM v$asm_operation;
```

### ASM File Management

```sql
-- Generate all ASM files for a database
COLUMN full_alias_path FORMAT a75
SELECT CONCAT('+'||gname, SYS_CONNECT_BY_PATH(aname, '/')) full_alias_path,
       system_created, alias_directory, file_type
FROM (
  SELECT b.name gname, a.parent_index pindex, a.name aname,
         a.reference_index rindex, a.system_created, a.alias_directory,
         c.type file_type
  FROM v$asm_alias a, v$asm_diskgroup b, v$asm_file c
  WHERE a.group_number = b.group_number
  AND a.group_number = c.group_number(+)
  AND a.file_number = c.file_number(+)
  AND a.file_incarnation = c.incarnation(+)
)
START WITH MOD(pindex, POWER(2,24))=0
  AND rindex IN (SELECT a.reference_index
                 FROM v$asm_alias a, v$asm_diskgroup b
                 WHERE a.group_number=b.group_number
                 AND MOD(a.parent_index,POWER(2,24))=0
                 AND a.name='&DATABASENAME')
CONNECT BY PRIOR rindex=pindex;
```

### Detect & Remove Orphan Files

```sql
-- Find orphan ASM files
SELECT f.group_number, f.file_number, f.incarnation,
       f.compound_index, f.type, f.bytes, f.space
FROM v$asm_file f
WHERE f.type IN ('DATAFILE','TEMPFILE')
AND NOT EXISTS (
  SELECT 1 FROM v$asm_alias a
  WHERE a.group_number = f.group_number
  AND a.file_number = f.file_number
  AND a.file_incarnation = f.incarnation
);
```

```bash
# Remove orphan files using ASMCMD
asmcmd rm +DATA/ORCL/orphan_file.256.123456
```

### Move Datafile Between Diskgroups

```sql
-- Method 1: SQL (offline required)
ALTER TABLESPACE users OFFLINE;

-- Copy file (from RMAN)
BACKUP AS COPY DATAFILE '/u01/oradata/orcl/users01.dbf' 
FORMAT '+NEWDG';

-- Rename
ALTER DATABASE RENAME FILE 
'/u01/oradata/orcl/users01.dbf' TO '+NEWDG/orcl/datafile/users.256.1';

-- Recover and online
RECOVER DATAFILE '+NEWDG/orcl/datafile/users.256.1';
ALTER TABLESPACE users ONLINE;

-- Verify
SELECT file_name FROM dba_data_files WHERE tablespace_name='USERS';
```

### Capacity Planning

```sql
-- Disk imbalance check
SELECT dg.name, dk.os_mb "Disk Size MB", 
       COUNT(*) "NR_Disks",
       TRUNC(COUNT(*)/100*30) "DisksToRequest"
FROM v$asm_diskgroup dg, v$asm_disk dk
WHERE dg.group_number = dk.group_number
AND dg.name = UPPER('&diskgroup_name')
GROUP BY dg.name, dk.os_mb;

-- Archive log growth by day
SELECT TRUNC(completion_time) time,
       SUM(blocks*block_size)/1024/1024 size_mb
FROM v$archived_log
GROUP BY TRUNC(completion_time)
ORDER BY 1;
```

### ASMSNMP User

```sql
-- Create ASMSNMP user (if missing)
CREATE USER asmsnmp IDENTIFIED BY <password>;
GRANT sysdba TO asmsnmp;
```

```bash
# Change ASMSNMP password
ASMCMD> orapwusr --modify --password ASMSNMP
```

---

## 4. ACFS Management

### Create ACFS Volume

```bash
# Create ASM volume
ASMCMD> volcreate -G DG_FLASH -s 1G advm_vg

# Check volume info
ASMCMD> volinfo -G DG_FLASH advm_vg

# Create ACFS filesystem
mkfs -t acfs -b 4k /dev/asm/advm_vg-86

# Register and mount
/sbin/acfsutil registry -a -f -n rac1,rac2 \
  /dev/asm/advm_vg-86 /u01/app/oracle/acfsmounts/db_home

mount -t acfs /dev/asm/advm_vg-86 /u01/app/oracle/acfsmounts/db_home
```

### Resize ACFS

```bash
# Increase size
/sbin/acfsutil size +512M /u01/app/oracle/acfsmounts/db_home
```

### Remove ACFS

```bash
# Unmount
umount /u01/app/oracle/acfsmounts/db_home

# Unregister
/sbin/acfsutil registry -d /dev/asm/advm_vg-86

# Drop volume
ASMCMD> voldrop -G DG_FLASH advm_vg
```

### ACFS Snapshots

```bash
# Create snapshot
/sbin/acfsutil snap create snap1 /u01/app/oracle/acfsmounts/db_home

# List snapshots
/sbin/acfsutil snap info /u01/app/oracle/acfsmounts/db_home
```

---

## 5. RAC Monitoring

### Long Operations

```sql
SELECT s.inst_id, s.sid, s.serial#, s.username, s.module,
       ROUND(sl.elapsed_seconds/60) || ':' || MOD(sl.elapsed_seconds,60) elapsed,
       ROUND(sl.time_remaining/60) || ':' || MOD(sl.time_remaining,60) remaining,
       ROUND(sl.sofar/(sl.totalwork+1)*100, 2) progress_pct
FROM gv$session s, gv$session_longops sl
WHERE s.sid = sl.sid
AND s.inst_id = sl.inst_id
AND s.serial# = sl.serial#
ORDER BY progress_pct;
```
### GC Blocks Lost and Waits

```sql
SELECT inst_id,
       name,
       value
FROM gv$sysstat
WHERE name LIKE 'gc%lost%';
```

### Interconnect Latency

```sql
SELECT inst_id,
       name,
       value
FROM gv$sysstat
WHERE name LIKE 'gc%time%';
```

### RAC Global Cache Statistics

```sql
SELECT inst_id,
       name,
       value
FROM gv$sysstat
WHERE name LIKE 'gc%'
ORDER BY inst_id, name;
```
### Session Waits

```sql
SELECT s.inst_id, NVL(s.username, '(oracle)') AS username,
       s.sid, s.serial#, sw.event, sw.wait_class,
       sw.wait_time, sw.seconds_in_wait, sw.state
FROM gv$session_wait sw, gv$session s
WHERE s.sid = sw.sid
AND s.inst_id = sw.inst_id
ORDER BY sw.seconds_in_wait DESC;
```

### Memory Allocation per Session

```sql
SELECT a.inst_id, NVL(a.username,'(oracle)') AS username,
       a.module, a.program, Trunc(b.value/1024) AS memory_kb
FROM gv$session a, gv$sesstat b, gv$statname c
WHERE a.sid = b.sid
AND a.inst_id = b.inst_id
AND b.statistic# = c.statistic#
AND b.inst_id = c.inst_id
AND c.name = 'session pga memory'
AND a.program IS NOT NULL
ORDER BY b.value DESC;
```

### List All Sessions

```sql
SELECT NVL(s.username, '(oracle)') AS username,
       s.inst_id, s.sid, s.serial#, p.spid,
       s.lockwait, s.status, s.module, s.machine, s.program,
       TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time
FROM gv$session s, gv$process p
WHERE s.paddr = p.addr
AND s.inst_id = p.inst_id
ORDER BY s.username, s.osuser;
```

### Temp Space Monitoring

```sql
-- Temp segment usage
SELECT inst_id, tablespace_name, segment_file, 
       total_blocks, used_blocks, free_blocks
FROM gv$sort_segment;

-- Temp extent pool usage
SELECT inst_id, tablespace_name, blocks_cached, blocks_used
FROM gv$temp_extent_pool;

-- Temp space header
SELECT inst_id, tablespace_name, blocks_used, blocks_free
FROM gv$temp_space_header;
```

---

## 6. Database Operations

### RAC to Single Instance Conversion

```bash
# Stop database and CRS
srvctl stop database -d RACDB
crsctl stop crs

# Relink Oracle without RAC
cd $ORACLE_HOME/rdbms/lib
make -f ins_rdbms.mk rac_off
make -f ins_rdbms.mk ioracle

# If using ASM
$ORACLE_HOME/bin/localconfig delete
$ORACLE_HOME/bin/localconfig add
```

```sql
-- Disable additional threads
ALTER DATABASE DISABLE THREAD 2;

-- Update parameters
ALTER SYSTEM SET remote_listener='' SCOPE=SPFILE;
ALTER SYSTEM SET cluster_database=FALSE SCOPE=SPFILE;

-- Drop extra undo tablespace
DROP TABLESPACE UNDOTBS2 INCLUDING CONTENTS AND DATAFILES;
```

```bash
# Remove CRS (if permanently converting)
$ORA_CRS_HOME/install/rootdelete.sh
rm -rf $ORA_CRS_HOME
```

### Single Instance to RAC Conversion

#### Prerequisites
1. Move storage to shared storage (ASM or cluster filesystem)
2. Create additional redo log groups with THREAD specification
3. Create additional UNDO tablespace

```sql
-- Enable cluster database
ALTER SYSTEM SET cluster_database=TRUE SCOPE=SPFILE;
ALTER SYSTEM SET cluster_database_instances=2 SCOPE=SPFILE;

-- Configure instance-specific parameters
ALTER SYSTEM SET ORCL1.THREAD=1 SCOPE=SPFILE;
ALTER SYSTEM SET ORCL2.THREAD=2 SCOPE=SPFILE;
ALTER SYSTEM SET ORCL1.UNDO_TABLESPACE='UNDOTBS1' SCOPE=SPFILE;
ALTER SYSTEM SET ORCL2.UNDO_TABLESPACE='UNDOTBS2' SCOPE=SPFILE;

-- Enable thread 2
ALTER DATABASE ENABLE THREAD 2;
```

```bash
# Register with CRS
srvctl add database -d ORCL -o $ORACLE_HOME
srvctl add instance -d ORCL -i ORCL1 -n node1
srvctl add instance -d ORCL -i ORCL2 -n node2
```

### ASM to Non-ASM Migration

```bash
# Create PFILE from SPFILE
SQL> CREATE PFILE='/tmp/initDB.ora' FROM SPFILE='+DATA/DB/spfileDB.ora';

# Edit PFILE - change ASM paths to filesystem
# control_files, db_recovery_file_dest, audit_file_dest, etc.

# Startup with new PFILE
SQL> STARTUP NOMOUNT PFILE='/tmp/initDB.ora';
SQL> ALTER DATABASE MOUNT;
```

```sql
-- Use RMAN to copy database
RMAN> BACKUP AS COPY DATABASE FORMAT '/u01/oradata/%U';
RMAN> SWITCH DATABASE TO COPY;
RMAN> ALTER DATABASE OPEN;
```

### Non-ASM to ASM Migration

```bash
# Configure ASM diskgroups
ASMCMD> lsdg
```

```sql
-- Update parameters
ALTER SYSTEM SET db_recovery_file_dest='+FRA' SCOPE=SPFILE;
ALTER SYSTEM SET control_files='+DATA','+FRA' SCOPE=SPFILE;

-- Shutdown and restore controlfile
SHUTDOWN IMMEDIATE;
STARTUP NOMOUNT;
RESTORE CONTROLFILE TO '+DATA' FROM '/u01/oradata/control01.ctl';
ALTER DATABASE MOUNT;
```

```sql
-- Copy database to ASM
RMAN> BACKUP AS COPY DATABASE FORMAT '+DATA';
RMAN> SWITCH DATABASE TO COPY;
RMAN> ALTER DATABASE OPEN;
```

### Delete Database

```bash
# Create staging directory
mkdir /tmp/$DBUNNAME

# Stop and remove from CRS
srvctl stop database -d $DBUNNAME
srvctl remove database -d $DBUNNAME -noprompt

# Remove ASM files
asmcmd rm -rf +DATA/$DBUNNAME
asmcmd rm -rf +FRA/$DBUNNAME

# Remove local files
find /u01/ -type f -iname "*$DBNAME*" -exec rm -f {} \;
find /u01/ -type d -iname "*$DBNAME*" -exec rm -rf {} \;

# Remove from /etc/oratab
vi /etc/oratab
```

---

## 7. Network & Services

### SCAN Configuration

```bash
# Check SCAN configuration
srvctl config scan
srvctl status scan
srvctl status scan_listener

# Relocate SCAN listener
srvctl relocate scan -i 3 -n node2
srvctl start scan_listener -i 3

# Verify from client
nslookup <SCAN_NAME>
tnsping <SERVICE_NAME>
```

### Service Management (Admin-Managed)

```bash
# Add service with preferred/available instances
srvctl add service -d ORCL -s PROD \
  -r ORCL1 -a ORCL2 \
  -P BASIC -y AUTOMATIC

# Start service
srvctl start service -d ORCL -s PROD

# Check service status
srvctl status service -d ORCL -s PROD
```

```sql
-- Verify service registration
SELECT inst_id, name FROM gv$services ORDER BY inst_id, name;
```

### Service Options

```bash
# Connection Load Balancing
srvctl modify service -d ORCL -s PROD -j SHORT

# Enable DTP (Distributed Transactions)
srvctl modify service -d ORCL -s PROD -x TRUE

# TAF Configuration
srvctl modify service -d ORCL -s PROD -P BASIC

# Management policy
srvctl modify service -d ORCL -s PROD -y MANUAL
```

---

## 8. Server Pools & Policy Management

### Create Server Pool

```bash
# Create server pool
srvctl add srvpool -g mypool -l 2 -u 2 -i 1

# Check pool status
srvctl status srvpool -g mypool
crsctl status serverpool -a
```

### Convert Admin to Policy-Managed (Online)

```bash
# Check current configuration
srvctl config database -d ORCL

# Create server pool
srvctl add srvpool -g prod01 -l 2 -u 2

# Assign database to server pool
srvctl modify database -d ORCL -g prod01 -f

# Verify
srvctl config database -d ORCL
```

### Convert Policy to Admin-Managed (Downtime Required)

```bash
# Stop database
srvctl stop database -d ORCL

# Remove from CRS
srvctl remove database -d ORCL

# Re-add as admin-managed
srvctl add database -d ORCL -o $ORACLE_HOME -y automatic
srvctl add instance -d ORCL -i ORCL1 -n node1
srvctl add instance -d ORCL -i ORCL2 -n node2

# Re-add services
srvctl add service -d ORCL -s PROD -r ORCL1 -a ORCL2
```

### Service Management (Policy-Managed)

```bash
# Singleton service (runs on one instance in pool)
srvctl add service -d ORCL -s PROD1 -g prod01 -c singleton -y manual

# Uniform service (runs on all instances in pool)
srvctl modify service -d ORCL -s PROD1 -g prod01 -c uniform -y automatic
```

---

## 9. RAC One Node

### What is RAC One Node?

Single-instance RAC database running on one node in a cluster with:
- Online instance relocation capability
- Faster failover than single instance
- Can scale up to full RAC

### Relocate Instance

```bash
# Relocate to different node
srvctl relocate database -d DBNAME -n node2 -v

# Check status
srvctl status database -d DBNAME
```

### Scale Up to Full RAC

```bash
# Convert to RAC
srvctl convert database -d DBNAME -c RAC

# Add second instance
srvctl add instance -d DBNAME -i DBNAME2 -n node2
srvctl start instance -d DBNAME -i DBNAME2

# Verify
srvctl status database -d DBNAME
```

### Scale Down to RAC One Node

```bash
# Remove services from second instance
srvctl stop service -d DBNAME -s SERVICE
srvctl remove service -d DBNAME -s SERVICE

# Stop and remove second instance
srvctl stop instance -d DBNAME -i DBNAME2 -o immediate
srvctl remove instance -d DBNAME -i DBNAME2

# Convert back
srvctl convert database -d DBNAME -c RACONENODE -i DBNAME1
```

### Password File Management

```bash
# Create password file in ASM (RAC)
orapwd FILE='+DATA/DBNAME/orapwDBNAME' \
  ENTRIES=10 \
  DBUNIQUENAME='DBNAME' \
  FORMAT=12

# Update CRS
srvctl modify database -d DBNAME -pwfile +DATA/DBNAME/orapwDBNAME
```

---

## 10. Backup & Recovery

### OCR Backup

```bash
# Check OCR backup status
ocrconfig -showbackup

# Manual OCR backup
ocrconfig -manualbackup

# Export OCR
ocrconfig -export /home/oracle/ocr_backup_$(date +%Y%m%d).dmp

# Verify OCR
ocrcheck
```

### Voting Disk Management

```bash
# Query voting disks
crsctl query css votedisk

# Add voting disk
crsctl add css votedisk <path>

# Remove voting disk
crsctl delete css votedisk <path>
```

### Convert to NOARCHIVELOG Mode

```bash
# Check current status
srvctl status database -d ORCL
```

```sql
-- Disable cluster temporarily
ALTER SYSTEM SET cluster_database=FALSE SCOPE=SPFILE;
```

```bash
# Shutdown database
srvctl stop database -d ORCL
```

```sql
-- Mount and disable archivelog
STARTUP MOUNT;
ALTER DATABASE NOARCHIVELOG;

-- Re-enable cluster
ALTER SYSTEM SET cluster_database=TRUE SCOPE=SPFILE;
```

```bash
# Restart database
srvctl start database -d ORCL
```

```sql
-- Verify
ARCHIVE LOG LIST;
```

---

## 11. Troubleshooting

### Common Startup Errors

#### Instance Fails with PRKP-1001 or CRS-0215

```bash
# Copy network configs to Oracle Home
cp $TNS_ADMIN/listener.ora $ORACLE_HOME/network/admin/
cp $TNS_ADMIN/tnsnames.ora $ORACLE_HOME/network/admin/
```

#### Database Doesn't Autostart

```bash
# Check auto-start policy
crsctl status res ora.orcl.db -p | grep AUTO_START

# Fix auto-start
crsctl modify res ora.orcl.db -attr "AUTO_START=always"
srvctl modify database -d orcl -y automatic
```

#### ASM Diskgroup Mount Failure (ORA-15032)

```bash
# Check ASM status
/etc/init.d/oracleasm status
/etc/init.d/oracleasm listdisks

# Restart ASM library
/etc/init.d/oracleasm restart
/etc/init.d/oracleasm enable
```

#### Service Fails with Global Transaction Error

```bash
# Enable DTP for service
srvctl modify service -d ORCL -s SERVICE -x TRUE

# Or disable cluster-wide transactions
```

```sql
ALTER SYSTEM SET "_clusterwide_global_transactions"=FALSE SCOPE=SPFILE;
```

### High gc block lost (Interconnect Issues)

```sql
-- Check lost block statistics
SELECT inst_id, name, value
FROM gv$sysstat
WHERE name LIKE 'gc%block lost'
ORDER BY inst_id;
```

```bash
# Check network statistics
netstat -s | egrep -i "fragments|reassembl|dropped"

# Check NIC errors
ethtool -S eth1
ifconfig eth1

# Disable offloads (for testing)
ethtool -K eth1 gro off gso off tso off

# Check bonding and IRQ
cat /proc/interrupts
cat /proc/net/bonding/bond0
```

### High log file sync Waits

```sql
-- Check redo write performance
SELECT name, value 
FROM v$sysstat 
WHERE name LIKE 'redo write%';

-- Check commit rate
SELECT inst_id, value 
FROM gv$sysstat 
WHERE name IN ('user commits','user rollbacks');
```

```bash
# Check IO latency
iostat -x 1 10

# Check LGWR process
ps -ef | grep lgwr
top -p <lgwr_pid>
```

### Diagnostic Data Collection

```bash
# Collect cluster diagnostics
$GRID_HOME/bin/diagcollection.pl --collect --all --crshome $GRID_HOME

# OSWatcher (recommended by Oracle Support)
# Download and run OSWatcher for 30-60 minutes during issue

# Packet capture (short duration)
tcpdump -i eth1 -s 0 -w /tmp/interconnect.cap -W 5 -C 100
```

### Log Locations

```bash
# CRS logs
$GRID_HOME/log/<hostname>/crsd/
$GRID_HOME/log/<hostname>/cssd/
$GRID_HOME/log/<hostname>/evmd/

# ASM diagnostics
$GRID_BASE/diag/asm/+asm/+ASM1/trace/

# Cluster alert log
$GRID_HOME/log/<hostname>/alert<hostname>.log

# Database alert log
$ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance>/trace/alert_<instance>.log
```

---

## 12. Quick Reference

### Essential Commands

```bash
# Cluster Status
crsctl check cluster -all
crsctl status resource -t
olsnodes -n

# Database Operations
srvctl status database -d ORCL
srvctl start database -d ORCL
srvctl stop database -d ORCL
srvctl config database -d ORCL

# ASM Operations
asmcmd lsdg
asmcmd lsop
crsctl status resource ora.DATA.dg

# Service Management
srvctl add service -d ORCL -s PROD -r ORCL1 -a ORCL2
srvctl start service -d ORCL -s PROD
srvctl status service -d ORCL

# Node Applications
srvctl status nodeapps
srvctl status scan
srvctl relocate scan -i 1 -n node2

# Voting Disk & OCR
crsctl query css votedisk
ocrcheck
ocrconfig -showbackup
```

### Key SQL Queries

```sql
-- Cluster instances
SELECT inst_id, instance_name, host_name, status 
FROM gv$instance 
ORDER BY inst_id;

-- ASM diskgroup usage
SELECT name, 
       total_mb/1024 total_gb, 
       free_mb/1024 free_gb,
       ROUND((1-(free_mb/total_mb))*100,2) pct_used
FROM v$asm_diskgroup
ORDER BY name;

-- Top session waits
SELECT inst_id, sid, serial#, event, wait_class, seconds_in_wait
FROM gv$session_wait
WHERE wait_class != 'Idle'
ORDER BY seconds_in_wait DESC;

-- Database files location
SELECT file_name FROM dba_data_files;
SELECT member FROM v$logfile;
SELECT name FROM v$controlfile;
SELECT name FROM v$tempfile;

-- Services registered
SELECT inst_id, name, network_name 
FROM gv$active_services 
ORDER BY inst_id, name;
```

### Parameter Reference

```sql
-- Key RAC parameters
ALTER SYSTEM SET cluster_database=TRUE SCOPE=SPFILE;
ALTER SYSTEM SET cluster_database_instances=2 SCOPE=SPFILE;
ALTER SYSTEM SET instance_number=1 SCOPE=SPFILE SID='ORCL1';
ALTER SYSTEM SET thread=1 SCOPE=SPFILE SID='ORCL1';
ALTER SYSTEM SET undo_tablespace='UNDOTBS1' SCOPE=SPFILE SID='ORCL1';
ALTER SYSTEM SET remote_listener='SCAN:1521' SCOPE=BOTH;
ALTER SYSTEM SET local_listener='(ADDRESS=(PROTOCOL=TCP)(HOST=vip)(PORT=1521))' SCOPE=BOTH;
```

### Network Configuration Example

```
# /etc/hosts example for 2-node RAC
# Public IPs
192.168.1.91    racnode1
192.168.1.92    racnode2

# Private IPs (Interconnect)
192.168.150.30  racnode1-priv
192.168.150.20  racnode2-priv

# Virtual IPs
192.168.1.95    racnode1-vip
192.168.1.96    racnode2-vip

# SCAN IPs
192.168.1.100   rac-scan
192.168.1.101   rac-scan
192.168.1.102   rac-scan
```

---

## Best Practices

### Before Any Operation

✅ **Always verify cluster status first**
```bash
crsctl check cluster -all
srvctl status database -d <db>
```

✅ **Take backups**
```bash
ocrconfig -manualbackup
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

✅ **Check alert logs before and after**
```bash
tail -100 $ORACLE_BASE/diag/rdbms/<db>/<instance>/trace/alert_*.log
```

### During Maintenance

✅ **Test in non-production first**  
✅ **Schedule maintenance windows**  
✅ **Keep support contracts active**  
✅ **Document all changes**  
✅ **Have rollback plan ready**

### After Operations

✅ **Verify cluster resources**
```bash
crsctl status resource -t
```

✅ **Check database status**
```sql
SELECT inst_id, instance_name, status FROM gv$instance;
```

✅ **Verify services**
```sql
SELECT inst_id, name FROM gv$services ORDER BY inst_id;
```

✅ **Test application connectivity**
```bash
tnsping <service_name>
sqlplus user/pass@service_name
```

---

## Performance Tuning Tips

### Interconnect Optimization

1. **Use dedicated network for interconnect** (separate from public network)
2. **Enable Jumbo Frames** (MTU 9000) if supported
3. **Disable unnecessary NIC offloads** if causing packet loss
4. **Use bonding/teaming** for redundancy (mode 4/802.3ad recommended)
5. **Monitor interconnect traffic** regularly

```bash
# Check MTU
ip link show eth1

# Set Jumbo Frames (example)
ifconfig eth1 mtu 9000

# Check bonding mode
cat /proc/net/bonding/bond0
```

### ASM Best Practices

1. **Allocate temp files equal to number of instances**
2. **Use NORMAL or HIGH redundancy** for production
3. **Monitor rebalance operations** (don't set power too high)
4. **Keep diskgroup balanced** (similar sized disks)
5. **Use ASM preferred read** for extended clusters

```sql
-- Set ASM preferred read
ALTER SYSTEM SET asm_preferred_read_failure_groups='DG.FG1' SCOPE=BOTH SID='ORCL1';
```

### Service Configuration

1. **Use services** instead of connecting to instances directly
2. **Configure TAF** for high availability
3. **Enable connection load balancing** (CLB)
4. **Use runtime load balancing** (RLB) with connection pools
5. **Set proper DTP settings** for distributed transactions

### Database Parameters

```sql
-- Recommended RAC parameters
ALTER SYSTEM SET parallel_execution_message_size=16384 SCOPE=SPFILE;
ALTER SYSTEM SET gc_files_to_locks='' SCOPE=SPFILE;
ALTER SYSTEM SET dml_locks=10000 SCOPE=SPFILE;
ALTER SYSTEM SET processes=1000 SCOPE=SPFILE;
```

---

## Troubleshooting Checklist

### Node Cannot Join Cluster

- [ ] Check network connectivity (ping, ssh)
- [ ] Verify time synchronization (chronyd/ntpd)
- [ ] Check /etc/hosts entries
- [ ] Verify voting disk access
- [ ] Check OCR permissions
- [ ] Review CRS logs

### Database Won't Start

- [ ] Check CRS resource status
- [ ] Verify ASM diskgroups mounted
- [ ] Check SPFILE location
- [ ] Review alert log
- [ ] Check listener status
- [ ] Verify network configuration

### Interconnect Issues

- [ ] Check network statistics (netstat -s)
- [ ] Verify NIC configuration (ethtool)
- [ ] Check switch port statistics
- [ ] Monitor packet loss
- [ ] Review gc wait events in AWR
- [ ] Check IRQ distribution

### Performance Problems

- [ ] Review AWR/STATSPACK reports
- [ ] Check wait events (gv$session_wait)
- [ ] Analyze SQL performance (ASH)
- [ ] Check interconnect latency
- [ ] Review storage I/O statistics
- [ ] Examine GC contention

---

## Common Errors & Solutions

| Error Code | Description | Solution |
|------------|-------------|----------|
| ORA-29702 | Error occurred in Cluster Group Service operation | Check CRS status, restart CRS if needed |
| ORA-12514 | TNS:listener does not currently know of service | Start service, check listener registration |
| ORA-15032 | Not all ASM disk candidates found | Check ASM disk discovery, restart oracleasm |
| ORA-15017 | Diskgroup cannot be mounted | Verify disk ownership and permissions |
| PRKP-1001 | Cannot find credentials | Copy password file to correct location |
| CRS-0215 | Could not start resource | Check resource dependencies, review logs |
| CRS-2771 | Maximum restart attempts reached | Clear resource failure, manually start |

### RAC-Specific Issues

1. Check interconnect latency
2. Review global cache statistics
3. Identify gc wait events
4. Analyze block transfer patterns
5. Consider application partitioning by instance

---

## Reference Documentation

### Oracle Documentation
- [Oracle RAC Administration and Deployment Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/)
- [Oracle Clusterware Administration and Deployment Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwadd/)
- [Oracle ASM Administrator's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/)

### Key MOS Notes
- **301137.1** - OSWatcher Diagnostic Tool
- **563566.1** - gc block lost Diagnostics
- **4882834** - Temp Segment Imbalance in RAC
- **1622502.1** - SCAN Listener Configuration
- **1584474.1** - Server Pools and Policy-Managed Databases

### Useful Utilities
- **OSWatcher** - OS monitoring (CPU, memory, I/O, network)
- **RDA** - Remote Diagnostic Agent (configuration collector)
- **TFA** - Trace File Analyzer (log collector)
- **CLUVFY** - Cluster verification utility
- **ORACHK** - Oracle health check tool

---

## Appendix: Sample Scripts

### Check Cluster Health

```bash
#!/bin/bash
# cluster_health_check.sh

echo "=== Cluster Status ==="
crsctl check cluster -all

echo -e "\n=== CRS Resources ==="
crsctl status resource -t

echo -e "\n=== Node Applications ==="
srvctl status nodeapps

echo -e "\n=== SCAN Status ==="
srvctl status scan
srvctl status scan_listener

echo -e "\n=== Database Status ==="
for db in $(srvctl config database); do
    echo "Database: $db"
    srvctl status database -d $db
done

echo -e "\n=== ASM Status ==="
srvctl status asm -a

echo -e "\n=== Voting Disks ==="
crsctl query css votedisk

echo -e "\n=== OCR Check ==="
ocrcheck
```

### ASM Disk Usage Report

```bash
#!/bin/bash
# asm_disk_report.sh

sqlplus -s / as sysasm <<EOF
SET PAGESIZE 100
SET LINESIZE 200
COLUMN name FORMAT A20
COLUMN total_gb FORMAT 999,999
COLUMN free_gb FORMAT 999,999
COLUMN used_gb FORMAT 999,999
COLUMN pct_used FORMAT 999.99

SELECT name,
       total_mb/1024 total_gb,
       (total_mb - free_mb)/1024 used_gb,
       free_mb/1024 free_gb,
       ROUND((total_mb - free_mb)/total_mb * 100, 2) pct_used
FROM v\$asm_diskgroup
ORDER BY name;
EOF
```

### Session Monitor

```bash
#!/bin/bash
# session_monitor.sh

sqlplus -s / as sysdba <<EOF
SET PAGESIZE 100
SET LINESIZE 200
COLUMN username FORMAT A15
COLUMN program FORMAT A30
COLUMN event FORMAT A30
COLUMN wait_class FORMAT A20

SELECT s.inst_id, s.username, s.sid, s.serial#,
       s.program, sw.event, sw.wait_class,
       sw.seconds_in_wait
FROM gv\$session s, gv\$session_wait sw
WHERE s.sid = sw.sid
AND s.inst_id = sw.inst_id
AND s.username IS NOT NULL
AND sw.wait_class != 'Idle'
ORDER BY sw.seconds_in_wait DESC;
EOF
```

---

## Support and Contributions

### Getting Help

1. **Oracle Support** (My Oracle Support) - For licensed customers
2. **Oracle Community** - Forums and discussion boards
3. **Oracle Documentation** - Official guides and references
4. **Oracle University** - Training and certification

### Report Issues

If you find errors or have suggestions for this guide:
- Open an issue on the repository
- Provide Oracle version and environment details
- Include error messages and logs where applicable

---

## Disclaimer

This guide is provided as-is for educational and operational purposes. Always:
- Test in non-production environments first
- Refer to official Oracle documentation
- Engage Oracle Support for production issues
- Maintain valid support contracts
- Keep systems patched and updated

**Author:** Monowar Mukul, Oracle DBA Community  
**Last Updated:** January 2025  
**Version:** 1.0  
**License:** MIT (or your chosen license)

---

**End of Document**
