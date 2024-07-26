### Create user
``
  CREATE USER PI_READONLY
  IDENTIFIED BY PI_READONLY
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  PROFILE &profile_DEFAULT
  ACCOUNT UNLOCK;
```
```
set long 150
set linesize 150
set longchunksize 150
select 'alter user '||username||' identified by values '||REGEXP_SUBSTR(DBMS_METADATA.get_ddl ('USER',USERNAME), '''[^'']+''')||';' from dba_users
```
```
alter session set nls_date_format = 'DY DD-MON-RRRR';
select username, account_status from dba_users where username like '%ARI%';
```
USERNAME                      ACCOUNT_STATUS
------------------------------------------------------------------------------------------------
ARIS71			EXPIRED

```
select spare4 from sys.user$ where name='ARIS71';
```
SPARE4
---------------------------------------------------------------------------------------------------------------------------------------
S:22153917B658C47DE9FDB9C18CEC5E129F03C07C1456B08E325A4288D367

alter user ARIS71 identified by values 'S:22153917B658C47DE9FDB9C18CEC5E129F03C07C1456B08E325A4288D367';

User altered.

select username, account_status from dba_users where username like '%ARI%';

USERNAME		ACCOUNT_STATUS
----------------------------------------------------------
ARIS71			OPEN

```
select  username
  ,       account_status
   ,       expiry_date
  ,       sysdate
from dba_users
where username='&usr';
```

### User Clone 
```
set pages 0 feed off verify off lines 500
accept oldname prompt "Enter user to model new user to: "
accept newname prompt "Enter new user name: "
accept psw prompt "Enter new user's password: "
 -- Create user...
select 'create user &&newname identified by &&psw'||
' default tablespace '||default_tablespace||
' temporary tablespace '||temporary_tablespace||' profile '||
profile||';'
from sys.dba_users
 where username = upper('&&oldname');
 -- Grant Roles...
 select 'grant '||granted_role||' to &&newname'||
 decode(ADMIN_OPTION, 'YES', ' WITH ADMIN OPTION')||';'
 from sys.dba_role_privs
 where grantee = upper('&&oldname');
 -- Grant System Privs...
select 'grant '||privilege||' to &&newname'||
decode(ADMIN_OPTION, 'YES', ' WITH ADMIN OPTION')||';'
 from sys.dba_sys_privs
 where grantee = upper('&&oldname');
 -- Grant Table Privs...
select 'grant '||privilege||' on '||owner||'.'||table_name||' to
&&newname;'
 from sys.dba_tab_privs
where grantee = upper('&&oldname');
 -- Grant Column Privs...
 select 'grant '||privilege||' on '||owner||'.'||table_name||
 '('||column_name||') to &&newname;'
from sys.dba_col_privs
where grantee = upper('&&oldname');
 -- Set Default Role...
 select 'alter user &&newname default role '|| granted_role ||';'
 from sys.dba_role_privs
 where grantee = upper('&&oldname')
and default_role = 'YES';
```
### Copy all the privileges from a source environment
```
set head off
set pages 0
set long 9999999
spool user_script.sql
select dbms_metadata.get_ddl( 'USER', '&user' ) from dual
UNION ALL
select dbms_metadata.get_granted_ddl( 'SYSTEM_GRANT', '&user' ) from dual
UNION ALL
select dbms_metadata.get_granted_ddl( 'OBJECT_GRANT', '&user' ) from dual
UNION ALL
select dbms_metadata.get_granted_ddl( 'ROLE_GRANT', '&user' ) from dual
union all
    select dbms_metadata.get_granted_ddl( 'TABLESPACE_QUOTA', '&user' ) from dual;
spool off;
```
Password Expire
==============
```
ALTER PROFILE SPER_PROFILE_TEMPORARY LIMIT
  FAILED_LOGIN_ATTEMPTS 3
  PASSWORD_LIFE_TIME UNLIMITED;
```
```
select 'alter user "'||username||'" identified by Password01 account unlock;' from dba_users where username not in ('SYS','SYSTEM','SYSMAN','OUTLN','SCOTT','ADAMS','JONES','CLARK','BLAKE','HR','OE','SH','DEMO','ANONYMOUS','AURORA$ORB$UNAUTHENTICATED',
'AWR_STAGE','CSMIG','CTXSYS','DBSNMP','DIP','DMSYS','EXFSYS','LBACSYS','MDSYS ','ORACLE_OCM','ORDPLUGINS','ORDPLUGINS','ORDSYS ','PERFSTAT ','TRACESVR',
'TSMSYS ','XDB','OLAPSYS','WMSYS','MDSYS','OWBSYS','OWBSYS_AUDIT','XS$NULL');
```
```
set long 1000000
 select dbms_metadata.get_ddl('USER','BACKUP') from dual;
 select dbms_metadata.get_granted_ddl('SYSTEM_GRANT','BACKUP') from dual;
 select dbms_metadata.get_granted_ddl('OBJECT_GRANT','BACKUP') from dual;
 select dbms_metadata.get_granted_ddl('ROLE_GRANT','BACKUP') from dual; 
