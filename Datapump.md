### Export Import 
```
grant imp_full_database to system;
grant exp_full_database to system;
create database link EXA_MIG connect to SYSTEM identified by tiger using 'IIBDEV';
x.par file
 /******** AS SYSDBA network_link=EXA_MIG directory=EXA_MIG logfile=exp_iibuat.log dumpfile=exp_iibuat_%U.dmp full=y parallel=3 reuse_dumpfiles=true

 /******** AS SYSDBA parfile=x.par
```

## For export on 9i with Flashback_SCN -    $ORACLE_HOME/rdbms/admin/catmeta.sql
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```
Error Message:
ORA-39083: Object type PROCACT_SCHEMA failed to create with error:
ORA-06550: line 2, column 1:
PLS-00201: identifier 'DBMS_CUBE_EXP.SCHEMA_INFO_IMP_BEG' must be declared
ORA-06550: line 2, column 1:
PL/SQL: Statement ignored
Cause  Oracle Text was installed in the source database but it has not been installed in target database which was causing the error.
Solution
1. The errors can be ignored. Then check for the data in target database after import.
- OR -
2. First install Oracle Text in target database (refer toHP Action 2003133 -- Checked for relevance Note 979705.1) and then perform the import.
```
```
ISSUES:
ORA-39083 and ORA-02304 during impdp or imp
ORA-39083: Object type TYPE failed to create with error:
ORA-02304: invalid object identifier literal
Failing sql is:
CREATE TYPE "MSHIST"."SAMPLE_TYPE"   OID '0E27E0EF55D546ED8385F49E80C687F6' IS OBJECT (     measureOid         NUMBER  ,       sampleTime         TIMESTAMP ,     sampleSMH          NUMBER ,          sampleValue        FLOAT   ,       fmiValue           NUMBER      ) 
Cause : Oracle was trying to import an object with OID which already existed in database. Since i was copying the user SQL1 to user SQL1_AUX no surprise that the OID already existed. And OID should be unique in a database.
Solution 1 : I added the clause transform=OID:n in my impdp command.
Processing object type SCHEMA_EXPORT/TABLE/TABLE
USERID="/ as sysdba"
CONTENT=ALL
DIRECTORY=UP_MIG
JOB_NAME=IMPDIR_JOB_01
LOGFILE=imp_all_2.log
DUMPFILE=exp_source_%U.dmp
PARALLEL=3
TRACE=480300
METRICS=y
TRANSFORM=oid:n

```
```
------------------------------------------------------------------------------------------------------------------------------------------
ERROR at line 1: ORA-00054: resource busy and acquire with NOWAIT specified
-----------------------------------------------------------------------------------------------------------------------------------------

execute the below query to find out the session which locked the target table

select
c.object_name,
b.sid,
b.serial#,
b.machine
from
v$locked_object a ,
v$session b,
dba_objects c
where
b.sid = a.session_id
and
a.object_id = c.object_id;

alter system kill session '11,37157';
```
```
--------------------------------------------------------------------------------------------------------------------------------------
IMP-00032;IMP-00008  During Import process if you will get below error message 
IMP-00032: SQL statement exceeded buffer length
IMP-00008: unrecognized statement in the export file: 
---------------------------------------------------------------------------------------------------------------------------------------
Solutions

use BUFFER parameter with import command.
Note: use MAXIMUM value for BUFFER parameters eg: BUFFER=1000000
```
```
Export> start_job
ORA-39004: invalid state
ORA-39016: Operation not supported when job is in EXECUTING state.

Export> status
Export> CONTINUE_CLIENT
. . exported "ODR_SM"."RESULT"                           1.355 GB 30889100 rows
. . exported "ODR_MQ2"."QUALITY"                         3.176 GB 59310305 rows
. . exported "ODR_DAS"."TB_DOWNTIMEHISTORICAL"           1.276 GB 7064200 rows
. . exported "DW_LND"."ICC_DELAY"                        963.0 MB 7355408 rows
. . exported "ODR_LEICA"."SHIFT_HAULS"                   887.5 MB 8099096 rows
```

```
Processing object type SCHEMA_EXPORT/TABLE/TABLE
ORA-39121: Table "BH_OWNER"."BH_ASSAY_RESULT_2" can't be replaced, data will be skipped. Failing error is:
ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired
ORA-00955: name is already used by an existing object
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Processing object type SCHEMA_EXPORT/TABLE/GRANT/OWNER_GRANT/OBJECT_GRANT
ORA-39112: Dependent object type OBJECT_GRANT:"BH_OWNER" skipped, base object type TABLE:"BH_OWNER"."BH_ASSAY_RESULT" creation failed
Processing object type SCHEMA_EXPORT/TABLE/INDEX/INDEX
ORA-39112: Dependent object type INDEX:"BH_OWNER"."IDX_ASSAY_RESULT_STATUS" skipped, base object type TABLE:"BH_OWNER"."BH_ASSAY_RESULT" creation failed
ORA-39112: Dependent object type INDEX:"BH_OWNER"."PK_BH_ASSAY_RESULT" skipped, base object type TABLE:"BH_OWNER"."BH_ASSAY_RESULT" creation failed

SQL> select object_id, object_name from dba_objects where object_name in ('BH_ASSAY_RESULT_2') and owner='BH_OWNER';

 OBJECT_ID        OBJECT_NAME
--------------------------------
     86219        BH_ASSAY_RESULT_2

SQL> select session_id, locked_mode, object_id from v$locked_object where object_id=86219;

SESSION_ID LOCKED_MODE  OBJECT_ID
---------- ----------- ----------
       166           6      86219
```
```
### UDE-00010: multiple job modes requested, schema and tables. 
Remove SCHEMA parameter and change tables as tables=HRI.TBL_BATCH,HRI.TBL_BATCHDATA,HRI.TBL_FILEHANDLER
EXPORT DUMP is corrupted
+++++++++++++++++++++++++++++
>impdp system/system_bhp parfile=imp.par
With the Partitioning, OLAP, Data Mining and Real Application Testing options
ORA-39002: invalid operation
ORA-31694: master table "SYSTEM"."SYS_IMPORT_SCHEMA_01" failed to load/unload
ORA-31640: unable to open dump file "G:\export\BLASTHOLES.DMP" for read
ORA-19505: failed to identify file "G:\export\BLASTHOLES.DMP"
ORA-27046: file size is not a multiple of logical block size
OSD-04012: file size mismatch (OS 1764408925)

>impdp system/system_bhp parfile=imp.par full=y

DataPump Import (IMPDP) Fails With Errors ORA-39002 ORA-31694 ORA-31640 ORA-19505 ORA-27046 (Doc ID 785473.1)

UDE-00008: operation generated ORACLE error 39078 
ORA-39078: unable to dequeue message for agent KUPC$A_1_20060706172828 from queue "KUPC$S_1_20060706172821" 
ORA-06512: at "SYS.DBMS_DATAPUMP", line 2745 
ORA-06512: at "SYS.DBMS_DATAPUMP", line 3712 
ORA-06512: at line 1 

problem can be related with STREAMS_POOL_SIZE. DataPump uses streams to generate the export. If the STREAMS_POOL_SIZE is too small, then it will report that error. 
connect / as sysdba alter system set STREAMS_POOL_SIZE=100M scope=spfile;
shutdown
startup  
```

