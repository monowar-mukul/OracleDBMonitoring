-- FOR CPU and Memory 
SELECT to_char(ssn.sid, '9999') || ' - ' || nvl(ssn.username, nvl(bgp.name, 'background')) ||
nvl(lower(ssn.machine), ins.host_name) "SESSION",
to_char(prc.spid, '999999999') "PID/THREAD",
to_char((se1.value/1024)/1024, '999G999G990D00') || ' MB' " CURRENT SIZE",
to_char((se2.value/1024)/1024, '999G999G990D00') || ' MB' " MAXIMUM SIZE"
FROM v$sesstat se1, v$sesstat se2, v$session ssn, v$bgprocess bgp, v$process prc,
v$instance ins, v$statname stat1, v$statname stat2
WHERE se1.statistic# = stat1.statistic# and stat1.name = 'session pga memory'
AND se2.statistic# = stat2.statistic# and stat2.name = 'session pga memory max'
AND se1.sid = ssn.sid
AND se2.sid = ssn.sid
AND ssn.paddr = bgp.paddr (+)
AND ssn.paddr = prc.addr (+);


-- Oracle: showing memory usage by user session/process
-- memory-usage-for-each-user-session.sql

select * from v$session;

select sess.username  as username
      ,sess.sid       as session_id
      ,sess.serial#   as session_serial
      ,sess.program   as session_program
      ,sess.server    as session_mode
      ,round(stat.value/1024/1024, 2) as "current_UGA_memory (in MB)"
  from v$session    sess
      ,v$sesstat    stat
      ,v$statname   name
  where sess.sid        = stat.sid
    and stat.statistic# = name.statistic#
    and name.name       = 'session uga memory'
    and sess.username   = 'SVC_WS_PRICING' -- your user/schema name
    --and stat.value      >= 10485760   -- (All Session Usage > 10MB)
order by
    value;
    
-- memory-usage-for-each-user-in-details.sql

SELECT
    s.sid                sid
  , lpad(s.username,12)  oracle_username
  , lpad(s.osuser,9)     os_username
  , s.program            session_program
  , lpad(s.machine,8)    session_machine
  , (select round(ss.value/1024/1024, 2) from v$sesstat ss, v$statname sn
     where ss.sid = s.sid and
           sn.statistic# = ss.statistic# and
           sn.name = 'session pga memory')        session_pga_memory
  , (select round(ss.value/1024/1024, 2) from v$sesstat ss, v$statname sn
     where ss.sid = s.sid and
           sn.statistic# = ss.statistic# and
           sn.name = 'session pga memory max')    session_pga_memory_max
  , (select round(ss.value/1024/1024, 2) from v$sesstat ss, v$statname sn
     where ss.sid = s.sid and
           sn.statistic# = ss.statistic# and
           sn.name = 'session uga memory')        session_uga_memory
  , (select round(ss.value/1024/1024, 2) from v$sesstat ss, v$statname sn
     where ss.sid = s.sid and
           sn.statistic# = ss.statistic# and
           sn.name = 'session uga memory max')    session_uga_memory_max
FROM
    v$session  s
WHERE s.username = 'SVC_WS_PRICING' -- your user/schema name
ORDER BY session_pga_memory DESC
;

-- memory-usage-in-details.sql

select
    to_char(ssn.sid, '9999')                             as session_id,
    ssn.serial#                                          as session_serial,
    nvl(ssn.username, nvl(bgp.name, 'background'))
    || '::'
    || nvl(lower(ssn.machine), ins.host_name)            as process_name,
    to_char(prc.spid, '999999999')                       as pid_thread,
    to_char((se1.value / 1024) / 1024, '999g999g990d00') as current_size_mb,
    to_char((se2.value / 1024) / 1024, '999g999g990d00') as maximum_size_mb
from
    v$statname    stat1,
    v$statname    stat2,
    v$session     ssn,
    v$sesstat     se1,
    v$sesstat     se2,
    v$bgprocess   bgp,
    v$process     prc,
    v$instance    ins
where
    stat1.name         = 'session pga memory'
    and stat2.name     = 'session pga memory max'
    and se1.sid        = ssn.sid
    and se2.sid        = ssn.sid
    and se2.statistic# = stat2.statistic#
    and se1.statistic# = stat1.statistic#
    and ssn.paddr      = bgp.paddr (+)
    and ssn.paddr      = prc.addr  (+)
    and ssn.sid in (
        select sid
        from v$session
        where username = '&USR' -- your user/schema name
    )
order by
    maximum_size_mb
    ;
    
    -- memory-usage-by-category.sql
    
    select
    category                          as category, -- like SQL, PL/SQL, Other etc
    round(allocated/1024/1024, 2)     as allocated,
    round(used/1024/1024, 2)          as used,
    round(max_allocated/1024/1024, 2) as max_allocated
from
    v$process_memory
where
    pid = (
        select  pid
        from    v$process
        where
            addr = (
                select  paddr
                from    V$session
                where   sid = &SID  -- user session id
            )
        )
;