```
----Second User 
```
select ' ,' || username || ' : ' || username || '_X' cmd
from dba_users
where username in ('ODS','ROSODS')
```
```
select 'create user '  || username || '_x identified by values ''' || password || ''';' cmd 
from dba_users
where username in ('ODS','ROSODS')
order by username;
```
```
select 'grant ' || privilege || ' on ' || owner  ||'.' || table_name || ' to ' || grantee || '_X;' CMD  
from dba_tab_privs
where grantee  in ('ODS','ROSODS')  
```
```
select 'REMAP_SCHEMA=' || username || ':' || username || '_2' cmd
from dba_users
where username not in in ('SYS','SYSTEM','OUTLN','SCOTT','ADAMS','JONES','CLARK','BLAKE','HR','OE','SH','DEMO','ANONYMOUS','AURORA$ORB$UNAUTHENTICATED',
'AWR_STAGE','CSMIG','CTXSYS','DBSNMP','DIP','DMSYS','EXFSYS','LBACSYS','MDSYS ','ORACLE_OCM','ORDPLUGINS','ORDPLUGINS','ORDSYS ','PERFSTAT ','TRACESVR',
'TSMSYS ','XDB ');
```

impdp 
-------------
imp.par
userid=system
directory=expmono1
dumpfile=pprod2ofull01.dmp
logfile=impurg.log 
parallel=2
REMAP_SCHEMA=,ROSBOAUDIT : ROSBOAUDIT_X                                                     
 ,SHERLOCK : SHERLOCK_X                                                         
 ,OLAPSYS : OLAPSYS_X                                                           
 ,DBAMON : DBAMON_X                                                             
 ,BOBJCMS : BOBJCMS_X                                                           
 ----                                              
                                 
 ,OE : OE_X                                                                     
 ,OUTLN : OUTLN_X                                                               
 --                                              
 ,FLOWS_FILES : FLOWS_FILES_X                                                   
 ,DBSNMP : DBSNMP_X                                                             
 ,MDM : MDM_X      

  



decrypt 

```

exec DBMS_SCHEDULER.CREATE_CREDENTIAL(
  credential_name => 'local_credential', 
  username => 'oracle',  password => 'welcome1'); 
```
```
select o.object_name credential_name, username, password 
 FROM SYS.SCHEDULER$_CREDENTIAL c, DBA_OBJECTS o
 WHERE c.obj# = o.object_id;
```
Nothing to blame here, but I mentioned, the password can be decrypted. So let's do so:
```
SELECT u.name CREDENTIAL_OWNER, O.NAME CREDENTIAL_NAME, C.USERNAME, 
  DBMS_ISCHED.GET_CREDENTIAL_PASSWORD(O.NAME, u.name) pwd
FROM SYS.SCHEDULER$_CREDENTIAL C, SYS.OBJ$ O, SYS.USER$ U
WHERE U.USER# = O.OWNER# 
  AND C.OBJ#  = O.OBJ# ;
```
  ```
Last login info:
col USERNAME for a15
col OS_USERNAME for a15
col USERHOST for a15
col ACTION_NAME for a15
set pages 200
set lines 100

SQL > alter session set nls_date_format=’DD-MON-YY HH24:MI:SS’;

SQL > select USERNAME,OS_USERNAME,USERHOST,TIMESTAMP,ACTION_NAME,
LOGOFF_TIME from dba_audit_trail where username=’&USERNAME’ order by USERHOST;

SELECT TO_CHAR(TIMESTAMP#,'MM/DD/YY HH:MI:SS') TIMESTAMP,
USERID, AA.NAME ACTION FROM SYS.AUD$ AT, SYS.AUDIT_ACTIONS AA
WHERE AT.ACTION# = AA.ACTION
and AA.name='LOGON'
and userid in
('&User_id')
ORDER BY TIMESTAMP# DESC;
select OS_USERNAME,action_name,USERNAME,to_char(timestamp, 'DD MON YYYY hh24:mi') logon_time,
to_char(logoff_time,'DD MON YYYY hh24:mi') logoff
from dba_audit_session where username ='&user'
AND (timestamp > (sysdate - 61))
order by logon_time,username,timestamp,logoff_time;
```



For Export 
```
set lines 2000
set heading off
spool exp_accounts.txt
select LISTAGG(username,',')
  within group (order by username)
from dba_users where username not in (
 'SYS'
,'SYSMAN'
,'SYSTEM'
,'OUTLN'
,'SCOTT'
,'ADAMS'
,'JONES'
,'CLARK'
,'BLAKE'
,'BI'
,'HR'
,'OE'
,'PM'
,'IX'
,'SH'
,'DEMO'
,'ANONYMOUS'
,'APEX_PUBLIC_USER'
,'AWR_STAGE'
,'CSMIG'
,'CTXSYS'
,'DBSNMP'
,'DIP'
,'DMSYS'
,'DSSYS'
,'EXFSYS'
,'FLOWS_040100'
,'FLOWS_FILES'
,'LBACSYS'
,'MGMT_VIEW'
,'MDDATA'
,'MDSYS'
,'OLAPSYS'
,'ORACLE_OCM'
,'ORDDATA'
,'ORDPLUGINS'
,'ORDSYS'
,'OUTLN'
,'OWBSYS'
,'PERFSTAT'
,'SI_INFORMTN_SCHEMA'
,'SPATIAL_CSW_ADMIN_USR'
,'SPATIAL_WFS_ADMIN_USR'
,'TRACESVR'
,'TSMSYS'
,'WK_TEST'
,'WKSYS'
,'WMSYS'
,'XDB'
,'XS$NULL'
,'FNMPAUDIT'
,'OWBSYS_AUDIT'
,'APEX_030200'
,'APPQOSSYS'
)
and username not like 'APAC%'
and profile not in ('&profile')
;
spool off
 ``` 



Identical objects 


```
column owner format a10
column object_name format a30
select a.owner,b.object_name,b.owner,b.object_name
from dba_objects a,dba_objects b
where a.owner not in ('SYS','SYSTEM')
and   b.owner not in ('SYS','SYSTEM')
and   a.owner <> b.owner
and   a.object_type = b.object_type
and   a.object_name = b.object_name;  
```


Move tablespace 

```

1. Create a new tablespace

CREATE BIGFILE TABLESPACE ICCGENERIC_TS DATAFILE 
  '+DATA' SIZE 43456000K AUTOEXTEND ON NEXT 100M MAXSIZE 43456000K
LOGGING
ONLINE
EXTENT MANAGEMENT LOCAL AUTOALLOCATE
BLOCKSIZE 8K
SEGMENT SPACE MANAGEMENT AUTO
FLASHBACK ON;

2. Make created tablespace as a default tablespace for ICCGENERIC user and allocate quota 

Alter user ICCGENERIC default tablespace ICCGENERIC_TS;
Alter user ICCGENERIC QUOTA unlimted on ICCGENERIC_TS;

3. Create a dynamic script to move ICCGENERIC tables to new tablespace from USERS tablespace:
spool mvtabindex.sql
select 'alter index '||owner||'.'||index_name||' rebuild tablespace ICCGENERIC_TS;' from DBA_INDEXES WHERE OWNER ='ICCGENERIC';
select 'ALTER TABLE '||OWNER||'.'||TABLE_NAME||' MOVE TABLESPACE ICCGENERIC_TS;' from DBA_TABLES where OWNER='ICCGENERIC';
spool off

Note: Alter table move invalidate all table's indexes. So this command is usually followed by "alter index rebuild

Run move table script
@mvtables.sql
4. Recompile the objects
@?/rdbms/admin/utlrp.sql  
```


Profile/Password 


```
How to recover password in oracle 10g?
You can query with the table user_history$. The password history is store in this table.

How can you track the password change for a user in oracle?
Oracle only tracks the date that the password will expire based on when it was latest changed. Thus listing the view DBA_USERS.EXPIRY_DATE and subtracting PASSWORD_LIFE_TIME you can determine when password was last changed. You can also check the last password change time directly from the PTIME column in USER$ table (on which DBA_USERS view is based). But If you have PASSWORD_REUSE_TIME and/or PASSWORD_REUSE_MAX set in a profile assigned to a user account then you can reference dictionary table USER_HISTORY$ for when the password was changed for this account.
SELECT user$.NAME, user$.PASSWORD, user$.ptime, user_history$.password_date
FROM SYS.user_history$, SYS.user$
WHERE user_history$.user# = user$.user#;
```
```
SELECT name,
 ctime,
 ptime
 FROM sys.user$
 WHERE name = 'BODSLOCADM';

SELECT sid, authentication_type, osuser 
FROM V$SESSION_CONNECT_INFO
where AUTHENTICATION_TYPE='OS';
```

## HAS THE PA SSWORD EVER CHANGED?
```
select name,to_char(ctime,'dd-mon-yy hh24:mi:ss')
,to_char(ptime,'dd-mon-yy hh24:mi:ss'),length(password)
from user$
where password is not null
and password not in ('GLOBAL','EXTERNAL')
and ctime=ptime;
```
In this script the CTIME column contains the timestamp of when the user was created. The PTIME column contains
the timestamp of when the password was changed. If the CTIME and PTIME are identical, then the password has
never changed.

all passwords that will soon expire
=================================
```
set pagesize  500
set linesize  200
set trimspool on
set feedback  off 
set heading off
column "EXPIRE DATE" format a20
select TO_CHAR(STARTUP_TIME, 'Month dd, YYYY HH24:MI:SS') "StartUp" from v$instance;
select username as "USER NAME", 
       TO_CHAR(expiry_date,'DD-MON-YYYY') as "EXPIRE DATE",
       account_status
from dba_users
where expiry_date < sysdate+30
and account_status IN ( 'OPEN', 'EXPIRED(GRACE)' );
/
```
```
select username as "USER NAME", 
       account_status
from dba_users
where  
  account_status LIKE ( '%LOCKED%' ) 
order by username;
 
/
```
CTIME is the date the user was created.
LTIME is the date the user was last locked. (Note that it doesn't get NULLed when you unlock the user).
PTIME is the date the password was last changed.
LCOUNT is the number of failed login

```
 alter session set nls_date_format='HH24:MI:SS DD/MM/YYYY';
select ctime,ltime,ptime from user$ where name = 'MTSSYS';
select ctime,ltime,ptime from user$ where name = 'MTSSYS' where ;
```

MTSSYS/mtssys

Must Run this Query after connecting by Sys user
```
SELECT user$.NAME, user$.PASSWORD, user$.ptime, user_history$.password_date
FROM SYS.user_history$, SYS.user$
WHERE user_history$.user# = user$.user#

select name, TO_CHAR(ctime,'DD-MM-YYYY HH:MI') CTIME, TO_CHAR(ptime,'DD-MM-YYYY HH:MI') PTIME
from sys.user$ where name ='MTSYS';

select
  username,
  osuser,
  terminal,
  utl_inaddr.get_host_address(terminal) IP_ADDRESS
from
  v$session
where
  username is not null
order by
  username,
  osuser;  
```


schema-Information 

```

--
-- Shows dba_users info
--

set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

accept usr prompt 'User Name    [*] : '
accept dts prompt 'Default TS   [*] : '
accept tts prompt 'Temporary TS [*] : '

col usr new_value usr noprint
col dts new_value dts noprint
col tts new_value tts noprint

column USERNAME format a27 head "Username"
column ACCOUNT_STATUS format a8 head "Status"
column LOCK_DATE format a10 head "Lock DT"
column EXPIRY_DATE format a10 head "Expire DT"
column DEFAULT_TABLESPACE format a20 head "Default TS"
column TEMPORARY_TABLESPACE format a20 head "Temporary TS"
column CREATED format a10 head "Created DT"

select USERNAME
      ,ACCOUNT_STATUS
      ,LOCK_DATE
      ,EXPIRY_DATE
      ,DEFAULT_TABLESPACE
      ,TEMPORARY_TABLESPACE
      ,CREATED
from DBA_USERS
where USERNAME like upper('%&usr%')
  and DEFAULT_TABLESPACE like upper('%&dts%')
  and TEMPORARY_TABLESPACE like upper('%&tts%')
order by CREATED,USERNAME;

REM This script displays privileges by grantee
REM It includes ROLES,SYSTEM,and OBJECT privileges
set verify off
set pagesize 80
set linesize 120
set echo off
set feedback off

column grantee    format a10
column sort_order noprint
column privilege  format a70
column type       format a15
column comparison new_value comp noprint

break on grantee skip 1 -
      on sort_order skip 1 -
      on type

accept grantee prompt "Enter grantee or blank for all: "

select decode('&grantee',NULL,'like ''%''','= upper(''&grantee'')') comparison
from dual;