## Clean old exp jobs 
```
SQL> select owner_name,job_name,operation,job_mode,state,attached_sessions from dba_datapump_jobs;

OWNER_NAME     JOB_NAME OPERATION      JOB_MODE  STATE     ATTACHED_SESSIONS
SYSTEM    SYS_IMPORT_FULL_01  IMPORT                         FULL  EXECUTING     1

cleanup orphaned datapump jobs we perform the following steps.

1.)  Check the orphaned datapump jobs.
SQL>select owner_name,job_name,operation,job_mode,state,attached_sessions from dba_datapump_jobs; 
OWNER_NAME     JOB_NAME  OPERATION         JOB_MODE             ATE                          ATTACHED_SESSIONS
------------------------------ ------------------------------ ------------------------------ ------------------------------ ------------------------------ 
SYSTEM                _EXPORT_SCHEMA_01                    EXPORT                  SCHEMA                 NOT RUNNING      0                 
SYSTEM           	     SYS_EXPORT_SCHEMA_03           EXPORT                   CHEMA                 NOT RUNNING       0                        
 SYSTEM                  EXPORT_SCHEMA_02                   EXPORT                  SCHEMA                NOT RUNNING     0                         
2.)  Check the status of  "state"  field

For orphaned jobs the state will be NOT RUNNING. So from the output we can say all the three are orphaned jobs. 
3.)  Drop the master table  
Since  the  above  jobs  are  stopped  or  not running  won't  be  restarted  anymore,  so  drop  the master table. The master  tables  above  are  SYS_EXPORT_SCHEMA_01,   SYS_EXPORT_SCHEMA_03, SYS_EXPORT_SCHEMA_02) .
SQL> drop table  system.SYS_EXPORT_SCHEMA_03 ; 
Table dropped.
SQL> drop table  system.SYS_EXPORT_SCHEMA_01 ; 
Table dropped.
SQL> drop table  system.SYS_EXPORT_SCHEMA_02 ; 
Table dropped.

4.) Check  for  existing  data  pump  jobs 
Now check the existing datapump job by  query  issued  in  step 1.  If  objects  are  in  recyclebin  then purge the objects from the recyclebin. 

SQL> SELECT owner_name, job_name, operation, job_mode, state, attached_sessions from dba_datapump_jobs;
No row selected
SQL> purge table system.SYS_EXPORT_SCHEMA_01;
Table purged.
SQL> purge table system.SYS_EXPORT_SCHEMA_02;
Table purged
SQL> purge table system.SYS_EXPORT_SCHEMA_03;
Table purged  
```

## export a package and a procedure (Metadata) 
```
SET lines 100 
COL privilege FOR a50 
SELECT grantee, granted_role, default_role FROM dba_role_privs 
WHERE granted_role IN ('DBA', 'EXP_FULL_DATABASE', 'IMP_FULL_DATABASE') 
ORDER BY 1,2;
```

NOTE: You can use DBMS_METADATA to extract the code for stored procedures/packages/functions. It is available in 9i and 10g, though expect bugs. Nothing that crashes a database, but it might not output the code exactly the way you want it. Test and make slight changes if you need. The code below is a subset of a script I use to extract all the ddl from a schema. Use with care and change what you need. 

```
SET LINESIZE 132 PAGESIZE 0 FEEDBACK off VERIFY off TRIMSPOOL on LONG 1000000 
COLUMN ddl_string FORMAT A100 WORD_WRAP

EXECUTE DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM,'PRETTY',true); 
EXECUTE DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM,'SQLTERMINATOR',true); 
```
```
select dbms_metadata.GET_DDL('PACKAGE BODY',&object_name)
from user_objects 
where object_type = 'PACKAGE BODY';
```

## getcode_all.sql
```
set heading off
set feedback off
set linesize 100
set trimspool on
spool getcode_all.out
select [1]'@getcode ' || object_name
from user_objects
where object_type in ( 'PROCEDURE', 'FUNCTION', 'PACKAGE' )
/
spool off
```
```
set heading on
set feedback on
set linesize 130
set termout on
@getcode_all.out
```

## getcode.sql
```
set feedback off
set heading off
set termout off
set linesize 1000
set trimspool on
set verify off
spool &1..sql
prompt set define off
prompt set echo   off
select decode( type||'-'||to_char(line,'fm99999'),
               'PACKAGE BODY-1', '/'||chr(10),
                null) ||
       decode(line,1,'create or replace ', '' ) ||
       text text
  from user_source
 where name = upper('&&1')
 order by type, line;
prompt /
prompt set define on
prompt set echo   on
spool off
set feedback on
set heading on
set termout on
set linesize 100

```

Information 
	EXP/IMP			Datapump
	owner			schema
	file			dumpfile
	log			logfile/nologfile
	fromuser,touser		remap_schema; remap table
