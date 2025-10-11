# Oracle Database Monitoring & Maintenance

A comprehensive collection of Oracle Database monitoring, maintenance, and administration queries and scripts. This repository serves as a practical reference guide for DBAs managing Oracle environments, covering everything from daily health checks to advanced RAC and Data Guard operations.

## ⚠️ **IMPORTANT DISCLAIMER**

> **🔴 CRITICAL: VALIDATE BEFORE EXECUTION 🔴**
>
> **ALWAYS validate and test all scripts in a non-production environment before running them on actual or production systems. Executing untested scripts can result in data loss, system downtime, or performance degradation.**
>
> **⚡ LICENSING REQUIREMENTS:**
> - Performance tuning and diagnostic features may require Oracle Diagnostics Pack and/or Tuning Pack licenses
> - Verify your Oracle license entitlements before using performance monitoring queries
> - Unauthorized use of licensed features may result in compliance issues
>
> **📢 USE AT YOUR OWN RISK:**
> - The author assumes NO responsibility for any issues, damages, or losses arising from the use of these scripts
> - You are solely responsible for validating, testing, and ensuring the suitability of these scripts for your environment
> - Always maintain valid backups and follow your organization's change management procedures
>
> **DON'T BLAME ME - YOU'VE BEEN WARNED!**

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Quick Start](#quick-start)
- [Documentation](#documentation)
- [Usage Guidelines](#usage-guidelines)
- [Contributing](#contributing)
- [Prerequisites](#prerequisites)

## Overview

This repository contains battle-tested SQL queries, scripts, and reference guides for Oracle Database administration. Whether you're performing daily health checks, troubleshooting performance issues, or managing complex RAC and Data Guard environments, you'll find practical solutions here.

**Key Features:**
- Production-ready SQL queries and scripts
- Comprehensive monitoring and diagnostic queries
- Step-by-step maintenance procedures
- Best practices and troubleshooting guides
- Support for Oracle 11g through 19c+ versions

## Repository Structure

### 📊 Monitoring & Health Checks

| File | Description |
|------|-------------|
| [DailyHealthCheck.md](DailyHealthCheck.md) | Essential daily monitoring queries for database health assessment |
| [session_history_info.md](session_history_info.md) | Active and historical session monitoring queries |
| [Lock_Session_monitoring.md](Lock_Session_monitoring.md) | Lock detection, blocking session analysis, and resolution scripts |
| [blocking_monitoring.md](blocking_monitoring.md) | Advanced blocking session detection and resolution techniques |
| [memory.md](memory.md) | SGA, PGA, and overall memory usage monitoring |
| [Size.md](Size.md) | Database size, growth trends, and space utilization queries |

### 💾 Storage Management

| File | Description |
|------|-------------|
| [Tablespace.md](Tablespace.md) | Tablespace creation, monitoring, and maintenance scripts |
| [Tablespace - Temp and Undo.md](Tablespace%20-%20Temp%20and%20Undo.md) | UNDO and TEMP tablespace management and troubleshooting |
| [ASM.md](ASM.md) | ASM diagnostic queries, disk group management, and operational procedures |
| [log&redo.md](log%26redo.md) | Redo log monitoring, switching frequency, and optimization |
| [buffer_related.md](buffer_related.md) | Buffer cache analysis, hit ratios, and memory optimization |

### 🔐 Security & User Management

| File | Description |
|------|-------------|
| [userinfo.md](userinfo.md) | User creation, privilege management, and security auditing queries |
| [database_audit.md](database_audit.md) | Database auditing configuration, audit trail analysis, and compliance queries |

### 🔧 Database Objects & Utilities

| File | Description |
|------|-------------|
| [dblinkSequenceSynonym.md](dblinkSequenceSynonym.md) | Database links, sequences, and synonym management scripts |
| [Jobs.md](Jobs.md) | DBMS_SCHEDULER and legacy job monitoring and management |
| [index_related.md](index_related.md) | Index analysis, rebuilding, and optimization queries |

### ⚡ Performance & Tuning

| File | Description |
|------|-------------|
| [Oracle-performance.md](Oracle-performance.md) | Comprehensive performance tuning queries, AWR analysis, and SQL optimization |

### 💼 Backup & Recovery

| File | Description |
|------|-------------|
| [RMAN.md](RMAN.md) | Complete RMAN backup, restore, and recovery command reference |
| [Datapump.md](Datapump.md) | DataPump export/import procedures with examples and best practices |

### 🔄 High Availability & Disaster Recovery

| File | Description |
|------|-------------|
| [GI_RAC.md](GI_RAC.md) | Grid Infrastructure and RAC cluster administration and diagnostics |
| [Physical_Standby.md](Physical_Standby.md) | Physical standby database monitoring, lag analysis, and maintenance |
| [Logical Standby.md](Logical%20Standby.md) | Logical standby configuration, SKIP rules, and log retention management |

### 🎯 Enterprise Manager

| File | Description |
|------|-------------|
| [OEM.md](OEM.md) | SQL queries for OEM repository analysis and monitoring |
| [emcli.md](emcli.md) | Enterprise Manager Command Line Interface reference and examples |

## Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/asiandevs/OracleDBMonitoring.git
   cd OracleDBMonitoring
   ```

2. **Navigate to relevant documentation:**
   - For daily checks: Start with `DailyHealthCheck.md`
   - For specific issues: Use the table above to find the relevant guide
   - For comprehensive tasks: Refer to specialized guides (RMAN, Datapump, etc.)

3. **Execute queries:**
   - Connect to your database using SQL*Plus, SQL Developer, or your preferred client
   - Copy queries from the markdown files
   - Review and modify parameters as needed for your environment
   - Execute and analyze results

## Documentation

Each markdown file contains:
- **Purpose and overview** of the topic area
- **SQL queries** with inline comments
- **Command examples** with expected outputs
- **Best practices** and recommendations
- **Troubleshooting tips** for common scenarios
- **Version-specific notes** where applicable

### Example Usage

```sql
-- From DailyHealthCheck.md
-- Check database status
SELECT instance_name, host_name, status, database_status
FROM v$instance;

-- Check tablespace usage
SELECT tablespace_name, 
       ROUND(used_percent, 2) used_pct
FROM dba_tablespace_usage_metrics
WHERE used_percent > 85;
```

## Usage Guidelines

### Prerequisites

- Oracle Database 11g or higher
- Appropriate privileges (most queries require SELECT on V$ and DBA_ views)
- For advanced operations: SYSDBA or specific administrative privileges
- Basic understanding of Oracle architecture and SQL
- **Valid Oracle licenses for diagnostic and tuning features**

### Best Practices

1. **Test in non-production first**: Always validate scripts in dev/test environments
2. **Review before execution**: Understand what each query does before running
3. **Check privileges**: Ensure you have necessary permissions
4. **Backup critical changes**: Take backups before making structural changes
5. **Monitor performance**: Some diagnostic queries can be resource-intensive
6. **Customize as needed**: Adapt thresholds and parameters to your environment
7. **Verify licensing**: Confirm you have appropriate Oracle licenses for tuning/diagnostic features

### Safety Notes

- Scripts marked with warnings should be executed with caution
- DDL operations (CREATE, ALTER, DROP) should be reviewed carefully
- Recovery operations should follow your organization's procedures
- Always maintain valid backups before major changes
- Performance tuning queries may require Oracle Diagnostics and Tuning Pack licenses

## Contributing

Contributions are welcome! To contribute:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-monitoring-script`)
3. Add your scripts with clear documentation
4. Test thoroughly in your environment
5. Submit a pull request with a detailed description

**Contribution Guidelines:**
- Follow existing markdown formatting
- Include clear descriptions and use cases
- Add comments to complex queries
- Specify Oracle version requirements if applicable
- Include example outputs where helpful

## Version Compatibility

These scripts are tested and compatible with:
- Oracle 11g (11.2.0.4)
- Oracle 12c (12.1, 12.2)
- Oracle 18c
- Oracle 19c
- Oracle 21c and above (most scripts)

Version-specific queries are clearly marked in the documentation.

## Support & Feedback

- **Issues**: Open an issue for bugs or script problems
- **Questions**: Use discussions for general questions
- **Enhancements**: Submit pull requests for improvements

## License

Please do check any licence requirements from your side.

## Acknowledgments

This repository represents collective knowledge and best practices from Oracle DBA community experience. Special thanks to all contributors who help maintain and improve these resources.

---

**Note**: Always refer to official Oracle documentation for authoritative information and follow your organization's change management procedures before implementing any changes in production environments.