SELECT grantee,
       'A' sort_order,
       'Roles...' type,
       granted_role||' '||
       decode(admin_option,'YES',' WITH ADMIN OPTION') privilege
FROM sys.dba_role_privs 
WHERE grantee &comp
UNION
SELECT grantee,
       'B' sort_order,
       'System Privs...' type,
       privilege||' '||
       decode(admin_option, 'YES',' WITH ADMIN OPTION') privilege
FROM sys.dba_sys_privs
WHERE grantee &comp
UNION
SELECT username,
       'C' sort_order,
       'DBA Privs...' type,
       DECODE(sysdba, 'TRUE', 'SYSDBA', NULL)||' '||
       ' WITH ADMIN OPTION' privilege
FROM V$PWFILE_USERS
WHERE username  &comp
AND sysdba = 'TRUE'
UNION
SELECT username,
       'C' sort_order,
       'Oper Privs...' type,
       DECODE(sysoper, 'TRUE', 'SYSOPER', NULL)||' '||
       'WITH ADMIN OPTION' privilege
FROM V$PWFILE_USERS
WHERE username  &comp
AND sysoper = 'TRUE'
UNION
SELECT grantee,
       'D' sort_order,
       'Object Privs...' type,
       privilege||' '||
       owner||'.'||table_name||
       decode(grantable, 'YES',' WITH GRANT OPTION') privilege
FROM sys.dba_tab_privs
WHERE grantee &comp ;
```
---------------
#### REM This script displays privileges by grantee
REM It includes ROLES,SYSTEM,and OBJECT privileges
```
set verify off
set pagesize 60
set linesize 80
set echo off
set feedback off

column grantee    format a30
column sort_order noprint
column privilege  format a45
column type       format a15
column comparison new_value comp noprint

break on privilege skip 1 -
      on sort_order skip 1

accept privilege prompt "Enter privilege or blank for all: "

--select decode('&grantee',NULL,'like ''%''','= upper(''&grantee'')') comparison
--from dual;

SELECT
       granted_role||' '||
       decode(admin_option,'YES',' WITH ADMIN OPTION') privilege,
       'A' sort_order,
       grantee
FROM sys.dba_role_privs 
WHERE granted_role like upper('%&privilege%')
UNION
SELECT
       privilege||' '||
       decode(admin_option, 'YES',' WITH ADMIN OPTION') privilege,
       'B' sort_order,
       grantee
