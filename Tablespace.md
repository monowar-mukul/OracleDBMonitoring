```
select *
from database_properties
where property_name like 'DEFAULT%TABLESPACE';
```

```
select	OWNER,
	SEGMENT_NAME,
	SEGMENT_TYPE,
	TABLESPACE_NAME,
	BYTES
from 	dba_segments
where	TABLESPACE_NAME = 'SYSTEM'
and	OWNER not in ('SYS','SYSTEM')
order 	by OWNER, SEGMENT_NAME
```

AUTOEXTEND
```
substr(file_name,1,50), AUTOEXTENSIBLE from dba_data_files;
```

## Tablespace free space and fragmentation
```
set linesize 150
        column tablespace_name format a20 heading 'Tablespace'
     column sumb format 999,999,999
     column extents format 9999
     column bytes format 999,999,999,999
     column largest format 999,999,999,999
     column Tot_Size format 999,999 Heading 'Total| Size(Mb)'
     column Tot_Free format 999,999,999 heading 'Total Free(MB)'
     column Pct_Free format 999.99 heading '% Free'
     column Chunks_Free format 9999 heading 'No Of Ext.'
     column Max_Free format 999,999,999 heading 'Max Free(Kb)'
     set echo off
     PROMPT  FREE SPACE AVAILABLE IN TABLESPACES
     select a.tablespace_name,sum(a.tots/1048576) Tot_Size,
     sum(a.sumb/1048576) Tot_Free,
     sum(a.sumb)*100/sum(a.tots) Pct_Free,
     sum(a.largest/1024) Max_Free,sum(a.chunks) Chunks_Free
     from
     (
     select tablespace_name,0 tots,sum(bytes) sumb,
     max(bytes) largest,count(*) chunks
     from dba_free_space a
     group by tablespace_name
     union
     select tablespace_name,sum(bytes) tots,0,0,0 from
      dba_data_files
     group by tablespace_name) a
     group by a.tablespace_name
order by pct_free;
```
```
SELECT F.TABLESPACE_NAME,
       TO_CHAR ((T.TOTAL_SPACE - F.FREE_SPACE),'999,999') "USED (MB)",
       TO_CHAR (F.FREE_SPACE, '999,999') "FREE (MB)",
       TO_CHAR (T.TOTAL_SPACE, '999,999') "TOTAL (MB)",
       TO_CHAR ((ROUND ((F.FREE_SPACE/T.TOTAL_SPACE)*100)),'999')||' %' PER_FREE
FROM   (
       SELECT       TABLESPACE_NAME, 
                    ROUND (SUM (BLOCKS*(SELECT VALUE/1024
                                        FROM V$PARAMETER 
                                        WHERE NAME = 'db_block_size')/1024)
                           ) FREE_SPACE
       FROM DBA_FREE_SPACE
       GROUP BY TABLESPACE_NAME
       ) F,
       (
       SELECT TABLESPACE_NAME,
       ROUND (SUM (BYTES/1048576)) TOTAL_SPACE
       FROM DBA_DATA_FILES
       GROUP BY TABLESPACE_NAME
       ) T
WHERE F.TABLESPACE_NAME = T.TABLESPACE_NAME
AND (ROUND ((F.FREE_SPACE/T.TOTAL_SPACE)*100)) < 10;
```
```
select TABLESPACE_NAME, FILE_NAME, sum(bytes)/1024/1024
from dba_data_files
where TABLESPACE_NAME='&TBSPACE'
group by TABLESPACE_NAME, FILE_NAME;
```
```
select tablespace_name from dba_tables where table_name like '%902%';
```

### Space available per tablespace.  
```
set pagesize 300  
set linesize 120  
select a.tablespace_name,sum(a.tots)/1024/1024 Tot_Size,   
sum(a.sumb)/1024/1024 Tot_Free,  
sum(a.sumb)*100/sum(a.tots) Pct_Free,   
sum(a.largest)/1024/1024 Max_Free,sum(a.chunks) Chunks_Free   
from   
(  
select tablespace_name,0 tots,sum(bytes) sumb,   
max(bytes) largest,count(*) chunks  
from dba_free_space a  
group by tablespace_name  
union  
select tablespace_name,sum(bytes) tots,0,0,0 from  
dba_data_files  
group by tablespace_name) a  
group by a.tablespace_name;  
```
### Find out the file_name which needs to increase: 
```
select file_name, sum(bytes)/1024/1024 from dba_data_files
where TABLESPACE_NAME='&TSpace'
group by file_name;
```
```
alter database datafile '&dfile' resize &size;
```
### Lists tables and indexes with more than 20 extents.  
```
select owner,segment_name,extents,bytes ,   
max_extents,next_extent   
from  dba_segments   
where segment_type in ('TABLE','INDEX') and extents>20   
order by owner,segment_name;  
```
### Lists tables/indexes whose next extent will not fit  
```
select  a.owner, a.segment_name, b.tablespace_name,  
     decode(ext.extents,1,b.next_extent,  
     a.bytes*(1+b.pct_increase/100)) nextext,   
     freesp.largest  
from    dba_extents a,  
     dba_segments b,  
     (select owner, segment_name, max(extent_id) extent_id,  
     count(*) extents   
     from dba_extents   
     group by owner, segment_name  
     ) ext,  
     (select tablespace_name, max(bytes) largest  
     from dba_free_space   
     group by tablespace_name  
     ) freesp  
where   a.owner=b.owner and  
     a.segment_name=b.segment_name and  
     a.owner=ext.owner and   
     a.segment_name=ext.segment_name and  
     a.extent_id=ext.extent_id and  
     b.tablespace_name = freesp.tablespace_name and   
     decode(ext.extents,1,b.next_extent,  
     a.bytes*(1+b.pct_increase/100)) > freesp.largest  
/  
```


### Create bigfile tablespace statements for migration
```
set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

column tsname format a30 head "Tablespace Name"
column total format 999,999,999,999,990 head Total
column used format 999,999,999,999,990 head Used
column free format 999,999,999,999,990 head Free
column pct_used format 990.99 head "Pct Used"

SELECT 'create bigfile tablespace '||ts.tablespace_name||
       ' datafile size '||
       ceil(decode(tf.tf_file_type
                ,'Temp',tf.tf_total - nvl(fs.free,0)
                       ,df.df_total - nvl(fs.free,0)) / 1048576) ||'M '||
       ' autoextend on next '||
       ceil(decode(tf.tf_file_type
                ,'Temp',tf.tf_total - nvl(fs.free,0)
                       ,df.df_total - nvl(fs.free,0)) * .10 / 1048576) ||'M;' as "Tablespace Create Statements"
FROM
      (SELECT tablespace_name
             ,'Data' df_file_type
             ,SUM(bytes) df_total
       FROM dba_data_files@sv_migrate
       GROUP BY tablespace_name) df
     ,(SELECT tablespace_name
             ,'Temp' tf_file_type
             ,SUM(bytes) tf_total
       FROM dba_temp_files@sv_migrate
       GROUP BY tablespace_name) tf
     ,(SELECT tablespace_name
             ,SUM(bytes) free
       FROM dba_free_space@sv_migrate
       GROUP BY tablespace_name) fs
     ,(SELECT tablespace_name
       FROM dba_tablespaces@sv_migrate
       minus
       SELECT tablespace_name
       FROM dba_tablespaces) ts
WHERE ts.tablespace_name = df.tablespace_name(+)
  and ts.tablespace_name = tf.tablespace_name(+)
  and ts.tablespace_name = fs.tablespace_name(+)
/
```
```
compute sum of blocks on report

select extent_id, bytes, blocks
from user_extents
where segment_name = '&sename'
and segment_type = 'TABLE'
 /
```
### Sample output
```
 EXTENT_ID      BYTES     BLOCKS
---------- ---------- ----------
         2      81920         10
         3     122880         15
         4     163840         20
         0      40960          5
         1      40960          5
                      ----------
sum                           55
```
```
select blocks, empty_blocks, avg_space, num_freelist_blocks
from user_tables
where table_name = 'T'
 /
```
### Sample Output:
```
    BLOCKS EMPTY_BLOCKS  AVG_SPACE NUM_FREELIST_BLOCKS
---------- ------------ ---------- -------------------
         1           53       6091                   1

Ok, the above shows us:
o we have 55 blocks allocated to the table
o 53 blocks are totally empty (above the HWM)
o 1 block contains data (the other block is used by the system)
o we have an average of about 6k free on each block used.

Therefore, our table 

o consumes 1 block
o of which  1block * 8k blocksize - 1 block * 6k free = 2k is used for our data.
```

