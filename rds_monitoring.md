Common Oracle DBA Tasks on Amazon RDS | In One Page!
In Amazon RDS you are not allowed to execute the following commands:
- ALTER DATABASE
- ALTER SYSTEM
- GRANT ANY ROLE/PRIVILEGE
- DROP/CREATE ANY DIRECTORY 
 
This is because AWS doesn't want you to mess up with their DB instance --which they manage, instead; they provide packages under rdsadmin schema to help you execute the DBA common tasks without the need to execute the above mentioned commands.
 
In this post I'll explain all the common admin commands that you will use frequently when managing Oracle on RDS.

Before I start please note that the commands in navy color represent the native Oracle commands --just thought to included them to make this article coherent and handy for you, and the commands in green color are the new RDS related ones.

Sessions Management:

-- Check the active Sessions:
SQL> select
substr(s.INST_ID||'|'||s.USERNAME||'| '||s.sid||','||s.serial#||' |'||substr(s.MACHINE,1,22)||'|'||substr(s.MODULE,1,18),1,69)"INS|USER|SID,SER|MACHIN|MODUL"
,substr(s.status||'|'||round(w.WAIT_TIME_MICRO/1000000)||'|'||LAST_CALL_ET||'|'||to_char(LOGON_TIME,'ddMon HH24:MI'),1,40) "ST|WAITD|ACT_SINC|LOGIN"
,substr(w.event,1,24) "EVENT"
,s.SQL_ID||'|'||round(w.TIME_REMAINING_MICRO/1000000) "CURRENT SQL|REMIN_SEC"
,s.FINAL_BLOCKING_INSTANCE||'|'||s.FINAL_BLOCKING_SESSION "I|BLK_BY"
from     gv$session s, gv$session_wait w where s.USERNAME is not null and s.sid=w.sid and s.STATUS='ACTIVE'
--and    w.EVENT NOT IN ('SQL*Net message from client','class slave wait','Streams AQ: waiting for messages in the queue','Streams capture: waiting for archive log','Streams AQ: waiting for time management or cleanup tasks','PL/SQL lock timer','rdbms ipc message')
order by "I|BLK_BY" desc,w.event,"INS|USER|SID,SER|MACHIN|MODUL","ST|WAITD|ACT_SINC|LOGIN" desc,"CURRENT SQL|REMIN_SEC";

-- Disconnect a session:
SQL> EXEC rdsadmin.rdsadmin_util.disconnect(<SID>, <SERIAL#>, 'IMMEDIATE');

-- Cancel Statement: [18c+]
 SQL> EXEC rdsadmin.rdsadmin_util.cancel(<SID>, <SERIAL#>, '<SQLID>');

-- Enable Restricted Sessions:
SQL> EXEC rdsadmin.rdsadmin_util.restricted_session(p_enable => true);
          select logins from v$instance;

-- Disable Restricted Sessions:
SQL> EXEC rdsadmin.rdsadmin_util.restricted_session(p_enable => false);

-- List the MASTER BLOCKING Sessions:
SQL> select /*+RULE*/
substr(s.INST_ID||'|'||s.OSUSER||'/'||s.USERNAME||'| '||s.sid||','||s.serial#||' |'||substr(s.MACHINE,1,22)||'|'||substr(s.MODULE,1,18),1,75)"I|OS/DB USER|SID,SER|MACHN|MOD"
,substr(s.status||'|'||round(w.WAIT_TIME_MICRO/1000000)||'|'||LAST_CALL_ET||'|'||to_char(LOGON_TIME,'ddMon HH24:MI'),1,34) "ST|WAITD|ACT_SINC|LOGIN"
,substr(w.event,1,24) "EVENT"
,s.PREV_SQL_ID||'|'||s.SQL_ID||'|'||round(w.TIME_REMAINING_MICRO/1000000) "PREV|CURRENT_SQL|REMAIN_SEC"
from    gv$session s, gv$session_wait w
where    s.sid in (select distinct FINAL_BLOCKING_SESSION from gv$session where FINAL_BLOCKING_SESSION is not null)
and       s.USERNAME is not null
and     s.sid=w.sid
and    s.FINAL_BLOCKING_SESSION is null;

-- KILL the MASTER BLOCKING SESSION: [I suppose you already know what you are doing :-)]
SQL> col "KILL MASTER BLOCKING SESSION"    for a75
select /*+RULE*/ 'EXEC rdsadmin.rdsadmin_util.disconnect( '||s.sid||','||s.serial#|| ',''IMMEDIATE'');' "KILL MASTER BLOCKING SESSION"
from     gv$session s
where    s.sid in (select distinct FINAL_BLOCKING_SESSION from gv$session where FINAL_BLOCKING_SESSION is not null)
and       s.USERNAME is not null and s.FINAL_BLOCKING_SESSION is null;

-- List of Victim LOCKED Sessions:
SQL> select /*+RULE*/
substr(s.INST_ID||'|'||s.USERNAME||'| '||s.sid||','||s.serial#||' |'||substr(s.MACHINE,1,22)||'|'||substr(s.MODULE,1,18),1,65)"INS|USER|SID,SER|MACHIN|MODUL"
,substr(w.state||'|'||round(w.WAIT_TIME_MICRO/1000000)||'|'||LAST_CALL_ET||'|'||to_char(LOGON_TIME,'ddMon'),1,38) "WA_ST|WAITD|ACT_SINC|LOG_T"
,substr(w.event,1,24) "EVENT"
,s.FINAL_BLOCKING_INSTANCE||'|'||s.FINAL_BLOCKING_SESSION "I|BLKD_BY"
from    gv$session s, gv$session_wait w
where   s.USERNAME is not null
and     s.FINAL_BLOCKING_SESSION is not null
and     s.sid=w.sid
and     s.STATUS='ACTIVE'
--and     w.EVENT NOT IN ('SQL*Net message from client','class slave wait','Streams AQ: waiting for messages in the queue','Streams capture: waiting for archive log'
--        ,'Streams AQ: waiting for time management or cleanup tasks','PL/SQL lock timer','rdbms ipc message')
order by "I|BLKD_BY" desc,w.event,"INS|USER|SID,SER|MACHIN|MODUL","WA_ST|WAITD|ACT_SINC|LOG_T" desc;

-- Lock Chain Analysis:
SQL> select /*+RULE*/ 'User: '||s1.username || ' | ' || s1.module || '(SID=' || s1.sid ||' ) running SQL_ID:'||s1.sql_id||'  is blocking
User: '|| s2.username || ' | ' || s2.module || '(SID=' || s2.sid || ') running SQL_ID:'||s2.sql_id||' For '||round(s2.WAIT_TIME_MICRO/1000000/60,0)||' Minutes' AS blocking_status
from gv$LOCK l1, gv$SESSION s1, gv$LOCK l2, gv$SESSION s2
 where s1.sid=l1.sid and s2.sid=l2.sid
 and l1.BLOCK=1 and l2.request > 0
 and l1.id1 = l2.id1
 and l2.id2 = l2.id2
 order by s2.WAIT_TIME_MICRO desc;

-- Blocking Locks On Object Level:
SQL> select  /*+RULE*/ OWNER||'.'||OBJECT_NAME "LOCKED_OBJECT", ORACLE_USERNAME||' | '||lo.OS_USER_NAME "LOCK HOLDER: DB_USER | OS_USER",
l.sid||' | '|| lo.PROCESS "DB_SID | OS_PID",decode(TYPE,'MR', 'Media Recovery',
'RT', 'Redo Thread','UN', 'User Name','TX', 'Transaction','TM', 'DML','UL', 'PL/SQL User Lock','DX', 'Distributed Xaction','CF',
'Control File','IS', 'Instance State','FS', 'File Set','IR', 'Instance Recovery','ST', 'Disk Space Transaction','TS', 'Temp Segment'
,'IV', 'Library Cache Invalidation','LS', 'Log Start or Switch','RW', 'Row Wait','SQ', 'Sequence Number','TE', 'Extend Table','TT',
'Temp Table', type)||' | '||
decode(LMODE, 0, 'None',1, 'Null',2, 'row share lock',3, 'row exclusive lock',4, 'Share',5, '(SSX)exclusive lock',6, 'Exclusive', lmode) lock_type,
l.CTIME LOCK_HELD_SEC,
decode(REQUEST,0, 'None',1, 'Null',2, 'row share lock',3, 'row exclusive lock',4, 'Share',5, '(SSX)exclusive lock',6, 'Exclusive', request) lock_requested,
decode(BLOCK,0, 'Not Blocking',1, 'Blocking',2, 'Global', block)
status from    v$locked_object lo, dba_objects do, v$lock l
where   lo.OBJECT_ID = do.OBJECT_ID AND     l.SID = lo.SESSION_ID AND l.BLOCK='1'
order by OWNER,OBJECT_NAME;

-- Long Running Operations:
SQL> select USERNAME||'| '||SID||','||SERIAL# "USERNAME| SID,SERIAL#",SQL_ID,round(SOFAR/TOTALWORK*100,2) "%DONE"
        ,to_char(START_TIME,'DD-Mon HH24:MI')||'| '||trunc(ELAPSED_SECONDS/60)||'|'||trunc(TIME_REMAINING/60) "STARTED|MIN_ELAPSED|REMAIN" ,MESSAGE
        from v$session_longops
    where SOFAR/TOTALWORK*100 <>'100'and TOTALWORK <> '0'
        order by "STARTED|MIN_ELAPSED|REMAIN" desc, "USERNAME| SID,SERIAL#";


Users Management:

-- Grant permission on SYS tables/views: [With GRANT option]
SQL> EXEC rdsadmin.rdsadmin_util.grant_sys_object(p_obj_name  => 'V_$SESSION', p_grantee => 'USER1', p_privilege => 'SELECT', p_grant_option => true);

-- REVOKE permission from a user on SYS object:
SQL> EXEC rdsadmin.rdsadmin_util.revoke_sys_object(p_obj_name  => 'V_$SESSION', p_revokee  => 'USER1', p_privilege => 'SELECT');

-- Grant SELECT on Dictionary:
SQL> grant SELECT_CATALOG_ROLE to user1;

-- Grant EXECUTE on Dictionary:
SQL> grant EXECUTE_CATALOG_ROLE to user1;

-- Create Password Verify Function:
Note: As you cannot create objects under SYS in RDS you have to use the following ready made procedure by AWS to create the Verify Function:
Note: The verify function name should contains one of these keywords: "PASSWORD", "VERIFY", "COMPLEXITY", "ENFORCE", or "STRENGTH"

SQL> begin
    rdsadmin.rdsadmin_password_verify.create_verify_function(
        p_verify_function_name         => 'CUSTOM_PASSWORD_VFY_FUNCTION',
        p_min_length                   => 8,
    p_max_length            => 256,
    p_min_letters            => 1,
    p_min_lowercase            => 1,
        p_min_uppercase                => 1,
        p_min_digits                   => 3,
        p_min_special                  => 2,
    p_disallow_simple_strings    => true,
    p_disallow_whitespace        => true,
    p_disallow_username        => true,
    p_disallow_reverse        => true,
    p_disallow_db_name        => true,
        p_disallow_at_sign             => false);
end;
/


Directory/S3 Management: [Files/Upload/Download]

-- Show all files under DATA_PUMP_DIR directory:
SQL> SELECT * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('DATA_PUMP_DIR')) order by mtime;

-- Read a logfile under BDUMP directory:
SQL> SELECT text FROM table(rdsadmin.rds_file_util.read_text_file('BDUMP','dbtask-1580377405757-6339.log'));

-- Read an import log under DATA_PUMP_DIR directory:
SQL> SELECT text FROM table(rdsadmin.rds_file_util.read_text_file('DATA_PUMP_DIR','import_schema.LOG'));     

-- Create a new Directory: [i.e. BKP_DIR]
SQL> EXEC rdsadmin.rdsadmin_util.create_directory(p_directory_name => 'BKP_DIR');

-- Delete a Directory:
Note: Deleting a directory will not delete the underlying files and they will keep utilizing space, so what shall I do?

First, list all the files under the directory to be deleted and delete them one by one: [Die Hard method]

SQL> SELECT * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('BKP_DIR')) order by mtime;
SQL> EXEC SYS.UTL_FILE.FREMOVE ('BKP_DIR','import.LOG');

Then, delete the Directory:
SQL> EXEC rdsadmin.rdsadmin_util.drop_directory(p_directory_name => 'BKP_DIR');

-- Upload a file to S3 bucket:
SQL> SELECT rdsadmin.rdsadmin_s3_tasks.upload_to_s3(
    p_bucket_name => '<bucket_name>',    --bucket name where you want to upload to
    p_prefix => '<file_name>',        --File name you want to upload
    prefix => '',
    p_directory_name => 'DATA_PUMP_DIR')    --Directory Name you want to upload from
     AS TASK_ID FROM DUAL;


-- Download all files exist in S3 bucket:
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
      p_bucket_name    =>  'my-bucket',                  
--bucket name where you want to download from.
      p_directory_name =>  'DATA_PUMP_DIR') --Directory Name you want to download to.
   AS TASK_ID FROM DUAL; 


-- Download all files exist inside a named folder in S3 bucket: SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
      p_bucket_name    =>  'my-bucket',      --bucket name where you want to download from.
      p_s3_prefix          =>  'export_files/',    --All files under this folder will be downloaded [don't forget the slash / after the directory name]
      p_directory_name =>  'DATA_PUMP_DIR') --Directory Name you want to download to.
   AS TASK_ID FROM DUAL;

-- Download one named file exist under a folder from an S3 bucket:
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
      p_bucket_name    =>  'my-bucket',      --bucket name where you want to download from.
      p_s3_prefix          =>  'prod_db/export_files',  --Folder name full path
      p_prefix                =>  'EXPORT_STG_04-03-19.dmp', --file name
      p_directory_name =>  'DATA_PUMP_DIR') --Directory Name you want to download to.
    AS TASK_ID FROM DUAL;


