#  Oracle Lock and Session Monitoring SQLs

## Table of Contents
- [List All Locked Objects Across the RAC](#-list-all-locked-objects-across-the-rac)
- [Locking Information in Last 1 Minute](#-locking-information-in-last-1-minute)
- [Locked Objects and Lock Details](#-locked-objects-and-lock-details)
- [Locks by Sessions (Various Lock Types)](#-locks-by-sessions-various-lock-types)
- [Session Lock Details (Formatted Output)](#-session-lock-details-formatted-output)
- [Find Which Sessions Are Waiting for Each Other](#-find-which-sessions-are-waiting-for-each-other)
- [Get Holder Session ID](#-get-holder-session-id)
- [Sessions Active Longer or Running SQL](#-sessions-active-longer-or-running-sql)
- [Kill a Problematic Session](#-kill-a-problematic-session)
- [Identify Waiting vs Holding Users (Deadlocks)](#-identify-waiting-vs-holding-users-deadlocks)
- [What SQL Is Being Run by a Session](#what-sql-is-being-run-by-a-session)
- [Solution](#solution)

---

##  List All Locked Objects Across the RAC
```sql
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60
SET LINESIZE 300

COLUMN owner FORMAT A20
COLUMN username FORMAT A20
COLUMN object_owner FORMAT A20
COLUMN object_name FORMAT A30
COLUMN locked_mode FORMAT A15

SELECT b.inst_id,
       b.session_id AS sid,
       NVL(b.oracle_username, '(oracle)') AS username,
       a.owner AS object_owner,
       a.object_name,
       DECODE(b.locked_mode, 0, 'None',
                              1, 'Null (NULL)',
                              2, 'Row-S (SS)',
                              3, 'Row-X (SX)',
                              4, 'Share (S)',
                              5, 'S/Row-X (SSX)',
                              6, 'Exclusive (X)',
                              b.locked_mode) locked_mode,
       b.os_user_name
FROM   dba_objects a,
       gv$locked_object b
WHERE  a.object_id = b.object_id
ORDER BY 1, 2, 3, 4
/
```

---

##  Locking Information in Last 1 Minute
```sql
COL event FOR A22
COL block_type FOR A18
COL objn FOR A18
COL otype FOR A10
COL fn FOR 99
COL sid FOR 9999
COL bsid FOR 9999
COL lm FOR 99
COL p3 FOR 99999
COL blockn FOR 99999

SELECT
    TO_CHAR(sample_time,'HH:MI') st,
    SUBSTR(event,0,20) event,
    a.session_id sid,
    MOD(a.p1,16) lm,
    a.p2,
    a.p3,
    NVL(o.object_name,ash.current_obj#) objn,
    SUBSTR(o.object_type,0,10) otype,
    CURRENT_FILE# fn,
    CURRENT_BLOCK# blockn,
    a.SQL_ID,
    BLOCKING_SESSION bsid
FROM 
    v$active_session_history a,
    all_objects o
WHERE 
    event LIKE 'enq: TX%'
    AND o.object_id (+)= a.CURRENT_OBJ#
    AND sample_time > SYSDATE - 40/(60*24)
ORDER BY 
    sample_time
/
```

---

##  Locked Objects and Lock Details
```sql
PROMPT +----------------------------------------------------------------------------+
PROMPT | LOCKED OBJECTS                                                             |
PROMPT +----------------------------------------------------------------------------+

SELECT
    i.instance_name           instance_name,
    RPAD(l.session_id,7)       sid,
    RPAD(s.serial#,7)          serial_number,
    s.status                  session_status,
    l.oracle_username         oracle_user,
    l.os_user_name             os_username,
    o.owner                   object_owner,
    o.object_name             object_name,
    o.object_type             object_type
FROM
    dba_objects o,
    gv$session s,
    gv$locked_object l,
    gv$instance i
WHERE
    i.inst_id = l.inst_id
    AND l.inst_id = s.inst_id
    AND l.session_id = s.sid
    AND o.object_id = l.object_id
ORDER BY
    l.session_id
/
```

---

##  Locks by Sessions (Various Lock Types)
```sql
SELECT 
    s.sid SID,
    s.serial# Serial#,
    l.type type,
    ' ' object_name,
    lmode held,
    request request
FROM 
    v$lock l, 
    v$session s, 
    v$process p
WHERE 
    s.sid = l.sid 
    AND s.username <> ' ' 
    AND s.paddr = p.addr 
    AND l.type <> 'TM' 
    AND (l.type <> 'TX' OR l.type = 'TX' AND l.lmode <> 6)
UNION
SELECT 
    s.sid SID,
    s.serial# Serial#,
    l.type type,
    object_name object_name,
    lmode held,
    request request
FROM 
    v$lock l,
    v$session s,
    v$process p,
    sys.dba_objects o
WHERE 
    s.sid = l.sid 
    AND o.object_id = l.id1 
    AND l.type = 'TM' 
    AND s.username <> ' ' 
    AND s.paddr = p.addr
UNION
SELECT 
    s.sid SID,
    s.serial# Serial#,
    l.type type,
    '(Rollback='||RTRIM(r.name)||')' object_name,
    lmode held,
    request request
FROM 
    v$lock l, 
    v$session s, 
    v$process p, 
    v$rollname r
WHERE 
    s.sid = l.sid 
    AND l.type = 'TX' 
    AND l.lmode = 6 
    AND TRUNC(l.id1/65536) = r.usn 
    AND s.username <> ' ' 
    AND s.paddr = p.addr
ORDER BY 5, 6
/
```

---

##  Session Lock Details (Formatted Output)
```sql
SET PAUSE OFF
SET PAGESIZE 9999
SET LINESIZE 150
SET FEEDBACK OFF
SET ECHO OFF

COLUMN "User" FORMAT A12
COLUMN "Session" FORMAT A14
COLUMN "Client Host" FORMAT A15
COLUMN "Client PID" FORMAT A6
COLUMN "Server PID" FORMAT A6
COLUMN "Status" FORMAT A8
COLUMN "Lock Type" FORMAT A4
COLUMN "Mode Held" FORMAT A14
COLUMN "Mode Requested" FORMAT A14
COLUMN "Lock ID" FORMAT A14
COLUMN "Object" FORMAT A30

SELECT /*+ ORDERED */
    s.USERNAME "User",
    '''' || TO_CHAR(s.SID) || ',' || TO_CHAR(s.serial#) || '''' "Session",
    s.MACHINE "Client Host",
    s.PROCESS "Client PID",
    TO_CHAR(p.spid) "Server PID",
    s.STATUS "Status",
    l.type "Lock Type",
    DECODE(l.lmode,
            0, 'None',
            1, 'Null',
            2, 'Row-S (SS)',
            3, 'Row-X (SX)',
            4, 'Share',
            5, 'S/Row-X (SSX)',
            6, 'Exclusive',
            TO_CHAR(l.lmode)) "Mode Held",
    DECODE(l.request,
            0, 'None',
            1, 'Null',
            2, 'Row-S (SS)',
            3, 'Row-X (SX)',
            4, 'Share',
            5, 'S/Row-X (SSX)',
            6, 'Exclusive',
            TO_CHAR(l.request)) "Mode Requested",
    TO_CHAR(l.id1) || '.' || TO_CHAR(l.id2) "Lock ID",
    DECODE(l.type,'TX',r.name,o.name) "Object"
FROM 
    v$lock l,
    v$session s,
    v$process p,
    sys.obj$ o,
    v$rollname r
WHERE 
    p.addr = s.paddr
    AND l.sid = s.sid
    AND o.obj#(+) = DECODE(l.id2,0,l.id1,l.id2)
    AND r.usn(+) = DECODE(l.type,'TX',TRUNC(l.id1/65536),9999999999)
    AND s.type != 'BACKGROUND'
ORDER BY 
    DECODE(l.id2,0,l.id1,l.id2), 8, 9
/
```

---

##  Find Which Sessions Are Waiting for Each Other
```sql
SELECT 
    DECODE(request, 0, 'Holder: ', 'Waiter: ') || sid sess,
    id1, id2, lmode, request, type
FROM 
    v$lock
WHERE 
    (id1, id2, type) IN (SELECT id1, id2, type FROM v$lock WHERE request > 0)
ORDER BY 
    id1, request
/
```

---

##  Get Holder Session ID
```sql
SELECT OSUSER, USERNAME, sid, serial#
FROM v$session
WHERE USERNAME = '&usr'
/
```

```sql
SELECT s.sid, s.serial#, s.username, s.osuser, p.spid, s.machine, p.terminal, s.program
FROM v$session s, v$process p
WHERE s.paddr = p.addr
AND s.sid = &SID
/
```

---

##  Sessions Active Longer or Running SQL
```sql
SELECT USERNAME, TERMINAL, PROGRAM, SQL_ID, LOGON_TIME,
       ROUND((SYSDATE-LOGON_TIME)*(24*60),1) AS MINUTES_LOGGED_ON,
       ROUND(LAST_CALL_ET/60,1) AS MINUTES_FOR_CURRENT_SQL
FROM v$session
WHERE STATUS = 'ACTIVE'
AND USERNAME IN ('MSGIS','MSHIST')
ORDER BY MINUTES_LOGGED_ON DESC
/
```

---

##  Kill a Problematic Session
```sql
SELECT sid, serial# 
FROM v$session 
WHERE sid = &SID
/
```

```sql
ALTER SYSTEM KILL SESSION '&SID,&SERIAL' IMMEDIATE;
```

If hanging → find process:
```sql
SELECT s.sid, s.serial#, s.username, s.osuser, p.spid, s.machine, p.terminal, s.program
FROM v$session s, v$process p
WHERE s.paddr = p.addr
AND s.sid = &SID
/
```

Then on the server:
```bash
kill -9 <PID>
```

---

##  Identify Waiting vs Holding Users (Deadlocks)
```sql
SELECT SUBSTR(s1.username,1,12) "WAITING User",
       SUBSTR(s1.osuser,1,8) "OS User",
       SUBSTR(TO_CHAR(w.session_id),1,5) "Sid",
       p1.spid "PID",
       SUBSTR(s2.username,1,12) "HOLDING User",
       SUBSTR(s2.osuser,1,8) "OS User",
       SUBSTR(TO_CHAR(h.session_id),1,5) "Sid",
       p2.spid "PID"
FROM 
    v$process p1, v$process p2,
    v$session s1, v$session s2,
    dba_locks w, dba_locks h
WHERE
    (((h.mode_held != 'None') AND (h.mode_held != 'Null') 
      AND ((h.mode_requested = 'None') OR (h.mode_requested = 'Null')))
    AND ((w.mode_held = 'None') OR (w.mode_held = 'Null') 
      AND ((w.mode_requested != 'None') AND (w.mode_requested != 'Null'))))
    AND w.lock_type = h.lock_type
    AND w.lock_id1 = h.lock_id1
    AND w.lock_id2 = h.lock_id2
    AND w.session_id != h.session_id
    AND w.session_id = s1.sid
    AND h.session_id = s2.sid
    AND s1.paddr = p1.addr
    AND s2.paddr = p2.addr
/
```

---

## What SQL Is Being Run by a Session
```sql
SELECT /*+ ORDERED */
    s.sid, s.username, s.osuser,
    NVL(s.machine, '?') machine,
    NVL(s.program, '?') program,
    s.process F_Ground, p.spid B_Ground,
    x.sql_text
FROM 
    v$session s,
    v$process p,
    v$sqlarea x
WHERE 
    s.osuser LIKE LOWER(NVL('&OS_User','%'))
    AND s.username LIKE UPPER(NVL('&Oracle_User','%'))
    AND s.sid LIKE NVL('&SID','%')
    AND s.paddr = p.addr
    AND s.type != 'BACKGROUND'
    AND s.sql_address = x.address
    AND s.sql_hash_value = x.hash_value
ORDER BY s.sid
/
```

---

## Solution
```
We have 3 options to fix this error:
1. Kill the DB session and get the tables unlocked
2. Kill the application which holds this particular session (SQL connection)
3. Ideally, debug/fix the issue in the application itself.

Option 1:
    Kill session from client connection
    Example:
        alter system kill session '953,40807';

Option 2:
    Find MACHINE from V$SESSION and locate processes:
    SELECT O.OBJECT_NAME, S.SID, S.SERIAL#, P.SPID, S.PROGRAM, S.USERNAME, S.MACHINE, S.PORT, S.LOGON_TIME, SQ.SQL_FULLTEXT
    FROM V$LOCKED_OBJECT L, DBA_OBJECTS O, V$SESSION S, V$PROCESS P, V$SQL SQ
    WHERE L.OBJECT_ID = O.OBJECT_ID
    AND L.SESSION_ID = S.SID
    AND S.PADDR = P.ADDR
    AND S.SQL_ADDRESS = SQ.ADDRESS;

    Then:
    ```bash
    ps aux | grep java
    kill -15 <PID>
    ```

Option 3:
    Find V$SESSION.PORT and then:
    ```bash
    netstat -ap | grep <PORT>
    ```

    Locate and debug/kill the correct process.
```

---
