## Identifying Installed Plugins

Identify all plugins installed on your system using the query provided in the documentation, run as SYSMAN against your repository database.
```
SELECT epv.display_name, epv.plugin_id, epv.version, epv.rev_version,decode(su.aru_file, null, 'Media/External', 'https://updates.oracle.com/Orion/Services/download/'||aru_file||'?aru='||aru_id||chr(38)||'patch_file='||aru_file) URL
FROM em_plugin_version epv, em_current_deployed_plugin ecp, em_su_entities su
WHERE epv.plugin_type NOT IN ('BUILT_IN_TARGET_TYPE', 'INSTALL_HOME')
AND ecp.dest_type='2'
AND epv.plugin_version_id = ecp.plugin_version_id
AND su.entity_id = epv.su_entity_id;
```

## JOB List
```
SELECT j.job_id ,
  e.execution_id ,
  j.job_name     ,
  j.job_type     ,
  j.job_status   ,
  e.status_detail,
  TO_CHAR(e.start_time,'DD-MON-YYYY HH24:MI:SS') stime
   FROM mgmt_job j,
  mgmt_job_exec_summary e
  WHERE j.job_id = e.job_id;
```
## Check Plugin
```
SQL> connect sysman/Oracle#123
SQL> SELECT
    epv.display_name
     , epv.plugin_id
     , epv.version
     , epv.rev_version
     , decode(su.aru_file, null,'Media/External' ,
     'https://updates.oracle.com/Orion/Services/download/' ||
     aru_file||'?aru='||aru_id||chr(38)||'patch_file='||aru_file) URL
     FROM em_plugin_version epv
    , em_current_deployed_plugin ecp
    , em_su_entities su
    WHERE epv.plugin_type NOT IN ('BUILT_IN_TARGET_TYPE', 'INSTALL_HOME')
    AND ecp.dest_type='2'
    AND epv.plugin_version_id = ecp.plugin_version_id
    AND su.entity_id = epv.su_entity_id;
```

## BLOCK Agent
```
SELECT target_name, blocked_reason_msg, blocked_reason_nls_id, blocked_reason_nls_params, to_char(blocked_timestamp, 'yyyy.mm.dd hh24:mi:ss') as blocked_ts, blocked_by, blocked_code FROM mgmt_targets tgt, mgmt_blocked_agents blk WHERE tgt.target_guid = blk.target_guid;
```