-- Check the download progress: [By TaskID provided from above commands i.e. 1580377405757-6339]
SQL> SELECT text FROM table(rdsadmin.rds_file_util.read_text_file('BDUMP','dbtask-1580377405757-6339.log'));
 

-- Delete a file from DATA_PUMP_DIR directory: [%U all the files ending with 01,02,03,etc ...]
SQL> EXEC utl_file.fremove('DATA_PUMP_DIR','export_RA%U.dmp');

-- Rename a file under DATA_PUMP_DIR directory:
SQL> EXEC UTL_FILE.FRENAME('DATA_PUMP_DIR', '<Original_filename>', 'DATA_PUMP_DIR', '<New_filename>', TRUE);

  
Database Management:

-- Enable Force Logging:
SQL> EXEC rdsadmin.rdsadmin_util.force_logging(p_enable => true );

-- Disable Force Logging:
SQL> EXEC rdsadmin.rdsadmin_util.force_logging(p_enable => false);

-- Flush Shared Pool:
SQL> EXEC rdsadmin.rdsadmin_util.flush_shared_pool;

-- Flush Buffer Cache:
SQL> EXEC rdsadmin.rdsadmin_util.flush_buffer_cache;

-- Force a Checkpoint:
SQL> EXEC rdsadmin.rdsadmin_util.checkpoint;