###  CREATE DYNAMIC TABLESPACE
```
set pages 0;
set head off;
Create table ts_gen_scripts(line int, text varchar2(2000));
Truncate table ts_gen_scripts;
DECLARE
cursor cur_ts is select * from dba_Tablespaces where tablespace_name !='SYSTEM';
cursor cur_ts_files (ts_name varchar2) is select * from dba_data_files where  tablespace_name=ts_name 
order by tablespace_name, file_id;
cur_ts_files_rec cur_ts_files%rowtype;
create_ts_string varchar2(2000);
counter int := 0; 
count_files int := 0;
BEGIN
for cur_ts_rec in cur_ts loop
create_ts_string:='CREATE TABLESPACE '||cur_ts_rec.tablespace_name||CHR(10)||'DATAFILE ';
open cur_ts_files(cur_ts_rec.tablespace_name);
Loop
     fetch cur_Ts_files into cur_ts_files_rec;  
     Exit when cur_ts_files%notfound; 
   IF count_files >= 1 THEN
    create_ts_string:=create_ts_string|| CHR(10) ||'         '''||cur_ts_files_rec.file_name||''''||
        ' SIZE '||cur_ts_files_rec.bytes/1024||' K '||' , ';
      ELSE 
           create_ts_string:=create_ts_string||''''||cur_ts_files_rec.file_name||''''||
       ' SIZE '||cur_ts_files_rec.bytes/1024||' K '||' , ';
   END IF;
   count_files := count_files + 1 ;
End Loop;
   count_files := 0;
   Close Cur_TS_Files;
  Create_Ts_String := RTRIM(Create_Ts_String,' , '); -- trim the trailing ',' 
  -- attach other parameters
 -- extent management  ? local / dictionary
  -- if locaL -  ? UNIFORM / AUTOALLOCATE      
  IF Cur_Ts_Rec.extent_management = 'LOCAL' THEN
   IF Cur_Ts_Rec.Allocation_Type = 'UNIFORM' THEN
     Create_Ts_String := Create_Ts_String||CHR(10)||'EXTENT MANAGEMENT LOCAL UNIFORM SIZE '||Cur_Ts_Rec.Initial_Extent||' ;' ;
  ELSE
     Create_Ts_String := Create_Ts_String||CHR(10)||'EXTENT MANAGEMENT LOCAL AUTOALLOCATE '||' ;' ;
  END IF;
ELSE
   Create_Ts_String := Create_Ts_String|| CHR(10) ||' DEFAULT STORAGE ( '||CHR(10) ||
         ' INITIAL       '||Cur_Ts_Rec.Initial_Extent  || CHR(10) ||
        ' NEXT          '||Cur_Ts_Rec.Next_Extent   || CHR(10) ||
        ' MINEXTENTS    '||Cur_Ts_Rec.Min_Extents   || CHR(10) ||
        ' PCTINCREASE   '||Cur_Ts_Rec.Pct_Increase||' ) ;' ;
 END IF;                 
 IF Cur_Ts_Rec.contents = 'TEMPORARY' THEN
  Create_Ts_String := RTRIM ( Create_Ts_String, ';');
  Create_Ts_String := Create_Ts_String||' TEMPORARY ; ';
 END IF;
 --DBMS_Output.Put_Line(Create_Ts_String);
 counter := counter + 1;
 Insert into ts_gen_scripts values(counter, Create_Ts_String);
  End Loop;
 commit;
Exception
When Others Then
Raise_Application_Error(-20999,'Something Wrong..'||SQLERRM);
END;
/
```
### Issue 
```
ERROR at line 1:
ORA-29339: tablespace block size 16384 does not match configured block sizes
```
```
ALTER SYSTEM SET DB_16K_CACHE_SIZE=64M SCOPE=BOTH;
ALTER SYSTEM SET db_32k_cache_size=64M SCOPE=BOTH;
```
```
SQL> show parameter db_block
db_block_size                        integer     8192
SQL> show parameter cache_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_16k_cache_size                    big integer 64M
db_2k_cache_size                     big integer 0
db_32k_cache_size                    big integer 64M

```
###  tablespace quota

```
set head off;

Create table ts_gen_scripts(line int, text varchar2(2000));

Truncate table ts_gen_scripts;

DECLARE
  
cursor cur_ts is select * from dba_Tablespaces where tablespace_name !='SYSTEM';
  
cursor cur_ts_files (ts_name varchar2) is select * from dba_data_files where  tablespace_name=ts_name 
order by tablespace_name, file_id;
  
cur_ts_files_rec cur_ts_files%rowtype;
  
create_ts_string varchar2(2000);

counter int := 0; 
count_files int := 0;
    
BEGIN

for cur_ts_rec in cur_ts loop
  
create_ts_string:='CREATE TABLESPACE '||cur_ts_rec.tablespace_name||CHR(10)||'DATAFILE ';
          
open cur_ts_files(cur_ts_rec.tablespace_name);
    
Loop
   
     fetch cur_Ts_files into cur_ts_files_rec;  
     Exit when cur_ts_files%notfound; 
   
   IF count_files >= 1 THEN

     create_ts_string:=create_ts_string|| CHR(10) ||'         '''||cur_ts_files_rec.file_name||''''||
        ' SIZE '||cur_ts_files_rec.bytes/1024||' K '||' , ';
      ELSE 
           create_ts_string:=create_ts_string||''''||cur_ts_files_rec.file_name||''''||
       ' SIZE '||cur_ts_files_rec.bytes/1024||' K '||' , ';
   
   END IF;
      
   count_files := count_files + 1 ;
  
End Loop;
   count_files := 0;
   
   Close Cur_TS_Files;
   
  Create_Ts_String := RTRIM(Create_Ts_String,' , '); -- trim the trailing ',' 
  
  -- attach other parameters

  -- extent management  ? local / dictionary
  -- if locaL -  ? UNIFORM / AUTOALLOCATE      

  IF Cur_Ts_Rec.extent_management = 'LOCAL' THEN
  
   IF Cur_Ts_Rec.Allocation_Type = 'UNIFORM' THEN
     Create_Ts_String := Create_Ts_String||CHR(10)||'EXTENT MANAGEMENT LOCAL UNIFORM SIZE '||Cur_Ts_Rec.Initial_Extent||' ;' ;
  
  ELSE
     Create_Ts_String := Create_Ts_String||CHR(10)||'EXTENT MANAGEMENT LOCAL AUTOALLOCATE '||' ;' ;
  
  END IF;

 ELSE
   Create_Ts_String := Create_Ts_String|| CHR(10) ||' DEFAULT STORAGE ( '||CHR(10) ||
         ' INITIAL       '||Cur_Ts_Rec.Initial_Extent  || CHR(10) ||
        ' NEXT          '||Cur_Ts_Rec.Next_Extent   || CHR(10) ||
        ' MINEXTENTS    '||Cur_Ts_Rec.Min_Extents   || CHR(10) ||
        ' PCTINCREASE   '||Cur_Ts_Rec.Pct_Increase||' ) ;' ;
        
 END IF;                 
 IF Cur_Ts_Rec.contents = 'TEMPORARY' THEN
  Create_Ts_String := RTRIM ( Create_Ts_String, ';');
  Create_Ts_String := Create_Ts_String||' TEMPORARY ; ';
 END IF;
      
 --DBMS_Output.Put_Line(Create_Ts_String);

 counter := counter + 1;

 Insert into ts_gen_scripts values(counter, Create_Ts_String);
  
  End Loop;
  
 commit;

Exception

When Others Then

Raise_Application_Error(-20999,'Something Wrong..'||SQLERRM);
  
END;
/
```