[http://cloudcontrol12c.blogspot.com.au/2014/03/em12c-database-data-inventory.html]
```
SELECT DISTINCT
             tbl_tar.target_guid,
            tbl_sid.sid AS instance_name,
             CASE
                WHEN tbl_tar.host_name LIKE '%.%'
                THEN
                   LOWER (SUBSTR (tbl_tar.host_name,
                                  1,
                                    INSTR (tbl_tar.host_name,
                                           '.',
                                           2,
                                           1)
                                  - 1))
                ELSE
                   LOWER (tbl_tar.host_name)
             END
                host_name,
             DECODE (tbl_ava.current_status,
                     0, 'Down',
                     1, 'Up',
                     2, 'Metric Error',
                     3, 'Agent Down',
                     4, 'Unreachable',
                     5, 'Blackout',
                     6, 'Unknown')
                status,
             tbl_groups.composite_target_name AS "GROUP",
             tbl_ver.version,
             CASE
                WHEN tbl_mem.mem_max > 0
                THEN
                   CEIL (tbl_mem.mem_max / 1024 / 1024)
                ELSE
                   CEIL (tbl_sga.sga / 1024 / 1024 + tbl_pga.pga / 1024 / 1024)
             END
               total_memory,
             tbl_dg.data_guard_status,
             tbl_port.port,
             tbl_home.PATH,
            tbl_company.company,
           tbl_location.location,
            tbl_appcontact.app_contact,
           tbl_costcenter.cost_center,
             tbl_tier.tier,
             tbl_department.department,
             tbl_dbplatform.db_platform,
             tbl_dbhostos.db_host_os,
             tbl_comment.notes
        FROM (SELECT p.target_guid, p.property_value AS port
                FROM mgmt_target_properties p
               WHERE p.property_name = 'Port') tbl_port,
             (SELECT s.target_guid, UPPER (s.property_value) AS sid
                FROM mgmt_target_properties s
               WHERE s.property_name = 'SID') tbl_sid,
             (SELECT s.target_guid, s.property_value AS version
                FROM mgmt_target_properties s
               WHERE s.property_name IN ('Version')) tbl_ver,
             (SELECT s.target_guid, s.property_value AS PATH
                FROM mgmt_target_properties s
               WHERE s.property_name IN ('OracleHome')) tbl_home,
             (SELECT s.target_guid, s.property_value AS data_guard_status
                FROM mgmt_target_properties s
               WHERE s.property_name IN ('DataGuardStatus')) tbl_dg,
             (SELECT s.target_guid, s.VALUE AS PGA
                FROM mgmt$db_init_params s
               WHERE s.name = 'pga_aggregate_target') tbl_pga,
             (SELECT s.target_guid, s.VALUE AS SGA
                FROM mgmt$db_init_params s
               WHERE s.name = 'sga_max_size') tbl_sga,
             (SELECT s.target_guid, s.VALUE AS mem_max
                FROM mgmt$db_init_params s
               WHERE s.name = 'memory_target') tbl_mem,
             (SELECT p.target_guid, p.property_value AS notes
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_comment') tbl_comment,
             (SELECT p.target_guid, p.property_value AS company
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_line_of_bus') tbl_company,
             (SELECT p.target_guid, p.property_value AS location
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_location') tbl_location,
             (SELECT p.target_guid, p.property_value AS app_contact
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_contact') tbl_appcontact,
             (SELECT p.target_guid, p.property_value AS cost_center
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_cost_center') tbl_costcenter,
             (SELECT p.target_guid, p.property_value AS tier
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_lifecycle_status') tbl_tier,
             (SELECT p.target_guid, p.property_value AS department
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_department') tbl_department,
             (SELECT p.target_guid, p.property_value AS db_platform
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_platform') tbl_dbplatform,
             (SELECT p.target_guid, p.property_value AS db_host_os
                FROM mgmt_target_properties p
               WHERE p.property_name = 'orcl_gtp_os') tbl_dbhostos,
             mgmt_target_properties tbl_main,
             mgmt_targets tbl_tar,
             mgmt_current_availability tbl_ava,
             (SELECT composite_target_name, member_target_guid
                FROM MGMT_TARGET_MEMBERSHIPS
               WHERE     composite_target_type = 'composite'
                     AND composite_target_name IN
                            ('Production', 'Non-Production', 'SuperCluster')
                     AND member_target_type = 'oracle_database') tbl_groups
       WHERE     tbl_main.target_guid = tbl_port.target_guid(+)
             AND tbl_main.target_guid = tbl_sid.target_guid(+)
             AND tbl_main.target_guid = tbl_tar.target_guid(+)
             AND tbl_main.target_guid = tbl_ver.target_guid(+)
             AND tbl_main.target_guid = tbl_home.target_guid(+)
             AND tbl_main.target_guid = tbl_dg.target_guid(+)
             AND tbl_main.target_guid = tbl_pga.target_guid(+)
             AND tbl_main.target_guid = tbl_sga.target_guid(+)
             AND tbl_main.target_guid = tbl_mem.target_guid(+)
             AND tbl_main.target_guid = tbl_ava.target_guid(+)
             AND tbl_main.target_guid = tbl_comment.target_guid(+)
             AND tbl_main.target_guid = tbl_company.target_guid(+)
             AND tbl_main.target_guid = tbl_location.target_guid(+)
             AND tbl_main.target_guid = tbl_appcontact.target_guid(+)
             AND tbl_main.target_guid = tbl_costcenter.target_guid(+)
             AND tbl_main.target_guid = tbl_tier.target_guid(+)
             AND tbl_main.target_guid = tbl_department.target_guid(+)
             AND tbl_main.target_guid = tbl_dbplatform.target_guid(+)
             AND tbl_main.target_guid = tbl_dbhostos.target_guid(+)
             AND tbl_main.target_guid = tbl_groups.member_target_guid(+)
             AND tbl_tar.target_type = 'oracle_database'
    GROUP BY tbl_tar.target_guid,
             tbl_port.port,
             tbl_sid.sid,
             tbl_tar.host_name,
             tbl_ver.version,
             tbl_home.PATH,
             tbl_dg.data_guard_status,
             tbl_pga.pga,
             tbl_sga.sga,
             tbl_mem.mem_max,
             tbl_ava.current_status,
             tbl_groups.composite_target_name,
             tbl_comment.notes,
             tbl_company.company,
             tbl_location.location,
             tbl_appcontact.app_contact,
             tbl_costcenter.cost_center,
             tbl_tier.tier,
             tbl_department.department,
             tbl_dbplatform.db_platform,
             tbl_dbhostos.db_host_os
    ORDER BY 2;
```
 
## Note: If you are going to use another account other than SYSMAN to select you will need to have the following privilege.
```
grant select on mgmt$storage_report_data to <username>;
grant select on mgmt_target_properties to <username>;
grant select on mgmt_targets to <username>;
grant exempt access policy to <username>;
```
```
select target_type,target_name 
from mgmt_targets_delete d 
where delete_complete_time is null 
group by target_type,target_name 
having count(*) > 1 ; 
```
## Report on OEM Alert metrics
```
SQL> SELECT to_char(s.load_timestamp,'yyyy/mm/dd hh24:mi:ss') ts,
 decode(v.violation_level,18,'Insecure/Invalid state',20,'Warning',25,'Critical',125,'Agent Unreachable',325,'Metric Error','Unkno    wn') alert_level,
 t.target_type, t.target_name, v.message
FROM sysman.mgmt_severity s, sysman.mgmt_targets t, sysman.mgmt_annotation a, sysman.mgmt_notification_log l, sysman.mgmt_violatio    ns v
WHERE s.target_guid = t.target_guid
and s.target_guid = v.target_guid and s.load_timestamp = v.load_timestamp
AND s.severity_guid = a.source_obj_guid (+)
AND s.severity_guid = l.source_obj_guid (+)
AND substr(l.message,1,11) = 'E-mail sent'
and v.violation_level in (18,20,25,125,325)
and s.load_timestamp > sysdate-4
ORDER BY 1
 /
```

```
SYSMAN@POEM1 SQL> SELECT 'OEM' source, to_char(s.load_timestamp,'yyyy/mm/dd') alert_date, t.target_type, t.target_name,
 decode(v.violation_level,18,'Insecure/Invalid state',20,'Warning',25,'Critical',125,'Agent Unreachable',325,'Metric Error','Unknown') alert_level,
 count(1) total
FROM sysman.mgmt_severity s, sysman.mgmt_targets t, sysman.mgmt_annotation a, sysman.mgmt_notification_log l, sysman.mgmt_violations v
WHERE s.target_guid = t.target_guid
and s.target_guid = v.target_guid and s.load_timestamp = v.load_timestamp
AND s.severity_guid = a.source_obj_guid (+)
AND s.severity_guid = l.source_obj_guid (+)
AND substr(l.message,1,11) = 'E-mail sent'
and s.load_timestamp > sysdate-4
and v.violation_level in (18,20,25,125,325)
group by to_char(s.load_timestamp,'yyyy/mm/dd'), t.target_type, t.target_name,
 decode(v.violation_level,18,'Insecure/Invalid state',20,'Warning',25,'Critical',125,'Agent Unreachable',325,'Metric Error','Unknown')
ORDER BY 2,3
 /
```

## Databases with no flashback enabled
```
SELECT
m.target_name,
t.type_qualifier4  AS Role,
m.column_label     AS Flashback,
m.value            AS Status
FROM
mgmt$metric_current m,
mgmt$target t
WHERE m.metric_label = 'Flash Recovery'
AND m.column_label = 'Flashback On'
AND m.value = 'NO'
--AND m.target_name like '%PRD%'
AND t.type_qualifier4 in('Primary','Physical Standby')
AND t.target_name=m.target_name
AND t.target_guid=m.target_guid
order by t.type_qualifier4 ,
m.value
```

## How to find the Agent’s with missing OH targets and missing home data?

### Agent targets and corresponding OH targets 
```
SELECT agt.*,
home_info.target_name AS OH_TARGET_NAME
FROM mgmt$oh_home_info home_info,
(SELECT tgt.target_name AS TARGET_NAME,
tgt.host_name AS HOST_NAME,
tp.property_value AS AGENT_OH_LOCATION
FROM mgmt$target tgt,
mgmt$target_properties tp
WHERE tgt.target_type = 'oracle_emd'
AND tgt.target_guid = tp.target_guid
AND tp.property_name = 'OracleHome'
) agt
WHERE home_info.home_location(+) = agt.AGENT_OH_LOCATION
AND home_info.host_name(+) = agt.HOST_NAME
```

### Number of Agent targets which are missing corresponding OH targets 
```
SELECT TARGET_NAME
FROM
(SELECT agt.*,
home_info.target_name AS OH_TARGET_NAME
FROM mgmt$oh_home_info home_info,
(SELECT tgt.target_name AS TARGET_NAME,
tgt.host_name AS HOST_NAME,
tp.property_value AS AGENT_OH_LOCATION
FROM mgmt$target tgt,
mgmt$target_properties tp
WHERE tgt.target_type = 'oracle_emd'
AND tgt.target_guid = tp.target_guid
AND tp.property_name = 'OracleHome'
) agt
WHERE home_info.home_location(+) = agt.AGENT_OH_LOCATION
AND home_info.host_name(+) = agt.HOST_NAME
)
WHERE OH_TARGET_NAME IS NULL
```

### Number of hosts in the setup 
```
select count(target_guid)
from 
mgmt$target
where target_type='host'
```
## Number of agents in the setup 
```
select count(target_guid)
from 
mgmt$target
where target_type='oracle_emd'
```
## Number of Agent targets for which OracleHome property is not yet set 
```
SELECT tgt.target_name,
tgt.host_name,
tgt.target_type
FROM mgmt$target tgt
WHERE tgt.target_guid NOT IN
(SELECT tp.target_guid
FROM mgmt$target_properties tp
WHERE tp.property_name = 'OracleHome'
AND tp.target_type = 'oracle_emd'
)
AND tgt.target_type = 'oracle_emd'
```
### Number of agents with proper collections and valid ARU ID (the way patching DPs query)
```
SELECT 
oh.aru_id,
tgt.target_name,
tgt.target_type,
tgt.host_name
FROM 
MGMT_TARGETS tgt,
MGMT_TARGET_PROPERTIES tgt_prop,
MGMT_TARGETS host_tgt, 
MGMT$OH_HOME_INFO oh 
WHERE
tgt_prop.target_guid = tgt.target_guid AND
tgt.target_type = 'oracle_emd' AND
host_tgt.target_name = tgt.host_name AND
host_tgt.target_type = 'host' AND
oh.host_name = host_tgt.target_name AND
oh.home_location = tgt_prop.PROPERTY_VALUE AND
tgt_prop.property_name ='OracleHome' ;
```

### AGENT TARGETS with problems 
a)OH target not yet added
```
SELECT TARGET_NAME as AGENT_INST_TARGET,
HOST_NAME,
AGENT_OH_LOCATION as OH_LOC,
'OH TARGET MISSING' as OH_TARGET,
'NO OH TARGET'as ARU_ID
FROM
(SELECT agt.*,
home_info.target_name AS OH_TARGET_NAME
FROM mgmt$oh_home_info home_info,
(SELECT tgt.target_name AS TARGET_NAME,
tgt.host_name AS HOST_NAME,
tp.property_value AS AGENT_OH_LOCATION
FROM mgmt$target tgt,
mgmt$target_properties tp
WHERE tgt.target_type = 'oracle_emd'
AND tgt.target_guid = tp.target_guid
AND tp.property_name = 'OracleHome'
) agt
WHERE home_info.home_location(+) = agt.AGENT_OH_LOCATION
AND home_info.host_name(+) = agt.HOST_NAME
)
WHERE OH_TARGET_NAME IS NULL
```
### No OH target because no OracleHome property fot agent instance target
``` 
SELECT tgt.target_name as AGENT_INST_TARGET,
tgt.host_name as HOST_NAME,
'NO OH PROPERTY SET' as OH_LOC,
'NO OH TARGET AS NO OH LOC' as OH_TARGET,
'NO ARU AS NO OH TARGET'as ARU_ID
FROM mgmt$target tgt
WHERE tgt.target_guid NOT IN
(SELECT tp.target_guid
FROM mgmt$target_properties tp
WHERE tp.property_name = 'OracleHome'
AND tp.target_type = 'oracle_emd'
)
AND tgt.target_type = 'oracle_emd'
```