-- Switch REDOLOG: [remember ALTER SYSTEM is restricted]
SQL> EXEC rdsadmin.rdsadmin_util.switch_logfile;

-- View REDOLOG switches per hour: [Last 24hours]
SQL> SELECT to_char(first_time,'YYYY-MON-DD') day,
to_char(sum(decode(to_char(first_time,'HH24'),'00',1,0)),'9999') "00",
to_char(sum(decode(to_char(first_time,'HH24'),'01',1,0)),'9999') "01",
to_char(sum(decode(to_char(first_time,'HH24'),'02',1,0)),'9999') "02",
to_char(sum(decode(to_char(first_time,'HH24'),'03',1,0)),'9999') "03",
to_char(sum(decode(to_char(first_time,'HH24'),'04',1,0)),'9999') "04",
to_char(sum(decode(to_char(first_time,'HH24'),'05',1,0)),'9999') "05",
to_char(sum(decode(to_char(first_time,'HH24'),'06',1,0)),'9999') "06",
to_char(sum(decode(to_char(first_time,'HH24'),'07',1,0)),'9999') "07",
to_char(sum(decode(to_char(first_time,'HH24'),'08',1,0)),'9999') "08",
to_char(sum(decode(to_char(first_time,'HH24'),'09',1,0)),'9999') "09",
to_char(sum(decode(to_char(first_time,'HH24'),'10',1,0)),'9999') "10",
to_char(sum(decode(to_char(first_time,'HH24'),'11',1,0)),'9999') "11",
to_char(sum(decode(to_char(first_time,'HH24'),'12',1,0)),'9999') "12",
to_char(sum(decode(to_char(first_time,'HH24'),'13',1,0)),'9999') "13",
to_char(sum(decode(to_char(first_time,'HH24'),'14',1,0)),'9999') "14",
to_char(sum(decode(to_char(first_time,'HH24'),'15',1,0)),'9999') "15",
to_char(sum(decode(to_char(first_time,'HH24'),'16',1,0)),'9999') "16",
to_char(sum(decode(to_char(first_time,'HH24'),'17',1,0)),'9999') "17",
to_char(sum(decode(to_char(first_time,'HH24'),'18',1,0)),'9999') "18",
to_char(sum(decode(to_char(first_time,'HH24'),'19',1,0)),'9999') "19",
to_char(sum(decode(to_char(first_time,'HH24'),'20',1,0)),'9999') "20",
to_char(sum(decode(to_char(first_time,'HH24'),'21',1,0)),'9999') "21",
to_char(sum(decode(to_char(first_time,'HH24'),'22',1,0)),'9999') "22",
to_char(sum(decode(to_char(first_time,'HH24'),'23',1,0)),'9999') "23"
from v$log_history where first_time > sysdate-1
GROUP by to_char(first_time,'YYYY-MON-DD') order by 1 asc;

