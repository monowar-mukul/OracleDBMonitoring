
RMAN 

### Different option to configure
```
RMAN> configure device type disk clear;
RMAN> configure device type disk backup type to backupset;
RMAN> configure device type disk backup type to compressed backupset;
RMAN> configure channel device type disk maxpiecesize 1g;
RMAN> configure retention policy to redundancy 2;
RMAN> show retention policy;
RMAN> configure default device type clear; # reverts to the default device type (DISK)
RMAN> configure channel device type sbt clear; # erases all options that were set for the sbt channel
RMAN> configure channel 1 device type disk clear; # erases configuration values set specifically for channel 1.
RMAN> configure backup optimization on;
RMAN> CONFIGURE DB_UNIQUE_NAME PIIB1_01 CONNECT IDENTIFIER 'PIIB1_01';
RMAN> CONFIGURE DB_UNIQUE_NAME PIIB1_02 CONNECT IDENTIFIER 'PIIB1_02';
RMAN> RESYNC CATALOG FROM DB_UNIQUE_NAME PIIB1_02;
```

### Multiple copies
```
RMAN> configure datafile backup copies for device type disk to 2;
RMAN> configure datafile backup copies for device type sbt to 2;
RMAN> configure channel device type disk format '/save1/%U','/save2/%U','save3/%U';
```
###TAG
```
RMAN> backup as copy tag users_bkp tablespace users;
```

### Debug Output
```
$ rman target / debug=all log=rman_output.log
The following example enables debugging just for I/O activities:
$ rman target / debug=io
```

### To Spool RMAN Log
```
spool log to '/tmp/rman/backuplog.f';
backup datafile 1;
spool log off;
```
```
$ rman | tee rman.log
```
```
$ rman target / log=rman_output.log
```

## Check Backup other than Archivelog backup
```
set linesize 500
col BACKUP_SIZE for a20
COL START_TIME FORMAT a20 heading "Start Time"
COL END_TIME FORMAT a20 heading "End Time"
COL STATUS FORMAT a14 heading "Status"

SELECT
INPUT_TYPE "BACKUP_TYPE",
--NVL(INPUT_BYTES/(1024*1024),0)"INPUT_BYTES(MB)",
--NVL(OUTPUT_BYTES/(1024*1024),0) "OUTPUT_BYTES(MB)",
STATUS,
TO_CHAR(START_TIME,'MM/DD/YYYY:hh24:mi:ss') as START_TIME,
TO_CHAR(END_TIME,'MM/DD/YYYY:hh24:mi:ss') as END_TIME,
TRUNC((ELAPSED_SECONDS/60),2) "ELAPSED_TIME(Min)",
--ROUND(COMPRESSION_RATIO,3)"COMPRESSION_RATIO",
--ROUND(INPUT_BYTES_PER_SEC/(1024*1024),2) "INPUT_BYTES_PER_SEC(MB)",
--ROUND(OUTPUT_BYTES_PER_SEC/(1024*1024),2) "OUTPUT_BYTES_PER_SEC(MB)",
--INPUT_BYTES_DISPLAY "INPUT_BYTES_DISPLAY",
OUTPUT_BYTES_DISPLAY "BACKUP_SIZE",
OUTPUT_DEVICE_TYPE "OUTPUT_DEVICE"
--INPUT_BYTES_PER_SEC_DISPLAY "INPUT_BYTES_PER_SEC_DIS",
--OUTPUT_BYTES_PER_SEC_DISPLAY "OUTPUT_BYTES_PER_SEC_DIS"
FROM V$RMAN_BACKUP_JOB_DETAILS
where start_time > SYSDATE -7
and INPUT_TYPE != 'ARCHIVELOG'
ORDER BY END_TIME DESC
/
```
#### Sample Output
```
BACKUP_TYPE   Status         Start Time           End Time             ELAPSED_TIME(Min) BACKUP_SIZE          OUTPUT_DEVICE
------------- -------------- -------------------- -------------------- ----------------- -------------------- -----------------
DB INCR       COMPLETED      08/21/2024:18:52:23  08/21/2024:18:54:28               2.08     3.60G            SBT_TAPE
```

## Check Archive Log backup

```
set linesize 500
col BACKUP_SIZE for a20
SELECT 
INPUT_TYPE "BACKUP_TYPE",
--NVL(INPUT_BYTES/(1024*1024),0)"INPUT_BYTES(MB)",
--NVL(OUTPUT_BYTES/(1024*1024),0) "OUTPUT_BYTES(MB)",
STATUS,
TO_CHAR(START_TIME,'MM/DD/YYYY:hh24:mi:ss') as START_TIME,
TO_CHAR(END_TIME,'MM/DD/YYYY:hh24:mi:ss') as END_TIME,
TRUNC((ELAPSED_SECONDS/60),2) "ELAPSED_TIME(Min)",
--ROUND(COMPRESSION_RATIO,3)"COMPRESSION_RATIO",
--ROUND(INPUT_BYTES_PER_SEC/(1024*1024),2) "INPUT_BYTES_PER_SEC(MB)",
--ROUND(OUTPUT_BYTES_PER_SEC/(1024*1024),2) "OUTPUT_BYTES_PER_SEC(MB)",
--INPUT_BYTES_DISPLAY "INPUT_BYTES_DISPLAY",
OUTPUT_BYTES_DISPLAY "BACKUP_SIZE",
OUTPUT_DEVICE_TYPE "OUTPUT_DEVICE"
--INPUT_BYTES_PER_SEC_DISPLAY "INPUT_BYTES_PER_SEC_DIS",
--OUTPUT_BYTES_PER_SEC_DISPLAY "OUTPUT_BYTES_PER_SEC_DIS"
FROM V$RMAN_BACKUP_JOB_DETAILS
where start_time > SYSDATE -1/24
and INPUT_TYPE = 'ARCHIVELOG'
ORDER BY END_TIME DESC
/

```
#### Sample Output
```
BACKUP_TYPE   Status         Start Time           End Time             ELAPSED_TIME(Min) BACKUP_SIZE          OUTPUT_DEVICE
------------- -------------- -------------------- -------------------- ----------------- -------------------- -----------------
ARCHIVELOG    COMPLETED      08/22/2024:08:16:26  08/22/2024:08:16:36                .16     1.50M            SBT_TAPE
```

## Check Backup Status
```
col STATUS format a9
col hrs format 999.99
select SESSION_KEY,SESSION_RECID,SESSION_STAMP, INPUT_TYPE, STATUS, to_char(START_TIME,'mm/dd/yy hh24:mi') start_time,
to_char(END_TIME,'mm/dd/yy hh24:mi') end_time,
elapsed_seconds/3600 hrs from V$RMAN_BACKUP_JOB_DETAILS
order by session_key;
```

#### Sample Output
```
SESSION_KEY SESSION_RECID SESSION_STAMP Backup Type Status    Start Time           End Time                 HRS
----------- ------------- ------------- ----------- --------- -------------------- -------------------- -------
          3             3    1177595692 ARCHIVELOG  COMPLETED 08/21/24 13:54       08/21/24 13:55           .00
          6             6    1177596976 ARCHIVELOG  COMPLETED 08/21/24 14:16       08/21/24 14:16           .00
          9             9    1177600636 ARCHIVELOG  COMPLETED 08/21/24 15:17       08/21/24 15:17           .00
         12            12    1177604201 ARCHIVELOG  COMPLETED 08/21/24 16:16       08/21/24 16:17           .00
         15            15    1177607795 ARCHIVELOG  COMPLETED 08/21/24 17:16       08/21/24 17:19           .05
         18            18    1177613532 DB INCR     COMPLETED 08/21/24 18:52       08/21/24 18:54           .03
         24            24    1177637225 ARCHIVELOG  COMPLETED 08/22/24 01:27       08/22/24 01:28           .02
         27            27    1177640750 ARCHIVELOG  COMPLETED 08/22/24 02:26       08/22/24 02:26           .00
         30            30    1177644172 ARCHIVELOG  COMPLETED 08/22/24 03:23       08/22/24 03:23           .00
         33            33    1177647817 ARCHIVELOG  COMPLETED 08/22/24 04:23       08/22/24 04:23           .00
         36            36    1177651577 ARCHIVELOG  COMPLETED 08/22/24 05:26       08/22/24 05:26           .00
         39            39    1177654954 ARCHIVELOG  COMPLETED 08/22/24 06:22       08/22/24 06:23           .01
         42            42    1177658855 ARCHIVELOG  COMPLETED 08/22/24 07:27       08/22/24 07:27           .00
         45            45    1177661774 ARCHIVELOG  COMPLETED 08/22/24 08:16       08/22/24 08:16           .00

14 rows selected.
```
## Get the SESSION_RECID,SESSION_STAMP from the above query and use to get the Backup job output for a specific backup job
```
set lines 200
set pages 1000
select output
from GV$RMAN_OUTPUT
where session_recid = &SESSION_RECID
and session_stamp = &SESSION_STAMP
order by recid;
```
#### Sample Output
```
Enter value for session_recid: 18
old   3: where session_recid = &SESSION_RECID
new   3: where session_recid = 18
Enter value for session_stamp: 1177613532
old   4: and session_stamp = &SESSION_STAMP
new   4: and session_stamp = 1177613532

OUTPUT
----------------------------------------------------------------------------------------------------------------------------------
connected to target database: ARDPRDSE (DBID=265749263)
connected to recovery catalog database
run {
CONFIGURE RETENTION POLICY TO NONE;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO BACKUPSET PARALLELISM 1;
CONFIGURE DATAFILE BACKUP COPIES FOR
----
crosschecked backup piece: found to be 'AVAILABLE'
backup piece handle=0d331t78_13_1_1 RECID=13 STAMP=1177613544
crosschecked backup piece: found to be 'AVAILABLE'
backup piece handle=0e331tas_14_1_1 RECID=14 STAMP=1177613661
Crosschecked 2 objects

DELETE EXPIRED BACKUP;
specification does not match any backup in the repository
366 rows selected.
```
### Backup of Archivelog related information
```
connect catalog rcat/Password@rcat

> sql "alter session set nls_date_format=''DD-MON-YYYY HH24:MI:SS''";
> LIST BACKUP OF ARCHIVELOG TIME between "TO_DATE('09/18/2015 22:00:00', 'MM/DD/YYYY hh24:mi:ss')" and "TO_DATE('09/18/2015 23:00:00','MM/DD/YYYY hh24:mi:ss')";
> LIST BACKUP OF DATABASE COMPLETED BETWEEN '18-SEP-2015 22:16:00' AND '18-SEP-2015 23:30:00';
> LIST BACKUP OF ARCHIVELOG FROM TIME "SYSDATE-6/24";
```
```
RMAN> crosscheck backup of database;
RMAN> delete expired backup;
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 1 DAYS;
RMAN> crosscheck backup of archivelog all;
RMAN> show all;
```
#### Restotre Archivelog
```
rman target sys/pass catalog owner_rmn/<password>@srvprmn2rmn
RMAN> 
run {
set archivelog destination to '/tmp/arch_restore';
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
restore archivelog
from time = "to_date('09-03-2016:09:00:00','DD-MM-YYYY:HH24:MI:SS')"
until time = "to_date('09-03-2016:11:20:00','DD-MM-YYYY:HH24:MI:SS')";
RELEASE CHANNEL c1;
RELEASE CHANNEL c2;
RELEASE CHANNEL c3;
RELEASE CHANNEL c4;
}
```
```
run{
set archivelog destination to 'F:\mstarData';
ALLOCATE CHANNEL ch1 device type disk;
restore archivelog from logseq=2831 until logseq=2835  thread=1;
RELEASE CHANNEL ch1;
}
```
```
run{
ALLOCATE CHANNEL ch1 device type disk;
restore archivelog sequence 89920;
RELEASE CHANNEL ch1;
}
```
### SCN based

## Got the SCN with the following:
```
column NEXT_CHANGE# format 99999999
select max(NEXT_CHANGE#)-1 n from v$backup_archivelog_details;

```
#### Restotre Archivelog frm TAPE BACKUP [ Here is an example- logseq=25318 until logseq=25324]
```
RUN {
ALLOCATE CHANNEL ch1 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch2 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch3 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch4 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch5 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
restore archivelog from logseq=25318 until logseq=25324  thread=1;
# Release channels
RELEASE CHANNEL ch1;
RELEASE CHANNEL ch2;
RELEASE CHANNEL ch3;
RELEASE CHANNEL ch4;
RELEASE CHANNEL ch5;
}
```

### Archived redo logs for a day
```
select count(1), avg(blocks*block_size)/1024 MB
from v$archived_log
where completion_time between sysdate-1 and sysdate;
```

### Retaining Backups for a Long Time [archival backups]
```
run
{
backup database
tag quarterly
keep until time 'sysdate+180'
restore point 2013Q1;
}
```

### Backing Up Only Those Files Previously Not Backed Up
```
RMAN> backup database not backed up;
RMAN> backup device type sbt archivelog all not backed up 2 times;
RMAN> backup database not backed up since time 'sysdate-31';
RMAN> report need backup;
RMAN> show retention policy;
RMAN> CONFIGURE RETENTION POLICY TO NONE;
RMAN> report need backup;
RMAN> report unrecoverable;
```
```
connect catalog rcat/xxxxxxb@rcat
rman target sys@xxxx
run {
  CONFIGURE BACKUP OPTIMIZATION ON;
  CONFIGURE DEFAULT DEVICE TYPE TO DISK;
  CONFIGURE CONTROLFILE AUTOBACKUP ON;
  CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET PARALLELISM 4;
  CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT 'E:\minestar\backup\minestar_archivelog_bkp_%U_%T';
  crosscheck archivelog all;
  delete expired archivelog all;
  backup as compressed backupset archivelog all not backed up;
  delete force noprompt archivelog until time "to_date(SYSDATE-(2/24))" backed up 1 times to disk;
}
```

### BACKUP USER to monitor backup for databases
```
set lines 200 pages 2000
create user mstbkp identified by xxxx;
grant connect,resource,dba to mstbkp;
```
```
CREATE OR REPLACE FORCE VIEW "SYS"."RMAN_REPORT"
("DB_NAME","INPUT_TYPE", "STATUS", "START_TIME", "END_TIME", "HRS", "SUM_BYTES_BACKED_IN_GB","SUM_BACKUP_PIECES_IN_GB", "OUTPUT_DEVICE_TYPE") AS
select a.NAME,
INPUT_TYPE,
STATUS,
TO_CHAR(START_TIME,'DD-MON-YYYY-HH24:mi:ss') start_time,
TO_CHAR(END_TIME,'DD-MON-YYYY-HH24:mi:ss') end_time,
ELAPSED_SECONDS/3600 hrs,
INPUT_BYTES/1024/1024/1024 SUM_BYTES_BACKED_IN_GB,
OUTPUT_BYTES/1024/1024/1024 SUM_BACKUP_PIECES_IN_GB,
OUTPUT_DEVICE_TYPE
FROM V_$RMAN_BACKUP_JOB_DETAILS,v$database a
order by a.name,SESSION_KEY;
```
```
CREATE OR REPLACE FORCE VIEW "RMAN_REPORT_DAILY"
("DB_NAME", "INPUT_TYPE", "STATUS", "START_TIME", "END_TIME")
AS
select name,INPUT_TYPE,STATUS,TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM v$database,V$RMAN_BACKUP_JOB_DETAILS where status in ('COMPLETED','COMPLETED WITH WARNINGS') and INPUT_TYPE in ('DB FULL','DB INCR') and OUTPUT_DEVICE_TYPE='SBT_TAPE' AND END_TIME=(SELECT MAX(END_TIME) FROM
V$RMAN_BACKUP_JOB_DETAILS WHERE STATUS in ('COMPLETED','COMPLETED WITH WARNINGS') AND INPUT_TYPE in ('DB FULL','DB INCR') and OUTPUT_DEVICE_TYPE='SBT_TAPE')
union
select name,INPUT_TYPE,STATUS,TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,TO_CHAR(END_TIME,'mm/dd/yy hh24:mi') end_time
FROM V$RMAN_BACKUP_JOB_DETAILS,v$database where status in ('COMPLETED','COMPLETED WITH WARNINGS') and INPUT_TYPE='ARCHIVELOG' and OUTPUT_DEVICE_TYPE='SBT_TAPE' AND END_TIME=(SELECT MAX(END_TIME) FROM
V$RMAN_BACKUP_JOB_DETAILS WHERE STATUS in ('COMPLETED','COMPLETED WITH WARNINGS') AND INPUT_TYPE='ARCHIVELOG' and OUTPUT_DEVICE_TYPE='SBT_TAPE');
```
```
create synonym mstbkp.RMAN_REPORT for sys.RMAN_REPORT;
create synonym mstbkp.RMAN_REPORT_DAILY for sys.RMAN_REPORT_DAILY;
grant select on sys.RMAN_REPORT_DAILY to mstbkp;
grant select on sys.rman_report to mstbkp;
```
```
conn mstbkp/xxxxx;
show user
select * from RMAN_REPORT_DAILY;
select * from rman_report where input_type<>'ARCHIVELOG';
```
  
### Backup_Info 

```
sqlplus  OWNER_RMN/<password>@<servicename>
```
```
--
-- Display RMAN backup information from dictionary 
--

set lines 150
set pages 25
COL INPUT_TYPE FORMAT a11 heading "Backup Type"
COL OUTPUT_DEVICE_TYPE FORMAT a8 heading "Dest"
COL START_TIME FORMAT a15 heading "Start Time"
COL END_TIME FORMAT a15 heading "End Time"
COL STATUS FORMAT a14 heading "Status"
COL IN_SIZE  FORMAT a10 heading "Size In"
COL IN_SEC FORMAT a10 heading "Bytes/Sec"
COL OUT_SIZE FORMAT a10 heading "Size Out"
COL OUT_SEC FORMAT a10 heading "Bytes/Sec"
COL TIME_TAKEN_DISPLAY FORMAT a10 heading "Duration"

SELECT INPUT_TYPE,
       OUTPUT_DEVICE_TYPE,
       STATUS,
       TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(END_TIME,'mm/dd/yy hh24:mi')   end_time,
       TIME_TAKEN_DISPLAY,
       INPUT_BYTES_DISPLAY in_size,
       INPUT_BYTES_PER_SEC_DISPLAY in_sec,
       OUTPUT_BYTES_DISPLAY out_size,
       OUTPUT_BYTES_PER_SEC_DISPLAY out_sec
FROM   V$RMAN_BACKUP_JOB_DETAILS
where START_TIME in (select max(START_TIME) from V$RMAN_BACKUP_JOB_DETAILS WHERE INPUT_TYPE != 'ARCHIVELOG')
ORDER BY SESSION_KEY
/
```
#### Sample Output
```
Backup Type Dest     Status         Start Time      End Time        Duration   Size In    Bytes/Sec  Size Out   Bytes/Sec
----------- -------- -------------- --------------- --------------- ---------- ---------- ---------- ---------- ----------
DB INCR     SBT_TAPE COMPLETED      08/21/24 18:52  08/21/24 18:54  00:02:05      30.72G    251.64M      3.60G     29.49M
```

### Without catalog then run under the database which backup is in progress
```
COLUMN CLIENT_INFO FORMAT a30
COLUMN SID FORMAT 999
COLUMN SPID FORMAT 9999

SELECT s.SID, p.SPID, s.CLIENT_INFO
FROM V$PROCESS p, V$SESSION s
WHERE p.ADDR = s.PADDR
AND CLIENT_INFO LIKE 'rman%'
;
```
### To monitor job progress:
Before starting the job, create a script file (called, for this example, longops) containing the following SQL statement:
```
SELECT SID, SERIAL#, CONTEXT, SOFAR, TOTALWORK,
       ROUND(SOFAR/TOTALWORK*100,2) "%_COMPLETE"
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%'
  AND OPNAME NOT LIKE '%aggregate%'
  AND TOTALWORK != 0
  AND SOFAR <> TOTALWORK
;  
```
### RESTORE DATABASE;

