## Most active session in last one hour can be found using active session history
```
SELECT sql_id,COUNT(*),ROUND(COUNT(*)/SUM(COUNT(*)) OVER(), 2) PCTLOAD
FROM gv$active_session_history
WHERE sample_time > SYSDATE - 1/24
AND session_type = 'BACKGROUND'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
### Get the wait event for the above session
```
SELECT sample_time, event, wait_time
FROM gv$active_session_history
WHERE session_id = &1
AND session_serial# = &2
```
## Most I/O intensive sql in last 1 hour
```
SELECT sql_id, COUNT(*)
FROM gv$active_session_history ash, gv$event_name evt
WHERE ash.sample_time > SYSDATE - 1/24
AND ash.session_state = 'WAITING'
AND ash.event_id = evt.event_id
AND evt.wait_class = 'User I/O'
GROUP BY sql_id
ORDER BY COUNT(*) DESC;
```
## Top sqls spent more on cpu/wait/io
```
select
ash.SQL_ID ,
sum(decode(a.session_state,'ON CPU',1,0)) "CPU",
sum(decode(a.session_state,'WAITING',1,0)) -
sum(decode(a.session_state,'WAITING', decode(en.wait_class, 'User I/O',1,0),0)) "WAIT" ,
sum(decode(a.session_state,'WAITING', decode(en.wait_class, 'User I/O',1,0),0)) "IO" ,
sum(decode(a.session_state,'ON CPU',1,1)) "TOTAL"
from v$active_session_history a,v$event_name en
where SQL_ID is not NULL and en.event#=ash.event#
```
##  A particular session sql analysis
```
SELECT C.SQL_TEXT,
B.NAME,
COUNT(*),
SUM(TIME_WAITED)
FROM v$ACTIVE_SESSION_HISTORY A,
v$EVENT_NAME B,
v$SQLAREA C
WHERE A.SAMPLE_TIME BETWEEN '&starttime' AND
'&endtime' AND
A.EVENT# = B.EVENT# AND
A.SESSION_ID= &sid AND
A.SQL_ID = C.SQL_ID
GROUP BY C.SQL_TEXT, B.NAME
```
## Top session on CPU in last 15 minute
```
SELECT * FROM
(
SELECT s.username, s.module, s.sid, s.serial#, s.sql_id,count(*)
FROM v$active_session_history h, v$session s
WHERE h.session_id = s.sid
AND h.session_serial# = s.serial#
AND session_state= 'ON CPU' AND
sample_time > sysdate - interval '15' minute
GROUP BY s.username, s.module, s.sid, s.serial#,s.sql_id
ORDER BY count(*) desc
)
where rownum <= 10;
```
## Find queries executed in the last 30 days.
```
SELECT
   h.sample_time,
   u.username,
   h.program,
   h.module,
   s.sql_text
FROM
   DBA_HIST_ACTIVE_SESS_HISTORY h,
   DBA_USERS u,
   DBA_HIST_SQLTEXT s
WHERE  sample_time between to_date('01/10/2024 19:45:00','DD/MM/YYYY HH24:MI:SS') 
and to_date('01/10/2024 19:50:00','DD/MM/YYYY HH24:MI:SS')
   AND h.user_id=u.user_id
AND h.sql_id = s.sql_iD
ORDER BY h.sample_time;
```
```
SELECT
   h.sample_time,
   u.username,
   h.program,
   h.module,
   s.sql_text
FROM
   DBA_HIST_ACTIVE_SESS_HISTORY h,
   DBA_USERS u,
   DBA_HIST_SQLTEXT s
WHERE  WHERE sample_time >= SYSDATE - 30
AND h.user_id=u.user_id
AND h.sql_id = s.sql_iD
ORDER BY h.sample_time;
```