FROM sys.dba_sys_privs
WHERE privilege like upper('%&privilege%')
UNION
SELECT
       DECODE(sysdba, 'TRUE', 'SYSDBA', NULL)||' '||
       ' WITH ADMIN OPTION' privilege,
       'C' sort_order,
       username
FROM V$PWFILE_USERS
WHERE sysdba  like upper('%&privilege%')
AND sysdba = 'TRUE'
UNION
SELECT
       DECODE(sysoper, 'TRUE', 'SYSOPER', NULL)||' '||
       'WITH ADMIN OPTION' privilege,
       'C' sort_order,
       username
FROM V$PWFILE_USERS
WHERE sysoper  like upper('%&privilege%')
AND sysoper = 'TRUE'
UNION
SELECT
       privilege||' '||
       owner||'.'||table_name||
       decode(grantable, 'YES',' WITH GRANT OPTION') privilege,
       'D' sort_order,
       grantee
FROM sys.dba_tab_privs
WHERE privilege like upper('%&privilege%') ;

--
-- pull user passwords
--
set pause off
set feedback off
set echo off
set verify off
set trimout on
set trimspool on
set pages 0
set lines 32767
set long  32767

column instance_name new_value instance_name noprint

select instance_name
from v$instance
/

column output format a32767

exec dbms_metadata.set_transform_param(dbms_metadata.session_transform,'SQLTERMINATOR',TRUE);
exec dbms_metadata.set_transform_param (dbms_metadata.session_transform, 'PRETTY', true);

spool set_passwd_&instance_name..sql

select REGEXP_REPLACE(dbms_metadata.get_ddl('USER', USERNAME),'CREATE USER','ALTER USER') output
FROM DBA_USERS
/

spool off
exit
```


SCHEMA SIZE
=====================
```
select sum( bytes )/1024/1024 total_size_mb
from dba_segments
where owner = 'INTERFACE'
```

```
--
-- Shows schema objects
--
set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

column OWNER format a20 head "Owner Name"
column OBJECT_NAME format a30 head "Object Name"
column OBJECT_TYPE format a12 head "Object Type"
column CREATED format a20 head "Date Created"
column LAST_DDL_TIME format a20 head "Date Last Defined"
column STATUS format a7 head "Status"

accept own prompt ' Owner Name [*] : '
accept obj prompt 'Object Name [*] : '
accept typ prompt 'Object Type [*] : '

col own new_value own noprint
col obj new_value obj noprint
col typ new_value typ noprint

select OWNER
     , OBJECT_NAME
     , OBJECT_TYPE
     , to_char(CREATED,'DD-MON-YYYY HH24:MI:SS') created
     , to_char(LAST_DDL_TIME,'DD-MON-YYYY HH24:MI:SS') last_ddl_time
     , STATUS
from dba_objects
where OWNER like decode('&own','','%',upper('&own'))
  and OBJECT_NAME like decode('&obj','','%',upper('&obj'))
  and OBJECT_TYPE like decode('&typ','','%',upper('&typ'))
order by LAST_DDL_TIME
/
```
```
--
-- Shows profiles
--
set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

column PROFILE format a20 head "Profile"
column RESOURCE_NAME format a30 head "Resource Name"
column RESOURCE_TYPE format a8 head "Resource Type"
column LIMIT format a20 head "Resource Type"

accept pro prompt ' Profile Name [*] : '
accept res prompt 'Resource Name [*] : '

col pro new_value pro noprint
col res new_value res noprint

select PROFILE
      ,RESOURCE_NAME
      ,RESOURCE_TYPE
      ,LIMIT
from dba_profiles
where profile like decode('&pro','','%',upper('&pro'))
  and resource_name like decode('&res','','%',upper('&res'))
order by profile
/
```
```
set verify off
set pagesize 0
set linesize 80
set echo off
set feedback off

column grantee    format a10
column sort_order noprint
column privilege  format a50
column type       format a15
column comparison new_value comp noprint

break on grantee skip 1 -
      on sort_order skip 1 -
      on type

accept grantee prompt "Enter grantee or blank for all: "

select decode('&grantee',NULL,'like ''%''','= upper(''&grantee'')') comparison
from dual;

SELECT 'grant '|| grantee || ' ' || granted_role ||' '|| decode(admin_option,'YES',' WITH ADMIN OPTION') || ';'
FROM sys.dba_role_privs 
WHERE grantee &comp
UNION
SELECT 'grant '|| grantee || ' ' || privilege ||' '|| decode(admin_option, 'YES',' WITH ADMIN OPTION') || ';'
FROM sys.dba_sys_privs
WHERE grantee &comp
UNION
SELECT 'grant '|| username || ' ' || DECODE(sysdba, 'TRUE', 'SYSDBA', NULL)||' '|| ' WITH ADMIN OPTION' || ';'
FROM V$PWFILE_USERS
WHERE username  &comp
AND sysdba = 'TRUE'
UNION
SELECT 'grant '|| username || ' ' || DECODE(sysoper, 'TRUE', 'SYSOPER', NULL)||' '|| 'WITH ADMIN OPTION' || ';'
FROM V$PWFILE_USERS
WHERE username  &comp
AND sysoper = 'TRUE'
UNION
SELECT 'grant '|| grantee || ' ' || privilege ||' on '|| owner||'.'||table_name|| decode(grantable, 'YES',' WITH GRANT OPTION') || ';'
FROM sys.dba_tab_privs
WHERE grantee &comp ;
```

OTHERS
--------
```
set linesize 132
set verify off
-- set feedback off
set pagesize 60
col grantee for a20 wrap
col owner for a20 wrap
col table_name for a30 wrap
col column_name for a30 wrap
col privilege for a30 wrap
col granted_rol for a20 wrap
col grantable for a10 wrap heading 'WITH GRANT'
col admin_option for a10 wrap heading 'WITH ADMIN'
accept grantee_nm prompt 'User or Role Name > '
prompt
prompt Object Privileges for &&grantee_nm....
select grantee,owner,table_name,privilege,grantable from sys.DBA_TAB_PRIVS where grantee=upper('&&grantee_nm')
 order by 2, 3, 1, 4;
prompt
prompt Column privileges for &&grantee_nm....
select grantee,owner,table_name,column_name,privilege,grantable from sys.DBA_COL_PRIVS where grantee=upper('&&grantee_nm')
 order by 2, 3,4, 5, 1;
prompt
prompt System privileges for &&grantee_nm....
select grantee,privilege,admin_option from sys.DBA_SYS_PRIVS where grantee=upper('&&grantee_nm') order by 1, 2;
prompt
prompt Role privileges for &&grantee_nm....
select grantee,granted_role,admin_option from sys.DBA_ROLE_PRIVS where grantee=upper('&&grantee_nm') order by 1, 2; 
```

### Given a db user name list the count of objects based on object type and the amount of space used by each object type 

```
SET LINESIZE 200
SET PAGESIZE 9999