### Check for tablespace quota
```
SET lines 100 numwidth 12
SELECT q.username, q.tablespace_name, q.bytes, q.max_bytes   
FROM dba_ts_quotas q, dba_users u  
WHERE q.username=u.username AND q.username in ('TEST'); 
```

## free tablespace (no segment) ? You may want to add an external join.
```
select b.tablespace_name, tbs_size SizeMb, a.free_space FreeMb
from  (select tablespace_name, round(sum(bytes)/1024/1024 ,2) as free_space
       from dba_free_space
       group by tablespace_name) a,
      (select tablespace_name, sum(bytes)/1024/1024 as tbs_size
       from dba_data_files
       group by tablespace_name) b
where a.tablespace_name(+)=b.tablespace_name;
```

### tablespace datafile, freespace ,availiable and autoextend or not.
```
set pagesize 100

column file_name format a32
column tablespace_name format a15
column status format a3 trunc
column t format 999,999.000 heading "Total MB"
column a format a4 heading "Aext"
column p format 990.00 heading "% Free"

SELECT df.file_name,
df.tablespace_name,
df. status,
(df.bytes/1024000) t,
(fs.s/df.bytes*100) p,
decode (ae.y,1,'YES','NO') a
FROM dba_data_files df,
(SELECT file_id,SUM(bytes) s
FROM dba_free_space
GROUP BY file_id) fs,
(SELECT file#, 1 y
FROM sys.filext$
GROUP BY file#) ae
WHERE df.file_id = fs.file_id
AND ae.file#(+) = df.file_id
ORDER BY df.tablespace_name, df.file_id;

column file_name clear
column tablespace_name clear
column status clear
column t clear
column a clear
column p clear
ttitle off
```
``` 
select file_name, bytes/1024/1024 current_size_MB, increment_by*8192/1024/1024 autoextend_size_MB, maxbytes/1024/1024 max_size_MB
from dba_data_files
where bytes>(maxbytes*0.90)
and autoextensible = 'YES'
or bytes/1024/1024+increment_by*8192/1024/1024>maxbytes/1024/1024*0.9
and autoextensible = 'YES';
```
```
SQL> select FILE_NAME, INCREMENT_BY from dba_data_files where FILE_NAME='/u01/ORADBS/BPELPRD/orabpel.dbf6.dbf';
```
### Sample Output
```
FILE_NAME                                                      INCREMENT_BY
------------
/u01/ORADBS/BPELPRD/orabpel.dbf6.dbf                                    12800


NEXT=100M [= (incrementby*block_size)/1024 M]
```
```
select owner, segment_type, segment_name, extents, max_extents/1024/1024, bytes/1024/1024 
from dba_segments
where segment_name ='AUDIT_TRAIL';
```

### To determine if your tablespaces are having a problem with fragmentation, you can use the below script:
```
set pages 50000 lines 32767
select tablespace_name,count(*) free_chunks,decode(round((max(bytes) / 1024000),2),null,0,
round((max(bytes) / 1024000),2)) largest_chunk, nvl(round(sqrt(max(blocks)/sum(blocks))*(100/sqrt(sqrt(count(blocks)) )),2),0) fragmentation_index
from sys.dba_free_space group by tablespace_name order by 2 desc, 1;
```
### Sample Output
```
TABLESPACE_NAME                                                                            FREE_CHUNKS LARGEST_CHUNK FRAGMENTATION_INDEX
------------------------------------------------------------------------------------------ ----------- ------------- -------------------
SYSAUX                                                                                            1308         52.22                2.24
IP_DATA_TS                                                                                        1062        913.41                4.16
IP_IDX_TS                                                                                          849        327.68                2.24
UNDOTBS1                                                                                           302       4063.23               13.42
USERS                                                                                               10         12.29               33.08
SYSTEM                                                                                               6         21.44               57.48
GG_DATA_TS                                                                                           2       1417.22               84.07
BI_MT_DATA                                                                                           1           .96                 100
IP_GADATA_TS                                                                                         1          8.13                 100
IP_GAIDX_TS                                                                                          1         10.18                 100
TOOLS                                                                                                1          1.09                 100

11 rows selected.
```

Note: Here,fragmentationindex column will give your tablespace an overall ranking with respect to how badly it is actually fragmented. A 100% score indicates no fragmentation at all. Lesser scores verify the presence of fragmentation.
The free chunks count column will tell you how many segments of free space are scattered throughout the tablespace. One thing to keep in mind is that tablespaces with multiple datafiles will always show a free chunk count greater than one because each datafile will likely have at least one pocket of free space.
```

set lines 132
set pages 60
column file_id format 999
column block_id format 99999999
column end_block format 99999999
column owner format a15
column segment_name format a25
column partition_name format a25
column segment_type format a12
select
    file_id,
    block_id,
    block_id + blocks - 1   end_block,
    owner,
    segment_name,
    partition_name,
    segment_type
from
    dba_extents
where
    tablespace_name = '&&tablespace'
union all
select
    file_id,
    block_id,
    block_id + blocks - 1   end_block,
    'free'          owner,
    'free'          segment_name,
    null            partition_name,
    null            segment_type
from
    dba_free_space
where
    tablespace_name = '&&tablespace'
order by
    1,2
/
```


