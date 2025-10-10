##  Blocking [in addition to Locking one]

Identify blocking chains and sessions causing locks.

### Blocking Sessions

```sql
SELECT s1.username || ' (SID=' || s1.sid || ')' blocker,
       s2.username || ' (SID=' || s2.sid || ')' waiter,
       l1.id1,
       l1.id2,
       l1.lmode,
       l2.request
FROM v$lock l1,
     v$session s1,
     v$lock l2,
     v$session s2
WHERE s1.sid = l1.sid
  AND s2.sid = l2.sid
  AND l1.id1 = l2.id1
  AND l1.id2 = l2.id2
  AND l1.block = 1
  AND l2.request > 0;
```

### Active Locks

```sql
SELECT sid,
       id1,
       id2,
       block,
       type,
       lmode,
       request,
       ctime
FROM v$lock
MINUS
SELECT sid,
       id1,
       id2,
       block,
       type,
       lmode,
       request,
       ctime
FROM v$enqueue_lock;
```

### Locked Objects

```sql
SELECT *
FROM v$locked_object;
```

### Locked Object Details

```sql
SELECT owner,
       object_name,
       object_type
FROM all_objects
WHERE object_id = &object_id;
```

### Waiting Sessions Due to Locks

```sql
SELECT *
FROM dba_waiters;
```

### Blocking Sessions List

```sql
SELECT *
FROM dba_blockers;
```

### Locked Row Details

```sql
SELECT SUBSTR(f.name, 1, 40) AS file_name,
       o.owner,
       o.object_name,
       o.object_type,
       s.row_wait_block#,
       s.row_wait_row#
FROM v$session s,
     v$datafile f,
     all_objects o
WHERE s.sid = &sid
  AND s.row_wait_file# = f.file#
  AND s.row_wait_obj#(+) = o.object_id;
```

### Active Transactions by SID

```sql
SELECT username,
       t.used_ublk,
       t.used_urec
FROM v$transaction t,
     v$session s
WHERE t.addr = s.taddr;
```
## Troubleshooting Guide
### Locking and Blocking

1. Identify blocking sessions
2. Check for long-running transactions
3. Review application logic for lock contention
4. Consider row-level locking strategies
5. Monitor deadlock frequency