SQL fileimp --indexfile=index.sql	impdp sqlfile=ddl.sql
```	
select 'exp system/<>@sid tables='|| owner||'.'||segment_name ||' file=/INTMAINT/datapump/'||segment_name||'.dmp log=/INTMAINT/datapump/'||segment_name||'.log statistics=none ;'
from dba_segments
where owner = 'INTERFACE'
and segment_type = 'TABLE'
order by bytes;
```
```
select 'expdp  system/manager schemas='||username||
  ' directory=dump_dir dumpfile='||username||'.dmp logfile=expdp'
 ||username||'.log'
 from dba_users order by; select 'expdp system/g1nt0nic tables='|| owner||'.'||segment_name ||' dumpfile='||segment_name||'.dmp logfile='||segment_name||'.log directory=dpm;'
from dba_segments
where owner = 'QHUB'
and segment_type = 'TABLE'
order by bytes;
```

## 
Overide dumpfile		reuse_dumpfiles=Y
consistent=y	FLASHBACK_SCN or FLASHBACK_TIME
example: flashback_time="TO_TIMESTAMP('2015-09-23 10:43:00', 'YYYY-MM-DD HH24:MI:SS')"
RAC		expdp system full=y directory=monoexp dumpfile=pprod16fulldb.dmp logfile=pprodexpfull.log content=All CLUSTER=N reuse_dumpfiles=y

## EXADATA	Export/Import of HCC Tables
```
HCC tables can be exported using the regular expdp utility, which preserves the table properties. When imported in a Exadata
machine, the tables automatically get compressed. If the import occurs at a non-Exadata machine, the import will fail with the
following error:
ORA-64307: Exadata Hybrid Columnar Compression is not supported for tablespaces on this storage type.
Cause: An attempt was made to use Exadata Hybrid Columnar Compression on unsupported storage.
Action: Create this table in a tablespace residing on Oracle Exadata, Oracle’s Sun ZFS or Pillar Axiom storage or use a
different compression type.
To override this, the table needs to be imported as an uncompressed table using the TRANSFORM:SEGMENT_ATTRIBUTES=n
option clause of the impdp command.
An uncompressed or OLTP-compressed table can be converted to Exadata HCC format during import. To convert a nonExadata
HCC table to an Exadata HCC table, do the following:
1. Specify default compression for the tablespace using the ALTER TABLESPACE … SET DEFAULT COMPRESS
command.
2. Override the SEGMENT_ATTRIBUTES option of the imported table during import.
```
## Compression of Dump File Sets
```
COMPRESSION
	ALL - Enables compression for the entire operation.
  metadata_only - The default setting. Causes only the metadata to be compressed.
  data_only - Only the data being written to the dump file set will be compressed.
  None - No compression will take place. 
partition_options	None - Tables will be imported such that they will look like those on the system on which the export was created.
Departition - Partitions will be created as individual tables rather than partitions of a partitioned table.
Merge - Causes all partitions to be merged into one, unpartitioned table.
```
## TABLE_EXISTS_ACTION 	
```
1.) SKIP  : leaves the table as is and moves on to the next object. This is not a valid option if the CONTENT parameter is set to DATA_ONLY.By default the value is SKIP . 
[ignore: ORA-39151:]
2.) APPEND loads rows from the source and leaves existing rows unchanged.[ignore: ORA-39151:]
3.) TRUNCATE deletes existing rows and then loads rows from the source.[ignore: ORA-39151:]
4.) REPLACE drops the existing table and then creates and loads it from the source. This is not a valid option if the CONTENT parameter is set to DATA_ONLY.
Hence, table_exists_action=replace parameter internally drop and recreate the table .Hence all the existing metadata also  get dropped and is recreated .
```

expdp system/system_bhp directory=expmono1 full=y filesize=10G dumpfile=pprod2ofull%U.dmp EXCLUDE=STATISTICS parallel=5 CONTENT=METADATA_ONLY logfile=exppprod20.log

## Export.sh
```
#!/bin/bash
ORACLE_HOME=/app/pprod26/product/11.2.0.2/db_1;export ORACLE_HOME
PATH=$ORACLE_HOME/bin:$PATH;export PATH
ORACLE_SID=pprod262;export ORACLE_SID
DIRECTORY='expmono';export DIRECTORY

--
-- Shows dba_directories
--
set pause off
set pagesize 9999
set linesize 128
set feedback off
set echo off
set verify off

column OWNER format a15 head "Owner"
column DIRECTORY_NAME format a22 head "Name"
column DIRECTORY_PATH format a80 head "Path"

select OWNER
      ,DIRECTORY_NAME
      ,DIRECTORY_PATH
from DBA_directories
order by OWNER,DIRECTORY_NAME;
```

Trace file:  using trace=480300 in the expdp/impdp command
Data Pump Export (expdp) and Data Pump Import (impdp) are server-based rather than client-based as is the case for the original export (exp) and import (imp). 
Because of this, dump files, log files, and sql files are accessed relative to the server-based directory paths.
Data Pump requires that directory objects mapped a file system directory be specified in the invocation of the data pump import or export. 

Data Pump and Partitioned Tables
impdp quest/quest DIRECTORY=quest_pump_dir DUMPFILE=quest.dmp tables=QUEST.NAMES partition_options=merge
---------------------------------------------
## Datapump - on fly feature
---------------------------------------------
```
expdp.par
USERID="/ as sysdba"
tables=ellipse.msf010
exclude=INDEX
table_exists_action=TRUNCATE
remap_tablespace=VICCNV_TABMSF:TAB_ELLIPSE
remap_tablespace=VICCNV_LOBMSF:TAB_ELLIPSE

1. create or replace directory impdir as '/dbuat01/oracle/viccnv1_export/dp';
2. Create a database link 'viccnv1.vicroads.com.au'
3. Set the env variables for the database to which the tables are imported
4. Start the import

impdp parfile=expdp.par parallel=4 directory=impdir logfile=imp.log network_link=viccnv1.vicroads.com.au
```
--------------------------------------
## Export Using GZIP
--------------------------------------
```
cat exp.par
userid="/ as sysdba"
file=./gzip_pipe
compress=n
direct=n
consistent=y
buffer=1000000
log=exp.log
tables=ellipse.msf203
statistics=none
query="where dstrct_code = '6030'
and supplier_no = '012762'"

cat exp.notes              [you can check /etc/mknod and run /etc/mknod gzip_pipe p   or check /bin/mknod]

mknod gzip_pipe p
gzip -c < gzip_pipe > ./dmp.gz &
nohup exp parfile=./exp.par &

[1.  mknod gzip_pipe
df  -k p
2.  gzip -c < gzip_pipe > ./dmp.gz &
3.  nohup exp parfile=./exp.par &

userid="/ as sysdba"
file=./gzip_pipe
buffer=10000000
log=exp.log
owner=ellipse
statistics=none
]

IMP:
-----------------
/etc/mknod gzip_pipe p
nohup gunzip -c ./dmp.gz > ./gzip_pipe &
nohup imp parfile=./imp.par &

vi imp.par

userid=system/ma1c0440
file=./gzip_pipe
fromuser=ELLIPSE
touser=ELLIPSE
buffer=100000000
ignore=y
log=imp_ellipse.log
feedback=1000000
commit=y

shell script   01_imp.ksh

-r-x------   1 oracle   dba          325 Dec  4  2007 05_imp.ksh

=============================================================================
> more imp_mono.ksh

mkfifo imp.dat
gunzip -c /oracle/admin/v6000/adhoc/sc2516247/expt053.dmp.gz | dd of=imp.dat &
imp \"/ as sysdba\" buffer=10000000 fromuser='S053' touser='S119' log=imp.log file=imp.dat statistics=none indexes=n ignore=y
rm imp.dat
=============================================================================
userid=system/aFrkovua 
file=exp.dmp 
log=exp.log 
tables=(tab1,tab2,tab3) 
direct=y
statistics=none
query="where entry_type ='G' and entity in ('HRASSIST','HRSYS','PAYROLL')"
```

