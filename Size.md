## Database Size with used and free space
```
col "Database Size" format a20
col "Free space" format a20
col "Used space" format a20
select  round(sum(used.bytes) / 1024 / 1024 / 1024 ) || ' GB' "Database Size"
,       round(sum(used.bytes) / 1024 / 1024 / 1024 ) -
        round(free.p / 1024 / 1024 / 1024) || ' GB' "Used space"
,       round(free.p / 1024 / 1024 / 1024) || ' GB' "Free space"
from    (select bytes
        from    v$datafile
        union   all
        select  bytes
        from    v$tempfile
        union   all
        select  bytes
        from    v$log) used
,       (select sum(bytes) as p
        from dba_free_space) free
group by free.p
/
```
Sample Output
```
Database Size        Used space           Free space
-------------------- -------------------- --------------------
115 GB               85 GB                30 GB

```
```
SELECT 'Database Size' "*****"
      ,round(sum(round(sum(nvl(fs.bytes/1024/1024,0)))) / 
        sum(round(sum(nvl(fs.bytes/1024/1024,0))) + round
        (df.bytes/1024/1024 , sum(nvl(fs.bytes/1024/1024,0))))
        * 100, 0) "%Free"
      ,round(sum(round(df.bytes/1024/1024 ,
        sum(nvl(fs.bytes/1024/1024,0)))) / sum(round(sum
        (nvl(fs.bytes/1024/1024,0))) + round(df.bytes/1024/
        1024 , sum(nvl(fs.bytes/1024/1024,0)))) * 100, 0) 
        "%Used"
      ,sum(round(sum(nvl(fs.bytes/1024/1024,0)))) "Mb Free"
      ,sum(round(df.bytes/1024/1024
      , sum(nvl(fs.bytes/1024/1024,0)))) "Mb Used"
      ,sum(round(sum(nvl(fs.bytes/1024/1024,0))) + round
        (df.bytes/1024/1024
      , sum(nvl(fs.bytes/1024/1024,0)))) "Size"
FROM dba_free_space fs, dba_data_files df
WHERE fs.file_id(+) = df.file_id
GROUP BY df.tablespace_name, df.file_id, df.bytes,
   df.autoextensible
ORDER BY df.file_id;
```
Sample Output
```
*****              %Free      %Used    Mb Free    Mb Used       Size
------------- ---------- ---------- ---------- ---------- ----------
Database Size         22         78      30568  107221.25  137789.25

```

## Size (Datafiles, temp files and redo members)
```
select 'Data Files' type, sum(bytes)/1024/1024/1024 szgb,count(*) cnt
from v$datafile group by substr(name,1,instr(name,'/',1,2)-1)
union
select 'Temp Files', sum(bytes)/1024/1024/1024 szgb,count(*) cnt
from v$tempfile group by substr(name,1,instr(name,'/',1,2)-1)
union
select 'Redo Member',sum(l.bytes)/1024/1024/1024 szgb,count(*) cnt
from v$logfile lf, v$log l
where l.group#=lf.group# group by substr(member,1,instr(member,'/',1,2)-1)
/
```
Sample Output
```
TYPE              SZGB        CNT
----------- ---------- ----------
Data Files  104.708252         16
Redo Member     .78125          4
Temp Files          10          1
```

### Schema Size 