-- Add new REDO LOG Group: [contains 1 redolog file inside with 1GB size (default is 128M)]
SQL> EXEC rdsadmin.rdsadmin_util.add_logfile(p_size => '1G');

-- Drop REDO LOG Group: [REDOLOG Group# 1]
    -- Force a Checkpoint:
        SQL> EXEC rdsadmin.rdsadmin_util.checkpoint;
    -- Switch REDOLOG: [ALTER SYSTEM is restricted]
        SQL> EXEC rdsadmin.rdsadmin_util.switch_logfile;

SQL> EXEC rdsadmin.rdsadmin_util.drop_logfile(1);

-- Set DEFAULT TABLESPACE for database:
SQL> EXEC rdsadmin.rdsadmin_util.alter_default_tablespace(tablespace_name => 'example');

-- Set DEFAULT TEMPORARY TABLESPACE for database:
SQL> EXEC rdsadmin.rdsadmin_util.alter_default_temp_tablespace(tablespace_name => 'temp2');

-- Resize TEMPORARY tablespace in a [Read Replica]:
SQL> EXEC rdsadmin.rdsadmin_util.resize_temp_tablespace('TEMP','4G');

-- Checking the Current retention for ARCHIVELOGS and TRACE Files:
SQL> set serveroutput on
     EXEC rdsadmin.rdsadmin_util.show_configuration;  
            
-- Increasing the Archivelog retention (to be kept on disk) to 1 day [24 hours]: [Default is 0]         
SQL> EXEC rdsadmin.rdsadmin_util.set_configuration(name  => 'archivelog retention hours', value => '24');

-- Add supplemental log: [For goldengate or any replication tool]
SQL> EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD','ALL');

-- Change DB timezone:
SQL> EXEC rdsadmin.rdsadmin_util.alter_db_time_zone(p_new_tz => 'Asia/Dubai');

-- Collect AWR report: [On Enterprise Edition and make sure you acquired Diagnostic+Tuning license]
copy the script content of $ORACLE_HOME/rdbms/admin/awrrpt from any database server you have [On premises or EC2] and run it on RDS database.

-- Change the default Application Edition:
SQL> EXEC rdsadmin.rdsadmin_util.alter_default_edition('APP_JAN2020');

-- Reset back to the default Application Edition:
SQL> EXEC rdsadmin.rdsadmin_util.alter_default_edition('ORA$BASE');


-- Gather Statistics:

--Gather Full DB Statistics:
SQL> BEGIN
     DBMS_STATS.GATHER_DATABASE_STATS(
     ESTIMATE_PERCENT    => DBMS_STATS.AUTO_SAMPLE_SIZE,
     --METHOD_OPT    => 'FOR ALL COLUMNS SIZE SKEWONLY',
     CASCADE         => TRUE,
     degree         => DBMS_STATS.AUTO_DEGREE,
     GATHER_SYS     => TRUE
     ,OPTIONS         => 'GATHER STALE'
     );
END;
/     

--Gather Full Schema Statistics:
SQL> BEGIN
     DBMS_STATS.GATHER_SCHEMA_STATS (
     ownname         => 'SCOTT',
     estimate_percent    => DBMS_STATS.AUTO_SAMPLE_SIZE,
     degree        => DBMS_STATS.AUTO_DEGREE,
     cascade        => TRUE
     --,OPTIONS     => 'GATHER STALE'
     );
END;
/         

--Gather one Table Statistics: (+Its indexes)
SQL> BEGIN
     DBMS_STATS.GATHER_TABLE_STATS (
     ownname         => 'SCOTT',
     tabname         => 'EMP',
     degree         => DBMS_STATS.AUTO_DEGREE,
     cascade         => TRUE,
     METHOD_OPT     => 'FOR COLUMNS SIZE AUTO',
     estimate_percent     => DBMS_STATS.AUTO_SAMPLE_SIZE
     );
END;
/         

                   
-- Re-Compile invalid objects:

--Check for invalid objects:
SQL> select object_name,object_type,status from dba_objects where status<>'VALID';

--Compile invalid objects on Full DB: (4 is the degree of parallelism, you can changed it based on system CPU power)
SQL> EXEC SYS.UTL_RECOMP.recomp_parallel(4);
 
    Or compile all DB objects Serially: (No Parallelism- It's efficently compile objects better than parallel mode but slower)
    SQL> EXECUTE SYS.UTL_RECOMP.RECOMP_SERIAL();

--Compile invalid objects on Full Schema: (4 is the degree of parallelism you can changed it based on system CPU power)
SQL> EXEC SYS.UTL_RECOMP.recomp_parallel(4, 'SCOTT');


Storage Management:

-- Tablespace Size:
SQL> select tablespace_name,
       round((tablespace_size*8192)/(1024*1024)) Total_MB,
       round((used_space*8192)/(1024*1024))      Used_MB,
       round(used_percent,2)              "%Used"
from dba_tablespace_usage_metrics;

-- Datafiles:
SQL> select tablespace_name,file_name,bytes/1024/1024 "Size",maxbytes/1024/1024 "Max_Size" from dba_data_files
union select tablespace_name,file_name,bytes/1024/1024 "Size",maxbytes/1024/1024 "Max_Size" from dba_temp_files order by 1;


Objects Management:

-- Object Size:
SQL> col SEGMENT_NAME format a30
SELECT /*+RULE*/ SEGMENT_NAME, TABLESPACE_NAME, SEGMENT_TYPE OBJECT_TYPE, ROUND(SUM(BYTES/1024/1024)) OBJECT_SIZE_MB
FROM   SYS.DBA_SEGMENTS
WHERE  OWNER =        upper('SPRINT_STAGE1')
AND    SEGMENT_NAME = upper('E_INTERLINE_MSGS20')
GROUP  BY SEGMENT_NAME, TABLESPACE_NAME, SEGMENT_TYPE;
--Lob Size:
SELECT /*+RULE*/ SEGMENT_NAME LOB_NAME, TABLESPACE_NAME, ROUND(SUM(BYTES/1024/1024)) OBJECT_SIZE_MB
FROM   SYS.DBA_SEGMENTS
WHERE  SEGMENT_NAME in (select /*+RULE*/ SEGMENT_NAME from dba_lobs where
owner=     upper('SPRINT_STAGE1') and
table_name=UPPER('E_INTERLINE_MSGS20'))
GROUP  BY SEGMENT_NAME, TABLESPACE_NAME;
--Indexes:
SELECT /*+RULE*/ SEGMENT_NAME INDEX_NAME, TABLESPACE_NAME, ROUND(SUM(BYTES/1024/1024)) OBJECT_SIZE_MB
FROM   SYS.DBA_SEGMENTS
WHERE  OWNER = upper('SPRINT_STAGE1')
AND    SEGMENT_NAME in (select index_name from dba_indexes where
owner=     upper('SPRINT_STAGE1') and
table_name=UPPER('E_INTERLINE_MSGS20'))
GROUP  BY SEGMENT_NAME, TABLESPACE_NAME;

-- Table Info:
SQL> set linesize 190
col "OWNER.TABLE"     for a35
col tablespace_name     for a25
col PCT_FREE        for 99999999
col PCT_USED        for 99999999
col "STATS_LOCKED|STALE|DATE"    for a23
col "READONLY" for a8
select t.owner||'.'||t.table_name "OWNER.TABLE",t.TABLESPACE_NAME,t.PCT_FREE
,t.PCT_USED,d.extents,t.MAX_EXTENTS,t.COMPRESSION,t.STATUS,o.created,s.stattype_locked||'|'||s.stale_stats||'|'||s.LAST_ANALYZED "STATS_LOCKED|STALE|DATE"
from dba_tables t, dba_objects o, dba_segments d, dba_tab_statistics s
where t.owner=     upper('SPRINT_STAGE1')
and t.table_name = upper('E_INTERLINE_MSGS20')
and o.owner=t.owner and o.object_name=t.table_name and o.owner=d.owner and t.table_name=d.SEGMENT_NAME and o.owner=s.owner and t.table_name=s.table_name;

-- Getting Table Size [TABLE + ITS LOBS + ITS INDEXES]...
SQL> col Name for a30
col Type for a30
set heading on echo off
COLUMN TABLE_NAME FORMAT A32
COLUMN OBJECT_NAME FORMAT A32
COLUMN OWNER FORMAT A30
SELECT owner, table_name, TRUNC(sum(bytes)/1024/1024) TOTAL_SIZE_MB FROM
(SELECT segment_name table_name, owner, bytes FROM dba_segments WHERE segment_type = 'TABLE'
 UNION ALL
 SELECT i.table_name, i.owner, s.bytes FROM dba_indexes i, dba_segments s WHERE s.segment_name = i.index_name AND   s.owner = i.owner AND s.segment_type = 'INDEX'
 UNION ALL
 SELECT l.table_name, l.owner, s.bytes FROM dba_lobs l, dba_segments s WHERE s.segment_name = l.segment_name AND s.owner = l.owner AND s.segment_type = 'LOBSEGMENT'
 UNION ALL
 SELECT l.table_name, l.owner, s.bytes FROM dba_lobs l, dba_segments s WHERE s.segment_name = l.index_name AND s.owner = l.owner AND s.segment_type = 'LOBINDEX')
 WHERE owner=       UPPER('SPRINT_STAGE1')
 and   table_name = UPPER('E_INTERLINE_MSGS20')
 GROUP BY table_name, owner
 ORDER BY SUM(bytes) desc;

-- INDEXES On the Table:
SQL> set pages 100 feedback on
set heading on
COLUMN OWNER FORMAT A25 heading "Index Owner"
COLUMN INDEX_NAME FORMAT A35 heading "Index Name"
COLUMN COLUMN_NAME FORMAT A30 heading "On Column"
COLUMN COLUMN_POSITION FORMAT 9999 heading "Pos"
COLUMN "INDEX" FORMAT A40
COLUMN TABLESPACE_NAME FOR A25
COLUMN INDEX_TYPE FOR A15
SELECT IND.OWNER||'.'||IND.INDEX_NAME "INDEX", IND.INDEX_TYPE, COL.COLUMN_NAME, COL.COLUMN_POSITION,IND.TABLESPACE_NAME,IND.STATUS,IND.UNIQUENESS,
       S.stattype_locked||'|'||S.stale_stats||'|'||S.LAST_ANALYZED "STATS_LOCKED|STALE|DATE"
FROM   SYS.DBA_INDEXES IND, SYS.DBA_IND_COLUMNS COL, SYS.DBA_IND_STATISTICS S
WHERE  IND.TABLE_NAME =  upper('OA_FREQUENT_FLYERS')
AND    IND.TABLE_OWNER = upper('SPRINT_STAGE1')
AND    IND.TABLE_NAME = COL.TABLE_NAME AND IND.OWNER = COL.INDEX_OWNER AND IND.TABLE_OWNER = COL.TABLE_OWNER AND IND.INDEX_NAME = COL.INDEX_NAME
AND    IND.OWNER = S.OWNER AND IND.INDEX_NAME = S.INDEX_NAME;

-- CONSTRAINTS On a Table:
SQL> col type format a10
col constraint_name format a40
COL COLUMN_NAME FORMAT A25 heading "On Column"
select    decode(d.constraint_type,'C', 'Check','O', 'R/O View','P', 'Primary','R', 'Foreign','U', 'Unique','V', 'Check view') type
,d.constraint_name, c.COLUMN_NAME, d.status,d.last_change
from    dba_constraints d, dba_cons_columns c
where    d.owner =      upper('SPRINT_STAGE1')
and    d.table_name = upper('E_INTERLINE_MSGS20')
and    d.OWNER=c.OWNER and d.CONSTRAINT_NAME=c.CONSTRAINT_NAME
order by 1;


Block Corruption on RDS: [Skipping Corrupted Blocks procedure]

In case you have a corrupted blocks on a table/index whenever any query try to access those corrupted blocks it will keep getting ORA-1578

From the error message try to find the corrupted object_name and its owner :
SQL> Select relative_fno,owner,segment_name,segment_type
from dba_extents
where
file_id = <DATAFILE_NUMBER_IN_THE_ERROR_MESSAGE_HERE>
and
<CORRUPTED_BLOCK_NUMBER_IN_THE_ERROR_MESSAGE_HERE> between block_id and block_id + blocks - 1;

Next follow the next steps to SKIP the corrupted blocks on the underlying object to allow queries running against to succeed:

-- Create REPAIR tables:
exec rdsadmin.rdsadmin_dbms_repair.create_repair_table;
exec rdsadmin.rdsadmin_dbms_repair.create_orphan_keys_table;                       
exec rdsadmin.rdsadmin_dbms_repair.purge_repair_table;
exec rdsadmin.rdsadmin_dbms_repair.purge_orphan_keys_table;

-- Check Corrupted blocks on the corrupted object and populate them in the REPAIR tables:
set serveroutput on
declare v_num_corrupt int;
begin
  v_num_corrupt := 0;
  rdsadmin.rdsadmin_dbms_repair.check_object (
    schema_name => '&corrupted_Object_Owner',
    object_name => '&corrupted_object_name',
    corrupt_count =>  v_num_corrupt
  );
dbms_output.put_line('number corrupt: '||to_char(v_num_corrupt));
end;
/

col corrupt_description format a30
col repair_description format a30
select object_name, block_id, corrupt_type, marked_corrupt, corrupt_description, repair_description from sys.repair_table;

select skip_corrupt from dba_tables where owner = upper('&corrupted_Object_Owner') and table_name = upper('&corrupted_object_name');                                                          

-- Enable the CORRUPTION SKIPPING on the corrupted object:
begin
  rdsadmin.rdsadmin_dbms_repair.skip_corrupt_blocks (
    schema_name => '&corrupted_Object_Owner',
    object_name => '&corrupted_object_name',
    object_type => rdsadmin.rdsadmin_dbms_repair.table_object,
    flags => rdsadmin.rdsadmin_dbms_repair.skip_flag);
end;
/

select skip_corrupt from dba_tables where owner =  upper('&corrupted_Object_Owner') and table_name = upper('&corrupted_object_name');        

              
-- Disable the CORRUPTION SKIPPING on the corrupted object:

begin
  rdsadmin.rdsadmin_dbms_repair.skip_corrupt_blocks (
    schema_name => '&corrupted_Object_Owner',
    object_name => '&corrupted_object_name',
    object_type => rdsadmin.rdsadmin_dbms_repair.table_object,
    flags => rdsadmin.rdsadmin_dbms_repair.noskip_flag);
end;
/

select skip_corrupt from dba_tables where owner =  upper('&corrupted_Object_Owner') and table_name = upper('&corrupted_object_name');                     

-- Finally DROP the repair tables:
exec rdsadmin.rdsadmin_dbms_repair.drop_repair_table;
exec rdsadmin.rdsadmin_dbms_repair.drop_orphan_keys_table;                                                


Auditing:
-- Enable Auditing for all the privileges on SYS.AUD$ table:
SQL> EXEC rdsadmin.rdsadmin_master_util.audit_all_sys_aud_table;
SQL> EXEC rdsadmin.rdsadmin_master_util.audit_all_sys_aud_table(p_by_access => true); 

-- Disable Auditing on SYS.AUD$ table:
SQL> EXEC rdsadmin.rdsadmin_master_util.noaudit_all_sys_aud_table; 


RMAN Tasks:

-- Full RMAN database Backup in RDS:
 SQL> BEGIN
    rdsadmin.rdsadmin_rman_util.backup_database_full(
        p_owner               => 'SYS',
        p_directory_name      => 'BKP_DIR',
        p_level               => 0,               -- 0 For FULL, 1 for Incremental
     --p_parallel            => 4,              -- To be hashed if using a Standard Edition
        p_section_size_mb     => 10,
        p_rman_to_dbms_output => TRUE);
END;
/
 
-- Backup ALL ARCHIVELOGS: 
SQL> BEGIN
    rdsadmin.rdsadmin_rman_util.backup_archivelog_all(
        p_owner                     => 'SYS',
        p_directory_name      => 'BKP_DIR',
     --p_parallel                   => 6,              -- To be hashed if using a Standard Edition
        p_rman_to_dbms_output => TRUE);
END;
/

-- Backup ARCHIVELOGS between a date range:
 SQL> BEGIN
    rdsadmin.rdsadmin_rman_util.backup_archivelog_date(
        p_owner                 => 'SYS',
        p_directory_name  => 'BKP_DIR',
        p_from_date           => '01/15/2020 00:00:00',
        p_to_date                => '01/16/2020 00:00:00',
     --p_parallel                => 4,              -- To be hashed if using a Standard Edition
        p_rman_to_dbms_output => TRUE);
END;
/
           
Note: in case of using SCN/sequence replace "p_from_date" with "p_from_scn" or "p_from_sequence" and "p_to_date" with "p_to_scn" or "p_to_sequence".

-- Show Running RMAN Backups: [Manual backups running by you not by RDS as RDS is using Hot Backup method to backup the DB]
 SQL> SELECT to_char (start_time,'DD-MON-YY HH24:MI') START_TIME, to_char(end_time,'DD-MON-YY HH24:MI') END_TIME, time_taken_display, status,
input_type, output_device_type,input_bytes_display, output_bytes_display, output_bytes_per_sec_display,COMPRESSION_RATIO COMPRESS_RATIO
FROM v$rman_backup_job_details
WHERE status like 'RUNNING%';

-- Show current Running Hot Backups:
SQL> SELECT t.name AS "TB_NAME", d.file# as "DF#", d.name AS "DF_NAME", b.status
FROM V$DATAFILE d, V$TABLESPACE t, V$BACKUP b
WHERE d.TS#=t.TS#
AND b.FILE#=d.FILE#
AND b.STATUS='ACTIVE';

-- Validate the database for Physical/Logical corruption on RDS: 
SQL> BEGIN
    rdsadmin.rdsadmin_rman_util.validate_database(
        p_validation_type     => 'PHYSICAL+LOGICAL', 
      --p_parallel                  => 2,              -- To be hashed if running a Standard Edition
        p_section_size_mb     => 10,
        p_rman_to_dbms_output => TRUE);
END;
/

-- Enable BLOCK CHANGE TRACKING on RDS: [Avoid enabling it on 11g as you will hit a bug will hamper you from restoring the database later consistently]
SQL> SELECT status, filename FROM V$BLOCK_CHANGE_TRACKING;
SQL> EXEC rdsadmin.rdsadmin_rman_util.enable_block_change_tracking;

-- Disable BLOCK CHANGE TRACKING on RDS:
SQL> EXEC rdsadmin.rdsadmin_rman_util.disable_block_change_tracking;

-- Crosscheck and delete expired ARCHIVELOGS: [Which are not exist anymore on disk]
 SQL> EXEC rdsadmin.rdsadmin_rman_util.crosscheck_archivelog(p_delete_expired => TRUE, p_rman_to_dbms_output => TRUE);


Oracle Scheduler Jobs Management:

-- List all jobs:
SQL> select OWNER||'.'||JOB_NAME "OWNER.JOB_NAME",ENABLED,STATE,FAILURE_COUNT,to_char(LAST_START_DATE,'DD-Mon-YYYY hh24:mi:ss TZR')LAST_RUN,to_char(NEXT_RUN_DATE,'DD-Mon-YYYY hh24:mi:ss TZR')NEXT_RUN,REPEAT_INTERVAL,
extract(day from last_run_duration) ||':'||
lpad(extract(hour from last_run_duration),2,'0')||':'||
lpad(extract(minute from last_run_duration),2,'0')||':'||
lpad(round(extract(second from last_run_duration)),2,'0') "DURATION(d:hh:mm:ss)"
from dba_scheduler_jobs order by ENABLED,STATE,"OWNER.JOB_NAME";

-- List AUTOTASK INTERNAL MAINTENANCE WINDOWS:
SQL> SELECT WINDOW_NAME,TO_CHAR(WINDOW_NEXT_TIME,'DD-MM-YYYY HH24:MI:SS') NEXT_RUN,AUTOTASK_STATUS STATUS,WINDOW_ACTIVE ACTIVE,OPTIMIZER_STATS,SEGMENT_ADVISOR,SQL_TUNE_ADVISOR FROM DBA_AUTOTASK_WINDOW_CLIENTS;

-- FAILED JOBS IN THE LAST 24H:
SQL> select JOB_NAME,OWNER,LOG_DATE,STATUS,ERROR#,RUN_DURATION from DBA_SCHEDULER_JOB_RUN_DETAILS where STATUS='FAILED' and LOG_DATE > sysdate-1 order by JOB_NAME,LOG_DATE;

-- Current running jobs:
SQL> select j.RUNNING_INSTANCE INS,j.OWNER||'.'||j.JOB_NAME ||' | '||SLAVE_OS_PROCESS_ID||'|'||j.SESSION_ID"OWNER.JOB_NAME|OSPID|SID"
,s.FINAL_BLOCKING_SESSION "BLKD_BY",ELAPSED_TIME,CPU_USED
,substr(s.SECONDS_IN_WAIT||'|'||s.WAIT_CLASS||'|'||s.EVENT,1,45) "WAITED|WCLASS|EVENT",S.SQL_ID
from dba_scheduler_running_jobs j, gv$session s
where   j.RUNNING_INSTANCE=S.INST_ID(+)
and     j.SESSION_ID=S.SID(+)
order by "OWNER.JOB_NAME|OSPID|SID",ELAPSED_TIME;

-- Disable a job owned by SYS: [Only available on 11.2.0.4.v21 & 12.2.0.2.ru-2019-07.rur-2019-07.r1 and higher]
 SQL> EXEC rdsadmin.rdsadmin_dbms_scheduler.disable('SYS.CLEANUP_NON_EXIST_OBJ');

-- Modify the repeat interval of a job: [Only available on 11.2.0.4.v21 & 12.2.0.2.ru-2019-07.rur-2019-07.r1 and higher] 
SQL> BEGIN
     rdsadmin.rdsadmin_dbms_scheduler.set_attribute(
          name      => 'SYS.CLEANUP_NON_EXIST_OBJ',
          attribute => 'repeat_interval',
          value     => 'freq=daily;byday=FRI,SAT;byhour=20;byminute=0;bysecond=0');
END;
/