## TRIGGERS
-------------------------------
```
export dump file will not have triggers.

triggers=N -- is an option which should be given for export utility and not for import utility
```
+++++++++++++++++++++++++++++++++++
## Without Index and then Index
+++++++++++++++++++++++++++++++++++
```
more imp.par
-------------------------
userid=system
log=imp.log
file=/oradump/export/rn01tst/rnw.dmp
feedback=100000
buffer=2000000
full=yes
statistics=none
indexes=n

more imp_indexfile.par
----------------------------------------
userid=system
log=imp_indexfile.log
file=/oradump/export/rn01tst/rnw.dmp
buffer=2000000
full=yes
indexfile=index_create.sql
```

## INCLUDE or EXCLUDE parameter in a parameter file is the preferred method.
-------------------------------------------------------------------------------------------------------------------------------
```
Parameter file: exp.par 
-------------------------
DIRECTORY = my_dir 
DUMPFILE  = exp_tab.dmp 
LOGFILE   = exp_tab.log 
SCHEMAS   = scott 
INCLUDE   = TABLE:"IN ('EMP', 'DEPT')" 

INCLUDE=TABLE:"IN ('EMP', 'DEPT')" 

or (all tables that have an 'E' and a 'P' in their name): 

INCLUDE=TABLE:"LIKE '%E%'" 
INCLUDE=TABLE:"LIKE '%P%'" 
EXCLUDE=CONSTRAINT
EXCLUDE=SCHEMA:"='SCOTT'"
exclude=INDEX:\"LIKE \'\%\'\" 
EXCLUDE=TABLE:\IN \(\’EMP\’,\’DEPT\’\)\
################################################
QUERY in Parameter file.
File: expdp_q.par
####################################################
DIRECTORY = my_dir
DUMPFILE  = exp_query.dmp
LOGFILE   = exp_query.log
SCHEMAS   = hr, scott
INCLUDE   = TABLE:"IN ('EMP', 'DEPARTMENTS')"
QUERY     = scott.emp:"WHERE job = 'ANALYST' OR sal >= 3000"
# place following 3 lines on one single line:
QUERY     = hr.departments:"WHERE department_id IN (SELECT DISTINCT
department_id FROM hr.employees e, hr.jobs j WHERE e.job_id=j.job_id
AND UPPER(j.job_title) = 'ANALYST' OR e.salary >= 3000)"

-- Run Export DataPump job:

%expdp system/manager parfile=expdp_q.par
```

################################################################
## QUERY on Command line.
###############################################################
```
table scott.dept; and 
from table scott.emp all employees whose name starts with an 'A' 
-- Example Windows platforms:
-- Note that the double quote character needs to be 'escaped'
-- Place following statement on one single line:

D:\> expdp scott/tiger DIRECTORY=my_dir DUMPFILE=expdp_q.dmp LOGFILE=expdp_q.log TABLES=emp,dept QUERY=emp:\"WHERE ename LIKE 'A%'\"

-- Example Unix platforms:
-- Note that all special characters need to be 'escaped'

% expdp scott/tiger DIRECTORY=my_dir \ 
DUMPFILE=expdp_q.dmp 
LOGFILE=expdp_q.log TABLES=emp,dept \
QUERY=emp:\"WHERE ename LIKE \'A\%\'\"

-- Example VMS platform:
-- Using three double-quote characters

$ expdp scott/tiger DIRECTORY=my_dir -
DUMPFILE=exp_cmd.dmp LOGFILE=exp_cmd.log TABLES=emp,dept -
QUERY=emp:"""WHERE ename LIKE 'A%'"""

-- In target database: 
-- Import all employees of department 10:

%impdp scott/tiger DIRECTORY=my_dir DUMPFILE=expdp_s.dmp \ 
LOGFILE=impdp_s.log TABLES=emp TABLE_EXISTS_ACTION=append \ 
QUERY=\"WHERE deptno = 10\" CONTENT=data_only

15 minutes ago, 
expdp system dumpfile=expdp_tuning.dmp logfile=expdp_tuning.log directory=backup reuse_dumpfiles=Y compression=ALL schemas=TUNING flashback_time=to_timestamp\(sysdate-15/1440\)

FLASHBACK_TIME="TO_TIMESTAMP(TO_CHAR(SYSDATE,'YYYY-MM-DD HH24:MI:SS'),'YYYY-MM-DD HH24:MI:SS')"

SQL
----------
export ORACLE_HOME=/u01/app/oracle/product/10.2.0 
export ORACLE_SID=PALLADIUM
export PATH=$ORACLE_HOME/bin:$PATH
export USER=MYSELF
export USER_PWD=SECRET
export directory=EXPORT
export dumpfile=expdp_PALLADIUM.dmp
export BU_DATE=`date +%Y%m%d`
export logfile=expdp_PALLADIUM_$BU_DATE.dmp.log
mv /u01/rman/PALLADIUM_EXPDP/$dumpfile /u01/rman/PALLADIUM_EXPDP/$dumpfile.old
$ORACLE_HOME/bin/sqlplus -s $USER/$USER_PWD <<EOF
set numwidth 16
set linesize 200
set head off
set feedback off
set pagesize0
spool /home/oracle/scripts/expdp_PALLADIUM.par
select 'directory=$directory         dumpfile=$dumpfile         logfile=$logfile         flashback_scn='||current_scn||q'! 
        full=y 
        exclude=schema:"='SYSMAN'"!' from v\$database;
spool off
exit
EOF
$ORACLE_HOME/bin/expdp $USER/$USER_PWD parfile=/home/oracle/scripts/expdp_PALLADIUM.par
[oracle: ~]$impdp system/g1nt0nic directory=dpump dumpfile=SBPELADMIN.dmp sqlfile=bpelscript.sql
```