### Agent targets which are perfect 
```
select 
tgt.target_name as AGENT_INST_TARGET,
tgt.host_name as HOST_NAME,
tp.OH_LOC,
home_info.target_name as OH_TARGET,
home_info.aru_id as ARU_ID
from 
mgmt$target tgt,
mgmt$oh_home_info home_info,
(
select target_guid, property_value as OH_LOC from mgmt$target_properties where target_type='oracle_emd' and property_name = 'OracleHome'
) tp
where
tgt.target_type = 'oracle_emd' and
tgt.target_guid = tp.target_guid (+) and
home_info.host_name = tgt.host_name and
home_info.home_location(+) = tp.OH_LOC)
9)query to find the status of the OH targets and their last collection times and target creation times
select target_guid, current_status , start_collection_timestamp, creation_date
from mgmt_current_availability , em_manageable_entities
where target_guid in (select target_guid from mgmt$target where emd_url in ( select emd_url from mgmt$target where target_type='oracle_home' and
target_name not in ( select distinct target_name from mgmt$oh_home_info ) and target_name like 'agent12%') and target_type='oracle_emd'
) and target_guid = entity_guid ;
10) query to find the OH targets for which collections haven’t happened yet
select target_name from mgmt$target where target_type='oracle_home' and 
target_name not in ( select distinct target_name from mgmt$oh_home_info ) and target_name like 'agent12%'
```

