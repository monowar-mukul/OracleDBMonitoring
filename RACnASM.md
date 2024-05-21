## Clean OLD Logs
```
du -a | sort -n | tail -15
find /opt/oraarch/PLANNING/1_3*.dbf -mtime +3 -exec rm {} \;
find <grid_home>/grid/rdbms/audit/*.aud* -mtime +130 -exec rm {} \;
find <db_home>/admin/pprod25/bdump -name "*.trc" -a -mtime +30 -exec rm  -rf {} \;
find <db_home>/admin/pprod25/udump -name "*.trc" -a -mtime +30 -exec rm -rf {} \;
find <db_home>/admin/pprod25/adump -name "*.aud" -a -mtime +30 -exec rm -rf {} \;
find <oracle_base>/diag/rdbms/wspsp/wspsp3/trace  -name "*.trc" -a -mtime +30 -exec rm  -rf {} \;
find <audit_directory(adump)>  -name "*.aud" -a -mtime +30 -exec rm  -rf {} \;
```
### Listener_Log
```
<grid_home>/grid/log/diag/tnslsnr/<clustername>
$ du -sh *
332K    listener_scan1
1.6G    listener_scan2
332K    listener_scan3

example: $ find /opt/oracle/product/diag/tnslsnr/<cls07>/listener/alert/*.xml* -mtime +20 -exec rm {} \;
```
### List long operations for RAC.
```
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 99
COLUMN sid FORMAT 9999
COLUMN serial# FORMAT 99999
COLUMN username FORMAT A24
COLUMN module FORMAT A40
COLUMN progress_pct FORMAT 999
COLUMN elapsed FORMAT A10
COLUMN remaining FORMAT A10
 
SELECT s.inst_id,
       s.sid,
       s.serial#,
       s.sql_address,
       s.username,
       s.module,
       ROUND(sl.elapsed_seconds/60) || ':' || MOD(sl.elapsed_seconds,60) elapsed,
       ROUND(sl.time_remaining/60) || ':' || MOD(sl.time_remaining,60) remaining,
       ROUND(sl.sofar/(sl.totalwork+1)*100, 2) progress_pct
FROM   gv$session s,
       gv$session_longops sl
WHERE  s.sid     = sl.sid
AND    s.inst_id = sl.inst_id
AND    s.serial# = sl.serial#
ORDER BY progress_pct
/
```


### List session waits for RAC.

``` 
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 99
COLUMN username FORMAT A20
COLUMN sid FORMAT 9999
COLUMN serial# FORMAT 99999
COLUMN event FORMAT A48
COLUMN wait_class FORMAT A15
 
SELECT s.inst_id,
       NVL(s.username, '(oracle)') AS username,
       s.sid,
       s.serial#,
       sw.event,
       sw.wait_class,
       sw.wait_time,
       sw.seconds_in_wait,
       sw.state
FROM   gv$session_wait sw,
       gv$session s
WHERE  s.sid     = sw.sid
AND    s.inst_id = sw.inst_id
ORDER BY sw.seconds_in_wait DESC
/
```

### List memory allocations RAC Sessions.
``` 
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 9
COLUMN username FORMAT A20
COLUMN module FORMAT A40
COLUMN program FORMAT A40
 
SELECT a.inst_id,
       NVL(a.username,'(oracle)') AS username,
       a.module,
       a.program,
       Trunc(b.value/1024) AS memory_kb
FROM   gv$session a,
       gv$sesstat b,
       gv$statname c
WHERE  a.sid = b.sid
AND    a.inst_id = b.inst_id
AND    b.statistic# = c.statistic#
AND    b.inst_id = c.inst_id
AND    c.name = 'session pga memory'
AND    a.program IS NOT NULL
ORDER BY b.value DESC
/
```

### List all sessions for RAC.
``` 
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300
 
COLUMN inst_id FORMAT 9
COLUMN username FORMAT A10
COLUMN lockwait FORMAT A8
COLUMN program FORMAT A26
COLUMN sid FORMAT 9999
COLUMN serial# FORMAT 99999
COLUMN spid FORMAT A5
COLUMN logon_time FORMAT A20
COLUMN status FORMAT A10
COLUMN module FORMAT A24
COLUMN machine FORMAT A23
 
SELECT NVL(s.username, '(oracle)') AS username,
       s.inst_id,
       s.sid,
       s.serial#,
       p.spid,
       s.lockwait,
       s.status,
       s.module,
       s.machine,
       s.program,
       TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time
FROM   gv$session s,
       gv$process p
WHERE  s.paddr   = p.addr
AND    s.inst_id = p.inst_id
ORDER BY s.username, s.osuser
/
```
 
### Determine the clustername and clusternodes:
```
[oraclecls06 ~]$ olsnodes
cls06
cls07
cls08
```

### No. Of Nodes configured in Cluster:
-----------------------------------------
The below command can be used to find out the number of nodes registered into the cluster. It also displays the node’s Public name, Private name and Virtual name along with their numbers.
```
olsnodes -n -p -i
```

### Viewing Votedisk Information:
```
$ crsctl query css votedisk
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   abece3ad7bfa4f2fbf910c494c5629e4 (ORCL:OCR_VOTE01) [OCR_VOTE]
 2. ONLINE   96e3c933f6384febbf7f1d7f6d7cfdcc (ORCL:OCR_VOTE02) [OCR_VOTE]
 3. ONLINE   85147357c65c4fd4bfda0e417985c4ee (ORCL:OCR_VOTE03) [OCR_VOTE]
Located 3 voting disk(s).
```
### Viewing OCR Information:
-------------------------------
It is primarily used to chck the integrity of the OCR files. 
It also displays the version of OCR as well as storage space information. 
You can only have 2 OCR files at max.
```
[grid@cls06 admin]$ ocrcheck
```
### check node status
```
[grid@cls06 ~]$ srvctl status nodeapps
```
### check the status on cluster registry
```
$crs_stat -t | grep vip

ipconfig -a 
```

## 10g
```
crsctl status resource -t
```
### HAS Status
```
$ cat /etc/oracle/scls_scr/cls16/root/ohasdstr
```
```
$ srvctl status asm
$ srvctl config vip -n <iorsdb01-adm>
$ srvctl config nodeapps -a
$ srvctl config asm
```

OIFCFG: -- Oracle Interface Configuration tool
A command line tool for both single instance Oracle databases and RAC databases that enables us 
to allocate and deallocate network interfaces to components, direct components to use specific 
network interfaces, and retrieve component configuration information.

### NETWORE
```
$ srvctl config listener
$ srvctl config scan_listener
$ oifcfg iflist [-- display a list of current subnets]
bond0  10.240.26.0
bond1  10.240.27.0
bond2  10.240.54.0
bond2  169.254.0.0

$ oifcfg getif  [-- To display a list of networks]
bond0  10.240.26.0  global  public
bond2  10.240.54.0  global  cluster_interconnect
```
### ExaData: 
```
$ oifcfg getif
bondeth0  10.241.148.0  global  public
ib0  172.16.108.0  global  cluster_interconnect
ib1  172.16.108.0  global  cluster_interconnect

[grid@csmper-cls06 ~]$ cat /etc/hosts
[grid@csmper-cls06 ~]$ cat /etc/resolv.conf
search apac.ent.xxxx.net
nameserver 10.27.40.20
nameserver 10.27.12.32

DNS server issue ---

ping -i 10.27.40.20
ping -i 10.27.12.32

Servers effected: CSMPER-CLS06/07/08
```

### Various Timeout Settings in Cluster:
```
Disktimeout: Disk Latencies in seconds from node-to-Votedisk. Default Value is 200. (Disk IO)

Misscount: Network Latencies in second from node-to-node (Interconnect). Default Value is 60 Sec (Linux) and 30 Sec in Unix platform. (Network IO) Misscount < Disktimeout

IF
(Disk IO Time > Disktimeout) OR (Network IO time > Misscount)
THEN
REBOOT NODE
ELSE
DO NOT REBOOT
END IF;

```
```
[root@node1-pub ~]# crsctl get css disktimeout
200

[root@node1-pub ~]# crsctl get css misscount
```
### Configuration parameter misscount is not defined.

The above message indicates that the Misscount is not set manually and it is set to its default Value which is 60 seconds on Linux. It can be changed as below.
```
[root@node1-pub ~]# crsctl set css misscount 100
Configuration parameter misscount is now set to 100.

[root@node1-pub ~]# crsctl get css misscount
100

The below command sets the value of misscount back to its default value.
crsctl unset css misscount
[root@node1-pub ~]# crsctl unset css misscount
[root@node1-pub ~]# crsctl get css reboottime  
crsd restart 

root@iorsdb02-adm::/root
cd /u01/app/grid/diag/crs/iorsdb01-adm/crs/trace/
tailf crsd.trc
```

### CRS Check
```
$CRS_HOME/bin/crsctl check cluster -all

$CRS_HOME/bin/crsctl status resource -t
```

### BACKUP
```
To show the backups, type the commands :             ocrconfig -showbackup 
Perform a manual backup:      ocrconfig -manualbackup
Logical backup:                    #ocrconfig -export /home/oracle/ocr.backup

Display only manual backup:        ocrconfig -showbackup manual
Location of OLR(Oracle Local Registry):  #ocrcheck -local
Logical backup of OLR:    ocrconfig -local -export /../olr.backup

Check the contents by doing ocrdump -backupfile my_file 
Go to bin directory and stop the CRS. crs stop on all nodes. 
Perform the restore ocrconfig restore my_file 
Restart the nodes crs start 
Get a verbose output of all of the nodes by doing this: cluvfy comp ocr -n all -verbose 

Importing Oracle Cluster Registry Content on UNIX-Based Systems

Note: Do note that most of the configuration changes to the OCR contents also cause the file and database object creation. 
Moreover, not all of these changes are restored when you restore the OCR. 
Performing an OCR restore in order to revert to an older working configuration will invariably fail, and it goes without saying, your RAC will be out of sync.

Select the OCR export file that you want to import by using the following command: 
 ocrconfig -export myfile

Stop the RAC Clusterware on all nodes by going to the bin directory of your cluster and doing: 
  ../bin/crs stop

Now we have to carry out the import by supplying the following commands. Here myfile will be the file from which you want to import your OCR configuration data 
 ocrconfig -import myfile
Doing a cluvfy comp ocr -n all [-verbose] will retrieve a list of all the nodes in the cluster. 
```
  
### Processes 

```

On RAC node 1
1.Log in as root or your custom OS USER (i.e. grid) depend upon what processes you are killing.
2.Set up environment

If you need the averages for a period of time of a specific process, try the accumulative -c option of top:

top -c a -pid PID

top -p 78198

top - 14:03:29 up 184 days,  2:53,  7 users,  load average: 8.68, 10.48, 11.81
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
Cpu(s): 16.4%us,  7.9%sy,  0.1%ni, 75.3%id,  0.1%wa,  0.0%hi,  0.2%si,  0.0%st
Mem:  528918448k total, 483413288k used, 45505160k free,  2903652k buffers
Swap: 25165820k total,  2196540k used, 22969280k free, 65650428k cached

   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 78198 root      20   0 2584m 201m  30m S  0.7  0.0  18:16.20 crsd.bin
```

### CRSD Process (root)
```
1.Check CRSD

[root@racpoc01 ~]# ps -aef | grep -i crsd

$ ps -ef|grep crsd
root      54351  47940  0 11:12 pts/1    00:00:00 grep crsd
root      78198      1  3 05:01 ?        00:12:35 /u01/app/12.1.0.2/gi_000/bin/crsd.bin reboot

2.Kill CRSD ( root )

# kill -9 10748
3.Check CRSD should restart automatically.

#  ps -aef | grep -i crsd
```

### EVMD Process (grid)
```
1.Check EVMD
grid@iorsdb01-adm:+ASM1:/home/grid
$  ps -aef | grep -i evmd
grid      14594      1  1 May11 ?        2-19:38:14 /u01/app/12.1.0.2/gi_000/bin/evmd.bin
grid      14822  14594  0 May11 ?        06:13:17 /u01/app/12.1.0.2/gi_000/bin/evmlogger.bin -o /u01/app/12.1.0.2/gi_000/log/[HOSTNAME]/evmd/evmlogger.info -l /u01/app/12.1.0.2/gi_000/log/[HOSTNAME]/evmd/evmlogger.log
grid      87509  84602  2 11:17 pts/1    00:00:00 grep -i evmd

2.Kill EVMD process
$ kill -9 14594

3.Check EVMD should restart automatically.
$  ps -aef | grep -i evmd
```

### GRID CRSD ORAAGENT Process
```
1.Check GRID CRSD ORAAGENT
$ locate oraagent_grid.pid
/u01/app/grid/crsdata/iorsdb01-adm/output/crsd_oraagent_grid.pid
/u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_oraagent_grid.pid

$ cat /u01/app/grid/crsdata/iorsdb01-adm/output/crsd_oraagent_grid.pid
81805
$  ps -aef | grep -i oraagent
grid       9538  84602  0 11:25 pts/1    00:00:00 grep -i oraagent
grid      14579      1  0 May11 ?        18:00:10 /u01/app/12.1.0.2/gi_000/bin/oraagent.bin
grid      81805      1  0 05:01 ?        00:02:35 /u01/app/12.1.0.2/gi_000/bin/oraagent.bin
oracle    81814      1 20 05:01 ?        01:17:58 /u01/app/12.1.0.2/gi_000/bin/oraagent.bin

2.Kill GRID CRSD ORAAGENT
kill -9 81805
3.Check GRID CRSD ORAAGENT should restart automatically.
ps -aef | grep -i oraagent

$ cat /u01/app/grid/crsdata/iorsdb01-adm/output/crsd_oraagent_grid.pid
<NEW Process>
```

### GRID OHASD ORAAGENT Process
```
$ locate oraagent_grid.pid

1.Check GRID OHASD ORAAGENT
$ cat /u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_oraagent_grid.pid
$ ps -aef | grep -i oraagent

2.Kill GRID OHASD ORAAGENT
kill -9 14579
3.Check GRID OHASD ORAAGENT should restart automatically.
 ps -aef | grep -i oraagent

cat /u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_oraagent_grid.pid
```

### GRID CRSD ROOTAGENT Process ( root )
```
$ locate orarootagent_root.pid
/u01/app/grid/crsdata/iorsdb01-adm/output/crsd_orarootagent_root.pid
/u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_orarootagent_root.pid

1. cat /u01/app/grid/crsdata/iorsdb01-adm/output/crsd_orarootagent_root.pid

	#  ps -aef | grep -i rootagent

2.Kill GRID CRSD ROOTAGENT as “root”
Kill -9 <> 

3.Check GRID CRSD ROOTAGENT should restart automatically.
	#  ps -aef | grep -i rootagent

cat /u01/app/grid/crsdata/iorsdb01-adm/output/crsd_orarootagent_root.pid
```

### GRID OHASD ROOTAGENT Process ( root )
```
1. 
cat /u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_orarootagent_root.pid

# ps -aef | grep -i rootagent

2.Kill GRID OHASD ROOTAGENT as “root”

3.Check GRID OHASD ROOTAGENT should restart automatically.

# ps -aef | grep -i rootagent

cat /u01/app/grid/crsdata/iorsdb01-adm/output/ohasd_orarootagent_root.pid
```

### RDBMS CRSD ORAAGENT Process (oracle)
```
1.Check RDBMS CRSD ORAAGENT
$ locate oraagent_oracle.pid
/u01/app/grid/crsdata/iorsdb01-adm/output/crsd_oraagent_oracle.pid

cat /u01/app/grid/crsdata/iorsdb01-adm/output/crsd_oraagent_oracle.pid

$ ps -aef | grep -i oraagent

Kill RDBMS CRSD ORAAGENT
$ kill -9 <> 

2.Check RDBMS CRSD ORAAGENT should restart automatically.
$ ps -aef | grep -i oraagent
```


### CSSD Process [THINK FIRST]
```
1.Check CSSD
grid@iorsdb01-adm:+ASM1:/home/grid
$ ps -aef | grep -i cssd
root      15288      1  0 May11 ?        05:51:45 /u01/app/12.1.0.2/gi_000/bin/cssdmonitor
root      15305      1  0 May11 ?        05:55:15 /u01/app/12.1.0.2/gi_000/bin/cssdagent
grid      15324      1  1 May11 ?        2-17:18:15 /u01/app/12.1.0.2/gi_000/bin/ocssd.bin
grid     105183  84602  3 11:19 pts/1    00:00:00 grep -i cssd

2.Kill CSSD – node reboot

kill -9 15324

3.Check node is rebooted and check VIP failed over during reboot and then after node restartVIP failed back automatically.
$ crsctl stat res –t

```

### LOGS:

```
$ORACLE_HOME/log/<node2>/cssd/ocssd.log
```

### How to Recreate OCR/Voting Disk Accidentally Deleted [ID 399482.1]
```
Depending on the issue, it may or may not be good idea to execute the steps provided.

OCR  
If the OCR has been deleted, then check if the OCR mirror is OK and vice versa. It may be prudent to use the OCR mirror to create the OCR. For steps on this check the documentation:  Oracle Database   Oracle Clusterware and Oracle Real Application Clusters Administration and Deployment Guide  
If the OCR mirror and OCR have been deleted, then it may be faster to restore the OCR using the OCR backups. For steps on this check the documentation: Oracle Database   Oracle Clusterware and Oracle Real Application Clusters Administration and Deployment Guide 
Voting Disk   
If there are multiple voting disks and one was accidentally deleted, then check if there are any backups of this voting disk. If there are no backups then we can add one using the crsctl add votedisk command. The complete steps are in the: Oracle Database   Oracle Clusterware and Oracle Real Application Clusters Administration and Deployment Guide 
SCLS directories 
These are internal only directories which are created by root.sh, if this directory is accidentally removed then they can only be created by the steps documented below 
Socket files in /tmp/.oracle or /var/tmp/.oracle 
If these files are accidentally deleted, then stop the Oracle Clusterware on that node and restart it again. This will  recreate these socket files. If the socket files for cssd is deleted then the Oracle Clusterware stack may not come down in which case the node has to be bounced. 
Solution
If none of the steps documented above can be used to restore the file that was accidentally deleted or is corrupted, then the following steps can be used to re-create/reinstantiate these files. The following steps require complete downtime on all the nodes.

Shutdown the Oracle Clusterware stack on all the nodes using command crsctl stop crs as root user. 
Backup the entire Oracle Clusterware home. 
Execute <CRS_HOME>/install/rootdelete.sh on all nodes 
Execute <CRS_HOME>/install/rootdeinstall.sh on the node which is supposed to be the first node 
The following commands should return nothing 
ps -e | grep -i 'ocs[s]d' 
ps -e | grep -i 'cr[s]d.bin' 
ps -e | grep -i 'ev[m]d.bin' 
Execute <CRS_HOME>/root.sh on first node 
After successful root.sh execution on first node Execute root.sh on the rest of the nodes of the cluster 
For 10gR2, use racgons; for 11g use onsconfig command. Using onsconfig stops and starts ONS so the changes take effect, while racgons doesn't do that so the changes won't take effect until ONS is restarted on all nodes. Examples for each are provided below.

For 10g
Execute as owner (generally oracle) of CRS_HOME command 
<CRS_HOME>/bin/racgons add_config hostname1:port hostname2:port
$/u01/crs/bin/racgons add_config halinux1:6251 halinux2:6251
For 11g
Execute as owner (generally oracle) of CRS_HOME command
<CRS_HOME>/install/onsconfig add_config hostname1:port hostname2:port
$/u01/crs/install/onsconfig add_config halinux1:6251 halinux2:6251
Execute as owner of CRS_HOME (generally oracle)  <CRS_HOME>/bin/oifcfg setif -global. Please review Note 283684.1 for details.

$/u01/crs/bin/oifcfg setif -global  eth0/192.168.0.0:cluster_interconnect eth1/10.35.140.0:public
Add listener using netca. This may give errors if the listener.ora contains the entries already. If this is the case, move the listener.ora to /tmp from the $ORACLE_HOME/network/admin or from the $TNS_ADMIN directory if the TNS_ADMIN environmental is defined and then run netca. Add all the listeners that were added earlier. 
Add ASM & database resource to the OCR using the appropriate srvctl add database command as the user who owns the ASM & database resource. Please ensure that this is not run as root user 
Add  Instance, services using appropriate srvctl add commands. Please refer to the documentation for the exact commands. 
execute cluvfy stage -post crsinst -n node1,node2    ### Please ensure to replace node1,node2 with the node names of the cluster 

 VotingDISK issue 

Linux: root.sh Fails to Format Voting disks when Placing OCR/Voting Disks on ASM Using asmlib [ID 955550.1] 
--------------------------------------------------------------------------------
  Modified 02-JUN-2011     Type PROBLEM     Status PUBLISHED   
In this Document
  Symptoms
  Changes
  Cause
  Solution
--------------------------------------------------------------------------------

Applies to: 
Oracle Server - Enterprise Edition - Version: 11.2.0.0 to 11.2.0.2 - Release: 11.2 to 11.2
Linux x86
Red Hat Enterprise Linux Advanced Server x86-64 (AMD Opteron Architecture)
x86 64 bit (for Enterprise Linux only)
Linux x86-64
x86 32 bit (for Enterprise Linux only)
Grid Infrastructure, clusterware, CRS, voting disk 
Symptoms
Oracle Grid Infrastructure installation with Oracle ASM using ASMlib fails during root.sh execution.

At the completion of Grid Infrastructure installation, root.sh script executed on the first node in the cluster fails to format the OCR and voting disks which are placed on ASM storage.

Example of the error when root.sh fails:

CRS-2676: Start of 'ora.ctssd' on 'auw2k3' succeeded
ASM created and started successfully.
DiskGroup DATA created successfully.

Errors in file :
ORA-27091: unable to queue I/O
ORA-15081: failed to submit an I/O operation to a disk
ORA-06512: at line 4
PROT-1: Failed to initialize ocrconfig
Command return code of 255 (65280) from command: /u01/app/11.2.0/grid/bin/ocrconfig -upgrade oragrid oinstall
Failed to create Oracle Cluster Registry configuration, rc 255
CRS-2500: Cannot stop resource 'ora.crsd' as it is not running
CRS-4000: Command Stop failed, or completed with errors.
Command return code of 1 (256) from command: /u01/app/11.2.0/grid/bin/crsctl stop resource ora.crsd -init
Stop of resource "ora.crsd -init" failed
Cause
Diskgroup is succesfully created, root.sh fails at generating the OCR keys when invoking ocrconfig because userid used for ASMlib driver differs from grid software owner.

Oracle Grid infrastructure installation generates a log file in $GRID_HOME/cfgtoollogs/crsconfig.
Per log file 'rootcrs_<hostname>.log':


2009-10-21 12:57:26: Querying for existing CSS voting disks
2009-10-21 12:57:26: Performing initial configuration for cluster
2009-10-21 12:57:28: Start of resource "ora.ctssd -init" Succeeded
2009-10-21 12:57:28: Configuring ASM via ASMCA
2009-10-21 12:57:28: Executing as oragrid: /u01/app/11.2.0/grid/bin/asmca -silent -diskGroupName DATA -diskList ORCL:ASMD40,ORCL:ASMD41 -redundancy EXTERNAL -configureLocalASM
2009-10-21 12:57:28: Running as user oragrid: /u01/app/11.2.0/grid/bin/asmca -silent -diskGroupName DATA -diskList ORCL:ASMD40,ORCL:ASMD41 -redundancy EXTERNAL -configureLocalASM
2009-10-21 12:57:28:   Invoking "/u01/app/11.2.0/grid/bin/asmca -silent -diskGroupName DATA -diskList ORCL:ASMD40,ORCL:ASMD41 -redundancy EXTERNAL -configureLocalASM" as user "oragrid"
2009-10-21 12:58:02: Creating or upgrading OCR keys
2009-10-21 12:58:04: Command return code of 255 (65280) from command: /u01/app/11.2.0/grid/bin/ocrconfig -upgrade oragrid oinstall
2009-10-21 12:58:04: Failed to create Oracle Cluster Registry configuration, rc 255
2009-10-21 12:58:04: Exiting exclusive mode
2009-10-21 12:58:04: Command return code of 1 (256) from command: /u01/app/11.2.0/grid/bin/crsctl stop resource ora.crsd -init
2009-10-21 12:58:04: Stop of resource "ora.crsd -init" failed

In above example the grid software owner is oragrid but the ASMlib device driver has been configured with owner oracle (different user) despite both being part of dba group


% /usr/sbin/oracleasm configure
ORACLEASM_ENABLED=true
ORACLEASM_UID=oracle
ORACLEASM_GID=dba
ORACLEASM_SCANBOOT=true
ORACLEASM_SCANORDER=""
ORACLEASM_SCANEXCLUDE=""

Solution
Please note all tasks below are done as the root user!

1. To Resolve this problem, first deconfigure Grid.
Run the following script on any node that had root.sh executed on :


% $GRID_HOME/crs/install/rootcrs.pl -deconfig -force
2. Reconfigure ASMlib
Delete the ASMlib device/s used for OCR and voting ASM diskgroup, in example above the devices used are:
ORCL:ASMD40
ORCL:ASMD41


% /usr/sbin/oracleasm deletedisk ASMD40
% /usr/sbin/oracleasm deletedisk ASMD41
3.  Re-configure ASMlib driver to use the correct userid, in this case "oragrid" user is the software owner for Grid Infrastructure installation, ensure followings are done on all nodes:


% /usr/sbin/oracleasm configure -u oragrid
% /usr/sbin/oracleasm configure
4.  stop/start ASMlib on all nodes


% /usr/sbin/oracleasm exit
% /usr/sbin/oracleasm init
% /usr/sbin/oracleasm scandisks
5.  Create the ASMlib disks again used for Grid Infrastructure installation


% oracleasm createdisk ASMD40 /dev/sdh1
% oracleasm createdisk ASMD41 /dev/sdi1
Then run the scandisks again on all nodes ->


% /usr/sbin/oracleasm scandisks


RESTORE VOTING DISK
------------------------------------------
Ensure you are logged in as root user while executing the commands:
1. Start up the Clusterware stack on the first node of the cluster in exclusive mode, using the following command:
./crsctl start crs -excl
2. Use the following command to restore the Voting disk to a diskgroup irrespective of where it was stored earlier either on a diskgroup or on shared storage:
./crsctl replace voting disk +DISKGROP_NAME
3. If the Voting disk was previously stored on shared storage and you want to restore it on shared storage, use the following command, applicable only in 11g R2:
./crsctl query css votedisk

Note down the File Universal Id (FUID) of the voting disk
./crsctl delete css votedisk <FUID>
./crsctl add css votedisk <destination_for_votedisk>
4. Bring down the cluster on the local node and start up the cluster stack
subsequently, by using the following set of commands as root user:
./crsctl stop crs -f
./crsctl start crs
```  

--------------------------------------------------------------------------------
			A	S	M
   -----------------------------------------------------------------------------

### asmdu.sh
```
#!/bin/bash
#
# du of each subdirectory in a directory for ASM
#
D=$1
 
if [[ -z $D ]]
then
 echo "Please provide a directory !"
 exit 1
fi
 
(for DIR in `asmcmd ls ${D}`
 do
     echo ${DIR} `asmcmd du ${D}/${DIR} | tail -1`
 done) | awk -v D="$D" ' BEGIN {  printf("\n\t\t%40s\n\n", D " subdirectories size")           ;
                                  printf("%25s%16s%16s\n", "Subdir", "Used MB", "Mirror MB")   ;
                                  printf("%25s%16s%16s\n", "------", "-------", "---------")   ;}
                               {
                                  printf("%25s%16s%16s\n", $1, $2, $3)                         ;
                                  use += $2                                                    ;
                                  mir += $3                                                    ;
                               }
                         END   { printf("\n\n%25s%16s%16s\n", "------", "-------", "---------");
                                 printf("%25s%16s%16s\n\n", "Total", use, mir)                 ;} '

```
## Sample command
```
. oraenv
+ASM1
./asmdu.sh DATA

Another one with many archivelogs directories :

[oracle@]$./asmdu.sh FRA/THE_DB/ARCHIVELOG/
```

### ASM ADD DISK
```
Step1. The current disk group configuration

set lines 255
set pagesize 3000
 col path for a35
 col Diskgroup for a15
 col DiskName for a20
 col disk# for 999
 col total_mb for 999,999,999
 col free_mb for 999,999,999
 compute sum of total_mb on DiskGroup
 compute sum of free_mb on DiskGroup
 break on DiskGroup skip 1 on report -
 set pages 255
 select a.name DiskGroup, b.disk_number Disk#, b.name DiskName, b.total_mb, b.free_mb, b.path, b.header_status
  from v$asm_disk b, v$asm_diskgroup a
    where a.group_number (+) =b.group_number
and a.name like '%PAMPRD%'
    order by b.group_number, b.disk_number, b.name
;

DISKGROUP       DISK# DISKNAME                 TOTAL_MB      FREE_MB PATH                                HEADER_STATU
--------------- ----- -------------------- ------------ ------------ ----------------------------------- ------------
PPROD24_DATA_01     0 ASMPPROD24DATA01           53,549        7,609 ORCL:ASMPPROD24DATA01               MEMBER
PPROD24_DATA_01     1 ASMPPROD24DATA02           53,549        7,612 ORCL:ASMPPROD24DATA02               MEMBER
PPROD24_DATA_01     2 ASMPPROD24DATA03           53,549        7,616 ORCL:ASMPPROD24DATA03               MEMBER
--------
PPROD24_DATA_01    34 ASMPPROD24DATA35           53,549        7,731 ORCL:ASMPPROD24DATA35               MEMBER


PPROD24 ---- 20GB storage shared and Replicated (4 x 52.29GB)
/dev/sddlmiao1
/dev/sddlmiap1
/dev/sddlmjaa1
/dev/sddlmjab1
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Step 2: Query the disk

Run on all clustered nodes:
sudo /usr/sbin/oracleasm querydisk /dev/sddlmiao1
sudo /usr/sbin/oracleasm querydisk /dev/sddlmiap1
sudo /usr/sbin/oracleasm querydisk /dev/sddlmjaa1
sudo /usr/sbin/oracleasm querydisk /dev/sddlmjab1

[grid@csmper-cls07 ~]$ sudo /usr/sbin/oracleasm querydisk /dev/sddlmiao1
Device "/dev/sddlmiao1" is not marked as an ASM disk
[grid@csmper-cls07 ~]$ sudo /usr/sbin/oracleasm querydisk /dev/sddlmiap1
Device "/dev/sddlmiap1" is not marked as an ASM disk
[grid@csmper-cls07 ~]$ sudo /usr/sbin/oracleasm querydisk /dev/sddlmjaa1
Device "/dev/sddlmjaa1" is not marked as an ASM disk
[grid@csmper-cls07 ~]$ sudo /usr/sbin/oracleasm querydisk /dev/sddlmjab1
Device "/dev/sddlmjab1" is not marked as an ASM disk


sudo /usr/sbin/oracleasm listdisks | grep PPROD24

sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA36 /dev/sddlmiao1
sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA37 /dev/sddlmiap1
sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA38 /dev/sddlmjaa1
sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA39 /dev/sddlmjab1

[grid@csmper-cls08 ~]$ sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA36 /dev/sddlmiao1
sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA39 /dev/sddlmjab1Writing disk header: done
Instantiating disk: done
[grid@csmper-cls08 ~]$ sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA37 /dev/sddlmiap1
Writing disk header: done
Instantiating disk: done
[grid@csmper-cls08 ~]$ sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA38 /dev/sddlmjaa1
Writing disk header: done
Instantiating disk: done
[grid@csmper-cls08 ~]$ sudo /usr/sbin/oracleasm createdisk  ASMPPROD24DATA39 /dev/sddlmjab1
Writing disk header: done
Instantiating disk: done

can for the new disk

Run on all clustered nodes:
sudo /usr/sbin/oracleasm scandisks /dev/sddlmiao1
sudo /usr/sbin/oracleasm scandisks /dev/sddlmiap1
sudo /usr/sbin/oracleasm scandisks /dev/sddlmjaa1
sudo /usr/sbin/oracleasm scandisks /dev/sddlmjab1

[grid@csmper-cls06 ~]$ sudo /usr/sbin/oracleasm scandisks /dev/sddlmiao1
Reloading disk partitions: done
Cleaning any stale ASM disks...
Scanning system for ASM disks...
Instantiating disk "ASMPPROD24DATA36"
[grid@csmper-cls06 ~]$ sudo /usr/sbin/oracleasm scandisks /dev/sddlmiap1
Reloading disk partitions: done
Cleaning any stale ASM disks...
Scanning system for ASM disks...
Instantiating disk "ASMPPROD24DATA37"
[grid@csmper-cls06 ~]$ sudo /usr/sbin/oracleasm scandisks /dev/sddlmjaa1
Reloading disk partitions: done
Cleaning any stale ASM disks...
Scanning system for ASM disks...
Instantiating disk "ASMPPROD24DATA38"
[grid@csmper-cls06 ~]$ sudo /usr/sbin/oracleasm scandisks /dev/sddlmjab1
Reloading disk partitions: done
Cleaning any stale ASM disks...
Scanning system for ASM disks...
Instantiating disk "ASMPPROD24DATA39"

 listdisks to make sure the disk is available from all the nodes

Run on all clustered nodes:
sudo /usr/sbin/oracleasm listdisks | grep ASMPPROD24DATA
...
ASMPPROD24DATA36
ASMPPROD24DATA37
ASMPPROD24DATA38
ASMPPROD24DATA39

Run on all clustered nodes:
Check if disks are marked as provisioned:
>sqlplus / as sysasm
select group_number,disk_number,label,header_status from v$asm_disk where label like 'ASMPPROD24%' order by header_status;

GROUP_NUMBER DISK_NUMBER LABEL              HEADER_STATU
------------ ----------- ------------------------------- -----------------------------------------------------
...
           0           0 ASMPPROD24DATA36                PROVISIONED
           0           3 ASMPPROD24DATA39                PROVISIONED
           0           2 ASMPPROD24DATA38                PROVISIONED
           0           1 ASMPPROD24DATA37                PROVISIONED


Add Disks to a Disk Group

Add disk to diskgroup and rebalance:
SQL>alter diskgroup PPROD24_DATA_01 add disk 'ORCL:ASMPPROD24DATA36' rebalance power 1;
SQL>alter diskgroup PPROD24_DATA_01 add disk 'ORCL:ASMPPROD24DATA37' rebalance power 1;
SQL>alter diskgroup PPROD24_DATA_01 add disk 'ORCL:ASMPPROD24DATA38' rebalance power 1;
SQL>alter diskgroup PPROD24_DATA_01 add disk 'ORCL:ASMPPROD24DATA39' rebalance power 1;
GROUP_NUMBER DISK_NUMBER LABEL                           HEADER_STATU
------------ ----------- ------------------------------- ------------
          49          36 ASMPPROD24DATA37                MEMBER
          49          37 ASMPPROD24DATA38                MEMBER
          49          38 ASMPPROD24DATA39                MEMBER
          49          35 ASMPPROD24DATA36                MEMBER
...


With the Real Application Clusters and Automatic Storage Management options

SYS@+ASM3 SQL>
set lines 255
set pagesize 3000
SYS@+ASM3 SQL>  col path for a35
SYS@+ASM3 SQL>  col Diskgroup for a15
 col DiskName for a20
 compute sum of total_mb on DiskGroup
 compute sum of free_mb on DiskGroup
 break on DiskGroup skip 1 on report -
 set pages 255
 select a.name DiskGroup, b.disk_number Disk#, b.name DiskName, b.total_mb, b.free_mb, b.path, b.header_status
 col disk# for 999
and a.name like '%PAMPRD%'
 order by b.group_number, b.disk_number, b.name;

 col total_mb for 999,999,999
 col free_mb for 999,999,999
 compute sum of total_mb on DiskGroup
 compute sum of free_mb on DiskGroup
 break on DiskGroup skip 1 on report -
 set pages 255
SP2-0158: unknown BREAK option "set"
 select a.name DiskGroup, b.disk_number Disk#, b.name DiskName, b.total_mb, b.free_mb, b.path, b.header_status
  from v$asm_disk b, v$asm_diskgroup a
    where a.group_number (+) =b.group_number
and a.name like '%PAMPRD%'
    order by b.group_number, b.disk_number, b.name
  6  ;

DISKGROUP       DISK# DISKNAME                 TOTAL_MB      FREE_MB PATH                                HEADER_STATUS
--------------- ----- -------------------- ------------ ------------ ----------------------------------- ------------------------------------
PAMPRD_ARC_01       0 ASMPAMPRDARC01             53,548       53,344 ORCL:ASMPAMPRDARC01                 MEMBER
PAMPRD_DATA_01      0 ASMPAMPRDDATA01            53,548        2,220 ORCL:ASMPAMPRDDATA01                MEMBER
PAMPRD_DATA_01      1 ASMPAMPRDDATA02            53,548        2,248 ORCL:ASMPAMPRDDATA02                MEMBER
PAMPRD_DATA_01      2 ASMWSESBP_FLASH_02         53,548        2,236 ORCL:ASMWSESBP_FLASH_02             MEMBER

SYS@+ASM3 SQL> select group_number,disk_number,label,header_status from v$asm_disk where header_status like '%PROVISIONED%';

no rows selected

SYS@+ASM3 SQL> select group_number,disk_number,label,header_status from v$asm_disk where header_status like '%FORMER%';

GROUP_NUMBER DISK_NUMBER LABEL                                                                                         HEADER_STATUS
------------ ----------- --------------------------------------------------------------------------------------------- ------------------------------------
           0           0 ASMARISBPARC02                                                                                FORMER
           0           1 ASMARISBSARC02                                                                                FORMER
           0          33 PORCFPEBKP03                                                                                  FORMER
           0           3 ASMICCRTCP_ARC_03                                                                             FORMER
           0           2 ASMICCRTCP_ARC_02                                                                             FORMER

SYS@+ASM3 SQL> select group_number,disk_number,label,header_status, os_mb from v$asm_disk where header_status like '%FORMER%';

GROUP_NUMBER DISK_NUMBER LABEL                                                                                         HEADER_STATUS                              OS_MB
------------ ----------- --------------------------------------------------------------------------------------------- ------------------------------------        ----------
           0           0 ASMARISBPARC02                                                                                FORMER                                      7649
           0           1 ASMARISBSARC02                                                                                FORMER                                      7649
           0          33 PORCFPEBKP03                                                                                  FORMER                                     53549
           0           3 ASMICCRTCP_ARC_03                                                                             FORMER                                      7649
           0           2 ASMICCRTCP_ARC_02                                                                             FORMER                                      7649


SYS@+ASM3 SQL> col label form a60
SYS@+ASM3 SQL> /

GROUP_NUMBER DISK_NUMBER LABEL                                                        HEADER_STATUS                             OS_MB
------------ ----------- ------------------------------------------------------------ ------------------------------------ ----------
           0           0 ASMARISBPARC02                                               FORMER                                     7649
           0           1 ASMARISBSARC02                                               FORMER                                     7649
           0          33 PORCFPEBKP03                                                 FORMER                                    53549
           0           3 ASMICCRTCP_ARC_03                                            FORMER                                     7649
           0           2 ASMICCRTCP_ARC_02                                            FORMER                                     7649

grid@csmper-cls08:+ASM3:/home/grid
$ sqlplus / as sysasm

SQL*Plus: Release 11.2.0.2.0 Production on Wed Aug 26 15:23:02 2015

Copyright (c) 1982, 2010, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Real Application Clusters and Automatic Storage Management options

SYS@+ASM3 SQL> alter diskgroup PAMPRD_DATA_01 add disk 'ORCL:PORCFPEBKP03';

Diskgroup altered.
```


### ASM information

```
SET LINESIZE  145
SET PAGESIZE  9999
SET VERIFY    off
COLUMN group_name             FORMAT a20           HEAD 'Disk Group|Name'
COLUMN sector_size            FORMAT 99,999        HEAD 'Sector|Size'
COLUMN block_size             FORMAT 99,999        HEAD 'Block|Size'
COLUMN allocation_unit_size   FORMAT 999,999,999   HEAD 'Allocation|Unit Size'
COLUMN state                  FORMAT a11           HEAD 'State'
COLUMN type                   FORMAT a6            HEAD 'Type'
COLUMN total_mb               FORMAT 999,999,999   HEAD 'Total Size (MB)'
COLUMN used_mb                FORMAT 999,999,999   HEAD 'Used Size (MB)'
COLUMN pct_used               FORMAT 999.99        HEAD 'Pct. Used'

break on report on disk_group_name skip 1
compute sum label "Grand Total: " of total_mb used_mb on report

SELECT
    name                                     group_name
  , sector_size                              sector_size
  , block_size                               block_size
  , allocation_unit_size                     allocation_unit_size
  , state                                    state
  , type                                     type
  , total_mb                                 total_mb
  , (total_mb - free_mb)                     used_mb
  , ROUND((1- (free_mb / total_mb))*100, 2)  pct_used
FROM v$asm_diskgroup
ORDER BY name;
```
### Sample Output
```
Disk Group            Sector   Block   Allocation
Name                    Size    Size    Unit Size State       Type   Total Size (MB) Used Size (MB) Pct. Used
-------------------- ------- ------- ------------ ----------- ------ --------------- -------------- ---------
DATA                     512   4,096    4,194,304 MOUNTED     NORMAL     108,859,392     95,291,888     87.54
DBFS_DG                  512   4,096    4,194,304 MOUNTED     NORMAL       1,038,240        123,576     11.90
RECO                     512   4,096    4,194,304 MOUNTED     NORMAL      27,221,760      9,792,836     35.97
                                                                     --------------- --------------
Grand Total:                                                             137,119,392    105,208,300
```

### If you are on Exadata or Oracle Database Appliance, the disk is put back online automatically. In all other cases an ASM administrator has to put the disk online with an alter diskgroup command, like this:

```
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
where name like '%RECO_CD_11_IORSSC03_ADM%'
order by group_number
,        disk_number
/

```
```
alter diskgroup DATA online all;
```
```
alter diskgroup DATA online disk '<ORCL:DISK077>';
```

### ASM DISK Throughput
```
SYS@+ASM1 SQL> select * from (select a.DBNAME as "DB Name",
    sum(a.READS) as "Read Request",
    sum(a.WRITES) as "Write Requests",
    sum (a.BYTES_READ/1024/1024) as "Total Read MB",
    sum(a.BYTES_WRITTEN/1024/1024) "Total Write MB",
    DECODE(sum(a.BYTES_WRITTEN/1024/1024),0,0,sum (a.BYTES_READ/1024/1024)/sum(a.BYTES_WRITTEN/1024/1024)) "Throughput"
    from v$asm_disk_iostat a, v$asm_disk b
where  a.GROUP_NUMBER=b.GROUP_NUMBER
    and a.DISK_NUMBER=b.DISK_NUMBER
     group by a.DBNAME
   order by 4 desc)
   where rownum <=10;
```

```
select * from (select a.DBNAME as "DB Name",
    b.PATH as "Device",
    a.READS as "Read Request",
    a.WRITES as "Write Requests",
    a.BYTES_READ/1024/1024 as "Total Read MB",
    a.BYTES_WRITTEN/1024/1024 "Total Write MB"
    from v$asm_disk_iostat a, v$asm_disk b
    where  a.GROUP_NUMBER=b.GROUP_NUMBER
    and a.DISK_NUMBER=b.DISK_NUMBER
   order by 3 desc)
   where rownum <=10;
```
### ASM DISKGROUP STATUS
```
SET LINESIZE  145
SET PAGESIZE  9999
SET VERIFY    off
COLUMN group_name             FORMAT a20           HEAD 'Disk Group|Name'
COLUMN sector_size            FORMAT 99,999        HEAD 'Sector|Size'
COLUMN block_size             FORMAT 99,999        HEAD 'Block|Size'
COLUMN allocation_unit_size   FORMAT 999,999,999   HEAD 'Allocation|Unit Size'
COLUMN state                  FORMAT a11           HEAD 'State'
COLUMN type                   FORMAT a6            HEAD 'Type'
COLUMN total_mb               FORMAT 999,999,999   HEAD 'Total Size (MB)'
COLUMN used_mb                FORMAT 999,999,999   HEAD 'Used Size (MB)'
COLUMN pct_used               FORMAT 999.99        HEAD 'Pct. Used'

break on report on disk_group_name skip 1
compute sum label "Grand Total: " of total_mb used_mb on report

SELECT
    name                                     group_name
  , sector_size                              sector_size
  , block_size                               block_size
  , allocation_unit_size                     allocation_unit_size
  , state                                    state
  , type                                     type
  , total_mb                                 total_mb
  , (total_mb - free_mb)                     used_mb
  , ROUND((1- (free_mb / total_mb))*100, 2)  pct_used
FROM v$asm_diskgroup
ORDER BY name;
```
```
$ srvctl status diskgroup -g DATA
```
### DETAILED INFO
```
select inst.inst_id,di.instname,g.name,di.disk_number,
sum(di.reads),
sum(di.writes),
sum(di.read_time)*1000,sum(di.write_time)*1000,sum(di.bytes_read)/1024,
sum(di.bytes_written)/1024,sum(di.reads+di.writes) 
from gv$asm_disk_stat ds, 
gv$asm_disk_iostat di, 
gv$instance inst, 
gv$asm_diskgroup_stat g 
where ds.inst_id=di.inst_id 
and ds.group_number=di.group_number 
and ds.disk_number = di.disk_number 
and ds.mount_status ='CACHED' 
and inst.inst_id=ds.inst_id 
and g.inst_id=inst.inst_id 
and g.group_number=di.group_number 
and di.instname not like '+ASM%'
group by inst.inst_id,di.instname,g.name,di.disk_number;
```

### ASMDATAFILE INFO
```
SET TERMOUT OFF;
COLUMN current_instance NEW_VALUE current_instance NOPRINT;
SELECT rpad(sys_context('USERENV', 'INSTANCE_NAME'), 17) current_instance FROM dual;
SET TERMOUT ON;

PROMPT 
PROMPT +------------------------------------------------------------------------+
PROMPT | Report   : ASM Files                                                   |
PROMPT | Instance : &current_instance                                           |
PROMPT +------------------------------------------------------------------------+

SET ECHO        OFF
SET FEEDBACK    6
SET HEADING     ON
SET LINESIZE    180
SET PAGESIZE    50000
SET TERMOUT     ON
SET TIMING      OFF
SET TRIMOUT     ON
SET TRIMSPOOL   ON
SET VERIFY      OFF

CLEAR COLUMNS
CLEAR BREAKS
CLEAR COMPUTES

COLUMN full_path              FORMAT a75                  HEAD 'ASM File Name / Volume Name / Device Name'
COLUMN system_created         FORMAT a8                   HEAD 'System|Created?'
COLUMN bytes                  FORMAT 9,999,999,999,999    HEAD 'Bytes'
COLUMN space                  FORMAT 9,999,999,999,999    HEAD 'Space'
COLUMN type                   FORMAT a18                  HEAD 'File Type'
COLUMN redundancy             FORMAT a12                  HEAD 'Redundancy'
COLUMN striped                FORMAT a8                   HEAD 'Striped'
COLUMN creation_date          FORMAT a20                  HEAD 'Creation Date'
COLUMN disk_group_name        noprint

BREAK ON report ON disk_group_name SKIP 1

COMPUTE sum LABEL ""              OF bytes space ON disk_group_name
COMPUTE sum LABEL "Grand Total: " OF bytes space ON report

SELECT
    CONCAT('+' || db_files.disk_group_name, SYS_CONNECT_BY_PATH(db_files.alias_name, '/')) full_path
  , db_files.bytes
  , db_files.space
  , NVL(LPAD(db_files.type, 18), '<DIRECTORY>')  type
  , db_files.creation_date
  , db_files.disk_group_name
  , LPAD(db_files.system_created, 4) system_created
FROM
    ( SELECT
          g.name               disk_group_name
        , a.parent_index       pindex
        , a.name               alias_name
        , a.reference_index    rindex
        , a.system_created     system_created
        , f.bytes              bytes
        , f.space              space
        , f.type               type
        , TO_CHAR(f.creation_date, 'DD-MON-YYYY HH24:MI:SS')  creation_date
      FROM
          v$asm_file f RIGHT OUTER JOIN v$asm_alias     a USING (group_number, file_number)
                                   JOIN v$asm_diskgroup g USING (group_number)
    ) db_files
WHERE db_files.type IS NOT NULL
START WITH (MOD(db_files.pindex, POWER(2, 24))) = 0
    CONNECT BY PRIOR db_files.rindex = db_files.pindex
UNION
SELECT
    '+' || volume_files.disk_group_name ||  ' [' || volume_files.volume_name || '] ' ||  volume_files.volume_device full_path
  , volume_files.bytes
  , volume_files.space
  , NVL(LPAD(volume_files.type, 18), '<DIRECTORY>')  type
  , volume_files.creation_date
  , volume_files.disk_group_name
  , null
FROM
    ( SELECT
          g.name               disk_group_name
        , v.volume_name        volume_name
        , v.volume_device       volume_device
        , f.bytes              bytes
        , f.space              space
        , f.type               type
        , TO_CHAR(f.creation_date, 'DD-MON-YYYY HH24:MI:SS')  creation_date
      FROM
          v$asm_file f RIGHT OUTER JOIN v$asm_volume    v USING (group_number, file_number)
                                   JOIN v$asm_diskgroup g USING (group_number)
    ) volume_files
WHERE volume_files.type IS NOT NULL
/
```
### ASM MAPPING
```
cat chk_asm_mapping.sh
/etc/init.d/oracleasm querydisk -d `/etc/init.d/oracleasm listdisks -d` |
cut -f2,10,11 -d" " | perl -pe 's/"(.*)".*\[(.*), *(.*)\]/$1 $2 $3/g;' |
while read v_asmdisk v_minor v_major
do
v_device=`ls -la /dev | grep " $v_minor, *$v_major " | awk '{print $10}'`
echo "ASM disk $v_asmdisk based on /dev/$v_device [$v_minor, $v_major]"
done
```

### display information on ASM
```
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

### list v$asm_files by size - should run against ASM instance for speed. (db instance can be slow)
```
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

### display information on ASM
```
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
where name like '%RECO_CD_11_IORSSC03_ADM%'
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

### Other Checks
```
COLUMN disk_group_name FORMAT a20 HEAD 'Disk Group Name'
    COLUMN file_name FORMAT a30 HEAD 'File Name'
    COLUMN bytes FORMAT 9,999,999,999,999 HEAD 'Bytes'
    COLUMN space FORMAT 9,999,999,999,999 HEAD 'Space'
    COLUMN type FORMAT a18 HEAD 'File Type'
    COLUMN redundancy FORMAT a12 HEAD 'Redundancy'
    COLUMN striped FORMAT a8 HEAD 'Striped'
    COLUMN creation_date FORMAT a20 HEAD 'Creation Date'

    break on report on disk_group_name skip 1
    compute sum label "" of bytes space on disk_group_name
    compute sum label "Grand Total: " of bytes space on report

    SELECT g.name disk_group_name
    , a.name file_name
    , f.bytes bytes
    , f.space space
    , f.type type
    , TO_CHAR(f.creation_date, 'DD-MON-YYYY HH24:MI:SS') creation_date
    FROM
    v$asm_file f JOIN v$asm_alias a USING (group_number, file_number)
    JOIN v$asm_diskgroup g USING (group_number)
    WHERE
    system_created = 'Y'
    ORDER BY g.name, file_number
    /
```
```
select g.name_disk_group, c.*
from v$asm_client c, v$asm_diskgroup G
where c.group_number=g.group_number
and g.name='DATA';
```
```
ASMCMD> lsct -g DATA
Instance_ID  DB_Name   Status     Software_Version  Compatible_version  Instance_Name  Disk_Group
          1  +ASM      CONNECTED        12.1.0.2.0          12.1.0.2.0  +ASM1          DATA
          2  +ASM      CONNECTED        12.1.0.2.0          12.1.0.2.0  +ASM2          DATA
          1  DBDR1_01  CONNECTED        11.2.0.4.0          11.2.0.4.0  DBDR1_1        DATA
          1  DBDW1_01  CONNECTED        11.2.0.4.0          11.2.0.4.0  DBDW1_1        DATA
          1  DBIC1_01  CONNECTED        11.2.0.4.0          11.2.0.4.0  DBIC1_1        DATA
          1  DBLH1_01  CONNECTED        11.2.0.4.0          11.2.0.4.0  DBLH1_1        DATA
```

### Database file ..... in ASM belong to

```
select lpad('>',LEVEL*3,'=')||name 
from v$asm_alias
start with name = '+DATA/TSIM1_01/DATAFILE/<xxxxxxx>'
connect by prior parent_index = reference_index ;
```

### Disk Operation
```
SELECT group_number, operation, state, power, est_minutes FROM v$asm_operation;
```

### altered the rebalance power for the diskgroup using the following command:
```
alter diskgroup DISKGROUP rebalance power 5; 
```
### Long Operation

V$ASM_OPERATION -- display one row for every long running operation executing in the ASM instance.
```
select g.name_disk_group, o.*
from v$asm_operation o, v$asm_diskgroup g
where o.group_number=g.group_number;

estimate --- number of allocation unit that require the movement
SOFAR -- how many moved at a rate (EST_RATE --unit minute)
```
```
asmcmd>lsop
```
### Full path name of the files in ASM diskgroups [ID 888943.1]
```
Connect to the ASM instance:
in 10g: sqlplus / as sysdba
in 11g: sqlplus / as sysasm

Then perform the following query:  

SELECT gnum, filnum, concat('+'||gname,sys_connect_by_path(aname, '/'))
FROM (SELECT g.name gname, a.parent_index pindex, a.name aname, 
a.reference_index rindex, a.group_number gnum,a.file_number filnum
FROM v$asm_alias a,v$asm_diskgroup g
WHERE a.group_number = g.group_number)
START WITH (mod(pindex, power(2, 24))) = 0
CONNECT BY PRIOR rindex = pindex;
```
### List hidden ASM parameters 
```
select a.ksppinm "name", c.ksppstvl "value"
  from x$ksppi a, x$ksppcv b, x$ksppsv c
 where a.indx = b.indx and a.indx = c.indx
 and ksppinm like '_asm%'
order by a.ksppinm; 
```
### Generate a list of all the asm files / directories / aliasses for a given database
--- RUN into ASM instance 
```
column full_alias_path format a75
column file_type format a15
select concat('+'||gname, sys_connect_by_path(aname, '/')) full_alias_path, 
       system_created, alias_directory, file_type
from ( select b.name gname, a.parent_index pindex, a.name aname, 
              a.reference_index rindex , a.system_created, a.alias_directory,
              c.type file_type
       from v$asm_alias a, v$asm_diskgroup b, v$asm_file c
       where a.group_number = b.group_number
             and a.group_number = c.group_number(+)
             and a.file_number = c.file_number(+)
             and a.file_incarnation = c.incarnation(+)
     )
start with (mod(pindex, power(2, 24))) = 0
            and rindex in 
                ( select a.reference_index
                  from v$asm_alias a, v$asm_diskgroup b
                  where a.group_number = b.group_number
                        and (mod(a.parent_index, power(2, 24))) = 0
                        and a.name = '&DATABASENAME'
                )
connect by prior rindex = pindex;
```

### QUERY TO FIND THE FILES IN USE BY AN ASM INSTANCE 
```
col full_path format a50
col full_alias_path format a50
SELECT concat('+'||gname, sys_connect_by_path(aname, '/')) full_alias_path
FROM (SELECT g.name gname, a.parent_index pindex, a.name aname,
a.reference_index rindex FROM v$asm_alias a, v$asm_diskgroup g
WHERE a.group_number = g.group_number)
START WITH (mod(pindex, power(2, 24))) = 0 CONNECT BY PRIOR rindex = pindex;
```
### ASM_IMBALANCE 

Execute the following query in sqlplus to determine the current disk sizes of the ASM diskgroup and the amount of additional disks to request:
```
col name form a30
select dg.name
,      dk.os_mb "Disk Size MB"
,      count(*) "NR_Disks"
,      trunc(count(*)/100*30) "DisksToRequest"
from   v$asm_diskgroup dg
,      v$asm_disk dk
where  dg.group_number = dk.group_number
and    dg.name = upper('&1')
group by dg.name,dk.os_mb
/
```
old   7: and    dg.name = upper('&1')
new   7: and    dg.name = upper('porcfpedata')

NAME                           Disk Size MB   NR_Disks DisksToRequest
------------------------------ ------------ ---------- --------------
PORCFPEDATA                           53549         15              4

dgs.sql

```
select name,TOTAL_MB/1024,FREE_MB/1024,(FREE_MB/TOTAL_MB)*100 "free %" from  v$asm_diskgroup;
select name,TOTAL_MB/1024,FREE_MB/1024,(FREE_MB/TOTAL_MB)*100 "free %" from  v$asm_diskgroup where (FREE_MB/TOTAL_MB)*100<20;
```
### Growth [ASM/NON-ASM] NOTE: need to filter by dest_id further, if there is multiple archive log destinations

Archivelog size each day:
```
SQL> select trunc(COMPLETION_TIME) TIME, SUM(BLOCKS * BLOCK_SIZE)/1024/1024 SIZE_MB from V$ARCHIVED_LOG group by trunc (COMPLETION_TIME) order by 1;
```
Archivelog size each hour:
```
SQL> alter session set nls_date_format = 'YYYY-MM-DD HH24';
```
```
SQL> select trunc(COMPLETION_TIME,'HH24') TIME, SUM(BLOCKS * BLOCK_SIZE)/1024/1024 SIZE_MB from V$ARCHIVED_LOG group by trunc (COMPLETION_TIME,'HH24') order by 1;
```
```
Select dg.name,dg.allocation_unit_size/1024/1024 "AU(Mb)",min(d.free_mb) Min,
    max(d.free_mb) Max, avg(d.free_mb) Avg
    from v$asm_disk d, v$asm_diskgroup dg
    where d.group_number = dg.group_number
    group by dg.name, dg.allocation_unit_size/1024/1024;
```
```
alter diskgroup data rebalance power 4;
alter system set "_asm_imbalance_tolerance"=0;
alter diskgroup data rebalance power 4;
```
### ASMSNMP 

Could not validate ASMSNMP password due to following error- “ORA-01031: insufficient privileges”
ORA-01990: error opening password file ‘/u02/app/11.2.0/grid/dbs/orapw’
ORA-15306: ASM password file update failed on at least one node
ORA-01918: user ‘ASMSNMP’ does not exist
```
Mostly the reason is missing asmsnmp user or missing password file in ASM instances.

1. How to create asmsnmp user

First Check which all users are present for ASM 

SQL> select * from v$pwfile_users;

OR
ASMCMD> lspwusr

 
A) Create a password file if not already present

$orapwd file=/u01/app/11.2.0/grid/dbs/orapw+ASM password=<sys_password>

Remember below important point about password file in ASM

If there are two nodes with +ASM1 running on node 1 and +ASM2 running on node2.

Pre 11gR2 
+++++++++++++
Password file on Node1: orapw+ASM1
Password file on Node2: orapw+ASM2

11gR2
++++++++++
Password file on Node1: orapw+ASM
Password file on Node2: orapw+ASM

B) Copy the password file to other nodes

$ scp orapw+ASM <Node2>:/u01/app/11.2.0/grid/dbs/
 
copy the password files to other nodes if you have more than 2 nodes in the RAC.

C) create user and give sysdba privileges
SQL>create user asmsnmp identified by <make_asmsnmp_password>;

SQL> grant sysdba to asmsnmp;

Check again for users in ASM  >>

SQL> select * from v$pwfile_users;
USERNAME              SYSDB                SYSOP             SYSAS
---------------------------------------------------------------------------------------
SYS                    TRUE                TRUE                 TRUE
ASMSNMP                TRUE                FALSE                FALSE

SQL> show parameter pass

NAME                                   TYPE                            VALUE
--------------------------------------------------------------------------------------------
remote_login_passwordfile             string                          EXCLUSIVE

The ASM instance parameter REMOTE_LOGIN_PASSWORDFILE has to be set to EXCLUSIVE or you will get an ORA-01999 error.

Also you can check  user detials from asmcmd

ASMCMD> lspwusr
Username        sysdba         sysoper       sysasm

SYS             TRUE           TRUE          FALSE
ASMSNMP         TRUE           FALSE         FALSE
```

### How to Change password for ASMSNMP
```
ASMCMD> lspwusr
Username sysdba sysoper sysasm
     SYS   TRUE    TRUE   TRUE
 ASMSNMP   TRUE   FALSE  FALSE

ASMCMD> pwget --asm
+DBFS_DG/orapwASM

ASMCMD> cd DBFS_DG/
ASMCMD> ls
ASM/
PCAL1_02/
_MGMTDB/
iorscl01/
orapwasm
ASMCMD> ls -ltr
WARNING:option 'r' is deprecated for 'ls'
please use 'reverse'

Type      Redund  Striped  Time             Sys  Name
                                            Y    ASM/
                                            Y    PCAL1_02/
                                            Y    _MGMTDB/
                                            Y    iorscl01/
PASSWORD  HIGH    COARSE   APR 20     2015  N    orapwasm => +DBFS_DG/ASM/PASSWORD/pwdasm.256.877529015

ASMCMD> orapwusr --modify --password ASMSNMP
Enter password: *******
(give new password and press enter)

orapwusr attempts to update passwords on all nodes in a cluster. The command requires the SYSASM privilege to run. A user logged in as SYSDBA cannot change its password using this command.

Test new asmsnmp password >>

$ sqlplus sys/asmsnmp as sysdba

```  

### Check Diskgroup before dropping/replacing 
```
with creating ASM volumes, ensure that the ASM instance on the local node is active, and the required ASM disk group is already created and mounted (as the ASM volumes are created within a disk group).

command prompt, run ./asmca command from the $GRID_HOME/bin               -GUI

ASMCMD [+] > volcreate -G DG_FLASH -s 1G advm_vg
 -G represents the name of the disk group that will hold the volume.
 -s represents the size of the volume to be created.

 ASMCMD [+] volinfo -G DG_FLASH advm_vg
###################################################################################################
Option                                                  Description 							Example
###################################################################################################
volcreate 		 Creates an ASM volume within aspecified disk group                             volcreate -G DG_FLASH -s 1G
voldelete			 Removes an ASM volume 				voldelete -G DG_FLASH advm_vg                
volinfo 			 Displays information about volume               		                 volinfo -G DG_FLASH advm_vg
voldisable                                  	Disables volume in a mounted disk group        			 voldisable -G DG_FLASH advm_vg
volenable                                  	Enables volume in a mounted disk group                                                volenable -G DG_FLASH advm_vg
volresize                                    	Resizes an ASM volume                                                                         volresize -G DG_FLASH -s 2G advm_vg
volstat 			Reports volume I/O statistics 					 volstat -G DG_FLASHASM
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

SQL> ALTER DISKGROUP dg_flash ADD VOLUME advm_vg SIZE 1G;
The preceding command will create a new ASM volume with 1G size under dg_flash disk group.
SQL> ALTER DISKGROUP dg_flash RESIZE VOLUME advm_vg SIZE 2G;
The preceding command will resize the existing volume advm_vg to 2G.
SQL> ALTER DISKGROUP dg_flash DISABLE VOLUME advm_vg;
SQL> ALTER DISKGROUP dg_flash ENABLE VOLUME advm_vg;
The preceding command will enable volume.
SQL> ALTER DISKGROUP ALL DISABLE VOLUME ALL;
The preceding command will disable all volumes in all disk groups.
SQL> ALTER DISKGROUP dg_flash DROP VOLUME advm_vg;
The preceding command will drop a volume.
```
### Creating an ACFS filesystem with ASMCMD
```
1. Before we start creating the ACFS with ASMCMD, ensure the ADVM is already configured. 
Ex: volcreate -G DG_FLASH -s 1G advm_vg

Get the volinfo :
volinfo -G DG_FLASH advm_vg
2. Create a filesystem using the operating system-specific filesystem creation command.
mkfs -t acfs -b 4k /dev/asm/advm_vg2-86

3. Map the mount point through the Oracle ACFS mount registry.
Before registering the filesystem with the ACFS mount registry option, create the required OS directory using the following 
example, under $ORACLE_BASE/acfsmounts and set appropriate permissions to the directory. 
To execute the following, you need to be the root user on the system:
mkdir -p $ORACLE_BASE/acfsmounts/db_home
chmod 775 -R $ORACLE_BASE/acfsmounts/db_home
chown -R oracle:oinstall $ORACLE_BASE/acfsmounts/db_home

register the filesystem as a cluster resource on the required
raclinux1 and raclinux2 using the following example:
/sbin/acfsutil registry -a -f -n raclinux1,raclinux2 /dev/asm/advm_vg-86 /u01/app/oracle/acfsmounts/db_home

4. Mount the specific mount point using the operating system-specific
command, for example, the mount command.

/bin/mount -t acfs /dev/asm/advm_vg2-86 /u01/app/oracle/acfsmounts/db_home
```

### Creating a snapshot
```
Using the following acfsutil snap create command, we will create a snapshot of db_home filesystem:
/sbin/acfsutil snap create acfs_snap1 /u01/app/oracle/acfsmounts/db_home
acfsutil snap create: Snapshot operation is complete.
After the command is successfully executed, a snapshot directory named acfs_snap1 of db_home ACFS filesystem is created under the .ACFS/snap location as
shown in the following output. 
ls -la /u01/app/oracle/acfsmounts/db_home/.ACFS/snaps

info
=====
/sbin/acfsutil info fs

SQL
========
v$asm_acfssnapshots dynamic view to obtain information about the newly created
snapshot. Use the following SQL statement:
select substr(fs_name,1,20) FS,vol_device device,substr
(snap_name,1,30) SNAP_NAME,create_time from v$asm_acfssnapshots;

Removing a snapshot
====================
To remove an existing snapshot created on an ACFS filesystem, use the following command:
/sbin/acfsutil snap delete acfs_snap1 /u01/app/oracle/acfsmounts/db_home
```

### CRS-4256 CRS-4602 While Replacing Voting Disk [ID 1475588.1]
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```
$ crsctl replace votedisk +VOTEDG

ASM diskgroup is not online or doesn't have enough free space or doesn't have sufficient failure groups (1 group for external redundance, 3 for normal, and 5 for high redundance). 

SQL> select group_number  "Group"
,disk_number   "Disk"
,header_status "Header"
,mode_status   "Mode"
,state         "State"
,redundancy    "Redundancy"
,total_mb      "Total MB"
,free_mb       "Free MB"
,name          "Disk Name"
,failgroup     "Failure Group"
,path          "Path"
from   v$asm_disk
order by group_number
,disk_number
/
```
### QUERY TO DETECT FILES IN AN ASM DISKGROUP BEFORE DROPPING
```
col full_path format a50
    col full_alias_path format a50
    SELECT concat('+'||gname, sys_connect_by_path(aname, '/')) full_alias_path
    FROM (SELECT g.name gname, a.parent_index pindex, a.name aname,
    a.reference_index rindex FROM v$asm_alias a, v$asm_diskgroup g
    WHERE a.group_number = g.group_number
    AND gname = 'MDDX1')
    START WITH (mod(pindex, power(2, 24))) = 0 CONNECT BY PRIOR rindex = pindex;
```
### Check ASM compatible attribute  (is not at least 11.2) , or the diskgroup mount status (is not mounted) on all nodes:
```
SQL> select inst_id, name, state, type, free_mb, substr(compatibility,1,10) compatibility from gv$asm_diskgroup;

To update diskgroup compatible, execute the following SQL statement:
SQL> ALTER DISKGROUP VOTEDG set ATTRIBUTE 'compatible.asm' = '11.2.0.0.0';

To mount on a specific node, connect to that node and issue the following or use asmca GUI interface to mount:
SQL> ALTER DISKGROUP VOTEDG mount;
```
### Doco ID: Doc ID:  752360.1 
```
Applies to: 
Oracle Server - Enterprise Edition - Version: 9.2.0.8 to 11.1.0.7
Information in this document applies to any platform.

Goal
This Document offers a step by step procedure to create ASM single instance physical standby from non ASM single instance primary database.

Solution
PROCEDURE:

==========
1.Enable force logging. 
2.Create SRL(standby redo logs). 
3.Backup the database that includes backup of datafiles, archivelogs and controlfile for standby 
4.Make proper changes in the parameter file of primary . 
5.Create the ASM instance at the standby server. 
6.Start the ASM instance and add directory 
7.Create the parameter file for standby, 
8.Establish the connectivity from primary to standby. 
9.Start the standby instance,. 
10.Use RMAN Duplicate command to create the standby database 
11.Start the MRP process, 
12. Verify whether the log are shipped and applied properly @the standby 


Consider two databases of names pri ,stdby 

1. Enable Forced Logging 
================= 


SQL> ALTER DATABASE FORCE LOGGING; 

Create a Password File(if not available). 
-------------------------------------------------- 
Create a password file if one does not already exist. Every database in a Data Guard configuration must use a password file, and the password for the SYS user must be identical on every system in order to transfer the redo data successfully.

2. Configure a Standby Redo Log 
======================= 



NOTE:SIZE OF STANDBY LOGFILE SHOULD BE SAME AS ONLINE LOGFILE 

a. Check the log files and sizes, 

SQL>SELECT GROUP# FROM V$LOGFILE; 

b. Create SRL, 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 3 '/arch1/stdby/redo1a.log' size 50m; 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 '/arch1/stdby/redo2b.log' size 50m; 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 '/arch1/stdby/redo3c.log' size 50m; 

c. Verify the standby redo log file groups were created(do this after the creation of stanby database; 

SQL> SELECT GROUP#,ThREAD#,SEQUENCE#,ARCHIVED,STATUS FROM V$STANDBY_LOG; 

3. Take the backup of entire database with archivelog including controlfile. 



RMAN> run 
{ 
allocate channel t1 type disk; 
allocate channel t2 type disk; 
allocate channel t3 type disk; 
allocate channel t4 type disk; 
backup database plus archivelog all; 
backup current controlfile for standby; 
} 

4. Make the necessary changes in primary init.ora file. 

For example: 

DB_NAME=pri 
DB_UNIQUE_NAME=pri 
LOG_ARCHIVE_CONFIG='DG_CONFIG=(pri,stdby)' 
CONTROL_FILES='/arch1/dbfiles/pri/control1.ctl', '/arch1/dbfiles/pri/control2.ctl' 
LOG_ARCHIVE_DEST_1= 
'LOCATION=/arch1/dbfiles/pri/ 
VALID_FOR=(ALL_LOGFILES,ALL_ROLES) 
DB_UNIQUE_NAME=pri' 
LOG_ARCHIVE_DEST_2= 
'SERVICE=stdby LGWR ASYNC 
VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) 
DB_UNIQUE_NAME=stdby' 
LOG_ARCHIVE_DEST_STATE_1=ENABLE 
LOG_ARCHIVE_DEST_STATE_2=DEFER 
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE 
LOG_ARCHIVE_FORMAT=%t_%s_%r.arc 
LOG_ARCHIVE_MAX_PROCESSES=30 
FAL_SERVER=stdby 
FAL_CLIENT=pri 
DB_FILE_NAME_CONVERT='+DATA_1/stdby/','/arch1/data/pri/' 
LOG_FILE_NAME_CONVERT='+FRA_1/stdby/','/arch1/dbfiles/pri/' 
STANDBY_FILE_MANAGEMENT=AUTO 
NOTE: After making necessary changes start the primary. 

5. Create the ASM instance at the standby server. 

Use dbca:to create ASM instance. 

1) proper global db name, eg., orcl.cos.us.oracle.com 
2) proper SID prefix, same as primary (orcl), limited to 5 characters 
3) same SYS password as the primary database 
4) ASM for storage, create the diskgroups, DATA_1, FRA_1 
5) Use Flash Recovery Area (+FRA_1), enable Archiving 
6) click Edit Archive Mode Parameters button adjacent to Enable Archiving 
checkbox and change format to match primary if necessary. The 
log_archive_format should include %t or %T, eg. %d_%t_%s_%r.arc 
7) Specify the same number and size of online redo logs as primary 

6. Start the ASM instance and add directory(if necessary) 

$ export ORACLE_SID=+ASM1 
$ sqlplus "/ as sysdba" 
SQL> alter diskgroup DATA_1 add directory '+DATA_1/stdby'; 
SQL> alter diskgroup FRA_1 add directory '+FRA_1/stdby'; 

7. Create the parameter file for standby, 

a. CREATE PFILE='<specify any location>' from spfile; (@primary,) 
b. Copy the file to standby server. 
c. Make the necessary changes, for example, 

DB_NAME=pri 
DB_UNIQUE_NAME=stdby 
LOG_ARCHIVE_CONFIG='DG_CONFIG=(pri,stdby)' 
CONTROL_FILES='/arch1/dbfiles/stdby/control1.ctl', '/arch1/dbfiles/stdby/control2.ctl' 
DB_FILE_NAME_CONVERT=’/arch1/data/pri/’,’+DATA_1/stdby/' 
LOG_FILE_NAME_CONVERT= '/arch1/redo/pri/','+FRA_1/stdby/' 
LOG_ARCHIVE_FORMAT=log%t_%s_%r.arc 
LOG_ARCHIVE_DEST_1= 
'LOCATION=+FRA_1/stdby/ 
VALID_FOR=(ALL_LOGFILES,ALL_ROLES) 
DB_UNIQUE_NAME=stdby' 
LOG_ARCHIVE_DEST_2= 
'SERVICE=pri LGWR ASYNC 
VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) 
DB_UNIQUE_NAME=pri' 
LOG_ARCHIVE_DEST_STATE_1=ENABLE 
LOG_ARCHIVE_DEST_STATE_2=ENABLE 
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE 
STANDBY_FILE_MANAGEMENT=AUTO 
FAL_SERVER=pri 
FAL_CLIENT=stdby 

8. Establish the connectivity from primary to standby. 

Configure listeners for the primary and standby databases. 
======================================================= 
L1 = 
(ADDRESS_LIST = 
(ADDRESS= (PROTOCOL= TCP)(Host= 192.9.200.108)(Port= 1521))) 
SID_LIST_LISTENER = 
(SID_LIST = 
(SID_DESC = 
(ORACLE_HOME= /home/oracle) 
(GLOBAL_DBNAME = stdby.server.com) 
(SID_NAME = stdby))) 


L1 = 
(ADDRESS_LIST = 
(ADDRESS= (PROTOCOL= TCP)(Host= 192.9.200.107)(Port= 1521))) 
SID_LIST_LISTENER = 
(SID_LIST = 
(SID_DESC = 
(ORACLE_HOME= /home/oracle) 
(GLOBAL_DBNAME = pri.server.com) 
(SID_NAME = pri))) 

To restart the listeners (to pick up the new definitions), enter the following LSNRCTL utility commands on both the primary and standby systems: 

$lsnrctl stop 
$lsnrctl start(starts the default listener) 
$lsnrctl start L1 

Create Oracle Net service names 
--------------------------------- 
The Oracle Net service name must resolve to a connect descriptor that uses the same protocol, host address, port, and service that you specified when you configured the listeners for the primary and standby databases. The connect descriptor must also specify that a dedicated server be used 

stdby = 
(DESCRIPTION = 
(ADDRESS = (PROTOCOL = TCP)(HOST = 192.9.200.108)(PORT = 1521)) 
(CONNECT_DATA = 
(SERVER = DEDICATED) 
(SERVICE_NAME = stdby.server.com) 
) 
) 


pri = 
(DESCRIPTION = 
(ADDRESS = (PROTOCOL = TCP)(HOST = 192.9.200.107)(PORT = 1521)) 
(CONNECT_DATA = 
(SERVER = DEDICATED) 
(SERVICE_NAME = pri.server.com) 
) 
) 

9. Start the stdby instance, 

NOTE: Make sure the ASM instance also running. 
$export ORACLE_SID=stdby 
SQL>create spfile from pfile=’<specify the newly created parameter location>’ 
SQL>startup nomount 

10. Use RMAN Duplicate command to create the standby database, 

NOTE: Before this step check the network connectivy from primary to standby and vice versa via TNSPING and SQL_PROMPT. 
NOTE: Connect to catalog if your primary database has catalog database. 


$RMAN target sys/<passwd>@primary catalog RMAN/RMAN@RMAN auxiliary sys/<passwd> 
RMAN>list backup; 
RMAN> RUN { 
allocate auxiliary channel C1 device type disk; 
duplicate target database for standby; 
} 

11. Start the MRP process, 

SQL>ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH; 

SQL>ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION; 

12: Verify whether the log are shipped and applied properly @the standby 

a. @pri execute on primary database 

SQL> ALTER SYSTEM SWITCH LOGFILE; 

SQL> SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME FROM V$ARCHIVED_LOG ORDER BY SEQUENCE#; 

b. @Primary (issue more log switch) 

SQL> ALTER SYSTEM SWITCH LOGFILE; 

Verify the new redo data was archived on the standby database.On the standby database, query the V$ARCHIVED_LOG view to verify the redo data was received and archived on the standby database: 

SQL> SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME 
2> FROM V$ARCHIVED_LOG ORDER BY SEQUENCE#; 

NOTE: Verify new archived redo log files were applied. At the standby database, query the V$ARCHIVED_LOG view to verify the archived redo log files were applied. 

SQL> SELECT SEQUENCE#,APPLIED FROM V$ARCHIVED_LOG ORDER BY SEQUENCE#; 
```
  



### ASM2NonASM 

```

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Database Migration from ASM storage to non-ASM storage
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
1. Check the current status and all the files (datafiles, controlfiles, logfiles) information.
SQL> select parallel from v$instance;

PAR
---
NO

SQL> select value from v$parameter where name='cluster_database';

VALUE
--------------------------------------------------------------------------------
FALSE

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
+DATA/TESTP/DATAFILE/system.278.831080263
+DATA/TESTP/DATAFILE/example.275.831080435
+DATA/TESTP/DATAFILE/sysaux.276.831080339
+DATA/TESTP/DATAFILE/undotbs1.273.831080481
+DATA/TESTP/DATAFILE/users.272.831080511

SQL> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
+DATA/TESTP/controlfile/testp.ctl
SQL> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
+FRA/TESTP/ONLINELOG/group_4.290.831082085
+FRA/TESTP/ONLINELOG/group_5.291.831082137
+FRA/TESTP/ONLINELOG/group_6.292.831082151
SQL> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
+DATA/TESTP/TEMPFILE/temp_01.dbf

2. TAKE A BACKUP of current ASM database (Make sure Controlfile backup is on)
-----------------------------------------

[oracle@rac1 ~]$ rman target /

Recovery Manager: Release 12.1.0.1.0 - Production on Sun Nov 10 08:47:14 2013

Copyright (c) 1982, 2013, Oracle and/or its affiliates.  All rights reserved.

connected to target database: TESTP (DBID=1042033562)

RMAN> alter system switch logfile;

using target database control file instead of recovery catalog
Statement processed

RMAN> alter system switch logfile;

Statement processed

RMAN> run {
configure channel device type disk format '/u01/app/oracle/BKP/%U';
backup database plus archivelog;
}
2> 3> 4> 
old RMAN configuration parameters:
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT   '/u01/app/oracle/BKP/%U';
new RMAN configuration parameters:
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT   '/u01/app/oracle/BKP/%U';
new RMAN configuration parameters are successfully stored


Starting backup at 10-NOV-13
current log archived
allocated channel: ORA_DISK_1
:
:
Finished backup at 10-NOV-13

Starting backup at 10-NOV-13
:
:
piece handle=/u01/app/oracle/BKP/0gooji8d_1_1 tag=TAG20131110T084756 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:03
Finished backup at 10-NOV-13

3. Change parameter and Create pfile for NON ASM database
==========================================================
RMAN> alter system set db_file_name_convert=' +DATA/TESTP/DATAFILE/',' /u01/app/oracle/oradata/TESTP' scope=spfile;

using target database control file instead of recovery catalog
Statement processed

RMAN> alter system set log_file_name_convert=' +FRA/TESTP/ONLINELOG/',' /u01/app/oracle/oradata/TESTP' scope=spfile;

Statement processed

RMAN> select value from v$parameter where name='spfile';

VALUE                                                                           
--------------------------------------------------------------------------------
+DATA/TESTP/spfile                                                                                                                                      
 
RMAN> create pfile from spfile;
Statement processed

4. Change the parameters for NON ASM and create the required directories for database
=====================================================================================

*.audit_file_dest='/u01/app/oracle/admin/TESTP/adump'
*.core_dump_dest='/u01/app/oracle/admin/TESTP/cdump'
*.audit_trail='db'
*.compatible='12.1.0.0.0'
*.control_files='/u01/app/oracle/oradata/TESTP/testp.ctl'#Restore Controlfile

5. Startup database at nomount 
============================================
RMAN> shutdown immediate;

database closed
database dismounted
Oracle instance shut down

RMAN> startup nomount pfile='/u01/app/oracle/product/12.1.0/dbhome_1/dbs/initTESTP.ora';

connected to target database (not started)
Oracle instance started

Total System Global Area     626327552 bytes

Fixed Size                     2291472 bytes
Variable Size                289409264 bytes
Database Buffers             331350016 bytes
Redo Buffers                   3276800 bytes

6. Mount database
=================
RMAN> restore controlfile from  '+DATA/TESTP/controlfile/testp.ctl';

Starting restore at 10-NOV-13
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=26 device type=DISK

channel ORA_DISK_1: copied control file copy
output file name=/u01/app/oracle/oradata/TESTP/control01.ctl
Finished restore at 10-NOV-13

RMAN> alter database mount;

Statement processed
released channel: ORA_DISK_1

7. Open database
=================

RMAN> backup  as  copy  database  format '/u01/app/oracle/oradata/TESTP/%U';

Starting backup at 10-NOV-13
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=34 device type=DISK
channel ORA_DISK_1: starting datafile copy
input datafile file number=00001 name=+DATA/TESTP/DATAFILE/system.278.831080263
output file name=/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSTEM_FNO-1_0qoojnah tag=TAG20131110T101753 RECID=12 STAMP=831118738
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:01:15
channel ORA_DISK_1: starting datafile copy
input datafile file number=00003 name=+DATA/TESTP/DATAFILE/sysaux.276.831080339
output file name=/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSAUX_FNO-3_0roojncs tag=TAG20131110T101753 RECID=13 STAMP=831118822
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:01:16
channel ORA_DISK_1: starting datafile copy
input datafile file number=00002 name=+DATA/TESTP/DATAFILE/example.275.831080435
output file name=/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-EXAMPLE_FNO-2_0soojnf8 tag=TAG20131110T101753 RECID=14 STAMP=831118858
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:35
channel ORA_DISK_1: starting datafile copy
input datafile file number=00004 name=+DATA/TESTP/DATAFILE/undotbs1.273.831080481
output file name=/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-UNDOTBS1_FNO-4_0toojngb tag=TAG20131110T101753 RECID=15 STAMP=831118876
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:25
channel ORA_DISK_1: starting datafile copy
input datafile file number=00006 name=+DATA/TESTP/DATAFILE/users.272.831080511
output file name=/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-USERS_FNO-6_0uoojnh4 tag=TAG20131110T101753 RECID=16 STAMP=831118885
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:01
Finished backup at 10-NOV-13

Starting Control File Autobackup at 10-NOV-13
piece handle=/u01/app/oracle/oradata/TESTP/FRA/TESTP/autobackup/2013_11_10/o1_mf_n_831117950_97xnd8l0_.bkp comment=NONE
Finished Control File Autobackup at 10-NOV-13

RMAN> alter database open;

Statement processed

RMAN> select name from v$datafile;
NAME                                                                            
--------------------------------------------------------------------------------

/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSTEM_FNO-1_0qoojnah
/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-EXAMPLE_FNO-2_0soojnf8
/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSAUX_FNO-3_0roojncs
/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-UNDOTBS1_FNO-4_0toojngb
/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-USERS_FNO-6_0uoojnh4
 
So datafiles have been moved to NON ASM database. 

Rename Datafiles
==================

NOTE: I am not using set newname as I am going to use 12c "move" command
Connect target /
	Run 
	{
	allocate channel tape1 device type sbt;
	set newname for datafile 1 to '/../system_01.dbf' ;
	set newname for datafile 2 to '............'
	.........
	
	set newname for datafile 4 to '...........'
:
:

	;

SQL> alter database move datafile '&pdf' to '&ndf';
Enter value for pdf: /u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSTEM_FNO-1_0qoojnah
Enter value for ndf: /u01/app/oracle/oradata/TESTP/system01.dbf                                      
old   1: alter database move datafile '&pdf' to '&ndf'
new   1: alter database move datafile '/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSTEM_FNO-1_0qoojnah' to '/u01/app/oracle/oradata/TESTP/system01.dbf'

Database altered.

SQL> /
Enter value for pdf: /u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-EXAMPLE_FNO-2_0soojnf8
Enter value for ndf: /u01/app/oracle/oradata/TESTP/example01.dbf                                     
old   1: alter database move datafile '&pdf' to '&ndf'
new   1: alter database move datafile '/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-EXAMPLE_FNO-2_0soojnf8' to '/u01/app/oracle/oradata/TESTP/example01.dbf'

Database altered.

SQL> /
Enter value for pdf: /u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSAUX_FNO-3_0roojncs
Enter value for ndf: /u01/app/oracle/oradata/TESTP/sysaux01.dbf                                      
old   1: alter database move datafile '&pdf' to '&ndf'
new   1: alter database move datafile '/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-SYSAUX_FNO-3_0roojncs' to '/u01/app/oracle/oradata/TESTP/sysaux01.dbf'

Database altered.

SQL> /
Enter value for pdf: /u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-UNDOTBS1_FNO-4_0toojngb
Enter value for ndf: /u01/app/oracle/oradata/TESTP/undotbs01.dbf                                     
old   1: alter database move datafile '&pdf' to '&ndf'
new   1: alter database move datafile '/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-UNDOTBS1_FNO-4_0toojngb' to '/u01/app/oracle/oradata/TESTP/undotbs01.dbf'

Database altered.

SQL> /
Enter value for pdf: /u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-USERS_FNO-6_0uoojnh4
Enter value for ndf: /u01/app/oracle/oradata/TESTP/user01.dbf                                       
old   1: alter database move datafile '&pdf' to '&ndf'
new   1: alter database move datafile '/u01/app/oracle/oradata/TESTP/data_D-TESTP_I-1042033562_TS-USERS_FNO-6_0uoojnh4' to '/u01/app/oracle/oradata/TESTP/user01.dbf'

Database altered.

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/TESTP/system01.dbf
/u01/app/oracle/oradata/TESTP/example01.dbf
/u01/app/oracle/oradata/TESTP/sysaux01.dbf
/u01/app/oracle/oradata/TESTP/undotbs01.dbf
/u01/app/oracle/oradata/TESTP/user01.dbf

Redo Log Migration
====================
SQL> alter database add logfile '/u01/app/oracle/oradata/TESTP/redo01.log' size 50M reuse;

Database altered.

SQL> alter database add logfile '/u01/app/oracle/oradata/TESTP/redo02.log' size 50M reuse;

Database altered.

SQL> alter database add logfile '/u01/app/oracle/oradata/TESTP/redo03.log' size 50M reuse;

Database altered.

SQL> alter database drop logfile group 4;
alter database drop logfile group 4
*
ERROR at line 1:
ORA-01623: log 4 is current log for instance TESTP (thread 1) - cannot drop
ORA-00312: online log 4 thread 1: '+FRA/TESTP/ONLINELOG/group_4.290.831082085'

SQL> c/4/5
  1* alter database drop logfile group 5
SQL> /

Database altered.

SQL> c/5/6
  1* alter database drop logfile group 6
SQL> /

Database altered.

SQL> alter system checkpoint;

System altered.

SQL> alter system switch logfile;

System altered.

SQL> /

System altered.

SQL> alter database drop logfile group 4;

Database altered.

SQL> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/TESTP/redo03.log
/u01/app/oracle/oradata/TESTP/redo02.log
/u01/app/oracle/oradata/TESTP/redo01.log

SQL> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/TESTP/control01.ctl


TEMPFILE MIGRATION
=====================
SQL> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
+DATA/TESTP/TEMPFILE/temp_01.dbf

SQL> alter tablespace temp add tempfile '/u01/app/oracle/oradata/TESTP/temp01.dbf' size 50M reuse;

Tablespace altered.

SQL> alter database tempfile '+DATA/TESTP/TEMPFILE/temp_01.dbf' drop;

Database altered.

SQL> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/TESTP/temp01.dbf


Check SPFILE (If ASM filesystem then move to NON ASM)
------------------------------------------------------
SQL> show parameter spfile;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string
SQL> create spfile from pfile;

File created.

SQL> shut immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup
ORACLE instance started.

Total System Global Area  626327552 bytes
Fixed Size                  2291472 bytes
Variable Size             289409264 bytes
Database Buffers          331350016 bytes
Redo Buffers                3276800 bytes
Database mounted.
Database opened.

SQL> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/product/12.1.0
                                                 /dbhome_1/dbs/spfileTESTP.ora

 ``` 



### NonASM2ASM 

```

Steps to migrate 11g R2 database from non-asm storage to asm storage 

1. Create the required directories for ASM database
============================================
ASMCMD> pwd
+DATA/TESTP
ASMCMD> mkdir DATAFILE CONROLFILE TEMPFILE PARAMETERFILE ONLINELOG 
ASMCMD> ls -l
Type  Redund  Striped  Time             Sys  Name
                                        N    CONROLFILE/
                                        N    DATAFILE/
                                        N    ONLINELOG/
                                        N    PARAMETERFILE/
                                        N    TEMPFILE/

2.  Configure FRA

RMAN> select value from v$parameter where name='spfile';

VALUE                                                                           
--------------------------------------------------------------------------------
/u01/app/oracle/product/12.1.0/dbhome_1/dbs/spfileTESTP.ora                                                                                                     
 
RMAN> create pfile from spfile;
Statement processed

RMAN> alter system set db_recovery_file_dest_size = 1G;
Statement processed
RMAN> alter system set db_recovery_file_dest = '+FRA';
Statement processed

RMAN> alter system set control_files='+DATA/control01.ctl' scope=spfile;
Statement processed

RMAN> alter system set db_recovery_file_dest = '+FRA';
Statement processed

RMAN> alter system set db_recovery_file_dest_size = 1G;
Statement processed

3. Moving the files (data and control) from NON ASM to ASM storage.
------------------------------------------------------------------------------------------------------------
RMAN> shutdown immediate;
RMAN> startup nomount;
RMAN> restore controlfile from '/u01/app/oracle/oradata/TESTP/control01.ctl';
RMAN> alter database mount;

RMAN> backup as copy database format '+DATA';
RMAN> switch database to copy;

Open the database
-----------------------
RMAN> alter database open;

Statement processed

RMAN> select name from v$datafile;
RMAN> select name from v$controlfile;

MOVE redologs
----------------
RMAN> select member,group# from v$logfile;
RMAN> alter database add logfile group 4 size 50M;
RMAN> alter database add logfile group 5 size 50M;
RMAN> alter database add logfile group 6 size 50M;
RMAN> alter database drop logfile group 1;
RMAN> alter database drop logfile group 2;
RMAN> alter database drop logfile group 3;
RMAN> select member from v$logfile;

Migrate temp tablespace
===================
RMAN> select tablespace_name, file_name from dba_temp_files;

TABLESPACE_NAME                 FILE_NAME                                                                       
--------------------------------------------------------------------------------
TEMP                                      /u01/app/oracle/oradata/TESTP/temp01.dbf

RMAN> alter tablespace TEMP add tempfile '+DATA/TESTP/temp_01.dbf' size 50m;

Statement processed

RMAN> alter database tempfile '/u01/app/oracle/oradata/TESTP/temp01.dbf' drop;
RMAN> select name from v$tempfile;
RMAN> select value from v$parameter where name='spfile';
VALUE                                                                           
--------------------------------------------------------------------------------
/u01/app/oracle/product/12.1.0/dbhome_1/dbs/spfileTESTP.ora         

RMAN> create pfile from spfile;

Statement processed

RMAN> create spfile='+DATA/TESTP/spfile' from pfile='/u01/app/oracle/product/12.1.0/dbhome_1/dbs/initTESTP.ora';
Statement processed

RMAN> run {
BACKUP AS BACKUPSET SPFILE;
RESTORE SPFILE TO  "+DATA/TESTP/spfile";
}
2> 3> 4> 
Starting backup at 10-NOV-13
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=51 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current SPFILE in backup set
channel ORA_DISK_1: starting piece 1 at 10-NOV-13
channel ORA_DISK_1: finished piece 1 at 10-NOV-13
piece handle=+FRA/TESTP/BACKUPSET/2013_11_10/nnsnf0_tag20131110t135115_0.309.831131477 tag=TAG20131110T135115 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 10-NOV-13

Starting restore at 10-NOV-13
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: restoring SPFILE
output file name=+DATA/TESTP/spfile
channel ORA_DISK_1: reading from backup piece +FRA/TESTP/BACKUPSET/2013_11_10/nnsnf0_tag20131110t135115_0.309.831131477
channel ORA_DISK_1: piece handle=+FRA/TESTP/BACKUPSET/2013_11_10/nnsnf0_tag20131110t135115_0.309.831131477 tag=TAG20131110T135115
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
Finished restore at 10-NOV-13

RMAN> startup

connected to target database (not started)
Oracle instance started
database mounted

database opened

Total System Global Area     551165952 bytes

Fixed Size                     2290608 bytes
Variable Size                297798736 bytes
Database Buffers             247463936 bytes
Redo Buffers                   3612672 bytes

RMAN> 
RMAN> select value from v$parameter where name='spfile';
VALUE                                                                           
--------------------------------------------------------------------------------
+DATA/TESTP/spfile                                                                    


MOVE DATAFILES TO ASM
------------------------
for moving the datafile to ASM use rman, 

allocate channel c1 type disk format "+diskgroup"; 
backup as copy datafile file#; 
switch datafile to copy; 


1.Check datafiles for the tablespace
------------------------------------
SQL> select FILE_ID, FILE_NAME from dba_data_files where TABLESPACE_NAME='ROSBO';

   FILE_ID
----------
FILE_NAME
--------------------------------------------------------------------------------
         6
+PDEV20_DATA_01/pdev20/datafile/rosbo.269.799948433

        13
/app/pdev20/product/10.2.0.5/db_1/dbs/PDEV20_DATA_01

2. connect to RMAN and put tablespace [respective datafiles] offline 

[oracle@csmper-cls18 ~]$ rman target /

Recovery Manager: Release 10.2.0.5.0 - Production on Sun Oct 5 09:44:05 2014

Copyright (c) 1982, 2007, Oracle.  All rights reserved.

connected to target database: PDEV20 (DBID=789467589)

RMAN> sql "alter tablespace ROSBO offline";

using target database control file instead of recovery catalog
sql statement: alter tablespace ROSBO offline


3. Copy datafile with RMAN copy command to +PDEV20_DATA_01 disk group
RMAN> copy datafile '/app/pdev20/product/10.2.0.5/db_1/dbs/PDEV20_DATA_01' to '+PDEV20_DATA_01';

Starting backup at 05-OCT-14
allocated channel: ORA_DISK_1
channel ORA_DISK_1: sid=471 devtype=DISK
channel ORA_DISK_1: starting datafile copy
input datafile fno=00013 name=/app/pdev20/product/10.2.0.5/db_1/dbs/PDEV20_DATA_01
output filename=+PDEV20_DATA_01/pdev20/datafile/rosbo.278.860147209 tag=TAG20141005T094648 recid=5 stamp=860147219
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:15
Finished backup at 05-OCT-14

Starting Control File and SPFILE Autobackup at 05-OCT-14
piece handle=+DATA_10G/c-789467589-20141005-01 comment=NONE
Finished Control File and SPFILE Autobackup at 05-OCT-14

4. Connect to SQL*Plus and rename old datafile to created ASM file

[oracle@csmper-cls18 ~]$ sqlplus " / as sysdba "

SQL*Plus: Release 10.2.0.5.0 - Production on Sun Oct 5 09:48:57 2014

Copyright (c) 1982, 2010, Oracle.  All Rights Reserved.


Connected to:
Oracle Database 10g Enterprise Edition Release 10.2.0.5.0 - 64bit Production
With the Partitioning, Real Application Clusters, OLAP, Data Mining
and Real Application Testing options

SQL> alter database rename file '/app/pdev20/product/10.2.0.5/db_1/dbs/PDEV20_DATA_01' to '+PDEV20_DATA_01/pdev20/datafile/rosbo.278.860147209';

Database altered.

5. Take tablespace [respective datafiles] online again 

SQL> alter tablespace ROSBO online;

Tablespace altered.

6. Check datafile location
SQL> select file_name from dba_data_files where tablespace_name='ROSBO';

FILE_NAME
--------------------------------------------------------------------------------
+PDEV20_DATA_01/pdev20/datafile/rosbo.269.799948433
+PDEV20_DATA_01/pdev20/datafile/rosbo.278.860147209

 ``` 



### RAC2Single 

```

Converting RAC to single instance on same machine

This is based on Oracle 10G Release 2 and assumes:
1. Oracle RAC running with cluster file system
2. You have basic knowledge about Oracle RAC

Test Server:
OS : Red Hat Enterprise Linux Server release 5.4
Database Version : 10.2.0.4
File system: OCFS2

1. Stop database and CRS on both node
$ srvctl stop database -d RACDB
# crsctl stop crs

2. Turn Off RAC

SQL> startup
ORA-29702 error occurred in Cluster Group Service operation

Relink with the RAC OFF.
$ cd $ORACLE_HOME/rdbms/lib
$ /usr/ccs/bin/make -f ins_rdbms.mk rac_off
Relinking oracle
$ make -f ins_rdbms.mk ioracle
## OR , both working fine
$ cd $ORACLE_HOME/bin
$ relink oracle

If ASM Instance Exist, run below command as root
# /u01/oracle/product/10.2.0/db/bin/localconfig delete
# /u01/oracle/product/10.2.0/db/bin/localconfig add

3.     Parameter(Pfile/spfile) & database changes
SQL> startup
SQL> alter database disable thread 2;
SQL> alter system set remote_listener='';

3a. Remove unwanted logfile
SQL> select thread#, group# from v$log;
SQL> alter database drop logfile group 3;
SQL> alter database drop logfile group 4;

3b. Remove unwanted tablespace
SQL> drop tablespace UNDOTBS2 including contents and datafiles;

3c.    Rename instance name.
SQL> alter system set instance_name=<new_name> scope=spfile;
SQL> shutdown immediate
SQL> startup
- Change your ORACLE_SID environment

4. Run $ORA_CRS_HOME/install/rootdelete.sh on both node
- This will stop and remove all CRS startup related file

5. Remove $ORA_CRS_HOME binary using Clusterware OUI installer
- Ignore any error if 2nd node already down
- rm -rf $ORA_CRS_HOME

6. Modify listener file
$ vi $ORACLE_HOME/network/admin/listener.ora

6a. Modify tnsname file
$ vi $ORACLE_HOME/network/admin/tnsnames.ora

That’s it. You have successfully converted your RAC database to Single Instance on same machine.
Note: You can convert your single instance DB to RAC again by following metalink note:747457.1

```
### CONVERT RAC DATABASE INTO STANDALONE DATABASE
```
There are several scenarios for this situation:

1. Converting RAC instances to non-RAC instances permanently without keeping the Oracle Clusterware.
2. Converting RAC instances to non-RAC instances permanently, but still keeping the Oracle Clusterware.
3. Converting RAC instances to non-RAC instances temporarily, so the production DB can continue running while troubleshooting of RAC issues is in progress.

In all cases, the Clusterware should be shutdown on other nodes to avoid conflicts of the operations.

1. Converting RAC instances to non-RAC instances permanently without keeping the Oracle Clusterware.

a.) Shutdown clusterware on ALL nodes with root user.
b.) Run rootdelete and rootdeinstall with root user.
c.) Run installer and remove the Clusterware home with crs user.
d.) Install a new single instance home with oracle user. Also a separate ASM home if preferred.
e.) As oracle user, remove the listener using netca from the OLD home. Create a new local listener using netca from the NEW home. The listener will not listen to VIP anymore. Change existing tnsnames.ora files on server and/or clients to use host IP instead of VIP.
f.) Configure ASM using dbca from the new home with oracle user. Follow the instruction from dbca to create non-RAC CSS using “localconfig add” with root user.
g.) With oracle user, copy the pfile/spfile from the old DB home to the new DB home, remove all the parameters for other instances in the pfile/spfile.

– Remove cluster_database and cluster_database_instances parameters.
– Remove undo_tablespace parameter for the other instances.
– Remove remote_listener and local_listener parameters if present.

h.) Startup new listener and ASM with oracle user. Make sure ASM diskgroups are mounted.
i.) With oracle user, startup the database in mount stage and execute
alter database disable thread <thread of other instance>;
alter database open;
j.) After opening database you can drop the redolog groups and/or tablespaces which are for other instances.
k.) With oracle user, modify the ORACLE_HOME on /etc/oratab. And remove instance_number and thread parameters in the pfile/spfile.
l.) With oracle user, run installer to remove the OLD ORACLE_HOME.

2. Converting RAC instances to non-RAC instances permanently, but still keeping the Oracle Clusterware.

In this scenario, it is also recommended to just install a Single Instance home and then start ASM and Database instances from the new home. (If preferred, a separate single-instance ASM home can be installed.) So, the inventory can be in sync of the changes, and this could prevent related problems in the future.

a.) Leave the Oracle Clusterware as it is.
b.) With oracle user, install a new SI home (runInstaller gives you option to install RAC enabled home or SI home). 
Also a separate ASM home if preferred.
c.) With root user, stop Clusterware on all node except the current node. 
Stop DB/ASM instances and remove instance registries in the OCR using the srvctl from the old home with crs user.
$ srvctl remove instance -d <name> -i <inst_name>
$ srvctl remove database -d <name>
$ srvctl remove asm -n <node_name> [-i <asm_inst_name>]

d.) With oracle user, remove listeners using netca from the old home.
e.) With oracle user, create a new listener using netca from the new home.
If the listener will not listen to VIP anymore, change existing tnsnames.ora files on server 
and/or clients to use host IP instead of VIP.
f.) With oracle user, configure single instance ASM using dbca from the new home.
g.) Convert and start DB instance from the new home. See step g to l in scenario #1.
h.) If preferred, register DB to the OCR.

3. Converting RAC instances to non-RAC instances temporarily, so the production DB can continue running while troubleshooting of RAC issues is in progress.

** Please note that while staying in this transition status, please DO NOT apply any RDBMS patch without first converting back to RAC.
a.) Shutdown all instances including ASM and DB instances in RAC environment on ALL nodes with oracle user.
b.) Shutdown all the listeners on ALL nodes with oracle user.
c.) With oracle user, relink Oracle executable with rac_off option. (For both ASM and DB homes)
$ make -f ins_rdbms.mk rac_off
$ make -f ins_rdbms.mk ioracle

d.) With oracle user, remove all parameters for other instances in the pfile/spfile.
– Remove cluster_database and cluster_database_instances parameters. (For both ASM and DB)
– Remove undo_tablespace parameter for the other instances. (For DB only)

e.) With oracle user, startup listener and ASM. Make sure ASM diskgroups are mounted.
f.) With oracle user, startup the database in mount stage and execute
alter database disable thread <thread of other instance>;
alter database open;

g.) After opening database you can drop the redolog groups and/or tablespaces which are for other instances.
h.) Disable autostart of ASM/DB/Clusterware. (Re-enable them after fixing the CRS/RAC issue)
$ srvctl modify database -d <DBNAME> -y manual
$ srvctl disable asm -n <node_name>
# crsctl disable crs

```  

### Single2RAC 

```

 STEPS TO CONVERT A SINGLE INSTANCE TO RAC  
(In a cluster environment)

Change the following parameters related to OMF & Flash_recovery_area to a shared location
   SQL> ALTER SYSTEM SET DB_CREATE_FILE_DEST='+DATA01';
   SQL> ALTER SYSTEM SET DB_RECOVERY_FILE_DEST='+FRA01';

Drop the log groups 1,2,3 and recreate log groups 1,2,3,4 with THREAD 1 & 2    ( give the switch logfile and checkpoint command when required )

   SQL> ALTER DATABASE DROP LOGFILE GROUP 1;
   SQL> ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 1;

   SQL> ALTER DATABASE DROP LOGFILE GROUP 2;
   SQL> ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 2;

   SQL> ALTER DATABASE DROP LOGFILE GROUP 3;
   SQL> ALTER DATABASE ADD LOGFILE THREAD 2 GROUP 3;
   SQL> ALTER DATABASE ADD LOGFILE THREAD 2 GROUP 4;

Add new Undo Tablespace for the new instance
    SQL> CREATE UNDO TABLESPACE UNDOTBS2 DATAFILE '+DATA01' SIZE 500M;

Enable the thread 2
    SQL> ALTER DATABASE ENABLE THREAD 2;

Take backup of controlfile to trace
    SQL> ALTER DATABASE BACKUP CONTROLFILE TO TRACE;

Change the control_files parameter to shared location
    SQL> ALTER SYSTEM SET CONTROL_FILES='+DATA01/CONTROL_01.CTL','+DATA01/CONTROL_02.CTL' SCOPE=SPFILE;

Shutdown the database and startup again in nomount to recreate the control files 
    SQL> SHUTDOWN IMMEDIATE;
    SQL> @<CONTROL_FILE TRACE NAME...>

Take an image copy backup to the shared location using RMAN
    RMAN> SHUTDOWN IMMEDIATE
    RMAN> STARTUP MOUNT;
    RMAN> BACKUP AS COPY DATABASE FORMAT '+DATA01';
    RMAN> SWITCH DATABASE TO COPY;

Create a new parameter file for the RAC instances on both the nodes. Add the following parameters
     CLUSTER_DATABASE=TRUE
     CLUSTER_DATABASE_INSTANCES=2
     TEST1.THREAD=1
     TEST2.THREAD=2
     TEST1.UNDOTABLESPACE=UNDOTBS1
     TEST2.UNDOTABLESPACE=UNDOTBS2
     TEST1.INSTANCE_NUMBER=1
     TEST2.INSTANCE_NUMBER=2

Shutdown down the single instance and startup the RAC instances one by one
   NODE-1
                export ORACLE_SID=TEST1
                SQL> STARTUP NOMOUNT;
                SQL> ALTER DATABASE MOUNT;
                SQL> ALTER DATABASE OPEN;

   NODE-2
                export ORACLE_SID=TEST2
                SQL> STARTUP NOMOUNT;
                SQL> ALTER DATABASE MOUNT;
                SQL> ALTER DATABASE OPEN;

Execute the catclust.sql script from $ORACLE_HOME/rdbms/admin directory
   SQL> @/u01/app/oracle/product/10.2.0/db_1/rdbms/admin/catclust.sql


Optionally register it with CRS for protection
     $ srvctl add database -d TEST -o /u01/app/oracle/product/10.2.0/db1 -y AUTOMATIC 
     $ srvctl add instance -d TEST -i TEST1 -n lmststdb1
     $ srvctl add instance -d TEST -i TEST2 -n lmststdb2

```
### --------------------------With RMAN BACKUP ---------------------------
```
Method to convert single instance to RAC
We have different ways to migrate non-RAC to RAC. Here we are using the DUPLICATE DATABASE feature of RMAN to migrate single instance non-ASM database to RAC Server, which is using ASM as shared storage. Using this manual method you have full control on the duplication process. In case of any issues/errors, you just need to fix the failed setup and you do not need to redo the whole process.
We are using a two phase approach to migrate non-RAC to RAC:
II.	Duplicate single instance Non-ASM database to ASM using RMAN
III.	Manually Convert single-instance to RAC.
Overview of Non-RAC environment
Hostname	Database Name	Instance Name	Database Storage
orasrv	DBORA	DBORA	ext3
Overview of RAC environment
Hostname	Database Name	Instance Name	Database Storage
orarac1	ORADB	ORADB1	ASM
orarac2	ORADB	ORADB2	ASM
If you prefer you can keep same name for RAC database. I am using a different database name in RAC environment to avoid confusion.
Please replace the xxxxxxx with actual password in below steps
1.1 Estimate used space for Non-RAC database:
Run the below query on Non-RAC to estimate used space and make sure you have enough space on RAC environment.
SYS@DBORA> SelectDF.TOTAL/1073741824 "DataFile Size GB", LOG.TOTAL/1073741824 "Redo Log Size GB", CONTROL.TOTAL/1073741824 "Control File Size GB", (DF.TOTAL + LOG.TOTAL + CONTROL.TOTAL)/ 1073741824 "Total Size GB" from dual, (select sum(a.bytes) TOTAL from dba_data_files a) DF, (select sum(b.bytes) TOTAL from v$log b) LOG, (select sum((cffsz+1)*cfbsz) TOTAL from x$kcccf c) CONTROL;
1.2 Create password file and init.ora file for RAC environment:
Make sure you set the ORACLE_SID and ORACLE_HOME and create password file under $ORACLE_HOME/dbs
[oracle@orarac1]$ export ORACLE_SID=ORADB
[oracle@orarac1]$ export ORACLE_HOME=/home/oracle/product/v10204
[oracle@orarac1]$ orapwd file= $ORACLE_HOME/dbs/orapwORADB password=xxxxxxx
Create initORADB.ora file under $ORACLE_HOME/dbs on RAC node1. Please note that we have two ASM diskgroups i.e. +DATA, +FLASH, If you are using different names replace the diskgroup names.
The below initORADB.ora file has minimum settings. Refer to your Non-RAC init.ora file and set the required parameters.
[oracle@orarac1]$  cat $ORACLE_HOME/dbs/initORADB.ora
##############################################################
# FILE : initORADB.ora
# DATABASE NAME : ORADB
##############################################################
# Set the RAC database name
db_name ="ORADB"
instance_name =ORADB

# set the location of the duplicate clone control file.
control_files =‘+DATA’, ‘+FLASH’

#set the below parameters for default location of data files
db_create_file_dest='+DATA'

#set the below parameters for default location of recovery area
db_recovery_file_dest='+FLASH'

# set below parameter to create two members for each redo
db_create_online_dest_1=’+DATA’
db_create_online_dest_2=’+FLASH’

# set two destinations if you want to multiplex the archive logs
log_archive_dest_1='location=+DATA'
log_archive_dest_2='location=+FLASH'

# set the location as per your environment
log_archive_dest_1='LOCATION=+FLASH’
log_archive_format='arch_%r_%s_%t.arc'
audit_file_dest =/home/oracle/admin/ORADB/adump
background_dump_dest=/home/oracle/admin/ORADB/bdump
core_dump_dest =/home/oracle/admin/ORADB/cdump
user_dump_dest =/home/oracle/admin/ORADB/udump

# In case of 11g set below parameter as per your environment
# diagnostic_dest= /home/apps/oracle

#Set the below to the same as the production target
db_block_size = 8192
sga_target=537919488
remote_login_passwordfile=exclusive
undo_management =AUTO
undo_retention =10800
undo_tablespace =UNDOTBS1
compatible = 10.2.0.4.0
We are not using DB_FILE_NAME_CONVERT, LOG_FILE_NAME_CONVERT parameters as most of the databases have data/redo files in multiple directories.
1.3 Configure Oracle Listener and tnsnames.ora file
Create a static listener for RAC node1 under $ORACLE_HOME/network/admin and reload because auxiliary database will not register itself with the listener:
[oracle@orarac1]$  cat $ORACLE_HOME/network/admin/listener.ora                     
SID_LIST_LISTENER =
(SID_LIST =
(SID_DESC =
(GLOBAL_DBNAME = ORADB)
(ORACLE_HOME = /home/oracle/product/v10204)
(SID_NAME = ORADB)
)
)
Add TNS entry in $ORACLE_HOME/network/admin/tnsnames.ora file on Non-RAC Server:
[oracle@orasrv]$ cat $ORACLE_HOME/network/admin/tnsnames.ora
ORADB =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = orarac1)(PORT = 1521))
(CONNECT_DATA =
(SERVER = DEDICATED)
(SERVICE_NAME = ORADB)
)
)
1.4 Start the database in nomount and test the connectivity 
Start the auxiliary database in NOMOUNT mode on RAC node1
[oracle@orarac1]$ export ORACLE_SID=ORADB
[oracle@orarac1]$ export ORACLE_HOME=/home/oracle/product/v10204
[oracle@orarac1>sqlplus /nolog

SQL*Plus: Release 10.2.0.4.0 - Production on Thu Jan 19 12:42:16 2012

Copyright (c) 1982, 2007, Oracle.  All Rights Reserved.

SQL> connect / as sysdba
Connected to an idle instance.
SQL> startup nomount
ORACLE instance started.

Total System Global Area  541065216 bytes
Fixed Size                  2085288 bytes
Variable Size             289410648 bytes
Database Buffers          239075328 bytes
Redo Buffers               10493952 bytes
SQL>exit
[oracle@orarac1]$
Test your Sql*Net connections from Non-RAC Server and you must be able to connect to the ORADB database on RAC Node1.
[oracle@orasrv]$ sqlplus sys/xxxxxxx@ORADB as sysdba

SQL*Plus: Release 10.2.0.4.0 - Production on Thu Jan 19 14:29:23 2012

Copyright (c) 1982, 2007, Oracle.  All Rights Reserved.

Connected to:
Oracle Database 10g Enterprise Edition Release 10.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL>
Here we are using RMAN catalog and testing the connection to RMAN Database too.
[oracle@orasrv]$ sqlplus  rman /xxxxxxx@rmancat

SQL*Plus: Release 10.2.0.4.0 - Production on Thu Jan 19 14:29:23 2012

Copyright (c) 1982, 2007, Oracle.  All Rights Reserved.

Connected to:
Oracle Database 10g Enterprise Edition Release 10.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL>

1.5 Take a backup of Non-RAC Database: 
Take a backup of your database and archive logs using below script. Here we are backing the database to a NFS file system/ backup/rman/ORADB.
[oracle@orasrv]$  rman TARGET / CATALOG rman/xxxxx@rmancat
RMAN> run{
run {
allocate channel d1 type disk;
allocate channel d2 type disk;
backup  database format '/backup/rman/ORADB/dbsdf_online_%d_t%t_s%s_p%p.rmn';
sql "alter system archive log current";
backup archivelog all delete input format '/backup/rman/ORADB/archdf_%d_t%t_s%s_p%p.rmn';
release channel d1;
release channel d2;
}
The RMAN multiplexing will help to decrease the backup time. If the database is large, allocate more channels and use set optimal value for filesperset and maxopenfiles. But please note that increasing filesperset, maxopenfiles values increases process memory requirement.
In RMAN, the FILESPERSET parameter determines how many datafiles to put in each backup set and MAXOPENFILES parameter of ALLOCATE CHANNEL defines how many datafiles RMAN can read from simultaneously.
Once the backup is completed you can either mount the backup file system or copy the backup files for duplicate process on RAC environment. If you are copying the files ensure that you create the same directory structure and copy the backup files:
[oracle@orarac1]$ mkdir /backup/rman/ORADB
[oracle@orarac1]$ scp oracle@orasrv:/backup/rman/ORADB/*.rmn /backup/rman/ORADB
1.6 Duplicate the database to RAC server 
In order to duplicate you must connect to the target database and auxiliary database started in NOMOUNT mode and also RMAN catalog, if you are using it.
As part of duplication process, RMAN restores the target data files to the duplicate database and performs the recovery using all available backups and archive logs. After recovery is completed RMAN restarts the duplicate database (auxiliary database) and opens with the RESETLOGS option and generates a new DBID for duplicate database. 
(i) Place the data files in one diskgroup : If you want to place all data files in one diskgroup then make sure you have set db_create_file_dest parameter in init.ora file
Here is the duplicate database script:
[oracle@orasrv]$ cat dup_ORADB.rmn
connect catalog rman/xxxxxxxx@rmancat
connect target /
connect auxiliary sys/xxxxxxx@ORADB
run{
allocate channel d1 device type disk;
allocate channel d2 device type disk;
allocate auxiliary channel a1 device type disk;
allocate auxiliary channel a2 device type disk;
duplicate target database to ORADB
pfile=/home/oracle/product/v10204/dbs/initORADB.ora
logfile
group 1 ('+DATA','+FLASH') SIZE 50M reuse,
group 2 ('+DATA','+FLASH') SIZE 50M reuse;
release channel d1;
release channel d2;
}
exit;
Run database duplication script as below:
[oracle@orasrv]$ $ORACLE_HOME/bin/rman cmdfile dup_ORADB.rmn | tee dup_ORADB.rmn
(ii) Place the data files in different diskgroups: If you have more than one diskgroup and want to place data files across different disk groups then prepare the duplication script using below command.
Here we are generating the SET NEWNAME command for each datafile in the database using below script:
SYS@DBORA> set head off
SYS@DBORA> set pagesize 100
SYS@DBORA> SQL>  select 'set newname for datafile '||file_id||' to '''||'+DATA'||''';' from dba_data_files;

set newname for datafile 1 to '+DATA';
set newname for datafile 2 to '+DATA';
set newname for datafile 3 to '+DATA';
set newname for datafile 4 to '+DATA';
set newname for datafile 5 to '+DATA';
set newname for datafile 6 to '+DATA';

6 rows selected.
Here we are generating the SET NEWNAME command for each tempfile in the database using below script:
SYS@DBORA >  select 'set newname for tempfile  '||file_id||' to '''||'+DATA'||''';' from dba_temp_files;
set newname for tempfile  1 to '+DATA';
Replace the diskgroup name that you want to place the data/temp files.
For ex: – we are placing the data files as below.
datafile1, datafile 2, datafile3 – +DATA1
datafile 4, datafile 5, datafile 6 – +DATA2
tempfile – +DATA2
Prepare the duplicate database script:
[oracle@orasrv]$ cat dup_ORADB.rmn
connect catalog rman/xxxxxxxx@rmancat
connect target /
connect auxiliary sys/xxxxxxx@ORADB
run{
allocate channel d1 device type disk;
allocate channel d2 device type disk;
allocate auxiliary channel a1 device type disk;
allocate auxiliary channel a2 device type disk;
set newname for datafile 1 to '+DATA1';
set newname for datafile 2 to '+DATA1';
set newname for datafile 3 to '+DATA1';
set newname for datafile 4 to '+DATA2';
set newname for datafile 5 to '+DATA2';
set newname for datafile 6 to '+DATA2';
duplicate target database to ORADB
pfile=/home/oracle/product/v10204/dbs/initORADB.ora
logfile
group 1 ('+DATA','+FLASH') SIZE 50M reuse,
group 2 ('+DATA','+FLASH') SIZE 50M reuse;
release channel d1;
release channel d2;
}
exit;
Run database duplication script as below:
[oracle@orasrv]$ $ORACLE_HOME/bin/rman cmdfile dup_ORADB.rmn | tee dup_ORADB.rmn

Copyright (c) 1982, 2007, Oracle.  All rights reserved.

RMAN> connect catalog *
2> connect target *
3> connect auxiliary *
4> run{
5> allocate channel d1 device type disk;
6> allocate channel d2 device type disk;
7> allocate auxiliary channel a1 device type disk;
8> allocate auxiliary channel a2 device type disk;
9> set newname for datafile 1 to '+DATA';
10> set newname for datafile 2 to '+DATA';
11> set newname for datafile 3 to '+DATA';
12> set newname for datafile 4 to '+DATA1';
13> set newname for datafile 5 to '+DATA1';
14> set newname for datafile 6 to '+DATA1';
15> set newname for tempfile 1 to '+DATA1';
16> duplicate target database to ORADB
17> pfile=/home/oracle/product/v10204/dbs/initORADB.ora
18> logfile
19> group 1 ('+DATA','+FLASH') size 50M reuse,
20> group 2 ('+DATA','+FLASH') size 50M reuse;
21> release channel d1;
22> release channel d2;
23> }
24> exit;
Connected to recovery catalog database

connected to target database: DBORA (DBID=10026343621)

connected to auxiliary database: ORADB (not mounted)

allocated channel: d1
channel d1: sid=201 devtype=DISK

---- (Some detail removed for brevity) ----
We have duplicated single instance Non-ASM database from ‘orasrv’ server into ASM storage on orarac1 server. In part 2 I explain to to convert the single instance database to RAC. 
  ```


### Upgrading from 10.2.0.4 to 10.2.0.5 (Clusterware, RAC, ASM) 
```
One of the pre-reqs is time zone value

SQL> SELECT version FROM v$timezone_file;
   VERSION
----------
         4
If this query reports version 4 (as in this case), no action is required. If this reports a version lower or higher than 4, see 1086400.1 for more information.
Current clusterware version

$ crsctl query crs activeversion
CRS active version on the cluster is [10.2.0.4.0]
$ crsctl query crs softwareversion
CRS software version on node [rac1] is [10.2.0.4.0]10gR2 clusterware could be upgraded in a rolling manner even though it's an in-place upgrade but as the Oracle home in-place upgrades does requires database to be shutdown. There will be warning message like

"OUI has detected that there are processes running in the currently selected Oracle Home. .........................."


1. Before the start of the clusterware upgrade all cluster applications are online.
crs_stat -t

2. Start the clusterware upgrade by executing runInstaller
Even though it is a rolling upgrade all nodes are selected by default.

When the installation has finished it will ask to run the root script on each node. This is where the rolling upgrade happens. 

3. Stop the clusterware stack on first node

stop on first node
crsctl stop crs
Stopping resources. This could take several minutes.
Successfully stopped CRS resources.
Stopping CSSD.
Shutting down CSS daemon.
Shutdown request successfully issued.Cluster stack is running on other node(s) and could be used by applications.

rac2 ~]$ crs_stat -t
4. Run the root102.sh on first node

# /opt/crs/oracle/product/10.2.0/crs/install/root102.sh

At the end of this clusterware stack will be up and running on this node (rac1) and will be available for use while root script is run on other nodes.

rac1 oracle]# crs_stat -t
5. Carry out the same steps on other node(s) (rac2)

$ crsctl stop crs
Run root102.sh

[root@rac2 oracle]# /opt/crs/oracle/product/10.2.0/crs/install/root102.sh

6. Verify clusterware is upgraded

$ crsctl query crs activeversion
CRS active version on the cluster is [10.2.0.5.0]
$ crsctl query crs softwareversion
CRS software version on node [rac2] is [10.2.0.5.0]

7. After the upgrade opatch exhbit an error which could be resolved by installing the latest version of opatch.

NOTE:
java.lang.NoClassDefFoundError: OsInfo 
After upgrading to 10.2.0.5 following error could be seen when using opatch.

/opt/app/oracle/product/10.2.0/ent/OPatch/opatch versionException in thread "main" java.lang.NoClassDefFoundError: OsInfoInvoking OPatch 10.2.0.4.9OPatch Version: 10.2.0.4.9OPatch succeeded.Believe this is due to missing OsInfo class in the opatch binaries bundled with 10.2.0.5 upgrade software. 
Download the latest opatch from metalink and unzip into affected oracle home and error should disappear.

8. Create a pfile of the database before the Oracle Home is upgraded. Shutdown all applications running out of the Oracle home before Oracle home ugprades

[oracle@rac1 ~]$ srvctl stop database -d rac10g2
[oracle@rac1 ~]$ srvctl stop asm -n rac1
[oracle@rac1 ~]$ srvctl stop asm -n rac2
[oracle@rac1 ~]$ srvctl stop nodeapps -n rac2
[oracle@rac1 ~]$ srvctl stop nodeapps -n rac1

9. Execute runInstaller and upgrade Oracle Home across the cluster

1. Database upgrade could be done manually or using DBUA. If database upgrade is done manually then set cluster_database=false before starting the database in upgrade mode. If DBUA is used setting cluster_database=false will be done by DBUA itself.

Since the 10.2.0.4 database and Oracle home had the PSU Jan 2012 applied and a new initialization parameter (_external_scn_rejection_threshold_hours) was introduced. 

As the newly upgrade 10.2.0.5 home doesn't have this patch (PSU Jan 2012) yet trying to start the database for upgrade could be a problem.
SQL> startup nomount
ORA-01078: failure in processing system parameters
LRM-00101: unknown parameter name '_external_scn_rejection_threshold_hours'This could be resolved with either by starting the database with the pfile created earlier (step 8) or applying the PSU Jan 2012 on the new 10.2.0.5 homes before running the DBUA.

In this case the latter option was selected. After the PSU Jan 2012 was applied on 10.2.0.5 DBUA ran without an issue.

12. After the upgrade has finished apply the post installation script of the PSU. This concludes the 10.2.0.4 to 10.2.0.5 upgrade.

Useful metalink notes
Oracle Clusterware (CRS or GI) Rolling Upgrades [ID 338706.1] 
http://asanga-pradeep.blogspot.co.uk/2012/02/upgrading-from-10204-to-10205.html

```  



#### to11.2.0.1 

```

1. Leave the existing RAC running. As of 11gR2 all upgrades are rolling upgrades

2. Unset ORACLE_BASE,ORACLE_HOME,ORACLE_SID,ORA_CRS_HOME,ORA_NLS10 and TNS_ADMIN. Remove from PATH and LD_LIBRARY_PATH variables any reference to existing system.

3. Prepare new locations for grid infrastructure home, scan IP, operating system user groups

4. runInstaller from the grid infrastructure software and select the upgrade option

5.Although Oracle says "Oracle recommends that you leave Oracle RAC instances running. When you start the root script on each node, that node's instances are shut down and then started up again by the rootupgrade.sh script" the installer complains but this could be ignored and proced to next step.

6. This test system only had a single node but if there were multiple nodes then Oracle "recommends that you select all cluster member nodes for the upgrade, and then shut down database instances on each node before you run the upgrade root script, starting the database instance up again on each node after the upgrade is complete"
Also Oracle "Oracle recommends that you upgrade Oracle ASM at the same time that you upgrade the Oracle Clusterware binaries. Until ASM is upgraded, Oracle databases that use ASM can't be created. Until ASM is upgraded, the 11g release 2 (11.2) ASM management tools in the Grid home (for example, srvctl) will not work."

7. Give the new SCAN IP

8. Oracle uses a less powerful asmsnmp user to monitor the asm upgrade. Give a password to be associated with this user.

9. New locations for the installation, 11gR2 grid upgrade is an out of place upgrade.

10. Summary

11. When prompted run the rootupgrade.sh script as root user

12. Click on the OUI window which will then run the ASMCA to upgrade the ASM followed by the DB Control upgrade. Once these upgrades are done, upgrade of clusterware is completed. 

13. crs_stat is deprecated in 11gR2 but still works. Instead should use crsctl
crs_stat -t

crsctl check crs
14. Vote disk is still in the block device, though not supported for new installation, it's still a valid location for upgrades
./crsctl query css votedisk
15. OCR is also in a block device same as vote disk. Not supported as a location for new installation but valid for upgrades.
ocrcheck
16. /etc/oratab is auto upgraded with the new location of the ASM home, with 11gR2 it is same as grid infrastructure home
+ASM1:/opt/app/11.2.0/grid:N17. /etc/inittab now has the oracle high availability service daemon entry not the three clusterware entires as before 
tail /etc/inittab
h1:35:respawn:/etc/init.d/init.ohasd run >/dev/null 2>&1 </dev/null

18.Finally to confrim the active,release and software versions

crsctl query crs activeversion

crsctl query crs releaseversion

crsctl query crs softwareversion
```
  
### TO 11.2.0.3 

```
Relevant section for 11.1 upgrade to 11.2 from GI Install guide. 
After you have completed the Oracle Clusterware 11g release 2 (11.2) upgrade, if you did not choose to upgrade Oracle ASM when you upgraded Oracle Clusterware, then you can do it separately using the Oracle Automatic Storage Management Configuration Assistant (asmca) to perform rolling upgrades. You can use asmca to complete the upgrade separately, but you should do it soon after you upgrade Oracle Clusterware, as Oracle ASM management tools such as srvctl will not work until Oracle ASM is upgraded.

ASMCA performs a rolling upgrade only if the earlier version of Oracle ASM is either 11.1.0.6 or 11.1.0.7. Otherwise, ASMCA performs a normal upgrade, in which ASMCA brings down all Oracle ASM instances on all nodes of the cluster, and then brings them all up in the new Grid home.

You can use Oracle Database release 9.2, release 10.x and release 11.1 with Oracle Clusterware 11g release 2 (11.2). However, placing Oracle Database homes on Oracle ACFS that are prior to Oracle Database release 11.2 is not supported, because earlier releases are not designed to use Oracle ACFS.

If you upgrade an existing version of Oracle Clusterware and Oracle ASM to Oracle Grid Infrastructure 11g release 11.2 (which includes Oracle Clusterware and Oracle ASM), and you also plan to upgrade your Oracle RAC database to 11.2, then the required configuration of existing databases is completed automatically when you complete the Oracle RAC upgrade, and this section does not concern you.

However, if you upgrade to Oracle Grid Infrastructure 11g release 11.2, and you have existing Oracle RAC installations you do not plan to upgrade, or if you install older versions of Oracle RAC (9.2, 10.2 or 11.1) on a release 11.2 Oracle Grid Infrastructure cluster, then you must complete additional configuration tasks or apply patches, or both, before the older databases will work correctly with Oracle Grid Infrastructure.

Before you start an Oracle RAC or Oracle Database installation on an Oracle Clusterware 11g release 11.2 installation, if you are upgrading from releases 11.1.0.7, 11.1.0.6, and 10.2.0.4, then Oracle recommends that you check for the latest recommended patches for the release you are upgrading from, and install those patches as needed on your existing database installations before upgrading.

During an upgrade, all cluster member nodes are pinned automatically, and no manual pinning is required for existing databases. This procedure is required only if you install older database versions after installing Oracle Grid Infrastructure release 11.2 software.

To upgrade existing 11.1 Oracle Clusterware installations to Oracle Grid Infrastructure 11.2.0.3 or later, you must patch the release 11.1 Oracle Clusterware home with the patch for bug 7308467. (Included in PSU 11.1.0.7.6 for CRS) This cluster has 11.1.0.7.7 CRS PSU installed and Jan 2012 PSU installed on RAC. 

With Oracle Clusterware 11g release 1 and later releases, the same user that owned the Oracle Clusterware 10g software must perform the Oracle Clusterware 11g upgrade. Before Oracle Database 11g, either all Oracle software installations were owned by the Oracle user, typically oracle, or Oracle Database software was owned by oracle, and Oracle Clusterware software was owned by a separate user, typically crs.

During a major version upgrade to 11g release 2 (11.2), the software in the 11g release 2 (11.2) Oracle Grid Infrastructure home is not fully functional until the upgrade is completed. Running srvctl, crsctl, and other commands from the 11g release 2 (11.2) home is not supported until the final rootupgrade.sh script is run and the upgrade is complete across all nodes. To manage databases in the existing earlier version (release 10.x or 11.1) database homes during the Oracle Grid Infrastructure upgrade, use the srvctl from the existing database homes.

From the Upgrade guide
A subset of nodes cannot be selected when upgrading from an earlier release to 11.2.0.3. Before the new database release 11.2.0.3 software can be installed on the system, the root script for upgrading Oracle Grid Infrastructure invokes ASMCA to upgrade Oracle ASM to release 11.2.0.3.


The cluster verification tool allows a pre-upgrade test. Some noteworthey checks that failed are 

Check: Kernel parameter for "shmmni"
  Node Name         Current       Configured    Required      Status        Comment
  ----------------  ------------  ------------  ------------  ------------  ------------
  rac2              4096          unknown       4096          failed        Configured value too low.
  rac1              4096          unknown       4096          failed        Configured value too low.
Result: Kernel parameter check failed for "shmmni"Even though this is an existing 11gR1 system and this value is set it is considered as unknown. Running the fix script generated by clufy fixed this.

11gR2 has a different port range than 11gR1

Check: Kernel parameter for "ip_local_port_range"Also install cvuqdisk-1.0.9-1.rpm for ocr block device sharedness check otherwise this check will fail.

If ntp service is not used for time synchronization remove it, to pass the ntp service check

NTP Configuration file check started...
The NTP configuration file "/etc/ntp.conf" is available on all nodes
NTP Configuration file check passed
No NTP Daemons or Services were found to be running
PRVF-5507 : NTP daemon or service is not running on any node but NTP configuration file exists on the following node(s):
rac2,rac1
Result: Clock synchronization check using Network Time Protocol(NTP) failedTo fix (if ntp is not used)

mv /etc/ntp.conf /etc/ntp.conf.orig

/sbin/chkconfig ntp off
/sbin/chkconfig --list | grep ntp
ntpd            0:off   1:off   2:off   3:off   4:off   5:off   6:offcluvfy also check for patch 11724953. This is the 2011 April CRS PSU so if this is applied no additional patches are needed.

The full output from the cluvfy check is given below.

./runcluvfy.sh stage -pre crsinst -upgrade -n rac1,rac2 -rolling -src_crshome /opt/crs/oracle/product/11.1.0/crs -dest_crshome /opt/app/11.2.0/grid -dest_version 11.2.0.3.0 -fixup -fixupdir /home/oracle/fixupscript -verbose

11gR2 GI uses a SCAN IP which is a pre-req for the upgrade.

Even though upgrade must be done with the same user new user groups could be created for ASM administration. Using the same DBA and OPER group used for oracle user would give a warning. 

Create the new asm admin groups.

groupadd asmadmin
groupadd asmdba
groupadd asmoperModify the Oracle user from

id oracle
uid=501(oracle) gid=501(oinstall) groups=501(oinstall),502(dba),503(oper)To

usermod -g oinstall -G dba,oper,asmdba,asmoper,asmadmin oracle
id oracle
uid=501(oracle) gid=501(oinstall) groups=501(oinstall),502(dba),503(oper),504(asmadmin),505(asmdba),506(asmoper)Start the clusterware upgrade while the cluster is up

crs_stat -t
Execute runInstaller and follow the wizard
Select both (or all) nodes
Specify the scan ip
Password for less privilege user (asmsnmp) to monitor ASM
Use the newly created ASM admin OS groups.
New GI Home location for the out-of-place ugprade
Pre-req check.
Summary

Until the rootupgrade.sh is run cluster stack is up on all nodes. When rootupgrade.sh is run on one node cluster stack on that node is brought down while other node's clusterware stack remains up and open for use. This way by default the upgrades are rolling upgrades. Once rootupgrade.sh is finished running, cluster stack on the node it was run will be brought up and ready for use while the other node's cluster stack is brought down with the rootupgrade.sh

Run rootupgrade.sh on rac1

# /opt/app/11.2.0/grid/rootupgrade.sh

crs_stat -t
Note: gsd is not up on 11gR2 this is normal.

Until all the nodes are upgraded the active version remains the lower version in this case 11.1.0.7 but the software version will be the new version on the upgraded node.

crsctl query crs activeversion
Oracle Clusterware active version on the cluster is [11.1.0.7.0]

crsctl query crs softwareversion
Oracle Clusterware version on node [rac1] is [11.2.0.3.0]Upgrade rac2

# /opt/app/11.2.0/grid/rootupgrade.sh

After all the nodes are upgraded active version will updated

crsctl query crs activeversion
Oracle Clusterware active version on the cluster is [11.2.0.3.0]

crsctl query crs softwareversion
Oracle Clusterware version on node [rac2] is [11.2.0.3.0]

After rootupgrade.sh has finished on the last node click the OK button on the configuration script dialog. This would start configurations steps which also include the ASM upgrade. 
ASM is upgraded in a rolling fashion, each database instance that is using the ASM instance being upgraded will be brought down automatically before the ASM upgrade and will be started once ASM is upgraded.


+++++++++++++++++++++++++++++++++++++++++
upgrade the RAC
+++++++++++++++++++++++++++++++++++++++++
It is possible to upgrade the software and database at the same time. 
Actions For DST Updates When Upgrading To Or Applying The 11.2.0.3 Patchset [ID 1358166.1] gives details on timezone upgrade information for 11.2.0.3. In this upgrade path (11.1.0.7 -> 11.2.0.3) there's no need to apply any patches beforehand but after upgrade it's advised to ugprade the timezone. This could be done during DBUA exeuction by selecting the upgrade timezone option. Current timezone is 

SQL> SELECT version FROM v$timezone_file;

   VERSION
----------
         4
If there are lot of em console related files in $ORACLE_HOME/hostname_instance/sysman/emd/upload/ file copying could take long time.

Execute runInstaller
Select all nodes
New location for out of place upgrade
RAC admin with same OS groups
Summary
Once the root.sh is run and OK button is clicked DBUA runs and database upgrade starts. For the cluster database all instances are shutdown and upgrade continues only on one node with one instance. At the end of the upgrade all instances are started.
Upgrade timezone with database upgrade.
Upgrade summary
Upgrade result
Once the database is upgraded RAC upgrade concludes

Once the upgrade is finished uninstall 11gR1 clusterware home and database software.

Post upgrade notes.

remote listener has both tnsnames.ora entries from 11gR1 and scan ip on all nodes

remote_listener                      string      LISTENERS_RAC11G1, rac-scan:1521

  ```



### TO      11.2.0.3 

Upgrade from 10.2.0.5 to 11.2.0.3 (Clusterware, RAC, ASM) 
========================================================
http://asanga-pradeep.blogspot.co.uk/2012/03/upgrade-from-10205-to-11203-clusterware.html

From 11gR2 GI Install Guide
---------------------------------------------------------
Before you start an Oracle RAC or Oracle Database installation on an Oracle Clusterware 11g release 11.2 installation, if you are upgrading from releases 11.1.0.7, 11.1.0.6, and 10.2.0.4, then Oracle recommends that you check for the latest recommended patches for the release you are upgrading from, and install those patches as needed on your existing database installations before upgrading.

For more information on recommended patches, refer to "Oracle Upgrade Companion," which is available through Note 785351.1.

The Cluster runs on RHEL 4

uname -rmi
2.6.9-89.ELsmp x86_64 x86_64With vote and ocr files in raw devices

crsctl query css votedisk
 0.     0    /dev/raw/raw2

$ ocrcheck

check cluster using  cluvfy 
------------------------------------------------------------------
./runcluvfy.sh stage -pre crsinst -upgrade -n rac1,rac2 -rolling -src_crshome /opt/crs/oracle/product/10.2.0/crs -dest_crshome /opt/app/11.2.0/grid -dest_version 11.2.0.3.0 -fixup -fixupdir /home/oracle/fixupscript -verbose

You can get some unsuccessful message
:
:
Checking Oracle Cluster Voting Disk configuration...
Oracle Cluster Voting Disk configuration check passed
Clusterware version consistency passed
Pre-check for cluster services setup was unsuccessful on all the nodes.Pre-check is unsuccessful because of the above mentioned check. 

Configure a SCAN IP to be used with 11gR2 GI upgrade.
------------------------------------------------------------------------------------------------

Create the new asm admin groups.
---------------------------------------------------------
groupadd asmadmin
groupadd asmdba
groupadd asmoperModify the Oracle user from

id oracle
uid=501(oracle) gid=501(oinstall) groups=501(oinstall),502(dba),503(oper)To

usermod -g oinstall -G dba,oper,asmdba,asmoper,asmadmin oracle
id oracle
uid=501(oracle) gid=501(oinstall) groups=501(oinstall),502(dba),503(oper),504(asmadmin),505(asmdba),506(asmoper)

Start the clusterware upgrade while the cluster stack is up
---------------------------------------------------------------------------------------------
crs_stat -t

Execute runInstaller
                 select Upgrade Oracle GI and ASM
	1. Select all nodes and ASM upgrade.
	2. It is not possible to do a rolling ASM upgrade with 10gR2. When it is time to do a ASM upgrade all database instance will be shutdown. But the GI install and 	configuration will happen in a rolling manner.
                 [Message --      INS-41710]
 	3.Specify the new scan ip
	4.Password for less privilege ASMSNMP user. Used to monitor ASM.
	5.Specify new ASM admin groups
	6.New location for GI used in the out-of-place upgrade. (Ex: /u01/app/11.2.0/grid)
	7.Manually verify the failed check by checking the ignoreall check box.
	8.Upgrade Summary
	9.Execute rootupgrade.sh script one node at a time. Database and ASM will be brought down and back up again once the script has finished executing.
	Running rootupgrade.sh on first node (rac1)

	# /opt/app/11.2.0/grid/rootupgrade.sh   [overwrite is no]
 CHECK
---------------------------------------
Software version is upgrade but active version remain 10.2.0.5 until all nodes are upgraded

crsctl query crs activeversion
CRS active version on the cluster is [10.2.0.5.0]

crsctl query crs softwareversion
CRS software version on node [rac1] is [11.2.0.3.0]
    	10. Running rootupgrade.sh on second node (rac2)

	# /opt/app/11.2.0/grid/rootupgrade.sh        [overwrite is no]

Active version is upgraded to 11.2.0.3

crsctl query crs activeversion
CRS active version on the cluster is [11.2.0.3.0]

When the OK button is clicked on Execute Configuration Script dialog other configuration tasks starts. During ASM upgrade all database instances will be brought down automatically and once ASM is upgraded will be brought up.

This conclude the clusterware upgrade to grid infrastructure and GI will be using the raw devices for ocr and vote disk.
crsctl query css votedisk
ocrcheck

++++++++++++++++++++++++++++++++++++++++++++++++++++++
upgrade the RAC software and upgrade the database
++++++++++++++++++++++++++++++++++++++++++++++++++++++

 As per 1358166.1 there's no pre-patch required if the timezone is 4 on 10gR2 but it's advised to upgrade the timezone to 14 after the upgrade. This could be done while upgrading the database. Current timezone is 4

SQL>  SELECT version FROM v$timezone_file;

   VERSION
----------
         4

Execute runInstaller from database software location.
	1. Upgrade an existing database
	2.Oracle Real Application Cluster Database Installation and selecy two nodes
	3. Specify the 11gR2 home (ex: /u01/app/oracle/product/11.2.0/db_1)
	4.select OS group
	5.check summary
	6.Execute root.sh on all nodes
	7. End of root.sh and clicking OK button will result in start of DBUA. All but one instance will be brought down, upgrade will continue on the instance that is up and once completed all instance will be brought up.
		
		select 10gR2 rac databadse	
 		upgrade option [Upgrade timezone during the database upgrade]
                                select * recompile invalid objects; 
		* Turn off Archiving for the duration of upgrade 
		* upgrade Timezone version and TIMESTAMP with TIME ZONE data
		Pre-upgrade summary
		Upgrade result
		Conclude the RAC software and database upgrade

remote listener entry will have both tnsnames.ora entries from 10gR2 and scan ip

remote_listener                      string      LISTENERS_RAC10G2, rac-scan:1521

  



### Move ASMdatafiles 
#### Rename Datafile
```
SELECT SUM(a.bytes)/1024/1024 as UNDO_SIZE
FROM v$datafile a,
       v$tablespace b,
       dba_tablespaces c
 WHERE c.contents = 'UNDO'
   AND c.status = 'ONLINE'
     AND b.name = c.tablespace_name
  AND a.ts# = b.ts#;

+PTEST4_DATA_01/ptest4/system01.dbf
+PTEST4_DATA_01/ptest4/sysaux01.dbf
+PTEST4_DATA_01/ptest4/undotbs01.dbf
+PTEST4_DATA_01/ptest4/users01.dbf

RMAN> convert datafile '+PTEST4_DATA_01/ptest4/users01.dbf' format '+PTEST4_DATA_01';
SYS@ptest41 SQL> alter database datafile '+PTEST4_DATA_01/ptest4/users01.dbf' offline;
SYS@ptest41 SQL> alter database rename file '+PTEST4_DATA_01/ptest4/users01.dbf' to '+PTEST4_DATA_01/ptest4/datafile/users.290.876737755';
SYS@ptest41 SQL> recover datafile 4;
SYS@ptest41 SQL> alter database datafile 4 online;
```

### Move datafiles
```
1. SYS@ptest241 SQL> SELECT file_name FROM dba_data_files;

2. Identify the Oracle ASM diskgroup to which the database file will be moved to
SYS@ptest241 SQL> SELECT name FROM v$asm_diskgroup;

3. Take the Oracle ASM data file to be moved OFFLINE. 
SQL> ALTER DATABASE DATAFILE '+CLU02_DATA_01/ptest24/datafile/dw_mtd_data.318.876396799' OFFLINE;

4. Make a copy the Oracle ASM database file to be moved. There are two methods that can be used to perform the copy operation; however, I will only cover the RMAN method. 

RMAN - (preferred) 
Using the COPY_FILE procedure of the DBMS_FILE_TRANSFER PL/SQL Package 

$ rman target /

RMAN> COPY DATAFILE '+CLU02_DATA_01/ptest24/datafile/dw_mtd_data.318.876396799' TO '+LATEST_DATA_01';

After successfully executing the RMAN statement above, you will now have two copies of the Oracle ASM database file. 
SQL> @asm_files

5. Now that the file has been copied, update the Oracle data dictionary with the location of the new Oracle ASM database file to use. 

SQL> ALTER DATABASE RENAME FILE   '+CLU02_DATA_01/ptest24/datafile/dw_mtd_data.318.876396799'
    TO '+LATEST_DATA_01/ptest24/datafile/***';

6. Use RMAN to rename the ASM database file copy. 
RMAN> SWITCH DATAFILE '+LATEST_DATA_01/ptest24/datafile/***' TO COPY;

7. Recovery the new ASM database file. 
SQL> RECOVER DATAFILE '+LATEST_DATA_01/ptest24/datafile/***';

8. Bring the new ASM database file ONLINE. 
SQL> ALTER DATABASE DATAFILE '+LATEST_DATA_01/ptest24/datafile/***' ONLINE;

9. Verify the new ASM data file location. 

SQL> SELECT file_name FROM dba_data_files;

10. Check and if required then
 
Delete the old ASM database file from its original location. 
$ ORACLE_SID=+ASM; export ORACLE_SID

$ sqlplus "/ as sysdba"

SQL> ALTER DISKGROUP +CLU02_DATA_01 DROP FILE '+CLU02_DATA_01/ptest24/datafile/dw_mtd_data.318.876396799';
 ``` 

### Move DB to different diskgroups 

```

How To Move The Database To Different Diskgroup (Change Diskgroup Redundancy) [ID 438580.1] 
--------------------------------------------------------------------------------
Modified 18-APR-2012     Type HOWTO     Status PUBLISHED   
In this Document
 Goal 
 Fix 
--------------------------------------------------------------------------------
Applies to: 
Oracle Server - Enterprise Edition - Version 10.1.0.2 to 11.1.0.7 [Release 10.1 to 11.1]
Oracle Server - Standard Edition - Version 10.1.0.2 to 11.1.0.7 [Release 10.1 to 11.1]
Information in this document applies to any platform.

Goal
Automatic Storage Management (ASM) is an integrated file system and volume manager expressly built for Oracle database files.

This note is applicable :

1. If you wish to move to different ASM storage / Hardware.
2. If you wish to change the redundancy of the diskgroup

How to collect the full path name of the files in ASM diskgroups
Set your ORACLE_SID to the ASM instance name.

Connect to the ASM instance:
in 10g: sqlplus / as sysdba
in 11g: sqlplus / as sysasm

Then perform the following query:  
SELECT gnum, filnum, concat('+'||gname,sys_connect_by_path(aname, '/'))
FROM (SELECT g.name gname, a.parent_index pindex, a.name aname, 
a.reference_index rindex, a.group_number gnum,a.file_number filnum
FROM v$asm_alias a,v$asm_diskgroup g
WHERE a.group_number = g.group_number)
START WITH (mod(pindex, power(2, 24))) = 0
CONNECT BY PRIOR rindex = pindex;



There are two ways to perform this:

1. Create a new Diskgroup with desired redundancy and move the existing data to newly created Diskgroup.

2. Drop the existing Diskgroup after backing up data and create a new Diskgroup with desired redundancy.
===============================================================================================
CASE 1: Create a new diskgroup with desired redundancy and move the existing data to newly created diskgroup.
===============================================================================================
1) If we have extra disk space available,then we can create a new diskgroup and move the files from old diskgroup to it.

-- Initially we have two diskgroup with external redundancy as:

SQL> select state,name from v$asm_diskgroup;

STATE NAME
----------- --------------------
MOUNTED DG2
MOUNTED DG3

2) Create a new diskgroup with normal redundancy as :

SQL > create diskgroup DG1 normal redundancy failgroup <failgroup1_name> disk 'disk1_name' failgroup <failgroup2_name> disk 'disk2_name';

SQL> select state,name,type from v$asm_diskgroup;

STATE NAME TYPE
----------- ------------------- ------
MOUNTED DG2 EXTERN
MOUNTED DG3 EXTERN
MOUNTED DG1 NORMAL

3)Backup the current database as follows:
SQL> show parameter db_name
NAME TYPE VALUE
---------------- ----------- ----------------------------
db_name string orcl10g

SQL> create pfile='d:\initsid.ora' from spfile;
SQL> alter database backup controlfile to '+DG1';
SQL> alter system set control_files='+DG1\ORCL10G\CONTROLFILE\<system generated control file name from diskgroup DG1>' SCOPE=SPFILE;

-- Connect to rman
$ rman target /
RMAN > shutdown immediate;
RMAN > startup nomount;
RMAN> restore controlfile from '+DG2\ORCL10G\CONTROLFILE\mycontrol.ctl' (specify the original (old) location of controlfile here) to '<new_diskgroup i.e +DG1>'

Mount the database and validate the controlfiles from v$controlfile

RMAN > alter database mount;

RMAN> backup as copy database format '+DG1';
 
With "BACKUP AS COPY", RMAN copies the files as image copies, bit-for-bit copies of database files created on disk.These are identical to copies of the same files that you can create with operating system commands like cp on Unix or COPY on Windows.However, using BACKUP AS COPY will be recorded in the RMAN repository and RMAN can use them in restore operations.

4)Switch the database to copy. At this moment we are switching to the new Diskgroup

RMAN> switch database to copy;

A SWITCH is equivalent to using the PL/SQL "alter database rename file" statement.

RMAN > alter database open resetlogs;

5)Add new tempfile to newly created database.

SQL> alter tablespace TEMP add tempfile '+DG1' SIZE 10M;

Drop any existing tempfile on the old diskgroup

SQL> alter database tempfile '+DG2/orcl10g/tempfile/temp.265.626631119' drop;

6)Find out how many members we have have in redolog groups, make sure that we have only one member in each log group.(drop other members).

Suppose we have 3 log groups, then add one member to each log group as following:

SQL> alter database add logfile member '+DG1' to group 1;
SQL> alter database add logfile member '+DG1' to group 2;
SQL> alter database add logfile member '+DG1' to group 3;

Then we can drop the old logfile member from earlier diskgroups as:

SQL> alter database drop logfile member 'complete_name';

7)Use the following query to verify that all the files are moved to new diskgroup with desired redundancy:

SQL> select name from v$controlfile
union
select name from v$datafile
union
select name from v$tempfile
union
select member from v$logfile
union
select filename from v$block_change_tracking

8) Enable block change tracking using  ALTER DATABASE command. 

SQL> alter database enable block change tracking using file ‘<FILE_NAME>’;

Case 2:Drop the existing diskgroup after database backup and create a new diskgroup with desired redundancy.

1.Shutdown(immediate) the database and then startup mount. Take a valid RMAN backup of existing database as:

RMAN> backup device type disk format 'd:\backup\%U' database ;

RMAN> backup device type disk format 'd:\backup\%U'archivelog all;

2. Make copy of spfile to accessible location:

SQL> create pfile='d:\initsid.ora' from spfile;
SQL> alter database backup controlfile to 'd:\control.ctl';

3. Shutdown the RDBMS instance

SQL> shutdown immediate

4. Connect to ASM Instance and Drop the existing Diskgroups
SQL> drop diskgroup DG1 including contents;

5. Shutdown ASM Instance;

6.Startup the ASM instance in nomount state  and Create the new ASM diskgroup

SQL>startup nomount
SQL> create diskgroup dg1 external redundancy disk'disk_name';

The new diskgroups name should be same as of previous diskgroup, it will facilitate the RMAN restore process. 

7. Connect to the RDBMS instance and startup in nomount state using pfile

startup nomount pfile='d:\initsid.ora'
SQL> create spfile from pfile='d:\initsid.ora'

8. Now restore the controlfile and backup's using RMAN

RMAN > restore controlfile from 'd:\control.ctl';
RMAN> alter database mount;
RMAN> restore database;
RMAN> recover database;

unable to find archive log
archive log thread=1 sequence=4
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 07/05/2007 18:24:32
RMAN-06054: media recovery requesting unknown log: thread 1 seq 4 lowscn 
570820 

While recovery it will give an error for archive log missing, this is expected we need to open the database with resetlogs as:

RMAN> alter database open resetlogs;

-We also need to change Flash Recovery Area to newly created diskgroup location. 

SQL> alter system set db_recovery_file_dest='+DG1' scope=both;

-We must now disable and re-enable Flashback Database so that the flashback log files are recreated in the +DG1 disk group and this can be done in mount state only.

SQL> alter database flashback off ;
SQL> alter database flashback on ;

- In case you want to use new name for diskgroup,in step 8 after mounting the database , you can use :

RMAN> run{
set newname for datafile 1 to '+DG2';
set newname for datafile 2 to '+DG2';
set newname for datafile 3 to '+DG2';
set newname for datafile 4 to '+DG2';
set newname for datafile 5 to '+DG2';
restore database;
switch datafile all;
recover database;
}
(assuming that we have 5 datafiles in our database)

```  



### OrphanDisks-Drop 

Detect all orphan files on ASM
```
DEFINE ASMGROUP="<DiskGroupName>"
 
set linesize 200
set pagesize 50000
col reference_index noprint
col type format a15
col files format a80

WITH v_asmgroup AS (SELECT group_number FROM v$asm_diskgroup WHERE name='&ASMGROUP'),
     v_parentindex AS (SELECT parent_index 
                    FROM v$asm_alias 
              WHERE group_number = (SELECT group_number FROM v_asmgroup) 
                AND alias_index=0),
  v_asmfiles AS (SELECT file_number, type 
              FROM v$asm_file 
           WHERE group_number = (SELECT group_number FROM v_asmgroup)),
 v_dbname AS (SELECT '/'||upper(db_unique_name)||'/' dbname from v$database)
SELECT 'test '|| files files FROM -- this line show the delete command
(
  SELECT '+&ASMGROUP'||files files, type 
  FROM (SELECT upper(sys_connect_by_path(aa.name,'/')) files, aa.reference_index, b.type
        FROM (SELECT file_number,alias_directory,name, reference_index, parent_index 
        FROM v$asm_alias) aa,
             (SELECT parent_index FROM v_parentindex) a,
             (SELECT file_number, type FROM v_asmfiles) b
  WHERE aa.file_number=b.file_number(+)
    AND aa.alias_directory='N'
   -- missing PARAMETERFILE, DATAGUARDCONFIG
   AND b.type in ('DATAFILE','ONLINELOG','CONTROLFILE','TEMPFILE')
  START WITH aa.PARENT_INDEX=a.parent_index
  CONNECT BY PRIOR aa.reference_index=aa.parent_index)
  WHERE substr(files,instr(files,'/',1,1),instr(files,'/',1,2)-instr(files,'/',1,1)+1) = (select dbname FROM v_dbname)
MINUS (
  SELECT upper(name) files, 'DATAFILE' type FROM v$datafile
    UNION ALL 
  SELECT upper(name) files, 'TEMPFILE' type FROM v$tempfile
    UNION ALL
 SELECT upper(name) files, 'CONTROLFILE' type FROM v$controlfile WHERE name like '+&ASMGROUP%'
    UNION ALL
 SELECT upper(member) files, 'ONLINELOG' type FROM v$logfile WHERE member like '+&ASMGROUP%'
)
);

```

FILES
--------------------------------------------------------------------------------
rm <command>


So then I just need to run from asmcmd:
```
ASMCMD> rm <command>
```
Because WITH clause does not work on MOUNTED databases (like a standby) - 
SQL Statement Using 'WITH' Clause Fails with ORA-01219 [ID 1166073.1] -, here the version without the WITH clause:

```
DEFINE ASMGROUP="PAMTEST_DATA_01"
 
set linesize 200
set pagesize 50000
col reference_index noprint
col type format a15
col files format a80

SELECT 'rm '|| files files FROM -- this line show the delete command
(
  SELECT '+&ASMGROUP'||files files, type 
  FROM (SELECT upper(sys_connect_by_path(aa.name,'/')) files, aa.reference_index, b.type
        FROM (SELECT file_number,alias_directory,name, reference_index, parent_index 
        FROM v$asm_alias) aa,
             (SELECT parent_index FROM (SELECT parent_index 
                    FROM v$asm_alias 
              WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name='&ASMGROUP') 
                AND alias_index=0)) a,
             (SELECT file_number, type FROM (SELECT file_number, type 
                                       FROM v$asm_file 
                                    WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name='&ASMGROUP'))) b
  WHERE aa.file_number=b.file_number(+)
    AND aa.alias_directory='N'
   -- missing PARAMETERFILE, DATAGUARDCONFIG
   AND b.type in ('DATAFILE','ONLINELOG','CONTROLFILE','TEMPFILE')
  START WITH aa.PARENT_INDEX=a.parent_index
  CONNECT BY PRIOR aa.reference_index=aa.parent_index)
  WHERE substr(files,instr(files,'/',1,1),instr(files,'/',1,2)-instr(files,'/',1,1)+1) = (select dbname FROM (SELECT '/'||upper(db_unique_name)||'/' dbname from v$database))
MINUS (
  SELECT upper(name) files, 'DATAFILE' type FROM v$datafile
    UNION ALL 
  SELECT upper(name) files, 'TEMPFILE' type FROM v$tempfile
    UNION ALL
 SELECT upper(name) files, 'CONTROLFILE' type FROM v$controlfile WHERE name like '+&ASMGROUP%'
    UNION ALL
 SELECT upper(member) files, 'ONLINELOG' type FROM v$logfile WHERE member like '+&ASMGROUP%'
)
);
```

### ACFS 

Oracle Automatic Storage Management Cluster File System (Oracle ACFS) is a multi-platform, scalable file system, and storage management technology that extends Oracle Automatic Storage Management (Oracle ASM) functionality to support customer files maintained outside of Oracle Database.
```
ASMCMD> volcreate -G data -s 2G –column 4 –width 128K –redundancy unprotected ACFS_VOLB
```
```
SQL>alter diskgroup data add volume ACFS_VOLA size 2G stripe_width 128K stripe_columns 4 ;
```

### Creating ACFS
```
1. Set the diskgroup attribute parameters. Compatible.asm and compatible.advm should be >= 11.2
$ sqlplus  / as sysasm
SQL> alter diskgroup flash set attribute 'compatible.asm'='11.2';
SQL> alter diskgroup flash set attribute 'compatible.advm'='11.2';2. 

2. Create a volume in the appropriate diskgroup
SQL> alter diskgroup flash add volume volume1 size 1g;
This could also be done using asmcmd. Volume device could be identified with 
select volume_name,volume_device from v$asm_volume;

VOLUME_NAME    VOLUME_DEVICE
-------------- ---------------------
VOLUMNE1       /dev/asm/volumne1-398

This could also confirmed with ls /dev/asm/*

3. Create a file system on the created volume
# mkfs -t acfs /dev/asm/volumne1-398
mkfs.acfs: version   = 11.2.0.2.0mkfs.acfs: on-disk version           = 39.0mkfs.acfs: volume                    = /dev/asm/volumne1-398mkfs.acfs: volume size               = 1073741824mkfs.acfs: Format complete.

4. Register the file system with 
acfsutil registry -a /dev/asm/volumne1-398       /opt/acfsvol/opt/acfsvol                    will be the mountpoint for this file system.

5. Mount the file system 
# mount -t acfs /dev/asm/volumne1-398 /opt/acfsvol

ASMCMD> volinfo -G flash VOLUMNE1

Diskgroup Name: FLASH         Volume Name: VOLUMNE1         Volume Device: /dev/asm/volumne1-398         State: ENABLED         Size (MB): 1024         Resize Unit (MB): 256         Redundancy: UNPROT         Stripe Columns: 4         Stripe Width (K): 128         Usage: ACFS         Mountpath: /opt/acfsvol6. Change ownership to Oracle user to allow oracle user processes to use the file system 
# chown oracle:oinstall /opt/acfsvol6. As oracle user create a file in the mount point 
cd /opt/acfsvoltouch xIn a RAC system the volume will be mounted on all nodes after server reboots and requires selinux to be permissive.
```

### Removing ACFS
```
1. Unmount the file system 
# umount /opt/acfsvol2. De-register the file system
# acfsutil registry -d /opt/acfsvolacfsutil registry: successfully removed ACFS mount point /opt/acfsvol from Oracle Registry3. Drop the volumne from the diskgroup
SQL> alter diskgroup flash drop volume volume1;Diskgroup altered.SQL> select volume_name,volume_device from v$asm_volume;no rows selected

SYS@+ASM2 SQL> select volume_name,volume_device from v$asm_volume;

VOLUME_NAME
------------------------------------------------------------------------------------------
VOLUME_DEVICE
------------------------------------------------------------------------------------------------------------------------------------------------------
SOFTWARE
/dev/asm/software-193

ASMCMD> volinfo --all
Diskgroup Name: DBFS_DG

         Volume Name: SOFTWARE
         Volume Device: /dev/asm/software-193
         State: ENABLED
         Size (MB): 51200
         Resize Unit (MB): 64
         Redundancy: MIRROR
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage: ACFS
         Mountpath: /software

ASMCMD> volinfo -G DBFS_DG -a
	Diskgroup Name: DBFS_DG

         Volume Name: SOFTWARE
         Volume Device: /dev/asm/software-193
         State: ENABLED
         Size (MB): 51200
         Resize Unit (MB): 64
         Redundancy: MIRROR
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage: ACFS
         Mountpath: /software


mount -t acfs /dev/asm/software-193 /software

$ ls -l /dev/asm/software-193
brwxrwx--- 1 root asmadmin 251, 98817 Jan  4 12:05 /dev/asm/software-193

ASMCMD> volinfo --all
Diskgroup Name: DBFS_DG

         Volume Name: SOFTWARE
         Volume Device: /dev/asm/software-193
         State: ENABLED
         Size (MB): 51200
         Resize Unit (MB): 64
         Redundancy: MIRROR
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage: ACFS
         Mountpath: /software

srvctl start volume -volume SOFTWARE -diskgroup DBFS_DG

root@iorsdb02-adm::/root
$ mkfs -t acfs /dev/asm/software-193
mkfs.acfs: version                   = 12.1.0.2.0
mkfs.acfs: on-disk version           = 39.0
mkfs.acfs: volume                    = /dev/asm/software-193
mkfs.acfs: ACFS-01010: Volume already contains an ACFS file system.  To reformat the volume, reissue mkfs.acfs with the -f option.
mkfs.acfs: ACFS-01004: /dev/asm/software-193 was not formatted.
```
#### ===================================================================Using acmcmd ==========================

```

1. create a volume
asmcmd>volcreate -G ACFS -s 3G TEST
2. check
asmcmd>volinfo -G ACFS -a
:
volume Device: /dev/asm/test-***
3. create an ACFS file system in the TEST volume
#mkfs -t acfs  /dev/asm/test-***
4. Mount the volume all the nodes
#mkdir -p /u01/app/oracle/acfsmount/test
#mount -t acfs  /dev/asm/test-***   /u01/app/oracle/acfsmount/test
5. register the volume
#acfsutil registry -a /dev/asm/test-***   /u01/app/oracle/acfsmount/test
Check
#acfsutil registry -l

RESIZE ACFS filesystem
======================
resize ACFS filesystem will automatically resize the volume

# acfsutil size +512M /u01/app/oracle/acfsmount/test
Check
asmcmd volinfo -G ACFS TEST

Check the ACFS:
Node1:
# df -k | grep 'Filesystem \|u01'  
```

### RAC 



Master Note ID 1096952.1

$ 
ggdev_app                                                                  cluster_resource  OFFLINE        OFFLINE
ggdevusr_app                                                               cluster_resource  ONLINE         ONLINE on iorsdb01-adm
iorscl01-ggdevusrvip                                                       app.appvipx.type  ONLINE         ONLINE on iorsdb01-adm
iorscl01-ggvip                                                             app.appvipx.type  ONLINE         ONLINE on iorsdb01-adm
ora.DATA.dg                                                                ora.diskgroup.typeONLINE         ONLINE on iorsdb01-adm
ora.DBFS_DG.SOFTWARE.advm                                                  ora.volume.type   ONLINE         ONLINE on iorsdb01-adm
ora.DBFS_DG.dg                                                             ora.diskgroup.typeONLINE         ONLINE on iorsdb01-adm
ora.LISTENER.lsnr                                                          ora.listener.type ONLINE         ONLINE on iorsdb01-adm
ora.LISTENER_DEF.lsnr                                                      ora.listener.type ONLINE         ONLINE on iorsdb01-adm
ora.LISTENER_DG.lsnr                                                       ora.listener.type ONLINE         ONLINE on iorsdb01-adm
ora.LISTENER_SCAN1.lsnr                                                    ora.scan_listener.typeONLINE         ONLINE on iorsdb01-adm
ora.LISTENER_SCAN2.lsnr                                                    ora.scan_listener.typeONLINE         ONLINE on iorsdb02-adm
ora.LISTENER_SCAN3.lsnr                                                    ora.scan_listener.typeONLINE         ONLINE on iorsdb02-adm
ora.MGMTLSNR                                                               ora.mgmtlsnr.type ONLINE         ONLINE on iorsdb01-adm
ora.RECO.dg                                                                ora.diskgroup.typeONLINE         ONLINE on iorsdb01-adm


$ crsctl status res ora.dbdw1_01.db -p | grep -i auto_start
AUTO_START=restore
[grid@iorsdb01-adm (16:05:57) +ASM1:~]
$ crsctl status res ora.pmq21_02.db -p | grep -i auto_start
AUTO_START=restore
  



12c 



Transaction Guard (12c)
To implement this feature, you need to create a service with two attributes, namely, commit_outcome and retention. 
The commit_outcome attribute determines whether the logical transaction outcome will be tracked or not. 
Attribute retention specifies the number of seconds a transaction outcome will be stored in the database (defaults to a value of 86,400 seconds = 1 day). 
The following command shows an example of po service creation with commit_outcome set to TRUE, enabling Transaction Guard.

$ srvctl add service -db orcl12 -service po -preferred oel6vm1 -available oel6vm2 -commit_outcome TRUE –retention 86400  



ADD/Remove nodes 



NOTE: http://oracle.su/docs/11g/rac.112/e10717/adddelclusterware.htm [4 Adding and Deleting Cluster Nodes]

After you clone your rac1 or rac2 -- rename that machine to rac3; finish all the normal setup checks for cluster you can follow the below:

Add New Node to cluster
======================
1. setup the ssh user equivalence for the GRID user between first node and the third node.
2. Check passwordless connection to 3rd node from 1st node
3.. set environment variable:
oraenv
+ASM
4. Pre installation check for 3rd node using cluvfy
$cluvfy stage -pre crsinst -n rac3
5. Generate the fixup script for the third node (rac3) with cluvfy with the -fixup option
$cluvfy stage -pre crsinst -n rac3 -fixup
6. Run the fixup script as directed by the above (step5 output)
$ssh root@rac3 /tmp/CVU_11.2.0.1.0_grid/runfixup.sh
7. using the cluster verification utility. make sure that you can add your third node to the cluster
$cluvfy stage -pre nodeadd -n rac3

###############################################
## NOTE: ignore error like check failed on nodes:rac3##
###############################################

8. Add your third node to the cluster from first node:
$ cd $GRID_HOME/oui/bin
$ ./addNode.sh -silent "CLUSTER_NEW_NODES={rac3}" "CLUSTER_NEW_VIRTUAL_HOSTNAMES={rac3-vip}"

9. Connected on third node (rac3) execute the following script

$ssh root@rac3
RNING:
The following configuration scripts need to be executed as the root user in each cluster node.
/u01/app/11.2.0/grid/root.sh #On nodes server3
To execute the configuration scripts:
    1. Open a terminal window
    2. Log in as root
    3. Run the scripts in each cluster node
The Cluster Node Addition of /u01/app/11.2.0/grid was successful.
Please check ‘/tmp/silentInstall.log’ for more details. - See more at: http://blog.grid-it.nl/index.php/2011/04/13/adding-a-new-node-to-an-oacle-11gr2-cluster/#sthash.KZu3NpeJ.dpuf

10.  from first node: 
verify the cluster integration after install
============================================
$GRID_HOME/bin/cluvfy stage -post nodeadd -n rac3 -verbose 

11. Make sure that the disk groups are mounted on all three nodes

$crsctl stat res ora.FRA.dg -t
$crsctl stat res ora.ACFS.dg -t
If 
$ssh rac3
. oraenv
+ASM3
asmcmd mount ACFS
asmcmd mounr FRA
 
Check
========
$ crs_stat -t -v

$ srvctl status asm -a
$ olsnodes -n

)Now we need to run the addNode.sh from the RDBMS home as oracle user.
[oracle@server1 ~]$ /u01/app/oracle/product/11.2.0/dbhome_1/oui/bin/addNode.sh -silent CLUSTER_NEW_NODES={rac3}

# /u01/app/oracle/product/11.2.0/dbhome_1/root.sh

An additional step to validate and to change if not correct, is required when you use role seperation, as in this example.
This is required to make sure the ASM instances are able to host the database instances when created.
Ensure permissions for Oracle executable are 6751, and group is asmadmin, as root user:
cd $ORACLE_HOME/bin
chgrp asmadmin oracle
chmod 6751 oracle
ls -l oracle

Now the database software is installed we add the database instance, we execute this is silent mode.
On any existing node, run DBCA ($ORACLE_HOME/bin/dbca) to add the new instance:
[oracle@server1 ~]$ /u01/app/oracle/product/11.2.0/dbhome_1/bin/dbca -silent -addInstance -nodeList server2 -gdbName ORCL -instanceName ORCL2 -sysDBAUserName sys -sysDBAPassword oracle


===============================================
Remove Node From Cluster:
============================================
Set your environment
# . oraenv
+ASM1
# export $GRID_HOME=/u01/app/11.2.0/grid

Check the Node Status
===================
#cd $GRID_HOME/bin/
#./olsnodes -s -t

Unpin the node (being removed)
#./crsctl unpin css -n rac3

Stop Agent/database control on rac3
=============================
ssh rac3
# emctl 
#export $GRID_HOME=/u01/app/11.2.0/grid
#cd $GRID_HOME/crs/install
# ./rootcrs.pl -deconfig -force

If there are issues with VIP then use to remove by force option

# srvctl stop vip -i rac3-vip -f
# srvctl remove vip -i rac3-vip -f

Update Cluster information to other node
========================================= 
#./crsctl delete node -n rac3
Check pin again
#./olsnodes -t -s

Update Inventory
================
cd $GRID_HOME/oui/bin
./runInstaller -updateNodeList ORACLE_HOME=$GRID_HOME "CLUSTER_NODES={rac3}" CRS=TRUE -local

11g: <grid_home>/gi_000/oui/bin

12c: 

/u01/app/12.1.0.2/gi_000/addnode

Check that node 3 is ready for node addition 
Connect from Node1 : 
$ cluvfy stage -pre nodeadd -n <Node3>
If anything failed with cluvfy -- verify you can ignore (Ez - Swap space /memory check) 
$ more addnode.sh
---
 if [ $HOME_TYPE = "crs" ];then
      echo "      addnode.sh \"CLUSTER_NEW_NODES={comma-separated node names}\""
      echo "                 \"[CLUSTER_NEW_VIRTUAL_HOSTNAMES={comma-separated virtual host names}]\""
      echo "                 \"[CLUSTER_NEW_NODE_ROLES={comma-separated roles of nodes specified}]\""
   fi
   if [ $HOME_TYPE = "db" ];then
      echo "      addnode.sh \"CLUSTER_NEW_NODES={comma-separated node names}\""
   fi
-----  



clusterware Log files 



12c: 
$ crsctl query crs activeversion;
Oracle Clusterware active version on the cluster is [12.1.0.2.0]

grid@iorsdb02-adm:+ASM2:/u01/app/grid/diag/crs/iorsdb02-adm/crs/trace

If not one node then go to other node:
/u01/app/grid/diag/crs/iorsdb02-adm/crs/trace/ocssd.trc
Another noticeable thing is that name of clusterware alert log has been changed to alert.log as compared to alert<hostname>.log in 11g.
11g: 

In 12.1.0.1 flex cluster though, the location and name of alert log location is same as in 11g i.e. $ORACLE_HOME/log/host01


°° $CRS_HOME\log\nodename\racg contains logfiles for the VIP and ONS resources

CRS(Cluster Ready Services) logs are in <Grid_HOME>/log/<hostname/crsd/                              -->crsd.log file is archived every 10MB
CSS(Cluster Synchronization Services) logs are in <Grid_HOME>/log/<hostname/css/                              -->cssd.log file is archived every 20MB
EVM(Event Manager logs are in <Grid_HOME>/log/<hostname/evmd                        
SRVM(srvctl) and OCR(ocdrump, ocrconfig, ocrcheck) logs are in  <Grid_HOME>/log/<hostname/client/ and $DB_HOME>/log/<hostname/client/

ALERT: <GRID_HOME/log/<hostname>/alert<nodename.log>

OCR(ocdrump, ocrconfig, ocrcheck) tools logs are in  <Grid_HOME>/log/<hostname>/client

ASM related trace and alert information --- <Grid_Base>/diag/asm/+asm/+ASMn directory

 --- To enable tracing for cluvfy, netca, and srvctl, set SRVM_TRACE to TRUE                   
$export  SRVM_TRACE=TRUE
$srvctl config database -d orcl > /tmp/srvctl.trc

ORACLE SUPPORT
======================
<GRID_HOME>/bin/diagcollection.pl    --collect                  script as a root user
--collect option : --crs for clusterware logs, --core foe collecting core files, or --all for collecting all logs (default)
-clean              clean files

#########################################################################################
Oracle Clusterware alert log:

1. . oraenv
+ASM1
cd $ORACLE_HOME/log/$ST_NODE1
view alert$ST_NODE1.log
2. Navigate to Oracle Cluster Synchronization Services daemon log directory and determine whether any log archivs exist
cd ./cssd
cd /u01/app/11.2.0/grid/log/rac1/cssd

3. To gather all log files  -- to send Oracle Support

# diagcollection.pl --collect --crshome /u01/app/11.2.0/grid

list the resulting log file archives that were generated with the diagcollection.pl script

#ls -la *tar.gz

OCRDUMP
===================
OCRDUMP utility to dump the binary file into text and XML representations.

$ocrdump -stdout | wc -l
#ocrdump -stdout | wc -l
dump the first 25 lines of the OCR to standard output using XML format
#ocrdump -stdout -xml | head -25
--automatic backup of the OCR
#ocrconfig -showbackup

  



crsctl-srvctl comands 



CRSCTL (Clusterware Control utility)
=================================================
crsctl check crs - checks the viability of the CRS stack
crsctl check cssd - checks the viability of CSS
crsctl check crsd - checks the viability of CRS
crsctl check evmd - checks the viability of EVM
crsctl set css <parameter> <value> - sets a parameter override
crsctl get css <parameter> - gets the value of a CSS parameter
crsctl unset css <parameter> - sets CSS parameter to its default
crsctl query css votedisk - lists the voting disks used by CSS
crsctl add css votedisk <path> - adds a new voting disk
crsctl delete css votedisk <path> - removes a voting disk
crsctl enable crs - enables startup for all CRS daemons
crsctl disable crs - disables startup for all CRS daemons
crsctl start crs - starts all CRS daemons.
crsctl stop crs - stops all CRS daemons. Stops CRS resources in case of cluster.
crsctl start resources - starts CRS resources.
crsctl stop resources - stops CRS resources.
crsctl debug statedump evm - dumps state info for evm objects
crsctl debug statedump crs - dumps state info for crs objects
crsctl debug statedump css - dumps state info for css objects
crsctl debug log css [module:level]{,module:level} ...- Turns on debugging for CSS
crsctl debug trace css - dumps CSS in-memory tracing cache
crsctl debug log crs [module:level]{,module:level} ...- Turns on debugging for CRS
crsctl debug trace crs - dumps CRS in-memory tracing cache
crsctl debug log evm [module:level]{,module:level} ...- Turns on debugging for EVM
crsctl debug trace evm - dumps EVM in-memory tracing cache
crsctl debug log res <resname:level> turns on debugging for resources
crsctl query crs softwareversion [<nodename>] - lists the version of CRS software installed
crsctl query crs activeversion - lists the CRS software operating version
crsctl lsmodules css - lists the CSS modules that can be used for debugging
crsctl lsmodules crs - lists the CRS modules that can be used for debugging
crsctl lsmodules

display the registered databases 
-----------------------------------------------------
srvctl config database

status
---------------------------------------------------------
srvctl status database -d <database
srvctl status instance -d <database> -i <instance> 
srvctl status nodeapps -n <node>
srvctl status service -d <database> 
srvctl status asm -n <node> 

stopping/starting 
------------------------------
srvctl stop database -d <database>
srvctl stop instance -d <database> -i <instance>,<instance>
srvctl stop service -d <database> [-s <service><service>] [-i <instance>,<instance>]
srvctl stop nodeapps -n <node>
srvctl stop asm -n <node> 

srvctl start database -d <database>
srvctl start instance -d <database> -i <instance>,<instance>
srvctl start service -d <database> -s <service><service> -i <instance>,<instance>
srvctl start nodeapps -n <node>
srvctl start asm -n <node>

adding/removing
---------------------------------------------------------
srvctl add database -d <database> -o <oracle_home>
srvctl add instance -d <database> -i <instance> -n <node>
srvctl add service -d <database> -s <service> -r <preferred_list>
srvctl add nodeapps -n <node> -o <oracle_home> -A <name|ip>/network
srvctl add asm -n <node> -i <asm_instance> -o <oracle_home> 

srvctl remove database -d <database> -o <oracle_home>
srvctl remove instance -d <database> -i <instance> -n <node>
srvctl remove service -d <database> -s <service> -r <preferred_list>
srvctl remove nodeapps -n <node> -o <oracle_home> -A <name|ip>/network
srvctl asm remove -n <node> 



CRS_STAT (Cluster Ready Services Statistics)
=========================================================
to examine the general status for CLUSTERWARE RESOURCES AND APPLICATIONS, the crs_stat -t command can be issued as shown next to
display the status of these resources in a tabular format:
$ crs_stat -t 

to examine the CLUSTERWARE PARAMETER settings by using the crs_stat pp option as shown here:
$ crs_stat -p

to examine the Clusterware permissions and ownership, you can use the crs_stat -ls option as shown here:
$ crs_stat -ls

OCRCHECK (Oracle Cluster Registry Check Utility)
=================================================
$ ocrcheck -help

OCRCONFIG (Oracle Cluster Registry Config Utility)
========================================================
# Exporting and importing OCR contents to a file
# Restoring a corrupted OCR from a physical backup
# Adding or replacing an OCR copy with another OCR file
# Repairing a damaged OCR with a new OCR version
$ ocrconfig -help
Note: A log file will be created in $ORACLE_HOME/log/<hostname>/client/ocrconfig_<pid>.log. Please
ensure you have file creation privileges in the above directory before running this tool.

CLSCFG (Clusterware Config Tool-11g)
==============================================================
This utility provides a host of features for managing
and updating your Oracle 11g RAC Clusterware configurations, allowing you to
perform the following administration tasks for the Oracle 11g Clusterware:
# Creating a new 11g Clusterware configuration
# Upgrading existing Clusterware
# Adding or removing nodes from the current 11g Clusterware
clscfg -help
NOTE: BE EXTREMELY CAREFUL WHEN USING CLSCFG IN A PRODUCTION 11G RAC ENVIRONMENT! AS ORACLE SUPPORT CAUTIONS, THE CARELESS USAGE OF THIS TOOL IS DANGEROUS AND MAY CORRUPT YOUR 11G CLUSTERWARE. 

CLUVFY (Clusterware Verification Utility)
$ cluvfy

cd $ORA_CRS_HOME/bin

--Pre Install
./runcluvfy.sh stage -pre crsinst -n <node1>,<node2> -verbose | tee /tmp/cluvfy_preinstall.log
./cluvfy stage -post crsinst -n all -verbose | tee /tmp/cluvfy_postinstall.log

--Post Install
$ cluvfy stage -post crsinst -n raclinux1 -verbose

  



Interview 



Cluster: A group of independent, but interconnected, computers that act as a single system.
Clusterware: cluster manager software  that is tightly integrated with the operating system (OS) to provide the cluster management functions that enable the Oracle database in the cluster environment.
 
What are Oracle Cluster Components?
Cluster Interconnect (HAIP)
Shared Storage (OCR/Voting Disk)
Clusterware software

What are Oracle RAC Components?
VIP, Node apps etc.

What is RAC?
RAC stands for Real Application cluster. It is a clustering solution from Oracle Corporation that ensures high availability of databases by providing instance failover, media failover features.

Oracle RAC software components:-
=============================
Oracle RAC is composed of two or more database instances. They are composed of Memory structures and background processes same as the single instance database.

Process
============
It has the same set of background processes
and the same memory structure, such as the System Global Area (SGA) and the Program Global Area (PGA). As
well as these, RAC instances also have the additional processes and memory structure that are dedicated to the GCS
processes, GES, and the global cache directory. These processes are as follows:
 LMS: Lock Manager Server process
        The Lock Manager Server is the Global Cache Service (GCS) process. 
        This process is responsible for transferring the data blocks between the RAC instances for cache fusion requests.
 LMON: Lock Monitor processes
        The Lock Monitor process is responsible for managing the Global Enqueue Service (GES). 
        It is also responsible for reconfiguration of lock resources when an instance joins or leaves the cluster and responsible for dynamic lock  remastering.
 LMD: Lock Monitor daemon process
        The Lock Monitor daemon process is the Global Enqueue Service (GES).
        The LMD process manages the incoming remote lock requests from other instances in the cluster.
 LCK: Lock process
         The Lock process manages non-cache fusion resource requests, such as row cache and library cache requests.
          Only one LCK process (lck0) per instance.
 DIAG: Diagnostic daemon
          The Diagnostic daemon process is responsible for all the diagnostic work in a RAC instance.

Oracle 11gR2 introduced a few new processes for RAC.
===========================================================
ACMS: Atomic Controlfile to Memory Server                     ==For Database
GTXn: Global Transaction process                                    == For Database
LMHB: LockMonitor heartbeat (monitors LMON, LMD, LMSn processes)
PING: Interconnect latency measurement process
RMS0: RAC management server                                     ==For database
RSMN: Remote Slave Monitor			     == For database

Transparent Application Failover (TAF) is a client-side feature that allows for clients to reconnect to surviving databases in the event of a failure of a database instance.

Fast Connection Failover (FCF) allows jdbc connections to failover to surviving nodes in a RAC when one or more nodes are unavailable. This is very useful to have with middleware application server connection pools so when rolling upgrades or patch application is done on the database the connections in the middleware server would failover to surviving instances and client connections would not receive any errors.

What is GRD?
GRD stands for Global Resource Directory. The GES and GCS maintains records of the statuses of each datafile and each cahed block using global resource directory.This process is referred to as cache fusion and helps in data integrity.

Give Details on Cache Fusion:-
Oracle RAC is composed of two or more instances. When a block of data is read from datafile by an instance within the cluster and another instance is in need of the same block,it is easy to get the block image from the insatnce which has the block in its SGA rather than reading from the disk. To enable inter instance communication Oracle RAC makes use of interconnects. The Global Enqueue Service(GES) monitors and Instance enqueue process manages the cahce fusion. 

Give Details on ACMS:-
ACMS stands for Atomic Controlfile Memory Service.In an Oracle RAC environment ACMS is an agent that ensures a distributed SGA memory update(ie)SGA updates are globally committed on success or globally aborted in event of a failure.
Give details on GTX0-j :-
The process provides transparent support for XA global transactions in a RAC environment.The database autotunes the number of these processes based on the workload of XA global transactions.
Give details on LMON:-
This process monitors global enques and resources across the cluster and performs global enqueue recovery operations.This is called as Global Enqueue Service Monitor.
Give details on LMD:-
This process is called as global enqueue service daemon. This process manages incoming remote resource requests within each instance.
Give details on LMS:-
This process is called as Global Cache service process.This process maintains statuses of datafiles and each cahed block by recording information in a Global Resource Dectory(GRD).This process also controls the flow of messages to remote instances and manages global data block access and transmits block images between the buffer caches of different instances.This processing is a part of cache fusion feature.
Give details on LCK0:-
This process is called as Instance enqueue process.This process manages non-cache fusion resource requests such as libry and row cache requests.
Give details on RMSn:-
This process is called as Oracle RAC management process.These pocesses perform managability tasks for Oracle RAC.Tasks include creation of resources related Oracle RAC when new instances are added to the cluster.
Give details on RSMN:-
This process is called as Remote Slave Monitor.This process manages background slave process creation andd communication on remote instances. This is a background slave process.This process performs tasks on behalf of a co-ordinating process running in another instance.

What components in RAC must reside in shared storage?
All datafiles, controlfiles, SPFIles, redo log files must reside on cluster-aware shred storage.

What is the significance of using cluster-aware shared storage in an Oracle RAC environment?
All instances of an Oracle RAC can access all the datafiles,control files, SPFILE's, redolog files when these files are hosted out of cluster-aware shared storage which are group of shared disks.

Give few examples for solutions that support cluster storage:-
ASM(automatic storage management),raw disk devices,network file system(NFS), OCFS2 and OCFS(Oracle Cluster Fie systems).

What is an interconnect network?
an interconnect network is a private network that connects all of the servers in a cluster. The interconnect network uses a switch/multiple switches that only the nodes in the cluster can access.

How can we configure the cluster interconnect?
Configure User Datagram Protocol(UDP) on Gigabit ethernet for cluster interconnect.On unia and linux systems we use UDP and RDS(Reliable data socket) protocols to be used by Oracle Clusterware.Windows clusters use the TCP protocol.

Can we use crossover cables with Oracle Clusterware interconnects?
No, crossover cables are not supported with Oracle Clusterware intercnects.

What is the use of cluster interconnect?
Cluster interconnect is used by the Cache fusion for inter instance communication.

How do users connect to database in an Oracle RAC environment?
Users can access a RAC database using a client/server configuration or through one or more middle tiers ,with or without connection pooling.Users can use oracle services feature to connect to database.
What is the use of a service in Oracle RAC environemnt?
Applications should use the services feature to connect to the Oracle database.Services enable us to define rules and characteristics to control how users and applications connect to database instances.
What are the characteriscs controlled by Oracle services feature?
The charateristics include a unique name, workload balancing and failover options,and high availability characteristics.

Which enable the load balancing of applications in RAC?
Oracle Net Services enable the load balancing of application connections across all of the instances in an Oracle RAC database.

What is a virtual IP address or VIP?
A virtl IP address or VIP is an alternate IP address that the client connectins use instead of the standard public IP address. To configureVIP address, we need to reserve a spare IP address for each node, and the IP addresses must use the same subnet as the public network.

What is the use of VIP?
If a node fails, then the node's VIP address fails over to another node on which the VIP address can accept TCP connections but it cannot accept Oracle connections.

Give situations under which VIP address failover happens:-
VIP addresses failover happens when the node on which the VIP address runs fails, all interfaces for the VIP address fails, all interfaces for the VIP address are disconnected from the network.

What is the significance of VIP address failover?
When a VIP address failover happens, Clients that attempt to connect to the VIP address receive a rapid connection refused error .They don't have to wait for TCP connection timeout messages.

What are the administrative tools used for Oracle RAC environments?
Oracle RAC cluster can be administered as a single image using OEM(Enterprise Manager),SQL*PLUS,Servercontrol(SRVCTL),cluster verification utility(cvu),DBCA,NETCA

How do we verify that RAC instances are running?
Issue the following query from any one node connecting through SQL*PLUS.
$connect sys/sys as sysdba
SQL>select * from V$ACTIVE_INSTANCES;
The query gives the instance number under INST_NUMBER column,host_:instancename under INST_NAME column.

What is FAN?
Fast application Notification as it abbreviates to FAN relates to the events related to instances,services and nodes.This is a notification mechanism that Oracle RAc uses to notify other processes about the configuration and service level information that includes service status changes such as,UP or DOWN events.Applications can respond to FAN events and take immediate action.

Where can we apply FAN UP and DOWN events?
FAN UP and FAN DOWN events can be applied to instances,services and nodes.

State the use of FAN events in case of a cluster configuration change?
During times of cluster configuration changes,Oracle RAC high availability framework publishes a FAN event immediately when a state change occurs in the cluster.So applications can receive FAN events and react immediately.This prevents applications from polling database and detecting a problem after such a state change.
Why should we have seperate homes for ASm instance?
It is a good practice to have ASM home seperate from the database hom(ORACLE_HOME).This helps in upgrading and patching ASM and the Oracle database software independent of each other.Also,we can deinstall the Oracle database software independent of the ASM instance.
What is the advantage of using ASM?
Having ASM is the Oracle recommended storage option for RAC databases as the ASM maximizes performance by managing the storage configuration across the disks.ASM does this by distributing the database file across all of the available storage within our cluster database environment.
What is rolling upgrade?
It is a new ASM feature from Database 11g.ASM instances in Oracle database 11g release(from 11.1) can be upgraded or patched using rolling upgrade feature. This enables us to patch or upgrade ASM nodes in a clustered environment without affecting database availability.During a rolling upgrade we can maintain a functional cluster while one or more of the nodes in the cluster are running in different software versions.
Can rolling upgrade be used to upgrade from 10g to 11g database?
No,it can be used only for Oracle database 11g releases(from 11.1).
State the initialization parameters that must have same value for every instance in an Oracle RAC database:-
Some initialization parameters are critical at the database creation time and must have same values.Their value must be specified in SPFILE or PFILE for every instance.The list of parameters that must be identical on every instance are given below:
ACTIVE_INSTANCE_COUNT
ARCHIVE_LAG_TARGET
COMPATIBLE
CLUSTER_DATABASE
CLUSTER_DATABASE_INSTANCE
CONTROL_FILES
DB_BLOCK_SIZE
DB_DOMAIN
DB_FILES
DB_NAME
DB_RECOVERY_FILE_DEST
DB_RECOVERY_FILE_DEST_SIZE
DB_UNIQUE_NAME
INSTANCE_TYPE (RDBMS or ASM)
PARALLEL_MAX_SERVERS
REMOTE_LOGIN_PASSWORD_FILE
UNDO_MANAGEMENT
Can the DML_LOCKS and RESULT_CACHE_MAX_SIZE be identical on all instances?
These parameters can be identical on all instances only if these parameter values are set to zero.
What two parameters must be set at the time of starting up an ASM instance in a RAC environment?The parameters CLUSTER_DATABASE and INSTANCE_TYPE must be set.
Mention the components of Oracle clusterware:-
Oracle clusterware is made up of components like voting disk and Oracle Cluster Registry(OCR). What is a CRS resource?
Oracle clusterware is used to manage high-availability operations in a cluster.Anything that Oracle Clusterware manages is known as a CRS resource.Some examples of CRS resources are database,an instance,a service,a listener,a VIP address,an application process etc.
What is the use of OCR?
Oracle clusterware manages CRS resources based on the configuration information of CRS resources stored in OCR(Oracle Cluster Registry).
How does a Oracle Clusterware manage CRS resources?
Oracle clusterware manages CRS resources based on the configuration information of CRS resources stored in OCR(Oracle Cluster Registry).
Name some Oracle clusterware tools and their uses?
OIFCFG - allocating and deallocating network interfaces
OCRCONFIG - Command-line tool for managing Oracle Cluster Registry
OCRDUMP - Identify the interconnect being used
CVU - Cluster verification utility to get status of CRS resources
What are the modes of deleting instances from ORacle Real Application cluster Databases?
We can delete instances using silent mode or interactive mode using DBCA(Database Configuration Assistant).
How do we remove ASM from a Oracle RAC environment?
We need to stop and delete the instance in the node first in interactive or silent mode.After that asm can be removed using srvctl tool as follows:
srvctl stop asm -n node_name
srvctl remove asm -n node_name
We can verify if ASM has been removed by issuing the following command:
srvctl config asm -n node_name
How do we verify that an instance has been removed from OCR after deleting an instance?
Issue the following srvctl command:
srvctl config database -d database_name
cd CRS_HOME/bin
./crs_stat
How do we verify an existing current backup of OCR?
We can verify the current backup of OCR using the following command : ocrconfig -showbackup
What are the performance views in an Oracle RAC environment?
We have v$ views that are instance specific. In addition we have GV$ views called as global views that has an INST_ID column of numeric data type.GV$ views obtain information from individual V$ views.
What are the types of connection load-balancing?
There are two types of connection load-balancing:server-side load balancing and client-side load balancing.
What is the differnece between server-side and client-side connection load balancing?
Client-side balancing happens at client side where load balancing is done using listener.In case of server-side load balancing listener uses a load-balancing advisory to redirect connections to the instance providing best service.
Give the usage of srvctl:-
srvctl start instance -d db_name -i "inst_name_list" [-o start_options]srvctl stop instance -d name -i "inst_name_list" [-o stop_options]srvctl stop instance -d orcl -i "orcl3,orcl4" -o immediatesrvctl start database -d name [-o start_options]srvctl stop database -d name [-o stop_options]srvctl start database -d orcl -o mount


1. What is cache fusion?
In a RAC environment, it is the combining of data blocks, which are shipped across the interconnect from remote database caches (SGA) to the local node, in order to fulfill the requirements for a transaction (DML, Query of Data Dictionary).

3. What is split brain?
When database nodes in a cluster are unable to communicate with each other, they may continue to process and modify the data blocks independently. If the same block is modified by more than one instance, synchronization/locking of the data blocks does not take place and blocks may be overwritten by others in the cluster. This state is called split brain.

4. What is the interconnect used for?
It is a private network which is used to ship data blocks from one instance to another for cache fusion. The physical data blocks as well as data dictionary blocks are shared across this interconnect.

5. How do you determine what protocol is being used for Interconnect traffic?
One of the ways is to look at the database alert log for the time period when the database was started up.

6. What methods are available to keep the time synchronized on all nodes in the cluster?
Either the Network Time Protocol(NTP) can be configured or in 11gr2, Cluster Time Synchronization Service (CTSS) can be used.

7. Where does the Clusterware write when there is a network or Storage missed heartbeat?
The network ping failure is written in CRS_HOME/log

8. How do you find out what OCR backups are available?
The ocrconfig -showbackup can be run to find out the automatic and manually run backups.

9. If your OCR is corrupted what options do have to resolve this?
You can use either the logical or the physical OCR backup copy to restore the Repository.

10. How do you find out what object has its blocks being shipped across the instance the most?

11. What is OCLUMON used for in a cluster environment?
The Cluster Health Monitor (CHM) stores operating system metrics in the CHM repository for all nodes in a RAC cluster. It stores information on CPU, memory, process, network and other OS data, This information can later be retrieved and used to troubleshoot and identify any cluster related issues. It is a default component of the 11gr2 grid install. The data is stored in the master repository and replicated to a standby repository on a different node.

12. What would be the possible performance impact in a cluster if a less powerful node (e.g. slower CPU’s) is added to the cluster?
All processing will show down to the CPU speed of the slowest server.

13. What is the purpose of OLR?

14. What is the default memory allocation for ASM?
In 10g the default SGA size is 1G in 11g it is set to 256M and in 12c ASM it is set back to 1G.

15. How do you backup ASM Metadata?
You can use md_backup to restore the ASM diskgroup configuration in-case of ASM diskgroup storage loss.

16. What files can be stored in the ASM diskgroup?
In 11g the following files can be stored in ASM diskgroups.
o Datafiles
o Redo logfiles
o Spfiles

In 12c the files below can also new be stored in the ASM Diskgroup
o Password file

17. What it the ASM POWER_LIMIT?
This is the parameter which controls the number of Allocation units the ASM instance will try to rebalance at any given time. In ASM versions less than 11.2.0.3 the default value is 11 however it has been changed to unlimited in later versions.

  



Issues 



[oracle@csmper-cls16 ~]$ srvctl start instance -i pqa182 -d pqa18
PRKP-1001 : Error starting instance pqa182 on node csmper-cls16
CRS-0215: Could not start resource 'ora.pqa18.pqa182.inst'.

$srvctl setenv database -d DB -T TNS_ADMIN=/opt/oracle/11.2.0.2_grid/grid/network/admin
$ srvctl start instance -i pqa182 -d pqa18

problem:
srvctl start instance -d MYDB -i MYINST
PRKP-1001 : Error starting instance MYINST on node mynode
CRS-0215: Could not start resource 'ora.MYDB.MYINST.inst'.

Solution:
cp $TNS_ADMIN/listener.ora $ORACLE_HOME/network/admin/listener.ora
cp $TNS_ADMIN/tnsnames.ora $ORACLE_HOME/network/admin/tnsnames.ora 

GLOBAL TRANSACTION

RAC Databases Distributed Transaction for Middleware
------------------------------------------------------
srvctl add service -d pamdev1 -s pamdevx -preferred csmper-cls09
[oracle@csmper-cls09 ~]$ srvctl config service -d pamdev
[oracle@csmper-cls09 ~]$ srvctl status database -d pamdev
Instance pamdev1 is running on node csmper-cls09

[oracle@csmper-cls09 ~]$ srvctl add service -d pamdev -s pamdevx -r pamdev1
[oracle@csmper-cls09 ~]$ srvctl config service -d pamdev
[oracle@csmper-cls09 ~]$ srvctl modify service -d pamdev -s pamdevx -x TRUE
[oracle@csmper-cls09 ~]$ srvctl config service -d pamdev
Service name: pamdevx
Service is enabled
Server pool: pamdev_pamdevx
Cardinality: 1
Disconnect: false
Service role: PRIMARY
Management policy: AUTOMATIC
DTP transaction: true
AQ HA notifications: false
Failover type: NONE
Failover method: NONE
TAF failover retries: 0
TAF failover delay: 0
Connection Load Balancing Goal: LONG
Runtime Load Balancing Goal: NONE
TAF policy specification: NONE
Edition:
Preferred instances: pamdev1
Available instances:

SQL> alter system set "_clusterwide_global_transactions"=FALSE scope=spfile ;

System altered.

[oracle@csmper-cls09 ~]$ srvctl stop instance -i pamdev1 -d pamdev
[oracle@csmper-cls09 ~]$ srvctl start instance -i pamdev1 -d pamdev

AUTOSTART

2. crsctl status res ora.orcl.db -p|grep -i AUTO_START
if result is "AUTO_START=never", execute step 3. 
if result is restore: it behave in this way "Restores the resource to the same state 
that it was in when the server stopped". so you can still decided to change it to always.

3. crsctl modify res ora.orcl.db -attr "AUTO_START=always"
srvctl modify database -d orcl -y automatic
Because the db which didn’t start there auto_start=restore and where it started automatically there it is set to =1
AUTO_START=never to disable automatic startup
[grid@csmper-cls06 dbs]$ crsctl status res ora.iosbmpa.db -p | grep -i auto_start
AUTO_START=restore

List of the databases start are as below :

Create ASM Pfile

[grid@csmper-cls06 ~]$ . oraenv
ORACLE_SID = [grid] ? +ASM1
The Oracle base for ORACLE_HOME=/opt/oracle/11.2.0.2_grid/grid is /opt/oracle/product
[grid@csmper-cls06 ~]$ sqlplus " / as sysdba "

SQL> show parameter pfile;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      +WSPSP_ARC_01/csmper-clu01/asm
                                                 parameterfile/registry.253.859
                                                 225615

SQL> create pfile='/tmp/init+ASM1.ora' from spfile;

File created.
[grid@csmper-cls06 ~]$ cd $ORACLE_HOME/dbs
[grid@csmper-cls06 dbs]$ ls -lt *ASM1*
-rw-r--r-- 1 grid oinstall 3648 Oct  7 08:55 init+ASM1.ora
-rw-r--r-- 1 grid oinstall 3648 Sep 26 07:43 DND_init+ASM1.ora
-rw-rw---- 1 grid oinstall 1544 Sep 25 20:07 hc_+ASM1.dat
-rw-rw---- 1 grid oinstall 1708 Sep 25 18:42 ab_+ASM1.dat

asmcmd spbackup +DATA/ASM/ASMPARAMETERFILE/REGISTRY.253.822856169 /tmp/ASMspfile.backup

And check out the contents of the file:

$ strings /tmp/ASMspfile.backup
+ASM.__oracle_base='/u01/app/grid'#ORACLE_BASE set from in memory value
+ASM.asm_diskgroups='RECO','ACFS'#Manual Mount
*.asm_power_limit=1
*.large_pool_size=12M
*.remote_login_passwordfile='EXCLUSIVE'

. oraenv
+ASM1
Startup nomount pfile=’ /opt/oracle/11.2.0.2_grid/grid/dbs/ init+ASM1.ora’;
Create spfile ‘+OCR_VOTE’ from pfile;
sqlplus / as sysasm

SQL> shutdown immediate;
ASM instance shutdown

SQL> startup nomount pfile= '/tmp/init+ASM1.ora';

SQL> create spfile= '+OCR_VOTE' from pfile;
File created.
SQL> shutdown immediate;

From other nodes and 
SQL> startup nomount
Show parameter pfile ---- check location

Bounce for Maintenance

[mokum9@csmper-cls08 ~]$ sudo su - oracle

. oraenv
iosbmpa

srvctl stop database –d  iosbmpa
Stop Clusterware
-----------------------
FOR Each NODE: 

Set  Oraenv to -                +ASM1 / +ASM2 / +ASM3

sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl disable has
Wait for 5 minutes

sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl stop has

Start Clusterware
-----------------------
FOR Each NODE: 

Set  Oraenv to -     +ASM1 / +ASM2 / +ASM3

sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl start has

Wait for 5 minutes 

sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl enable has

Check all databases; manually starup which are not coming up yet.

Start GRID agent
+++++++++++++++++
cd /opt/oracle/product/agent12c/core/12.1.0.4.0/bin
[oracle@csmper-cls08 bin]$ ./emctl start agent

BLACKOUT MODE off
++++++++++++++++++
 Find the respective target and make it clear.

Stop database, asm, listener, nodeapps finally crs
+++++++++++++++++++++++++++++++++++++++++++++++++++++++
1. STOP Services/ Instance / database for single node 

srvctl stop service -d <db>
srvctl stop instance -d DB_NAME -i instance_NAME1

2. STOP LISTENER
srvctl stop listener -n csmper-cls18
srvctl stop scan_listener -n csmper-cls18

3. STOP NODEAPPS
srvctl stop nodeapps -n csmper-cls18

4. STOP CRS
crsctl check crs -n csmper-cls18
crsctl stop crs -- (on csmper-cls18)
crsctl stop crs -- (on NODE_NAME1)


sudo <crsctl path>/
sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl stop crs -n csmper-cls18


ps -fu oracle
ps -fu grid

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Start crs, nodeapps, listner, asm and finally database
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++
1. START CRS
crsctl start crs -- (on csmper-cls18)

2. START NODEAPPS
srvctl start nodeapps -n csmper-cls18

3. START LISTENER [remote / local]

srvctl start listener -n csmper-cls18
srvctl start scan_listener -n csmper-cls18

4. Start Instance
srvctl start instance -d DB_NAME -i DB_NAME1

5. Start Services
srvctl start service -d <db>

]]]]


crsd(3786)]CRS-2771:Maximum restart attempts reached for resource 'ora.iccitmq.iccitmq.svc'; will not restart. 
See /opt/oracle/11.2.0.2_grid/grid/log/csmper-cls18/alertcsmper-cls18.log for details.

---stop and satrt database using srvctl


+++++++++++++
[oracle@csmper-cls17 ~]$ srvctl start instance -d smqa -i smqa3
PRCR-1013 : Failed to start resource ora.smqa.db
PRCR-1064 : Failed to start resource ora.smqa.db on node csmper-cls17
CRS-5017: The resource action "ora.PQA26ARC.dg start" encountered the following error:
ORA-15032: not all alterations performed
ORA-15017: diskgroup "PQA26ARC" cannot be mounted
ORA-15063: ASM discovered an insufficient number of disks for diskgroup "PQA26ARC"


CRS-2674: Start of 'ora.PQA26ARC.dg' on 'csmper-cls17' failed
[oracle@csmper-cls17 ~]$ srvctl stop instance -d smqa -i smqa3
PRCC-1017 : smqa was already stopped on csmper-cls17

[oracle@csmper-cls17 ~]$ srvctl config database -d smqa
Database unique name: smqa
Database name: smqa
Oracle home: /app/lims/product/11.2.0.2/db_1
Oracle user: oracle
Spfile: +SMQA_DATA_01/smqa/spfilesmqa.ora
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: smqa
Database instances: smqa1,smqa2,smqa3
Disk Groups: SMQA_DATA_01,SMQA_ARC_01,PQA26ARC
Mount point paths:
Services:
Type: RAC
Database is administrator managed

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
+SMQA_DATA_01/smqa/datafile/system.256.802023217
+SMQA_DATA_01/smqa/datafile/sysaux.257.802023217
+SMQA_DATA_01/smqa/datafile/undotbs1.258.802023217
+SMQA_DATA_01/smqa/datafile/users.259.802023217
+SMQA_DATA_01/smqa/datafile/undotbs2.264.802023315
+SMQA_DATA_01/smqa/datafile/undotbs3.265.802023315
+SMQA_DATA_01/smqa/datafile/sm_gg_data.271.849283675
+SMQA_DATA_01/smqa/datafile/sm_data.272.809540031
+SMQA_DATA_01/smqa/datafile/sm_index.273.809540043

9 rows selected.

SQL> archive log list
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            +SMQA_ARC_01
Oldest online log sequence     6299
Next log sequence to archive   6300
Current log sequence           6300


[oracle@csmper-cls17 ~]$ srvctl modify database -d smqa -a 'SMQA_DATA_01,SMQA_ARC_01'
[oracle@csmper-cls17 ~]$ srvctl config database -d smqa
Database unique name: smqa
Database name: smqa
Oracle home: /app/lims/product/11.2.0.2/db_1
Oracle user: oracle
Spfile: +SMQA_DATA_01/smqa/spfilesmqa.ora
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: smqa
Database instances: smqa1,smqa2,smqa3
Disk Groups: SMQA_DATA_01,SMQA_ARC_01
Mount point paths:
Services:
Type: RAC
Database is administrator managed



$ . oraenv
ORACLE_SID = [pdev20] ? pdev201
The Oracle base has been set to /app/pdev26/product
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl config database -d pdev20
PRCD-1120 : The resource for database pdev20 could not be found.
PRCR-1001 : Resource ora.pdev20.db does not exist
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl add database -d pdev20 -o /app/pdev26/product/11.2.0.2/db_1
PRCS-1007 : Server pool pdev20 already exists
PRCR-1086 : server pool ora.pdev20 is already registered
oracle@csmper-cls18:pdev201:/home/oracle
$ crsctl delete serverpool ora.pdev20 -f
-bash: crsctl: command not found
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl delete serverpool ora.pdev20 -f
Usage: srvctl <command> <object> [<options>]
    commands: enable|disable|start|stop|relocate|status|add|remove|modify|getenv|setenv|unsetenv|config
    objects: database|instance|service|nodeapps|vip|network|asm|diskgroup|listener|srvpool|server|scan|scan_listener|oc4j|home|filesystem|gns|cvu
For detailed help on each command and object and its options use:
  srvctl <command> -h or
  srvctl <command> <object> -h
PRKO-2010 : Invalid command specified on command line: delete
oracle@csmper-cls18:pdev201:/home/oracle
$ crsctl delete srvpool ora.pdev20 -f
-bash: crsctl: command not found
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl remove srvpool ora.pdev20 -f
PRKO-2002 : Invalid command line option: ora.pdev20
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl remove srvpool pdev20 -f
PRKO-2002 : Invalid command line option: pdev20
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl remove srvpool pdev20
PRKO-2002 : Invalid command line option: pdev20
oracle@csmper-cls18:pdev201:/home/oracle

$ sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl delete serverpool pdev20
CRS-2553: Server pool 'pdev20' cannot be unregistered as it does not exist
CRS-4000: Command Delete failed, or completed with errors.
grid@csmper-cls18:+ASM1:/home/grid
$ sudo /opt/oracle/11.2.0.2_grid/grid/bin/crsctl delete serverpool ora.pdev20
grid@csmper-cls18:+ASM1:/home/grid

$ srvctl add database -d pdev20 -o /app/pdev26/product/11.2.0.2/db_1
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl config database -d pdev20
Database unique name: pdev20
Database name:
Oracle home: /app/pdev26/product/11.2.0.2/db_1
Oracle user: oracle
Spfile:
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: pdev20
Database instances:
Disk Groups:
Mount point paths:
Services:
Type: RAC
Database is administrator managed
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl status database -d pdev20
Database is not running.
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl start database -d pdev20
PRKO-3119 : Database pdev20 cannot be started since it has no configured instances.
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl add instance -d pdev20 -i pdev201 -n csmper-cls18
oracle@csmper-cls18:pdev201:/home/oracle
$ srvctl start database -d pdev20
oracle@csmper-cls18:pdev201:/home/oracle


CRS-2765:Resourc
It seems cluster services can’t find voting disk. Possibly asm storage is not accessible or due to currupted voting disks.
https://learnwithme11g.wordpress.com/2013/04/03/troubleshooting-rac-services-startup/

Also check ASM disk status: 
#/etc/init.d/oracleasm listdisks
#/etc/init.d/oracleasm status

 Tried re-enabling the asmlibs as follows on both the nodes …

/etc/init.d/oracleasm restart
/etc/init.d/oracleasm status
/etc/init.d/oracleasm start
/etc/init.d/oracleasm enable
/etc/init.d/oracleasm status
/etc/init.d/oracleasm listdisks


[root@askmdbrac01 ~]# /etc/init.d/oracleasm restart
Dropping Oracle ASMLib disks:                              [  OK  ]
Shutting down the Oracle ASMLib driver:                    [  OK  ]
[root@askmdbrac01 ~]# /etc/init.d/oracleasm status
Checking if ASM is loaded: no
Checking if /dev/oracleasm is mounted: no
[root@askmdbrac01 ~]# /etc/init.d/oracleasm start
[root@askmdbrac01 ~]# /etc/init.d/oracleasm status
Checking if ASM is loaded: no
Checking if /dev/oracleasm is mounted: no
[root@askmdbrac01 ~]# /etc/init.d/oracleasm enable
Writing Oracle ASM library driver configuration: done
Initializing the Oracle ASMLib driver:                     [  OK  ]
Scanning the system for Oracle ASMLib disks:               [  OK  ]
[root@askmdbrac01 ~]# /etc/init.d/oracleasm status
Checking if ASM is loaded: yes
Checking if /dev/oracleasm is mounted: yes
[root@askmdbrac01 ~]# /etc/init.d/oracleasm listdisks
CRSVOL1
DISK1
FRADISK1
[root@askmdbrac01 ~]#[root@askmdbrac02 ~]# /etc/init.d/oracleasm status
Checking if ASM is loaded: yes
Checking if /dev/oracleasm is mounted: yes
[root@askmdbrac02 ~]# /etc/init.d/oracleasm restart
Dropping Oracle ASMLib disks:                              [  OK  ]
Shutting down the Oracle ASMLib driver:                    [  OK  ]
Initializing the Oracle ASMLib driver:                     [  OK  ]
Scanning the system for Oracle ASMLib disks:               [  OK  ]
[root@askmdbrac02 ~]#  /etc/init.d/oracleasm listdisks
CRSVOL1
DISK1
FRADISK1
[root@askmdbrac02 ~]# /etc/init.d/oracleasm status
Checking if ASM is loaded: yes
Checking if /dev/oracleasm is mounted: yes
[root@askmdbrac02 ~]#

Tried to re-configure the asmlib as below …..


+++++++++++++++++++++++++++++++++++++++++++++
Troubleshooting tfa issues 
Check the tfa process on both the nodes
ps -ef | grep tfa

Verify TFA runtime status
# /u01/app/grid/tfa/bin/tfactl print status

Check for errors:
# /u01/app/grid/tfa/bin/tfactl print errors 

Check for database startups:
# /u01/app/grid/tfa/bin/tfactl print startups

Restart TFA
# /etc/init.d/init.tfa restart

Stop TFAMain process and removes related inittab entries
# /etc/init.d/init.tfa shutdown
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

To list hosts
tfactl print hosts

To add the host successfully, try the following steps:
1. Get the cookie in node1 using:
./tfa_home/bin/tfactl print cookie
2. Set the cookie from Step 1 in node2 using:
./tfa_home/bin/tfactl set cookie=
3. After Step 2, add host again:
./tfa_home/bin/tfactl host add node2

Verify TFA runtime status
tfactl print status

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default TFA will try to use ports 5000 to 5005 for secure root communications however if that port range is not
available on your system this can be changed.

The file TFA_HOME/internal/usuableports.txt when installed looks like this.

#cat /opt/oracle/tfa/tfa_home/internal/usableports.txt
5000
5001
5002
5003
5004
5005

To change the ports TFA will try to use you need to:-
1) Stop TFA on all nodes
2) Edit usableports.txt to replace the ports you wish tfa to use.
3) Remove the file tfa_home/internal/port.txt on all nodes.
4) Start TFA on all nodes.

Note: The usableports.txt file must be identical on all nodes.

TFA will also use one of ports 5006 to 5011 on the loopback interface for non root communications however
if that port range is not available on your system then this can also be changed.

The file TFA_HOME/internal/NonRootusuableports.txt when installed looks like this.

#cat /opt/oracle/tfa/tfa_home/internal/usableNonRootports.txt
5006
5007
5008
5009
5010
5011

To change the ports TFA will try to use you need to:-
1) Stop TFA on all nodes
2) Edit usableports.txt to replace the ports you wish tfa to use.
3) Remove the file tfa_home/internal/NonRootports.txt on all nodes.
4) Start TFA on all nodes.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
re-installation with -local will resolve the adding the hosts.
Trace File Analyzer Collector (TFA): collects log and trace files from all nodes and products into a single location. Unfortunately it’s written in Java with its own Java Virtual Machine, again requiring a large memory footprint for the heap etc. It can be removed wit ha single command, though note that next time you run rootcrs.pl (patching for example) it will reinstall itself.

# tfactl uninstall

  



Network/Listener 



Configure Network for oracle RAC installation 

Configure Network for oracle RAC installation 
1)Determine public node names, private node names, and virtual node names for each node in the cluster.
For the public node name, use the primary host name of each node.
In my environment my one node name is racnode-1 and another node name is racnode-2. So I determine the public, private and virtual node name for node racnode-1 as racnode-1, racnode-1-priv and racnode-1-vip accordingly.

Similarly on racnode-2 as racnode-2, racnode-2-priv and racnode-2-vip accordingly.

2)Identify the interface names and associated IP addresses.

On racnode-1,
#/sbin/ifconfig
eth0 Link encap:Ethernet HWaddr 00:16:76:B0:46:7D
inet addr:192.168.1.91 Bcast:192.168.15.255 Mask:255.255.240.0
inet6 addr: fe80::216:76ff:feb0:467d/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:96114 errors:0 dropped:0 overruns:0 frame:0
TX packets:1769 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:7704446 (7.3 MiB) TX bytes:215309 (210.2 KiB)

eth1 Link encap:Ethernet HWaddr 00:11:3B:0A:42:DC
inet addr:192.168.150.30 Bcast:192.168.150.255 Mask:255.255.255.0
inet6 addr: fe80::211:3bff:fe0a:42dc/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:27116 errors:0 dropped:0 overruns:0 frame:0
TX packets:48666 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:7171014 (6.8 MiB) TX bytes:4353054 (4.1 MiB)
Interrupt:58 Base address:0xa000

lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING MTU:16436 Metric:1
RX packets:1201 errors:0 dropped:0 overruns:0 frame:0
TX packets:1201 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:3069384 (2.9 MiB) TX bytes:3069384 (2.9 MiB)

On racnode-2,

[root@racnode-2 ~]# /sbin/ifconfig
eth0 Link encap:Ethernet HWaddr 00:16:76:B2:EB:27
inet addr:192.168.1.92 Bcast:192.168.15.255 Mask:255.255.240.0
inet6 addr: fe80::216:76ff:feb2:eb27/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:104567 errors:0 dropped:0 overruns:0 frame:0
TX packets:1424 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:8546472 (8.1 MiB) TX bytes:197747 (193.1 KiB)
Interrupt:169 Base address:0x6200

eth1 Link encap:Ethernet HWaddr 00:11:95:1F:02:09
inet addr:192.168.150.20 Bcast:192.168.150.255 Mask:255.255.255.0
inet6 addr: fe80::211:95ff:fe1f:209/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:27495 errors:0 dropped:0 overruns:0 frame:0
TX packets:50130 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:6697016 (6.3 MiB) TX bytes:4488642 (4.2 MiB)
Interrupt:177 Base address:0x8100

lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING MTU:16436 Metric:1
RX packets:1203 errors:0 dropped:0 overruns:0 frame:0
TX packets:1203 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:2857788 (2.7 MiB) TX bytes:2857788 (2.7 MiB)

Here I will use network interface eth0 for the public network and eth1 is for the private network. So, on racnode-1 IP address on public network will be 192.168.1.91 and IP address on private network will be 192.168.150.30. Similarly on racnode-2 IP address on public network will be 192.168.1.92 and IP address on private network is 192.168.150.20

3)On each node in the cluster, determine a third IP address that will serve as a virtual IP address which must not use currently in the network and its subnet must be same as on public network. I determine virtual IP address for racnode-1 as 192.168.1.95 and on racnode-2 as 192.168.1.96

4)After completing the network configuration, the IP address and network interface configuration should be as follows,

Node(Hostname) Node Name Type IP Address
racnode-1 racnode-1 public 192.168.1.91
racnode-1 racnode-1-vip virtual 192.168.1.95
racnode-1 racnode-1-priv private 192.168.150.30
racnode-2 racnode-2 public 192.168.1.92
racnode-2 racnode-2-vip virtual 192.168.1.96
racnode-2 racnode-2-priv private 192.168.150.20 


5)In all nodes modify /etc/hosts file so that they contain the host IP addresses, virtual IP addresses, and private network IP addresses from both nodes in the cluster, as follows,

[root@racnode-2 ~]# vi /etc/hosts
# Do not remove the following line, or various programs
# that require network functionality will fail.
127.0.0.1 localhost.localdomain localhost
192.168.1.91 racnode-1
192.168.1.95 racnode-1-vip
192.168.150.30 racnode-1-priv

192.168.1.92 racnode-2
192.168.1.96 racnode-2-vip
192.168.150.20 racnode-2-priv

6)Verify the network configuration by using the ping command to test the connection.
ping -c 3 racnode-2
ping -c 3 racnode-2-vip
ping -c 3 racnode-2-priv

ping -c 3 racnode-1
ping -c 3 racnode-1-vip
ping -c 3 racnode-1-priv

LISTENER
Stop and Start listener Resources

[grid@csmper-cls07 ~]$ crsctl status resource ora.LISTENER_SCAN1.lsnr
NAME=ora.LISTENER_SCAN1.lsnr
TYPE=ora.scan_listener.type
TARGET=ONLINE
STATE=ONLINE on csmper-cls07
[grid@csmper-cls07 admin]$ crsctl status resource ora.LISTENER_SCAN1.lsnr
NAME=ora.LISTENER_SCAN1.lsnr
TYPE=ora.scan_listener.type
TARGET=ONLINE
STATE=ONLINE on csmper-cls07
NOTE: srvctl stop scan_listener -- not worked
[grid@csmper-cls07 admin]$ crsctl stop resource ora.LISTENER_SCAN1.lsnr
CRS-2673: Attempting to stop 'ora.LISTENER_SCAN1.lsnr' on 'csmper-cls07'
CRS-2677: Stop of 'ora.LISTENER_SCAN1.lsnr' on 'csmper-cls07' succeeded
[grid@csmper-cls07 admin]$ crsctl start resource ora.LISTENER_SCAN1.lsnr
CRS-2672: Attempting to start 'ora.LISTENER_SCAN1.lsnr' on 'csmper-cls07'
CRS-2676: Start of 'ora.LISTENER_SCAN1.lsnr' on 'csmper-cls07' succeeded

[grid@csmper-cls17 ~]$ lsnrctl status LISTENER_SCAN2 | grep -i pqa20
Service "pqa20" has 3 instance(s).
  Instance "pqa201", status READY, has 1 handler(s) for this service...
  Instance "pqa202", status READY, has 1 handler(s) for this service...
  Instance "pqa203", status READY, has 1 handler(s) for this service...
Service "pqa20XDB" has 3 instance(s).
  Instance "pqa201", status READY, has 1 handler(s) for this service...
  Instance "pqa202", status READY, has 1 handler(s) for this service...
  Instance "pqa203", status READY, has 1 handler(s) for this service...
Service "pqa20_XPT" has 3 instance(s).
  Instance "pqa201", status READY, has 1 handler(s) for this service...
  Instance "pqa202", status READY, has 1 handler(s) for this service...
  Instance "pqa203", status READY, has 1 handler(s) for this service...
[grid@csmper-cls17 ~]$ lsnrctl status LISTENER_SCAN2 | grep -i uptime
Uptime                    0 days 2 hr. 19 min. 13 sec

lsnrctl status LISTENER_SCAN1 | grep -i pprod26

After server reboting one node didn't get up services for all the instances: 
alter system set remote_listener = '';
alter system set remote_listener = 'csmper-scan01.apac.ent.bhpbilliton.net:1521';
alter system register;

SQL>  show parameter listener

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
listener_networks                    string
local_listener                       string      (DESCRIPTION=(ADDRESS_LIST=(AD
                                                 DRESS=(PROTOCOL=TCP)(HOST=10.2
                                                 41.26.32)(PORT=1521))))
remote_listener                      string      csmper-scan03.apac.ent.bhpbill
                                                 iton.net:1521

If REMOTE_LISTENER parameter not set or set incorrectly.
Should be: csmper-scan01.apac.ent.bhpbilliton.net:1521
Use command: alter system set remote_listener = 'csmper-scan01.apac.ent.bhpbilliton.net:1521' scope = spfile sid = '*';

Verifying the Interconnect
list of IP addresses registered with the Oracle database kernel
SELECT addr,
indx,
inst_id,
pub_ksxpia,
picked_ksxpia,
name_ksxpia,
ip_ksxpia
FROM x$ksxpia;

ADDR                   INDX    INST_ID PUB_KSXPIA        PICKED_KSXPIA
---------------------------------------------------------------------------------------------------------
NAME_KSXPIA                                   IP_KSXPIA
--------------------------------------------- ------------------------------------------------
00002B09DC939EC0          0          3 N                    GPnP
bond2:1                                       169.254.56.140

00002B09DC939EC0          1          3 Y                      GPnP
bond0                                         10.241.26.28

bond0 has the value Y in column Public (PUB_KSXPIA), indicating it’s the public interface; and
bond2 is the private interface (identified by the value N). If the correct IP addresses are not visible, it is an indication of
incorrect installation and configuration of the RAC environment.
The private interconnect is no longer recorded in the OCR file; instead, Oracle keeps the information locally in
“GPnP” (Grid Plug and Play) profiles.

The preceding output illustrates there are two interfaces currently configured on the node: bond0 and bond2. Both
of these interfaces have used NIC bonding for high availability.

SELECT inst_id,
name,
ip_address,
is_public
FROM gv$cluster_interconnects
ORDER BY inst_id;

   INST_ID NAME                                          IP_ADDRESS                                       IS_PUBLIC
---------- --------------------------------------------- ------------------------------------------------ ---------
         1 bond2:1                                       169.254.10.88                                    NO
         2 bond2:1                                       169.254.17.141                                   NO
         3 bond2:1                                       169.254.56.140                                   NO

grid@csmper-cls17:+ASM3:/home/grid
$ /sbin/ifconfig -a

the lost blocks for just one day of up time
SELECT
A1.INST_ID ,
A3.INSTANCE_NAME,
A3.STARTUP_TIME,
ROUND (SYSDATE - STARTUP_TIME) "Days",
A1.VALUE ,
A2.VALUE
FROM GV$SYSSTAT A1,
GV$SYSSTAT A2,
GV$INSTANCE A3
WHERE A1.NAME = 'gc blocks lost'
AND A2.NAME = 'gc blocks corrupt'
AND A1.INST_ID = A2.INST_ID
AND A1.INST_ID = A3.INSTANCE_NUMBER
ORDER BY 1;

Issue: 

grid@iorsdb02-adm::/u01/app/grid/diag/tnslsnr/iorsdb02-adm/listener_scan2/trace
$ zgrep 12514 listener_scan2.log.20151112-0700.gz | grep PMQ2
grid@iorsdb02-adm::/u01/app/grid/diag/tnslsnr/iorsdb02-adm/listener_scan2/trace
$ zgrep 12514 listener_scan2.log.20151112-0700.gz | grep PRAH
grid@iorsdb02-adm::/u01/app/grid/diag/tnslsnr/iorsdb02-adm/listener_scan2/trace
$ zgrep 12514 listener_scan2.log.20151112-0700.gz | grep PICS
  



Performance 



Cluster latancy in RAC database;

select b1.inst_id, b2.value "RECEIVED",
b1.value "RECEIVE TIME",
((b1.value / b2.value) * 10) "AVG RECEIVE TIME (ms)"
from gv$sysstat b1, gv$sysstat b2
where b1.name = 'gc cr block receive time' and
b2.name = 'gc cr blocks received' and b1.inst_id = b2.inst_id;

Oracle cluster interconnects
To verify the interconnect settings, query the (G)V$CLUSTER_INTERCONNECTS and the (G)V$CONFIGURED_INTERCONNECTS. For example, in order to verify interconnect
settings with V$CLUSTER_INTERCONNECTS, issue the following SQL statement:

SQL> SELECT * FROM V$CLUSTER_INTERCONNECTS;
NAME IP_ADDRESS IS_ _ SOURCE
--------------- ---------------- --- -------------------------------
Eth0 10.10.10.34 NO Oracle Cluster Repository
To verify the interconnect settings with V$CONFIGURED_INTERCONNECTS, issue the
SQL statement, as specified in the following example:
SQL> SELECT * FROM V$CONFIGURED_INTERCONNECTS;

We can benefit from the following RAC-specific ADDM findings:
 Interconnect latencies
 LMS congestions
 Top SQL
 Instance contention
 Hot blocks
 Hot objects
Using a top-down approach for performance analysis can be helpful. We can start
with ADDM analysis and then continue with AWR detail statistics and historical
data. Active Session History will provide you with session-specific data that can
also be very useful for the performance tuning of Oracle RAC databases due to
the following:
 I/O issues at log switch time imply either checkpoint occurrence or log
archiver being slow
 Process stuck waiting for I/O
 Connection storm
Active Session History (ASH) information is available through the following:
 Dumping to trace file:
SQL>oradebug setmypid
Statement Processed.
SQL>oradebug dump ashdump 10
Statement Processed.
Or 
SQL>alter session set events 'immediate trace name ashdump level 10';

 Querying the V$ACTIVE_SESSION_HISTORY view
 Querying the DBA_HIST_ACTIVE_SESS_HISTORY data dictionary table
 Running the ashrpt.sql from the $ORACLE_HOME/rdbms/admin directory
 Using Oracle Enterprise Manager

AWR Reports
=================
In this section, we describe the AWR and show how to generate reports from its contents. AWR
snapshots of the current values of various database statistics are generated by the MMON background
process using non-SQL kernel calls. AWR reporting tools allow you to generate a report in text or HTML
format using either EM or SQL*Plus to compare the delta between two snapshots. This can help you
determine the database workload during the intervening period.

create the AWR manually using the script,
$ORACLE_HOME/rdbms/admin/catawr.sql
AWR snapshots are performed if the STATISTICS_LEVEL parameter is set to the default value of
TYPICAL or to ALL

You can check the current snapshot interval and retention time with this snippet:
SQL> SELECT snap_interval, retention FROM dba_hist_wr_control;
You can modify these values using the MODIFY_SNAPSHOT_SETTINGS procedure in
DBMS_WORKLOAD_REPOSITORY. For example, you can use this line to change the snapshot interval to 30
minutes and set the retention period to two weeks:
SQL> EXECUTE dbms_workload_repository.modify_snapshot_settings -
> (interval => 30, retention => 20160);
Both the interval and retention parameters should be specified in minutes. If you specify an interval
of 0, Oracle will set the snapshot interval to 1 year; if you specify a retention period of 0, Oracle will set
the retention period to 100 years.
You can take a snapshot manually at any time for the particular instance you are connected to using
the CREATE_SNAPSHOT procedure at the SQL*Plus prompt, as in this example:
SQL> EXECUTE dbms_workload_repository.create_snapshot;
This procedure can accept a single parameter specifying the flush level for the snapshot, which is
either TYPICAL or ALL. You can also create snapshots from within EM; you can access this functionality by
navigating from the Administration page to the Workload Repository page, and then to the Snapshots
page.
When you have a stable workload, you can take a baseline, which records the difference between
two snapshots. First, identify a suitable pair of snapshot IDs using the DBA_HIST_SNAPSHOT view:
SQL> SELECT snap_id, instance_number, startup_time,
> begin_interval_time,end_interval_time
> FROM dba_hist_snapshot;

Both snapshots must belong to the same instance. They must also have the same instance start-up
time. For example, assuming that the start and end snapshots for the period of interest are 234 and 236,
you can generate a baseline:
SQL> EXECUTE dbms_workload_repository.create_baseline -
( -
 start_snap_id => 234, -
 end_snap_id => 236, -
 baseline_name => 'Morning Peak' -
 );
You can check the current baselines in the DBA_HIST_BASELINE view. Unlike automatic snapshots,
which will be purged at the end of the retention period, baselines will not be removed automatically.
Baselines can be removed manually using the DROP_BASELINE procedure:
SQL> EXECUTE dbms_workload_repository.drop_baseline -
 ( -
 baseline_name => 'Morning Peak' -
 );
You can extract reports from the AWR tables with the script, $ORACLE_HOME/rdbms/admin/awrrpt.sql.
As shown in the following code example, this script asks whether you want the report output to be
created in HTML or text format. It then asks you to specify the number of days of snapshots that you
wish to view. Based on your response, it displays a list of the snapshots taken during this period. The
script prompts you for the start and end snapshots for the report, as well as for a file name to write the
report to:
SQL> @?/rdbms/admin/awrrpt.

Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks) 
  Issue #2: High log file sync waits 
  Issue #3: High Waits on Mutexes 
  Issue #4: High gc buffer busy, enq: TX -row lock contention, enq: TX - index contention, enq: TX - ITL allocate entry waits 
  Issue #5: High CPU and memory consumption 


Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks)
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Symptoms

I. AWR reports show high number of lost blocks.

II. netstat -s reports increasing number of packet reassambly failure, dropped packets.
ex: `netstat -s` IP stat counters: 
     3104582 fragments dropped after timeout 
     34550600 reassemblies required 
     8961342 packets reassembled ok 
     3104582 packet reassembles failed.

In Oracle RAC environments, RDBMS gathers global cache work load statistics which are reported in STATSPACK, AWRs and GRID CONTROL. Global cache lost blocks statistics ("gc cr block lost" and/or "gc current block lost") for each node in the cluster as well as aggregate statistics for the cluster represent a problem or inefficiencies in packet processing for the interconnect traffic. These statistics should be monitored and evaluated regularly to guarantee efficient interconnect Global Cache and Enqueue Service (GCS/GES) and cluster processing. Any block loss indicates a problem in network packet processing and should be investigated.

RDBMS global cache lost blocks can be directly related to faulty or mis-configured interconnects.

  Issue #2: High log file sync waits 
++++++++++++++++++++++++++++++++++
Symptoms

I. AWR reports show that log file sync is one of the TOP 5 waits consistently.
II. Average log file sync is high ( > 20 ms).
III. Average log file parallel write is high ( > 10 ms).
III. Average redo write broadcast ack time or wait for scn ack is high ( > 10 ms).
IV. Average log file sync is low, but there are too many log file sync waits.

Background

When a user session COMMITs or ROLLBACKs, the session's redo information needs to be flushed by LGWR to the redo logfile. The user session waits on 'log file sync' while waiting for LGWR to post it back to confirm that all redo changes are safely on disk.

Example:

WAIT #0: nam='log file sync' ela= 977744 buffer#=754 p2=0 p3=0 obj#=114927 tim=1212384670107387

Parameters:

  P1 = buffer#
  P2 = Not used
  P3 = Not used
  obj# = object_id

All changes up to this buffer# (in the redo log buffer) must be flushed to disk and the writes confirmed to ensure that the transaction is committed, and will remain committed upon an instance crash.

A typical life cycle of 'log file sync' wait

1. A user sessions issues a commit or rollback and starts waiting on log file sync.
2. LGWR gather redo information to be written to redo log files, issues IO and queues BOC to an LMS process and posts LMS process.
3. LGWR waits for redo buffers to be flushed to disk and SCN to be ACK'd
4. Completion of IO and receiving ACK from LMS signal write complete. LGWR then posts foreground process to continue.
5. Foreground process wakes up and log file sync wait ends.

Script to Collect Log File Sync Diagnostic Information (lfsdiag.sql)  collects useful information for troubleshooting log file sync issues.

sqlplus '/ as sysdba'
SQL> @lfsdiag.sql

Possible Causes

I. Poor IO service time and throughput of the devices where redo logs reside.
II. Oracle bugs. Please review WAITEVENT: "log file sync" Reference (Document 34592.1) for known oracle bugs.
III. LMS not running in RT class.
IV. Scheduling delays with LGWR process.
V. Excessive number of commits.
VI. OS resource starvation.

Solutions

I. Move redo log files to disks with higher throughput and better response time if log file parallel write is consistently high ( > 10 ms). log file parallel write should ideally be within 1-2 ms range
II. Apply fixes for the known oracle bugs, which are applicable to your environment. The most effective way to get those fixes in is to apply the latest PSU patches. Document 756671.1 has more information on the latest PSUs.
III. Ensure that LMS processes are running in RT class. LMS processes run in RT class by default.
IV. Reduce number of commits by using batch commits instead of commiting after every DML operation.
V. See if any activity can safely be done with NOLOGGING / UNRECOVERABLE options.
VI. See if COMMIT NOWAIT option can be used. 

  



lfsdiag.sql 



Script to Collect Log File Sync Diagnostic Information (lfsdiag.sql) [ID 1064487.1]

Script
Paste the script below into a file called lfsdiag.sql and give it execute permissions:

-- NAME: LFSDIAG.SQL
-- ------------------------------------------------------------------------
-- AUTHOR: Michael Polaski - Oracle Support Services
-- ------------------------------------------------------------------------
-- PURPOSE:
-- This script is intended to provide a user friendly guide to troubleshoot
-- log file sync waits. The script will look at important parameters involved
-- in log file sync waits, log file sync wait histogram data, and the script
-- will look at the worst average log file sync times in the active session
-- history data and AWR data and dump information to help determine why those
-- times were the highest. The script will create a file called
-- lfsdiag_<timestamp>.out in your local directory.

set echo off
set feedback off
column timecol new_value timestamp
column spool_extension new_value suffix
select to_char(sysdate,'Mondd_hh24mi') timecol,
'.out' spool_extension from sys.dual;
column output new_value dbname
select value || '_' output
from v$parameter where name = 'db_name';
spool lfsdiag_&&dbname&&timestamp&&suffix
set trim on
set trims on
set lines 140
set pages 100
set verify off
alter session set optimizer_features_enable = '10.2.0.4';

PROMPT LFSDIAG DATA FOR &&dbname&&timestamp
PROMPT Note: All timings are in milliseconds (1000 milliseconds = 1 second)

PROMPT
PROMPT IMPORTANT PARAMETERS RELATING TO LOG FILE SYNC WAITS:
column name format a40 wra
column value format a40 wra
select inst_id, name, value from gv$parameter
where ((value is not null and name like '%log_archive%') or
name like '%commit%' or name like '%event=%' or name like '%lgwr%')
and name not in (select name from gv$parameter where (name like '%log_archive_dest_state%'
and value = 'enable') or name = 'log_archive_format')
order by 1,2,3;

PROMPT
PROMPT ASH THRESHOLD...
PROMPT
PROMPT This will be the threshold in milliseconds for average log file sync
PROMPT times. This will be used for the next queries to look for the worst
PROMPT 'log file sync' minutes. Any minutes that have an average log file
PROMPT sync time greater than the threshold will be analyzed further.
column threshold_in_ms new_value threshold format 999999999.999
select min(threshold_in_ms) threshold_in_ms
from (select inst_id, to_char(sample_time,'Mondd_hh24mi') minute,
avg(time_waited)/1000 threshold_in_ms
from gv$active_session_history
where event = 'log file sync'
group by inst_id,to_char(sample_time,'Mondd_hh24mi')
order by 3 desc)
where rownum <= 10;

PROMPT
PROMPT ASH WORST MINUTES FOR LOG FILE SYNC WAITS:
PROMPT
PROMPT APPROACH: These are the minutes where the avg log file sync time
PROMPT was the highest (in milliseconds).
column minute format a12 tru
column event format a30 tru
column program format a40 tru
column total_wait_time format 999999999999.999
column avg_time_waited format 999999999999.999
select to_char(sample_time,'Mondd_hh24mi') minute, inst_id, event,
sum(time_waited)/1000 TOTAL_WAIT_TIME , count(*) WAITS,
avg(time_waited)/1000 AVG_TIME_WAITED
from gv$active_session_history
where event = 'log file sync'
group by to_char(sample_time,'Mondd_hh24mi'), inst_id, event
having avg(time_waited)/1000 > &&threshold
order by 1,2;

PROMPT
PROMPT ASH LFS BACKGROUND PROCESS WAITS DURING WORST MINUTES:
PROMPT
PROMPT APPROACH: What is LGWR doing when 'log file sync' waits
PROMPT are happening? LMS info may be relevent for broadcast
PROMPT on commit and LNS data may be relevant for dataguard.
PROMPT If more details are needed see the ASH DETAILS FOR WORST
PROMPT MINUTES section at the bottom of the report.
column inst format 999
column minute format a12 tru
column event format a30 tru
column program format a40 wra
select to_char(sample_time,'Mondd_hh24mi') minute, inst_id inst, program, event,
sum(time_waited)/1000 TOTAL_WAIT_TIME , count(*) WAITS,
avg(time_waited)/1000 AVG_TIME_WAITED
from gv$active_session_history
where to_char(sample_time,'Mondd_hh24mi') in (select to_char(sample_time,'Mondd_hh24mi')
from gv$active_session_history
where event = 'log file sync'
group by to_char(sample_time,'Mondd_hh24mi'), inst_id
having avg(time_waited)/1000 > &&threshold and sum(time_waited)/1000 > 1)
and (program like '%LGWR%' or program like '%LMS%' or
program like '%LNS%' or event = 'log file sync') 
group by to_char(sample_time,'Mondd_hh24mi'), inst_id, program, event
order by 1,2,3,5 desc, 4;

PROMPT
PROMPT HISTOGRAM DATA FOR LFS AND OTHER RELATED WAITS:
PROMPT
PROMPT APPROACH: Look at the wait distribution for log file sync waits
PROMPT by looking at "wait_time_milli". Look at the high wait times then
PROMPT see if you can correlate those with other related wait events.
column event format a40 wra
select inst_id, event, wait_time_milli, wait_count
from gv$event_histogram
where event in ('log file sync','gcs log flush sync',
'log file parallel write','wait for scn ack',
'log file switch completion','gc cr grant 2-way',
'gc buffer busy','gc current block 2-way') or
event like '%LGWR%' or event like '%LNS%'
order by 2 desc,1,3;

PROMPT
PROMPT ORDERED BY WAIT_TIME_MILLI
select inst_id, event, wait_time_milli, wait_count
from gv$event_histogram
where event in ('log file sync','gcs log flush sync',
'log file parallel write','wait for scn ack',
'log file switch completion','gc cr grant 2-way',
'gc buffer busy','gc current block 2-way')
or event like '%LGWR%' or event like '%LNS%'
order by 3,1,2 desc;

PROMPT
PROMPT REDO WRITE STATS
PROMPT
PROMPT "redo write time" in centiseconds (100 per second)
PROMPT 11.1: "redo write broadcast ack time" in centiseconds (100 per second)
PROMPT 11.2: "redo write broadcast ack time" in microseconds (1000 per millisecond)
column value format 99999999999999999999
column milliseconds format 99999999999999.999
select v.version, ss.inst_id, ss.name, ss.value,
decode(substr(version,1,4),
'11.1',decode (name,'redo write time',value*10,
'redo write broadcast ack time',value*10),
'11.2',decode (name,'redo write time',value*10,
'redo write broadcast ack time',value/1000),
decode (name,'redo write time',value*10)) milliseconds
from gv$sysstat ss, v$instance v
where name like 'redo write%' and value > 0
order by 1,2,3;

PROMPT
PROMPT AWR WORST AVG LOG FILE SYNC SNAPS:
PROMPT
PROMPT APPROACH: These are the AWR snaps where the average 'log file sync'
PROMPT times were the highest.
column begin format a12 tru
column end format a12 tru
column name format a13 tru
select dhs.snap_id, dhs.instance_number inst, to_char(dhs.begin_interval_time,'Mondd_hh24mi') BEGIN,
to_char(dhs.end_interval_time,'Mondd_hh24mi') END,
en.name, se.time_waited_micro/1000 total_wait_time, se.total_waits,
se.time_waited_micro/1000 / se.total_waits avg_time_waited
from dba_hist_snapshot dhs, wrh$_system_event se, v$event_name en
where (dhs.snap_id = se.snap_id and dhs.instance_number = se.instance_number)
and se.event_id = en.event_id and en.name = 'log file sync' and
dhs.snap_id in (select snap_id from (
select se.snap_id, se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and en.name = 'log file sync'
order by avg_time_waited desc)
where rownum < 4)
order by 1,2;

PROMPT
PROMPT AWR REDO WRITE STATS
PROMPT
PROMPT "redo write time" in centiseconds (100 per second)
PROMPT 11.1: "redo write broadcast ack time" in centiseconds (100 per second)
PROMPT 11.2: "redo write broadcast ack time" in microseconds (1000 per millisecond)
column stat_name format a30 tru
select v.version, ss.snap_id, ss.instance_number inst, sn.stat_name, ss.value,
decode(substr(version,1,4),
'11.1',decode (stat_name,'redo write time',value*10,
'redo write broadcast ack time',value*10),
'11.2',decode (stat_name,'redo write time',value*10,
'redo write broadcast ack time',value/1000),
decode (stat_name,'redo write time',value*10)) milliseconds
from wrh$_sysstat ss, wrh$_stat_name sn, v$instance v
where ss.stat_id = sn.stat_id
and sn.stat_name like 'redo write%' and ss.value > 0
and ss.snap_id in (select snap_id from (
select se.snap_id, se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and en.name = 'log file sync'
order by avg_time_waited desc)
where rownum < 4)
order by 1,2,3;

PROMPT
PROMPT AWR LFS AND OTHER RELATED WAITS FOR WORST LFS AWRs:
PROMPT
PROMPT APPROACH: These are the AWR snaps where the average 'log file sync'
PROMPT times were the highest. Look at related waits at those times.
column name format a40 tru
select se.snap_id, se.instance_number inst, en.name,
se.total_waits, se.time_waited_micro/1000 total_wait_time,
se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and (en.name in
('log file sync','gcs log flush sync',
'log file parallel write','wait for scn ack',
'log file switch completion','gc cr grant 2-way',
'gc buffer busy','gc current block 2-way')
or en.name like '%LGWR%' or en.name like '%LNS%')
and se.snap_id in (select snap_id from (
select se.snap_id, se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and en.name = 'log file sync'
order by avg_time_waited desc)
where rownum < 4)
order by 1, 6 desc;

PROMPT
PROMPT AWR HISTOGRAM DATA FOR LFS AND OTHER RELATED WAITS FOR WORST LFS AWRs:
PROMPT Note: This query won't work on 10.2 - ORA-942
PROMPT
PROMPT APPROACH: Look at the wait distribution for log file sync waits
PROMPT by looking at "wait_time_milli". Look at the high wait times then
PROMPT see if you can correlate those with other related wait events.
select eh.snap_id, eh.instance_number inst, en.name, eh.wait_time_milli, eh.wait_count
from wrh$_event_histogram eh, v$event_name en
where eh.event_id = en.event_id and
(en.name in ('log file sync','gcs log flush sync',
'log file parallel write','wait for scn ack',
'log file switch completion','gc cr grant 2-way',
'gc buffer busy','gc current block 2-way')
or en.name like '%LGWR%' or en.name like '%LNS%')
and snap_id in (select snap_id from (
select se.snap_id, se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and en.name = 'log file sync'
order by avg_time_waited desc)
where rownum < 4)
order by 1,3 desc,2,4;

PROMPT
PROMPT ORDERED BY WAIT_TIME_MILLI
PROMPT Note: This query won't work on 10.2 - ORA-942
select eh.snap_id, eh.instance_number inst, en.name, eh.wait_time_milli, eh.wait_count
from wrh$_event_histogram eh, v$event_name en
where eh.event_id = en.event_id and
(en.name in ('log file sync','gcs log flush sync',
'log file parallel write','wait for scn ack',
'log file switch completion','gc cr grant 2-way',
'gc buffer busy','gc current block 2-way')
or en.name like '%LGWR%' or en.name like '%LNS%')
and snap_id in (select snap_id from (
select se.snap_id, se.time_waited_micro/1000 / se.total_waits avg_time_waited
from wrh$_system_event se, v$event_name en
where se.event_id = en.event_id and en.name = 'log file sync'
order by avg_time_waited desc)
where rownum < 4)
order by 1,4,2,3 desc;

PROMPT
PROMPT ASH DETAILS FOR WORST MINUTES:
PROMPT
PROMPT APPROACH: If you cannot determine the problem from the data
PROMPT above, you may need to look at the details of what each session
PROMPT is doing during each 'bad' snap. Most likely you will want to
PROMPT note the times of the high log file sync waits, look at what
PROMPT LGWR is doing at those times, and go from there...
column program format a45 wra
column sample_time format a25 tru
column event format a30 tru
column time_waited format 999999.999
column p1 format a40 tru
column p2 format a40 tru
column p3 format a40 tru
select sample_time, inst_id inst, session_id, program, event, time_waited/1000 TIME_WAITED,
p1text||': '||p1 p1,p2text||': '||p2 p2,p3text||': '||p3 p3
from gv$active_session_history
where to_char(sample_time,'Mondd_hh24mi') in (select
to_char(sample_time,'Mondd_hh24mi')
from gv$active_session_history
where event = 'log file sync'
group by to_char(sample_time,'Mondd_hh24mi'), inst_id
having avg(time_waited)/1000 > &&threshold)
and time_waited > 0.5
order by 1,2,3,4,5;

select to_char(sysdate,'Mondd hh24:mi:ss') TIME from dual;

spool off

PROMPT
PROMPT OUTPUT FILE IS: lfsdiag_&&dbname&&timestamp&&suffix
PROMPT

  



Performance issue 



In this Document

 Purpose 
 Scope 
 Details 
  Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks) 
  Issue #2: High log file sync waits 
  Issue #3: High Waits on Mutexes 
  Issue #4: High gc buffer busy, enq: TX -row lock contention, enq: TX - index contention, enq: TX - ITL allocate entry waits 
  Issue #5: High CPU and memory consumption 
 References 
--------------------------------------------------------------------------------
Applies to: 
Oracle Database - Enterprise Edition - Version 10.1.0.2 to 11.2.0.3 [Release 10.1 to 11.2]
Information in this document applies to any platform.

Purpose
The purpose of this document is to provide a summary of  top database and/or instance performance issues and the possible solutions in a RAC environment. 

Please note that Issue #3 (High Waits on Mutexes) and Issue #5 (High CPU and Memory Consumption) are generic database issues and not RAC-specific. However, they have been included in this document because they are one of the TOP issues, and can occur locally on an instance in a RAC environment.

Scope
DBAs

Details
Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks)
Symptoms

I. AWR reports show high number of lost blocks.

II. netstat -s reports increasing number of packet reassambly failure, dropped packets.

Solutions

Use the following document to troubleshoot and resolve lost block issue. The document describes symptoms, possible causes and solutions.

Document 563566.1 - gc block lost diagnostics 

Issue #2: High log file sync waits
Symptoms

I. AWR reports show that log file sync is one of the TOP 5 waits consistently.

II. Average log file sync is high ( > 20 ms).

III. Average log file parallel write is high ( > 10 ms).

III. Average redo write broadcast ack time or wait for scn ack is high ( > 10 ms).

IV. Average log file sync is low, but there are too many log file sync waits.

Background

When a user session COMMITs or ROLLBACKs, the session's redo information needs to be flushed by LGWR to the redo logfile. The user session waits on 'log file sync' while waiting for LGWR to post it back to confirm that all redo changes are safely on disk.

Example:

WAIT #0: nam='log file sync' ela= 977744 buffer#=754 p2=0 p3=0 obj#=114927 tim=1212384670107387

Parameters:

  P1 = buffer#
  P2 = Not used
  P3 = Not used
  obj# = object_id

All changes up to this buffer# (in the redo log buffer) must be flushed to disk and the writes confirmed to ensure that the transaction is committed, and will remain committed upon an instance crash.

A typical life cycle of 'log file sync' wait

1. A user sessions issues a commit or rollback and starts waiting on log file sync.
2. LGWR gather redo information to be written to redo log files, issues IO and queues BOC to an LMS process and posts LMS process.
3. LGWR waits for redo buffers to be flushed to disk and SCN to be ACK'd
4. Completion of IO and receiving ACK from LMS signal write complete. LGWR then posts foreground process to continue.
5. Foreground process wakes up and log file sync wait ends.

Important log file sync related statistics and events

redo write time - Total elapsed time of the write from the redo log buffer to the current redo log file in 10s of milliseconds.
redo writes - Total number of writes by LGWR to the redo log files. "redo blocks written" divided by this statistic equals the number of blocks per write.
log file parallel write - Time it takes for the I/Os to complete. Even though redo records are written in parallel, the parallel write is not complete until the last I/O is on disk.
redo write broadcast ack time - Total amount of the latency associated with broadcast on commit beyond the latency of the log write (in microseconds).
user commits - Number of user commits. When a user commits a transaction, the redo generated that reflects the changes made to database blocks must be written to disk. Commits often represent the closest thing to a user transaction rate.
user rollbacks - Number of times users manually issue the ROLLBACK statement or an error occurs during a user's transactions.
The script provided in Document 1064487.1 - Script to Collect Log File Sync Diagnostic Information (lfsdiag.sql)  collects useful information for troubleshooting log file sync issues.

Possible Causes

I. Poor IO service time and throughput of the devices where redo logs reside.

II. Oracle bugs. Please review WAITEVENT: "log file sync" Reference (Document 34592.1) for known oracle bugs.

III. LMS not running in RT class.

IV. Scheduling delays with LGWR process.

V. Excessive number of commits.

VI. OS resource starvation.

Solutions

I. Move redo log files to disks with higher throughput and better response time if log file parallel write is consistently high ( > 10 ms). log file parallel write should ideally be within 1-2 ms range

II. Apply fixes for the known oracle bugs, which are applicable to your environment. The most effective way to get those fixes in is to apply the latest PSU patches. Document 756671.1 has more information on the latest PSUs.

III. Ensure that LMS processes are running in RT class. LMS processes run in RT class by default.

IV. Reduce number of commits by using batch commits instead of commiting after every DML operation.

V. See if any activity can safely be done with NOLOGGING / UNRECOVERABLE options.

VI. See if COMMIT NOWAIT option can be used. Please refer to Document 857576.1 for more information.          @For Support Only: Renice LGWR to run at higher priority or run LGWR in RT class by adding LGWR to the parameter: _high_priority_processes='VKTM|LMS|LGWR". Consider doing this only if log file sync is high and scheduling delay of LGWR is found to be causing it. Be prepared to test it thoroughly.Issue #3: High Waits on Mutexes
Mutexes are a lighter-weight and more granular concurrency mechanism than latches.The reason for obtaining a mutex is to ensure that certain operations are properly managed for concurrency. e.g., if one session is changing a data structure in memory, then other session must wait to acquire the mutex before it can make a similar change. The following are most common mutex related waits:

A. cursor: pin S wait on X
B. cursor: pin S
C. library cache: Mutex X

Symptoms (A)

AWR reports show cursor: pin S wait on X as one of the top wait.

Background (A) 

A session may wait for this event when it is trying to get a mutex pin in Share mode but another session is holding the mutex pin on the same cursor object in exclusive. Frequently, waits for 'Cursor: pin S wait on X' is a symptom and not the cause. There may be underlying tuning requirements or known issues. 

Possible Causes (A)

Please review Troubleshooting 'cursor: pin S wait on X' waits (Document 1349387.1).

Solutions (A)

Please review Troubleshooting 'cursor: pin S wait on X' waits (Document 1349387.1).
Symptoms (B)

AWR reports show cursor: pin S as one of the top waits

Background (B)

A session waits for "cursor: pin S" when it wants a specific mutex in S (share) mode on a specific cursor and there is no concurrent X holder but it could not acquire that mutex immediately. This may seem a little strange as one might question why there should be any form of wait to get a mutex which has no session holding it in an incompatible mode. The reason for the wait is that in order to acquire the mutex in S mode (or release it) the session has to increment (or decrement) the mutex reference count and this requires an exclusive atomic update to the mutex structure itself. If there are concurrent sessions trying to make such an update to the mutex then only one session can actually increment (or decrement) the reference count at a time. A wait on "cursor: pin S" thus occurs if a session cannot make that atomic change immediately due to other concurrent requests.
Mutexes are local to the current instance in RAC environments.  

Parameters:
   P1 = idn 
   P2 = value
   P3 = where (where|sleeps in 10.2) 

 idn is the mutex identifier value which matches to the HASH_VALUE of the SQL statement that we are waiting to get the mutex on. The SQL can usually be found using the IDN value in a query of the form:

SELECT sql_id, sql_text, version_count 
FROM V$SQLAREA where HASH_VALUE=&IDN;

If the SQL_TEXT shown for the cursor is of the form "table_x_x_x_x" then this is a special internal cursor - see Document 1298471.1 for information about mapping such cursors to objects.

P1RAW is the same value in hexadecimal and it can be used to search in tracefiles for SQL matching to that hash value.

Possible Causes (B)

I. Heavy concurrency for a specific mutex, especially on systems with multiple CPUs.

II. Waits for very many different "idn" values when under load.

III. Oracle Bugs
      Bug 10270888 - ORA-600[kgxEndExamine-Bad-State] / mutex waits after a self deadlock
      Bug 9591812 - Wrong wait events in 11.2 ("cursor: mutex S" instead of "cursor: mutex X")
      Bug 9499302 - Improve concurrent mutex request handling
      Bug 7441165 - Prevent preemption while holding a mutex (fix only works on Solaris)
      Bug 8575528 - Missing entries in V$MUTEX_SLEEP.location

Solutions (B)

I. Apply fix for Bug 10411618.

II. For any identified "hot" SQLs, one can reduce the concurrency on specific cursors by replacing the one SQL with some variants which are executed by different sessions. Please review WAITEVENT: cursor: pin S Reference (Document 1310764.1) for further details.

III. Apply fixes for other known oracle bugs. The most effective way to get the fixes in is to apply the latest PSU patches. Document 756671.1 has more information on recommended patches.
Symptoms (C)

AWR reports show library cache: Mutex X as one of the TOP waits.

Background (C)

The library cache mutex is acquired for similar purposes that the library cache latches were acquired in prior versions of Oracle. In 10g, mutexes were introduced for certain operations in the library cache. Starting with 11g, the library cache latches were replaced by mutexes. This wait event is present whenever a library cache mutex is held in exclusive mode by a session and other sessions need to wait for it to be released.

Individual Waits:
Parameters:
  P1 = "idn" = Unique Mutex Identifier 
  P2 = "value" = Session ID holding the mutex
  P3 = "where" = location in code (internal identifier) where mutex is being waited for 

Systemwide Waits:
At a systemwide level, there are two views which will help diagnose this wait:

GV$MUTEX_SLEEP (or V$MUTEX_SLEEPS for non-RAC)
and GV$MUTEX_SLEEP_HISTORY (or V$MUTEX_SLEEP_HISTORY for non-RAC)

These views track the instance wide usage of mutexes since instance startup. Since these views show values that are total values since startup, they are most meaningful when you obtain the difference in values during a short time interval when there was problem. The easiest way to see this information is through an AWR or statspack report in the "Mutex Sleep Summary" section.

Possible Causes (C)

I. Frequent Hard Parses.

II. High Version Counts.

III. Invalidations and reloads.

IV. Oracle Bugs. Please review WAITEVENT: "library cache: mutex X" (Document 727400.1) for the
      list of known oracle bugs.

Solutions (C)


I. Reduce hard parsing, invalidations and reloads. Please review Troubleshooting: Tuning the Shared Pool and Tuning Library Cache Latch Contention (Document 62143.1) for more information.

II. Review  Troubleshooting: High Version Count Issues (Document 296377.1) to   resolve high version count issue. 

III. Apply fixes for the known oracle bugs. The most effective way to get the fixes in is to apply the latest PSU patches. Document 756671.1 has more information on recommended patches.Issue #4: High gc buffer busy, enq: TX -row lock contention, enq: TX - index contention, enq: TX - ITL allocate entry waits
A. enq: TX - Index Contention
B. enq: TX - ITL allocate entry waits
C. enq: TX - row lock contention

Symptoms (A)

I. AWR reports show high wait on enq: TX - index contention and enq: TX - row lock contention in mode 4.

II. AWR reports show high values for branch node splits, leaf node splits and leaf node 90-10 splits

III. AWR reports (Segments by Row Lock Waits) show high row lock waits for a particular segment

IV. AWR reports show other waits such as gc buffer busy waits on index branch or Leaf blocks, gc current block busy on Remote Undo Headers and gc current split.

Example:

Top 5 Timed Foreground Events

Event Waits Time(s) Avg wait (ms) % DB time Wait Class
enq: TX - index contention 29,870 1,238 41 9.52 Concurrency

Instance Activity Stats:

Statistic Total per Second per Trans
branch node splits 945 0.26 0.00
leaf node 90-10 splits 1,670 0.46 0.00
leaf node splits 35,603 9.85 0.05

Segments by Row Lock Waits:

Owner Tablespace Object Name Obj.Type Row Lock Waits % of Capture
ACSSPROD ACSS_IDX03 ACSS_ORDER_HEADER_PK INDEX 3,425 43.62
ACSSPROD ACSS_IDX03 ACSS_ORDER_HEADER_ST INDEX 883 11.25
ACSSPROD ACSS_IDX03 ACSS_ORDER_HEADER_DT INDEX 682 8.69

Background (A)

When a transaction inserting a row in an index has to wait for the end of an index block split being done by another transaction the session would wait on event enq: TX - index contention.

Possible causes (A)

I. High number of concurrent inserts leading to excessive index block splits.

II. Right-growing index with key generated from a sequence.

Solutions (A)

Solution is to reorganize the index in a way to avoid the contention or hot spots. The options are

I. Recreate the index as reverse key index

II. Hash partition the index

III. If index key is generated from a sequence, increase cache size of the sequence and make the sequence 'no order' if application supports it.
Symptom (B)

AWR reports show high wait on enq: TX - allocate ITL entry and enq: TX - row lock contention in mode 4.

Background (B)

A session waits on enq: TX - allocate ITL entry when it wants to lock a row in the block but one or more other sessions have rows locked in the same block, and there is no free ITL slot in the block. Usually, Oracle Database dynamically adds another ITL slot. This may not be possible if there is insufficient free space in the block to add an ITL.

Possible Causes (B)

Low initrans setting relative to the number of concurrent transactions.

Solutions (B)

Find the segments those have high ITL waits from AWR reports or using the following SQL:

SELECT OWNER, OBJECT_NAME, OBJECT_TYPE
  FROM V$SEGMENT_STATISTICS 
 WHERE STATISTIC_NAME = 'ITL waits' AND VALUE > 0
ORDER BY VALUE;

Increase initrans value of the segments, which are seeing high ITL waits.
Symptom (C)

AWR reports show high wait on enq: TX - row lock contention in Exclusive mode (6).

Background (C)

A session waits on enq: TX - row lock contention, when it waits for a row level lock that is held by another session. This happens when one user has updated or deleted a row, but has not commited or rolled back yet, and another session wants to update or delete the same row.

Solution (C)

This is an application issue and application developer needs to be engaged to look into the SQL statements involved. The following documents might be helpful in drilling down further:

Document 102925.1 - Tracing sessions: waiting on an enqueue
Document 179582.1 - How to Find TX Enqueue Contention in RAC or OPS
Document 1020008.6 - SCRIPT: FULLY DECODED LOCKING
Document 62354.1 - TX Transaction locks - Example wait scenarios 
Document 224305.1 -Autonomous Transaction can cause lockingIssue #5: High CPU and memory consumption
A. High CPU Utilization
B. High Memory Utilization

Symptoms (A)

I. OS tools such as TOP, prstat, vmstat shows user CPU usage is very high, and top cpu consuming
   processes are either oracle shadow or background processes.

II. AWR reports show top waits are one or more of the following:
    latch free
    cursor pin S wait on X or cursor pin S wait or library cache mutex X
    latch: cache buffers chains
    resmgr: cpu quantum
    enq: RO - fast object reuse
    DFS lock handle

III. AWR reports (SQLs ordered by buffer gets) show SQLs with very high buffer gets per execution 
      and cpu time.

IV. Process stack of the high cpu consuming process shows that the process is spinning.

Possible Causes (A)

I. Oracle bugs: 
  Bug 12431716 - Mutex waits may cause higher CPU usage in 11.2.0.2.2 PSU / GI PSU
  Bug 8199533 - NUMA enabled by default can cause high CPU
  Bug 9226905 - Excessive CPU usage / OERI:kkfdPaPrm from Parallel Query / High Version 
  count on PX_MISMATCH
  Bug 7385253 - Slow Truncate / DBWR uses high CPU / CKPT blocks on RO enqueue
  Bug 10326338 - High "resmgr:cpu quantum" but CPU has idle time
  Bug 6455161 - Higher CPU / Higher "cache buffer chains" latch gets / Higher "consistent gets" 
  after truncate/Rebuild

II. Very large SGA on Linux x86-64 platform without the implementation of Hugepages.

III. Expensive SQL queries with sub-optimal execution plans.

IV. Runaway processes.

Solutions (A)

I. Apply fix for the bug(s) that you are encountering. Most effective way to get all those fixes in is to apply the latest PSU patches. Document 756671.1 has more information on the latest PSUs.

II. Implement hugepages. Please refer to Document 361670.1 -  Slow Performance with High CPU Usage on 64-bit Linux with Large SGA for further explanation.

III. Tune the SQLs, which are incurring excessive buffer gets and cpu time. Please refer to Document 404991.1 - How to Tune Queries with High CPU Usage for more info.
Symptoms (B)

I. OS tools such as ps, pmap show process is growing in memory. pmap shows the growth is in heap and/or stack area of the process. For example,

#pmap -x 26677

Address        Kbytes   RSS   Anon   Locked Mode   Mapped File
00010000    496        480     -             - r-x--             bash
0009A000   80          80       24          - rwx--             bash
000AE000  160        160     40           - rwx--             [ heap ]
FF100000   688        688     -             - r-x--              libc.so.1

II. Session uga and/or pga memory of a oracle shadow process is growing.
        
        select se.sid,n.name,max(se.value) maxmem
           from v$sesstat se, 
                    v$statname n
        where n.statistic# = se.statistic#
            and n.name in ('session pga memory','session pga memory max',
                                    'session uga memory','session uga memory max')
         group by n.name,se.sid
         order by 3;

III. Number of open cursors for a session/process is growing monotonically.

Possible Causes (B)

I. Oracle bugs:
      Bug 9919654 - High resource / memory use optimizing SQL with UNION/set functions with   
      many branches
      Bug 10042937 HIGH MEMORY GROUP IN GES_CACHE_RESS AND ORA-4031 ERRORS
      Bug 7429070 BIG PGA MEM ALLOC DURING PARSE TIME - KXS-HEAP-C
      Bug 8031768 - ORA-04031 SHARED POOL "KKJ JOBQ WOR"
      Bug 10220046 - OCI memory leak using non-blocking connection for fetches
      Bug 6356566 - Memory leak / high CPU selecting from V$SQL_PLAN
      Bug 7199645 - Long parse time / high memory use from query rewrite
      Bug 10636231 - High version count for INSERT .. RETURNING statements with 
      reason INST_DRTLD_MISMATCH
      Bug 9412660 - PLSQL cursor leak / ORA-600[kglLockOwnersListDelete]
      Bug 6051972 - Memory leak if connection to database is frequently opened and closed
      Bug 4690571 - Memory leak using BULK COLLECT within cursor opened using native 
      dynamic sql

II. Cursor leaks caused by application not closing cursors explicitly.

III.  SQLs with abnormally large hash join and/or sort operation.


Possible Solutions (B)

I. Apply fix for the applicable bug(s). The most effective way to get those fixes in is to apply the latest PSU patches. Document 756671.1 has more information on the latest PSUs.

II. Ensure cursors are explicitly closed by application.

III. Avoid very large hash join and/or sort operation.

  



Top5Issues 



Top 5 Database and/or Instance Performance Issues in RAC Environment (Doc ID 1373500.1)

 Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks) 
  Issue #2: High log file sync waits 
  Issue #3: High Waits on Mutexes 
  Issue #4: High gc buffer busy, enq: TX -row lock contention, enq: TX - index contention, enq: TX - ITL allocate entry waits 
  Issue #5: High CPU and memory consumption
  
Please note that Issue #3 (High Waits on Mutexes) and Issue #5 (High CPU and Memory Consumption) are generic database issues and not RAC-specific. However, they have been included in this document because they are one of the TOP issues, and can occur locally on an instance in a RAC environment.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Issue #1: High counts of lost blocks(gc lost blocks, gc current/cr lost blocks)
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Symptoms

I. AWR reports show high number of lost blocks.
II. netstat -s reports increasing number of packet reassambly failure, dropped packets.

Solutions : Document 563566.1 - gc block lost diagnostics


In Oracle RAC environments, RDBMS gathers global cache work load statistics which are reported in STATSPACK, AWRs and GRID CONTROL. Global cache lost blocks statistics ("gc cr block lost" and/or "gc current block lost") for each node in the cluster as well as aggregate statistics for the cluster represent a problem or inefficiencies in packet processing for the interconnect traffic. These statistics should be monitored and evaluated regularly to guarantee efficient interconnect Global Cache and Enqueue Service (GCS/GES) and cluster processing. Any block loss indicates a problem in network packet processing and should be investigated.
The vast majority of escalations attributed to RDBMS global cache lost blocks can be directly related to faulty or mis-configured interconnects.

Cause
In most cases, gc block lost has been attributed to (a) A missing OS patch (b) Bad network card (c) Bad cable (d) Bad switch (e) One of the network settings. Customer should open a ticket with their OS vendor and provide them the OSwatcher output.

Solution
Customers should start collecting OSwatcher data (see Note 301137.1).
This data should be shared with OS vendor to find root cause of the problem. 

+++++++++++++++++++++++++++++++
Issue #2: High log file sync waits
+++++++++++++++++++++++++++++++
Symptoms

I. AWR reports show that log file sync is one of the TOP 5 waits consistently.
II. Average log file sync is high ( > 20 ms).
III. Average log file parallel write is high ( > 10 ms).
III. Average redo write broadcast ack time or wait for scn ack is high ( > 10 ms).
IV. Average log file sync is low, but there are too many log file sync waits.

A typical life cycle of 'log file sync' wait

1. A user sessions issues a commit or rollback and starts waiting on log file sync.
2. LGWR gather redo information to be written to redo log files, issues IO and queues BOC to an LMS process and posts LMS process.
3. LGWR waits for redo buffers to be flushed to disk and SCN to be ACK'd
4. Completion of IO and receiving ACK from LMS signal write complete. LGWR then posts foreground process to continue.
5. Foreground process wakes up and log file sync wait ends.

Important log file sync related statistics and events

redo write time - Total elapsed time of the write from the redo log buffer to the current redo log file in 10s of milliseconds.
redo writes - Total number of writes by LGWR to the redo log files. "redo blocks written" divided by this statistic equals the number of blocks per write.
log file parallel write - Time it takes for the I/Os to complete. Even though redo records are written in parallel, the parallel write is not complete until the last I/O is on disk.
redo write broadcast ack time - Total amount of the latency associated with broadcast on commit beyond the latency of the log write (in microseconds).
user commits - Number of user commits. When a user commits a transaction, the redo generated that reflects the changes made to database blocks must be written to disk. Commits often represent the closest thing to a user transaction rate.
user rollbacks - Number of times users manually issue the ROLLBACK statement or an error occurs during a user's transactions.
The script provided in Document 1064487.1 - Script to Collect Log File Sync Diagnostic Information (lfsdiag.sql)  collects useful information for troubleshooting log file sync issues.

Possible Causes

I. Poor IO service time and throughput of the devices where redo logs reside.
II. Oracle bugs. Please review WAITEVENT: "log file sync" Reference (Document 34592.1) for known oracle bugs.
III. LMS not running in RT class.
IV. Scheduling delays with LGWR process.
V. Excessive number of commits.
VI. OS resource starvation.

Solutions

I. Move redo log files to disks with higher throughput and better response time if log file parallel write is consistently high ( > 10 ms). log file parallel write should ideally be within 1-2 ms range
II. Apply fixes for the known oracle bugs, which are applicable to your environment. The most effective way to get those fixes in is to apply the latest PSU patches. Document 756671.1 has more information on the latest PSUs.
III. Ensure that LMS processes are running in RT class. LMS processes run in RT class by default.
IV. Reduce number of commits by using batch commits instead of commiting after every DML operation.
V. See if any activity can safely be done with NOLOGGING / UNRECOVERABLE options.
VI. See if COMMIT NOWAIT option can be used. Please refer to Document 857576.1 for more information. 
++++++++++++++++++++++++++++++++
Issue #3: High Waits on Mutexes
++++++++++++++++++++++++++++++++
Mutexes are a lighter-weight and more granular concurrency mechanism than latches.The reason for obtaining a mutex is to ensure that certain operations are properly managed for concurrency. e.g., if one session is changing a data structure in memory, then other session must wait to acquire the mutex before it can make a similar change. The following are most common mutex related waits:

A. cursor: pin S wait on X  -->Please review Troubleshooting 'cursor: pin S wait on X' waits (Document 1349387.1).
B. cursor: pin S
Solns:
I. Apply fix for Bug 10411618.
II. For any identified "hot" SQLs, one can reduce the concurrency on specific cursors by replacing the one SQL with some variants which are executed by different sessions. Please review WAITEVENT: cursor: pin S Reference (Document 1310764.1) for further details.
III. Apply fixes for other known oracle bugs. The most effective way to get the fixes in is to apply the latest PSU patches. Document 756671.1 has more information on recommended patches.

C. library cache: Mutex X

Solutions (C)

I. Reduce hard parsing, invalidations and reloads. Please review Troubleshooting: Tuning the Shared Pool and Tuning Library Cache Latch Contention (Document 62143.1) for more information.
II. Review  Troubleshooting: High Version Count Issues (Document 296377.1) to   resolve high version count issue. 
III. Apply fixes for the known oracle bugs. The most effective way to get the fixes in is to apply the latest PSU patches. Document 756671.1 has more information on recommended patches.

Issue #4: High gc buffer busy, enq: TX -row lock contention, enq: TX - index contention, enq: TX - ITL allocate entry waits
A. enq: TX - Index Contention
B. enq: TX - ITL allocate entry waits
C. enq: TX - row lock contention

A.  Solution is to reorganize the index in a way to avoid the contention or hot spots. The options are

I. Recreate the index as reverse key index
II. Hash partition the index
III. If index key is generated from a sequence, increase cache size of the sequence and make the sequence 'no order' if application supports it.
B. Solutions (B)

Find the segments those have high ITL waits from AWR reports or using the following SQL:

SELECT OWNER, OBJECT_NAME, OBJECT_TYPE
  FROM V$SEGMENT_STATISTICS 
 WHERE STATISTIC_NAME = 'ITL waits' AND VALUE > 0
ORDER BY VALUE;

Increase initrans value of the segments, which are seeing high ITL waits.

Issue #5: High CPU and memory consumption
A. High CPU Utilization
B. High Memory Utilization

  



RAC-One Node 



RAC ONE NODE
--------------------------
RAC One Node claims to be a multiple instances of RAC running on a single node in a cluster, 
and has a fast "instance relocation" feature in cases of catastrophic server failure.

 RAC One Node uses "instance relocation", and when an instance fails,
 RAC One Node re-starts a failed instance on another node, 
by re-mounting the disk on the new server and using the pfile/spfile to re-start the instance.

While RAC One node will not protect you in a case of server failure (unlike regular RAC, running nodes on many servers), 
RAC One Node does offer instance duplication within a single server environment

In sum, Oracle RAC One Node is a solution aimed at smaller shops and systems that need fast failover, 
not not necessarily 100% continuous availability.


As Oracle user invoke dbca
 --> Advanced mode
  --> Oracle ONE Node database 
   --> Admin-Managed 
    --> Select cluster Nodes : node1,node2 

Resources status and database status 

$ srvctl status database -d DICP1_01
Instance DICP1_1 is running on node iorsdb01-adm
Online relocation: INACTIVE


$ srvctl config database -d DICP1_01
Database unique name: DICP1_01
Database name: DICP1
Oracle home: /u01/app/oracle/product/11.2.0.4/db_000
Oracle user: oracle
Spfile: +DATA/DICP1_01/spfileDICP1.ora
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: DICP1_01
Database instances:
Disk Groups: DATA,RECO
Mount point paths:
Services: SRV0DICP101,SRVDICP1RMN
Type: RACOneNode
Online relocation timeout: 30
Instance name prefix: DICP1
Candidate servers: iorsdb01-adm,iorsdb02-adm
Database is administrator managed


Relocate RAC One Node database
++++++++++++++++++++++++++++++++
srvctl relocate database -d DICP1_01 -v -n iorsdb02-adm
NOTE: instance name is different


$ srvctl status database -d DRAH2_01
Instance DRAH2_2 is running on node iorsdb01-adm
Online relocation: INACTIVE

SO FOR SAME INSTANCE NAME:

$ srvctl stop database -d DRAH2_01
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl start database -d DRAH2_01 -n iorsdb01-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DRAH2_01
Instance DRAH2_1 is running on node iorsdb01-adm
Online relocation: INACTIVE


Scaling up RAC ONE NODE to Standard RAC
-- convert for RAC
srvctl convert database -d DICP1_01 -c RAC
-- check status
srvctl config database -d DICP1_01

--
Type: RAC
Database instances: DICP1_1

srvctl status database -d DICP1_01
-- add and start second instance 
srvctl add  instance -d DICP1_01 -i DICP1_2 -n iorsdb02-adm
srvctl start instance -d DICP1_01 -i DICP1_2
-- check status
srvctl status database -d DICP1_01


Scaling Down to RAC One Node

--Checck the service first
srvctl config  database  -d DMTR3_01

--Stop and remove service
srvctl stop service -d DMTR3_01 -s SRV0DMTR301
srvctl remove service -d DMTR3_01 -s SRV0DMTR301

--stop and remove second instance
srvctl stop instance -d DMTR3_01 -i "DMTR32" -o immediate;
srvctl remove instance -d DMTR3_01 -i DMTR32
[Y]

-- add service and convert to one node
srvctl add  service -d DMTR3_01 -s SRV0DMTR301 -r DMTR31
srvctl convert database -d DMTR3_01 -c RACONENODE -i DMTR31

-- Verify db status 
srvctl status database -d DMTR3_01


-- issues --
srvctl remove instance -d DICP1_01 -i DICP1_2

[Y]
PRKO-3147 : Instance <> cannot be removed because it is the only preferred instance for service(s) 
<> for database rac1node
srvctl remove service -d DICP1_01 -s SRV0DICP101
srvctl remove instance -d DICP1_01 -i DICP1_2

[Y]

srvctl convert database -d DICP1_01 -c RACONENODE -i DICP1_1

PRCD-1242 : Unable to convert RAC database rac1node to RAC One Node database because the database had no service added
srvctl add  service -d DICP1_01 -s SRV0DICP101 -r DICP1_1
srvctl convert database -d DICP1_01 -c RACONENODE -i DICP1_1


srvctl relocate database -d DPBH1_01 -n iorsdb01-adm
srvctl relocate database -d DPII1_01 -n iorsdb01-adm



$ srvctl status database -d DRAH2_01
Instance DRAH2_2 is running on node iorsdb01-adm
Online relocation: INACTIVE

srvctl relocate database -d DRAH2_01 -n iorsdb02-adm


$ srvctl stop database -d DRAH2_01
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl start database -d DRAH2_01 -n iorsdb01-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DRAH2_01
Instance DRAH2_1 is running on node iorsdb01-adm
Online relocation: INACTIVE


$ srvctl relocate database -d DRAH1_01 -n iorsdb02-adm
$ srvctl status database -d DRAH1_01
Instance DRAH1_1 is running on node iorsdb02-adm
Online relocation: ACTIVE
Source instance: DRAH1_2 on iorsdb01-adm
Destination instance: DRAH1_1 on iorsdb02-adm

Check again:

oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DRAH1_01
Instance DRAH1_1 is running on node iorsdb02-adm
Online relocation: INACTIVE

$ srvctl start database -d DPBH1_01 -n iorsdb01-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DPII1_01
Instance DPII1_2 is running on node iorsdb01-adm
Online relocation: ACTIVE
Source instance: DPII1_2 on iorsdb01-adm
Destination instance: DPII1_1 on iorsdb02-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DPII1_01
Instance DPII1_1 is running on node iorsdb01-adm
Instance DPII1_2 is running on node iorsdb01-adm
Online relocation: ACTIVE
Source instance: DPII1_2 on iorsdb01-adm
Destination instance: DPII1_1 on iorsdb02-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DPII1_01
Instance DPII1_1 is running on node iorsdb02-adm
Online relocation: ACTIVE
Source instance: DPII1_2 on iorsdb01-adm
Destination instance: DPII1_1 on iorsdb02-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DPII1_01
Instance DPII1_1 is running on node iorsdb02-adm
Online relocation: INACTIVE
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl stop database -d DPII1_01
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl start database -d DPII1_01 -n iorsdb01-adm
oracle@iorsdb02-adm:DRAH2_1:/home/oracle
$ srvctl status database -d DPII1_01
Instance DPII1_1 is running on node iorsdb01-adm
Online relocation: INACTIVE  



RAC-password file 



After a database is created via dbca in 12c Oracle Restart environment, the database password file location is empty in OCR as shown below

[oracle@node bin]$ ./srvctl config database -d DEV20
Database unique name: DEV20
Database name: DEV20
Oracle home: /u01/app/oracle/product/12.1.0/dbhome_1
Oracle user: oracle
Spfile: +DATA/DEV20/spfileDEV20.ora
Password file:
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Database instance: DEV20
Disk Groups: DATA
Services:
Not like a RAC database, the password file is created in ASM diskgroup (shared among all cluster nodes) and the password file location is stored in OCR when a database is created via dbca.

Cause
For single instance Database(Oracle Restart environment), DBCA still creates the password file in $ORACLE_HOME(DB Home)/dbs directory same as pre 12.1 version.

If one would like to move the password file to an ASM diskgroup and store the location in OCR, the below procedure can be used.

Solution
1. To store the location of the password file in OCR in Oracle Restart environment, use srvctl modify database command:

$ srvctl modify database -d DEV20 -pwfile /u01/app/oracle/product/12.1.0/dbhome_1/dbs/orapwDEV20

$ srvctl config database -d DEV20
Database unique name: DEV20
Database name: DEV20
Oracle home: /u01/app/oracle/product/12.1.0/dbhome_1
Oracle user: oracle
Spfile: +DATA/DEV20/spfileDEV20.ora
Password file: /u01/app/oracle/product/12.1.0/dbhome_1/dbs/orapwDEV20
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Database instance: DEV20
Disk Groups: DATA
Services:

2. To create a password file in an ASM diskgroup in Oracle Restart environment, use orapwd command:

$ orapwd FILE='+DATA/DEV20/orapwDEV20' ENTRIES=10 DBUNIQUENAME='DEV20' FORMAT=12 

The syntax of the ORAPWD command is as follows:

orapwd FILE=filename [ENTRIES=numusers] [FORCE={y|n}] [ASM={y|n}] 
[DBUNIQUENAME=dbname] [FORMAT={12|legacy}] [SYSBACKUP={y|n}] [SYSDG={y|n}] 
[SYSKM={y|n}] [DELETE={y|n}] [INPUT_FILE=input-fname]

orapwd DESCRIBE FILE=filename

To store this password file location in OCR, run the below command.

$ srvctl modify database -d DEV20 -pwfile +DATA/DEV20/orapwDEV20

  



RACStandby 



Create RAC Standby Database for RAC Primary Database
Primary and Standby databases are of 2 node RAC.

Primary Database : TESTP
---------------------------------
DBNAME : TESTP
DB UNIQUE NAME : TESTP
Instances :	 TESTP1 on rac1
		TESTP2 on rac2

Standby Database : TESTS
------------------------------------
DBNAME : TESTP
DB UNIQUE NAME : TESTS
Instances : 	TESTS1 on drrac1
		TESTS2 on drrac2

Database version: Oracle 11.2.0.1
Below are the steps to create a RAC standby database for a RAC  Primary database.

Step 1: Add the following standby parameters on the Primary database.
log_archive_dest_1='location=use_db_recovery_file_dest valid_for=(all_logfiles,all_roles) db_unique_name=TESTP'
log_archive_dest_2='service=TESTS valid_for=(online_logfiles,Primary_role) db_unique_name=TESTS'
fal_client='TESTP' #Oracle net service name of Primary database
fal_server='TESTS' #oracle net service name of standby database.

Step 2: Create a pfile of the Primary database and copy this file to the standby server.
The contents of the pfile of Primary database is as below. (This is the pfile taken from instance TESTP1 of Primary DB.)

[oracle@rac1 u02]$ cat initTESTP1.ora
TESTP1.__db_cache_size=385875968
TESTP2.__db_cache_size=436207616
TESTP1.__java_pool_size=16777216
TESTP2.__java_pool_size=16777216
TESTP1.__large_pool_size=16777216
TESTP2.__large_pool_size=16777216
TESTP1.__oracle_base='/u01/app/oracle'#ORACLE_BASE set from environment
TESTP1.__pga_aggregate_target=520093696
TESTP2.__pga_aggregate_target=520093696
TESTP1.__sga_target=754974720
TESTP2.__sga_target=754974720
TESTP1.__shared_io_pool_size=0
TESTP2.__shared_io_pool_size=0
TESTP1.__shared_pool_size=318767104
TESTP2.__shared_pool_size=268435456
TESTP1.__streams_pool_size=0
TESTP2.__streams_pool_size=0
*.audit_file_dest='/u01/app/oracle/admin/TESTP/adump'
*.audit_trail='db'
*.cluster_database=true
*.compatible='11.2.0.0.0'
*.control_files='+DATA/TESTP/controlfile/current.260.826037247'
*.db_block_size=8192
*.db_create_file_dest='+DATA'
*.db_domain=''
*.db_name='TESTP'
*.db_unique_name='TESTP'
*.db_recovery_file_dest_size=2147483648
*.db_recovery_file_dest='+FRA'
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=TESTPXDB)'
*.fal_client='TESTP'
*.fal_server='TESTS'
TESTP1.instance_number=1
TESTP2.instance_number=2
TESTP1.local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=rac1-vip)(PORT=1521))))'
TESTP2.local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=rac2-vip)(PORT=1521))))'
*.log_archive_dest_1='location=use_db_recovery_file_dest
valid_for=(all_logfiles,all_roles) db_unique_name=TESTP'
*.log_archive_dest_2='service=TESTS
valid_for=(online_logfiles,Primary_role) db_unique_name=TESTS'
*.memory_target=1264582656
*.open_cursors=300
*.processes=150
*.remote_listener='node-scan:1521'
*.remote_login_passwordfile='exclusive'
TESTP2.thread=2
TESTP1.thread=1
TESTP2.undo_tablespace='UNDOTBS2'
TESTP1.undo_tablespace='UNDOTBS1'

Step 3. Listener File contents on Primary Database server.

[oracle@rac1 admin]$ cat listener.ora

LISTENER=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER)))) # line added by Agent
LISTENER_SCAN1=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER_SCAN1)))) # line added by Agent
ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER_SCAN1=ON # line added by Agent
ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER=ON # line added by Agent
 
SID_LIST_LISTENER=
(SID_LIST=
 (SID_DESC=
   (ORACLE_HOME=/u01/app/oracle/product/11.2.0/db1)
   (SID_NAME=TESTP1)
 )
)

Step 4: TNS entries on Primary database server.
### PRIMARY ENTRIES ###
TESTP =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = node-scan)(PORT = 1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTP)
  )
)
TESTP1 =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = rac1-vip)(PORT=1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTP)
    (INSTANCE_NAME = TESTP1)
  )
)
 
TESTP2 =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = rac2-vip)(PORT=1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTP)
    (INSTANCE_NAME = TESTP2)
  )
)

### Standby TNS ENTRIES ###
TESTS =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = drnode-scan)(PORT = 1521)  )
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTS)
    (UR = A)
  )
)
 
TESTS2 =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = drrac2-vip)(PORT = 1521)  )
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTS)
    (UR = A)
    (INSTANCE_NAME = TESTS2)
  )
)
 
TESTS1 =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = drrac1-vip)(PORT = 1521)  )
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = TESTS)
    (UR = A)
    (INSTANCE_NAME = TESTS1)
  )
)
Note: (UR=A) is needed to connect to a BLOCKED unmounted database instance.
Step 5: Copy the above TNS entries to standby database server node drrac1.

Step 6: Copy the password file of any instance ("orapwTESTP1? or "orapwTESTP2?) of  Primary database located at "$ORACLE_HOME/dbs" location to the standby database server node drrac1 location "$ORACLE_HOME/dbs" and rename it to "orapwTESTS1?.

Step 7: On the standby database server, perpare the pfile for TESTS1 instance as initTESTS1.ora .
Can create from primary instance and remove all entries for TESTP2 and replace TESTP1 to TESTS1 and modified cluster parameters [cluster_database, undo tablespace, instance number, thread number]

Contents of initTESTS1.ora File:
[oracle@drrac1 dbs]$ cat initTESTS1.ora
TESTS1.__db_cache_size=419430400
TESTS1.__java_pool_size=16777216
TESTS1.__large_pool_size=16777216
TESTS1.__oracle_base='/u01/app/oracle'#ORACLE_BASE set from environment
TESTS1.__pga_aggregate_target=520093696
TESTS1.__sga_target=754974720
TESTS1.__shared_io_pool_size=0
TESTS1.__shared_pool_size=285212672
TESTS1.__streams_pool_size=0
*.audit_file_dest='/u01/app/oracle/admin/TESTS/adump'
*.audit_trail='db'
*.cluster_database=false
*.compatible='11.2.0.0.0'
*.control_files='+DATA','+FRA'
*.db_block_size=8192
*.db_create_file_dest='+DATA'
*.db_name='TESTP'
*.db_unique_name='TESTS'
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=TESTSXDB)'
*.log_archive_dest_1='location=use_db_recovery_file_dest valid_for=(all_logfiles,all_roles) db_unique_name=TESTS'
*.log_archive_dest_2='service=TESTP valid_for=(online_logfiles,Primary_role) db_unique_name=TESTP'
*.db_recovery_file_dest='+FRA'
*.db_recovery_file_dest_size=2G
*.memory_target=1264582656
*.open_cursors=300
*.processes=150
*.remote_listener='drnode-scan:1521'
*.remote_login_passwordfile='EXCLUSIVE'
TESTS1.undo_tablespace='UNDOTBS1'
TESTS1.local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=drrac1-vip)(PORT=1521))))'
TESTS1.fal_client='TESTS1'
*.fal_server='TESTP'


Step 8: Set up the listener.ora file on the standby database server drrac1.
Contents of Listener.ora file on DRRAC1:
[oracle@drrac1 admin]$ cat listener.ora
LISTENER=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER)))) # line added by Agent
LISTENER_SCAN1=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER_SCAN1)))) # line added by Agent
ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER_SCAN1=ON # line added by Agent
ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER=ON # line added by Agent
 
SID_LIST_LISTENER=
(SID_LIST=
 (SID_DESC=
  (ORACLE_HOME=/u01/app/oracle/product/11.2.0.1/db1)
  (SID_NAME=TESTS1)
 )
)
Step 9: Nomount the standby instance TESTS1 using the above pfile "initTESTS1.ora" on drrac1 node.
SQL> startup nomount pfile='?/dbs/initTESTS1.ora';

Step 10: Now connect to the Primary database as target database and standby database as auxiliary instance through RMAN.  Make sure that the Primary database is open.

[oracle@drrac1 ~]$ rman target sys/oracle@TESTP auxiliary sys/oracle@TESTS1
 
RMAN> duplicate target database for standby from active database;
 
Step 11: Once the duplication is completed, close the RMAN prompt and connect to the standby database through SQL.

sqlplus sys/<password>@TESTS1 as sysdba
Check the status of the standby database by making sure it is in mount stage.
sql>select status,instance_name,database_role from v$instance,v$database;

Step 12: Now start the managed recovery process on the standby database.
sql>recover managed standby database disconnect from session;

Step 13: Now check if the managed recovery process (MRP) has been started on the standby database or not.

SQL> select process,status,sequence#,thread# from v$managed_standby; ---Check MRP process status

Step 14: On the Primary database, perform a few log switches and check if the logs are applied to the standby database.

Primary Database Archive Sequence:
sqlplus sys/<password>@TESTP1 as sysdba
 SQL> select thread#,max(sequence#) from v$archived_log group by thread#;

Standby database archive sequence being applied: 
sqlplus sys/<password>@TESTS1 as sysdba
 SQL> select thread#,max(sequence#) from v$archived_log applied='YES' group by thread#;

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Now add the 2nd instance TESTS2 to the Standby Database TESTS.
Step 15: Create a pfile from the standby instance TESTS1 to add the cluster parameters.

cluster_database=TRUE
TESTS1.undo_tablespace='UNDOTBS1'
TESTS2.undo_tablespace='UNDOTBS2'
TESTS2.local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=drrac2-vip)(PORT=1521))))'
TESTS1.instance_number=1
TESTS2.instance_number=2
TESTS1.thread=1
TESTS2.thread=2
TESTS1.fal_client='TESTS1'
TESTS2.fal_client='TESTS2'

Step 16: Shutdown the instance TESTS1, mount it using the newly above pfile, create an spfile to be placed in the shared location (ASM diskgroup, as it is being shared by both the instances TESTS1 and TESTS2.)
SQL> create spfile='+DATA/TESTS/spfileTESTS.ora' from pfile;

Step 17: Create a new pfile initTESTS1.ora in drrac1 located at $ORACLE_HOME/dbs with just one entry to point to the spfile location.
[oracle@drrac1 dbs]$ cat initTESTS1.ora
SPFILE='+DATA/TESTS/spfileTESTS.ora'

Copy the password file of TESTS1 (orapwTESTS1) to drrac2 location "$ORACLE_HOME/dbs" and rename it as orapwTESTS2.

Step 18: Copy the newly created pfile (initTESTS1.ora) fto drrac2 location "$ORACLE_HOME/dbs" and rename it as initTESTS2.ora

Step 19: Mount both TESTS1 and TESTS2 instances.

Step 20: Start MRP on any one instance using the below query.
SQL> alter database recover managed standby database disconnect from session;

Step 21: Check the max archive sequence generated on Primary and compare it with the max archive sequence applied on the standby.
Primary:
SQL> select thread#,max(sequence#) from v$archived_log group by thread#;
Standby:
SQL> select thread#,max(sequence#) from v$archived_log where applied='YES' group by thread#;

Step 22: The next step would be to add the standby database TESTS to the cluster.
[oracle@drrac1 ~]$ srvctl add database -d TESTS -o /u01/app/oracle/product/11.2.0.1/db1 -r PHYSICAL_STANDBY -s MOUNT

Step 23: Now, we also need to add the instances entrires to the standby database.
[oracle@drrac1 ~]$ srvctl add instance -d TESTS -i TESTS1 -n drrac1
[oracle@drrac1 ~]$ srvctl add instance -d TESTS -i TESTS2 -n drrac2

Step 24: Now check the status of the standby database using srvctl.
[oracle@drrac1 ~]$ srvctl start database -d TESTS
[oracle@drrac1 ~]$ srvctl status database -d TESTS
Instance TESTS1 is running on node drrac1
Instance TESTS2 is running on node drrac2

  



SCAN 



BLACKOUT First

LISTENER_SCAN3_iormcl01 

Restart a specific SCAN listener

Restart scan Listener LISTENER_SCAN2: 
$ srvctl start scan_listener -i 3

Verify new SCAN listener status:
$ srvctl status scan_listener

Relocate listener -

As a grid user move sacn3 to node - iorsdb02-adm
$srvctl relocate scan -i 3 -n iorsdb02-adm

grid@iormdb01-adm:+ASM1:/home/grid
$ srvctl status scan
SCAN VIP scan1 is enabled
SCAN VIP scan1 is running on node iormdb01-adm
SCAN VIP scan2 is enabled
SCAN VIP scan2 is running on node iormdb01-adm
SCAN VIP scan3 is enabled
SCAN VIP scan3 is running on node iormdb01-adm

grid@iormdb01-adm:+ASM1:/home/grid
$ srvctl relocate scan -i 3 -n iormdb02-adm
grid@iormdb01-adm:+ASM1:/home/grid
$ srvctl status scan
SCAN VIP scan1 is enabled
SCAN VIP scan1 is running on node iormdb01-adm
SCAN VIP scan2 is enabled
SCAN VIP scan2 is running on node iormdb01-adm
SCAN VIP scan3 is enabled
SCAN VIP scan3 is running on node iormdb02-adm


Testing connectivity from workstation:
$ tnsping SRV0QMTR201

TNS Ping Utility for Linux: Version 11.2.0.4.0 - Production on 27-OCT-2015 09:20:23

Copyright (c) 1997, 2013, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/admin/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=iorscl01-scan)(PORT=39000))(LOAD_BALANCE=yes)(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=SRV0QMTR201)(FAILOVER_MODE=(TYPE=select)(METHOD=BASIC)(RETRIES=180)(DELAY=5))))
OK (0 msec)

Test SCAN listener(s) Status is/are alive and there is network issue. If it is network problem, the issue should be 
reproduced on all three IP addresses bound with SCAN name (host-scan.dbaplus.ca). 

Let’s find IPs first:
oracle@iorsdb01-adm:QMTR21:/home/oracle
$ nslookup iorscl01-scan
Server:         10.249.64.70
Address:        10.249.64.70#53

Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.25
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.26
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.27

oracle@iorsdb01-adm:QMQ221:/opt/oracle.SupportTools/exachk/exachk_iorsdb01-adm_QMQ22_01_081015_130528
$ ping -c 2 iorscl01-scan
PING iorscl01-scan.apac.ent.bhpbilliton.net (10.241.148.26) 56(84) bytes of data.
64 bytes from iorscl01-lsnr02.apac.ent.bhpbilliton.net (10.241.148.26): icmp_seq=1 ttl=64 time=0.012 ms
64 bytes from iorscl01-lsnr02.apac.ent.bhpbilliton.net (10.241.148.26): icmp_seq=2 ttl=64 time=0.012 ms

--- iorscl01-scan.apac.ent.bhpbilliton.net ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms

$ cat /etc/resolv.conf
--
#domain apac.ent.bhpbilliton.net
nameserver 10.249.64.70
nameserver 10.251.64.70
nameserver 10.251.64.71


Checking the IP availability:
oracle@iorsdb01-adm:QMTR21:/home/oracle
$ ping 10.241.148.25
PING 10.241.148.25 (10.241.148.25) 56(84) bytes of data.
64 bytes from 10.241.148.25: icmp_seq=1 ttl=64 time=0.014 ms
64 bytes from 10.241.148.25: icmp_seq=2 ttl=64 time=0.016 ms
64 bytes from 10.241.148.25: icmp_seq=3 ttl=64 time=0.019 ms
^C
--- 10.241.148.25 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2118ms
rtt min/avg/max/mdev = 0.014/0.016/0.019/0.004 ms
oracle@iorsdb01-adm:QMTR21:/home/oracle
$ ping 10.241.148.26
PING 10.241.148.26 (10.241.148.26) 56(84) bytes of data.
64 bytes from 10.241.148.26: icmp_seq=1 ttl=64 time=0.013 ms
64 bytes from 10.241.148.26: icmp_seq=2 ttl=64 time=0.015 ms
64 bytes from 10.241.148.26: icmp_seq=3 ttl=64 time=0.013 ms
^C
--- 10.241.148.26 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2550ms
rtt min/avg/max/mdev = 0.013/0.013/0.015/0.004 ms
oracle@iorsdb01-adm:QMTR21:/home/oracle
$ ping 10.241.148.27
PING 10.241.148.27 (10.241.148.27) 56(84) bytes of data.
64 bytes from 10.241.148.27: icmp_seq=1 ttl=64 time=0.047 ms
64 bytes from 10.241.148.27: icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from 10.241.148.27: icmp_seq=3 ttl=64 time=0.052 ms
64 bytes from 10.241.148.27: icmp_seq=4 ttl=64 time=0.049 ms
^C
--- 10.241.148.27 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 0.044/0.048/0.052/0.003 ms

NOTE: Request timed out. or (100% loss) Then
+ASM
$ crsctl status res -t

check SCAN listener status [ if offline ] 
When I connect to database from workstation, Oracle client will work as following:
1. Ask DNS server for IPs for the SCAN name host-scan.dbaplus.ca, and get three IPs
2. Choose one of them to connect. If I am lucky, the one should be SCAN1 and I will not find any issues. Actually, I was not. scan2 (or scan3) was chosen and the request waited for response from server until time out because the SCAN VIP is not ONLINE, then failed over to another SCAN VIP scan3 (or scan2) or scan1. The worst thing is failover to scan3, it will get timeout again and have to fail over finally to scan1.
3. No matter where it starts, SCAN VIP scan1 will be the final and only choice for the connection. How fast the connection can be established depends on how fast the ONLINE SCAN VIP scan1 is chosen.

To fix the issue, try to start them:

$ srvctl start scan_listener -i 2
$ srvctl start scan_listener -i 3

oracle@iorsdb01-adm:QMTR21:/home/oracle
$ srvctl status scan_listener
SCAN Listener LISTENER_SCAN1 is enabled
SCAN listener LISTENER_SCAN1 is running on node iorsdb01-adm
SCAN Listener LISTENER_SCAN2 is enabled
SCAN listener LISTENER_SCAN2 is running on node iorsdb02-adm
SCAN Listener LISTENER_SCAN3 is enabled
SCAN listener LISTENER_SCAN3 is running on node iorsdb01-adm

zgrep 12514 listener_scan1.log.20151203-0700.gz |grep PMTR > LISQMTR1.log

Client Trace
----------------

TRACE_UNIQUE_CLIENT = ON 
TRACE_LEVEL_CLIENT = 16 
TRACE_DIRECTORY_CLIENT = C:\temp  
TRACE_FILE_CLIENT = SQLNetTrace 
TRACE_TIMESTAMP_CLIENT = ON 

RENAME SCAN in Cluster
------------------------------------------
After the installation of the Oracle 11g R2 cluster with SCAN, If you need to change the scan name from scan.freeoraclehelp.com to newscan.freeoraclehelp.com, here is how. 

1. Stop the SCAN: Source Grid home, srvctl stop scan_listener, and srvctl stop scan then. 

[grid@rac1 ~]$ srvctl stop scan_listener
[grid@rac1 ~]$ srvctl stop scan2.  Configure the new SCAN in your DNS, or /etc/hosts, or GNS and make sure that lookups are working for the new name. 

[root@rac2 ~]# nslookup newscan.freeoraclehelp.com
Server:         192.168.1.20
Address:        192.168.1.20#53

Name:   newscan.freeoraclehelp.com
Address: 192.168.1.32
Name:   newscan.freeoraclehelp.com
Address: 192.168.1.33
Name:   newscan.freeoraclehelp.com
Address: 192.168.1.34
[root@rac2 ~]#
3. Configure the Cluster to take the new VIPs 
As root user on one of the cluster nodes (not needed on both the nodes):

[root@rac1 ~]# /oracle/product/11.2.0/11.2.0/grid/bin/srvctl modify scan -n newscan.freeoraclehelp.com
[root@rac1 ~]# As grid user on one of the cluster nodes (not needed on both the nodes):

[grid@rac1 ~]$ srvctl modify scan_listener -u
[grid@rac1 ~]$ srvctl start scan_listenerVerify that configuration is right and three SCAN listeners are started.  

[grid@rac1 ~]$ srvctl config scan
SCAN name: newscan.freeoraclehelp.com, Network: 1/192.168.1.0/255.255.255.0/eth0
SCAN VIP name: scan1, IP: /newscan.freeoraclehelp.com/192.168.1.34
SCAN VIP name: scan2, IP: /newscan.freeoraclehelp.com/192.168.1.33
SCAN VIP name: scan3, IP: /newscan.freeoraclehelp.com/192.168.1.32
[grid@rac1 ~]$ srvctl config scan_listener
SCAN Listener LISTENER_SCAN1 exists. Port: TCP:1521
SCAN Listener LISTENER_SCAN2 exists. Port: TCP:1521
SCAN Listener LISTENER_SCAN3 exists. Port: TCP:1521
[grid@rac1 ~]$

RENAME SCAN Port In Cluster
-----------------------------------------------
As grid user, source the grid environment to make sure $GRID_HOME/bin is in PATH and 

1. Modify SCAN listener port: 
[oid1@rac1 ~]$ srvctl modify scan_listener -p 1522
2. Restart SCAN listener so the new port will be effective: 
[oid1@rac1 ~]$ srvctl stop scan_listener
[oid1@rac1 ~]$ srvctl start scan_listener
3. Confirm the change: 
[oid1@rac1 ~]$ srvctl config scan_listener
SCAN Listener LISTENER_SCAN1 exists. Port: TCP:1522
SCAN Listener LISTENER_SCAN2 exists. Port: TCP:1522
SCAN Listener LISTENER_SCAN3 exists. Port: TCP:1522


DNS-GNS
----------------

For GNS, Oracle Cluster starts a DHCP and 
DNS Server for you and you do not need to setup the SCAN in DNS at all.

If you have DNS server in your network, you do not need to select the GNS (unlike the one in above screenshot). I see that you have setup DNS record for SCAN. You just need to supply this SCAN info and not select "Configure GNS" during the installation. Refer to the following two grid installations.. first one is without GNS (with DNS.. just like your case) and second one is with GNS (and without DNS Setup for SCAN):

RAC SCAN Listener
-----------------
[grid@csmper-cls07 ~]$ srvctl config scan
SCAN name: csmper-scan01.apac.ent.bhpbilliton.net, Network: 1/10.240.26.0/255.255.255.0/bond0
SCAN VIP name: scan1, IP: /10.240.26.34/10.240.26.34
SCAN VIP name: scan2, IP: /10.240.26.36/10.240.26.36
SCAN VIP name: scan3, IP: /10.240.26.35/10.240.26.35

[grid@csmper-cls07 ~]$ srvctl config scan_listener
SCAN Listener LISTENER_SCAN1 exists. Port: TCP:1521
SCAN Listener LISTENER_SCAN2 exists. Port: TCP:1521
SCAN Listener LISTENER_SCAN3 exists. Port: TCP:1521

grid@csmper-cls18 ~]$ srvctl status listener
Listener LISTENER is enabled
Listener LISTENER is running on node(s): csmper-cls18
[grid@csmper-cls18 ~]$ srvctl status scan_listener
SCAN Listener LISTENER_SCAN1 is enabled
SCAN listener LISTENER_SCAN1 is running on node csmper-cls18
SCAN Listener LISTENER_SCAN2 is enabled
SCAN listener LISTENER_SCAN2 is running on node csmper-cls18
SCAN Listener LISTENER_SCAN3 is enabled
SCAN listener LISTENER_SCAN3 is running on node csmper-cls18

[grid@csmper-cls07 ~]$ nslookup csmper-scan01.apac.ent.bhpbilliton.net
Server:         10.27.12.32
Address:        10.27.12.32#53

Name:   csmper-scan01.apac.ent.bhpbilliton.net
Address: 10.240.26.36
Name:   csmper-scan01.apac.ent.bhpbilliton.net
Address: 10.240.26.35
Name:   csmper-scan01.apac.ent.bhpbilliton.net
Address: 10.240.26.34

lsnrctl service LISTENER_SCAN3

[grid@csmper-cls06 ~]$ cat /etc/resolv.conf
search apac.ent.bhpbilliton.net
#nameserver 10.27.40.20
#nameserver 10.27.12.20
nameserver 10.27.12.32
nameserver 10.27.12.20

[grid@csmper-cls06 ~]$  ping 10.27.12.32
PING 10.27.12.32 (10.27.12.32) 56(84) bytes of data.
64 bytes from 10.27.12.32: icmp_seq=1 ttl=125 time=0.163 ms
64 bytes from 10.27.12.32: icmp_seq=2 ttl=125 time=0.227 ms
64 bytes from 10.27.12.32: icmp_seq=3 ttl=125 time=0.152 ms

Check On Exa
+++++++++++++++
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ srvctl config scan
SCAN name: iorscl01-scan, Network: 1/10.241.148.0/255.255.255.0/bondeth0
SCAN VIP name: scan1, IP: /iorscl01-scan/10.241.148.25
SCAN VIP name: scan2, IP: /iorscl01-scan/10.241.148.27
SCAN VIP name: scan3, IP: /iorscl01-scan/10.241.148.26
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ srvctl config scan_listener
SCAN Listener LISTENER_SCAN1 exists. Port: TCP:39000
SCAN Listener LISTENER_SCAN2 exists. Port: TCP:39000
SCAN Listener LISTENER_SCAN3 exists. Port: TCP:39000
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ srvctl status scan
SCAN VIP scan1 is enabled
SCAN VIP scan1 is running on node iorsdb01-adm
SCAN VIP scan2 is enabled
SCAN VIP scan2 is running on node iorsdb02-adm
SCAN VIP scan3 is enabled
SCAN VIP scan3 is running on node iorsdb01-adm
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ srvctl status scan_listener
SCAN Listener LISTENER_SCAN1 is enabled
SCAN listener LISTENER_SCAN1 is running on node iorsdb01-adm
SCAN Listener LISTENER_SCAN2 is enabled
SCAN listener LISTENER_SCAN2 is running on node iorsdb02-adm
SCAN Listener LISTENER_SCAN3 is enabled
SCAN listener LISTENER_SCAN3 is running on node iorsdb01-adm
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ nslookup iorscl01-scan
Server:         10.249.64.70
Address:        10.249.64.70#53

Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.25
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.27
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.26

oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ cat /etc/resolv.conf
# 13714588 add timeout, rotate, attempts to mitigate issues with poor or misconfigured single DNS server
# timeout:n Initial timeout for a query to a nameserver. The default value is five seconds. The maximum value is 30 seconds.
# For the second and successive rounds of queries, the resolver doubles the initial timeout and is divided by the number
# of nameservers in the resolv.conf file.
options timeout:4
# attempts:n How many queries the resolver should send to each nameserver in the resolv.conf file before it stops execution.
# The default value is 2. The maximum value is 5.
options attempts:2
# rotate Enables the resolver to use all the nameservers in the resolv.conf file, not just the first one.
options rotate
# Search domain and name server
search apac.ent.bhpbilliton.net
# Commented it out, because OUI complains about it
#domain apac.ent.bhpbilliton.net
nameserver 10.249.64.70
nameserver 10.251.64.70
nameserver 10.251.64.71
oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ nslookup iorscl01-scan
Server:         10.249.64.70
Address:        10.249.64.70#53

Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.25
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.26
Name:   iorscl01-scan.apac.ent.bhpbilliton.net
Address: 10.241.148.27

oracle@iorsdb01-adm:DBDW1_1:/home/oracle
$ ping 10.249.64.70
PING 10.249.64.70 (10.249.64.70) 56(84) bytes of data.
64 bytes from 10.249.64.70: icmp_seq=1 ttl=123 time=0.967 ms
64 bytes from 10.249.64.70: icmp_seq=2 ttl=123 time=0.920 ms
64 bytes from 10.249.64.70: icmp_seq=3 ttl=123 time=0.914 ms
^C
--- 10.249.64.70 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2822ms
rtt min/avg/max/mdev = 0.914/0.933/0.967/0.042 ms


  



ServerPool&Service 



Create a serverpool pool mypool with min = 2, max = 2, imp = 1
# srvctl add srvpool -g mypool -l 2 -u 2 -i 1

Convert an admin managed database to a policy managed database,The conversion to policy managed database can be done without downtime. Converting from Policy to Admin managed database it requires downtime.

Step 1: Validate current setup.
You can use the srvctl config database -d <db_unique_name> to valide the type of management you are using.
 
$ srvctl config database -d PSIM1_01
Database unique name: PSIM1_01

--
Services: SRV0PSIM101,SRVPSIM1RMN
Type: RAC
Database is administrator managed

Step 2: Create a Server Pool.
We will create a serverpool called prod01, where we define a minimum of 2 server (-l) and a maximum of 2 server (-u).
[oracle@server1 ~]$ srvctl add srvpool -g prod01 -l 2 -u 2
  
[oracle@server1 ~]$ srvctl status srvpool
Server pool name: Free
Active servers count: 0
Server pool name: Generic
Active servers count: 2
Server pool name: prod01
Active servers count: 0
  
Step 3: Modify the database configuration.
Modify the current admin managed configuration to the new created server pool.
[oracle@server1 ~]$ srvctl modify database -d rac -g prod01
  -------------------------------------------------------------
$ crsctl check cluster -all
**************************************************************
iorsdb01-adm:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
iorsdb02-adm:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************

$ crsctl status serverpool
 
ACL:  String in the following format:owner:user:rwx,pgrp:group:rwx,other::r— Defines the owner of the server pool and which privileges are granted to various operating system users and groups.

    r: Read only
    w: Modify attributes of the pool or delete it
    x: Assign resources to this pool

By default, the identity of the client that creates the server pool is the owner.
Clusterware can move servers from server pool robotically to maintain important server pool configuration. By default two pools get created after installation of the cluster : Generic Pool and Free Pool

- Generic Pool: The servers in the pool are used to host Admin Managed databases.
  When you upgrade your existing version of the clusterware to 11.2, all the nodes get mapped to the in-built Generic pool.This is an internally managed server pool and the modification of the attributes of this pool is not allowed. When you create a database which is Admin-managed, that also becomes a part of the Generic Pool as a child pool of it.

 Free Pool: All the servers which have not been assigned to any pool are part of this pool. When the clusterware starts , initially all the servers are assigned to this pool  only. As applications start, servers move from this pool to Generic or user defined pool dynamically. As like the Generic pool, this is also an internally managed pool but still some attributes are available to be modified by the dba like IMPORTANCE.
In addition to above pools, users can also define server pools.
 
The allocation of the hosts is going to be happening in the following order:
     1. Generic Pool
     2. Explicitly created pool
     3. Free Pool

Servers will be assigned to explicitly created pools  as per the following rules:
1.    Fill all server pools in order of importance until they meet  their minimum.
2.    Fill all server pools in order of importance until they meet  their maximum.
3.    By default any left over go to FREE.
 
When you define services for a policy-managed database, you define the service to a server pool where the database is running. You can define the service as either UNIFORM (running on all instances in the server pool) or SINGLETON (running on only one instance in the server pool). For SINGLETON services, Oracle RAC chooses on which instance in the server pool the service is active. If that instance fails, then the service fails over to another instance in the server pool. A service can only run in one server pool.
Services for administrator-managed databases continue to be defined by the PREFERRED and AVAILABLE definitions. 
How do I convert from a Policy-Managed Database to Administrator-Managed Database?
You cannot directly convert a policy-managed database to an administrator-managed database. Instead, you can remove the policy-managed configuration using the 'srvctl remove database' and 'srvctl remove service' commands, and then create a new administrator-managed database with the 'srvctl add database' command.
Cluster database configuration can be Policy-Managed or Administrator-Managed. A Policy-Managed database is dynamic with instances managed automatically based on server pools for effective resource utilization. 
Admin-Managed database results in instances tied to specific servers.

Case #1 : Convert Administrator-Managed Database to Policy-Managed Database
==================================================================
$ srvctl config database -d RACTDB  
Log : --An entry at the bottom Database is administrator managed
1.	Create a server pool for the Policy-Managed database
$ srvctl add srvpool -g TESTMONO -l 0 -u 2  
$ srvctl config serverpool  
 
-g is the server pool name
-l is the minimum number of nodes
-u maximum number of nodes
-i importance given to the server pool 
-n is the node name

2.	Modify the database to be in the new server pool, as follows:
$ srvctl modify database -d orcl -g TESTMONO -f  
$ srvctl config database -d orcl   

Log : Database is policy managed  
Servers already  located in PMSRVPOOL
$ srvctl status srvpool  

3. Last step is configuration of Oracle Enterprise Manager

Documentation says :  
Configure Oracle Enterprise Manager to recognize the change you made in the previous procedure, as follows:
1.	In order for Oracle Enterprise Manager Database Control to recognize the new database instances, you must change the instance name fromdb_unique_name# to db_unique_name_# (notice the additional underscore (_) before the number sign (#) character).
2.	Rename the orapwd file in the dbs/database directory (or create a new orapwd file by running the orapwd command). By default, there is an orapwd file with the instance name appended to it, such as orapwdORCL1. You must change the name of the file to correspond to the instance name you changed in the previous step. For example, you must change orapwdORCL1 to orapwdORCL_1 or create a new orapwd file.
3.	Run emca to reconfigure Oracle Enterprise Manager Database Control, as follows:
emca -config dbcontrol db -cluster
Database is Policy Managed, now.
$ srvctl config database -d racdb   

Case #2 : Convert Policy Managed Database to Administrator Managed Database

You cannot directly convert a policy-managed database to an administrator-managed database. Instead, you can remove the policy-managed configuration using the srvctl remove database and srvctl remove service commands, and then register the same database as an administrator-managed database using the srvctl add database and srvctl add instance commands. Once you register the database and instance, you must use the srvctl add service command to add back the services as you removed them.
1.	Remove database with SRVCTL tool. 

Check
++++++++++
[oracle@rac1 bin]$ srvctl config database -d RACTDB
Database unique name: RACTDB
Database name: 
Oracle home: /u01/app/oracle/product/12.1.0/dbhome_1
Oracle user: oracle
Spfile: 
Password file: 
Domain: 
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: TESTMONO
Database instances: 
Disk Groups: FRA,DATA
Mount point paths: 
Services: 
Type: RAC
Start concurrency: 
Stop concurrency: 
Database is policy managed

[oracle@rac1 bin]$ srvctl remove database -d RACTDB  
PRKO-3141 : Database RACTDB could not be removed because it was running
[oracle@rac1 bin]$ srvctl status database -d RACTDB  
Instance RACTDB_1 is running on node rac1
Instance RACTDB_2 is running on node rac2
[oracle@rac1 bin]$ srvctl stop database -d RACTDB  
l[oracle@rac1 bin]$ srvctl status database -d RACTDB  
Instance RACTDB_1 is not running on node rac1
Instance RACTDB_2 is not running on node rac2
[oracle@rac1 bin]$ srvctl remove database -d RACTDB
Remove the database RACTDB? (y/[n]) y
[oracle@rac1 bin]$ srvctl status database -d RACTDB  
PRCD-1120 : The resource for database RACTDB could not be found.
PRCR-1001 : Resource ora.ractdb.db does not exist
2.	Add database Administrator Managed
[oracle@rac1 bin]$ srvctl add database -d RACTDB -o /u01/app/oracle/product/11.2.0/dbhome -y automatic
PRCD-1025 : Failed to create database RACTDB
PRCT-1406 : Oracle Home location: /u01/app/oracle/product/11.2.0/dbhome does not contain bin/srvctl

$ srvctl add database -d RACTDB -o /u01/app/oracle/product/11.2.0/dbhome -y automatic
$ srvctl config database -d RACTDB 
Database is administrator managed  
3. Adding database instances
$ srvctl add instance -d RACTDB -i RACTDB_1 -n rac1  
$ srvctl add instance -d RACTDB -i RACTDB_2 -n rac2

$ srvctl status database -d RACTDB 
4. You must  configure Oracle Enterprise Manager on last step.

 Cleanup 
--------------
[oracle@rac1 bin]$ ./srvctl remove srvpool -g MONOTST
PRCS-1012 : Failed to remove server pool MONOTST
PRCD-1002 : Database orcl is dependent upon server pool MONOTST
Create a new serverpool:
[oracle@rac1 bin]$ srvctl add srvpool -g TESTMONO -l 2 -u 2  
Assign database to new server pool:
[oracle@rac1 bin]$ srvctl modify database -d orcl -g TESTMONO -f  
Remove:
[oracle@rac1 bin]$ srvctl remove serverpool -g MONOTST

 
CHILD POOLS

 Create a parent pool pmpool3 with min = 3, max=3 and importance = 1

[root@host02 host02]# srvctl add srvpool -g pmpool3 -l 3 -u 3 -i 1

Check that all the three servers have been assigned to pmpool3 

./srvctl status srvpool -g TESTMONO 

++++++++++++++++++++++++++++++++++++++
SERVICE
+++++++++++++++++++++++++++++++
Service defined for policy-managed databases.
++++++++++++++++++++++++++++++++++++++++
There are restrictions creating services on policy-managed databases, services there are assigned to a server pool and can be defined as a Singleton or Uniform service.

A Singleton service runs only on one database instance on its server pool, and the user does not have the control over wich instance will serve the service.

[oracle@rac1 admin]$ srvctl add service -d orcl -s PROD1 -g MONOTST -c singleton -y manual
[oracle@rac1 admin]$ srvctl status service -d ORCL -s PROD1
Service PROD1 is not running.
[oracle@rac1 admin]$ srvctl start service -d ORCL -s PROD1
[oracle@rac1 admin]$ srvctl status service -d ORCL -s PROD1
Service PROD1 is running on nodes: rac1

A Uniform service runs on all database instances in its server pool.

[oracle@rac1 admin]$ srvctl stop service -d ORCL -s PROD1
[oracle@rac1 admin]$ srvctl modify service -d orcl -s PROD1 -g MONOTST -c uniform -y manual
[oracle@rac1 admin]$ srvctl start service -d ORCL -s PROD1[oracle@rac1 admin]$ srvctl status service -d ORCL -s PROD1Service PROD1 is running on nodes: rac1,rac2

Service defined for admin-managed
+++++++++++++++++++++++++++++++++
The administrator-managed database runs the service assigning it in the preferred instances, if the preferred instance fails the service will run on the available instance.

This sql statement show information about services from v$services view.

SQL> select name, network_name, creation_date, goal, dtp, aq_ha_notification,clb_goal from v$services;

NAME       NETWORK_NAME      CREATION_ GOAL    D AQ_ CLB_G
-------------------- ------------------------------ --------- ------------ - --- -----
servicetest      servicetest      22-JUL-13 NONE    N NO  LONG
RACDBXDB      RACDBXDB       07-JUL-13 NONE    N NO  LONG
RACDB.test      RACDB.test      07-JUL-13 NONE    N NO  LONG
SYS$BACKGROUND         07-JUL-13 NONE    N NO  SHORT
SYS$USERS         07-JUL-13 NONE    N NO  SHORT

Management Policy.

Oracle 11 allows specifying the management policy, manual or automatic, occasionally the DBA may want to start the database services manually, with srvct utility change the management policy to MANUAL to accomplish this goal.

Changing Service goal.
---------------------------------------
Service goal consist in a classification per service, this parameter could be SHORT or LONG due to the session life.

[oracle@racnode1 ~]$ srvctl modify service -s SERVICE_NAME -d RACDB -j SHORT

[oracle@racnode1 ~]$ srvctl config service -d RACDB
Connection Load Balancing Goal has changed from LONG to short.

Changing DTP (Distributed Transaction Processing).

By default all services running distributed transactions will be distributed along all rac instances, this option can be disabled or enabled.

[oracle@racnode1 ~]$ srvctl modify service -s servicetest -d RACDB -x TRUE
[oracle@racnode1 ~]$ srvctl config service -d RACDB

Changing TAF (Transparent Application Failover).

Established sessions on a database are reallocated on other database instance in case of a source instance failure.
There are three policies: NONE, BASIC and PRECONNECT.
None: No TAF policy applied.
Basic: Restarts the failed query on the new database instance upon failover.
Preconnect: The same as the Basic method but in this case Oracle anticipates connection failure creating a shadow connection on the other instance, which is always available on the available instance, to improve the failover time.

[oracle@racnode1 ~]$ srvctl modify service -s servicetest -d RACDB -P BASIC

### Creating new services.
```
The following command creates a service called TEST that defines RACDB1 as the preferred instance, RACDB2 as available instance, and BASIC TAF policy, also configures the service to start automatically using automatic management policy.

[oracle@rac1 bin]$ srvctl add service -d RACTDB -s TEST -r RACTDB_1 -a RACTDB_2 -P basic -y AUTOMATIC
PRKO-3114 : Policy-managed database RACTDB can not support administrator-managed service TEST.

This sql statement show information about services from v$services view.

SQL> select name, network_name, creation_date, goal, dtp, aq_ha_notification,clb_goal from v$services;

11g:srvctl add service -d orcl12 -s po -r oel6vm1 -a oel6vm2 -m BASIC -e SESSION -z 5 -w 60

export ORACLE_HOME=$DBOH; $DBOH/bin/srvctl add service -d $DBUNNAME -s $SRVNAME -e SELECT -m BASIC -j SHORT -w 3 -z 180 -pdb $PDBNAME"

12c
In version 12c, adding a service requires a different syntax, and options are specified more verbally. The earlier
command is written in version 12c as follows.
$ srvctl add service -db orcl12 -service po -preferred oel6vm1 -available oel6vm2 -tafpolicy BASIC -failovermethod SESSION -failoverretry 5 -failoverdelay 60

Failure -Test failed: Listener refused the connection with the following error:
ORA-12514, TNS:listener does not currently know of service requested in connect descriptor

$ srvctl config database -d TICL1_01
Database unique name: TICL1_01
Database name: TICL1
Oracle home: /u01/app/oracle/product/11.2.0.4/db_000
Oracle user: oracle
Spfile: +DATA/TICL1_01/spfileTICL1.ora
Domain:
--
Services: SRVTICL1RMN
Type: RACOneNode
Online relocation timeout: 30
Instance name prefix: TICL1
Candidate servers: iorsdb01-adm,iorsdb02-adm
Database is administrator managed
oracle@iorsdb02-adm:TICL1_1:/home/oracle
$ srvctl add service -d TICL1_01 -s SRV0TICL101 -r TSIM1
$ srvctl status service -s SRV0TICL101 -d TICL1_01
Service SRV0TICL101 is not running.
$ srvctl start service -s SRV0TICL101 -d TICL1_01
$ srvctl status service -s SRV0TICL101 -d TICL1_01
Service SRV0TICL101 is running on instance(s) TICL1_1

QWIC CHECK
---------------------
select inst_id,name
from gv$services
order by inst_id,name;

   INST_ID NAME
---------- ----------------------------------------------------------------
         1 SRV0TICL101
         1 SRVTICL1RMN
         1 SYS$BACKGROUND
         1 SYS$USERS
         1 TICL1XDB
         1 TICL1_01


Database-service-with-physical-standby-role-option-in-11gr2
RAC Database Service With Physical Standby Role Option In 11gR2 - New Feature (Doc ID 1129143.1)

1) Since standby is Read only first try to start the service on primary
$ srvctl add service -d PAWR1_02 -s SRV0PAWR102 -r "PAWR13,PAWR14" -l PHYSICAL_STANDBY -e SELECT -m BASIC -j SHORT -w 3 -z 180
This will register the service with database
2) Perform couple of log switches
3) Allow standby to catch-up
4) Now stop the service on primary and start it on standby
srvctl stop service -d PAWR1_01 -s SRV0PAWR102
srvctl start service -d PAWR1_02 -s SRV0PAWR102

$ srvctl config database -d PAWR1_02
Database unique name: PAWR1_02
Database name: PAWR1
Oracle home: /u01/app/oracle/product/12.1.0.2/db_000
Oracle user: oracle
Spfile: +DATA/PAWR1_02/spfilepawr1.ora
Password file:
Domain:
Start options: mount
Stop options: immediate
Database role: PHYSICAL_STANDBY
Management policy: AUTOMATIC
Server pools:
Disk Groups: DATA,RECO
Mount point paths:
Services: SRV0PAWR101,SRVPAWR1RMN
Type: RAC
Start concurrency:
Stop concurrency:
OSDBA group: dba
OSOPER group: racoper
Database instances: PAWR13,PAWR14
Configured nodes: iorsdb01-adm,iorsdb02-adm
Database is administrator managed


$ srvctl add service -d PAWR1_02 -s SRV0PAWR102 -r "PAWR13,PAWR14" -l PHYSICAL_STANDBY -e SELECT -m BASIC -j SHORT -w 3 -z 180

$ srvctl config service -d PAWR1_02 -s SRV0PAWR102
Service name: SRV0PAWR102
Server pool:
Cardinality: 2
Disconnect: false
Service role: PHYSICAL_STANDBY
Management policy: AUTOMATIC
DTP transaction: false
AQ HA notifications: false
Global: false
Commit Outcome: false
Failover type: SELECT
Failover method: BASIC
TAF failover retries: 180
TAF failover delay: 3
Connection Load Balancing Goal: SHORT
Runtime Load Balancing Goal: NONE
TAF policy specification: NONE
Edition:
Pluggable database name:
Maximum lag time: ANY
SQL Translation Profile:
Retention: 86400 seconds
Replay Initiation Time: 300 seconds
Session State Consistency:
GSM Flags: 0
Service is enabled
Preferred instances: PAWR13,PAWR14
Available instances:

$ srvctl status database -d pawr1_02 -v
Instance PAWR13 is running on node iorsdb01-adm. Instance status: Open,Readonly.
Instance PAWR14 is running on node iorsdb02-adm. Instance status: Open,Readonly.


SQL> select database_role,open_mode from v$database;

DATABASE_ROLE                                    OPEN_MODE
------------------------------------------------ ------------------------------------------------------------
PHYSICAL STANDBY                                 READ ONLY WITH APPLY

$ srvctl start service -d PAWR1_02 -s SRV0PAWR102

### SSH setup4user Equ 

```

On NODE1: Login as oracle user.

[oracle@node1 ~]$ mkdir ~/.ssh 
[oracle@node1 ~]$ chmod 755 ~/.ssh 
[oracle@node1 ~]$ /usr/bin/ssh-keygen -t rsa 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/oracle/.ssh/id_rsa): [Just press enter]
Enter passphrase (empty for no passphrase): [We entered oracle as a passphrase]
Enter same passphrase again: [Enter oracle again]
Your identification has been saved in /home/oracle/.ssh/id_rsa.
Your public key has been saved in /home/oracle/.ssh/id_rsa.pub.
The key fingerprint is:
8d:83:1f:87:4a:1f:c5:24:ad:c7:3f:88:b3:0c:86:75 oracle@node1.oracleaceonline.com
[oracle@node1 ~]$ /usr/bin/ssh-keygen -t dsa
Generating public/private dsa key pair.
Enter file in which to save the key (/home/oracle/.ssh/id_dsa): [Just press enter]
Enter passphrase (empty for no passphrase): [We entered oracle as a passphrase]
Enter same passphrase again: [Enter oracle again]
Your identification has been saved in /home/oracle/.ssh/id_dsa.
Your public key has been saved in /home/oracle/.ssh/id_dsa.pub.
The key fingerprint is:
22:c5:40:77:b8:76:f1:e0:9d:d0:a7:34:81:b3:c0:e6 oracle@node1.oracleaceonline.com
[oracle@node1 ~]$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
[oracle@node1 ~]$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys 
[oracle@node1 ~]$ chmod 644 ~/.ssh/authorized_keys


Repeat the same steps on NODE2 after logging in as oracle user. 

Now you have /home/oracle/.ssh/authorized_keys file on both nodes that contains rsa and dsa keys for that node.

Issuing the following command on NODE1 will add NODE2's credentials to NODE1's authorized_keys file. By doing so you will be able to SSH NODE1 from NODE2 without specifying any password, but not vice versa.

[oracle@node1 ~]$ ssh oracle@rac2 cat ~/.ssh/authorized_keys >> ~/.ssh/authorized_keys
The authenticity of host 'node2 (10.140.3.56)' can't be established.
RSA key fingerprint is 0e:3c:2f:02:9b:44:e5:92:4f:21:72:18:a8:64:ff:d7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node2,10.140.3.56' (RSA) to the list of known hosts.
oracle@node2's password: [Enter oracle's password]

To make it two way repeat the above step on NODE2 as well.

[oracle@node2 ~]$ ssh oracle@192.168.2.13 cat ~/.ssh/authorized_keys >> ~/.ssh/authorized_keys
The authenticity of host 'node1 (10.140.3.55)' can't be established.
RSA key fingerprint is 0e:3c:2f:02:9b:44:e5:92:4f:21:72:18:a8:64:ff:d7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node1' (RSA) to the list of known hosts.
Enter passphrase for key '/home/oracle/.ssh/id_rsa': [Enter oracle]


This ends the configuration of SSH for User Equivalence. Now if you try to ssh NODE1 from NODE2 or vice versa it should not ask for any password.

[oracle@node2 ~]$ ssh node1 date
Enter passphrase for key '/home/oracle/.ssh/id_rsa': [Enter oracle]
Tue Nov 18 13:50:46 AST 2008
[oracle@node1 ~]$ ssh node2 date
Enter passphrase for key '/home/oracle/.ssh/id_rsa': [Enter oracle]
Tue Nov 18 13:51:16 AST 2008

As you can see it is not asking for oracle's password any more, but now it is asking for the passphrase that you entered while configuring the SSH. To avoid this you have two solutions: (1) During the configuration when asked for passphrase key just press Enter and do not specify any passphrase key. This way it will not ask for any password or passphrase key. (2) With the current configuration, for every new session, execute the following on the node from where you will SSH other nodes.

[oracle@node2 ~]$ exec /usr/bin/ssh-agent $SHELL
[oracle@node2 ~]$ /usr/bin/ssh-add
Enter passphrase for /home/oracle/.ssh/id_rsa:
Identity added: /home/oracle/.ssh/id_rsa (/home/oracle/.ssh/id_rsa)
Identity added: /home/oracle/.ssh/id_dsa (/home/oracle/.ssh/id_dsa)
[oracle@node2 ~]$ ssh node1 date
Tue Nov 18 13:51:18 AST 2008
[oracle@node2 ~]$ ssh node1 date
The authenticity of host 'node1 (10.140.3.55)' can't be established.
RSA key fingerprint is 0e:3c:2f:02:9b:44:e5:92:4f:21:72:18:a8:64:ff:d7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node1' (RSA) to the list of known hosts.
Tue Nov 18 15:26:29 AST 2008

[oracle@node2 ~]$ ssh node1.oracleaceonline.com date
The authenticity of host 'node1.oracleaceonline.com (10.140.3.55)' can't be established.
RSA key fingerprint is 0e:3c:2f:02:9b:44:e5:92:4f:21:72:18:a8:64:ff:d7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node1.oracleaceonline.com' (RSA) to the list of known hosts.
Tue Nov 18 15:51:18 AST 2008

[oracle@node2 ~]$ ssh 10.140.3.55 date
The authenticity of host '10.140.3.55 (10.140.3.55)' can't be established.
RSA key fingerprint is 0e:3c:2f:02:9b:44:e5:92:4f:21:72:18:a8:64:ff:d7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.140.3.55' (RSA) to the list of known hosts.
Tue Nov 18 16:07:31 AST 2008

[oracle@node2 ~]$ ssh node1 date
Tue Nov 18 16:51:18 AST 2008
[oracle@node2 ~]$ ssh node1.oracleaceonline.com date
Tue Nov 18 16:51:21 AST 2008
[oracle@node2 ~]$ ssh 10.140.3.55 date
Tue Nov 18 16:51:34 AST 2008

Once User Equivalence is established you can also copy files by using scp on remote machines without specifying any password. We have configured User Equivalence for the following purposes:

To clone E-Business we have scripts that automate the cloning process. Those scripts copy Oracle Home, last RMAN backup and archive logs of the database, and Application tier on to the test server without asking for any password.
In E-Business we have created a concurrent program which is used to migrate newly developed reports from test instances to the Production instance without logging in to the test or production server.
And of course for RAC.
References: 
Metalink Note ID: 300548.1 - How To Configure SSH for a RAC Installation
Metalink Note ID: 372795.1 - How to Configure SSH for User Equivalence
```
  
### TempSpaceUsage 

Monitor the temp space allocation to make sure each instance has enough temp space available and that the temp space is allocated evenly among the instances. The following SQL is used:
```
select inst_id, tablespace_name, segment_file, total_blocks, 
used_blocks, free_blocks, max_used_blocks, max_sort_blocks 
from gv$sort_segment;

select inst_id, tablespace_name, blocks_cached, blocks_used 
from gv$temp_extent_pool;

select inst_id,tablespace_name, blocks_used, blocks_free 
from gv$temp_space_header;

select inst_id,free_requests,freed_extents 
from gv$sort_segment;
```
If temp space allocation between instances has become imbalanced, it might be necessary to manually drop temporary segments from an instance. The following command is used for this:alter session set events 'immediate trace name drop_segments level <TS number + 1>';See Bug 4882834 for details.

For each temporary tablespace, allocate at least as many temp files as there are instances in the cluster.

 
### Convert Archivelog Mode to NoArchivelog Mode in RAC
```
Step 1: Set the Environment Variable
TRAH1_1
$ srvctl status database -d TRAH1_01
Instance TRAH1_1 is running on node iorsdb02-adm
Online relocation: INACTIVE

Step 2: Disable Cluster
SQL>  alter system set CLUSTER_DATABASE=FALSE scope=spfile;

System altered.

Step 3: Set the parameters (Optional)
SQL> alter system set LOG_ARCHIVE_START= FALSE scope=spfile;
system altered

Step 4: Shutdown the database
SQL> exit 
$ srvctl stop database -d TRAH1_01

Step 5: Mount the database
sq

SQL> startup mount
Step 6: Disable Archive log
SQL> alter database noarchivelog;

Step 7: Enable the cluster
SQL> alter system set CLUSTER_DATABASE=TRUE scope=spfile;
system altered

Step 8: Shutdown the instance
SQL> shutdown immediate;

Step 9: Start all the instances
$ srvctl start database -d TRAH1_01 -n iorsdb02-adm
oracle@iorsdb02-adm:TRAH1_1:/home/oracle

Step 10: verify the status.
$ srvctl status database -d TRAH1_01
Instance TRAH1_1 is running on node iorsdb02-adm
Online relocation: INACTIVE

sq
$ sqlplus "sys as sysdba"
SQL> archive log list
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     45864
Current log sequence           45866 
```