## CONTENTSIZE
```
exp user/pwd  file=exp.dmp log=exp.log tables=owner1.tablename statistics=none
imp user/pwd file=exp.dmp fromuser=owner1 touser=owner2 show=y log=show.log

Estimate Parameter in Data Pump Export 
=================================
ESTIMATE  parameter  of data pump  export specify the method that Export will use to estimate how much disk space  each  table in the export  job will  consume (in bytes)  before  performing actual data pump export operation . The ESTIMATE parameter can take two parameters. Either BLOCKS (default) or STATISTICS. The meaning of these two parameter values are specified below.

BLOCKS : The estimate is calculated by multiplying the number of database blocks used by the target objects with the appropriate block sizes.
STATISTICS : The estimate is calculated using statistics for each table. So to be accurate you must analyze table recently.

Note that the outcome specified by ESTIMATE=BLOCKS is far away from the size of the actual dumpfile. In fact, ESTIMATE=BLOCKS method generates more inaccurate result from dump file size when,

a) The table was created with a much bigger initial extent size than was needed for the actual table data.
b) Many rows have been deleted from the table, or a very small percentage of each block is used.
The outcome generated by ESTIMATE=STATISTICS is most accurate to dump file size if recently table is analyzed.

without  actually   exporting  any data, we  can use the ESTIMATE_ONLY=y 

expdp system@migdb directory=exp_dir schemas=sh  ESTIMATE_ONLY=y    

expdp system@migdb directory=exp_dir schemas=sh dumpfile=fbtst_sh.dmp2 logfile=fbexptxt.log ESTIMATE=BLOCKS
expdp system@migdb directory=exp_dir schemas=sh dumpfile=fbtst_sh.dmp2 logfile=fbexptxt.log ESTIMATE=STATISTICS
```

## export only user's synonyms 
---------------------------------------------------
```
select 'create synonym ' || synonym_name || ' for ' || table_owner || '.' || table_name || decode(db_link, null, null, '@' || db_link) || ';'
from user_synonyms 
  
EXP only captures public synonyms in full export mode (full=y), not schema-level, or table level mode.
---DB LINKS
select 'CREATE '|| decode(owner,'PUBLIC','PUBLIC ',null) ||
'SYNONYM ' || decode(owner,'PUBLIC',null, owner || '.') ||
lower(synonym_name) || ' FOR ' || lower(table_owner) ||
'.' || lower(table_name) ||
decode(db_link,null,null,'@'||db_link) || ';'
from sys.dba_synonyms
where table_owner != 'SYS'
order by owner
/

expdp system/manager DIRECTORY=my_dir DUMPFILE=exp_tab.dmp  LOGFILE=exp_tab.log SCHEMAS=scott INCLUDE=SYNONYM

INCLUDE=SYNONYM:IN(SELECT synonym_name FROM dba_synonyms WHERE owner = ‘PUBLIC’ and table_owner=’TEST’)
=====================================================================
Instead of using export/import for re-creating your PUBLIC synonyms,  the DDL statements to create the PUBLIC synonyms and then run that DDL on the development database? 

The following SQL can be run on your production database. 
SPOOL create_pub_syn.sql
SELECT 'CREATE PUBLIC SYNONYM '||synonym_name||
 ' FOR '||table_owner||'.'||table_name||';'
FROM dba_synonyms
WHERE owner='PUBLIC'
AND WHERE TABLE_OWNER IN ([list of schemas]);
SPOOL OFF

set linesize 256;
spool oracleSynonyms.sql;
select 'create or replace '|| decode(owner,'PUBLIC','public ',null) ||  
'synonym ' || decode(owner,'PUBLIC',null, lower(owner) || '.') ||  
lower(synonym_name) || ' for ' || lower(table_owner) || '.' || lower(table_name) || decode(db_link,null,null,'@'||db_link) || ';'
from sys.dba_synonyms 
where table_owner not in('SI_INFORMTN_SCHEMA','SYS','SYSTEM','ORDSYS','XDB','CTXSYS','DMSYS','EXFSYS','MDSYS','SYSMAN','WKSYS','WMSYS')
order by owner, table_name;
spool off;
```