### Steps to Check and Remove Table Fragmentation:- 
```
1. Gather table stats:
---------------------
To check exact difference in table actual size (dba_segments) and stats size (dba_tables). The difference between these value will report actual fragmentation to DBA. So, We have to have updated stats on the table stored in dba_tables. Check LAST_ANALYZED value for table in dba_tables. If this value is recent you can skip this step. Other wise i would suggest to gather table stats to get updated stats.

exec dbms_stats.gather_table_stats('&schema_name','&table_name');

2. Check Table size:
-------------------
Now again check table size using and will find reduced size of the table.

select table_name,bytes/(1024*1024*1024) from dba_table where table_name='&table_name';

3. Check for Fragmentation in table:
-----------------------------------
Below query will show the total size of table with fragmentation, expected without fragmentation and how much % of size we can reclaim after removing table fragmentation. Database Administrator has to provide table_name and schema_name as input to this query.

set pages 50000 lines 32767
select owner,table_name,round((blocks*8),2)||'kb' "Fragmented size", round((num_rows*avg_row_len/1024),2)||'kb' "Actual size", round((blocks*8),2)-round((num_rows*avg_row_len/1024),2)||'kb',
((round((blocks*8),2)-round((num_rows*avg_row_len/1024),2))/round((blocks*8),2))*100 -10 "reclaimable space % " from dba_tables where table_name ='&table_Name' AND OWNER LIKE '&schema_name'
/
Note: This query fetch data from dba_tables, so the accuracy of result depends on dba_table stats.

If you find reclaimable space % value more than 20% then we can expect fragmentation in the table. Suppose, DBA find 50% reclaimable space by above query, So he can proceed for removing fragmentation.

4. How to reset HWM / remove fragemenation?
---------------------------------------
We have four options to reorganize fragmented tables:

1. Alter table move (to another tablespace, or same tablespace) and rebuild indexes:- 
   (Depends upon the free space available in the tablespace) 
2. Export and import the table:- (difficult to implement in production environment)
3. Shrink command (fron Oracle 10g)
   (Shrink command is only applicable for tables which are tablespace with auto segment space management)

Here, I am following Options 1 and 3 option by keeping table availability in mind. 


Option: 1 Alter table move (to another tablespace, or same tablespace) and rebuild indexes:-
------------------------------------------------------------------------------------------
Collect status of all the indexes on the table:-
----------------------------------------------
We will record Index status at one place, So that we get back them after completion of this exercise,  

select index_name,status from dba_indexes where table_name like '&table_name';

Move table in to same or new tablespace:
---------------------------------------
In this step we will move fragmented table to same tablespace or from one tablespace to another tablespace to reclaim fragmented space. Find Current size of you table from dba_segments and check if same or any other tablespace has same free space available. So, that we can move this table to same or new tablespace.

Steps to Move table in to same tablespace:
-----------------------------------------
alter table <table_name> move;   ------> Move to same tablespace

OR

Steps to Move table in to new tablespace:
----------------------------------------
alter table <table_name> enable row movement;
alter table <table_name> move tablespace <new_tablespace_name>;

Now, get back table to old tablespaces using below command

alter table table_name move tablespace old_tablespace_name;

Now,Rebuild all indexes:
-----------------------
We need to rebuild all the indexes on the table because of move command all the index goes into unusable state.

SQL> select status,index_name from dba_indexes where table_name = '&table_name';

STATUS INDEX_NAME
-------- ------------------------------
UNUSABLE INDEX_NAME                            -------> Here, value in status field may be valid or unusable.

SQL> alter index <INDEX_NAME> rebuild online;  -------> Use this command for each index
Index altered.

SQL> select status,index_name from dba_indexes where table_name = '&table_name';

STATUS INDEX_NAME
-------- ------------------------------
VALID INDEX_NAME                               -------> Here, value in status field must be valid.

Gather table stats:
------------------
SQL> exec dbms_stats.gather_table_stats('&owner_name','&table_name');
PL/SQL procedure successfully completed.

Check Table size:
-----------------
Now again check table size using and will find reduced size of the table.

select table_name,bytes/(1024*1024*1024) from dba_table where table_name='&table_name';

Check for Fragmentation in table:
--------------------------------
set pages 50000 lines 32767
select owner,table_name,round((blocks*8),2)||'kb' "Fragmented size", round((num_rows*avg_row_len/1024),2)||'kb' "Actual size", round((blocks*8),2)-round((num_rows*avg_row_len/1024),2)||'kb',
((round((blocks*8),2)-round((num_rows*avg_row_len/1024),2))/round((blocks*8),2))*100 -10 "reclaimable space % " from dba_tables where table_name ='&table_Name' AND OWNER LIKE '&schema_name'
 /
==================================================================================================================
Option: 3 Shrink command (fron Oracle 10g):-
------------------------------------------

Shrink command: 
--------------
Its a new 10g feature to shrink (reorg) the tables (almost) online which can be used with automatic segment space management.

This command is only applicable for tables which are tablespace with auto segment space management.

Before using this command, you should have row movement enabled.

SQL> alter table <table_name> enable row movement;
Table altered.

There are 2 ways of using this command.

1. Rearrange rows and reset the HWM:
-----------------------------------
Part 1: Rearrange (All DML's can happen during this time)
SQL> alter table <table_name> shrink space compact;
Table altered.

Part 2: Reset HWM (No DML can happen. but this is fairly quick, infact goes unnoticed.)
SQL> alter table <table_name> shrink space;
Table altered.

2. Directly reset the HWM:
-------------------------
SQL> alter table <table_name> shrink space; (Both rearrange and restting HWM happens in one statement)
Table altered.

Advantages over the conventional methods are:
--------------------------------------------
1. Unlike "alter table move ..",indexes are not in UNUSABLE state.After shrink command,indexes are updated also.
2. Its an online operation, So you dont need downtime to do this reorg.
3. It doesnot require any extra space for the process to complete.
```


## find free extends
```
select file_id, block_id, blocks, 
owner||'.'||segment_name "Name" 
from sys.dba_extents 
where tablespace_name = 'INDX' 
UNION 
select file_id, block_id, blocks, 
'Free' 
from sys.dba_free_space 
where tablespace_name = 'INDX' 
order by 1,2,3 
```
```
select file_id, block_id, blocks, 
owner||'.'||segment_name "Name" 
from sys.dba_extents 
where tablespace_name = 'DATA' 
UNION 
select file_id, block_id, blocks, 
'Free' 
from sys.dba_free_space 
where tablespace_name = 'DATA' 
order by 1,2,3; 
```

## ALTER DATABASE DATAFILE ... RESIZE is failed 

###  Find the High Water Mark and resize the upto that. 

Anthor method If your database is oracle10g and all tablespaces are segmentspace management auto then shrink the tables upto HWM using below command. 

alter table <tblname> enable row movement; 

alter table <tblname> shrink space compact; 

How to shrink a tablespace ? 

You cannot shrink a file below the HWM of the file. The HWM of a file is the last extent in the file that has been allocated to hold a table or index extent. 

You must move the file HWM by reorganizing the objects in the file toward the beginning of the file before free extents within the file can be released to the OS. 

The alter table move and alter index rebuild commands can be used to do this for most tables. 

For tables with long data type columns export the table, drop it, and import it back into existence. 

Then after the objects have been moved toward the front of the file a shrink operation should have better success. 

