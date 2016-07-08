# postgres-commands
List of postgres commands

###### Find INVALID indices

```
SELECT schemaname || '.' || relname AS table, indexrelname AS index, pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size, indisvalid AS is_index_valid
FROM pg_stat_user_indexes ui 
   JOIN pg_index i ON ui.indexrelid =i.indexrelid WHERE 
    indisvalid is false; 
```
    
    


##### List Blocker sessions recurssively

```
WITH RECURSIVE
     c(requested, CURRENT) AS
       ( VALUES
         ('AccessShareLock'::text, 'AccessExclusiveLock'::text),
         ('RowShareLock'::text, 'ExclusiveLock'::text),
         ('RowShareLock'::text, 'AccessExclusiveLock'::text),
         ('RowExclusiveLock'::text, 'ShareLock'::text),
         ('RowExclusiveLock'::text, 'ShareRowExclusiveLock'::text),
         ('RowExclusiveLock'::text, 'ExclusiveLock'::text),
         ('RowExclusiveLock'::text, 'AccessExclusiveLock'::text),
         ('ShareUpdateExclusiveLock'::text, 'ShareUpdateExclusiveLock'::text),
         ('ShareUpdateExclusiveLock'::text, 'ShareLock'::text),
         ('ShareUpdateExclusiveLock'::text, 'ShareRowExclusiveLock'::text),
         ('ShareUpdateExclusiveLock'::text, 'ExclusiveLock'::text),
         ('ShareUpdateExclusiveLock'::text, 'AccessExclusiveLock'::text),
         ('ShareLock'::text, 'RowExclusiveLock'::text),
         ('ShareLock'::text, 'ShareUpdateExclusiveLock'::text),
         ('ShareLock'::text, 'ShareRowExclusiveLock'::text),
         ('ShareLock'::text, 'ExclusiveLock'::text),
         ('ShareLock'::text, 'AccessExclusiveLock'::text),
         ('ShareRowExclusiveLock'::text, 'RowExclusiveLock'::text),
         ('ShareRowExclusiveLock'::text, 'ShareUpdateExclusiveLock'::text),
         ('ShareRowExclusiveLock'::text, 'ShareLock'::text),
         ('ShareRowExclusiveLock'::text, 'ShareRowExclusiveLock'::text),
         ('ShareRowExclusiveLock'::text, 'ExclusiveLock'::text),
         ('ShareRowExclusiveLock'::text, 'AccessExclusiveLock'::text),
         ('ExclusiveLock'::text, 'RowShareLock'::text),
         ('ExclusiveLock'::text, 'RowExclusiveLock'::text),
         ('ExclusiveLock'::text, 'ShareUpdateExclusiveLock'::text),
         ('ExclusiveLock'::text, 'ShareLock'::text),
         ('ExclusiveLock'::text, 'ShareRowExclusiveLock'::text),
         ('ExclusiveLock'::text, 'ExclusiveLock'::text),
         ('ExclusiveLock'::text, 'AccessExclusiveLock'::text),
         ('AccessExclusiveLock'::text, 'AccessShareLock'::text),
         ('AccessExclusiveLock'::text, 'RowShareLock'::text),
         ('AccessExclusiveLock'::text, 'RowExclusiveLock'::text),
         ('AccessExclusiveLock'::text, 'ShareUpdateExclusiveLock'::text),
         ('AccessExclusiveLock'::text, 'ShareLock'::text),
         ('AccessExclusiveLock'::text, 'ShareRowExclusiveLock'::text),
         ('AccessExclusiveLock'::text, 'ExclusiveLock'::text),
         ('AccessExclusiveLock'::text, 'AccessExclusiveLock'::text)
       ),
     l AS
       (
         SELECT
             (locktype,DATABASE,relation::regclass::text,page,tuple,virtualxid,transactionid,classid,objid,objsubid) AS target,
             virtualtransaction,
             pid,
             mode,
             GRANTED
           FROM pg_catalog.pg_locks
       ),
     t AS
       (
         SELECT
             blocker.target  AS blocker_target,
             blocker.pid     AS blocker_pid,
             blocker.mode    AS blocker_mode,
             blocked.target  AS target,
             blocked.pid     AS pid,
             blocked.mode    AS mode
           FROM l blocker
           JOIN l blocked
             ON ( NOT blocked.GRANTED
              AND blocker.GRANTED
              AND blocked.pid != blocker.pid
              AND blocked.target IS NOT DISTINCT FROM blocker.target)
           JOIN c ON (c.requested = blocked.mode AND c.CURRENT = blocker.mode)
       ),
     rview AS
       (
         SELECT
             blocker_target,
             blocker_pid,
             blocker_mode,
             '1'::INT        AS depth,
             target,
             pid,
             mode,
             blocker_pid::text || ',' || pid::text AS seq
           FROM t
         UNION ALL
         SELECT
             blocker.blocker_target,
             blocker.blocker_pid,
             blocker.blocker_mode,
             blocker.depth + 1,
             blocked.target,
             blocked.pid,
             blocked.mode,
             blocker.seq || ',' || blocked.pid::text
           FROM rview blocker
           JOIN t blocked
             ON (blocked.blocker_pid = blocker.pid)
           WHERE blocker.depth < 1000
       )
SELECT distinct rview.pid, rview.blocker_pid 
       , seq blocking_pid_seq 
       , (CASE WHEN session_info.waiting = 't' THEN '(Waiter) ' ELSE '(Blocker) ' END) ||session_info.usename              AS user
       , session_info.query AS user_sql
       , session_info.query_start AS sql_start_time
       , session_info.application_name
       , CASE WHEN host(session_info.client_addr) is null THEN 'Local' ELSE host(session_info.client_addr) END client_addr
       , session_info.client_hostname
 FROM rview
	JOIN pg_catalog.pg_stat_activity session_info  ON  (/*rview.blocker_pid in (19122) and */(rview.pid =session_info.pid or rview.blocker_pid =session_info.pid))
  ORDER BY seq
;
```