## ORA-31693, ORA-02354 and ORA-01555 with Export Datapump (Doc ID 1580798.1)
```
select COLUMN_NAME,SECUREFILE,PCTVERSION,RETENTION from dba_lobs 
where OWNER=upper('&OWNER') 
and TABLE_NAME=upper('&TABLE_NAME') ;

ORA-31693: Table data object "ODR_RH"."OP_WEIGHT_UNDERCARRIAGE" failed to load/unload and is being skipped due to error:
ORA-02354: error in exporting/importing data
ORA-01555: snapshot too old: rollback segment number 7 with name "_SYSSMU7_2494227896$" too small

KEEP_MASTER=y to you datapump command then the master table where datapump tackes all of its metadata is retained
query this table to get the elapsed runtimes - this isn;t stored in the simplest format to query but a simple analytic function gives you what you want (in this example the master table is called OPS$ORACLE.SYS_EXPORT_SCHEMA_03)

select tab_owner, tab_name, completed_rows, tab_size, elapsed_time 
from (select base_object_schema, 
object_name, 
(lag(object_name, 1) over(order by process_order)) as TAB_NAME, 
(lag(base_object_schema, 1) over(order by process_order)) as TAB_OWNER, 
completed_rows, 
(dump_length / (1024 * 1024 * 1024)) || 'GB' TAB_SIZE, 
elapsed_time 
from ops$oracle.sys_export_schema_03 
where object_type_path = 'SCHEMA_EXPORT/TABLE/TABLE_DATA' 
and process_order >= 1) 
where object_name is null

Flashback_Scn  and  flashback_Time  are  two  important  feature  of  the  datapump 11g . If  we  want  to  run  a  large  export  whilst  the  database  is  in  use  then  ideally  we  should  always use  one  of  the  two  flashback  parameters. The export  operation  is  performed  with  data  that is  consistent  as  of  the  specified  SCN .  FLASHBACK_SCN and FLASHBACK_TIME are mutually exclusive .

FLASHBACK_TIME : The SCN that most closely matches the specified time is found, and this SCN is used to enable the Flashback utility. The export operation is performed with data that is consistent as of this SCN. The FLASHBACK_SCN parameter pertains only to the Flashback Query capability of Oracle Database. It is not applicable to Flashback Database, Flashback Drop, or Flashback Data Archive. We can get the scn number from the following query

SQL> select current_scn from v$database ;       or
SQL>select dbms_flashback.get_system_change_number from dual ;
SQL> conn / as sysdba
Connected.
SQL> create directory exp_dir as '/EOYFIN/TESTP/';

Directory created.
SQL> grant all on directory exp_dir to system;

Grant succeeded.

SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    1386538

SQL> conn sh/sh
Connected.
SQL> create table test as select * from sh.SALES;

Table created.

SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    1386808

SQL>  alter system set db_recovery_file_dest='/EOYFIN/TESTP/';

System altered.

SQL> alter database flashback on;

Database altered.

SQL> select dbms_flashback.get_system_change_number from dual ;

GET_SYSTEM_CHANGE_NUMBER
---------------------------------------------------------
                 1386913

!expdp system/oracle directory=exp_dir schemas=SH dumpfile=FlashDpmp.dmp logfile=FlashDpmp.log flashback_scn=1386808 reuse_dumpfiles=y
```
  
## Progress 

```
select sid, serial#, sofar, totalwork, dp.owner_name, dp.state, dp.job_mode
from gv$session_longops sl, gv$datapump_job dp
where sl.opname = dp.job_name and sofar != totalwork;
select x.job_name,b.state,b.job_mode,b.degree
, x.owner_name,z.sql_text, p.message
, p.totalwork, p.sofar
, round((p.sofar/p.totalwork)*100,2) done
, p.time_remaining
from dba_datapump_jobs b 
left join dba_datapump_sessions x on (x.job_name = b.job_name)
left join v$session y on (y.saddr = x.saddr)
left join v$sql z on (y.sql_id = z.sql_id)
left join v$session_longops p ON (p.sql_id = y.sql_id)
WHERE y.module='Data Pump Worker'
AND p.time_remaining > 0;
```
```
TTITLE 'Currently Active DataPump Operations'
COL owner_name          FORMAT A06      HEADING 'Owner'
COL job_name            FORMAT A20      HEADING 'JobName'
COL operation           FORMAT A12      HEADING 'Operation'
COL job_mode            FORMAT A12      HEADING 'JobMode'
COL state               FORMAT A12      HEADING 'State'
COL degree              FORMAT 9999     HEADING 'Degr'
COL attached_sessions   FORMAT 9999     HEADING 'Sess'

SELECT 
     owner_name
    ,job_name
    ,operation
    ,job_mode
    ,state
    ,degree
    ,attached_sessions
  FROM dba_datapump_jobs
;
```
## Sample Output
```
Thu Apr 07                                                             page    1
                      Currently Active DataPump Operations

SYS    IIB_QA_SEC_INTERIM   IMPORT       FULL         EXECUTING        4     1
SYS    IIB_QA_SEC_IMP       IMPORT       FULL         NOT RUNNING      0     0

```

```
TTITLE 'Currently Active DataPump Sessions'
COL owner_name          FORMAT A06      HEADING 'Owner'
COL job_name            FORMAT A20      HEADING 'Job'
COL osuser              FORMAT A12      HEADING 'UserID'

SELECT 
     DPS.owner_name
    ,DPS.job_name
    ,S.osuser
  FROM 
     dba_datapump_sessions DPS
    ,v$session S
 WHERE S.saddr = DPS.saddr
;
```
```
Thu Apr 07                                                             page    1
                       Currently Active DataPump Sessions

SYS    IIB_QA_SEC_INTERIM   oracle
SYS    IIB_QA_SEC_INTERIM   oracle
SYS    IIB_QA_SEC_INTERIM   oracle
```
```
select
    sid,
    serial#
 from
    v$session s,
    dba_datapump_sessions d
 where
  s.saddr = d.saddr;
```
```
Thu Apr 07                                                             page    1
                       Currently Active DataPump Sessions

        60       1521
        63         71
        70         51

```
```
col table_name format a30

select substr(sql_text, instr(sql_text,'"')+1, 
instr(sql_text,'"', 1, 2)-instr(sql_text,'"')-1) 
table_name, 
rows_processed, 
round((sysdate
- to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))
*24*60, 1) minutes, 
trunc(rows_processed / 
((sysdate-to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))
*24*60)) rows_per_min 
from 
v$sqlarea 
where 
upper(sql_text) like 'INSERT % INTO "%' 
and 
command_type = 2 
and 
open_versions > 0;
```
```
Thu Apr 07                                                             page    1
                       Currently Active DataPump Sessions

WMB_BINARY_DATA                             0      195.7            0
ICCSYSDB                                    0      232.7            0

```