-----
### ASM
------------
 bytes with this query: 
```
select key_value2,trunc(rollup_timestamp) COLL_DAY , to_char(rollup_timestamp,'HH24MI') COLL_TIME ,trunc(sum(to_number(average))) AV_READS 
from mgmt$metric_hourly where target_guid in ('D426887DFBDBEE5AF6A2F23B59ADBC64' , '39972B08F5E84AD30B79D6B28631B678') 
and metric_name='Instance_Disk_Performance' and metric_column='bytes_read' 
and key_value2='DATA' 
group by key_value2,trunc(rollup_timestamp) , to_char(rollup_timestamp,'HH24MI') 
order by 1,2,3; 
```
I get the read counts with this query: 
```
select key_value2,trunc(rollup_timestamp) COLL_DAY , to_char(rollup_timestamp,'HH24MI') COLL_TIME ,trunc(sum(to_number(average))) AV_READS 
from mgmt$metric_hourly where target_guid in ('D426887DFBDBEE5AF6A2F23B59ADBC64' , '39972B08F5E84AD30B79D6B28631B678') 
and metric_name='Instance_Disk_Performance' and metric_column='reads' 
and key_value2='DATA' 
group by key_value2,trunc(rollup_timestamp) , to_char(rollup_timestamp,'HH24MI') 
order by 1,2,3; 
```
```
TTITLE CENTER BOLD 'Missing EM12c Target Properties' SKIP 2 CENTER ''

select tar.target_name "Target Name"
,      tar.target_type "Target Type"
,      tar.type_display_name "Type Display Name"
from   mgmt$target tar
where  tar.target_type not in ('composite')
and    (not exists   (select 1
                      from mgmt$target_properties prop
                      where tar.target_guid = prop.target_guid
                      and   prop.property_name = 'orcl_gtp_lifecycle_status'
                      )
        or not exists (select 1
                       from mgmt$target_properties prop
                       where tar.target_guid = prop.target_guid
                       and   prop.property_name = 'orcl_gtp_line_of_bus'
                      )
        or not exists (select 1
                       from mgmt$target_properties prop
                       where tar.target_guid = prop.target_guid
                       and   prop.property_name = 'orcl_gtp_cost_center'
                      )
        )
;
```
```
TTITLE CENTER BOLD 'Incorrect EM12c Target Properties' SKIP 2 CENTER ''

select prop.target_name "Target Name"
,      tar.type_display_name "Target Type"
,      pdef.property_display_name "Property Display Name"
,      prop.property_value "Incorrect Propery Value"
from   mgmt$target tar
,      mgmt$target_properties prop
,      mgmt$all_target_prop_defs pdef
where  tar.target_guid    = prop.target_guid
and    prop.property_name = pdef.property_name
and    (( prop.property_name = 'orcl_gtp_cost_center' and prop.property_value <> 'BHP Billiton Iron Ore')
        or     ( prop.property_name = 'orcl_gtp_line_of_bus' and prop.property_value <> 'IM Oracle DBA')
        or     ( prop.property_name = 'orcl_gtp_lifecycle_status' and prop.property_value not in ('Production','Stage','Test','Development','MissionCritical'))
        )
;
```