COLUMN owner FORMAT A15 HEADING "OWNER"
COLUMN tablespace_name FORMAT a15 HEADING "TABLESPACE NAME"
COLUMN segment_type FORMAT A10 HEADING "SEGMENT TYPE"
COLUMN segment_name FORMAT A20 HEADING "SEGMENT NAME"
COLUMN bytes FORMAT 9,999,999,999,999 HEADING "SIZE (IN BYTES)"
COLUMN seg_count FORMAT 9,999,999,999 HEADING "SEGMENT COUNT"

break on report on owner skip 2
--compute sum label "" of seg_count bytes on owner
compute sum label "Grand Total: " of seg_count bytes on report

SELECT
owner
, tablespace_name
, segment_name
, segment_type
, sum(bytes) bytes
, count(*) seg_count
FROM
dba_segments
where lower(owner)='&owner'
GROUP BY
owner
, tablespace_name
, segment_type
, segment_name
ORDER BY
owner
, tablespace_name
, segment_type

/ 
```  



Tables 

```
--
-- Shows table counts for an owner
--

accept owner prompt "Enter table owner:  "
set echo off
set feed off
set verify off
set pages 0
set termout off
col line2 newline
spool count_em.sql
prompt spool tabcount
SELECT 'SELECT rpad('''||table_name||''',39,''.'')||COUNT(*)',
       'FROM &owner..'||table_name||' ;' line2
  FROM dba_tables
  Where owner = upper('&owner')
  ORDER by 1
/
spool off
set termout on
@count_em

Table Row Counts
UNDEFINE user
SPOOL tabcount_&&user..sql
SET LINESIZE 132 PAGESIZE 0 TRIMSPO OFF VERIFY OFF FEED OFF TERM OFF
SELECT
'SELECT RPAD(' || '''' || table_name || '''' ||',30)'
|| ',' || ' COUNT(*) FROM &&user..' || table_name || ';'
FROM dba_tables
WHERE owner = UPPER('&&user')
ORDER BY 1;
SPO OFF;
SET TERM ON
@@tabcount_&&user..sql
SET VERIFY ON FEED ON

```

```
select table_name,
   to_number(extractvalue(xmltype(dbms_xmlgen.getxml('select count(*) X from '||table_name))
              ,'/ROWSET/ROW/X')) count
    from dba_tables
  where owner = '&owner'
```

--------------------- Otherwise ----------------
```
create or replace
function get_rows( p_tname in varchar2 ) return number
as
l_columnValue number default NULL;
begin
execute immediate
‘select count(*)
from ‘ || p_tname INTO l_columnValue;

return l_columnValue;
end;
/
```
```
select table_name,
   to_number(extractvalue(xmltype(dbms_xmlgen.getxml('select count(*) X from '||table_name))
              ,'/ROWSET/ROW/X')) count
    from dba_tables
  where owner = '&owner'

select  table_name, get_rows cnt
from dba_tables
  where owner = '&owner'
/
```
```
COunt ROWS
set termout off echo off feed off trimspool on head off 
spool counts.log;
set column table_name format a16;
select 'SELECT '''||table_name||','',
   TO_CHAR(count(*),''999,999,990'')
from '||table_name||';' from user_tables;
spool off;
set termout on;
```



If you have accurate statistics, you can query the NUM_ROWS column of the CDB/DBA/ALL/USER_TABLES views.
This column normally has a close row count if statistics are generated on a regular basis. The following query selects
NUM_ROWS from the USER_TABLES view:
SQL > select table_name, num_rows from user_tables;

partitioned tables [show row counts by partition]
```
UNDEFINE user
SET SERVEROUT ON SIZE 1000000 VERIFY OFF
SPO part_count_&&user..txt
DECLARE
counter NUMBER;
sql_stmt VARCHAR2(1000);
CURSOR c1 IS
SELECT table_name, partition_name
FROM dba_tab_partitions
WHERE table_owner = UPPER('&&user');
BEGIN
FOR r1 IN c1 LOOP
sql_stmt := 'SELECT COUNT(*) FROM &&user..' || r1.table_name
||' PARTITION ( '||r1.partition_name ||' )';
EXECUTE IMMEDIATE sql_stmt INTO counter;
DBMS_OUTPUT.PUT_LINE(RPAD(r1.table_name
||'('||r1.partition_name||')',30) ||' '||TO_CHAR(counter));
END LOOP;
END;
/
SPO OFF  
```


User copy/drop/Metadata 

```

set heading off; 
set echo off; 
Set pages 999;
set long 90000; 
spool ddl_list.sql select dbms_metadata.get_ddl('TABLE','DEPT','SCOTT') from dual; 
select dbms_metadata.get_ddl('INDEX','DEPT_IDX','SCOTT') from dual; 
spool off; 

--
-- Drops user objects after prompting for input.
--

--whenever sqlerror exit 1;
set echo off
set feedback off
set trimout on
set trimspool on
set verify off
set pagesize 0
set linesize 132

accept owner   prompt 'Owner                   : '
accept objtype prompt 'Object Type [all]       : '

set term off
set feed off

select 'drop '||object_type||' '||owner||'.'||object_name||' '||
       decode(object_type,'TABLE','CASCADE CONSTRAINTS')||';'
from sys.dba_objects
where owner = upper('&&owner')
  and object_type = decode(upper(nvl('&&objtype','ALL'))
                       ,'ALL',decode(object_type
                                ,'INDEX','NO_MATCH'
                                ,object_type)
                       ,upper('&&objtype'))

spool run_drop.sql
/
spool off

set term on
set feed on
```
```
spool run_drop
@run_drop.sql
spool off
exit;
```
---------------------
USERS METADATA
------------------
```
set echo off
set heading off
set feedback off
set long 1999999999
set linesize 32767
set longchunksize 10000
set serveroutput on size unlimited format word_wrapped
set pagesize 0
set verify off
set trimspool on

col owner format a30
col segment_name format a30
col segment_type format a30
col tablespace_name format a30

prompt This script extracts DDL for the given schema 
prompt Usage:  "md.sql "  or "md.sql .", 
prompt   e.g.  "md.sql scott or md.sql scott.departments"
prompt 
prompt Parameters: 1) schema_name.[object_name]
define sch_obj_name = &1
prompt 	              schema_name = &sch_obj_name
prompt   
prompt Output: a sql script named as "..sql", e.g. "scott.dev1.sql"
prompt 

variable v_schema_name varchar2(60)
variable v_object_name varchar2(60)

exec :v_schema_name  := upper('&&sch_obj_name')

declare 
  i int;
begin 
  i := instr(:v_schema_name, '.');
  if i > 1 then 
    :v_object_name := substr(:v_schema_name, i + 1);
    :v_schema_name := substr(:v_schema_name, 1, i - 1);
  end if;