## Performance
```
DISK_ASYNCH_IO=TRUE
DB_BLOCK_CHECKING=FALSE
DB_BLOCK_CHECKSUM=FALSE

Additionally, the following initialization parameters must have values set high enough to allow for maximum parallelism:

PROCESSES
SESSIONS
PARALLEL_MAX_SERVERS
increase the pga_agreegate_target value to a large number may reduce time 

parallel to multiple files (export and import) ; 
pga_aggregate_target to big number might increate import jobs 

```
```
select sid, serial#, username, process, program
from v$session s, dba_datapump_sessions d
where s.saddr = d.saddr;
```
```
col table_name for a30
select substr(sql_text,instr(sql_text,' INTO '),30) table_name,
         rows_processed,
         round((sysdate-to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60,1) minutes,
         trunc(rows_processed/((sysdate-to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60)) rows_per_min   
  from   sys.v$sqlarea
  where   (ADDRESS,HASH_VALUE) in (select sql_address,sql_hash_value from v$session where sid= &sid_number)
    and  command_type = 2
    and  open_versions > 0;
```
```sample output
Enter value for sid_number: 202
old   6:   where   (ADDRESS,HASH_VALUE) in (select sql_address,sql_hash_value from v$session where sid= &sid_number)
new   6:   where   (ADDRESS,HASH_VALUE) in (select sql_address,sql_hash_value from v$session where sid= 202)

TABLE_NAME                     ROWS_PROCESSED    MINUTES ROWS_PER_MIN
------------------------------ -------------- ---------- ------------
 INTO "MSF900_I" ("DSTRCT_CODE        1966020       76.9        25571
```
## To see the wait event
```
select * from v$session_event where sid=202
```
```
SET lines 200
COL owner_name FORMAT a10;
COL job_name FORMAT a20
COL state FORMAT a11
COL operation LIKE state
COL job_mode LIKE state
-- locate Data Pump jobs:
SELECT owner_name, job_name, operation, job_mode,
state, attached_sessions
FROM dba_datapump_jobs
WHERE job_name NOT LIKE 'BIN$%'
ORDER BY 1,2;
```
## impdp/exp dp progress
```
select to_char(v$session.sid,'99999') SID,
to_char(logon_time,'DD-MON:HH24MISS') start_time,
substr(nvl(program,machine),1,15) prog_machine,
substr(opname,1,15),
substr(target,1,15),
to_char(sofar,'9999999999') completed,
to_char(totalwork,'9999999999') totalwork,
time_remaining
from v$session, v$session_longops 
where v$session.sid=v$session_longops.sid 
and sofar <> totalwork 
and program like '%exp%'
order by totalwork;
```
## Monitoring import performance
```
select 
   substr(sql_text,instr(sql_text,'into "'),30) table_name, 
   rows_processed, round((sysdate-to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60,1) minutes,
   trunc(rows_processed/((sysdate-to_date(first_load_time,'yyyy-mm-dd hh24:mi:ss'))*24*60)) rows_per_minute 
from   
   sys.v_$sqlarea 
where  
   sql_text like 'insert %into "%' and command_type = 2 and open_versions > 0; 
```
```
select b.username,a.sid,b.opname,b.target,round(b.SOFAR*100 / b.TOTALWORK,0) ||'%' as "%DONE",
b.TIME_REMAINING,to_char(b.start_time,'YYYY/MM/DD HH24:MI:SS') START_TIME
from V$SESSION_LONGOPS b,V$SESSION a
where a.sid=b.sid
order by b.SOFAR/b.TOTALWORK;
```
```sample output
USERNAME            SID OPNAME                    TARGET               %DONE    TIME_REMAINING START_TIME
--------------- ------- ------------------------- -------------------- -------- -------------- -------------------
SYSTEM               74 SYS_IMPORT_TABLE_01                            0%                      2010/12/24 11:01:56
SYSTEM               78 SYS_IMPORT_TABLE_02                            0%                      2010/12/24 11:25:52
SYS                         87 Advisor                                                             100%                  0             2010/12/23 22:00:04  
```

## RestartDataPump Job 
```
SQL> SELECT owner_name, job_name, operation, job_mode, state FROM dba_datapump_jobs;

OWNER_NAME              JOB_NAME     OPERATION                      JOB_MODE            STATE
---------------------------------------------------------------------------------------------------------------------
SYSTEM                 EXPFULL            EXPORT                          FULL                 EXECUTING

Export> kill_job
Are you sure you wish to stop this job ([yes]/no): yes

SQL> SELECT owner_name, job_name, operation, job_mode, state FROM dba_datapump_jobs;

no rows selected

expdp system attach=expfull

expdp system directory=expdir full=y job_name=expfull dumpfile=expf.dmp
Export> stop_job=immediate
Are you sure you wish to stop this job ([yes]/no): yes

SQL> /

OWNER_NAME                JOB_NAME     OPERATION                      JOB_MODE      STATE
---------------------------------------------------------------------------------------------------------------------
SYSTEM                   EXPFULL            EXPORT                         FULL                    NOT RUNNING

[oracle@csmper-cls07 monowar]$ expdp system/system_bhp attach=SYS_EXPORT_FULL_01
Export> kill_job
Are you sure you wish to stop this job ([yes]/no): yes

oracle@oraintdev:/IJISDEV2/oradata$ expdp system attach=expfull
Export> start_job=skip_current
ORA-39002: invalid operation
ORA-39032: function SKIP_CURRENT is not supported in EXPORT jobs

$ impdp attach="SYS"."EXAMIG_JOB_01"

Username: sys as sysdba
Password:

Export> status
Export> CONTINUE_CLIENT
Import> stop_job=immediate
Import> continue_client

SYS@BHQA1 SQL> SELECT owner_name, job_name, operation, job_mode, state
FROM dba_datapump_jobs;  

OWNER_NAME   	JOB_NAME    			OPERATION   JOB_MODE             STATE
------------------------------------------------------------------------------------------
SYSTEM 		SYS_IMPORT_TABLE_01 		IMPORT   TABLE             EXECUTING 
BH_OWNER 	SYS_IMPORT_TABLE_02 		IMPORT TABLE NOT RUNNING
BH_OWNER 	BIN$EZtZ6j0tBy/gUPAKHRpKFw==$0 	EXPORT SCHEMA NOT RUNNING
BH_OWNER  	BIN$EZtZ6j0yBy/gUPAKHRpKFw==$0  EXPORT SCHEMA NOT RUNNING
BH_OWNER  	SYS_IMPORT_TABLE_01  		IMPORT  TABLE NOT RUNNING
BH_OWNER 	SYS_IMPORT_TABLE_03 		IMPORT TABLE NOT RUNNING

SQL> SELECT * FROM DBA_DATAPUMP_SESSIONS ;

SYS	IIB_QA_SEC_INTERIM 1 000000007B607538 DBMS_DATAPUMP

----
```
select owner_name,job_name,operation,job_mode,state,attached_sessions
from dba_datapump_jobs;
```
```Sample output