```
set pagesize 10000
BREAK ON REPORT
COMPUTE SUM LABEL TOTAL OF "Size of Each Segment in MB" ON REPORT
select segment_type, sum(bytes/1024/1024) "Size of Each Segment in MB" 
from dba_segments where owner='&Owner' 
group by segment_type order by 1;
```
## Segment SIze based on Type
```
set pagesize 10000
BREAK ON REPORT
COMPUTE SUM LABEL TOTAL OF "Size of Each Segment in GB" ON REPORT
select segment_type, sum(bytes/1024/1024/1024) "Size of Each Segment in GB" 
from dba_segments 
--where owner='BPELADMIN' 
group by segment_type order by 1;
```
### Sample Output
```
SEGMENT_TYPE       Size of Each Segment in MB
------------------ --------------------------
INDEX                               .179870605
LOBINDEX                        .00994873
LOBSEGMENT               12.975464
TABLE                              2.84216309
                   --------------------------
TOTAL                              216.007446
```
```
set pagesize 10000
BREAK ON REPORT
COMPUTE SUM LABEL TOTAL OF "Size of Each Segment in GB" ON REPORT
select segment_type, sum(bytes/1024/1024/1024) "Size of Each Segment in GB" 
from dba_segments 
where owner='&owner' 
And segment_name='&Segment'
group by segment_type order by 1;

```
```
select extent_id, bytes, blocks
from dba_extents
where segment_name = '&segmentname'
and segment_type = 'TABLE'
and owner='&owner';
```

## Table Size
```
select sum(bytes)/(1024*1024), segment_name
from dba_segments
where segment_name='&TABLE_NAME'
and owner='&owner'
group by segment_name;
```
### Tablespaces Status
consider the last column "MAX_PCT_USED" to identify current tablespace usage regardless autoextend status (ON/OFF).
```
set linesize 300;
SELECT a.tablespace_name TBS_NAME, ROUND (a.bytes_alloc / 1024 / 1024, 0) MEGS_ALLOC,
ROUND (NVL (b.bytes_free, 0) / 1024 / 1024, 0) MEGS_FREE,
ROUND ((a.bytes_alloc - NVL (b.bytes_free, 0)) / 1024 / 1024,0) MEGS_USED,
ROUND ((NVL (b.bytes_free, 0) / a.bytes_alloc) * 100, 2) PCT_FREE,
100 - ROUND ((NVL (b.bytes_free, 0) / a.bytes_alloc) * 100, 2) PCT_USED,
ROUND (maxbytes / 1048576, 2) MAX_MEGS_ALLOC,
100 - ROUND ((a.bytes_alloc - NVL (b.bytes_free, 0)) / maxbytes * 100, 2) MAX_PCT_FREE,
ROUND ((a.bytes_alloc - NVL (b.bytes_free, 0)) / maxbytes * 100, 2) MAX_PCT_USED
FROM (SELECT f.tablespace_name, SUM (f.BYTES) bytes_alloc,
SUM (DECODE (f.autoextensible,'YES',f.maxbytes,'NO', f.BYTES)) maxbytes
FROM dba_data_files f
GROUP BY tablespace_name) a,
(SELECT f.tablespace_name, SUM (f.BYTES) bytes_free
FROM dba_free_space f
GROUP BY tablespace_name) b
WHERE a.tablespace_name = b.tablespace_name(+)
order by 8;
```
### Sample Output:
```
TBS_NAME                       MEGS_ALLOC  MEGS_FREE  MEGS_USED   PCT_FREE   PCT_USED MAX_MEGS_ALLOC MAX_PCT_FREE MAX_PCT_USED
------------------------------ ---------- ---------- ---------- ---------- ---------- -------------- ------------ ------------
SYSAUX                               1077        146        932      13.52      86.48       32767.98        97.16         2.84
UNDOTBS1                            31665      31366        299      99.05        .95       32767.98        99.09          .91
ORABPEL                            216840      12428     204412       5.73      94.27      225839.92         9.49        90.51
BPELADMIN                           93756       1788      91968       1.91      98.09       99035.97         7.14        92.86
USERS                                  42         23         19      54.95      45.05       32767.98        99.94          .06
SYSTEM                               1024        305        719       29.8       70.2       32767.98        97.81         2.19

6 rows selected.
```