To monitor the sbt events, you can run the following SQL query:
```
COLUMN EVENT FORMAT a10
COLUMN SECONDS_IN_WAIT FORMAT 999
COLUMN STATE FORMAT a20
COLUMN CLIENT_INFO FORMAT a30

SELECT p.SPID,sw.EVENT,s.SECONDS_IN_WAIT AS SEC_WAIT, sw.STATE, s.CLIENT_INFO
FROM V$SESSION_WAIT sw, V$SESSION s, V$PROCESS p
WHERE sw.EVENT LIKE 's%bt%'
AND s.SID=sw.SID
AND s.PADDR=p.ADDR;
```

### LAST BACKUP OF DATAFILES:

```
set pagesize 1000
column file#  heading 'File#'      format 999 
column name   heading 'File name'  format a55           
column days   heading 'Days'       format 99,999.90 

SELECT dbf.file#,substr(dbf.name,1,55) name,
              to_char(
                least(
                  nvl(sysdate-max(bdf.completion_time),999999),
                  nvl(sum(sysdate-dbf.creation_time),999999),
                  nvl(sum(sysdate-b.time),999999)
                ) ,'999999.00') days
         FROM v$backup_datafile bdf, v$datafile dbf, v$backup b
        WHERE dbf.file# = bdf.file#(+)
          AND dbf.file# = b.file#
        GROUP BY dbf.file#,substr(dbf.name,1,55)
        ORDER BY days DESC;
```
### Sample Output
```
File# File name                                               Days
----- ------------------------------------------------------- ----------
    9 +DATA/ARDPRDSE/DATAFILE/spatial.691.1177585707                 .60
   10 +DATA/ARDPRDSE/DATAFILE/spatial_index.690.1177585717           .60
    4 +DATA/ARDPRDSE/DATAFILE/users.703.1177583837                   .60
    5 +DATA/ARDPRDSE/DATAFILE/data.699.1177585671                    .60
    6 +DATA/ARDPRDSE/DATAFILE/indexes.698.1177585679                 .60
    8 +DATA/ARDPRDSE/DATAFILE/sde.692.1177585697                     .60
    7 +DATA/ARDPRDSE/DATAFILE/perfstat_data.694.1177585689           .60
    1 +DATA/ARDPRDSE/DATAFILE/system.696.1177583831                  .60
    3 +DATA/ARDPRDSE/DATAFILE/undotbs1.685.1177583833                .60
    2 +DATA/ARDPRDSE/DATAFILE/sysaux.684.1177583833                  .60

```
### The following SQL query demonstrates the internal checks that Oracle performs to determine whether media recovery is required:
```
set lines 200
column name format a70
column status a40
SELECT
a.name,
a.checkpoint_change#,
b.checkpoint_change#,
CASE
WHEN ((a.checkpoint_change# - b.checkpoint_change#) = 0) THEN 'Startup Normal'
WHEN ((a.checkpoint_change# - b.checkpoint_change#) > 0) THEN 'Media Recovery'
WHEN ((a.checkpoint_change# - b.checkpoint_change#) < 0) THEN 'Old Control File'
ELSE 'what the ?'
END STATUS
FROM v$datafile a, -- control file SCN for datafile
v$datafile_header b -- datafile header SCN
WHERE a.file# = b.file#;
```
### Sample output
```
NAME                                                                   CHECKPOINT_CHANGE# CHECKPOINT_CHANGE# STATUS
---------------------------------------------------------------------- ------------------ ------------------ ----------------
+DATA/ARDPRDSE/DATAFILE/system.791.1176206567                                  1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/sysaux.792.1176206569                                  1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/undotbs1.793.1176206571                                1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/users.795.1176206575                                   1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/data.799.1176217347                                    1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/indexes.800.1176217353                                 1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/perfstat_data.801.1176217361                           1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/sde.802.1176217367                                     1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/spatial.803.1176217375                                 1.4019E+12         1.4019E+12 Startup Normal
+DATA/ARDPRDSE/DATAFILE/spatial_index.804.1176217383                           1.4019E+12         1.4019E+12 Startup Normal

10 rows selected.
```

### RMANjobs&Scripts

Target:
```
select session_key,input_type,status, to_char(start_time,'yyyy-mm-dd hh24:mi') start_time,to_char(end_time,'yyyy-mm-dd hh24:mi') end_time,output_bytes_display,time_taken_display
from V$RMAN_BACKUP_JOB_DETAILS
order by session_key desc;
```
Catalog DB:
```
select session_key,input_type,status, to_char(start_time,'yyyy-mm-dd hh24:mi') start_time,to_char(end_time,'yyyy-mm-dd hh24:mi') end_time,output_bytes_display,time_taken_display
from RC_RMAN_BACKUP_JOB_DETAILS
order by session_key desc;
```
```
select input_type,status, to_char(start_time,'yyyy-mm-dd hh24:mi') start_time,to_char(end_time,'yyyy-mm-dd hh24:mi') end_time
from RC_RMAN_BACKUP_JOB_DETAILS
where DB_NAME like '%&DB%'
order by session_key desc;
```
```
select r.sequence#,r.set_stamp,p.handle,p.tag,p.start_time,p.completion_time,r.first_change#,r.next_change#
from RC_BACKUP_PIECE p, RC_BACKUP_REDOLOG r
where r.set_stamp = p.set_stamp 
and r.set_count = p.set_count
and r.DB_NAME like '%&DB%'
order by sequence# desc;
```
### SNAPSHOT Check
```
select ctime "Date",
       decode(backup_type, 'L', 'Archive Log', 'D', 'Full', 'Incremental') backup_type,
        bsize "Size MB"
 from (select trunc(bp.completion_time) ctime
              , backup_type
              , round(sum(bp.bytes/1024/1024),2) bsize
       from v$backup_set bs, v$backup_piece bp
       where bs.set_stamp = bp.set_stamp
       and bs.set_count  = bp.set_count
       and bp.status = 'A'
       group by trunc(bp.completion_time), backup_type)
order by 1, 2;
```
### Sample output
```
Date      BACKUP_TYPE    Size MB
--------- ----------- ----------
21/AUG/24 Archive Log    3398.75
21/AUG/24 Full              13.7
21/AUG/24 Incremental    3685.25
22/AUG/24 Archive Log    1235.75
22/AUG/24 Full               4.5
```
### Query the details for a single archive log:

Target :
```
select r.sequence#,r.set_stamp,p.handle,p.tag,p.start_time,p.completion_time,r.first_change#,r.next_change#
from V$BACKUP_PIECE p, V$BACKUP_REDOLOG r
where r.set_stamp = p.set_stamp
and r.set_count = p.set_count
and r.sequence# = &seq;
```

Catalog DB:
```
select r.sequence#,r.set_stamp,p.handle,p.tag,p.start_time,p.completion_time,r.first_change#,r.next_change#
from RC_BACKUP_PIECE p, RC_BACKUP_REDOLOG r
where r.set_stamp = p.set_stamp 
and r.set_count = p.set_count
and r.sequence# = 743399;
```

### Find last archive log backed up:

Target:
```
select r.sequence#,r.set_stamp,p.handle,p.tag,p.start_time,p.completion_time,r.first_change#,r.next_change#
from V$BACKUP_PIECE p, V$BACKUP_REDOLOG r
where r.set_stamp = p.set_stamp 
and r.set_count = p.set_count
order by sequence# desc;
```
Catalog DB:
```
select r.sequence#,r.set_stamp,p.handle,p.tag,p.start_time,p.completion_time,r.first_change#,r.next_change#
from RC_BACKUP_PIECE p, RC_BACKUP_REDOLOG r
where r.set_stamp = p.set_stamp 
and r.set_count = p.set_count
order by sequence# desc;
```
### Display RMAN backup information from dictionary 

```
set lines 150
set pages 25
COL INPUT_TYPE FORMAT a11 heading "Backup Type"
COL OUTPUT_DEVICE_TYPE FORMAT a8 heading "Dest"
COL START_TIME FORMAT a15 heading "Start Time"
COL END_TIME FORMAT a15 heading "End Time"
COL STATUS FORMAT a14 heading "Status"
COL IN_SIZE  FORMAT a10 heading "Size In"
COL IN_SEC FORMAT a10 heading "Bytes/Sec"
COL OUT_SIZE FORMAT a10 heading "Size Out"
COL OUT_SEC FORMAT a10 heading "Bytes/Sec"
COL TIME_TAKEN_DISPLAY FORMAT a10 heading "Duration"

SELECT INPUT_TYPE,
       OUTPUT_DEVICE_TYPE,
       STATUS,
       TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(END_TIME,'mm/dd/yy hh24:mi')   end_time,
       TIME_TAKEN_DISPLAY,
       INPUT_BYTES_DISPLAY in_size,
       INPUT_BYTES_PER_SEC_DISPLAY in_sec,
       OUTPUT_BYTES_DISPLAY out_size,
       OUTPUT_BYTES_PER_SEC_DISPLAY out_sec
FROM   V$RMAN_BACKUP_JOB_DETAILS
where START_TIME in (select max(START_TIME) from V$RMAN_BACKUP_JOB_DETAILS WHERE INPUT_TYPE != 'ARCHIVELOG')
ORDER BY SESSION_KEY
/
```

### rman_bkup_last_err.sql
```
--
-- Display RMAN backup information from dictionary 
--

set lines 150
set pages 25
COL INPUT_TYPE FORMAT a11 heading "Backup Type"
COL OUTPUT_DEVICE_TYPE FORMAT a8 heading "Dest"
COL START_TIME FORMAT a15 heading "Start Time"
COL END_TIME FORMAT a15 heading "End Time"
COL STATUS FORMAT a14 heading "Status"
COL IN_SIZE  FORMAT a10 heading "Size In"
COL IN_SEC FORMAT a10 heading "Bytes/Sec"
COL OUT_SIZE FORMAT a10 heading "Size Out"
COL OUT_SEC FORMAT a10 heading "Bytes/Sec"
COL TIME_TAKEN_DISPLAY FORMAT a10 heading "Duration"

SELECT INPUT_TYPE,
       OUTPUT_DEVICE_TYPE,
       STATUS,
       TO_CHAR(START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(END_TIME,'mm/dd/yy hh24:mi')   end_time,
       TIME_TAKEN_DISPLAY,
       INPUT_BYTES_DISPLAY in_size,
       INPUT_BYTES_PER_SEC_DISPLAY in_sec,
       OUTPUT_BYTES_DISPLAY out_size,
       OUTPUT_BYTES_PER_SEC_DISPLAY out_sec
FROM   V$RMAN_BACKUP_JOB_DETAILS
where START_TIME in (select max(START_TIME) from V$RMAN_BACKUP_JOB_DETAILS WHERE INPUT_TYPE != 'ARCHIVELOG')
  AND STATUS not in ('COMPLETED','RUNNING')
ORDER BY SESSION_KEY
/
```

### rman_bkup_last_tape.sql
```
--
-- Display RMAN backup information from dictionary 
--

set lines 150
set pages 25
COL HOST_NAME FORMAT a32 heading "Host"
COL INSTANCE_NAME FORMAT a10 heading "Instance"
COL INPUT_TYPE FORMAT a11 heading "Backup Type"
COL OUTPUT_DEVICE_TYPE FORMAT a8 heading "Dest"
COL START_TIME FORMAT a15 heading "Start Time"
COL END_TIME FORMAT a15 heading "End Time"
COL STATUS FORMAT a14 heading "Status"
COL IN_SIZE  FORMAT a10 heading "Size In"
COL IN_SEC FORMAT a10 heading "Bytes/Sec"
COL OUT_SIZE FORMAT a10 heading "Size Out"
COL OUT_SEC FORMAT a10 heading "Bytes/Sec"
COL TIME_TAKEN_DISPLAY FORMAT a10 heading "Duration"

SELECT i.HOST_NAME,
       i.INSTANCE_NAME,
       r.INPUT_TYPE,
       r.OUTPUT_DEVICE_TYPE,
       r.STATUS,
       TO_CHAR(r.START_TIME,'mm/dd/yy hh24:mi') start_time,
       TO_CHAR(r.END_TIME,'mm/dd/yy hh24:mi')   end_time,
       r.TIME_TAKEN_DISPLAY,
       r.INPUT_BYTES_DISPLAY in_size,
       r.INPUT_BYTES_PER_SEC_DISPLAY in_sec,
       r.OUTPUT_BYTES_DISPLAY out_size,
       r.OUTPUT_BYTES_PER_SEC_DISPLAY out_sec
FROM   V$RMAN_BACKUP_JOB_DETAILS r
      ,V$INSTANCE i
where r.END_TIME in (select max(a.END_TIME) from V$RMAN_BACKUP_JOB_DETAILS a WHERE a.INPUT_TYPE = 'DB INCR' and a.OUTPUT_DEVICE_TYPE = 'SBT_TAPE')
 AND r.INPUT_TYPE = 'DB INCR'
 AND r.OUTPUT_DEVICE_TYPE = 'SBT_TAPE'
 AND (r.END_TIME < sysdate - 1  or r.STATUS not in ('COMPLETED','RUNNING'))
ORDER BY SESSION_KEY
/
```

### rman_error.sql
```
--
-- display RMAN errors from output for job - only available since bounce of DB (i.e. in memory)
--

set pages 0
set lines 132
col output format a130
SELECT output
 FROM v$rman_output
WHERE SESSION_RECID = &key
  AND (output like 'ORA-%' or output like 'RMAN-%')
ORDER BY recid;

-- IDENTIFIER=ORACLE_RMAN_BACKUP_DATAFILE:<database>
--
-- $Id: oracle_rman_backup_datafile.sql,v 1.7 2010/05/19 14:35:16 bfelton Exp $
--
-- Prints 0 if any data file belonging to a specific database
-- is not backed up using RMAN for the last NN hours. NN Hours
-- is provided as an argument to this script.
--
-- First parameter to this script is the database name
-- for output formatting purposes only.
--
set linesize 1000 pagesize 1000
set echo off feedback off verify off
set heading off
--

select *
from
    (select file#
    from v$datafile
    where enabled != 'READ ONLY'
    and creation_time < sysdate - (26/24)
    minus
    select DF_FILE#
    from v$backup_files
    group by DF_FILE#
    having max(completion_time) > sysdate - (26/24)
)
/

SELECT SID, SPID, CLIENT_INFO 
  FROM V$PROCESS p, V$SESSION s 
  WHERE p.ADDR = s.PADDR 
  AND CLIENT_INFO LIKE '%id=rman%';
```

### Monitoring RMAN Job Progress

how many RMAN jobs have been issued, the status of each job, what time they started and completed, what types of jobs they were, and so on. You would issue a query as follows: 
```
col STATUS format a9
col hrs format 999.99
select SESSION_KEY, INPUT_TYPE, STATUS,
to_char(START_TIME,'mm/dd/yy hh24:mi') start_time,
to_char(END_TIME,'mm/dd/yy hh24:mi')   end_time,
elapsed_seconds/3600                   hrs
from V$RMAN_BACKUP_JOB_DETAILS
order by session_key;l
```

```
SQL> select sid, serial#, sofar, totalwork, opname,
round(sofar/totalwork*100,2) "% Complete"
from v$session_longops
where opname LIKE 'RMAN%'
and opname NOT LIKE '%aggregate%'
and totalwork != 0
and sofar <> totalwork;
```
```
SQL> select type, item, units, sofar, total from v$recovery_progress;
```
the rate of the backup produced and how fast data was read and written by the process. This information helps you diagnose any slowness in the RMAN jobs. 
```
col ins format a10
col outs format a10
select SESSION_KEY,
OPTIMIZED,
COMPRESSION_RATIO,
INPUT_BYTES_PER_SEC_DISPLAY ins,
OUTPUT_BYTES_PER_SEC_DISPLAY outs,
TIME_TAKEN_DISPLAY
from V$RMAN_BACKUP_JOB_DETAILS
order by session_key;  
```

Next, use the V$SESSION and V$PROCESS views to identify which database server
sessions correspond to RMAN channels:
```
SQL> SELECT b.sid, b.serial#, a.spid, b.client_info
FROM v$process a, v$session b
WHERE a.addr = b.paddr
AND b.client_info LIKE '%rman%';
```
### Measuring Backup Performance

determine whether backups are taking longer and longer.
```
SQL> SELECT session_recid, input_bytes_per_sec_display,
output_bytes_per_sec_display,
time_taken_display, end_time
 FROM v$rman_backup_job_details
 ORDER BY end_time;
```
### RMAN Command History
```
SQL> select  sid, recid, output
 from v$rman_output
 order by recid
 /
```
You can also join V$RMAN_OUTPUT to V$RMAN_STATUS to get additional information.
This useful query shows the type of command RMAN is running, its current status, and its
associated output messages:
```
SQL> select
 a.sid,
 a.recid,
 b.operation,
 b.status,
 a.output
 from v$rman_output a,
 v$rman_status b
 where a.rman_status_recid = b.recid
 and a.rman_status_stamp = b.stamp
 order by a.recid
 /
```

Script – Query the RMAN catalog to list backup completion status
####################################################################################

Note run this query connected as the owner of the RMAN catalog
```
set lines 80

set pages 250

ttitle "Daily Backup........"

select DB NAME,dbid,NVL(TO_CHAR(max(backuptype_db),'DD/MM/YYYY HH24:MI'),'01/01/0001:00:00') DBBKP,
NVL(TO_CHAR(max(backuptype_arch),'DD/MM/YYYY HH24:MI'),'01/01/0001:00:00') ARCBKP
from (
select a.name DB,dbid,
decode(b.bck_type,'D',max(b.completion_time),'I', max(b.completion_time)) BACKUPTYPE_db,
decode(b.bck_type,'L',max(b.completion_time)) BACKUPTYPE_arch
from rc_database a,bs b
where a.db_key=b.db_key
and b.bck_type is not null
and b.bs_key not in(Select bs_key from rc_backup_controlfile where AUTOBACKUP_DATE
is not null or AUTOBACKUP_SEQUENCE is not null)
and b.bs_key not in(select bs_key from rc_backup_spfile)
group by a.name,dbid,b.bck_type
) group by db,dbid
ORDER BY least(to_date(DBBKP,'DD/MM/YYYY HH24:MI'),to_date(ARCBKP,'DD/MM/YYYY HH24:MI'))
/
```
```
---------------------------------------------------------------------
-- File Name    : archive-backup-size.sql
-- Description  : Size for Archive Files backed up daily
-- Last Modified: 08-Mar-2012
--
---------------------------------------------------------------------

SELECT db_name,
       ceil(sum(FILESIZE)/(1024*1024*1024)) redo_archive_size_gb,           
to_char(FIRST_TIME, 'yy-mm-dd') date_backup
FROM rc_backup_archivelog_details
GROUP BY db_name,to_char(first_time, 'yy-mm-dd')
ORDER BY db_name, 3;
```
```
---------------------------------------------------------------------
-- File Name    : datafiles-backup-size.sql
-- Replace
-- Description  : Size for Archive Files backed up daily
-- Last Modified: 08-Mar-2012
---------------------------------------------------------------------

SELECT db_name,
       ceil(sum(FILESIZE)/(1024*1024*1024)),
       to_char(checkpoint_time, 'yy-mm-dd') date_backup,
       handle
FROM rc_backup_datafile_details
GROUP db_name,to_char(checkpoint_time, 'yy-mm-dd'), handle
ORDER BY db_name, 3;
```
```
---------------------------------------------------------------------
-- File Name    : backup-async-io-rate

-- Description  : recovery async io rate
-- Last Modified: 08-Mar-2012
---------------------------------------------------------------------

COL filename FOR a78;  
COL device FOR a6;
COL lw FOR 99999;
COL sw FOR 99999 ;
SET PAGES 50 LINES 250;
COL sid FOR 9999;

SELECT sid,
       inst_id,
       type,
       filename,
       status,
       to_char (open_time, 'mm/dd/yyyy hh24:mi:ss') open,
       to_char (close_time, 'mm/dd/yyyy hh24:mi:ss') CLOSE,
       round(effective_bytes_per_second/1024/1024) mbps
FROM gv$backup_async_io    
WHERE close_time > SYSDATE - 1
AND type='AGGREGATE'
ORDER BY close_time ;

SELECT SID, SERIAL#, CONTEXT, SOFAR, TOTALWORK, 
ROUND (SOFAR/TOTALWORK*100, 2) "% COMPLETE"
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%' AND OPNAME NOT LIKE '%aggregate%'
AND TOTALWORK! = 0 AND SOFAR <> TOTALWORK;
```
### FROM RMAN catalog database 
```
set lines 80
set pages 250
ttitle "Daily Backup........"
select DB NAME,dbid,
NVL(TO_CHAR(max(backuptype_db),'DD/MM/YYYY HH24:MI'),'01/01/0001:00:00') DBBKP,
NVL(TO_CHAR(max(backuptype_arch),'DD/MM/YYYY HH24:MI'),'01/01/0001:00:00') ARCBKP
from (
select a.name DB,dbid,
decode(b.bck_type,'D',max(b.completion_time),'I',
max(b.completion_time)) BACKUPTYPE_db,
decode(b.bck_type,'L',
max(b.completion_time)) BACKUPTYPE_arch
from rc_database a,bs b
where a.db_key=b.db_key
and b.bck_type is not null
and b.bs_key not in(Select bs_key from rc_backup_controlfile
where AUTOBACKUP_DATE is not null or AUTOBACKUP_SEQUENCE is not null)
and b.bs_key not in(select bs_key from rc_backup_spfile)
group by a.name,dbid,b.bck_type
) group by db,dbid
ORDER BY least(to_date(DBBKP,'DD/MM/YYYY HH24:MI'),
to_date(ARCBKP,'DD/MM/YYYY HH24:MI'));
 ```

