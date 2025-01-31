Disconnecting a Session
begin

rdsadmin.rdsadmin_util.disconnect(

sid => sid,

serial => serial_number);

end;

/

 

Killing a Session
begin

rdsadmin.rdsadmin_util.kill(

sid => sid,

serial => serial_number);

end;

/

 

Cancelling a SQL Statement in a Session
begin

rdsadmin.rdsadmin_util.cancel(

sid => sid,

serial => serial_number,

sql_id => sql_id);

end;

/

Enabling and Disabling Restricted Sessions

/* Verify that the database is currently unrestricted. */

select LOGINS from V$INSTANCE;

LOGINS
——-

ALLOWED

 

/* Enable restricted sessions */

exec rdsadmin.rdsadmin_util.restricted_session(p_enable => true);

/* Verify that the database is now restricted. */

select LOGINS from V$INSTANCE;

LOGINS
———-

RESTRICTED

/* Disable restricted sessions */

exec rdsadmin.rdsadmin_util.restricted_session(p_enable => false);

/* Verify that the database is now unrestricted again. */

select LOGINS from V$INSTANCE;

LOGINS
——-

ALLOWED

 

Flushing the Shared Pool

In Non RDS :

Alter system flush shared pool;

In RDS :

exec rdsadmin.rdsadmin_util.flush_shared_pool;

 

Flushing the Buffer Cache

In Non RDS :

Alter system flush buffer cache;

In RDS :

exec rdsadmin.rdsadmin_util.flush_buffer_cache;


Privileges

The following privileges are not available for the DBA role on an Amazon RDS DB instance using the Oracle engine:

ALTER DATABASE
ALTER SYSTEM
CREATE ANY DIRECTORY
DROP ANY DIRECTORY
GRANT ANY PRIVILEGE
GRANT ANY ROLE
When you create a DB instance, the master user account that you use to create the instance gets DBA privileges (with some limitations). Use the master user account for any administrative tasks such as creating additional user accounts in the database. You can’t use the SYS user, SYSTEM user, and other Oracle-supplied administrative accounts.

Granting SELECT or EXECUTE Privileges to SYS Objects

The following example grants select privileges on an object named V_$SESSION to a user named USER1:

begin

rdsadmin.rdsadmin_util.grant_sys_object(

p_obj_name => ‘V_$SESSION‘,

p_grantee => ‘USER1‘,

p_privilege => ‘SELECT‘);

end;

/

The following example grants select privileges on an object named V_$SESSION to a user named USER1 with the grant option :

begin

rdsadmin.rdsadmin_util.grant_sys_object(

p_obj_name => ‘V_$SESSION‘,

p_grantee => ‘USER1‘,

p_privilege => ‘SELECT‘,

p_grant_option => true);

end;

/

The following example grants the SELECT_CATALOG_ROLE and EXECUTE_CATALOG_ROLE to USER1. Since the with admin option is used, USER1 can now grant access to SYS objects that have been granted to SELECT_CATALOG_ROLE.

 

grant SELECT_CATALOG_ROLE to USER1 with admin option;

grant EXECUTE_CATALOG_ROLE to USER1 with admin option;

 

Revoking SELECT or EXECUTE Privileges on SYS Objects

begin

rdsadmin.rdsadmin_util.revoke_sys_object(

p_obj_name => ‘V_$SESSION‘,

p_revokee => ‘USER1‘,

p_privilege => ‘SELECT‘);

end;

/

 

Granting Privileges to Non-Master Users
grant SELECT_CATALOG_ROLE to user1;

grant EXECUTE_CATALOG_ROLE to user1;

Creating of non master users is the same process as we follow in non RDS oracle database.

 

The create_verify_function Procedure

The create_verify_function procedure is supported for Oracle version 11.2.0.4.v9 and later, Oracle version 12.1.0.2.v5 and later, all 12.2.0.1 versions, all 18.0.0.0 versions, and all 19.0.0 versions.

 

Custom Password Function 

You can create a custom function to verify passwords by using the Amazon RDS procedure rdsadmin.rdsadmin_password_verify.create_verify_function.

begin

rdsadmin.rdsadmin_password_verify.create_verify_function(

p_verify_function_name => ‘CUSTOM_PASSWORD_FUNCTION‘,

p_min_length => 10,

p_min_uppercase => 1,

p_min_digits => 1,

p_min_special => 1,

p_disallow_at_sign => true);

end;

/

 

Changing the Global Name of a Database
In NON RDS :

ALTER DATABASE RENAME GLOBAL_NAME TO database.domain;

In RDS :


exec rdsadmin.rdsadmin_util.rename_global_name(p_new_global_name => ‘new_global_name‘);

 

Creating and Sizing Tablespaces

Amazon RDS only supports Oracle Managed Files (OMF) for data files, log files, and control files. When you create data files and log files, you can’t specify the physical file names.

By default, the tablespace created is a bigfile tablespace.

To create a smallfile tablespace, you need to mention the “smallfile” keyword after create in your syntax.

create smallfile tablespace users2 datafile size 1G autoextend on maxsize 10G;

create temporary tablespace temp01;

Don’t use smallfile tablespaces because you can’t resize smallfile tablespaces with Amazon RDS for Oracle. However, you can add a datafile to a smallfile tablespace.

alter tablespace users2 add datafile size 100000M autoextend on next 250m maxsize UNLIMITED;


Setting the Default Tablespace

In Non RDS :
alter user username default tablespace tablespace_name;

In RDS :

exec rdsadmin.rdsadmin_util.alter_default_tablespace(tablespace_name => ‘users2’);



Checkpointing a Database