### Extent Size
```
COL Object FORMAT a24;
COL Type FORMAT a5;
SELECT segment_name "Object", segment_type "Type"
      ,ROUND(SUM(bytes)/1024/1024) "Mb", ROUND
         (SUM(bytes)/1024) "Kb"
      ,SUM(bytes) "Bytes", SUM(blocks) "Blocks"
FROM dba_extents
WHERE owner = '&User' AND segment_type IN ('TABLE','INDEX')
GROUP BY segment_name, segment_type
ORDER BY segment_name, segment_type DESC;
``` 
### Table space vs Index space
```
SELECT segment_type, ROUND(SUM(size_mb)) size_mb, 
ROUND((RATIO_TO_REPORT(SUM(size_mb)) OVER ())*100, 2)|| '%' PERC 
FROM (SELECT owner, segment_name, segment_type, partition_name, ROUND(bytes/(1024*1024),2) SIZE_MB, tablespace_name 
FROM DBA_SEGMENTS 
WHERE SEGMENT_TYPE IN ('TABLE', 'INDEX', 'TABLE PARTITION', 'INDEX PARTITION', 
'TABLE SUBPARTITION', 'INDEX SUBPARTITION') 
ORDER BY bytes DESC) 
GROUP BY segment_type 
ORDER BY size_mb DESC;
```
### Sample Output
```
SEGMENT_TYPE          SIZE_MB PERC
------------------ ---------- ------
INDEX SUBPARTITION    3365434 34.94%
TABLE SUBPARTITION    3316337 34.43%
TABLE                 1329498 13.8%
TABLE PARTITION        767524 7.97%
INDEX PARTITION        559988 5.81%
INDEX                  292943 3.04%
```

### Grouping like tables against indexes space
```
SELECT SUBSTR(segment_type, 0, 5) SEGMENT, SUM(size_mb) size_mb, 
ROUND((RATIO_TO_REPORT(SUM(size_mb)) OVER ())*100, 2)|| '%' PERC
FROM (
SELECT segment_type, ROUND(SUM(size_mb)) size_mb 
FROM (SELECT owner, segment_name, segment_type, partition_name, ROUND(bytes/(1024*1024),2) SIZE_MB, tablespace_name 
FROM DBA_SEGMENTS 
WHERE SEGMENT_TYPE IN ('TABLE', 'INDEX', 'TABLE PARTITION', 'INDEX PARTITION', 
'TABLE SUBPARTITION', 'INDEX SUBPARTITION') 
ORDER BY bytes DESC) 
GROUP BY segment_type 
ORDER BY size_mb DESC) 
GROUP BY SUBSTR(segment_type, 0, 5);
```
Sample Output:
```
SEGME    SIZE_MB PERC
----- ---------- -----------------------------------------
INDEX      46373 64.83%
TABLE      25158 35.17%
```


#### FRA 
```
set linesize 500
col NAME for a50
select name, ROUND(SPACE_LIMIT/1024/1024/1024,2) "Allocated Space(GB)", 
round(SPACE_USED/1024/1024/1024,2) "Used Space(GB)",
round(SPACE_RECLAIMABLE/1024/1024/1024,2) "SPACE_RECLAIMABLE (GB)" ,
(select round(ESTIMATED_FLASHBACK_SIZE/1024/1024/1024,2) 
from V$FLASHBACK_DATABASE_LOG) "Estimated Space (GB)"
from V$RECOVERY_FILE_DEST;
```

Sample Output:
```
NAME                 Allocated Space(GB) Used Space(GB) SPACE_RECLAIMABLE (GB) Estimated Space (GB)
-------------------- ------------------- -------------- ---------------------- --------------------
+FRA                                 220           27.2                   1.74                12.42
```


