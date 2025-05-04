#  Table of Contents
- [JOB STATUS](#job-status)
- [Failed scheduled tasks](#failed-scheduled-tasks)
- [RUN THE JOB](#run-the-job)
- [Create The JOB](#create-the-job)
- [What Jobs are Actually Running](#what-jobs-are-actually-running)
- [What Sessions are Running the Jobs](#what-sessions-are-running-the-jobs)
- [Change job schedule](#change-job-schedule)
- [Check long running queries](#check-long-running-queries)
- [Check long running process for particular session and sid](#check-long-running-process-for-particular-session-and-sid)
- [Check the time remaining](#check-the-time-remaining)
- [Autotask Scheduled Jobs (Space advisor, Optimizer stats, SQL tuning)](#autotask-scheduled-jobs-space-advisor-optimizer-stats-sql-tuning)
- [Enable auto stats collection job](#enable-auto-stats-collection-job)
- [Disable auto stats collection job](#disable-auto-stats-collection-job)
- [Disable all the automated tasks](#disable-all-the-automated-tasks)
- [Broken Jobs](#broken-jobs)
- [Long Running Jobs](#long-running-jobs)

---

# SQL Scripts and Commands

## JOB STATUS
```sql
ALTER SESSION SET nls_date_format = 'DD-MM-YYYY HH24:MI:SS';

SELECT job, what, last_date, last_sec, this_date, this_sec, next_date, next_sec, failures, broken 
FROM dba_jobs;
```

---

## Failed scheduled tasks
```sql
COL log_id FORMAT 9999 HEADING 'Log#'
COL job_name FORMAT a15
COL status FORMAT a10

SELECT log_id, job_name, status, actual_start_date, run_duration, error#
FROM dba_scheduler_job_run_details
WHERE status = 'FAILED'
ORDER BY log_id DESC;
```

---

## RUN THE JOB
```sql
EXEC dbms_ijob.run(7);
```

---

## Create The JOB
```sql
DECLARE
  job_id NUMBER;
BEGIN
  DBMS_JOB.SUBMIT(
    job => job_id,
    what => 'schema_name.proc_name;',
    next_date => SYSDATE,
    interval => 'SYSDATE+1/24'
  );
  COMMIT;
END;
/
```

---

## What Jobs are Actually Running
```sql
SET LINESIZE 250

SELECT * 
FROM dba_jobs_running;
```

---

## What Sessions are Running the Jobs
```sql
SET LINESIZE 250

SELECT * 
FROM v$session
WHERE program LIKE '%J%';
```

---

## Change job schedule
```sql
EXEC dbms_job.change(
  42,
  'SUMM_REFRESH.PROC_REFRESH;',
  TRUNC(SYSDATE+1) + 5.5/24,
  'TRUNC(SYSDATE+1) + 5.5/24'
);
```

---

## Check long running queries
```sql
COL sql_text FORMAT a100

SELECT a.sql_text, b.sql_id, b.sql_child_number, b.event, b.status
FROM v$sqlarea a, v$session b
WHERE a.address = b.sql_address
  AND a.hash_value = b.sql_hash_value
  AND b.status = 'ACTIVE';
```

---

## Check long running process for particular session and sid
```sql
SELECT *
FROM (
  SELECT opname, target, sofar, totalwork, units, elapsed_seconds, message
  FROM v$session_longops
  WHERE sofar <> totalwork
)
WHERE ROWNUM < 20;
```

---

## Check the time remaining
```sql
SELECT 
  opname, 
  target,
  ROUND((sofar/totalwork),4) * 100 AS percentage_complete,
  sofar,
  totalwork,
  start_time,
  last_update_time,
  elapsed_seconds,
  time_remaining
FROM v$session_longops
WHERE totalwork > 0
  AND sofar <> totalwork;
```

---

## Autotask Scheduled Jobs (Space advisor, Optimizer stats, SQL tuning)
```sql
SELECT client_name, window_name, jobs_created, jobs_started, jobs_completed 
FROM dba_autotask_client_history
WHERE client_name LIKE '%stats%';
```

---

## Enable auto stats collection job
```sql
BEGIN
  DBMS_AUTO_TASK_ADMIN.ENABLE(
    client_name => 'auto optimizer stats collection',
    operation => NULL,
    window_name => NULL
  );
END;
/
```

---

## Disable auto stats collection job
```sql
BEGIN
  DBMS_AUTO_TASK_ADMIN.DISABLE(
    client_name => 'auto optimizer stats collection',
    operation => NULL,
    window_name => NULL
  );
END;
/
```

---

## Disable all the automated tasks
```sql
EXEC dbms_auto_task_admin.disable;
```

---

## Broken Jobs
```sql
SELECT 
  'EXEC dbms_job.broken(job =>' || job || ', next_date =>' || interval || ', broken => FALSE)' AS fix_script
FROM dba_jobs
WHERE schema_user = '&user'
  AND broken = 'Y';
```

---

## Long Running Jobs
```sql
SET PAUSE ON
SET PAUSE 'Press Return to Continue'
SET PAGESIZE 60

SELECT * 
FROM dba_jobs_running;
```

---