Re-analyzing the objects after reorganization is a good idea but otherwise has no effect on the process. 
-----------------------------------------------------------------------------------------------------------
### Use the following script to decide the HWM of each file. Resize the file a little above the HWM. 
------------------------------------------------------------------------------------------------------------
```
set echo off 
set linesize 150 
column file_name format a45; 
column tablespace_name format a15; 
column highwater format 999,999,999,999 
column ActualSize format 999,999,999,999 
set pagesize 9999 

select a.tablespace_name,a.file_name,tot_size,nvl(actual_size,1) "Actual",
nvl(tot_size-actual_size,0) "HWM" from
(select tablespace_name,file_name,file_id,ceil(blocks*8/1024) tot_size 
from dba_data_files) a,
(select file_id,ceil(max(block_id+blocks-1)*8/1024) actual_size from dba_extents
 group by file_id)b
where a.file_id=b.file_id(+)
order by a.tablespace_name,a.file_name
/
```

### Check HWM
```
set verify off
column file_name format a50 word_wrapped
column smallest format 999,990 heading "Smallest|Size|Poss."
column currsize format 999,990 heading "Current|Size"
column savings  format 999,990 heading "Poss.|Savings"
break on report
compute sum of savings on report
column value new_val blksize
select value from v$parameter where name = 'db_block_size';
/
select file_name,
       ceil( (nvl(hwm,1)*&&blksize)/1024/1024 ) smallest,
       ceil( blocks*&&blksize/1024/1024) currsize,
       ceil( blocks*&&blksize/1024/1024) -
       ceil( (nvl(hwm,1)*&&blksize)/1024/1024 ) savings
from dba_data_files a,
     ( select file_id, max(block_id+blocks-1) hwm
         from dba_extents
        group by file_id ) b
where a.file_id = b.file_id(+) order by savings desc
/
```

## Sample Output
```
VALUE
------------------------------
8192


                                                  Smallest
                                                       Size  Current    Poss.
FILE_NAME                                             Poss.     Size  Savings
-------------------------------------------------- -------- -------- --------
/IJISDEV/oradata/undotbs01.dbf                          125    9,040    8,915
/IJISDEV/oradata/qhub_index02.dbf                    27,538   29,056    1,518
/IJISDEV/oradata/qhub_index01.dbf                    28,245   29,740    1,495
/IJISDEV/oradata/interface_data05.dbf                21,459   22,880    1,421
/IJISDEV/oradata/qhub_index03.dbf                    27,692   28,844    1,152
/IJISDEV/oradata/qhub_data01.dbf                     19,477   20,240      763
/IJISDEV/oradata/interface_index01.dbf               10,625   11,155      530
/IJISDEV/oradata/qhub_data02.dbf                     19,467   19,856      389
/IJISDEV/oradata/system01.dbf                           752    1,024      272
/IJISDEV/oradata/users01.dbf                            566      817      251
/IJISDEV/oradata/sysaux01.dbf                         1,280    1,521      241
/IJISDEV/oradata/interface_data04.dbf                24,706   24,944      238
/IJISDEV/oradata/qhub_data03.dbf                     19,500   19,608      108
/IJISDEV/oradata/interface_data03.dbf                30,998   31,000        2
/IJISDEV/oradata/interface_data01.dbf                32,767   32,768        1
/IJISDEV/oradata/interface_data02.dbf                30,240   30,240        0
                                                                     --------
sum                                                                    17,296

16 rows selected.
```
By the output i can reclaim space around 17gb of space, without reorganizing any objects. 

## For lower version 
```
set linesize 400
col tablespace_name format a15
col file_size format 99999
col file_name format a50
col hwm format 99999
col can_save format 99999
SELECT tablespace_name, file_name, file_size, hwm, file_size-hwm can_save
FROM (SELECT /*+ RULE */ ddf.tablespace_name, ddf.file_name file_name,
ddf.bytes/1048576 file_size,(ebf.maximum + de.blocks-1)*dbs.db_block_size/1048576 hwm
FROM dba_data_files ddf,(SELECT file_id, MAX(block_id) maximum FROM dba_extents GROUP BY file_id) ebf,dba_extents de,
(SELECT value db_block_size FROM v$parameter WHERE name='db_block_size') dbs
WHERE ddf.file_id = ebf.file_id
AND de.file_id = ebf.file_id
AND de.block_id = ebf.maximum
ORDER BY 1,2);
```

### Resize Datafiles:-
-----------------------------
We can resize datafile up to "Smallest Size Poss" value (or) we can assign any fixed size (or) On top of that we can enable autoextend up to maximum size of datafile.
```
 SQL> alter database datafile '/IJISDEV/oradata/qhub_index02.dbf' resize 27600M;

Database altered.
```

### TEMP
```
SQL> ALTER TABLESPACE TEMP SHRINK SPACE;
```
```
set linesize 1000 pagesize 0 feedback off trimspool on
with
 hwm as (
  -- get highest block id from each datafiles ( from x$ktfbue as we don't need all joins from dba_extents )
  select /*+ materialize */ ktfbuesegtsn ts#,ktfbuefno relative_fno,max(ktfbuebno+ktfbueblks-1) hwm_blocks
  from sys.x$ktfbue group by ktfbuefno,ktfbuesegtsn
 ),
 hwmts as (
  -- join ts# with tablespace_name
  select name tablespace_name,relative_fno,hwm_blocks
  from hwm join v$tablespace using(ts#)
 ),
 hwmdf as (
  -- join with datafiles, put 5M minimum for datafiles with no extents
  select file_name,nvl(hwm_blocks*(bytes/blocks),5*1024*1024) hwm_bytes,bytes,autoextensible,maxbytes
  from hwmts right join dba_data_files using(tablespace_name,relative_fno)
 )
select
 case when autoextensible='YES' and maxbytes>=bytes
 then -- we generate resize statements only if autoextensible can grow back to current size
  '/* reclaim '||to_char(ceil((bytes-hwm_bytes)/1024/1024),999999)
   ||'M from '||to_char(ceil(bytes/1024/1024),999999)||'M */ '
   ||'alter database datafile '''||file_name||''' resize '||ceil(hwm_bytes/1024/1024)||'M;'
 else -- generate only a comment when autoextensible is off
  '/* reclaim '||to_char(ceil((bytes-hwm_bytes)/1024/1024),999999)
   ||'M from '||to_char(ceil(bytes/1024/1024),999999)
   ||'M after setting autoextensible maxsize higher than current size for file '
   || file_name||' */'
 end SQL
from hwmdf
where
 bytes-hwm_bytes>1024*1024 -- resize only if at least 1MB can be reclaimed
order by bytes-hwm_bytes desc
/
```