### OEM
----
Extracting Backup success/failure reports from the OEM repository

If OEM Grid control is being used to monitor a majority of your databases and if RMAN is being used to backup the databases (either to Tape or to disk), then a quick way to get backup status reports directly off the OEM repository database is to run a script similar to the below.

In one of my client environments, there was no RMAN catalog being used and hence the need to pull this out from the OEM repository to report on a regular basis
```
set lines 200 pages 999
col database_name format a40
col host format a40
col status format a20
col start_time format a20
col end_time format a20
col time_taken_displat format a20
col output_device_type format a10
select DATABASE_NAME, host, status, to_char(start_time,'dd-MON-yyyy hh24:mi') start_time, to_char(end_time,'dd-MON-yyyy hh24:mi') end_time, input_type, output_device_type
from mgmt$ha_backup
order by host, database_name;
```
```
--
-- Display RMAN backup information from dictionary
--

set lines 180
set pages 25
COL SESSION_KEY FORMAT 9999999 heading "Key"
COL DOW FORMAT a3 heading "Day"
COL INPUT_TYPE FORMAT a11 heading "Backup Type"
COL TYPE FORMAT a5 heading "Sets"
COL LOGS FORMAT a5 heading "Logs"
COL CTRL FORMAT a5 heading "Ctrl"
COL OUTPUT_DEVICE_TYPE FORMAT a8 heading "Dest"
COL START_TIME FORMAT a15 heading "Start Time"
COL END_TIME FORMAT a15 heading "End Time"
COL STATUS FORMAT a14 heading "Status"
COL IN_SIZE  FORMAT a10 heading "Size In"
COL IN_SEC FORMAT a10 heading "Bytes/Sec"
COL OUT_SIZE FORMAT a10 heading "Size Out"
COL OUT_SEC FORMAT a10 heading "Bytes/Sec"
COL TIME_TAKEN_DISPLAY FORMAT a10 heading "Duration"

WITH
s as
(
select
   d.session_recid
 , d.session_stamp
 , decode(sum(d.incremental_level),NULL
      ,decode(sum(case when d.backup_type = 'L' then d.pieces else 0 end),0,'Full','Arch')
      ,0,'Lvl-0','Lvl-1') as type
 , decode(sum(case when d.backup_type = 'L' then d.pieces else 0 end),0,'','+Arch') as logs
 , decode(sum(case when d.controlfile_included = 'YES' then d.pieces else 0 end),0,NULL,'+Ctrl') as ctrl
from
   V$BACKUP_SET_DETAILS d
   join V$BACKUP_SET s on s.set_stamp = d.set_stamp and s.set_count = d.set_count
      where s.input_file_scan_only = 'NO'
group by d.session_recid
       , d.session_stamp
)
SELECT j.SESSION_KEY
     ,decode(to_char(j.start_time, 'd')
        ,1, 'Sun'
        ,2, 'Mon'
        ,3, 'Tue'
        ,4, 'Wed'
        ,5, 'Thr'
        ,6, 'Fri'
        ,7, 'Sat') dow
     , j.INPUT_TYPE
     , j.OUTPUT_DEVICE_TYPE
     , s.TYPE
     , s.LOGS
     , s.CTRL
     , replace(j.STATUS,'COMPLETED WITH','**') status
     , TO_CHAR(j.START_TIME,'mm/dd/yy hh24:mi') start_time
     , TO_CHAR(j.END_TIME,'mm/dd/yy hh24:mi')   end_time
     , j.TIME_TAKEN_DISPLAY
     , j.INPUT_BYTES_DISPLAY in_size
     , j.INPUT_BYTES_PER_SEC_DISPLAY in_sec
     , j.OUTPUT_BYTES_DISPLAY out_size
     , j.OUTPUT_BYTES_PER_SEC_DISPLAY out_sec
FROM   V$RMAN_BACKUP_JOB_DETAILS j
JOIN   s on j.session_recid = s.session_recid and j.session_stamp = s.session_stamp
ORDER BY SESSION_KEY
/
```
Sample Output:
```
     Key Day Backup Type Dest     Sets  Logs  Ctrl  Status         Start Time      End Time        Duration   Size In    Bytes/Sec  Size Out   Bytes/Sec
-------- --- ----------- -------- ----- ----- ----- -------------- --------------- --------------- ---------- ---------- ---------- ---------- ----------
   99461 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 15:23  08/15/24 15:23  00:00:15     104.70M      6.98M      8.00M    546.13K
   99466 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 15:43  08/15/24 15:43  00:00:15     109.11M      7.27M      9.00M    614.40K
   99471 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 16:03  08/15/24 16:03  00:00:15     109.94M      7.33M      9.00M    614.40K
   99476 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 16:23  08/15/24 16:23  00:00:16     101.23M      6.33M      7.25M    464.00K
   99481 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 16:42  08/15/24 16:43  00:00:15     107.22M      7.15M      8.50M    580.27K
   99486 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 17:02  08/15/24 17:03  00:00:16     110.63M      6.91M      9.00M    576.00K
   99491 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 17:22  08/15/24 17:23  00:00:15     100.55M      6.70M      7.00M    477.87K
   99496 Thr ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/15/24 17:42  08/15/24 17:43  00:00:16     104.95M      6.56M      8.00M    512.00K
   99501 Thr DB INCR     SBT_TAPE Lvl-1       +Ctrl ** WARNINGS    08/15/24 21:11  08/15/24 21:14  00:02:30     138.61G    946.26M    127.75M    872.11K
   99507 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 04:38  08/16/24 04:39  00:00:28     951.50M     33.98M    229.25M      8.19M
   99512 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 04:42  08/16/24 04:42  00:00:15      87.70M      5.85M      3.25M    221.87K
   99517 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 05:03  08/16/24 05:03  00:00:17     108.52M      6.38M      8.75M    527.06K
   99522 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 05:22  08/16/24 05:23  00:00:16      99.45M      6.22M      6.75M    432.00K
   99527 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 05:42  08/16/24 05:43  00:00:15     104.39M      6.96M      8.00M    546.13K
   99532 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 06:03  08/16/24 06:03  00:00:16     108.60M      6.79M      8.75M    560.00K
   99537 Fri ARCHIVELOG  SBT_TAPE Arch  +Arch +Ctrl ** WARNINGS    08/16/24 06:23  08/16/24 06:23  00:00:15      99.74M      6.65M      7.00M    477.87K
```
Note: in my case it was relate to deletion policy setup - which I know I can ignore.

### Get the SCN for the BACKUP for Database
---------------------------------------------------------------------------------------------
```
sqlplus "/ as sysdba"
select dbid from v$database;

connect rman/recman00@recman
select 'set until scn ' || to_char( min(next_scn) - 1 ) ||
 '; ## ' || to_char( min(next_time) - 1/86440, 'DD-MON-RRRR HH24:MI:SS' )
from al
where	
dbinc_key = ( select curr_dbinc_key from db where db_id = &dbid ) and
next_time  - (1/86440) > ( select max(completion_time)
from bdf
where dbinc_key = ( select curr_dbinc_key from db where db_id = &dbid ) )
/
```
### For a specific date backup
```
SELECT   D.FILE#, N.NAME, P.HANDLE ,P.device_type, P.media, P.TAG, P.status, TO_CHAR(P.START_TIME,'DD-MON-RR HH24:MI:SS') START_TIME, P.completion_time
FROM   V$BACKUP_PIECE P, V$BACKUP_DATAFILE D , V$DATAFILE N
WHERE   D.SET_STAMP = P.SET_STAMP 
 AND    D.FILE# = N.FILE#  
 AND    D.SET_COUNT = P.SET_COUNT 
 AND P.completion_time >sysdate -1 
--order by D.FILE#, P.completion_time;
-- order by P.completion_time, D.file#;*/
order by D.file#;

```
```
select backup_type, incremental_level, round(sum(original_input_bytes)/1024/1024,2) "MB in",
 round(sum(output_bytes)/1024/1024,2) "MB out", status, min(start_time), max(completion_time),
 round((sum(output_bytes)/1024/1024)/((max(completion_time)-min(start_time))*86400), 1) "MB/s"
 from v$backup_set_details
 group by backup_type, incremental_level, status, session_key, session_recid, session_stamp
 order by 6
 /
```
```
select b.sid, b.serial#, a.spid, b.client_info
from v$process a, v$session b
where a.addr=b.paddr and
client_info like 'rman%'; 
```
```
set linesize 200
SELECT SESSION_KEY,TO_CHAR(START_TIME,'DD-MON-RR HH24:MI:SS') START_TIME,
TO_CHAR(END_TIME,'DD-MON-RR HH24:MI:SS') END_TIME,ROUND(INPUT_BYTES/1048576) INPUT_BYTES ,
ROUND(OUTPUT_BYTES/1048576) OUTPUT_BYTES,STATUS,INPUT_TYPE,ROUND(ELAPSED_SECONDS/60,2) ELAPSED_MINUTES
FROM V$RMAN_BACKUP_JOB_DETAILS  ORDER BY SESSION_KEY;
```
```
set linesize 200
SELECT   D.FILE#, N.NAME, P.HANDLE ,P.device_type, P.media, P.TAG, P.status, TO_CHAR(P.START_TIME,'DD-MON-RR HH24:MI:SS') START_TIME, P.completion_time
FROM   V$BACKUP_PIECE P, V$BACKUP_DATAFILE D , V$DATAFILE N
WHERE   D.SET_STAMP = P.SET_STAMP 
 AND    D.FILE# = N.FILE#  
 AND    D.SET_COUNT = P.SET_COUNT 
 --order by D.FILE#, P.completion_time;
 order by P.completion_time, D.file#;
-- P.completion_time =<>
-- order by D.file#;
```
```
set lines 200 pages 999
col database_name format a40
col host format a40
col status format a20
col start_time format a20
col end_time format a20
col time_taken_displat format a20
col output_device_type format a10

select sid, serial#, sofar, totalwork, opname,
round(sofar/totalwork*100,2) "% Complete"
from v$session_longops
where opname LIKE 'RMAN%'
and opname NOT LIKE '%aggregate%'
and totalwork != 0
and sofar <> totalwork;
```
```
> sql "alter session set nls_date_format=''DD-MON-YYYY HH24:MI:SS''";
> LIST BACKUP OF DATABASE COMPLETED BETWEEN '10-OCT-2014 00:00:00' AND '12-OCT-2014 10:50:00';
> LIST BACKUP OF ARCHIVELOG TIME between "TO_DATE('10/10/2014 22:00:00', 'MM/DD/YYYY hh24:mi:ss')" and "TO_DATE('12/10/2014 11:00:00','MM/DD/YYYY hh24:mi:ss')";
>delete noprompt archivelog all completed before 'sysdate -3'; 
>delete noprompt expired archivelog all ;
select vadg.name
      ,round(vadg.total_mb/1024,2) total_gb
      ,round(vadg.free_mb/1024,2) free_gb
      ,round((vadg.total_mb - vadg.free_mb)/1024,2) used_gb
      ,round(100-(vadg.free_mb/vadg.total_mb*100),2) pct_used_gb
FROM v$asm_diskgroup vadg
where name =  (select substr(value,length('location=+')+1,999) log_dest 
                 from v$parameter 
               where name = 'log_archive_dest_1');
      ,round(vadg.total_mb/1024,2) total_gb
      ,round(vadg.free_mb/1024,2) free_gb
      ,round((vadg.total_mb - vadg.free_mb)/1024,2) used_gb
      ,round(100-(vadg.free_mb/vadg.total_mb*100),2) pct_used_gb
FROM v$asm_diskgroup vadg
where name =  (select substr(value,length('location=+')+1,999) log_dest 
                 from v$parameter 
               where name = 'log_archive_dest_1');

SELECT MIN(ROUND ( (total_mb - usable_file_mb) * 100 / total_mb, 2)) full_percent
  FROM V$ASM_DISKGROUP
 WHERE name IN
          (SELECT DISTINCT REPLACE (VALUE, 'LOCATION=+', '')
             FROM gv$parameter@ptest20
            WHERE name LIKE '%archive%dest%' AND VALUE LIKE '%LOCATION%')
where pct_used > '20';
```

### Restore Archivelog
----------------------
```
run{
set archivelog destination to '/tmp/archbkp';
ALLOCATE CHANNEL ch1 TYPE 'SBT_TAPE'
PARMS='ENV=(NB_ORA_CLIENT=ironper-orc03b,NB_ORA_POLICY=RMAN-ORACLE-HOST)';
restore archivelog from logseq=120200 until logseq=120220;
RELEASE CHANNEL c1;
}
```

### Begin/End Backup
backup_begin.sql
```
set echo off
set feed off
set termout off
set pages 0
spool hot_em.sql
prompt spool hot_em
SELECT 'alter tablespace '||tablespace_name||' begin backup;'
FROM dba_tablespaces
Where CONTENTS != 'TEMPORARY'
/
spool off
set echo on
set feed on
set termout on
@hot_em.sql
exit;
```

### Check_bkp.sql
```
set lines 80      
column name format a10
 
SELECT t.name, d.file# as, b.status      
FROM V$DATAFILE d, V$TABLESPACE t, V$BACKUP b       
WHERE d.TS#=t.TS# AND b.FILE#=d.FILE#;
```

end_bkp.sql
-----------
```
set echo off
set feed off
set termout off
set pages 0
spool unhot_em.sql
prompt spool unhot_em
SELECT 'alter tablespace '||tablespace_name||' end backup;'
FROM dba_tablespaces
WHERE CONTENTS != 'TEMPORARY'
/
spool off
set echo on
set feed on
set termout on
@unhot_em.sql
exit;
```
### BLOCKCHANGETRACKING
Where is the block change tracking file. used for 
---------------------------------------------------------------------------------------------
FAST incremental backups. CTWR background process keeps track of block changes and RMAN use it to do quicker incremental backups, then without the block change tracking file

```
set linesize 130
col filename format a50
select filename, status, (bytes)/1024/1024 as "Size in MB"
 from v$block_change_tracking;
```
```
FILENAME                                           STATUS                         Size in MB
-------------------------------------------------- ------------------------------ ----------
+DATA/qpdw1_01/changetracking/ctf.2257.888149015   ENABLED                           21.0625
```
### To check whether the block change tracking file is being used or not, use the below command .
```
select  file#,  avg(datafile_blocks), avg(blocks_read),  avg(blocks_read/datafile_blocks) * 100 as  "% read for backup"  
from v$backup_datafile  where incremental_level > 0  
and  used_change_tracking = 'YES'  
group by file#   
order by file# ;
```
```
Example Output:
     FILE# AVG(DATAFILE_BLOCKS) AVG(BLOCKS_READ) % read for backup
---------- -------------------- ---------------- -----------------
         1               450560       1879.58333        .417166045
         2           218986.667       11788.6667        5.37255894
---
---
        56               144640                1        .000691372
        57                12800       8.16666667        .063802083
        58                61440       3087.92308        5.02591647


### encrypt backup
```
1. Configure the Oracle Encryption Wallet (Oracle Wallet
SQL> alter system set encryption key identified by "welcome1";

Configure encrypted backups using the configure command, as shown in the following
example:
RMAN> configure encryption for database on;
RMAN> backup database;
```

### Password Encryption without wallet

If you don’t want to configure an Oracle Wallet, you can still perform encrypted backups by using the set encryption command. This method is called password encryption of backups
```
RMAN> set encryption on identified by monowar only;

executing command: SET encryption

RMAN> configure encryption for database off;

new RMAN configuration parameters:
CONFIGURE ENCRYPTION FOR DATABASE OFF;
new RMAN configuration parameters are successfully stored
```

### Faster backup
```
RMAN> configure device type sbt parallelism 3;
Execute the backup, specifying the section size parameter:
RMAN> backup section size 150m tablespace system;
```
### CATALOG and UNCATALOG
```
1) Catalog an archive log
RMAN>CATALOG ARCHIVELOG '/oracle/oradata/arju/arc001_223.arc', '/oracle/oradata/arju/arc001_224.arc';

2) Catalog a datafile copy
RMAN>CATALOG DATAFILECOPY '/oradata/backup/users01.dbf' LEVEL 0;

3) Catalog multiple copies in a directory
RMAN>CATALOG START WITH '/tmp/backups' NOPROMPT;

4) Catalog all files in the Flash Recovery Area
RMAN>CATALOG RECOVERY AREA NOPROMPT;

5) Catalog a backup piece
RMAN>CATALOG BACKUPPIECE '/oradata2/o4jccf4'; 

6) Uncatalog all archive logs
RMAN>CHANGE ARCHIVELOG ALL UNCATALOG;

7) Uncatalog a tablespace
RMAN>CHANGE BACKUP OF TABLESPACE USERS UNCATALOG;

8) Uncatalog a backuppiece
RMAN>CHANGE BACKUPPIECE '/oradata2/oft7qq' UNCATALOG;

```

### KILL BACKUP JOBS from OS
```
$ sqlplus / as sysdba 

### Check what Session will be killed
```
select s.sid, s.serial#, s.username,
       to_char(s.logon_time,'DD-MON HH24:MI:SS') logon_time,
       p.pid oraclepid, p.spid "ServerPID", s.process "ClientPID",
       s.program clientprogram, s.module, s.machine, s.osuser,
       s.status, s.last_call_et
from  v$session s, v$process p
where s.paddr=p.addr
and s.program like 'rman%'
order by s.sid
```
### Generate command
```
select '! kill -9 '||p.spid  kill_rman_process
from  v$session s, v$process p
where s.paddr=p.addr
and s.program like 'rman%'
order by s.sid
```
```
KILL_RMAN_PROCESS
----------------------------------
! kill -9 2516
! kill -9 2517
! kill -9 2497

### Just copy and paste
SQL> ! kill -9 2516
SQL> ! kill -9 2517
SQL> ! kill -9 2497  

```

### delete 