end;
/

prompt Schema / object name:
prompt 
print v_schema_name
print v_object_name

rem Switch end of line between Windows (#13#10) and Unix (#13).
rem define eol = chr(13)
define eol = chr(13)||chr(10)

prompt Extracting: &&sch_obj_name@&_CONNECT_IDENTIFIER into  &&sch_obj_name..&_CONNECT_IDENTIFIER..sql

spool &&sch_obj_name..&_CONNECT_IDENTIFIER..sql
```
```
rem --------------------------------------------------;
rem objects ddl 
rem --------------------------------------------------;

begin 
  -- set metadata
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'PRETTY',true);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'SQLTERMINATOR',true);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'SEGMENT_ATTRIBUTES',false);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'STORAGE',false);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'TABLESPACE',true);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'BODY',true);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'SPECIFICATION',true);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'BODY',false);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'CONSTRAINTS',false);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'REF_CONSTRAINTS',false);
  dbms_metadata.set_transform_param(dbms_metadata.session_transform,'CONSTRAINTS_AS_ALTER',false);
end;
/
-- select objects   
select  '/* owner=' || oo.owner || ',object_name='|| case when oo.object_type = 'INDEX' then ii.table_name else oo.object_name end ||',object_type='||oo.object_type||',status='||oo.status|| ' */' || &&eol || 
    regexp_replace(regexp_replace(dbms_metadata.get_ddl (replace(oo.object_type, ' ', '_'), oo.object_name, oo.owner), '\s+;', ';', 1, 0), ' /$',  chr(10) || '/' || chr(10), 1, 1, 'm')
  from dba_objects oo 
  left outer join 
  dba_indexes ii on ii.index_name = oo.object_name and ii.owner = oo.owner 
  where oo.owner = :v_schema_name and 
    object_name = nvl(:v_object_name, object_name) and
    oo.object_type not in ('RULE', 'LOB', 'QUEUE', 'EVALUATION CONTEXT', 'TABLE PARTITION', 'RULE SET', 'JAVA CLASS', 'TABLE SUBPARTITION', 'INDEX PARTITION', 'INDEX SUBPARTITION', 'PROGRAM', 'SCHEDULE', 'JOB', 'DATABASE LINK') and
    oo.object_name not like 'SYS_IOT_OVER%' and
    oo.object_name not like 'BIN$%' and
    oo.subobject_name is null 
  order by oo.object_name, oo.object_type;
  ```
-- constraints, ordered  
select '/* owner=' || owner || ',object_name=' || table_name || ',object_type=CONSTRAINT,status=' || status || ' */ ' || &&eol || 
    dbms_metadata.get_ddl(case constraint_type when 'R' then 'REF_CONSTRAINT' else 'CONSTRAINT' end , constraint_name, owner)
  from dba_constraints 
  where owner = :v_schema_name and 
    table_name = nvl(:v_object_name, table_name) and
    constraint_type in ('C', 'P', 'U', 'R') and
    constraint_name not like 'BIN$%'
  order by owner, table_name, constraint_name;
```
```
rem --------------------------------------------------;
rem user grants
rem --------------------------------------------------;
  
select '/* owner=' || owner || ',object_name=' || table_name || ',object_type=USER_GRANT,status=VALID */ ' || &&eol ||
    'grant ' || privilege  || ' on ' || owner || '.' || table_name || ' to ' || grantee ||  case grantable when 'YES' then  ' with grant option ' else  null end || ';'
  from (
    select unique owner, table_name, privilege, grantee, grantable 
      from dba_tab_privs  
      where grantor = :v_schema_name and
        table_name = nvl(:v_object_name, table_name) 
      order by 1,2,3,4,5);

rem --------------------------------------------------;
rem role grants
rem --------------------------------------------------;
  
select '/* owner=SYSTEM,object_name=ROLES,object_type=ROLE_GRANT,status=VALID */ ' || &&eol ||
   'grant ' || granted_role || ' to ' ||  grantee  ||  case admin_option when 'YES' then ' with admin option' else null end || ';'  
  from (
    select grantee, granted_role, admin_option 
      from dba_role_privs  
      where grantee = :v_schema_name and
        grantee = nvl(:v_object_name, grantee) 
      order by 1,2,3);  

rem --------------------------------------------------;
rem comments
rem --------------------------------------------------;
  
select  '/* owner=' || owner || ',object_name=' || table_name || ',object_type=COMMENT,status=VALID */ ' || &&eol ||
    'comment on table "' || owner || '".' || table_name  || ' is  ''' || comments || ''';'
  from dba_tab_comments  
    where owner = :v_schema_name and
      table_name = nvl(:v_object_name, table_name) 
    order by owner, table_name;

select  '/* owner=' || owner || ',object_name=' || mview_name || ',object_type=COMMENT,status=VALID */ ' || &&eol ||
    'comment on materialized view "' || owner || '".' || mview_name  || ' is  ''' || comments || ''';'
  from dba_mview_comments  
    where owner = :v_schema_name and
      mview_name = nvl(:v_object_name, mview_name) 
    order by owner, mview_name;

select  '/* owner=' || owner || ',object_name=' || table_name || ',object_type=COMMENT,status=VALID */ ' || &&eol ||
    'comment on column "' || owner || '".' || table_name  || '.' || column_name || ' is  ''' || comments || ''';'
  from dba_col_comments  
    where owner = :v_schema_name and
      table_name = nvl(:v_object_name, table_name) 
  order by owner, table_name, column_name;



rem --------------------------------------------------;
rem tablespaces
rem --------------------------------------------------;

select '/* owner=' || :v_schema_name || ',object_name=SEGMENTS,object_type=SEGMENT,status=VALID */' from dual;
select distinct owner, segment_name, segment_type, tablespace_name 
  from dba_segments 
  where owner = :v_schema_name and
    segment_name = nvl(:v_object_name, segment_name) 
  order by owner, segment_name, tablespace_name;

rem --------------------------------------------------;
rem extract end
rem --------------------------------------------------;

spool off

prompt Extracted: &&sch_obj_name@&_CONNECT_IDENTIFIER into  &&sch_obj_name..&_CONNECT_IDENTIFIER..sql

select max('main_object_type=' || replace(object_type, ' ', '_'))
  from dba_objects
  where owner = :v_schema_name and 
    object_name = nvl(:v_object_name, object_name);

quit


```


  



User Security and review 


