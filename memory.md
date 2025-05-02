````markdown
# Memory Usage Monitoring Wiki

This wiki provides various SQL queries for monitoring memory usage in the Oracle database system, specifically for user sessions, process memory, and statistics related to CPU and memory.

## Table of Contents
- [CPU and Memory Usage by Session](#cpu-and-memory-usage-by-session)
- [Memory Usage for Each User Session](#memory-usage-for-each-user-session)
- [Memory Usage for Each User - Detailed View](#memory-usage-for-each-user---detailed-view)
- [Memory Usage - Detailed Breakdown](#memory-usage---detailed-breakdown)
- [Memory Usage by Category](#memory-usage-by-category)

---

## CPU and Memory Usage by Session

This query retrieves the current and maximum session memory usage for each user session, along with CPU process ID (PID) and thread information.

```sql
SELECT 
    to_char(ssn.sid, '9999') || ' - ' || nvl(ssn.username, nvl(bgp.name, 'background')) ||
    nvl(lower(ssn.machine), ins.host_name) "SESSION",
    to_char(prc.spid, '999999999') "PID/THREAD",
    to_char((se1.value/1024)/1024, '999G999G990D00') || ' MB' "CURRENT SIZE",
    to_char((se2.value/1024)/1024, '999G999G990D00') || ' MB' "MAXIMUM SIZE"
FROM 
    v$sesstat se1, v$sesstat se2, v$session ssn, v$bgprocess bgp, v$process prc,
    v$instance ins, v$statname stat1, v$statname stat2
WHERE 
    se1.statistic# = stat1.statistic# AND stat1.name = 'session pga memory'
    AND se2.statistic# = stat2.statistic# AND stat2.name = 'session pga memory max'
    AND se1.sid = ssn.sid
    AND se2.sid = ssn.sid
    AND ssn.paddr = bgp.paddr (+)
    AND ssn.paddr = prc.addr (+);
````

---

## Memory Usage for Each User Session

This query displays detailed memory usage by each user session, including the current session memory in MB for a specific user.

```sql
SELECT 
    sess.username AS username,
    sess.sid AS session_id,
    sess.serial# AS session_serial,
    sess.program AS session_program,
    sess.server AS session_mode,
    round(stat.value/1024/1024, 2) AS "current_UGA_memory (in MB)"
FROM 
    v$session sess,
    v$sesstat stat,
    v$statname name
WHERE 
    sess.sid = stat.sid
    AND stat.statistic# = name.statistic#
    AND name.name = 'session uga memory'
    AND sess.username = 'SVC_WS_PRICING'  -- Replace with your user/schema name
ORDER BY 
    value;
```

---

## Memory Usage for Each User - Detailed View

This query provides a detailed view of the memory usage for each session, including PGA and UGA memory for each user session.

```sql
SELECT
    s.sid SID,
    lpad(s.username, 12) oracle_username,
    lpad(s.osuser, 9) os_username,
    s.program session_program,
    lpad(s.machine, 8) session_machine,
    (SELECT round(ss.value/1024/1024, 2) FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session pga memory') session_pga_memory,
    (SELECT round(ss.value/1024/1024, 2) FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session pga memory max') session_pga_memory_max,
    (SELECT round(ss.value/1024/1024, 2) FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session uga memory') session_uga_memory,
    (SELECT round(ss.value/1024/1024, 2) FROM v$sesstat ss, v$statname sn
     WHERE ss.sid = s.sid 
     AND sn.statistic# = ss.statistic# 
     AND sn.name = 'session uga memory max') session_uga_memory_max
FROM 
    v$session s
WHERE 
    s.username = 'SVC_WS_PRICING'  -- Replace with your user/schema name
ORDER BY 
    session_pga_memory DESC;
```

---

## Memory Usage - Detailed Breakdown

This query provides detailed information on session memory, showing current and maximum memory usage (PGA and UGA) for each user session.

```sql
SELECT
    to_char(ssn.sid, '9999') AS session_id,
    ssn.serial# AS session_serial,
    nvl(ssn.username, nvl(bgp.name, 'background')) || '::' || 
    nvl(lower(ssn.machine), ins.host_name) AS process_name,
    to_char(prc.spid, '999999999') AS pid_thread,
    to_char((se1.value / 1024) / 1024, '999g999g990d00') AS current_size_mb,
    to_char((se2.value / 1024) / 1024, '999g999g990d00') AS maximum_size_mb
FROM
    v$statname stat1,
    v$statname stat2,
    v$session ssn,
    v$sesstat se1,
    v$sesstat se2,
    v$bgprocess bgp,
    v$process prc,
    v$instance ins
WHERE 
    stat1.name = 'session pga memory'
    AND stat2.name = 'session pga memory max'
    AND se1.sid = ssn.sid
    AND se2.sid = ssn.sid
    AND se2.statistic# = stat2.statistic#
    AND se1.statistic# = stat1.statistic#
    AND ssn.paddr = bgp.paddr (+)
    AND ssn.paddr = prc.addr (+)
    AND ssn.sid IN (
        SELECT sid
        FROM v$session
        WHERE username = '&USR'  -- Replace with your user/schema name
    )
ORDER BY 
    maximum_size_mb;
```

---

## Memory Usage by Category

This query displays memory usage statistics for each category such as SQL, PL/SQL, etc., based on process memory.

```sql
SELECT
    category AS category,  -- Like SQL, PL/SQL, Other, etc.
    round(allocated/1024/1024, 2) AS allocated,
    round(used/1024/1024, 2) AS used,
    round(max_allocated/1024/1024, 2) AS max_allocated
FROM 
    v$process_memory
WHERE 
    pid = (
        SELECT pid
        FROM v$process
        WHERE addr = (
            SELECT paddr
            FROM V$session
            WHERE sid = &SID  -- Replace with user session ID
        )
    );
```

---