In Non RDS :

ALTER SYSTEM CHECKPOINT

In RDS :

exec rdsadmin.rdsadmin_util.checkpoint;

 

Creating New Directories in the Main Data Storage Space

exec rdsadmin.rdsadmin_util.create_directory(p_directory_name => ‘directory_name‘);

 

 Listing Files in a DB Instance Directory

 select * from table (rdsadmin.rds_file_util.listdir(p_directory => ‘directory_name‘));

 

Reading Files in a DB Instance Directory

 select * from table
                 (rdsadmin.rds_file_util.read_text_file(

                  p_directory => ‘directory_name‘,

                  p_filename => ‘file_name‘

                  )

);


 

Enabling Auditing for the SYS.AUD$ Table

If your auditing is not set for the database, perform the below actions

1. Click on the Parameter groups from your RDS Dashboard

2. Create a Parameter Group



3. Edit the concerned Parameter group and click on Save Changes after making the changes as given below :



4. Click on your RDS and click on MODIFY

5. From the Database Options under MODIFY, choose your recently added Parameter Group



6. Apply the changes and BOUNCE the RDS instance, since the parameter is a Static parameter.

7. Run the below commands to enable the changes after RDS restart

exec rdsadmin.rdsadmin_master_util.audit_all_sys_aud_table;

 exec rdsadmin.rdsadmin_master_util.audit_all_sys_aud_table(p_by_access => true);


Disabling Auditing for the SYS.AUD$ Table

exec rdsadmin.rdsadmin_master_util.noaudit_all_sys_aud_table;

 

Purging the Recycle Bin

exec rdsadmin.rdsadmin_util.purge_dba_recyclebin;

 

Setting Force Logging

In force logging mode, Oracle logs all changes to the database except changes in temporary tablespaces and temporary segments (NOLOGGING clauses are ignored).

exec rdsadmin.rdsadmin_util.force_logging(p_enable => true);

 

Setting Supplemental Logging

Supplemental logging ensures that LogMiner and products that use LogMiner technology have sufficient information to support chained rows and storage arrangements such as cluster tables.

Oracle Database doesn’t enable supplemental logging by default. To enable and disable supplemental logging, use the Amazon RDS procedure rdsadmin.rdsadmin_util.alter_supplemental_logging.

begin

        rdsadmin.rdsadmin_util.alter_supplemental_logging(

        p_action => ‘ADD‘);

end;

/


Adding Online Redo Logs
An Amazon RDS DB instance running Oracle starts with four online redo logs, 128 MB each.

To add additional redo logs, use the Amazon RDS procedure rdsadmin.rdsadmin_util.add_logfile.

exec rdsadmin.rdsadmin_util.add_logfile(p_size => ‘Size in M‘);

exec rdsadmin.rdsadmin_util.drop_logfile(grp => Number of groups);


Dropping Online Redo Logs

Drop each inactive log using the group number

exec rdsadmin.rdsadmin_util.drop_logfile(grp => 1);

 

Switch Log Files

exec rdsadmin.rdsadmin_util.switch_logfile;


Retaining Archived Redo Logs

You can retain archived redo logs locally on your DB instance for use with products like Oracle LogMiner (DBMS_LOGMNR). After you have retained the redo logs, you can use LogMiner to analyze the logs.

begin

     rdsadmin.rdsadmin_util.set_configuration(

     name => ‘archivelog retention hours’,

     value => ‘24‘);

end;

/

commit;

The following example shows the log retention time.

set serveroutput on

exec rdsadmin.rdsadmin_util.show_configuration;

 

View Files present in  BDUMP Directory

 SELECT * FROM table(rdsadmin.rds_file_util.listdir(‘BDUMP’)) order by mtime;

 

View the contents of a file in the BDUMP directory

 SELECT text FROM table(rdsadmin.rds_file_util.read_text_file(‘BDUMP’,’rds-rman-validate-nnn.txt’));


Oracle file size limits in AWS RDS

The maximum file size on Amazon RDS Oracle DB instances is 16 TiB (tebibytes).

 

Supported Features of AWS RDS

Amazon RDS Oracle supports the following Oracle Database features:

Advanced Compression
Application Express (APEX)
Automatic Memory Management
Automatic Undo Management
Automatic Workload Repository (AWR)
Active Data Guard with Maximum Performance in the same AWS Region or across AWS Regions
Continuous Query Notification (version 12.1.0.2.v7 and later)
Data Redaction
Database Change Notification (version 11.2.0.4.v11 and later 11g versions.
Database In-Memory (version 12.1 and later)

Distributed Queries and Transactions

Edition-Based Redefinition

Enterprise Manager Database Control (11g) and EM Express (12c)
Fine-Grained Auditing

Flashback Table, Flashback Query, Flashback Transaction Query

Import/export (legacy and Data Pump) and SQL*Loader

Java Virtual Machine (JVM)
Materialized Views

Multimedia

Network encryption
Partitioning

Spatial and Graph

Streams and Advanced Queuing

Summary Management – Materialized View Query Rewrite

Text (File and URL data store types are not supported)

Total Recall

Transparent Data Encryption (TDE)

XML DB (without the XML DB Protocol Server)
Virtual Private Database
Unsupported Features of AWS RDS

Amazon RDS Oracle doesn’t support the following Oracle Database features:

Automatic Storage Management (ASM)

Database Vault

Flashback Database

Multitenant

Oracle Enterprise Manager Cloud Control Management Repository

Real Application Clusters (Oracle RAC)

Real Application Testing

Unified Auditing, Pure Mode

Workspace Manager (WMSYS) schema