### Shows tablespace utilization
```
set serveroutput on
declare

v_tablespace_name      dba_data_files.tablespace_name%TYPE;
v_bytes                dba_data_files.bytes%TYPE;
v_free_bytes           dba_free_space.bytes%TYPE;
o_bytes                char(10);
o_used                 char(10);
o_free                 char(10);
o_pct                  char(10);
x_pct                  number(7,2);
o_tablespace_name      char(25);


cursor df_cursor is
select tablespace_name, sum(bytes)
  from dba_data_files
  group by tablespace_name;

begin

open df_cursor;

dbms_output.put_line('TABLESPACE NAME                  Bytes      Used      Free       Pct.');
dbms_output.put_line('_                                 (M)       (M)       (M)        Free');
dbms_output.put_line('_');

LOOP

   fetch df_cursor into v_tablespace_name, v_bytes;
   exit when df_cursor%NOTFOUND;

   select sum(bytes)
     into v_free_bytes
     from dba_free_space
    where tablespace_name = v_tablespace_name;
    o_tablespace_name := v_tablespace_name;
    o_bytes := lpad(to_char(v_bytes / 1024 / 1024),10,' ');
    o_used  := lpad(to_char(round((v_bytes - v_free_bytes)/1024/1024,2)),10,' ');
    o_free  := lpad(to_char(round(v_free_bytes /1024/1024,2)),10,' ');
    o_pct   := lpad(to_char(round(v_free_bytes / v_bytes * 100 ,2)),10,' ');
    x_pct  := v_free_bytes / v_bytes * 100;
    dbms_output.put_line(o_tablespace_name ||  ' ' || o_bytes || ' ' || o_used || ' ' || o_free || ' ' || o_pct);
/*    printf("%s  %6.2f %6.2f %6.2f %6.2f\n",v_tablespace_name,
           v_bytes / 1024 / 1024,v_free_bytes / 1024 / 1024,(v_bytes - v_free_bytes)/1024/1024,
           v_free_bytes / v_bytes / 1024 / 1024);
*/


END LOOP;
   close df_cursor;
end;
/
```

```
select table_name,pct_free from(select table_name,pct_free from user_tables order by pct_free)
user_tables where rownum < 11
order by pct_free
/
select table_name,pct_free from dba_tables
where pct_free = (select min(pct_free) from user_tables)
and owner='&OWNER'
/
```
### Check Blocksize
Run this first query to confirm block size is 8192
```
SELECT * FROM V$PARAMETER WHERE NAME = 'db_block_size';
```
Then run the following:
```
SELECT 'ALTER DATABASE DATAFILE ''' || FILE_NAME || ''' RESIZE ' || 
       CEIL( (NVL(HWM,1)*8192)/1024/1024 ) || 'M;' SHRINK_DATAFILES
       ,round(bytes/1024/1024,2) curr_size_mb
       ,round(bytes/1024/1024,0) - CEIL( (NVL(HWM,1)*8192)/1024/1024 ) saving_mb 
from dba_data_files dbadf,
     (select file_id, max(block_id+blocks-1) hwm from dba_extents group by file_id ) dbafs
       where dbadf.file_id = dbafs.file_id(+) 
         and ceil(blocks*8192/1024/1024)- ceil((nvl(hwm,1)* 8192)/1024/1024 ) > 0;
```
```
set verify off
set lines 150
column file_name format a60 word_wrapped
column smallest format 999990 heading "Smallest|Size|Poss."
column currsize format 999,990 heading "Current|Size"
column savings  format 999,990 heading "Poss.|Savings"
break on report
compute sum of savings on report
 
-- column value new_val blksize
-- select value from v$parameter where name = 'db_block_size'
-- /

select file_name,
       ceil( (nvl(hwm,1)*4096)/1024/1024 ) smallest,
       ceil( blocks*4096/1024/1024) currsize,
       ceil( blocks*4096/1024/1024) -
       ceil( (nvl(hwm,1)*4096)/1024/1024 ) savings
from dba_data_files a,
     ( select file_id, max(block_id+blocks-1) hwm
         from dba_extents
        group by file_id ) b
where a.file_id = b.file_id(+)
and a.tablespace_name in (select tablespace_name from dba_tablespaces where block_size = 4096)
union
select file_name,
       ceil( (nvl(hwm,1)*8192)/1024/1024 ) smallest,
       ceil( blocks*8192/1024/1024) currsize,
       ceil( blocks*8192/1024/1024) -
       ceil( (nvl(hwm,1)*8192)/1024/1024 ) savings
from dba_data_files a,
     ( select file_id, max(block_id+blocks-1) hwm
         from dba_extents
        group by file_id ) b
where a.file_id = b.file_id(+)
and a.tablespace_name in (select tablespace_name from dba_tablespaces where block_size = 8192)
union
select file_name,
       ceil( (nvl(hwm,1)*16384)/1024/1024 ) smallest,
       ceil( blocks*16384/1024/1024) currsize,
       ceil( blocks*16384/1024/1024) -
       ceil( (nvl(hwm,1)*16384)/1024/1024 ) savings
from dba_data_files a,
     ( select file_id, max(block_id+blocks-1) hwm
         from dba_extents
        group by file_id ) b
where a.file_id = b.file_id(+)
and a.tablespace_name in (select tablespace_name from dba_tablespaces where block_size = 16384)
/
```
```
SQL> alter database datafile '&dbfile' resize &size;
```

### generate sql for datafile resize
```
set pagesize 0
set linesize 132
set trimout on
set trimspool on
set feedback off
set verify off
set echo off

accept tsn prompt ' Table Space Name [*] : '
accept pct prompt 'Target Pct Full [100] : '

spool dbfile_resize.sql
select  'alter database datafile '||''''||f.file_name||''''||' resize '|| greatest(ceil(used_blocks * v.BLOCK_SIZE/1048576 /(nvl('&pct',100)/100)),ceil(e.hwm * v.BLOCK_SIZE / 1048576)) ||'M;'
from    (select  a.file_id,
                 --sum(nvl(b.blocks,0)) used_blocks,
                 sum(nvl(b.blocks,1)) used_blocks,
                 max(nvl(b.block_id,0)+nvl(b.blocks,0)) hwm
         from dba_data_files a
             ,dba_extents b
         where a.tablespace_name like upper('%&tsn%')
         and a.file_id = b.file_id(+)
         group by a.file_id) e,
        dba_data_files f,
        v$datafile v
where   f.tablespace_name like upper('%&tsn%')
and     e.file_id  = f.file_id
and     f.file_id  = v.file#
/
spool off
exit
```
```
set lines 132
col owner format a15
col segment_name format a25
col segment_type format a10
col init format a20
col next format a20
col pctincr format a20

select s.OWNER
, s.SEGMENT_NAME
, s.SEGMENT_TYPE
, s.INITIAL_EXTENT || '(' ||t.INITIAL_EXTENT||')' init
, s.NEXT_EXTENT || '(' ||t.NEXT_EXTENT||')' next
, s.PCT_INCREASE || '(' ||t.PCT_INCREASE||')' pctincr
from dba_segments s, dba_tablespaces t
where s.TABLESPACE_NAME = t.TABLESPACE_NAME
and t.TABLESPACE_NAME like upper('%&tablespace%')
and (s.NEXT_EXTENT = t.NEXT_EXTENT
     or s.PCT_INCREASE != t.PCT_INCREASE)
/
```

####segment size

```
COL Object FORMAT a24;
COL Type FORMAT a5;
SELECT segment_name "Object", segment_type "Type"
      ,ROUND(bytes/1024/1024) "Mb", ROUND(bytes/1024) "Kb"
      ,bytes "Bytes", blocks "Blocks"
FROM dba_segments WHERE owner = 'ELLIPSE'
AND segment_type IN ('TABLE','INDEX')
ORDER BY segment_name, segment_type DESC;
```
```
COL Object FORMAT a24;
COL Type FORMAT a5;
SELECT segment_name "Object", segment_type "Type"
      ,ROUND(bytes/1024/1024) "Mb", ROUND(bytes/1024) "Kb"
      ,bytes "Bytes", blocks "Blocks"
FROM dba_segments WHERE owner = 'INTERFACE'
AND segment_type IN ('TABLE','INDEX')
-- AND segment_name='INTERNAL_MESSAGE_LOG_IML'
ORDER BY segment_name, segment_type DESC;
```
```
SELECT s.owner,SUM (s.BYTES) / (1024 * 1024 * 1024) SIZE_IN_GB
FROM dba_segments s
GROUP BY s.owner;
```