```
RMAN> delete backup;
RMAN> delete backuppiece 999;
RMAN> delete copy of controlfile like '/u01/%';
RMAN> delete backup tag='old_production';
RMAN> delete backup of tablespace sysaux device type sbt;
RMAN> backup device type disk archivelog all delete all input;
RMAN> backup device type disk archivelog all delete all input;
RMAN> delete archivelog all;
RMAN> delete archivelog all backed up 3 times to sbt;
RMAN> delete archivelog until sequence = 999;
RMAN> backup archivelog like '/arch%' delete input;
RMAN> configure archivelog deletion policy to backed up 2 times to device type sbt;
```
#### Delete Archivelog Backups - options example
The following command can be used to manage the backup of the archive log when storage space needs to be released.
```
RMAN>DELETE BACKUP OF archivelog UNTIL TIME=’sysdate-3;

```
run {
    backup format 'J:\backup\pqa7\pqa7_archivelog_bkp_%U_%T'
       archivelog until time 'sysdate-3' delete input;
    }
```
```
delete noprompt archivelog until time "to_date(SYSDATE-4)" backed up 1 times to disk;
```


### Deleting Obsolete RMAN Backups
==================================
```
RMAN> delete obsolete;
RMAN> delete obsolete redundancy = 2;
RMAN> delete obsolete recovery window of 14 days;

RUN
   {
   ALLOCATE CHANNEL c1 DEVICE TYPE sbt_tape;
   ALLOCATE CHANNEL c2 DEVICE TYPE sbt_tape';
    BACKUP DATABASE;
    sql 'alter system archive log current';
    sql 'alter system archive log current';
    change archivelog all validate;
    delete noprompt expired archivelog all;
     BACKUP ARCHIVELOG ALL;
    RELEASE CHANNEL c1;
    RELEASE CHANNEL c2;
   }
```
```
RMAN> connect target /
RMAN> delete noprompt archivelog until time 'SYSDATE - 3';
```

### Issue
```
ORA-15028: ASM file '+//thread_1_seq_**** not dropped; currently being accessed

In this case the solution was quite simple. The steps I followed were (bearing in mind that arc? processes are being restarted automatically by oracle):
1.- find the process id for arc:
ps -ef | grep -i ora_arc*
 oracle 5607 1 1 19:02 ? 00:00:00 ora_arc9_prod1
2.- kill the running process:
kill -9 5607
3.- check the process is started again before killing more:
ps -ef | grep -i ora_arc9_prod1
4.- perform 2 and 3 for all arc? running for your instance.
```  

### debug enabled 
```
$export NLS_DATE_FORMAT="DD-MON-RRRR HH24:MI:SS" 
$rman target sys/<password>@<tns string of target > catalog <catalog schema>/<password>@<tns string> auxiliary sys/<password>@<tns string> debug all trace=debug-duplicate.trc log =duplicate.log 
set echo on; 
show all; 
<<run the duplicate comman to re-produce the issue>> 
exit 

check 
debug-duplicate.trc 
duplicate.log 
alert log of target and auxiliary database 

```

### RMAN Duplicate  [ORA-19625 ORA-1517 ]
--------------------------------------------------- 
```
When doing a Active Duplicate Database, the command to copy the logfiles from source to target are issued correctly. 
However, log does not appear in the directory on the target database. 
This has previously when doing an 'Active Duplicate". 

Required archives not backedup due to backup optimization feature. Can you apply the patch 17843104 or turn off the backup optimization (RMAN>CONFIGURE BACKUP OPTIMIZATION OFF;) and let us know the status. 
Bug 17843104 - "DUPLICATE FROM ACTIVE" FAILED BECAUSE ARCHIVELOG FILES WERE SKIPPE
RMAN>CONFIGURE BACKUP OPTIMIZATION OFF; 

SCENARIOS:

Starting Control File and SPFILE Autobackup at 28-11-2015:03:44:46
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03015: error occurred in stored script arch_backup
RMAN-03009: failure of Control File and SPFILE Autobackup command on ORA_DISK_1 channel at 11/28/2015 03:45:04
ORA-19809: limit exceeded for recovery files
ORA-19804: cannot reclaim 2144337920 bytes disk space from 32212254720 limit

Solution  :
1. We need to increase the size of FRA (Flash Recovery Area).
2. You can offer delete command. It will delete old backup files (RMAN RETENTION POLICY).

You can use below command to check used/free space for flash recovery area:
    select name,
    floor(space_limit/1024/1024) "Size_MB",
    ceil(space_used/1024/1024) "Used_MB"
    from v$recovery_file_dest
    order by name
    /

Offer below command in sql promt to increase the size of FRA. It will resolve the problem.
    ALTER SYSTEM SET db_recovery_file_dest_size=12G SCOPE=BOTH ;

```
```
released channel: dev_0
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03009: failure of resync command on default channel at 03/16/2009 10:57:28
RMAN-20003: target database incarnation not found in recovery catalog

Recovery Manager complete.
[Major] From: ob2rman@msdaix05 "v5299"  Time: 03/16/09 10:57:29
	The database reported error while performing requested operation.

 RMAN-00571: ===========================================================
 RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
 RMAN-00571: ===========================================================
 RMAN-03009: failure of resync command on default channel at 03/16/2009 10:57:28
 RMAN-20003: target database incarnation not found in recovery catalog
 
SOLVE:
RMAN> list incarnation of database;
RMAN> register database;
```
```
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03009: failure of register command on default channel at 03/16/2009 11:04:18
RMAN-20002: target database already registered in recovery catalog

RMAN> reset database;

new incarnation of database registered in recovery catalog
starting full resync of recovery catalog
full resync complete
```
```
RMAN> register database;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of register command at 05/11/2009 18:51:58
RMAN-08040: full resync skipped, control file is not current or backup 

Go to rman session
connect to production (not DR)
register database
```
```
Starting backup at 08-NOV-10
current log archived
released channel: c1
released channel: c2
released channel: c3
released channel: c4
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of backup plus archivelog command at 11/08/2010 10:51:39
RMAN-06059: expected archived log not found, lost of archived log compromises recoverability
ORA-19625: error identifying file /IJISPRODARCH1/fmapprod/1_274_729188006.arc
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory Additional information: 3

RMAN> 
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=122 device type=DISK
RMAN-08138: WARNING: archived log not deleted - must create more backups archived log file name=/IJISPRODARCH1/fmapprod/1_251_729188006.arc thread=1 sequence=251

RMAN> connect target
connected to target database: FMAPPROD (DBID=1450765346)

RMAN> delete expired archivelog all;
Then run backup 
```

SCENARIO: 
```
Starting backup at 07/28/2009 [00:17:49]
current log archived
released channel: dev_0
released channel: dev_1
released channel: dev_2
released channel: dev_3
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ============
RMAN-00571: ===========================================================
RMAN-03002: failure of backup command at 07/28/2009 00:17:54
ORA-04031: unable to allocate 41456 bytes of shared memory ("shared pool","DBMS_RCVMAN","PL/SQL MPCODE","BAMIMA: Bam Buffer")
ORA-06508: PL/SQL: could not find program unit being called: "SYS.DBMS_RCVMAN"

sys@opmpprd> show parameter shared_

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
hi_shared_memory_address             integer     0
max_shared_servers                   integer
shared_memory_address                integer     0
shared_pool_reserved_size            big integer 5M
shared_pool_size                     big integer 84M
shared_server_sessions               integer
shared_servers                       integer     0

sys@opmpprd> alter system set shared_pool_size=100M scope=both;
System altered.
```
```
channel dev_0: finished piece 1 at 04/16/2008 [05:35:42]
piece handle=daily_oracle_de_spitzer_mwhprd_dr<mwhprd_7516:652165977:1>.dbf comment=API Version 2.0,MMS Version 65.5.50.0
channel dev_0: backup set complete, elapsed time: 00:22:45
channel dev_0: starting incremental level 0 datafile backupset
channel dev_0: specifying datafile(s) in backupset
input datafile fno=00062 name=/dbmwhprd02/oracle/mwhprd/mwhprd_pssind03_17.dbf
input datafile fno=00063 name=/dbmwhprd02/oracle/mwhprd/mwhprd_pssind03_18.dbf
input datafile fno=00067 name=/dbmwhprd02/oracle/mwhprd/mwhprd_pssind03_19.dbf
released channel: dev_0
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03009: failure of backup command on dev_0 channel at 04/16/2008 05:35:42
ORA-19573: cannot obtain sub-shared enqueue for datafile 78

RMAN> **end-of-file**

Solution:
A new datafile was added last night and a lock is held by the Managed Recovery Process:

From alert log:
Successfully added datafile 13 to media recovery
Datafile #13: '/dbmwhprd/oracle/mwhprd/mwhprd_pssind03_21.dbf'
Wed May  2 18:26:07 2007

This matches an Oracle bug:
Bug 2749174  Auto addition of standby datafile prevents RMAN backup from running
Description
RMAN backup on the standby fails with "ORA-19573: cannot obtain 
sub-shared enqueue for datafile <n>" for a file added to the standby
by the recovery process. This occurs as the MRP process gets the
enqueue in X mode preventing RMAN from obtaining it.

Workaround:
  Stop and restart the MRP process

stopped/restarted the Managed Recovery Process on Spitzer for MWHPRD_DR:

sys@mwhprd> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

Database altered.

sys@mwhprd> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;

Database altered.
```
```
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ============
RMAN-00571: ===========================================================
RMAN-03002: failure of backup command at 12/06/2007 09:22:29
RMAN-06059: expected archived log not found, lost of archived log compromises recoverability
ORA-19625: error identifying file /oracle/admin/aluiprd/archsb/aluiprd_0001_0623422764_0000006896.arc
ORA-27037: unable to obtain file status
SVR4 Error: 2: No such file or directory
Additional information: 3

Action:
crosscheck archivelog all ;
delete noprompt expired archivelog all;

RMAN:

select sequence#, completion_time, backup_count, deleted, status 
from v$archived_log 
where name = '&archivelogseqno';

	 SEQUENCE# COMPLETIO BACKUP_COUNT DEL S
	---------- --------- ------------ --- -
	    162723 23-OCT-06            0 YES D
	    162723 23-OCT-06            9 NO  A

SCENARIO:

Export: Release 10.2.0.3.0 - Production on Wed Feb 20 08:05:51 2008

Copyright (c) 1982, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Release 10.2.0.3.0 - 64bit Production
EXP-00056: ORACLE error 68 encountered
ORA-00068: invalid value 1000000 for parameter sort_area_size, must be between 19 and 1
ORA-06512: at "SYS.DBMS_EXPORT_EXTENSION", line 742
ORA-06512: at line 1
EXP-00056: ORACLE error 68 encountered
ORA-00068: invalid value 1000000 for parameter sort_area_size, must be between 19 and 1
ORA-06512: at "SYS.DBMS_EXPORT_EXTENSION", line 742
ORA-06512: at line 1
EXP-00000: Export terminated unsuccessfully
[Normal] From: ob2rman.exe@dsebux02 "dseprd"  Time: 02/20/08 08:05:51
        Starting post-exec command ( rm -f /tmp/$ORACLE_SID.rman_running).

ACTION
Check uptime of the database and when patch is applied
shutdown recman database
startup database
solve this problem

SCENARIO:

/opt/omni/lbin/ob2rman.exe[2]: /var/opt/omni/tmp/svrmgrl_daily_oracle_dse_dsebux02_dseprd.report: cannot create
[Major] From: ob2rman.exe@dsebux02 "dseprd"  Time: 02/20/08 13:41:36
        	Script 'obkbackup' failed to start.

Remove svrmgrl_daily_oracle_dse_dsebux02_dseprd.report entry (if exist) from /var/opt/omni/tmp/
---------------------------------------------------------------------------------------------------

Better to format date to check :
sys@mtpp> ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY:MM:DD:HH24:MI:SS';

[Major] From: ob2rman.exe@atlantis "delprd"  Time: 11/05/07 23:20:40 Post-exec command failed with status 127.

Check that it exist on ?tmp directory
find . -name '*running*'
./delprd.rman_running
Then 
Chk
select * from v$session  where client_info = 'id=mms_dp_rman';

if nothing relate to this <SID> then
rm -f /tmp/$ORACLE_SID.rman_running
```
```
RMAN> connect catalog rman/recman00@recman

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ============
RMAN-00571: ===========================================================
RMAN-04004: error from recovery catalog database: ORA-12154: TNS:could not resolve the connect identifier specified


Solution:

check where from listener is running.

check tnsentry for all oracle homes.

RMAN-06004: ORACLE error from recovery catalog database: 
RMAN-20004: target database name does not match name in recovery catalog

RMAN> list incarnation of database; 
RMAN> register database;

Recovery Manager complete.
oracle@vrpuxbne02.recman: /oracle 
> sql

SQL*Plus: Release 10.2.0.4.0 - Production on Wed Apr 1 08:37:12 2009

Copyright (c) 1982, 2007, Oracle.  All Rights Reserved.

Connected to:
Oracle Database 10g Enterprise Edition Release 10.2.0.4.0 - 64bit Production
With the Partitioning, Data Mining and Real Application Testing options

sys@vrpuxbne02.recman> select to_char(dbid) from v$database; 

TO_CHAR(DBID)
----------------------------------------
1446215485

1 row selected.

sys@vrpuxbne02.vicprd> select to_char(dbid) from v$database; 

TO_CHAR(DBID)
----------------------------------------
617731099

1 row selected.
========================Without incarnation===================

RMAN> connect target

connected to target database: VICPRD (DBID=617731099)

RMAN> list incarnation of database; 

using target database control file instead of recovery catalog

List of Database Incarnations
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
------- ------- -------- ---------------- --- ---------- ----------
1       1       VICPRD   617731099        PARENT  1851442    20-OCT-08
2       2       VICPRD   617731099        CURRENT 32062508   07-FEB-09


RMAN> connect target

connected to target database: VICPRD (DBID=617731099)

RMAN> connect catalog rman/recman00@recman;

connected to recovery catalog database

RMAN> unregister database;

database name is "VICPRD" and DBID is 617731099

Do you really want to unregister the database (enter YES or NO)? YES
database unregistered from the recovery catalog

RMAN> register database
2> ;

database registered in recovery catalog
starting full resync of recovery catalog
full resync complete

RMAN> list incarnation of database; 


List of Database Incarnations
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
------- ------- -------- ---------------- --- ---------- ----------
72989   73009   VICPRD   617731099        PARENT  1851442    20-OCT-08
72989   72990   VICPRD   617731099        CURRENT 32062508   07-FEB-09
============================================================================

RMAN-06004: ORACLE error from recovery catalog database: RMAN-20004: target database name does not match name in recovery catalog

SQL>  Select dbid,name from v$database ;

      DBID NAME
---------- ---------
4288714669 IJISPROD

SQL> Select * from v$database_incarnation ;


INCARNATION# RESETLOGS_CHANGE# RESETLOGS PRIOR_RESETLOGS_CHANGE# PRIOR_RES STATUS  RESETLOGS_ID PRIOR_INCARNATION# FLASHBACK_DATABASE_ALLOWED
------------ ----------------- --------- ----------------------- --------- ------- ------------ ------------------ --------------------------
           1         967935139 28-JAN-09                  880332 02-DEC-08 PARENT     677338620                  0 NO
           2         970463888 02-FEB-09               967935139 28-JAN-09 CURRENT    677774428                  1 NO

SQL> SQL> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.1.0.7.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle: ~]$sqlplus rman/rman@rman

SQL*Plus: Release 11.1.0.7.0 - Production on Tue Jan 8 08:42:06 2013

Copyright (c) 1982, 2008, Oracle.  All rights reserved.


Connected to:
Oracle9i Enterprise Edition Release 9.2.0.8.0 - 64bit Production
With the Partitioning and Oracle Data Mining options
JServer Release 9.2.0.8.0 - Production

SQL> Select distinct DBID from rc_database;

SQL> Select dbid,name ,DBINC_KEY,RESETLOGS_CHANGE# from rc_database ;


      DBID NAME      DBINC_KEY RESETLOGS_CHANGE#
---------- -------- ---------- -----------------
4081318632 PROD              2         415596757
4288714669 BPELPRD      475406        1524384976 ----- DBID is same for BPELPRD and IJISPROD
1484995170 FMWPROD      241647            880332
1450765346 FMAPPROD     242387            880332

SQL> Select dbid,name from v$database ;

      DBID NAME
---------- ---------
4288714669 BPELPRD
```
```

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 06/25/2009 09:26:40
RMAN-05501: aborting duplication of target database
RMAN-05517: temporary file /dbv6000/oracle/v6000/v6000_temp_02.dbf conflicts with file used by target database
RMAN-05517: temporary file /dbv6000/oracle/v6000/v6000_temp.dbf conflicts with file used by target database

Check tempfile information from production and put newname for auxiliary database

set newname for tempfile 1 to '/dbv6999/oracle/v6999/temp.dbf';
set newname for tempfile 2 to '/dbv6999/oracle/v6999/v6000_temp_02.dbf';

RMAN> show all;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of show command at 11/08/2011 01:35:50
RMAN-03014: implicit resync of recovery catalog failed
RMAN-03009: failure of full resync command on default channel at 11/08/2011 01:35:50
ORA-01580: error creating control backup file /u01/snapcf_ggdb1.f
ORA-27040: file create error, unable to create file
Linux-x86_64 Error: 13: Permission denied
Additional information: 1

RMAN> CONFIGURE SNAPSHOT CONTROLFILE NAME CLEAR;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of configure command at 11/08/2011 01:36:05
RMAN-03014: implicit resync of recovery catalog failed
RMAN-03009: failure of full resync command on default channel at 11/08/2011 01:36:05
ORA-01580: error creating control backup file /u01/snapcf_ggdb1.f
ORA-27040: file create error, unable to create file
Linux-x86_64 Error: 13: Permission denied
Additional information: 1

Solution: log in to the database as sysdba:

oracle@host: sqlplus "/ as sysdba"

SQL*Plus: Release 11.2.0.3.0 Production on Tue Nov 8 01:33:32 2011

Copyright (c) 1982, 2011, Oracle. All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, Oracle Label Security,
Data Mining and Real Application Testing options

sys@db1> EXECUTE SYS.DBMS_BACKUP_RESTORE.CFILESETSNAPSHOTNAME('/u01/app/oracle/product/11.2/db/dbs/snapcf_db1.f');

PL/SQL procedure successfully completed.

login to rman no catalog:

rman target / 
RMAN> CONFIGURE SNAPSHOT CONTROLFILE NAME TO 'G:\Database\Correct\path\SNCFX7AGK.ORA';


Logout from rman and login using catalog..then resync catalog.
--------------------------------------------------------------------------------------------------------------------------------------------------
```
```
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03015: error occurred in stored script arch_backup
RMAN-03009: failure of Control File and SPFILE Autobackup command on ORA_DISK_1 channel at 01/25/2016 08:43:50
ORA-19809: limit exceeded for recovery files
ORA-19804: cannot reclaim 2144337920 bytes disk space from 17179869184 limit

RMAN-08137: WARNING: archived log not deleted, needed for standby or upstream capture process

RMAN cannot delete old archived logs while extracts are working with the database and registered for logretention.
 
When GG extract is disabled the extract needs to be unregistered. Because this is not the case, archive deletion can't take place.
Solution is to unregister the extract:
 
SYS@pprod241 SQL> select capture_name, queue_owner, capture_user, start_scn, status from dba_capture;
 
CAPTURE_NAME              QUEUE_OWNER               CAPTURE_USER              START_SCN   STATUS
------------------------  ------------------------  ------------------------  ----------  ---------
CDC$C_MQ2_DELTA_TO_DW     DW_CDC_PUBLISHER          DW_CDC_PUBLISHER          5.9260E+12  DISABLED
CDC$C_MT_DELTA_TO_DW      DW_CDC_PUBLISHER          DW_CDC_PUBLISHER          5.9260E+12  DISABLED
 