### Check Object Growth Report
```
select * from (select to_char(end_interval_time, 'MM/DD/YY') mydate, sum(space_used_delta) / 1024 / 1024 "Space used (MB)", avg(c.bytes) / 1024 / 1024 "Total Object Size (MB)",
round(sum(space_used_delta) / sum(c.bytes) * 100, 2) "Percent of Total Disk Usage"
from
dba_hist_snapshot sn,
dba_hist_seg_stat a,
dba_objects b,
dba_segments c
where begin_interval_time > trunc(sysdate) - &days_back
and sn.snap_id = a.snap_id
and b.object_id = a.obj#
and b.owner = c.owner
and b.object_name = c.segment_name
and c.segment_name = '&segment_name'
group by to_char(end_interval_time, 'MM/DD/YY'))
order by to_date(mydate, 'MM/DD/YY')
/
```
### Database Growth
```
SELECT SUBSTR(B.END_INTERVAL_TIME,1,9) DAY,SUM(A.SPACE_ALLOCATED_DELTA)/1024/1024/1024 "Size GB"
FROM DBA_HIST_SEG_STAT A,DBA_HIST_SNAPSHOT B
WHERE A.SNAP_ID=B.SNAP_ID
GROUP BY SUBSTR(B.END_INTERVAL_TIME,1,9)
ORDER BY TO_DATE(SUBSTR(B.END_INTERVAL_TIME,1,9),'DD-MON-YY')
/
```
Sample Output:
```
DAY          Size GB
--------- ----------
16/JUL/24 .007568359
17/JUL/24 .295166016
18/JUL/24 .248901367
19/JUL/24 .225830078
20/JUL/24 .264526367
21/JUL/24 .213012695
22/JUL/24 .442260742
23/JUL/24    .203125
24/JUL/24 .223144531
25/JUL/24 .246337891
26/JUL/24 .133544922

11 rows selected.
```

```

SET LINESIZE 200
SET PAGESIZE 200
COL "Database Size" FORMAT a13
COL "Used Space" FORMAT a11
COL "Used in %" FORMAT a11
COL "Free in %" FORMAT a11
COL "Database Name" FORMAT a13
COL "Free Space" FORMAT a12
COL "Growth DAY" FORMAT a11
COL "Growth WEEK" FORMAT a12
COL "Growth DAY in %" FORMAT a16
COL "Growth WEEK in %" FORMAT a16

SELECT
(select min(creation_time) from v$datafile) "Create Time",
(select name from v$database) "Database Name",
ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2) || ' MB' "Database Size",
ROUND((SUM(USED.BYTES) / 1024 / 1024 ) - ROUND(FREE.P / 1024 / 1024 ),2) || ' MB' "Used Space",
ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 )) / ROUND(SUM(USED.BYTES) / 1024 / 1024 ,2)*100,2) || '% MB' "Used in %",
ROUND((FREE.P / 1024 / 1024 ),2) || ' MB' "Free Space",
ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - ((SUM(USED.BYTES) / 1024 / 1024 ) - ROUND(FREE.P / 1024 / 1024 )))/ROUND(SUM(USED.BYTES) / 1024 / 1024,2 )*100,2) || '% MB' "Free in %",
ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile),2) || ' MB' "Growth DAY",
ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)/ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2)*100,3) || '% MB' "Growth DAY in %",
ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)*7,2) || ' MB' "Growth WEEK",
ROUND((((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)/ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2)*100)*7,3) || '% MB' "Growth WEEK in %"
FROM (SELECT BYTES FROM V$DATAFILE
UNION ALL
SELECT BYTES FROM V$TEMPFILE
UNION ALL
SELECT BYTES FROM V$LOG) USED,
(SELECT SUM(BYTES) AS P FROM DBA_FREE_SPACE) FREE
GROUP BY FREE.P;
```
Sample Output
```
Create Ti Database Name Database Size Used Space  Used in %   Free Space   Free in %   Growth DAY  Growth DAY in %  Growth WEEK  Growth WEEK in %
--------- ------------- ------------- ----------- ----------- ------------ ----------- ----------- ---------------- ------------ ----------------
17/APR/19 IVUPROD       118261.25 MB  87693.25 MB 74.15% MB   30567.63 MB  25.85% MB   45.5 MB     .038% MB         318.48 MB   .269% MB


```