## OWNER, TABLESPACE, SEGMENTS
```
SQL>            select substr(a.owner,1,15) owner ,
                          substr(a.tablespace_name,1,20) tablespace_name,
                          substr(a.segment_name,1,30) Name,substr(a.segment_type,1,12) Type,
                                  to_number(substr(to_char(round (a.bytes/(1024*1024))),1,6)) MB_Cons,'  ',
                                  substr(to_char(round (count(b.extent_id))),1,8) Extents,
                          to_number(substr(to_char(round (a.initial_extent/(1024*1024),2)),1,6)) Initial_EXT,
                          to_number(substr(to_char(round (a.next_extent   /(1024*1024),2)),1,6)) Next_Extent
               from dba_segments a , dba_extents b
               where a.owner NOT IN ('SYS')
              AND to_number(substr(to_char(round (a.bytes/(1024*1024),2)),1,6))> 0
              and a.segment_name=b.segment_name
              group by a.owner,a.tablespace_name,a.segment_name,a.segment_type,a.initial_extent,a.next_extent,
              a.min_extents,a.bytes
             order by 5 DESC
           /
```

### List segments with fewer than 5 extents remaining 
```
SELECT segment_name,segment_type,
max_extents, extents
FROM dba_segments
WHERE extents+5 > max_extents
AND segment_type<>'CACHE';
 ```
### Extent information for a table 
===================================
```
SELECT segment_name, extent_id, blocks, bytes/1024/1024
FROM dba_extents
WHERE owner='&own' and segment_name = '&TNAME' ;
```
```
select segment_name,segment_type,(bytes/1024/1024)MB 
from dba_segments where owner='PRODOWNER';
```
```
col owner format a15
col segment_name format a35

select * from
(
select owner, segment_Type, segment_name, bytes/1024 kbytes
from dba_segments
where tablespace_name = '&ts_name'
order by bytes/1024 desc
)
where rownum < 10
/
```
### Extents reaching maximum

TABLES AND EXTENTS WITHIN 3 EXTENTS OF MAXIMUM :
```
select owner "Owner",
       segment_name "Segment Name",
       segment_type "Type",
       tablespace_name "Tablespace",
       extents "Ext",
       max_extents "Max"
from dba_segments
where ((max_extents - extents) <= 3) 
and owner not in ('SYS','SYSTEM')
order by owner, segment_name;
```
### Segment Fragmentation

OBJECTS WITH MORE THAN 50% OF MAXEXTENTS NOTES:
Owner - Owner of the object 
Tablespace Name - Name of the tablespace 
Segment Name - Name of the segment 
Segment Type - Type of segment 
Size - Size of the object (bytes) 
Extents - Current number of extents 
Max Extents - Maximum extents for the segment 
Percentage - Percentage of extents in use 

As of v7.3.4, you can set MAXEXTENTS=UNLIMITED to avoid ORA-01631: max # extents (%s) reached in table $s.%s. 
To calculate the MAXEXTENTS value on versions < 7.3.4 use the following equation: DBBLOCKSIZE / 16 - 7 
Here are the MAXEXTENTS for common blocksizes: 1K=57, 2K=121, 4K=249, 8K=505, and 16K=1017 
Multiple extents in and of themselves aren't bad. However, if you also have chained rows, this can hurt performance. 
```
select 	OWNER,
	TABLESPACE_NAME,
	SEGMENT_NAME,
	SEGMENT_TYPE,
	BYTES,
	EXTENTS,
	MAX_EXTENTS,
	(EXTENTS/MAX_EXTENTS)*100 percentage
from 	dba_segments
where 	SEGMENT_TYPE in ('TABLE','INDEX')
and 	EXTENTS > MAX_EXTENTS/2
order 	by (EXTENTS/MAX_EXTENTS) desc
```
### column differnce 
```
SELECT 'TABLE_A has these columns that are not in TABLE_B', DIFF.*
  FROM (
        SELECT  COLUMN_NAME, DATA_TYPE, DATA_LENGTH
          FROM all_tab_columns
         WHERE table_name = 'TABLE_A'
         MINUS
        SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
          FROM all_tab_columns
         WHERE table_name = 'TABLE_B' 
      ) DIFF
UNION
SELECT 'TABLE_B has these columns that are not in TABLE_A', DIFF.*
  FROM (
        SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
          FROM all_tab_columns
         WHERE table_name = 'TABLE_B'
         MINUS
        SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
          FROM all_tab_columns
         WHERE table_name = 'TABLE_A'
      ) DIFF;
```
```
select count (*) from TABLENAME
minus select count (*) from TABLENAME@DATABASE2
```
### DATAFILES with TABLESPACE INFO
```
spool tablespace_info.log
set linesize 200 LONG 50000 pagesize 5000
select dbms_metadata.get_ddl('TABLESPACE',tablespace_name)
from dba_tablespaces
where tablespace_name in ('USERS','FASMON_CEI_TS_BASE','FASMON_CEI_TS_CATALOG','FASMON_CEI_TS_EXTENDED','FASMON_CEI_TS_TEMP');
spool off;
```
```
select owner, constraint_name,table_name,index_owner,index_name
from dba_constraints
where (index_owner,index_name) in (select owner,index_name from dba_indexes
 where tablespace_name='FASMON_CEI_TS_BASE');
```
```
col tablespace_name format a30
col file_name format a60

SELECT dd.tablespace_name tablespace_name, dd.file_name file_name, dd.bytes/(1024*1024) TABLESPACE_MB, SUM(fs.bytes)/(1024*1024) MBYTES_FREE, MAX(fs.bytes)/(1024*1024) NEXT_FREE 
FROM sys.dba_free_space fs, sys.dba_data_files dd 
WHERE dd.tablespace_name = fs.tablespace_name AND dd.file_id = fs.file_id 
GROUP BY dd.tablespace_name, dd.file_name, dd.bytes/(1024*1024) 
ORDER BY SUM(fs.bytes)/(1024*1024);
```
```
SELECT DISTINCT OWNER, SEGMENT_NAME
FROM DBA_EXTENTS
WHERE FILE_ID =(select file# from v$datafile where name ='&dbfile_name' );  
```

## MOVE DATAFILES TO ASM

for moving the datafile to ASM use rman, 
```
allocate channel c1 type disk format "+diskgroup"; 
backup as copy datafile file#; 
switch datafile to copy; 
```

1.Check datafiles for the tablespace
------------------------------------
```
SQL> select FILE_ID, FILE_NAME from dba_data_files where TABLESPACE_NAME='ROSBO';
```
   FILE_ID
----------
FILE_NAME
--------------------------------------------------------------------------------
         6
+PDEV20_DATA_01/pdev20/datafile/rosbo.269.799948433

        13
/app/pdev20/product/10.2.0.5/db_1/dbs/PDEV20_DATA_01