SYS@pprod241 SQL> exec DBMS_CAPTURE_ADM.DROP_CAPTURE ('CDC$C_MQ2_DELTA_TO_DW');
SYS@pprod241 SQL> exec DBMS_CAPTURE_ADM.DROP_CAPTURE ('CDC$C_MT_DELTA_TO_DW');
SYS@pprod241 SQL> select capture_name, queue_owner, capture_user, start_scn, status from dba_capture;
 
no rows selected
 
After this the archives can be deleted as per normal without the force command.

```
```
RMAN-08137 --> this error will be reported for few reasons.
 
1) remote archive log destination did not receive or applied as per your ARCHIVE deletion policy (verify the below)
 
RMAN> show all;
CONFIGURE ARCHIVELOG DELETION POLICY TO  XXXXXX'
 
as per your configuration.
 
CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON ALL STANDBY;
 
You will have to verify all standby databases are upto date (sync with primary)....
 
SQL> select SEQUENCE#,thread#,FIRST_TIME,ARCHIVED,APPLIED,DELETED,STATUS from v$archived_log;
 
you can check, are there reported as applied for all archives.
 
2) you may have streams or golden gate configured, this may need these archives.
 
select capture_name, queue_owner, capture_user, start_scn, status from dba_capture;
select MIN_REQUIRED_CAPTURE_CHANGE# from v$database;
select SEQUENCE#,thread#,FIRST_TIME,ARCHIVED,APPLIED,DELETED,STATUS from v$archived_log where (select MIN_REQUIRED_CAPTURE_CHANGE# from v$database) between FIRST_CHANGE# and NEXT_CHANGE#;
 
you need to check this.
 
3) you may have flash backup enabled OR guarantee restore point created.
 
select * from GV_$RESTORE_POINT;
select flashback_on from v$database;

issue on the pprod24 is related to support note:
 
Bug 11872103 - RMAN RESYNC CATALOG very slow / V$RMAN_STATUS incorrectly shows RUNNING (Doc ID 11872103.8)

RMAN Fails with RMAN-10035, ORA-19550 When Using MTS (Doc ID 215934.1)

TESTR =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = CSMJIM-VODBQ05)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = MINESTAR)
    )
  )

```  
## Flashback 

```

-- Show Fast Recovery Area information
--
SET LINESIZE 145
SET PAGESIZE 9999

COLUMN name               FORMAT a15                  HEADING 'Name'
COLUMN space_limit        FORMAT 99,999,999,999,999   HEADING 'Space Limit'
COLUMN space_used         FORMAT 99,999,999,999,999   HEADING 'Space Used'
COLUMN space_used_pct     FORMAT 999.99               HEADING '% Used'
COLUMN space_reclaimable  FORMAT 99,999,999,999,999   HEADING 'Space Reclaimable'
COLUMN pct_reclaimable    FORMAT 999.99               HEADING '% Reclaimable'
COLUMN space_full         FORMAT 99,999,999,999,999   HEADING 'Space Full'
COLUMN pct_full           FORMAT 999.99               HEADING '% Full'
COLUMN number_of_files    FORMAT 999,999              HEADING 'Number of Files'

show parameter db_recovery_file_dest
set head off
show parameter control_file_record_keep_time
set head on

prompt
prompt Current location, disk quota, space in use, space reclaimable by deleting files,
prompt and number of files in the Flash Recovery Area.
prompt

SELECT
    name
  , space_limit
  , space_used
  , ROUND((space_used / space_limit)*100, 2) space_used_pct
  , space_reclaimable
  , ROUND((space_reclaimable / space_limit)*100, 2) pct_reclaimable
  , space_used - space_reclaimable space_full
  , ROUND(((space_used - space_reclaimable) / space_limit)*100, 2) pct_full
  , number_of_files
FROM
    v$recovery_file_dest
ORDER BY
    name
/
```
```

COLUMN file_type                  FORMAT a30     HEADING 'File Type'
COLUMN percent_space_used                        HEADING 'Percent Space Used'
COLUMN percent_space_reclaimable                 HEADING 'Percent Space Reclaimable'
COLUMN number_of_files            FORMAT 999,999 HEADING 'Number of Files'

SELECT
    file_type
  , percent_space_used
  , percent_space_reclaimable
  , number_of_files
FROM
    v$flash_recovery_area_usage
/
```
```
File Type                      Percent Space Used Percent Space Reclaimable Number of Files
------------------------------ ------------------ ------------------------- ---------------
CONTROL FILE                                  .03                         0               1
REDO LOG                                        0                         0               0
ARCHIVED LOG                                 3.62                      3.31              55
BACKUP PIECE                                 3.97                      1.24             131
IMAGE COPY                                      0                         0               0
FLASHBACK LOG                                6.66                      1.67              68
FOREIGN ARCHIVED LOG                            0                         0               0
```
```
set numformat 999999999999999999999999
select oldest_flashback_scn, oldest_flashback_time from v$flashback_database_log;
```
```
     OLDEST_FLASHBACK_SCN 	OLDEST_FLASHBACK_TI
------------------------- -------------------------------------
            6003442432117 	17-12-2015:12:56:52
