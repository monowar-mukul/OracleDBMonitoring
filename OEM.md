# SQL Queries for Oracle Enterprise Manager (OEM) Database Management

## Table of Contents
1. [Identifying Installed Plugins](#identifying-installed-plugins)
2. [Job List](#job-list)
3. [Check Plugin](#check-plugin)
4. [Blocked Agents](#blocked-agent)
5. [Report on OEM Alert Metrics](#report-on-oem-alert-metrics)
6. [Databases with No Flashback Enabled](#databases-with-no-flashback-enabled)
7. [Missing OH Targets and Home Data](#missing-oh-targets-and-home-data)
8. [ASM Metrics](#asm)

---

## 1. Identifying Installed Plugins

To identify all plugins installed on your system, run the following query as SYSMAN against your repository database:

```sql
SELECT 
  epv.display_name, 
  epv.plugin_id, 
  epv.version, 
  epv.rev_version, 
  decode(su.aru_file, null, 'Media/External', 'https://updates.oracle.com/Orion/Services/download/' || aru_file || '?aru=' || aru_id || chr(38) || 'patch_file=' || aru_file) URL
FROM 
  em_plugin_version epv, 
  em_current_deployed_plugin ecp, 
  em_su_entities su
WHERE 
  epv.plugin_type NOT IN ('BUILT_IN_TARGET_TYPE', 'INSTALL_HOME')
  AND ecp.dest_type = '2'
  AND epv.plugin_version_id = ecp.plugin_version_id
  AND su.entity_id = epv.su_entity_id;
```

---

## 2. Job List

To get a list of all jobs:

```sql
SELECT 
  j.job_id,
  e.execution_id,
  j.job_name,
  j.job_type,
  j.job_status,
  e.status_detail,
  TO_CHAR(e.start_time, 'DD-MON-YYYY HH24:MI:SS') stime
FROM 
  mgmt_job j,
  mgmt_job_exec_summary e
WHERE 
  j.job_id = e.job_id;
```

---

## 3. Check Plugin

To check installed plugin versions:

```sql
SQL> connect sysman/Oracle#123
SQL> SELECT 
    epv.display_name, 
    epv.plugin_id, 
    epv.version, 
    epv.rev_version, 
    decode(su.aru_file, null, 'Media/External', 'https://updates.oracle.com/Orion/Services/download/' || aru_file || '?aru=' || aru_id || chr(38) || 'patch_file=' || aru_file) URL
FROM 
    em_plugin_version epv, 
    em_current_deployed_plugin ecp, 
    em_su_entities su
WHERE 
    epv.plugin_type NOT IN ('BUILT_IN_TARGET_TYPE', 'INSTALL_HOME')
    AND ecp.dest_type = '2'
    AND epv.plugin_version_id = ecp.plugin_version_id
    AND su.entity_id = epv.su_entity_id;
```

---

## 4. Blocked Agent

To check for blocked agents:

```sql
SELECT 
  target_name, 
  blocked_reason_msg, 
  blocked_reason_nls_id, 
  blocked_reason_nls_params, 
  to_char(blocked_timestamp, 'yyyy.mm.dd hh24:mi:ss') as blocked_ts, 
  blocked_by, 
  blocked_code 
FROM 
  mgmt_targets tgt, 
  mgmt_blocked_agents blk 
WHERE 
  tgt.target_guid = blk.target_guid;
```

---

## 5. Report on OEM Alert Metrics

To generate a report on alert metrics:

```sql
SQL> 
SELECT 
  to_char(s.load_timestamp, 'yyyy/mm/dd hh24:mi:ss') ts,
  decode(v.violation_level, 
         18, 'Insecure/Invalid state',
         20, 'Warning', 
         25, 'Critical', 
         125, 'Agent Unreachable', 
         325, 'Metric Error', 
         'Unknown') alert_level,
  t.target_type, 
  t.target_name, 
  v.message
FROM 
  sysman.mgmt_severity s, 
  sysman.mgmt_targets t, 
  sysman.mgmt_annotation a, 
  sysman.mgmt_notification_log l, 
  sysman.mgmt_violations v
WHERE 
  s.target_guid = t.target_guid
  AND s.target_guid = v.target_guid
  AND s.load_timestamp = v.load_timestamp
  AND s.severity_guid = a.source_obj_guid (+)
  AND s.severity_guid = l.source_obj_guid (+)
  AND substr(l.message, 1, 11) = 'E-mail sent'
  AND v.violation_level IN (18, 20, 25, 125, 325)
  AND s.load_timestamp > sysdate-4
ORDER BY 
  1;
```

---

## 6. Databases with No Flashback Enabled

To check databases without flashback enabled:

```sql
SELECT 
  m.target_name,
  t.type_qualifier4 AS Role,
  m.column_label AS Flashback,
  m.value AS Status
FROM 
  mgmt$metric_current m,
  mgmt$target t
WHERE 
  m.metric_label = 'Flash Recovery'
  AND m.column_label = 'Flashback On'
  AND m.value = 'NO'
  AND t.type_qualifier4 IN ('Primary', 'Physical Standby')
  AND t.target_name = m.target_name
  AND t.target_guid = m.target_guid
ORDER BY 
  t.type_qualifier4, m.value;
```

---

## 7. Missing OH Targets and Home Data

### Agent Targets and Corresponding OH Targets

```sql
SELECT 
  agt.*, 
  home_info.target_name AS OH_TARGET_NAME
FROM 
  mgmt$oh_home_info home_info,
  (SELECT 
      tgt.target_name AS TARGET_NAME, 
      tgt.host_name AS HOST_NAME, 
      tp.property_value AS AGENT_OH_LOCATION
   FROM 
      mgmt$target tgt,
      mgmt$target_properties tp
   WHERE 
      tgt.target_type = 'oracle_emd'
      AND tgt.target_guid = tp.target_guid
      AND tp.property_name = 'OracleHome'
  ) agt
WHERE 
  home_info.home_location(+) = agt.AGENT_OH_LOCATION
  AND home_info.host_name(+) = agt.HOST_NAME;
```

### Missing OH Targets

```sql
SELECT 
  TARGET_NAME
FROM 
  (SELECT 
      agt.*, 
      home_info.target_name AS OH_TARGET_NAME
   FROM 
      mgmt$oh_home_info home_info,
      (SELECT 
          tgt.target_name AS TARGET_NAME, 
          tgt.host_name AS HOST_NAME, 
          tp.property_value AS AGENT_OH_LOCATION
       FROM 
          mgmt$target tgt,
          mgmt$target_properties tp
       WHERE 
          tgt.target_type = 'oracle_emd'
          AND tgt.target_guid = tp.target_guid
          AND tp.property_name = 'OracleHome'
      ) agt
   WHERE 
      home_info.home_location(+) = agt.AGENT_OH_LOCATION
      AND home_info.host_name(+) = agt.HOST_NAME
  )
WHERE 
  OH_TARGET_NAME IS NULL;
```

---

## 8. ASM Metrics

### Disk Performance (Bytes Read)

```sql
SELECT 
  key_value2, 
  trunc(rollup_timestamp) COLL_DAY, 
  to_char(rollup_timestamp, 'HH24MI') COLL_TIME, 
  trunc(sum(to_number(average))) AV_READS 
FROM 
  mgmt$metric_hourly 
WHERE 
  target_guid IN ('D426887DFBDBEE5AF6A2F23B59ADBC64', '39972B08F5E84AD30B79D6B28631B678') 
  AND metric_name = 'Instance_Disk_Performance' 
  AND metric_column = 'bytes_read' 
  AND key_value2 = 'DATA' 
GROUP BY 
  key_value2, trunc(rollup_timestamp), to_char(rollup_timestamp, 'HH24MI') 
ORDER BY 
  1, 2, 3;
```

### Read Counts

```sql
SELECT 
  key_value2, 
  trunc(rollup_timestamp) COLL_DAY, 
  to_char(rollup_timestamp, 'HH24MI') COLL_TIME, 
  trunc(sum(to_number(average))) AV_READS 
FROM 
  mgmt$metric_hourly 
WHERE 
  target_guid IN ('D426887DFBDBEE5AF6A2F23B59ADBC64', '39972B08F5E84AD30B79D6B28631B678') 
  AND metric_name = 'Instance_Disk_Performance' 
  AND metric_column = 'reads' 
  AND key_value2 = 'DATA' 
GROUP BY 
  key_value2, trunc(rollup_timestamp), to_char(rollup_timestamp, 'HH24MI') 
ORDER BY 
  1, 2, 3;
```

---

This document provides a collection of SQL queries useful for managing Oracle Enterprise Manager (OEM) environments, including tasks such as plugin identification, job status, agent management, alert reporting, and more. Each section offers practical queries with explanations to help you efficiently monitor and manage your OEM setup.