OWNER_NAME
------------------------------------------------------------------------------------------
JOB_NAME
SYS@BHQA1 SQL> show parameter recyclebin

NAME                                 TYPE            VALUE
------------------------------------ --------------- 
recyclebin                           string          on

SYS@BHQA1 SQL> purge dba_recyclebin;
DBA Recyclebin purged.

SYS@BHQA1 SQL> select owner_name,job_name,operation,job_mode,state,attached_sessions
from dba_datapump_jobs;

no rows selected

Import> start_job
ORA-39004: invalid state
ORA-39015: job is not running

Import>status
Job: IIB_QA_SEC_INTERIM
  Operation: IMPORT
  Mode: FULL
  State: STOP PENDING
  Bytes Processed: 0
  Current Parallelism: 4
  Job Error Count: 0
  Dump File: /software/export/remap_tab_qa_intr%u.dmp
  Dump File: /software/export/remap_tab_qa_intr01.dmp
  Dump File: /software/export/remap_tab_qa_intr02.dmp
Import> stop_job=immediate
Are you sure you wish to stop this job ([yes]/no): yes
Import> start_job
or,
Import> continue_client
Job IIB_QA_SEC_IMP has been reopened at Thu Apr 7 09:05:54 2016
Restarting "SYS"."IIB_QA_SEC_IMP":  /******** AS SYSDBA parfile=imp.par
crtl+c
Import>
Import> status
or
HELP
STATUS
CONTINUE_CLIENT 
KILL_JOB 
START_JOB 
PARALLEL 
STOP_JOB 
Are you sure you wish to stop this job ([yes]/no): yes

Other useful command for interactive command for Data Pump:
HELP 
STATUS : Check the status
CONTINUE_CLIENT : Resume the job
KILL_JOB : Detach all currently attached client sessions and terminate the current job
START_JOB : Restart a stopped job to which you are attached.
PARALLEL : Increase or decrease the number of active worker processes for the current job 
STOP_JOB : Stop the current job.

$ impdp system attach=SYS_IMPORT_FULL_02

Import: Release 11.2.0.4.0 - Production on Thu Mar 10 14:47:33 2016

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.
Password:

Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Data Mining and Real Application Testing options

Job: SYS_IMPORT_FULL_02
  Owner: SYSTEM
  Operation: IMPORT
  Creator Privs: TRUE
  GUID: 2DABD0D24A46ECAFE053158CF10A268A
  Start Time: Thursday, 10 March, 2016 13:09:15
  Mode: FULL
  Instance: DIIB1_1
  Max Parallelism: 1
  EXPORT Job Parameters:
     CLIENT_COMMAND        userid=system/******** directory=exp_remap dumpfile=remap.dmp tables=ICCSYSDB.WMB_BINARY_DATA_OR parallel=3
  IMPORT Job Parameters:
  Parameter Name      Parameter Value:
     CLIENT_COMMAND        userid=system/******** directory=exp_remap dumpfile=remap.dmp REMAP_TABLE=ICCSYSDB.WMB_BINARY_DATA_OR:WMB_BINARY_DATA TABLE_EXISTS_ACTION=APPEND
     TABLE_EXISTS_ACTION   APPEND
  State: EXECUTING
  Bytes Processed: 0
  Current Parallelism: 1
  Job Error Count: 0
  Dump File: /iorsdb_gg_migration/TIIB1/remap.dmp

Worker 1 Status:
  Process Name: DW00
  State: EXECUTING
  Object Schema: ICCSYSDB
  Object Name: WMB_BINARY_DATA_OR
  Object Type: TABLE_EXPORT/TABLE/TABLE_DATA
  Completed Objects: 1
  Completed Rows: 287,922
  Completed Bytes: 6,806,777,136
  Percent Done: 90
  Worker Parallelism: 1

```


## EXPORT  [database_exportdp.sh]

```
#!/bin/sh
case "$PATH" in
        */usr/local/bin*)       ;;
        *:)                     PATH=${PATH}/usr/local/bin   ;;
        "")                     PATH=/usr/local/bin          ;;
        *)                      PATH=${PATH}:/usr/local/bin  ;;
esac

export PATH

if [ "$1" ] ; then
        ORACLE_SID=$1
        export ORACLE_SID
else
        echo "ERROR: Database name not passed as parameter to io1docpr_export_backup.sh ... Aborting"
        exit 1
fi

ORAENV_ASK=NO
export ORAENV_ASK

. /usr/local/bin/oraenv

CURDATE=`date '+%Y%m%d%H%M'`                                ;export CURDATE
#BACKUP_ROOT=/app/io1docpr/oradump/
BKPLOG=/app/io1docpr/oradump/io1docpr_export_log_${CURDATE}   ;export BKPLOG
MAILLIST="monowar.mukul@bhpbilliton.com"                    ;export MAILLIST
MAILHEADER="`uname -n` BHPBIO_DISK export BACKUP ${ORACLE_SID} `date '+%a %h %d'`"     ;export MAILHEADER


#       -----------------------------------------------------------------------
#       Perform a consistent full database export
#       -----------------------------------------------------------------------

#rm -f $BACKUP_ROOT/${ORACLE_SID}_pipe
#/bin/mknod $BACKUP_ROOT/${ORACLE_SID}_pipe p
#gzip -c >$BACKUP_ROOT/${ORACLE_SID}.dmp.gz <$BACKUP_ROOT/${ORACLE_SID}_pipe &

expdp system/io1docproper dumpfile=io1docpr_$CURDATE%U.dmp directory=exportdump filesize=6g Full=y logfile=io1docpr_$CURDATE.log



#       -----------------------------------------------------------------------
#       Cleanup log files from this job (keep logs from the last 5 weeks)
#       and exit
#       -----------------------------------------------------------------------

echo "***************************************************************"
echo
echo "Cleaning up log files from database_export.sh"
echo

#find /app/io1docpr/oradump/exportdump/ -name "io1docpr_*.dmp" -type f -mtime +1 -ls -exec rm  {} \;
#find /app/io1docpr/oradump/exportdump/ -name "io1docpr_*.log" -type f -mtime +2 -ls -exec rm  {} \;

echo
echo "ORACLE EXPORT Finished `date`"
echo
echo "********************* END OF JOB ******************************"

exit
```