```
```
show parameter retention
```
```
NAME                                 TYPE            VALUE
------------------------------------ --------------- --------------------------------------------------------------------------------
db_flashback_retention_target        integer         120
undo_retention                       integer         32400
```
```
select * from V$FLASHBACK_DATABASE_STAT;
```
```
BEGIN_TIME          END_TIME            FLASHBACK_DATA    DB_DATA  REDO_DATA ESTIMATED_FLASHBACK_SIZE
------------------- ------------------- -------------- ---------- ---------- ------------------------
17-12-2015:12:56:50 17-12-2015:13:56:56     7281827840 1.0949E+10 4168720896               1726502912
```
```
select applied,deleted,decode(rectype,11,'YES','NO') reclaimable ,count(*),min(sequence#),max(sequence#)
from v$archived_log left outer join sys.x$kccagf using(recid) 
where is_recovery_dest_file='YES' 
and name is not null 
group by applied,deleted,decode(rectype,11,'YES','NO') order by 5 
/ 
```
NOTE: The problem is there: Because of a bug (Bug 14227959 : STANDBY DID NOT RELEASE SPACE IN FRA) the archivelogs are not marked as reclaimable when the database is in mount mode.
The workaround is to execute dbms_backup_restore.refreshagedfiles. This is what must be scheduled (maybe daily) on the standby. It can be a good idea to do it at the same time as a daily ‘delete obsolete’, so here is the way to call it from RMAN:
RMAN> sql "begin dbms_backup_restore.refreshagedfiles; end;";
  
## Performance 

Inspect RMAN’s output messages to your terminal to identify your session identifier (SID). 
If you’re sending output to a log file, then look for the session ID in that file.
When you start an RMAN job, you should see output similar to this on your screen. In this
example, the SID is 146:
allocated channel: ORA_DISK_1
channel ORA_DISK_1: sid=146 devtype=DISK
Next, use the V$SESSION and V$PROCESS views to identify which database server
sessions correspond to RMAN channels:
```
SQL> SELECT b.sid, b.serial#, a.spid, b.client_info
 FROM v$process a, v$session b
 WHERE a.addr = b.paddr
 AND b.client_info LIKE '%rman%';
```
```
SID SERIAL# SPID CLIENT_INFO
---------- ---------- ------------ -------------------------
146 29 4376 rman channel=ORA_DISK_1
```

## Measuring Backup Performance
```
SQL> SELECT session_recid, input_bytes_per_sec_display,
 output_bytes_per_sec_display,
 time_taken_display, end_time
 FROM v$rman_backup_job_details
 ORDER BY end_time;
```
You should see output similar to the following:
```
SESSION_RECID INPUT_BYTES_PER OUTPUT_BYTES_PE TIME_TAKEN_DISPLAY END_TIME
------------- --------------- --------------- -------------------- ---------
1096 8.60M 7.69M 00:14:25 20-DEC-06
1101 1.88M 1.78M 00:42:03 21-DEC-06
1110 9.59M 8.56M 00:14:56 22-DEC-06
1114 9.75M 8.71M 00:14:52 23-DEC-06
1116 10.73M 9.58M 00:14:31 24-DEC-06
```
Incase - if you need to kill then - 
```
alter system kill session '&SID,&SERIAL' immediate;
```
##Monitoring RMAN Job Progress
```
select sid, serial#, sofar, totalwork, opname,
 round(sofar/totalwork*100,2) "% Complete"
 from v$session_longops
 where opname LIKE 'RMAN%'
 and opname NOT LIKE '%aggregate%'
 and totalwork != 0
 and sofar <> totalwork;
```
```
      SID    SERIAL#    CONTEXT      SOFAR  TOTALWORK %_complete
---------- ---------- ---------- ---------- ---------- ----------
       508          2          1      23862   15161290        .16
       507          3          1       9592   14710272        .07
       509          2          1      72564   14648017         .5
       510         18          1    5330620   14792964      36.03
```
```
select * from v$database_block_corruption;
```
You should now see some output similar to the following:
```
SID SERIAL# SOFAR TOTALWORK OPNAME % Complete
------ ------- ---------- ---------- ------------------------------ ----------
136 7 3259 51840 RMAN: full datafile backup 6.29
141 57 28671 74880 RMAN: full datafile backup 38.29
```

##Identifying I/O Bottlenecks

Query V$BACKUP_ASYNC_IO and V$BACKUP_SYNC_IO to determine I/O bottlenecks. Ideally,
the EFFECTIVE_BYTES_PER_SECOND column should return a rate that is close to the
capacity of the backup device. The following query returns statistics for asynchronous I/O for
backup and restore operations that have occurred within the last seven days:

PGA Memory
```
sho parameter disk_asynch_io;
```
```
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
disk_asynch_io                       boolean     TRUE
```
When asynchronous i/o is performed at the O/S level, the buffers needed by RMAN are
allocated from PGA. 

-- Query the pga memory usage by rman  session .
-- Note that pga consumption increases as backup progresses.
-- ALso note that size of buffers allocated = 18 MB (41-23) which is slightly > 16 MB
```
 col name for a30
     set line 500
     select s.sid, n.name , s.value/1024/1024 session_pga_mb
      from  v$statname n, v$sesstat s
      where s.sid = (select sess.SID
                     FROM V$PROCESS p, V$SESSION sess 
                     WHERE p.ADDR = sess.PADDR 
                       AND CLIENT_INFO LIKE '%rman%')
        and n.name = 'session pga memory'
        and s.statistic# = n.statistic#;
```
If OS does not support asynchronous I/O, we can simulate by setting parameter dbwr_io_slaves to a non zero value.  4 slave processes will be allocated irrespective of the value of the parameter dbwr_io_Slaves. IN this case, buffers for RMAN will be allocated from large pool. 
If large pool is sized to a value lower than the size of the buffers required, RMAN will switch to synchronous I/O and write a message to the alert log. 

-------------------------  
##Restore/Recovery 
-------------------------
Preview BACKUP Information
```
RMAN> restore database preview;
RMAN> restore database from tag TAG20060927T183743 preview;
RMAN> restore datafile 1, 2, 3, 4 preview;
RMAN> restore archivelog all preview;
RMAN> restore archivelog from time 'sysdate - 1' preview;
RMAN> restore archivelog from scn 3243256 preview;
RMAN> restore archivelog from sequence 29 preview;
RMAN> restore database preview summary;
RMAN> restore tablespace system preview;
RMAN> LIST BACKUP OF ARCHIVELOG TIME between "TO_DATE('08/04/2015 19:15:00', 'MM/DD/YYYY hh24:mi:ss')" and "TO_DATE('08/04/2015 22:30:00','MM/DD/YYYY hh24:mi:ss')";

RMAN> restore archivelog from sequence 207 until sequence 232 thread=2;
restore archivelog from sequence 10712 until lsequence 25324  thread=2;
```
cancelbased
RMAN> connect target /
RMAN> startup mount;
RMAN> restore database;                     # restore database from last backup

SQL> connect / as sysdba
SQL> recover database until cancel;
You will now be prompted by SQL*Plus to manually apply each archived redo log file. The
following is the prompt that you’ll get for each log file:
Specify log: {<RET>=suggested | filename | AUTO | CANCEL}

CANCEL
SQL> alter database open resetlogs;
```
## Based on SCN
Change/SCN based
You can also use set until scn within a run{} block to perform SCN-based incomplete
database recovery without having to repeat the SCN number for each command:
```
RMAN> connect target /
RMAN> startup mount;
RMAN> restore database until scn 950;
RMAN> recover database until scn 950;
RMAN> alter database open resetlogs;
```
```
RMAN> connect target /
RMAN> startup mount;
RMAN> run{
set until scn 950;
restore database;
recover database;
}
RMAN> alter database open resetlogs;
```

### Performing Log Sequence Based Recovery
The following example restores and recovers the target database up to, but not including,
log sequence number 50:
```
RMAN> connect target /
RMAN> startup mount;
RMAN> restore database until sequence 50;
RMAN> recover database until sequence 50;
RMAN> alter database open resetlogs;
```
## logsequencebased
```
RMAN> connect target /
RMAN> startup mount;
RMAN> restore database until sequence 50;
RMAN> recover database until sequence 50;
RMAN> alter database open resetlogs;
```
```
select sequence#, first_change#, first_time
from v$log_history
order by first_time;
```
## Timebased

Get the time period
----------------------------------------------
LIST BACKUP OF DATABASE COMPLETED BETWEEN '26-MAY-2010 00:00:00' AND '27-MAY-2010 10:00:00'; [7395743010602 21-JUN-2009 20:24:23]
LIST BACKUP OF ARCHIVELOG ALL COMPLETED BETWEEN '20-JUN-2009 20:00:00' AND '21-JUN-2009 21:50:00'; [7395743010909 21-JUN-2009 20:29:05]

set until scn 7395743010602; ## 21-JUN-2009 20:24:23
--- Run on Catalog Database -----------------
```
SELECT db.db_key, db.curr_dbinc_key, db.db_id
      FROM rman.db, rman.dbinc
      WHERE db.curr_dbinc_key = dbinc.dbinc_key
      AND dbinc.db_name   = 'EDPRD'; 
```

```
export NLS_DATE_FORMAT='DD-MON-YYYY HH24:MI:SS'
rman 
set DBID = 442659090 
connect target / 
connect catalog rman/recman00@recman
sql "alter session set nls_date_format=''DD-MON-YY HH24:MI:SS''";
run { 
set until time '18-APR-2009 20:19:44'; 
allocate channel ch1 type 'sbt_tape' parms 'SBT_LIBRARY=/usr/omni/lib/libob2oracle8_64bit.a';
allocate channel ch2 type 'sbt_tape' parms 'SBT_LIBRARY=/usr/omni/lib/libob2oracle8_64bit.a';
allocate channel ch3 type 'sbt_tape' parms 'SBT_LIBRARY=/usr/omni/lib/libob2oracle8_64bit.a';
allocate channel ch4 type 'sbt_tape' parms 'SBT_LIBRARY=/usr/omni/lib/libob2oracle8_64bit.a';
restore controlfile;
restore database;
}
```
```
RESTORE ARCHIVELOG FROM SEQUENCE 20 UNTIL SEQUENCE 28;
alter database open resetlogs;
```
```
Previous Incarnation
RMAN-03002: failure of restore command ...
RMAN-20207: UNTIL TIME or RECOVERY WINDOW is before RESETLOGS time

RMAN> connect target /
RMAN> startup nomount;
RMAN> restore controlfile from autobackup until time  "to_date('03-sep-2006 00:00:00', 'dd-mon-rrrr hh24:mi:ss')";
RMAN> alter database mount;
RMAN> list incarnation of database;
RMAN> reset database to incarnation 1;
RMAN> restore database until time  "to_date('03-sep-2006 00:00:00', 'dd-mon-rrrr hh24:mi:ss')";
RMAN> recover database until time  "to_date('03-sep-2006 00:00:00', 'dd-mon-rrrr hh24:mi:ss')";
RMAN> alter database open reset logs;
The database is now as it was on September 3, 2006 at 12 a.m.
```
```
RMAN> connect target
RMAN> connect catalog rman/recman00@recman
RMAN> List Incarnation of database victst1;
RMAN> reset database to incarnation 527589; [RESET DATABASE TO INCARNATION <inc Key>;   ]
database reset to incarnation 527589

Check the SCN number to recovery for point in time 

RMAN> connect target
connected to target database: VICTST1 (DBID=2859628576)
RMAN> connect catalog rman/recman00@recman
connected to recovery catalog database
RMAN> sql "alter session set nls_date_format=''DD-MON-YY HH24:MI:SS''";
sql statement: alter session set nls_date_format=''DD-MON-YY HH24:MI:SS''
12c: RMAN> alter session set nls_date_format="DD-MON-YY HH24:MI:SS";

LIST BACKUP OF DATABASE COMPLETED BETWEEN '25-MAR-2009 14:10:00' AND '25-MAR-2009 15:10:00';

list backup of archivelog all completed between '25-MAR-2009 10:10:00' AND '25-MAR-2009 15:10:00';

RMAN>LIST BACKUP OF DATABASE COMPLETED BETWEEN '01-APR-2007' AND '08-MAY-2007';
 
      # find the last archive log backed up on the 24th MArch ( NOTE: Specific Date as you want)

      # take the value of the NEXT_SCN from that one and subtract 1

RMAN> LIST INCARNATION OF DATABASE DBA10;
RMAN> RESET DATABASE TO INCARNATION  464 ;
RMAN> restore database;
RMAN> alter database open resetlogs;
```

##redo log failureORA-00312

Solution
Follow these steps when dealing with online redo log file failures:
1. Inspect the alert.log file to determine which online redo log files have experienced a media failure.
2. Query V$LOG and V$LOGFILE to determine the status of the log group and degree of multiplexing.

Query V$LOG and V$LOGFILE views to determine the status of your log group and the
member files in each group:
```
select
 a.group#, a.thread#,
 a.status grp_status,
 b.member member,
 b.status mem_status
 from v$log a,
 v$logfile b
 where a.group# = b.group#
 order by a.group#, b.member;
```
```
Type of Failure                         Status Column of V$LOG                                        Action                                            Recipe
======================================================================================================
One member failed in                              N/A                                                        Re-create member.                      Recipe 14-2
```
```
Restoring After Losing One Member of the Multiplexed Group Problem
You notice this message in your alert.log file:
Errors in file c:\oracle\product\10.2.0\admin\orcl\bdump\orcl_lgwr_5800.trc:
ORA-00321: log 1 of thread 1, cannot update log file header
ORA-00312: online log 1 thread 1:'C:\ORACLE\PRODUCT\10.2.0\ORADATA\ORCL\REDO01B.LOG'
```
#### All Members drop ACTIVE log group
```
1. Verify the damage to the members.
2. Verify that the status is ACTIVE.
3. Attempt to issue a checkpoint.
4. If the checkpoint is successful, the status should now be INACTIVE, and you can clear
the log group.
5. If the log group that was cleared was unarchived, back up your database immediately.
6. If the checkpoint is unsuccessful, then you will have to perform incomplete recovery
(SQL> alter database clear unarchived logfile group <group#>;).

SQL> connect / as sysdba
SQL> startup mount;
SQL> select group#, status, archived, thread#, sequence# from v$log;

SQL> alter system checkpoint;
SQL> alter database clear logfile group <group#>;
SQL> alter database clear unarchived logfile group <group#>;
```
Example
```
RMAN> select GROUP#, STATUS, FIRST_CHANGE# from v$log;

    GROUP# STATUS           FIRST_CHANGE#
---------- ---------------- -------------
         1 INACTIVE              19063768
         2 ACTIVE                19063819
         3 CURRENT               19077852

RMAN> alter database drop logfile group 2;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of sql statement command at 12/02/2013 15:16:55
ORA-01624: log 2 needed for crash recovery of instance MONDB1 (thread 1)
ORA-00312: online log 2 thread 1: '/BPELAIT2/MONDB1/MONDB1/redo02.log'
RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         67
         3 CURRENT          NO           1         69
         2 INACTIVE         YES          1         68

RMAN> alter system checkpoint;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of sql statement command at 12/02/2013 15:18:38
ORA-01109: database not open

RMAN> alter database open;

Statement processed

RMAN> alter system checkpoint;

Statement processed

RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         67
         2 INACTIVE         YES          1         68
         3 CURRENT          NO           1         69

RMAN> alter system checkpoint;

Statement processed

RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         67
         2 INACTIVE         YES          1         68
         3 CURRENT          NO           1         69

RMAN> alter database clear logfile group 2;

Statement processed

RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#

---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         67
         2 UNUSED           YES          1          0
         3 CURRENT          NO           1         69

RMAN> alter system switch logfile;

Statement processed

RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         67
         2 CURRENT          NO           1         70
         3 ACTIVE           YES          1         69
```

### All members loss Current log group

Perform an incomplete recovery up to the last good SCN.
If flashback is enabled, flash your database back to the last good SCN.
If you’re using Oracle Data Guard, fail over to your physical or logical standby database.
Contact Oracle Support for suggestions.
In preparation for an incomplete recovery, first

```
shutdown immediate;
startup mount;
select group#, status, archived, thread#, sequence#, first_change# from v$log;

GROUP# STATUS ARC THREAD# SEQUENCE# FIRST_CHANGE#
------ -------- --- ------- ---------- -------------
1 INACTIVE YES 1 50 1800550
2 INACTIVE YES 1 49 1800468
3 CURRENT NO 1 51 1800573 ------------------- upto but not including this SCN

RMAN> restore database until scn 1800573;
RMAN> recover database until scn 1800573;
RMAN> alter database open resetlogs;
```

------TEST
```
RMAN> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 CURRENT          NO           1         71
         2 INACTIVE         YES          1         70
         3 INACTIVE         YES          1         69

RMAN> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/BPELAIT2/MONDB1/MONDB1/redo03.log
/BPELAIT2/MONDB1/MONDB1/redo02.log
/BPELAIT2/MONDB1/MONDB1/redo01.log
/BPELAIT2/MONDB1/MONDB1/redo03b.log

$ mv /BPELAIT2/MONDB1/MONDB1/redo01.log /BPELAIT2/MONDB1/MONDB1/redo01o.log

RMAN> startup mount;

Oracle instance started
database mounted

Total System Global Area     626507776 bytes

Fixed Size                     2281600 bytes
Variable Size                360714112 bytes
Database Buffers             255852544 bytes
Redo Buffers                   7659520 bytes

RMAN>  select group#, status, archived, thread#, sequence#, first_change# from v$log;

using target database control file instead of recovery catalog
    GROUP# STATUS           ARC    THREAD#  SEQUENCE# FIRST_CHANGE#
---------- ---------------- --- ---------- ---------- -------------
         1 INACTIVE         NO           1         71      19079753
         3 INACTIVE         NO           1         72      19081512
         2 CURRENT          NO           1         73      19081650

RMAN> restore database until scn 19081650;

Starting restore at 02-DEC-13
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=277 device type=DISK
flashing back control file to SCN 19081650

skipping datafile 5; already restored to file /BPELAIT2/MONDB1/MONDB1/pdbseed/system01.dbf
skipping datafile 7; already restored to file /BPELAIT2/MONDB1/MONDB1/pdbseed/sysaux01.dbf
channel ORA_DISK_1: restoring datafile 00001
input datafile copy RECID=7 STAMP=833120933 file name=/BPELAIT2/MONDB1/MONDB1/system01.dbf01
destination for restore of datafile 00001: /BPELAIT2/MONDB1/MONDB1/system01.dbf
channel ORA_DISK_1: copied datafile copy of datafile 00001
output file name=/BPELAIT2/MONDB1/MONDB1/system01.dbf RECID=0 STAMP=0
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00003 to /BPELAIT2/MONDB1/MONDB1/sysaux01.dbf
channel ORA_DISK_1: restoring datafile 00004 to /BPELAIT2/MONDB1/MONDB1/undotbs01.dbf
channel ORA_DISK_1: restoring datafile 00006 to /BPELAIT2/MONDB1/MONDB1/users01.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/28oqgpoc_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/28oqgpoc_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:25
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00021 to /BPELAIT2/MONDB1/MONDB1/CDB1/testpdb/system01.dbf
channel ORA_DISK_1: restoring datafile 00022 to /BPELAIT2/MONDB1/MONDB1/CDB1/testpdb/sysaux01.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/29oqgpor_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/29oqgpor_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:15
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00023 to /BPELAIT2/MONDB1/MONDB1/CDB2/MONDB1/E7DFA21380C174CCE044002128C4E19/datafile/o1_mf_system_94w01ffy_.dbf
channel ORA_DISK_1: restoring datafile 00024 to /BPELAIT2/MONDB1/MONDB1/CDB2/MONDB1/E7DFA21380C174CCE044002128C4E19/datafile/o1_mf_sysaux_94w01glc_.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/2aoqgpp2_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/2aoqgpp2_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:15
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00026 to /BPELAIT2/MONDB1/MONDB1/CDB2/tstpd01.dbf
channel ORA_DISK_1: restoring datafile 00027 to /BPELAIT2/MONDB1/MONDB1/CDB1/testpdb/test01.dbf
channel ORA_DISK_1: restoring datafile 00028 to /BPELAIT2/MONDB1/MONDB1/CDB1/testpdb/users01.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/2coqgpq1_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/2coqgpq1_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00029 to /BPELAIT2/MONDB1/MONDB1/CDB2/MONDB1/users02.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/2doqgpq2_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/2doqgpq2_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00025 to /BPELAIT2/MONDB1/MONDB1/mon01.dbf
channel ORA_DISK_1: reading from backup piece /BPELAIT2/MONDB1/BKP/2eoqgpq4_1_1
channel ORA_DISK_1: piece handle=/BPELAIT2/MONDB1/BKP/2eoqgpq4_1_1 tag=TAG20131202T141331
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 02-DEC-13

RMAN> recover database until scn 19081650;

Starting recover at 02-DEC-13
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 68 is already on disk as file /BPELAIT2/MONDB1/MONDB1/arch/1_68_828368340.dbf
archived log for thread 1 with sequence 69 is already on disk as file /BPELAIT2/MONDB1/MONDB1/arch/1_69_828368340.dbf
archived log for thread 1 with sequence 70 is already on disk as file /BPELAIT2/MONDB1/MONDB1/arch/1_70_828368340.dbf
archived log for thread 1 with sequence 72 is already on disk as file /BPELAIT2/MONDB1/MONDB1/redo03.log
archived log file name=/BPELAIT2/MONDB1/MONDB1/arch/1_68_828368340.dbf thread=1 sequence=68
archived log file name=/BPELAIT2/MONDB1/MONDB1/arch/1_69_828368340.dbf thread=1 sequence=69
archived log file name=/BPELAIT2/MONDB1/MONDB1/arch/1_70_828368340.dbf thread=1 sequence=70
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 12/02/2013 16:06:39
ORA-00283: recovery session canceled due to errors
RMAN-11003: failure during parse/execution of SQL statement: alter database recover logfile '/BPELAIT2/MONDB1/MONDB1/arch/1_70_828368340.dbf'
ORA-00283: recovery session canceled due to errors
ORA-00313: open failed for members of log group 1 of thread 1
ORA-00312: online log 1 thread 1: '/BPELAIT2/MONDB1/MONDB1/redo01.log'
ORA-27037: unable to obtain file status
SVR4 Error: 2: No such file or directory
Additional information: 3
$ mv /BPELAIT2/MONDB1/MONDB1/redo01o.log /BPELAIT2/MONDB1/MONDB1/redo01.log

RMAN> recover database until scn 19081650;

Starting recover at 02-DEC-13
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 02-DEC-13
```

## All Members of the INACTIVE
```
Database mounted.
ORA-00313: open failed for members of log group 1 of thread 1
ORA-00312: online log 1 thread 1:'C:\ORACLE\PRODUCT\10.2.0\ORADATA\ORCL\REDO01.LOG'
ORA-00312: online log 1 thread 1:'C:\ORACLE\PRODUCT\10.2.0\ORADATA\ORCL\REDO01B.LOG'

1. Verify that all members of a group have been damaged.
2. Verify that the log group status is INACTIVE.
3. Re-create the log group with the clear logfile command.
4. If the re-created log group has not been archived, then immediately back up your
database.

SQL> connect / as sysdba
SQL> startup mount;

verify that the damaged log group is INACTIVE and determine whether it has been archived
===============================================================================
SQL> select group#, status, archived, thread#, sequence# from v$log;

If the status is INACTIVE, then this log group is no longer needed for crash recovery

SQL> alter database clear logfile group 1; [clear logfile will drop and re-create all members of a log group for you]

If the log group has not been archived
SQL> alter database clear unarchived logfile group 1;

### DROP AND ADD LOG GROUP

A log group has to have an inactive status before you can drop it. 

SQL> select group#, status, archived, thread#, sequence# from v$log;

If you attempt to drop the current online log group, Oracle will return an ORA-01623 error stating that you cannot drop the current group. 

alert system switch logfile;
 to switch the logs and make the next group the current group.
After a log switch, the log group that was previously the current group will retain an active status as long as it contains redo that Oracle requires to perform crash recovery. If you attempt to drop a log group
with an active status, Oracle will throw an ORA-01624 error stating that the log group is required for crash recovery. 
alter system checkpoint;             ---------- command to make the log group inactive.

Additionally, you cannot issue a drop logfile group command if it leaves you with only one log group left in your database. If you attempt to do this, Oracle will throw an ORA-01567 error and inform you
that dropping the log group is not permitted because it would leave you with less than two logs groups for your database (Oracle minimally requires two log groups to function). 
SQL> alter database drop logfile group <group #>;
You can add a new log group with the add logfile group command:

SQL> alter database add logfile group <group_#> ('\directory\file') SIZE <bytes> K|M;

You can specify the size of the log file in bytes, kilobytes, or megabytes. The following example adds log
group 4 with two members sized at 50MB:
SQL> alter database add logfile group 4
 ('C:\ORADATA\ORCL\REDO04A.LOG',
 'D:\ORADATA\ORCL\REDO04B.LOG') SIZE 50M;
```

### One Member of the Multiplexed Group

Errors in file c:\oracle\product\10.2.0\admin\orcl\bdump\orcl_lgwr_5800.trc:
ORA-00321: log 1 of thread 1, cannot update log file header
ORA-00312: online log 1 thread 1:'C:\ORACLE\PRODUCT\10.2.0\ORADATA\ORCL\REDO01B.LOG'

```
1. Identify the online redo log file experiencing media failure.
2. Ensure that the online redo log file is not part of the current online log group.
3. Drop the damaged member.
4. Add a new member to the group.
```

Once you’ve identified the bad online redo log file, execute the following query to check
whether that online redo log file’s group has a CURRENT status:
109
■¡Note T he only tool provided by Oracle that can protect you and preserve all committed transactions in the event you
lose all members of the current online redo log group is Oracle Data Guard, implemented in maximum protection mode.
See MOS note 239100.1 for more details regarding Oracle Data Guard protection modes.
The online redo log files are never backed up by an RMAN backup or by a user-managed hot backup. If you did
back up the online redo log files, it would be meaningless to restore them. The online redo log files contain the latest
redo generated by the database. You wouldn’t want to overwrite them from a backup with old redo information.
For a database in archivelog mode the online redo log files contain the most recently generated transactions that
are required to perform a complete recovery.
Displaying Online Redo Log Information
Use the V$LOG and V$LOGFILE views to display information about online redo log groups and corresponding members:
```
COL group# FORM 99999
COL thread# FORM 99999
COL grp_status FORM a10
COL member FORM a30
COL mem_status FORM a10
COL mbytes FORM 999999
--
SELECT
a.group#
,a.thread#
,a.status grp_status
,b.member member
,b.status mem_status
,a.bytes/1024/1024 mbytes
FROM v$log a,
v$logfile b
WHERE a.group# = b.group#
ORDER BY a.group#, b.member;
```

If the failed member is in the current log group, then use the alter system switch
logfile command to make the next group the current group. Then drop the failed member
as follows:
SQL> alter database drop logfile member '<\directory\member>';
Then re-create the online redo log file member:
SQL> alter database add logfile member '<\new directory\member>' to group <group#>;
If an unused log file already happens to exist in the target location, you can use the reuse
parameter to overwrite and reuse that log file. The log file must be the same size as the other
log files in the group.
SQL> alter database add logfile member '\directory\member>' reuse to group <group#>;

## POINT INTIME RECOVERY
```
rman target /
Find out  Piece Name which contained -   Control File Included: 
show controlfile autobackup;
export NLS_DATE_FORMAT='DD-MON-YY HH24:MI:SS'
RMAN> catalog start with 'F:\Restored Backup from 8pm Sunday';
--

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
849     Full    1.20M      DISK        00:00:00     14-NOV-15
        BP Key: 849   Status: AVAILABLE  Compressed: YES  Tag: TAG20151114T200006
        Piece Name: F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PDQM91VA_1_1_20151114
  Control File Included: Ckp SCN: 84093477     Ckp time: 14-NOV-15

RMAN> shutdown immediate;

RMAN> startup nomount;
Restore the controlfile from autobackup:
RMAN> restore controlfile from  'F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PDQM91VA_1_1_20151114';

RMAN> sql 'alter database mount';
rman target /
catalog start with 'F:\Restored Backup from 8pm Sunday';

restore database;
RMAN-00571: ===========================================================
RMAN-03002: failure of restore command at 11/18/2015 08:15:51
ORA-19693: backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PLQMBMA8_1_1_20151115 already included


Manually catalog all the required backup pieces ---
RMAN> catalog backuppiece 'F:\Restored Backup from 8pm Sunday\QMJS1_ARCHIVELOG_BKP_PIQM921E_1_1_20151114';

verify:
RMAN> RESTORE DATABASE VALIDATE;

Starting restore at 18-NOV-15
using channel ORA_DISK_1

channel ORA_DISK_1: starting validation of datafile backup set
channel ORA_DISK_1: reading from backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PLQMBMA8_1_1_20151115
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of restore command at 11/18/2015 08:46:26
ORA-19870: error while restoring backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PLQMBMA8_1_1_20151115
ORA-19872: Unexpected end of file at block 12033 while decompressing backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PLQMBMA8_1_1_20151115

NOTE: This happened as more backup piece was cataloged and mismatch with the controlfile header. [restored from 14th backup but it was reading 15th backup]

Then reconnect RMAN session | restored controlfiles | mount | cataloged only required backup pieces

RMAN> restore database;

Starting restore at 18-NOV-15
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00003 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\O1_MF_UNDOTBS1_BVXNV5FQ_.DBF
channel ORA_DISK_1: restoring datafile 00007 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_ECF02.DBF
channel ORA_DISK_1: restoring datafile 00013 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_IDXENT02.DBF
channel ORA_DISK_1: restoring datafile 00014 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_IDXOBJ01.DBF
channel ORA_DISK_1: restoring datafile 00015 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_IDXOBJ02.DBF
channel ORA_DISK_1: restoring datafile 00016 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_OBJ01.DBF
channel ORA_DISK_1: reading from backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PAQM91U7_1_1_20151114
channel ORA_DISK_1: piece handle=F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PAQM91U7_1_1_20151114 tag=TAG20151114T
200006
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:01:05
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00002 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\O1_MF_SYSAUX_BVXNTWP1_.DBF
channel ORA_DISK_1: restoring datafile 00005 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MS_GG_TB.DBF
channel ORA_DISK_1: restoring datafile 00006 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_ECF01.DBF
channel ORA_DISK_1: restoring datafile 00008 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_ENT01.DBF
channel ORA_DISK_1: restoring datafile 00011 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_IDXECF02.DBF
channel ORA_DISK_1: restoring datafile 00012 to E:\MSTARDATA\ORADATA\QMJS1_01\DATAFILE\MW_IDXENT01.DBF
channel ORA_DISK_1: reading from backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_FULL_DATABASE_BKP_PBQM91U7_1_1_20151114
channel ORA_DISK_1: piece handle=F:\RESTORED BACKUP FROM 8PM 
--
--
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:01:25
Finished restore at 18-NOV-15

recover database;
-----
channel ORA_DISK_1: starting archived log restore to default destination
channel ORA_DISK_1: restoring archived log
archived log thread=1 sequence=9026
channel ORA_DISK_1: reading from backup piece F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_ARCHIVELOG_BKP_PIQM921E_1_1_20151114
channel ORA_DISK_1: piece handle=F:\RESTORED BACKUP FROM 8PM SUNDAY\QMJS1_ARCHIVELOG_BKP_PIQM921E_1_1_20151114 tag=TAG20151114
113
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
archived log file name=F:\MSTARDATA\FAST_RECOVERY_AREA\QMJS1_01\ARCHIVELOG\2015_11_18\O1_MF_1_9026_C4QMFXYR_.ARC thread=1 sequ
=9026
channel default: deleting archived log(s)
archived log file name=F:\MSTARDATA\FAST_RECOVERY_AREA\QMJS1_01\ARCHIVELOG\2015_11_18\O1_MF_1_9026_C4QMFXYR_.ARC RECID=9032 ST
896086846
unable to find archived log
archived log thread=1 sequence=9027
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 11/18/2015 09:00:47
RMAN-06054: media recovery requesting unknown archived log for thread 1 with sequence 9027 and starting SCN of 84094893

RMAN> sql 'alter database open resetlogs';
sql statement: alter database open resetlogs
```
```
RMAN> run {
set until time '16-NOV-2015 11:00:00';  
restore database;
recover database;
}
```
  
## unregister 
```
set lines 132
set pages 60
alter session set nls_date_format = 'DD-MON-YY HH24:MI';

select i.DB_NAME "DB Name"
      ,i.DB_KEY "DB Key"
      ,db.DB_ID "DB ID"
      ,i.DBINC_KEY "DB Incarnation"
      ,i.RESET_TIME "Reset Time"
      ,bsf.min_time "First Full"
      ,bsf.max_time "Last Full"
      ,bsa.min_time "First Arch"
      ,bsa.max_time "Last Arch"
from dbinc i
    ,db
    ,(select DB_KEY
            ,BCK_TYPE
            ,min(COMPLETION_TIME) min_time
            ,max(COMPLETION_TIME) max_time
      from bs
      where BCK_TYPE != 'L'
      group by DB_KEY, BCK_TYPE) bsf
    ,(select DB_KEY
            ,BCK_TYPE
            ,min(COMPLETION_TIME) min_time
            ,max(COMPLETION_TIME) max_time
      from bs
      where BCK_TYPE = 'L'
      group by DB_KEY, BCK_TYPE) bsa
where i.DB_KEY = db.DB_KEY
  and i.DB_KEY = bsa.DB_KEY(+)
  and i.DB_KEY = bsf.DB_KEY(+)
  and i.DBINC_STATUS = 'CURRENT'
order by i.DB_NAME, i.DB_KEY, db.DB_ID, i.RESET_TIME
/

prompt Enter Database Key and Id to remove database from catalog.

EXECUTE dbms_rcvcat.unregisterdatabase(&db_key, &db_id);
```
```
1. Connect both to the recovery catalog and to the target database:
$ rman target / catalog rman/rman@catdb
RMAN>
2. Issue the unregister database command to unregister the target database to which you’re currently connected:
RMAN> unregister database;
database name is "TENNER" and DBID is 922224687
Do you really want to unregister the database (enter YES or NO)? yes
database unregistered from the recovery catalog

It is further possible that you might have another database of the same name, perhaps because it is running on a different server. In such a case, use the set dbid command to specify the particular database that you want to unregister. The following example shows how to remove a specific database named testdb when multiple databases named testdb are registered:

RMAN> run
{set dbid 1234567899;
unregister database testdb;
}
```
###################################################
## Unregister a Database From the Recovery Catalog 
###################################################
```
Taken from Metalink: Doc ID: Note:1058332.6 
This section describes the steps on how to remove (unregister) a database (the target database) from the recovery catalog. 

You can unregister a database by running the following procedure from the while logged into the recovery catalog: 

SQL> execute dbms_rcvcat.unregisterdatabase(db_key, db_id);

To unregister a database, do the following: 

NOTE: If the target database does not exists anymore, the only steps to execute are (1) and (3). Because the backupsets cannot be deleted from the catalog (requires to connect to the target database) they are not be deleted from disk or tape either. So you have to remove these backupsets manually. A list of the related backupsets are: 
SQL> select handle from rc_backup_piece where db_id = <see step (1)>;
 
Identify the database that you want to unregister. Run the following query from the recovery catalog using Server Manager or SQL*Plus (connected as the RMAN user): 
SQL> select * from rc_database ;

    DB_KEY  DBINC_KEY       DBID 		NAME     RESETLOGS_CHANGE# RESETLOGS
---------- ---------- ---------- -------- ----------------- ---------
         1          2 		2498101982 	TARGDB                 1 	15-JAN-04
       105        106 		2457750772 	OIDDB                     1 	14-DEC-03
       128        129 		2351019032 	OMSDB                    1 	15-JAN-04
       301        302 		2498937635 	TARGDB              	140831 	25-JAN-04

SQL> set lines 80
SQL> set pages 250
SQL> ttitle "DB_KEY and DBID Info........"
SQL> select * from rc_database;

Thu Jan 03                                                             page    1
                          DB_KEY and DBID Info........

    DB_KEY  DBINC_KEY       DBID 		NAME           RESETLOGS_CHANGE#      RESETLOGS
---------- ---------- ---------- -------- ----------------- ------------------------------------------------------------------------------------------------------------
         1          2 		4081318632 	PROD                415596757                         13-AUG-05
    454744     454745 	4288714669 	IJISPROD          970463888                          02-FEB-09
    241646     241647 	1484995170 	FMWPROD             880332                          18-AUG-10
    242386     242387 	1450765346 	FMAPPROD            880332                          08-SEP-10

For this example, I want to unregister all databases with this catalog. 

Remove the backupsets that belong to the database that you want to unregister. 

Find the backupsets of the database that you want to unregister. 
RMAN> list backupset of database;

Remove the backupsets that belongs only to the database you want to unregister. 
RMAN> allocate channel for delete type disk;
RMAN> change backupset XXX delete;
NOTE: You need to allocate a channel for the delete. In this example a disk drive is being used and not a tape. The procedure for a backup done to tape is the same except you have to allocate a different channel for tape. Example: 
RMAN> allocate channel for delete type 'sbt_tape';
The XXX value is the 'list of key' value from the 'list backupset of database' command  

Unregister the database by executing the following procedure from the recovery catalog: 
SQL> execute dbms_rcvcat.unregisterdatabase(db_key, db_id)

The "db_key" and "db_id" values you will get by running the following query from the recovery catalog: 
SQL> select * from rc_database;

Unregister: 
Make sure you are using the correct values by looking at the 'NAME' column of the "rc_database" table. Here is an example of how to unregister all databases within this catalog: 
SQL> execute dbms_rcvcat.unregisterdatabase(1, 4081318632)

PL/SQL procedure successfully completed.

 ```

## verifying Integrity& BlockRecovery 

```
RMAN> restore database validate check logical;
RMAN> validate backupset 193;
RMAN> validate backupset 193 check logical;
=================
Error reported by user pointing to block corruption.

 POPULATE_MACSDATA - ORA-01578: ORACLE data block corrupted
(file # 48, block # 142713)
ORA-01110: data file 48: '/hqlinux08db06/ORACLE/macsl/MACSDAT_2006_06.dbf'
ORA-02063: preceding 2 lines from MODSL_MACSL_LINK
File name : /hqlinux08db06/ORACLE/macsl/MACSDAT_2006_06.dbf

Check first if the there is only one(few) blocks corrupted or most of the blocks are corrupted.

Issue command below at UNIX prompt.
=====================================
dbv file=<file_name> BLOCKSIZE=8192 LOGFILE=test.log

> vi test.log
You can get the list of corrupted blocks from  v$database_block_corruption
ORA-01578: ORACLE data block corrupted (file # 11, block # 84)
ORA-01110: data file 4: '/oradata/users.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
ORA-1578 / ORA-26040 Corrupt blocks by NOLOGGING - Error explanation and solution (Doc ID 794505.1)
SOLUTION
Note that the data inside the affected blocks is not salvageable. Methods like "Media Recovery" or "RMAN blockrecover" will not fix the problem unless the data file was backed up after the NOLOGGING operation was registered in the Redo Log.
Is error after RMAN DUPLICATE?
If the error is after a RMAN DUPLICATE or RESTORE, enable FORCE LOGGING at SOURCE database and perform the DUPLICATE or RESTORE (after new BACKUP) steps again:
alter database force logging; 
Is error produced in a PHYSICAL STANDBY Database?
If the error is produced in a PHYSICAL STANDBY database, the option is to restore the affected file from the PRIMARY database (only if the problem is not present in the PRIMARY) and to avoid the problem from being introduced there is the option to force logging in the PRIMARY database with:
alter database force logging;
=============
> validate database;                   
>validate check logical database;

Output:
:
:
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
5    OK     3              1            XXXXXX          XXXXXX


recover datafile 1 block 101760;

SQL> select * from v$database_block_corruption;
 
SYS@DRAH2_1 SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTION_TYPE
---------- ---------- ---------- ------------------ ---------------------------
         1     101819          3                  0 CORRUPT
         1     101823         65                  0 CORRUPT
         1     101761         56                  0 CORRUPT


$ rman target /

RMAN> recover datafile 5 block 130 to 131,142;

verify
=============
$ sqlplus / as sysdba

SQL> select * from v$database_block_corruption;

If you have multiple block list as corrupt, You can use single command to recover them.

RMAN> BLOCKRECOVER corruption list;
========================================================
corrupt block in a large datafile
SQL> Select * from v$database_block_corruption;

RMAN> recover corruption list;
RMAN> recover datafile 5 block 24;
RMAN> recover datafile 7 block 22 datafile 8 block 43;
RMAN> recover datafile 5 block 24 from tag=tues_backup;
RMAN> recover datafile 6 block 89 restore until sequence 546;
RMAN> recover datafile 5 block 32 restore until 'sysdate-1';
RMAN> recover datafile 5 block 65 restore until scn 23453;

Once you issue the advise failure command, you can query the V$IR_
MANUAL_CHECKLIST view to examine the recommended repair, as shown here:

SQL> select advise_id, rank, message from v$ir_manual_checklist;
ADVISE_ID  	RANK  		MESSAGE
------ ------ -------------------------------------------
21 		0 		if file /u01/app/oracle/nick/users01.dbf
				was unintentionally renamed or moved,
				restore it
21 		 0 		if file /u01/app/oracle/nick/example01.dbf
				was unintentionally renamed or moved,
					restore it

fix this problem by using the Data Recovery Advisor as follows:

ORA-01110: data file 4: 'C:\ORACLE\PRODUCT\10.2.0\ORADATA\NICK\USERS01.DBF'

SQL> select * from v$database_block_corruption ; 

Checker run found 1 new persistent data failures
$ rman target /
RMAN> recover corruption list;
>backup check logical validate database;
SQL> select * from V$DATABASE_BLOCK_CORRUPTION;

Error backing up file 8, block 1006: logical corruption
Error backing up file 9, block 2072: logical corruption
:
:
:
Error backing up file 14, block 10328: logical corruption
Error backing up file 8, block 1010: logical corruption

DEBUG
---------------
SQL> oradebug setmypid 
Statement processed. 
SQL> alter session set tracefile_identifier='BLOCKSCORR1_DUMP'; 
SQL> alter system dump datafile '/IJISPROD/oradata/interface_data07.dbf' block min 5000 block max 8010; 
SQL> oradebug tracefile_name 
/usr/app/oracle/diag/rdbms/ijisprod/IJISPROD/trace/IJISPROD_ora_2654_BLOCKSCORR1_DUMP.trc 
SQL> select value from v$diag_info where name='Default Trace File'; 

VALUE 
-------------------------------------------------------------------------------- 
/usr/app/oracle/diag/rdbms/ijisprod/IJISPROD/trace/IJISPROD_ora_2654_BLOCKSCORR1_DUMP.trc 

SQL>Select segment_name,segment_type,owner from dba_extents 
where file_id=8 and 1006 between block_id and block_id + blocks -1 ; 
--
SQL>Select segment_name,segment_type,owner from dba_extents 
where file_id=14 and 10328 between block_id and block_id + blocks -1 ; 

For the table on which we see Logical corruption 

Can you check the following :- 

SQL>Select table_name, DEPENDENCIES,owner from dba_tables where table_name='<name of table occuyping corrupted block>' ; 

SQL> Select segment_name,segment_type,owner from dba_extents 
where file_id=14 and 10328 between block_id and block_id + blocks -1 ; 

SEGMENT_NAME              SEGMENT_TYPE OWNER 
------------------ ------------------------------ 
CASE_FILE_CAFH            TABLE QHUB 

SQL> SQL> Select table_name, DEPENDENCIES,owner from dba_tables where table_name='QHUB.CASE_FILE_CAFH'; 

no rows selected 

- For the LOB 
SYS_LOB0002427660C00003$$ 

Please follow the Note 452341.1 and try to identify the affected rows of this lob. 

To know the table, you can : 

select table_name, column_name 
from dba_lobs 
where segment_name = 'SYS_LOB0002427660C00003$$' 
and owner = 'INTERFACE'; 

2- For the table, please run the ANALYZE 
SQL> ANALYZE TABLE QHUB.CASE_FILE_CAFH VALIDATE STRUCTURE CASCADE; 

ANALYZE TABLE QHUB.CASE_FILE_CAFH VALIDATE STRUCTURE CASCADE 
* 
ERROR at line 1: 
ORA-01498: block check failure - see trace file 

Then I have run the job again but this time within 10 minutes it created 8.3 GB trace file. 


ACTION PLAN 
---------------------- 
Create dummy table with the same table structure of "QHUB.CASE_FILE_CAFH " 
Get the metadata of table 'QHUB.CASE_FILE_CAFH', 

select dbms_metadata.get_ddl('TABLE','<table name>','<owner>') FROM dual ; 

ex, 

SQL>set long 1000000 
SQL>select dbms_metadata.get_ddl('TABLE','CASE_FILE_CAFH','QHUB') FROM dual ; 
then, 
create table <owner> as selecct * from <Original> where 1=2 ; 
insert into.... <salvage > select * from <original> 
SQL> select dbms_metadata.get_ddl('TABLE','CASE_FILE_CAFH','QHUB') FROM dual ; 
CREATE TABLE "QHUB"."CASE_FILE_CAFH" 
( "CAFH_CASE_FILE_ID" VARCHAR2(14) NOT NULL ENABLE, 
"CAFH_LODGING_AUTH_ID" CHAR(5), 
"CAFH_CREATED_BY" VARCHAR2(30) NOT NULL ENABLE, 
"CAFH_CREATED_DATE" DATE NOT NULL ENABLE, 
"CAFH_COURT_LOCN_ID" CHAR(5) NOT NULL ENABLE, 
"CAFH_COURT_TYPE_ID" CHAR(5) NOT NULL ENABLE, 
"CAFH_LAST_UPDATED_BY" VARCHAR2(30) NOT NULL ENABLE, 
"CAFH_CASE_FILE_STATUS_ID" CHAR(5) NOT NULL ENABLE, 
"CAFH_LAST_UPDATED_DATE" DATE NOT NULL ENABLE, 
"CAFH_LOCAL_FILE_REF" VARCHAR2(17), 
"CAFH_ORIGINATING_ID" CHAR(5) NOT NULL ENABLE, 
"CAFH_LOCN_ID" CHAR(10), 
"CAFH_TITLE" VARCHAR2(50) NOT NULL ENABLE, 
"CAFH_JUVENILE_FILE_FLAG" CHAR(1) NOT NULL ENABLE, 
"CAFH_LOCN_LAST_UPDATED_DATE" DATE NOT NULL ENABLE, 
"CAFH_LOCN_LAST_UPDATED_BY" VARCHAR2(30) NOT NULL ENABLE, 
"CAFH_CASE_CREATED_DATE" DATE NOT NULL ENABLE, 
"CAFH_CASE_CREATED_BY" VARCHAR2(30) NOT NULL ENABLE, 
"CAFH_PRE_SYSTEM_FLAG" CHAR(1) NOT NULL ENABLE, 
"CAFH_DELETED_REASON" VARCHAR2(255), 
"CAFH_VERIFIED_BY" VARCHAR2(30), 
"CAFH_VERIFIED_DATE" DATE, 
"CAFH_DATA_CLASS_LEVEL" NUMBER(2,0) NOT NULL ENABLE, 
"CAFH_ARCHIVE_STATUS" NUMBER(1,0) NOT NULL ENABLE, 

"CAFH_FILE_SUPPRESSION_FLAG" CHAR(1) NOT NULL ENABLE, 
"CAFH_PRINT_STATUS" CHAR(1) NOT NULL ENABLE, 
"CAFH_GROUP_NUMBER" NUMBER(9,0), 
"CAFH_FROM_CHARGE_PREP_FLAG" CHAR(1) NOT NULL ENABLE, 
"CAFH_QPS_DOCUMENT_ID" NUMBER(10,0), 
"CAFH_DOWNLOADED_TO_QSTATS_DATE" DATE, 
"CAFH_ROW_START_TS" TIMESTAMP (9) NOT NULL ENABLE, 
"CAFH_ROW_END_TS" TIMESTAMP (9) NOT NULL ENABLE, 
"CAFH_PROPAGATION_TS" TIMESTAMP (9) DEFAULT systimestamp NOT NULL ENABLE, 
"CAFH_ROW_STATE_ID" VARCHAR2(5) NOT NULL ENABLE, 
CONSTRAINT "PK_CASE_FILE_CAFH" PRIMARY KEY ("CAFH_CASE_FILE_ID", "CAFH_ROW_END 
_TS", "CAFH_ROW_START_TS") 
USING INDEX PCTFREE 10 INITRANS 2 MAXTRANS 255 NOLOGGING COMPUTE STATISTICS 

STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645 
PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1 BUFFER_POOL DEFAULT) 
TABLESPACE "QHUB_INDEX" ENABLE, 
CONSTRAINT "FK_CAFH_KEY_CAFH" FOREIGN KEY ("CAFH_CASE_FILE_ID") 
REFERENCES "QHUB"."KEY_CAFH" ("CAFH_CASE_FILE_ID") ENABLE, 
CONSTRAINT "FK_CAFH_ROS" FOREIGN KEY ("CAFH_ROW_STATE_ID") 
REFERENCES "QHUB"."ROW_STATE_ROS" ("ROS_ROW_STATE_ID") ENABLE 
) PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255 NOCOMPRESS LOGGING 
STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645 
PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1 BUFFER_POOL DEFAULT) 
TABLESPACE "QHUB_DATA" ROWDEPENDENCIES 
 
SQL> Select segment_name,segment_type,owner from dba_extents 
where file_id=8 and 1006 between block_id and block_id + b locks -1 ;

SEGMENT_NAME				SEGMENT_TYPE OWNER
------------------ ------------------------------ ---------------------------------------------------------------------------
SYS_LOB0002427660C00003$$                        LOBSEGMENT INTERFACE

SQL> Select segment_name,segment_type,owner from dba_extents 
where file_id=9 and 2072 between block_id and block_i d + blocks -1 ;

SEGMENT_NAME				SEGMENT_TYPE OWNER
------------------ --------------------------------------------------------------------------------------------------------------------
SYS_LOB0002427660C00003$$		LOBSEGMENT INTERFACE

SQL> SQL> Select segment_name,segment_type,owner from dba_extents 
where file_id=14 and 10328 between block_id and block_id + blocks -1 ;

SEGMENT_NAME				SEGMENT_TYPE OWNER
------------------ --------------------------------------------------------------------------------------------------------------------------------
CASE_FILE_CAFH				TABLE QHUB

select table_name, column_name
from dba_lobs
where segment_name = 'SYS_LOB0002427660C00003$$'
and owner = 'INTERFACE';

2- For the table, please run the ANALYZE

ANALYZE TABLE QHUB.CASE_FILE_CAFH VALIDATE STRUCTURE CASCADE;

Analyze job created 8GB trace files within 10 minutes.

sqlplus / as sysdba 
set pagesize 2000
set linesize 250 
SELECT e.owner, e.segment_type, e.segment_name, e.partition_name, c.file#
     , greatest(e.block_id, c.block#) corr_start_block#
     , least(e.block_id+e.blocks-1, c.block#+c.blocks-1) corr_end_block#
     , least(e.block_id+e.blocks-1, c.block#+c.blocks-1)
       - greatest(e.block_id, c.block#) + 1 blocks_corrupted
     , null description
  FROM dba_extents e, v$database_block_corruption c
 WHERE e.file_id = c.file#
   AND e.block_id <= c.block# + c.blocks - 1
   AND e.block_id + e.blocks - 1 >= c.block#
UNION
SELECT s.owner, s.segment_type, s.segment_name, s.partition_name, c.file#
     , header_block corr_start_block#
     , header_block corr_end_block#
     , 1 blocks_corrupted
     , 'Segment Header' description
  FROM dba_segments s, v$database_block_corruption c
 WHERE s.header_file = c.file#
   AND s.header_block between c.block# and c.block# + c.blocks - 1
UNION
SELECT null owner, null segment_type, null segment_name, null partition_name, c.file#
     , greatest(f.block_id, c.block#) corr_start_block#
     , least(f.block_id+f.blocks-1, c.block#+c.blocks-1) corr_end_block#
     , least(f.block_id+f.blocks-1, c.block#+c.blocks-1)
       - greatest(f.block_id, c.block#) + 1 blocks_corrupted
     , 'Free Block' description
  FROM dba_free_space f, v$database_block_corruption c
 WHERE f.file_id = c.file#
   AND f.block_id <= c.block# + c.blocks - 1
   AND f.block_id + f.blocks - 1 >= c.block#
order by file#, corr_start_block#;

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
If the issue has to be fixes, then the below are the possible ways

1. using dbms_repair soft corrupted the block so that no further reading writing will be done on those corrupted block)

2. put 10231 event at instance level, export good data, truncate teh table and import good data back

3. salvaga the good data by putting 10231 at session level, doing CTAS (create structure first and then insert data into new table from old table, rename new table as old and put all the constraint.

In all the three option, their will be data loss (actuall data loss is already happened and above 3 options is to get the good data and discard the bad data)


Extracting Data from a Corrupt Table using DBMS_REPAIR or Event 10231 [ID 33405.1]
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

What is event 10231 ?
~~~~~~~~~~~~~~~~~~~~~
	This event allows Oracle to skip certain types of corrupted blocks
	on full table scans ONLY hence allowing export or "create table as
	select" type operations to retrieve rows from the table which are not
	in the corrupt block. Data in the corrupt block is lost.


Oracle8i,9i,10g,11g
  ~~~~~~~~~~~~~~~~~~~
        Connect as a SYSDBA user and mark the table as needing to skip 
        corrupt blocks thus:
          execute DBMS_REPAIR.SKIP_CORRUPT_BLOCKS('','');

	Now you should be able to issue a CREATE TABLE AS SELECT / ALTER TABLE <> MOVE operation
	against the corrupt table to extract data from all non-corrupt
	blocks, or EXPORT the table.
	Eg:
		CREATE TABLE salvage_emp 
		 AS SELECT * FROM corrupt_emp;
        or
               ALTER TABLE <> MOVE

        To clear the attribute for a table use:
          execute DBMS_REPAIR.SKIP_CORRUPT_BLOCKS('','',flags=>dbms_repair.noskip_flag);

        Note that when a session skips a corrupt block due to SKIP_CORRUPT being set then a message is written to the trace file (not the alert log) for each block skipped in the form:
                 table scan: segment: file# 6 block# 11
                    skipping corrupt block file# 6 block# 12

Setting the Event in a Session
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 	Connect to Oracle as a user with access to the corrupt table and
	issue the command:

	ALTER SESSION SET EVENTS '10231 TRACE NAME CONTEXT FOREVER, LEVEL 10';

The event can be turned off by doing this:

SQL> ALTER SESSION SET EVENTS '10231 TRACE NAME CONTEXT OFF';

Now you should be able to issue a CREATE TABLE AS SELECT operation
	against the corrupt table to extract data from all non-corrupt
	blocks, but an export would still fail as the event is only set 
        within your current session.
	Eg:
		CREATE TABLE salvage_emp 
		 AS SELECT * FROM corrupt_emp;

        or 

                ALTER TABLE <> MOVE

  Setting the Event at Instance level
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	This requires that the event be added to the init$ORACLE_SID.ora file
 	used to start the instance:

		shutdown the database

		Edit your init.ora startup configuration file and ADD
		a line that reads:

			event="10231 trace name context forever, level 10"

		  Make sure this appears next to any other EVENT= lines in the
	 	  init.ora file. 

                If you are using an spfile please refer to Note:160178.1 
                'How to set EVENTS in the SPFILE'.

		STARTUP 
			If the instance fails to start check the syntax
			of the event parameter matches the above exactly.
			Note the comma as it is important. 

		SHOW PARAMETER EVENT 
			To check the event has been set in the correct place.
			You should see the initial portion of text for the
			line in your init.ora file. If not check which 
			parameter file is being used to start the database.

		Select out the data from the table using a full table scan
		operation.
			Eg: Use a table level export 
			    or create table as select  or ALTER TABLE <> MOVE

  Export Warning: If the table is very large then some versions of export
		  may not be able to write more than 2Gb of data to the
		  export file. See Note:62427.1 for general information
		  on 2Gb limits in various Oracle releases.

			    
Salvaging data from the corrupt block itself
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  SKIP_CORRUPT and event 10231 extract data from good blocks but
  skip over corrupt blocks. To extract information from the corrupt
  block there are three main options:

    - Select column data from any good indexes
        This is discussed towards the end of the following 2 articles:
           Oracle7 - using ROWID range scans    Note:34371.1
           Oracle8/8i - using ROWID range scans Note:61685.1

    - See if Oracle Support can extract any data from HEX dumps of the
      corrupt block.

    - It may be possible to salvage some data using Log Miner
     
Once you have the data extracted
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	Once you have the required data extracted either into an export file
	or into another table make sure you have a valid database backup before
	proceeding. The importance of this cannot be over-emphasised.

	Double check you have the SQL to rebuild the object and its indexes 
	etc..

	Double check that you have any diagnostic information if requested by 
	Oracle support. Once you proceed with dropping the object certain 
	information is destroyed so it is important to capture it now.

	Now you can:

	    	If 10231 was set at instance level:
		   Remove the 'event' line from the init.ora file 

		   SHUTDOWN and RESTART the database.

		   SHOW PARAMETER EVENT
			Make sure the 10231 event is no longer shown

		RENAME or DROP the problem table
			If you have space it is advisable to RENAME the
			problem table rather than DROP it at this stage.

		Recreate the table.
			Eg: By importing.
			    Take special care to get the storage clauses 
			    correct when recreating the table.

		Create any indexes, triggers etc.. required
			Again take care with storage clauses.

		Re-grant any access to the table.

		If you RENAMEd the original table you can drop it once
		the new table has been tested.
-------------

create table COPY_CASE_FILE_CAFH  as select * from CASE_FILE_CAFH where 1=2 ; 
insert into COPY_CASE_FILE_CAFH  select * from CASE_FILE_CAFH; 


SQL> select count (*) from MONOTEST.COPY_CASE_FILE_CAFH ; 

COUNT(*) 
---------- 
6812200 

SQL> select count (*) from QHUB.CASE_FILE_CAFH ; 

COUNT(*) 
---------- 

alter table QHUB.COPY_CASE_FILE_CAFH rename to QHUB.CASE_FILE_CAFH_bak; 

alter table MONOTEST.COPY_CASE_FILE_CAFH rename to QHUB.CASE_FILE_CAFH; 

<---- once the above has been done and you have verified that CASE_FILE_CAFH is in place and has the data. 


drop table QHUB.CASE_FILE_CAFH_bak; 

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

(1) Using the DBMS_REPAIR package:

The DBMS_REPAIR package can be created by running the dbmsrpr.sql script. On unix, the script to install the package can be found under $ORACLE_HOME/rdbms/admin location.

Step 1: Create a table to hold information about blocks that are corrupted:

exec dbms_repair.admin_tables(table_type=>DBMS_REPAIR.REPAIR_TABLE,DBMS_ 
REPAIR.CREATE_ACTION,tablespace=>'users')

Step 2: Check the objects and populate the repair_table with corrupt block information:

SQL> variable corrupt_cnt number;

SQL> exec dbms_repair.CHECK_OBJECT(schema_name=>'DEVUSER',

object_name=>'DEVOBJECT',CORRUPT_COUNT =>:CORRUPT_CNT)

-- This should return the total number of corrupt blocks identified

SQL> print corrupt_cnt

Step 3: Try fixing the corrupted blocks:

SQL> variable fix_cnt number;

SQL> exec dbms_repair.fix_corrupt_blocks(schema_name=>'DEVUSER',

object_name=>'DEVOBJECT',FIX_COUNT =>:FIX_CNT)

-- This should return the total number of corrupt blocks that were fixed

SQL> print fix_cnt

If for some reason oracle is unable to fix the corrupt blocks then the option available is to salvage the uncorrupted blocks from the table. Step 4 below explains how.

Step 4: Saving uncorrupted blocks:

If corrupted blocks cannot be repaired then we are left with the only option of trying to save uncorrupted blocks.

-- Have oracle mark the table to skip over corrupt blocks

SQL> exec dbms_repair.skip_corrupt_blocks(schema_name=>'DEVUSER',object_name=> 'DEVOBJECT')

Backup the uncorrupted blocks by doing a Create Table As Select (CTAS) or export and then you can rename the table later. 

You can disable the skipping of corrupt blocks on the table by doing this:

SQL> exec dbms_repair.skip_corrupt_blocks(schema_name=>'DEVUSER',object_name=> 'DEVOBJECT', flags=>dbms_repair.noskip_flag)

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
SQL> BEGIN
DBMS_HM.RUN_CHECK (
check_name   => 'Data Block Integrity Check',
run_name     => 'datablockint',
input_params => 'BLC_DF_NUM=5;BLC_BL_NUM=216337');
END;
/ 

PL/SQL procedure successfully completed.

SQL> SELECT DBMS_HM.GET_RUN_REPORT('datablockint') FROM DUAL;


###CDB&PDB 

RMAN Pluggable Database Backup

The RMAN user must have either SYSDBA or the new SYSBACKUP priviledge.
RMAN can be run from ROOT container: rman target sys/<pw>@t12ccdb
rman target /
or from the PDB: rman target sys/<pw>@t12cpdb1

When connected to a PDB, all commands pertain to that PDB only.
When connected to ROOT, commands pertain to any file in the CDB unless qualified by the PDB name. 

RMAN command REPORT SCHEMA can be used to identify the files in a Container Database.
This example shows a CDB (T12cCDB) with one PDB (T12cPDB1):
% rman target sys/<pw>@t12ccdb
RMAN> report schema;

using target database control file instead of recovery catalog
Report of database schema for database with db_unique_name T12CCDB
** (filenames have been edited for clarity)

List of Permanent Datafiles
===========================
File Size(MB) Tablespace RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1 960 SYSTEM *** .../oradata/T12CCDB/datafile/o1_mf_system_8008cm5s_.dbf
3 660 SYSAUX *** .../oradata/T12CCDB/datafile/o1_mf_sysaux_80089voz_.dbf
4 50 UNDOTBS1 *** .../oradata/T12CCDB/datafile/o1_mf_undotbs1_8gtp7g6l_.dbf
5 250 PDB$SEED:SYSTEM *** .../oradata/T12CCDB/C4B70772D4DF1DF8E0437108DC0A7D20/datafile/o1_mf_system_8008jc7k_.dbf
6 5 USERS *** .../oradata/T12CCDB/datafile/o1_mf_users_8008fnov_.dbf
7 490 PDB$SEED:SYSAUX *** .../oradata/T12CCDB/C4B70772D4DF1DF8E0437108DC0A7D20/datafile/o1_mf_sysaux_8008jc8m_.dbf
8 250 T12CPDB1:SYSTEM *** .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_system_8008r3wh_.dbf
9 510 T12CPDB1:SYSAUX *** .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_sysaux_8008r3vl_.dbf
10 5 T12CPDB1:USERS *** .../oradata/T12CCDB/datafile/o1_mf_users_8gtp7ghf_.dbf
20 100 T12CPDB1:RECTBL *** .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_rectbl_8hfcv26r_.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1 530 TEMP 32767 .../oradata/T12CCDB/datafile/o1_mf_temp_8bz0jcxg_.tmp
2 20 PDB$SEED:TEMP 32767 .../oradata/T12CCDB/C40F9B49FC9C19E0E0430BAAE80AFF01/datafile/o1_mf_temp_8bz0jfkj_.tmp
3 20 T12CPDB1:TEMP 32767 .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_temp_8bz0jh7x_.tmp 

REPORT SCHEMA command is currently the only command that makes it easy to determine the name of the PDB that a file belongs to.
If connected to PDB, only the PDB datafiles are listed:

% rman target sys/<pw>@t12cpdb1
RMAN> report schema;

List of Permanent Datafiles
===========================
File Size(MB) Tablespace RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
8 250 T12CPDB1:SYSTEM *** .../oradata/T12CCDB/datafile/o1_mf_system_8hloc72d_.dbf
9 510 T12CPDB1:SYSAUX *** .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_sysaux_8008r3vl_.dbf
10 5 T12CPDB1:USERS *** .../oradata/T12CCDB/datafile/o1_mf_users_8hlowbh2_.dbf
20 100 T12CPDB1:RECTBL *** .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_rectbl_8hfcv26r_.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
3 20 T12CPDB1:TEMP 32767 .../oradata/T12CCDB/C4B71645EF062616E0437108DC0A91E4/datafile/o1_mf_temp_8bz0jh7x_.tmp 

1. Complete CDB backup

Backup CDB$ROOT, PDB$SEED and ALL PDBS:

% rman target sys/<pw>@t12ccdb
RMAN> BACKUP DATABASE PLUS ARCHIVELOG ALL DELETE INPUT;
RMAN> LIST BACKUP OF DATABASE;

2. Partial CDB backup

Backup only PDB T12CPDB1:

%rman target sys/<pw>@t12ccdb
RMAN> BACKUP PLUGGABLE DATABASE T12CPDB1 TAG 'T12CPDB1';
RMAN> LIST BACKUP;

You do not need to specify a TAG, as in example above to identify backups of Pluggable Database, TAG is just used in this sample.
RMAN LIST BACKUP command shows you the information to which Database, or Pluggable Database a rman backup belongs to.

However as FRA shows GUID in its PATH name, so in case needed you may use alternate following 
sample query to identify to which PDB a Backup belongs to:

In this sample: The GUID for T12CPDB1 is C4B71645EF062616E0437108DC0A91E4.

From the CDB:

SQL> SET LINES 150
SQL> SELECT CON_ID, DBID, CON_UID, GUID, NAME FROM v$pdbs;
CON_ID DBID CON_UID GUID NAME
---------- ---------- ---------- -------------------------------- ------------------------------
2 4031181962 4031181962 C40F9B49FC9C19E0E0430BAAE80AFF01 PDB$SEED
3 575001283 575001283 C4B71645EF062616E0437108DC0A91E4 T12CPDB1


3. Partial PDB backup

3a. Backup system and sysaux tablespace from PDB T12CPDB1 whilst connected to ROOT:

% rman target sys/<pw>@t12ccdb 
RMAN>BACKUP TABLESPACE T12CPDB1:SYSTEM, T12CPDB1:SYSAUX; 

3b. Backup system tablespace from pluggable database T12CPDB1 and the SYSAUX tablespace from ROOT CDB:

When connected to ROOT if you do not specify the PDB prefix, the ROOT container is assumed.

% rman target sys/<pw>@t12ccdb 
RMAN>BACKUP TABLESPACE T12CPDB1:SYSTEM, SYSAUX; 

3c. File# however is unique so you can backup datafiles when connected to ROOT without having to specify the container name if you use file#:

To backup datafile 3 from CDB$ROOT and datafile 20 from PDB T12CPDB1

% rman target sys/<pw>@t12ccdb
RMAN> BACKUP DATAFILE 3,20; 

RMAN Pluggable Database Recovery

1. Loss of system datafile from PDB T12cPDB1

The Container Database and all other PDBs are usually unaffected, only PDB T12CPDB1 is unavailable.
Restore must be done from ROOT.
However loss of a SYSTEM datafile of PDB is as critical as loss of a SYSTEM datafile of CDB/non-CDB, i.e. this will may lead to unpredictable behaviour, mostly crash the entire CDB (i.e. all PDBs will be unavailable).
In this case, you need to restore/recover this SYSTEM datafile of PDB in MOUNT state of CDB.
This behaviour will be enhanced in future releases, i.e., loss of SYSTEM datafile of PDB will NOT crash the CDB or other PDBs.

% rman target /
RMAN> RESTORE DATAFILE 8;
RMAN> RECOVER DATAFILE 8;
RMAN> ALTER PLUGGABLE DATABASE T12CPDB1 OPEN; 

2. Loss of any non-system datafile from PDB eg datafile 10 USERS tablespace

Depending on the circumstances, the file may be already offlined if not - offline it: 

% rman sys/<pw>@t12cpdb1 
RMAN> ALTER DATABASE DATAFILE 10 OFFLINE;
RMAN> RESTORE DATAFILE 10;
RMAN> RECOVER DATAFILE 10;
RMAN> ALTER DATABASE DATAFILE 10 ONLINE;

3. Loss of a complete tablespace from PDB

PDB T12CPDB1 remains open. 

% rman target sys/oracle@t12ccpdb1
RMAN> ALTER TABLESPACE USERS OFFLINE;
RMAN> RESTORE TABLESPACE USERS;
RMAN> RECOVER TABLESPACE USERS;
RMAN> ALTER TABLESPACE USERS ONLINE; 

4: Loss of entire PDB

% rman target sys/<pw>@t12ccdb
RMAN> RESTORE PLUGGABLE DATABASE T12CPDB1;
RMAN> RECOVER PLUGGABLE DATABASE T12CPDB1;
RMAN> ALTER PLUGGABLE DATABASE T12cPDB1 open; 
Note:
# LOSS OF PLUGGABLE DATABASE is not the same as if Pluggable DAtabase is DROPPED
- LOSS OF PLUGGABLE DATABASE: This is for example if pluggable database/datafiles are accidently deleted, corrupted etc...
but the repository/metadta are still known and existing.
In this case the Metadata for the PDB do still exist, so REstore form backup is possible

- IF DROP PLUGGABLE DATABASE <PDBNAME> is done 
This will drop the PDB and remove the metadata from repository, 
so restore ( including PDB - PITR to before the dropped time ) fails
like
.
RMAN-06813: could not translate pluggable database PDB1
 
---

## Create controlfile 
```
CREATE CONTROLFILE SET DATABASE "PTEST1" RESETLOGS ARCHIVELOG
    MAXLOGFILES 192
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 32
    MAXLOGHISTORY 5840
LOGFILE
  GROUP 1 (
    '+PTEST1_DATA_01',
    '+PTEST1_ARC_01'
  ) SIZE 50M BLOCKSIZE 512,
  GROUP 2 (
    '+PTEST1_DATA_01',
    '+PTEST1_ARC_01'
  ) SIZE 50M BLOCKSIZE 512,
  GROUP 3 (
    '+PTEST1_DATA_01',
    '+PTEST1_ARC_01'
  ) SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '+PTEST1_DATA_01/ptest1/datafile/system.259.840123923',
  '+PTEST1_DATA_01/ptest1/datafile/sysaux.271.840123917',
  -----

CHARACTER SET WE8ISO8859P1
;

SQL> @crontrol_create.sql

Control file created.

SQL> select to_char(TIME,'dd/mm/yy:hh24:mi:ss'),count(*) from v$recover_file group by to_char(TIME,'dd/mm/yy:hh24:mi:ss');

TO_CHAR(TIME,'DD/   COUNT(*)
----------------- ----------
21/02/14:04:28:29         30

SQL> spool off;
```

### restore_ptest11.sh
```
#!/usr/bin/ksh
TODAY=`date +%y%m%d_%H%M`
export NLS_DATE_FORMAT="YYYY-MM-DD:HH24:MI:SS"
#export NB_ORA_CLIENT=csmper-cls09
export ORACLE_SID=ptest11
export ORACLE_HOME=/app/ptest1/product/11.2.0.2/db_1
ORAENV_ASK=NO
. oraenv
rman target / msglog /home/oracle/sthati/restore/restore_ptest11_$TODAY.out append << EOF
RUN {
ALLOCATE CHANNEL ch1 TYPE 'SBT_TAPE'
     PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch2 TYPE 'SBT_TAPE'
     PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
set newname for datafile 1 to '+PTEST1_DATA_01';
set newname for datafile 2 to '+PTEST1_DATA_01';
set newname for datafile 3 to '+PTEST1_DATA_01';
set newname for datafile 4 to '+PTEST1_DATA_01';
set newname for datafile 5 to '+PTEST1_DATA_01';
set newname for datafile 6 to '+PTEST1_DATA_01';
set newname for datafile 7 to '+PTEST1_DATA_01';
set newname for datafile 8 to '+PTEST1_DATA_01';
set newname for datafile 9 to '+PTEST1_DATA_01';
---
set newname for datafile 30 to '+PTEST1_DATA_01';
set until time "2014-02-21:04:28:26','YYYY-MM-DD:HH24:MI:SS";
restore database;
#restore database from tag 'NBUHOTBACK_*****';
switch datafile all;
#set until time "2014-02-21:04:28:26','YYYY-MM-DD:HH24:MI:SS";
recover database;
# Release channels
RELEASE CHANNEL ch1;
RELEASE CHANNEL ch2;
}
EOF
```
### vi recover_ptest11.sh
```
#!/usr/bin/ksh
TODAY=`date +%y%m%d_%H%M`
export NLS_DATE_FORMAT="YYYY-MM-DD:HH24:MI:SS"
#export NB_ORA_CLIENT=csmper-cls09
export ORACLE_SID=ptest11
export ORACLE_HOME=/app/ptest1/product/11.2.0.2/db_1
ORAENV_ASK=NO
. oraenv
rman target / msglog /home/oracle/sthati/restore/restore_ptest11_$TODAY.out append << EOF
RUN {
ALLOCATE CHANNEL ch1 TYPE 'SBT_TAPE'
     PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
ALLOCATE CHANNEL ch2 TYPE 'SBT_TAPE'
     PARMS='ENV=(NB_ORA_SERVER=ironper-bak01d,NB_ORA_CLIENT=csmper-cls08b,NB_ORA_POLICY=IronOre-DB-Cluster-Unix-ORA)';
set until time "2014-02-21:05:00:00','YYYY-MM-DD:HH24:MI:SS";
recover database;
# Release channels
RELEASE CHANNEL ch1;
RELEASE CHANNEL ch2;
}
EOF
```
 