## To determine the platform id of the target host use emcli with the list_add_host_platforms verb. 
```
emcli list_add_host_platforms -all
```

### I like to also check which named credentials are available. You can do so using the list_named_credentials verb. 
```
$ ./emcli list_named_credentials
Credential Name  Credential Owner  Authenticating target type.  Cred Type Name  Target Name   Target Username
NC_ASMCLUS_SASM  OEM_SEC_USER      osm_cluster                  ASMCreds                      sys
NC_ASMCLUS_SDBA  OEM_SEC_USER      osm_cluster                  ASMCreds                      sys
NC_ASM_SASM      OEM_SEC_USER      osm_instance                 ASMCreds                      sys
NC_ASM_SDBA      OEM_SEC_USER      osm_instance                 ASMCreds                      sys
NC_DBS_NORM      OEM_SEC_USER      oracle_database              DBCreds                       dbsnmp
NC_DBS_SDBA      OEM_SEC_USER      oracle_database              DBCreds                       sys
NC_HOST_GRID     OEM_SEC_USER      host                         HostCreds                     grid
NC_HOST_ORACLE   OEM_SEC_USER      host                         HostCreds                     oracle
NC_HOST_PRIV     OEM_SEC_USER      host                         HostCreds                     root
```

### Generate SQL to remove alerts from specific targets
-----------------------------------------------------------------------------------
```
select t.target_name
,      t.target_type
,      collection_timestamp
,      message
,      'exec em_severity.delete_current_severity(''' ||
           t.target_guid || ''',''' ||
           metric_guid || ''',''' ||
           key_value || ''')' em_severity
from   mgmt_targets t
inner join
       mgmt_current_severity s
on
       t.target_guid = s.target_guid
where
       target_name like '&TARGET'
```