2. connect to RMAN and put tablespace [respective datafiles] offline 
```
$ rman target /

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
```
col tablespace format A16
SELECT /* + RULE */ df.tablespace_name "Tablespace",
df.bytes / (1024 * 1024) "Size (MB)",
SUM(fs.bytes) / (1024 * 1024) "Free (MB)",
Nvl(Round(SUM(fs.bytes) * 100 / df.bytes),1) "% Free",
Round((df.bytes - SUM(fs.bytes)) * 100 / df.bytes) "% Used"
FROM dba_free_space fs,
(SELECT tablespace_name,SUM(bytes) bytes
FROM dba_data_files
GROUP BY tablespace_name) df
WHERE fs.tablespace_name (+) = df.tablespace_name
GROUP BY df.tablespace_name,df.bytes
Order by 4;
```

###  Shows tablespace utilization
```

set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

column tsname format a30 head "Tablespace Name"
column allocated format 999,999,990.99 head "Allocated MB"
column used format 999,999,990.99 head "Used MB"
column free format 999,999,990.99 head "Free MB"
column pct_used format 990.99 head "Pct Used"
column used_over_allocated format 990.99 head "Pct Used|of Allocated"
column used_over_max format 990.99 head "Pct Used|of Max"
column max format 999,999,990.99 head "Max Size MB"
compute sum of allocated on report
compute sum of used on report
compute sum of free on report
break on report

accept tsn prompt 'Table Space Name [*] : '

col tsn new_value tsn noprint

select decode('&tsn','','','where tablespace_name like '||chr(39)||upper('%&tsn%')||chr(39)) tsn
from   dual
/

SELECT ts.tablespace_name
       ||df.auto
       ||decode(ts.extent_management, 'LOCAL','','(**DMT**)')
       ||decode(df.file_type,'Data','','('||df.file_type||')')
       ||decode(ts.status,'ONLINE','','(**'||ts.status||'**)') tsname
      ,df.allocated - nvl(fs.free,0) used
      ,nvl(fs.free, 0) free
      ,df.allocated
      ,((df.allocated - nvl(fs.free,0)) / df.allocated) * 100  used_over_allocated
      ,max
      ,((df.allocated - nvl(fs.free,0)) / df.max) * 100  used_over_max
FROM
      (SELECT tablespace_name
             ,decode(sum(decode(AUTOEXTENSIBLE,'NO',0,1)),0,'',' (Auto)') auto
             ,'Data' file_type
             ,SUM(bytes)/1048576 allocated
             ,SUM(decode(maxbytes,0,bytes,maxbytes))/1048576 max
       FROM dba_data_files &tsn
       GROUP BY tablespace_name
       union
       SELECT tablespace_name
             ,decode(sum(decode(AUTOEXTENSIBLE,'NO',0,1)),0,'',' (Auto)') auto
             ,'Temp' file_type
             ,SUM(bytes)/1048576 allocated
             ,SUM(decode(maxbytes,0,bytes,maxbytes))/1048576 max
       FROM dba_temp_files &tsn
       GROUP BY tablespace_name) df
     ,(SELECT tablespace_name
             ,SUM(bytes)/1048576 free
       FROM dba_free_space &tsn
       GROUP BY tablespace_name) fs
     ,(SELECT tablespace_name
             ,status
             ,extent_management
       FROM dba_tablespaces &tsn) ts
WHERE ts.tablespace_name = df.tablespace_name(+)
  and ts.tablespace_name = fs.tablespace_name(+)
order by 5;
```
```

set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
column tablespace_name format a25
column pct_used format 990.99
column total format 999,999,999,990
column used format 999,999,999,990
column free format 999,999,999,990
compute sum of total on report
compute sum of used on report
compute sum of free on report
break on report

SELECT a.tablespace_name
      ,a.total
      ,a.total - nvl(b.free,0) used
      ,nvl(b.free, 0) free
      ,( (a.total - nvl(b.free,0)) / a.total) * 100 pct_used
FROM
      (SELECT tablespace_name
             ,SUM(bytes) total
       FROM dba_data_files
       GROUP BY tablespace_name) a
     ,(SELECT tablespace_name
             ,SUM(bytes) free
       FROM dba_free_space
       GROUP BY tablespace_name) b
     , dba_tablespaces c
WHERE
  c.tablespace_name = a.tablespace_name
  and c.tablespace_name = b.tablespace_name(+);
```
### High Water Marks
--
-- Show the High Water Mark for a given table, or all tables if ALL is specified for Table_Name.
--
``` 
SET LINESIZE 300
SET SERVEROUTPUT ON
SET VERIFY OFF
 
DECLARE
  CURSOR cu_tables IS
    SELECT a.owner,
           a.table_name
    FROM   all_tables a
    WHERE  a.table_name = Decode(Upper('&&Table_Name'),'ALL',a.table_name,Upper('&&Table_Name'))
    AND    a.owner      = Upper('&&Table_Owner') 
    AND    a.partitioned='NO'
    AND    a.logging='YES'
order by table_name;
 
  op1  NUMBER;
  op2  NUMBER;
  op3  NUMBER;
  op4  NUMBER;
  op5  NUMBER;
  op6  NUMBER;
  op7  NUMBER;
BEGIN
 
  Dbms_Output.Disable;
  Dbms_Output.Enable(1000000);
  Dbms_Output.Put_Line('TABLE                             UNUSED BLOCKS     TOTAL BLOCKS  HIGH WATER MARK');
  Dbms_Output.Put_Line('------------------------------  ---------------  ---------------  ---------------');
  FOR cur_rec IN cu_tables LOOP
    Dbms_Space.Unused_Space(cur_rec.owner,cur_rec.table_name,'TABLE',op1,op2,op3,op4,op5,op6,op7);
    Dbms_Output.Put_Line(RPad(cur_rec.table_name,30,' ') ||
                         LPad(op3,15,' ')                ||
                         LPad(op1,15,' ')                ||
                         LPad(Trunc(op1-op3-1),15,' ')); 
  END LOOP;
 
END;
/
```


### Shows tablespace object utilization
```

set pause off
set pagesize 9999
set linesize 132
set feedback off
set echo off
set verify off

column TABLESPACE_NAME format a30 head "Tablespace Name"
column OWNER format a15 head "Owner Name"
column SEGMENT_NAME format a30 head "Object Name"
column SEGMENT_TYPE format a12 head "Object Type"
column EXTENTS format 999,990 head "Extents"
column BYTES format 999,999,999,999,990 head "Bytes"
compute sum of BYTES on report
break on report

accept tsn prompt 'Table Space Name [*] : '
accept own prompt '      Owner Name [*] : '
accept seg prompt '    Segment Name [*] : '
accept typ prompt '    Segment Type [*] : '

col tsn new_value tsn noprint
col own new_value own noprint
col seg new_value seg noprint
col typ new_value typ noprint

select TABLESPACE_NAME
     , OWNER
     , SEGMENT_NAME
     , SEGMENT_TYPE
     , EXTENTS
     , BYTES
from dba_segments
where TABLESPACE_NAME like decode('&tsn','','%',upper('&tsn'))
  and OWNER like decode('&own','','%',upper('&own'))
  and SEGMENT_NAME like decode('&seg','','%',upper('&seg'))
  and SEGMENT_TYPE like decode('&typ','','%',upper('&typ'))
order by bytes
/
```


