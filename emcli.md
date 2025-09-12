# Oracle Enterprise Manager EMCLI Reference Guide

## Table of Contents
- [Initial Setup](#initial-setup)
- [Target Management](#target-management)
- [Agent Management](#agent-management)
- [Blackout Management](#blackout-management)
- [Job Management](#job-management)
- [Template Management](#template-management)
- [Backup and Export](#backup-and-export)
- [Troubleshooting](#troubleshooting)

## Initial Setup

### Environment Variables and Login
```bash
export JAVA_HOME=/u01/app/oracle/product/middleware/oms
export PATH=$JAVA_HOME/jdk/bin:$PATH
cd $OMS_HOME/bin

# Login to EMCLI
./emcli login -username=SYSMAN -password=<sysman_passwd>
./emcli login -username=oem_admin

# Sync with OMS
./emcli sync

# Logout
./emcli logout
```

### Software Library Configuration
```bash
# Add software library storage location
emcli add_swlib_storage_location -name=swlib -path=/some/path/on/OMS/server

# List existing storage locations
emcli list_swlib_storage_locations
```

## Target Management

### Database Target Management

#### Add Single Instance Database
```bash
./emcli add_target -name="ORCL_DB" \
  -type="oracle_database" \
  -host="oradb1.example.com" \
  -properties="SID:orcl;Port:1521;OracleHome:/u01/app/oracle/product/12.1.0/dbhome_1;MachineName:oradb1-vip.example.com" \
  -credentials="Role:NORMAL;UserName:DBSNMP;password:<password>"
```

#### Add Database on Windows
```bash
./emcli add_target -name="PMJS1" \
  -type="oracle_database" \
  -host="CSMJIM-VODBP05.apac.ent.bhpbilliton.net" \
  -credentials="UserName:dbsnmp;password:oem0racle1;Role:Normal" \
  -properties="SID&PMJS1;Port&1521;OracleHome&\"c:\app\oracle\product\11.2.0\dbhome_1\";MachineName&CSMJIM-VODBP05" \
  -subseparator=properties="&"
```

#### Add RAC Database Instances
```bash
# Add first RAC instance
./emcli add_target -name="racdb_racdb_1" \
  -type="oracle_database" \
  -host="test1.oracle.com" \
  -credentials="UserName:dbsnmp;password:welcome1;Role:Normal" \
  -properties="SID:racdb_1;Port:1521;OracleHome:/u01/racdb/11202;MachineName:test2-vip1.oracle.com"

# Add second RAC instance
./emcli add_target -name="racdb_racdb_2" \
  -type="oracle_database" \
  -host="test2.oracle.com" \
  -credentials="UserName:dbsnmp;password:welcome1;Role:Normal" \
  -properties="SID:racdb_2;Port:1521;OracleHome:/u01/racdb/11202;MachineName:test2-vip2.oracle.com"

# Add RAC database cluster
./emcli add_target -name="racdb" \
  -type="rac_database" \
  -host="test1.oracle.com" \
  -monitor_mode="1" \
  -properties="ServiceName:racdb;ClusterName:emcluster" \
  -instances="racdb_racdb_1:oracle_database;racdb_racdb_2:oracle_database"
```

### Target Modification and Renaming

#### Modify Target Display Name
```bash
emcli modify_target -name="CSMYAN-VODBP01" -type="host" -display_name="csmyan-vodbp01" -on_agent
```

#### Rename Targets
```bash
# Rename host target
emcli rename_target -target_type="host" -target_name="10.113.95.240" -new_target_name="csmjim-vodbq01"

# Rename agent target
emcli rename_target -target_type="oracle_emd" -target_name="10.113.77.34:3872" -new_target_name="csmjim-vodbp01:3872"
```

### Delete Targets
```bash
# Delete database target
./emcli delete_target -name="SMJS1_01" -type="oracle_database"

# Delete database system target
./emcli delete_target -name="PMJS2_sys" -type="oracle_dbsys"

# Delete RAC database
./emcli delete_target -name="racdb" -type="rac_database"

# Using SQL (alternative method)
exec mgmt_admin.delete_target('target_name','target_type');
exec mgmt_admin.delete_target('DBNAME','oracle_database');
```

### List Targets
```bash
# List all targets
emcli get_targets

# List specific target pattern
emcli get_targets -targets="TMJS:%"

# List specific target type
emcli get_targets -targets="oracle_emd" -name="name:csv"
```

## Agent Management

### Agent Operations
```bash
# Stop agent
emcli stop_agent -agent=acme_qa:3872 -host_username=oracle

# Start agent
emcli start_agent -agent=acme_qa:3872 -host_username=oracle

# Restart agent
emcli restart_agent -agent=acme_qa:3872 -host_username=oracle

# Secure agent
emcli secure_agent -agent=acme_qa:3872 -host_username=oracle

# Resecure agent
emcli resecure_agent -agent=acme_qa:3872 -host_username=oracle

# Resync agent
emcli resyncAgent -agent=acme_qa:3872

# Get agent properties
emcli get_agent_properties -agent_name=acme_qa:3872
```

### Target Relocation
```bash
# Relocate targets between agents
emcli relocate_targets -src_agent=exadb02.localhost.localdomain:3872 \
  -dest_agent=exadb04.localhost.localdomain:3872 \
  -target_name=exacel05.localhost.localdomain \
  -target_type=oracle_exadata \
  -copy_from_src
```

## Blackout Management

### Create Blackouts
```bash
# Single target blackout
emcli create_blackout -name='Blackout1' \
  -add_targets='em12cr3.example.com:host' \
  -schedule='frequency:once;duration:3' \
  -reason='Testing'

# Multiple database blackout
emcli create_blackout -name='DB_Blackout' \
  -add_targets="db1_name:oracle_database;db2_name:oracle_database" \
  -schedule="frequency:once;duration:-1" \
  -reason='Maintenance'

# RAC database blackout
./emcli create_blackout -name='Blackout1' \
  -add_targets='BHTEST:rac_database' \
  -schedule="frequency:once;duration:-1" \
  -reason='Testing'

# Server and associated services blackout
emcli create_blackout -name="Server_Blackout" \
  -reason="reboot due to memory errors" \
  -add_targets="server01:host" \
  -schedule="duration:-1" \
  -propagate_targets
```

### Manage Blackouts
```bash
# List blackouts
emcli get_blackouts -name=Blackout_Name
emcli get_blackouts -noheader -script | cut -f1

# Get blackout details
emcli get_blackout_details -name='Blackout1'

# Stop blackout
emcli stop_blackout -name='Blackout1'

# Delete blackout
emcli delete_blackout -name='Blackout1'
```

## Job Management

### Job Operations
```bash
# List job types
emcli get_job_types

# Describe job type
emcli describe_job_type -job_type=discoverPromoteOHTargets

# Create job from property file
emcli create_job -input_file=property_file:inputs.prop

# Get jobs
emcli get_jobs -name=PROMOTE_OH
emcli get_jobs -status=2

# Get job execution details
emcli get_job_execution_detail -execution=<Execution_ID> -xml -showOutput
```

### Job Import/Export
```bash
# Export jobs (preview)
./emcli export_jobs -preview

# Export jobs by owner
./emcli export_jobs -export_file=/u01/export/all_regular_jobs.zip -owner=OEM_JOB_USER

# Import jobs (preview)
./emcli import_jobs -preview -import_file=/u01/export/all_regular_jobs.zip

# Import jobs
./emcli import_jobs -import_file=/u01/export/all_regular_jobs.zip
```

## Template Management

### Export Templates
```bash
# Export database templates
./emcli export_template -name="ORA_DEV_DATABASE_INSTANCE_TEMPLATE" \
  -target_type="oracle_database" > /u01/export/DEV_Database.xml

./emcli export_template -name="ORA_DEV_CLUSTER_DATABASE_TEMPLATE" \
  -target_type="rac_database" > /u01/export/DEV_Cluster_Database.xml

./emcli export_template -name="ORA_DEV_ASM_TEMPLATE" \
  -target_type="osm_instance" > /u01/export/ORA_DEV_ASM_TEMPLATE.xml

./emcli export_template -name="ORA_DEV_LISTENER_TEMPLATE" \
  -target_type="oracle_listener" > /u01/export/ORA_DEV_LISTENER_TEMPLATE.xml

./emcli export_template -name="ORA_DEV_HOST_TEMPLATE" \
  -target_type="host" > /u01/export/ORA_DEV_HOST_TEMPLATE.xml
```

## Backup and Export

### Deployment Procedures (PAR Files)
```bash
# List procedures
emcli get_procedures | grep -i BHPB

# Check PAR tool
emctl partool check

# Export deployment procedures
emctl partool export -guid "<GUID>" \
  -file "/tmp/Create_Oracle_Database.par" \
  -displayName "DP_NAME" \
  -description "description" \
  -repPasswd <password>

# Import deployment procedures
emctl partool deploy -parFile $ORACLE_HOME/sysman/prov/paf/<par_file_name>
```

## Troubleshooting

### Common Issues

#### EMD_URL is invalid Error
When encountering "ORA-20247: EMD_URL is invalid: Cannot provide null emd url":
1. Check agent URL with `emctl status agent`
2. Use the correct hostname from Agent URL in the add_target command

#### Windows Target Addition
For Windows targets, ensure:
- Correct Oracle Home path format with double quotes
- Proper host resolution
- Agent is running and accessible

#### Agent Synchronization Issues
```bash
# Resync agent
emcli resyncAgent -agent=<agent_name>

# Check agent status
emctl status agent
```

### Repository Queries

#### Check Host Status
```sql
SELECT TARGET_NAME target,
       decode(MANAGE_STATUS,2,'Managed',MANAGE_STATUS) status,
       HOST_NAME host,
       EMD_URL
FROM mgmt_targets
WHERE TARGET_TYPE = 'host'
  AND (TARGET_NAME in ('10.113.95.146', 'CSMJIM-VODBP05') 
       OR TARGET_NAME like 'CSMJIM%');
```

#### Check Target Monitoring
```sql
SELECT target_name, target_type, agent_name, agent_type, agent_is_master
FROM MGMT$AGENTS_MONITORING_TARGETS
WHERE target_name = 'exacel01.localhost.localdomain';
```

### Debug and Logging

#### Enable OMS Debug
```bash
cd <OMS_HOME>/bin
./emctl set property -name log4j.rootCategory \
  -value 'DEBUG, emlogAppender, emtrcAppender' \
  -module logging
```

#### Enable Agent Debug
```bash
cd <AGENT_HOME>/bin
emctl setproperty agent -name 'Logger.log.level' -value 'DEBUG'
```

#### Disable Debug Logging
```bash
# OMS Server
./emctl set property -name log4j.rootCategory \
  -value 'WARN, emlogAppender, emtrcAppender' \
  -module logging

# Agent Server
./emctl setproperty agent -name 'Logger.log.level' -value 'INFO'
```

## File Locations

### Important Paths
- **OMS Log**: `/u01/app/oracle/product/12.1.0/gc_inst/em/EMGC_OMS1/sysman/log/emoms.log`
- **Agent Log**: `/u01/app/oracle/product/12.1.0/agent/agent_inst/sysman/log/emctl.log`
- **EMCLI Setup**: `/u01/app/oracle/product/12.1.0/emcli/emcli_dir`

### EMCLI Installation
```bash
# Install EMCLI
cd /u01/app/oracle/product/12.1.0/emcli
java -jar emclikit.jar client -install_dir=/u01/app/oracle/product/12.1.0/emcli

# Setup EMCLI
./emcli setup -dir=/u01/app/oracle/product/12.1.0/emcli/emcli_dir \
  -url="https://iorper-oms01:7802/em" \
  -username=oem_admin \
  -autologin \
  -trustall
```

## Notes

- Duration = "-1" denotes unlimited duration for blackouts
- Always sync EMCLI after major operations: `./emcli sync`
- Use proper subseparator for Windows paths: `-subseparator=properties="&"`
- Agent URLs must match exactly for target operations
- Repository queries require SYSMAN privileges

## References

- Oracle Enterprise Manager Documentation
- EMCLI Command Reference
- Oracle Support Documents mentioned in original content