```
Alter user &usr identified by &pass account unlock; 


Select username, account_status from dba_users
where username in ('BULLOCKAS', 'PAINE_DBA', 'PAPASDJ', 'PAINELR_RO', 'PAINELR', 'RICHARDSNA', 'HOFFMANCL_RO', 'HOFFMANCL', 'COUDERAS', 'COLESJP');


REM  ----------------------------------------------------------------------
REM --                  ORACLE DATABASE SECURITY & USER REVIEW
REM : ----------------------------------------------------------------------
REM : Script-name: ORACLE SECURITY SCRIPT.TXT
REM : Version: 
REM : Created by: Jon Paddock 10/05/2007
REM :
REM : Updated 13/08/07 - Includes updated default password list
REM : Updated 21/08/07 - Includes test 'Extracting User IDs last logon'
REM : Updated 04/09/07 - Includes explicit test for sysdba and sysoper
REM : Updated 03/10/07 - Removed update_comment field from v$parameter column for Oracle 8i compatability
REM : Updated 23/01/08 - Removed join in audit_session table and instead extracted entire table to address performance issues
REM :
REM : Description: This script extracts various information from the Oracle database to a series of text files.
REM :                     These files are then entered into a formatting and report creation tool.
REM :
REM : Requirements: Needs to be run via SQL*Plus with an account with SYSDBA privileges.
	  
REM :    -------------------------------------------------------------------
REM :    OUTPUT FORMATTING:   
REM :    The section below formats the output
REM :    -------------------------------------------------------------------

set trimspool on
set linesize 4000
SET NUMWIDTH 17
set pagesize 0
set head off
set trimspool on
set feedback off
set verify off
set colsep ','

PROMPT
PROMPT	------------------------------------------------------------
PROMPT
PROMPT	DEFINING OUTPUT-PATH
PROMPT	Specify the full path, making sure it currently exists
PROMPT	------------------------------------------------------------
PROMPT
ACCEPT file_dest prompt "Enter output file path egs. (Win) c:\audit  or  (UNIX) /audit/  :  "
PROMPT

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Database component versions
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/VERSION.TXT

SELECT *
FROM PRODUCT_COMPONENT_VERSION;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting sysdba and sysoper users
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/ADMIN.TXT

SELECT * 
FROM V$PWFILE_USERS;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting User IDs
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/USERID.TXT

SELECT *
FROM sys.dba_users
ORDER BY username;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting User IDs last logon
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/LOGON.TXT

SELECT username, os_username, timestamp, logoff_time, returncode, terminal, userhost
FROM dba_audit_session;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting Password Parameters
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/PASSWORD_PARAM.TXT

SELECT profile, resource_name, limit
FROM sys.dba_profiles
WHERE resource_type = 'PASSWORD'
order by 1;


SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting Database Initialisation Parameters
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/PARAMETERS.txt

SELECT name, value, isdefault, description
FROM v$parameter
WHERE name = 'audit_trail' or name = 'db_name' or name = 'audit_sys_operations' or name = 'remote_os_authent' or name = 'remote_os_roles' or name = 'os_roles' or name = 'remote_login_passwordfile' or name ='O7_DICTIONARY_ACCESSIBILITY' or name = 'sql92_security'
ORDER BY name;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting Database Links
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/LINKS.txt

SELECT * 
FROM sys.dba_db_links;
REM All database links

SPOOL OFF

SPOOL &FILE_DEST/$LINKS.txt

SELECT * 
FROM sys.link$;
REM Table contains passwords for fixed user links

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting User Object Privileges with DML privileges
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/USER_OBJECT_PRIVILEGES.txt



SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting User System Privileges and System Privileges Granted via Roles
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/USER_SYSTEM_PRIVILEGES.txt

SELECT * 
FROM sys.dba_sys_privs
WHERE grantee <> 'PUBLIC'
ORDER BY grantee;

SPOOL OFF

SPOOL &FILE_DEST/USER_SYSTEM_ROLE_PRIVILEGES.txt

SELECT * 
FROM sys.dba_role_privs
WHERE grantee <> 'PUBLIC'
ORDER BY grantee;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting Public System and Object pivileges
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/PUBLIC_SYSTEM_PRIVILEGES.txt

SELECT * 
FROM sys.dba_sys_privs 
WHERE grantee = 'PUBLIC';

SPOOL OFF

SPOOL &FILE_DEST/PUBLIC_TABLE_PRIVILEGES.txt

SELECT * 
FROM sys.dba_tab_privs
WHERE grantee = 'PUBLIC'
ORDER BY table_name,privilege,grantee;

SPOOL OFF

PROMPT
PROMPT ------------------------------------------------------------
PROMPT
PROMPT Extracting Object and System auditing options
PROMPT
PROMPT ------------------------------------------------------------
PROMPT

SPOOL &FILE_DEST/SYSTEM_AUDIT_OPTIONS.txt

SELECT *
FROM sys.dba_priv_audit_opts;

SPOOL OFF

SPOOL &FILE_DEST/OBJECT_AUDIT_OPTIONS.txt

SELECT *
FROM sys.dba_obj_audit_opts;

SPOOL OFF

How to find the users that have restricted session access:

SELECT b.grantee, a.grantee || ' (Role)' AS granted
FROM dba_sys_privs a, dba_role_privs b
WHERE a.privilege = 'RESTRICTED SESSION'
AND a.grantee = b.granted_role
UNION
SELECT b.username, 'User (Direct)'
FROM dba_sys_privs a, dba_users b
 WHERE a.privilege = 'RESTRICTED SESSION'
AND a.grantee = b.username;  
```


whocanaccesswhat 

