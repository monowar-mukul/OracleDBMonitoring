📚 Table of Contents (click to expand)</summary>

- [1. Disk Group Space Usage](#1--disk-group-space-usage)
- [2.  Disk Group Layout and IO Statistics](#2--disk-group-layout-and-io-statistics)
- [3.  ASM File Details Per Disk Group](#3--asm-file-details-per-disk-group)
- [4.  Disk Mapping and Host Visibility](#4--disk-mapping-and-host-visibility)
- [5.  ASM Performance (IOPS / Throughput)](#5--asm-performance-iops--throughput)
- [6.  Capacity Summary by Redundancy Type](#6--capacity-summary-by-redundancy-type)
- [ Quick Reference ASCII Table](#-quick-reference-ascii-table)

---

# Oracle ASM Diagnostic and Operational Scripts

## 1.  Disk Group Space Usage

```sql
SELECT
    name AS disk_group,
    total_mb,
    free_mb,
    ROUND((total_mb - free_mb) / total_mb * 100, 2) AS pct_used,
    ROUND(free_mb / total_mb * 100, 2) AS pct_free,
    usable_file_mb,
    offline_disks,
    type AS redundancy,
    state
FROM
    v$asm_diskgroup;
````

---

## 2.  Disk Group Layout and IO Statistics

```sql
SELECT
    d.group_number,
    g.name AS disk_group_name,
    d.disk_number,
    d.name AS disk_name,
    d.path,
    d.mount_status,
    d.state,
    d.total_mb,
    d.free_mb,
    d.reads,
    d.writes,
    d.read_errs,
    d.write_errs
FROM
    v$asm_disk d
JOIN
    v$asm_diskgroup g ON d.group_number = g.group_number
ORDER BY
    g.name, d.disk_number;
```

---

## 3.  ASM File Details Per Disk Group

```sql
SELECT
    f.group_number,
    g.name AS disk_group_name,
    f.file_number,
    f.file_type,
    f.bytes / 1024 / 1024 AS size_mb,
    f.blocks,
    f.block_size,
    f.space,
    f.redundancy,
    f.database,
    f.name
FROM
    v$asm_file f
JOIN
    v$asm_diskgroup g ON f.group_number = g.group_number
ORDER BY
    g.name, f.file_type;
```

---

## 4.  Disk Mapping and Host Visibility

```sql
SELECT
    d.name AS disk_name,
    d.path,
    d.mount_status,
    d.label,
    d.failgroup,
    d.library,
    d.os_mb,
    d.total_mb,
    d.free_mb,
    d.state
FROM
    v$asm_disk d
ORDER BY
    d.path;
```

---

## 5.  ASM Performance (IOPS / Throughput)

```sql
SELECT
    name AS disk_name,
    reads,
    writes,
    read_errs,
    write_errs,
    read_time,
    write_time,
    bytes_read,
    bytes_written
FROM
    v$asm_disk
WHERE
    mount_status = 'CACHED'
ORDER BY
    bytes_written DESC;
```

---

## 6.  Capacity Summary by Redundancy Type

```sql
SELECT
    type AS redundancy,
    COUNT(*) AS diskgroup_count,
    SUM(total_mb) AS total_mb,
    SUM(free_mb) AS free_mb,
    ROUND(SUM((total_mb - free_mb)) / SUM(total_mb) * 100, 2) AS pct_used
FROM
    v$asm_diskgroup
GROUP BY
    type;
```

---

##  Quick Reference ASCII Table

```
+----+-------------------------------------------+-------------------------------+
| ID | Purpose                                   | SQL View                      |
+----+-------------------------------------------+-------------------------------+
| 1  | Disk Group Space Usage                    | v$asm_diskgroup               |
| 2  | Disk IO and Layout per Disk Group         | v$asm_disk + v$asm_diskgroup |
| 3  | ASM Files and Sizes                       | v$asm_file + v$asm_diskgroup |
| 4  | Disk Path and Mount Mapping               | v$asm_disk                    |
| 5  | Disk IOPS and Throughput Performance      | v$asm_disk                    |
| 6  | Capacity Summary by Redundancy Type       | v$asm_diskgroup               |
+----+-------------------------------------------+-------------------------------+
```

---