```

---cams.versions_audit ====> put the table_name [ example S086.MSF700]

rem ========================================================================= 
rem 
rem objacc8.sql cams.versions_audit
rem 
rem Copyright (C) Oriole Software, 1998, 1999 
rem 
rem Downloaded from http://www.oriolecorp.com 
rem 
rem This script for Oracle database administration is free software; you 
rem can redistribute it and/or modify it under the terms of the GNU General 
rem Public License as published by the Free Software Foundation; either 
rem version 2 of the License, or any later version. 
rem 
rem This script is distributed in the hope that it will be useful, 
rem but WITHOUT ANY WARRANTY; without even the implied warranty of 
rem MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the 
rem GNU General Public License for more details. 
rem 
rem You should have received a copy of the GNU General Public License 
rem along with this program; if not, write to the Free Software 
rem Foundation, Inc., 675 Mass Ave, Cambridge, MA 02139, USA. 
rem 
rem ========================================================================= 
rem 
rem This scripts lists, giving privileges in a Unix fashion (r : right 
rem to SELECT, w : right to insert, update or delete, x : right to execute) 
rem who can do what on a given Oracle object (table, view, sequence, stored 
rem procedure ...). When grantees are roles, their name appears between square 
rem brackets. It follows the orimenu convention, i.e. that passing 
rem the '-' argument means all objects in the database. 
rem 
rem For DBAs only, as usual. Oracle8 version. 
rem 
rem # Who can access a given object [v8] (arg : [ownername.]objectname) 
rem 
column object format A40 
column type format A2 
column "ACCESS" format A6 
column grantees format A25 
clear breaks 
break on object on type on "ACCESS" 
select u.name || '.' || o.name OBJECT, decode(o.type#, 2, 'T', 4, 
       'V', 6, 'Sq', 7, 'F', 8, 'P', 9, 'PK', '?') type, 
       decode(sum(decode(m.name, 'SELECT', 1, 0)), 0, '-', 'S') || 
       decode(sum(decode(m.name, 'DELETE', 1, 0)), 0, '-', 'D') ||
       decode(sum(decode(m.name, 'INSERT', 1, 0)), 0, '-', 'I') ||
       decode(sum(decode(m.name, 'UPDATE', 1, 0)), 0, '-', 'U') "ACCESS", 
       decode(g.type#, 0, '[' || g.name || ']', g.name)
      GRANTEES 
from sys.user$ u, 
     sys.obj$ o, 
     sys.objauth$ oa, 
     sys.table_privilege_map m, 
     sys.user$ g 
where o.obj# = oa.obj# and 
      o.name = decode('cams.versions_audit', '-', o.name, substr(upper('cams.versions_audit'), 1 + 
instr('cams.versions_audit', '.'))) and 
      o.type# in (2, 4, 6, 7, 8, 9) and 
      u.user# = o.owner# and 
      u.name = decode('cams.versions_audit', '-', u.name, decode(instr('cams.versions_audit', '.'), 0, 
u.name, 
               substr(upper('cams.versions_audit'), 1, instr('cams.versions_audit', '.') - 1))) and 
      oa.privilege# = m.privilege and 
      m.name in ('DELETE', 'INSERT', 'SELECT', 'UPDATE', 'EXECUTE') and 
      oa.grantee# = g.user# 
group by u.name, o.name, o.type#, g.type#, g.name 
union 
select u.name || '.' || o.name OBJECT, 
       decode(o.type#, 2, 'T', 4, 'V', 6, 'Sq', 7, 'F', 8, 'P', 9, 'PK', '?') type, 
       '---', '*** Strictly private ***' 
from sys.user$ u, 
     sys.obj$ o 
where o.name = decode('cams.versions_audit', '-', o.name, substr(upper('cams.versions_audit'), 1 + 
instr('cams.versions_audit', '.'))) and 
      o.type# in (2, 4, 6, 7, 8, 9) and 
      u.user# = o.owner# and 
      u.name = decode('cams.versions_audit', '-', u.name, 
               decode(instr('cams.versions_audit', '.'), 0, u.name, 
               substr(upper('cams.versions_audit'), 1, instr('cams.versions_audit', '.') - 1))) and 
      not exists (select null from sys.objauth$ oa, sys.table_privilege_map m 
                  where oa.obj# = o.obj# and 
                  oa.privilege# = m.privilege and m.name in 
                  ('DELETE', 'INSERT', 'SELECT', 'UPDATE', 'EXECUTE')) 
order by 1, 3, 4 
/ 
clear breaks 

```

### QUERY TO GET SIZE OF ALL TABLES IN AN ORACLE DATABASE SCHEMA
``` 
SELECT * FROM 
(
SELECT
OWNER, 
OBJECT_NAME, 
OBJECT_TYPE, 
TABLE_NAME, 
--ROUND(BYTES)/1024/1024 AS MB,
ROUND(BYTES) / 1024 / 1024 / 1024 AS GB,
--ROUND(100*RATIO_TO_REPORT(ROUND(BYTES) / 1024 / 1024 / 1024) OVER(),2) AS GB_PERCENT,
ROUND(100*RATIO_TO_REPORT(BYTES) OVER (), 2) PERCENTAGE,
TABLESPACE_NAME, 
EXTENTS, 
INITIAL_EXTENT,
ROUND(SUM(BYTES/1024/1024/1024) OVER (PARTITION BY TABLE_NAME)) AS TOTAL_TABLE_GB
--ROUND(SUM(BYTES)/1024/1024/1024) OVER (PARTITION BY TABLE_NAME)) AS TOTAL_TABLE_GB
FROM 
(
--TABLES
SELECT OWNER, SEGMENT_NAME AS OBJECT_NAME, 'TABLE' AS OBJECT_TYPE,
SEGMENT_NAME AS TABLE_NAME, BYTES,
TABLESPACE_NAME, EXTENTS, INITIAL_EXTENT
FROM DBA_SEGMENTS /*DBA_SEGMENTS*/
WHERE SEGMENT_TYPE IN ('TABLE', 'TABLE PARTITION', 'TABLE SUBPARTITION')
UNION ALL
--INDEXES
SELECT I.OWNER, I.INDEX_NAME AS OBJECT_NAME, 'INDEX' AS OBJECT_TYPE,
I.TABLE_NAME, S.BYTES,
S.TABLESPACE_NAME, S.EXTENTS, S.INITIAL_EXTENT
FROM DBA_INDEXES I /*DBA_INDEXES*/
, DBA_SEGMENTS S /*DBA_SEGMENTS*/
WHERE S.SEGMENT_NAME = I.INDEX_NAME
AND S.OWNER = I.OWNER
AND S.SEGMENT_TYPE IN ('INDEX', 'INDEX PARTITION', 'INDEX SUBPARTITION')
--LOB SEGMENTS
UNION ALL
SELECT L.OWNER, L.COLUMN_NAME AS OBJECT_NAME, 'LOB_COLUMN' AS OBJECT_TYPE,
L.TABLE_NAME, S.BYTES,
S.TABLESPACE_NAME, S.EXTENTS, S.INITIAL_EXTENT
FROM DBA_LOBS L, /*DBA_LOBS*/
DBA_SEGMENTS S /*DBA_SEGMENTS*/
WHERE S.SEGMENT_NAME = L.SEGMENT_NAME
AND S.OWNER = L.OWNER
AND S.SEGMENT_TYPE = 'LOBSEGMENT'
--LOB INDEXES
UNION ALL
SELECT L.OWNER, L.COLUMN_NAME AS OBJECT_NAME, 'LOB_INDEX' AS OBJECT_TYPE,
L.TABLE_NAME, S.BYTES,
S.TABLESPACE_NAME, S.EXTENTS, S.INITIAL_EXTENT
FROM DBA_LOBS L, /*DBA_LOBS*/
DBA_SEGMENTS S /*DBA_SEGMENTS*/
WHERE S.SEGMENT_NAME = L.INDEX_NAME
AND S.OWNER = L.OWNER
AND S.SEGMENT_TYPE = 'LOBINDEX'
)
WHERE OWNER IN UPPER('&SCHEMA_NAME')
)
--WHERE TOTAL_TABLE_MB > 10
ORDER BY TOTAL_TABLE_GB DESC, GB DESC
/
```
