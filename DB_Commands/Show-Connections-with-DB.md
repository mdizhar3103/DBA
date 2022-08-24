```sql
set lines 999 pages 9999;
col username for a10;
col machine for a20;
col module for a33;
col event for a30;
col sql_id for a15;
select distinct inst_id , sid , serial# , username , machine , module , sql_id , event ,round((last_call_et/60),2) time ,command CMD,blocking_session,final_blocking_session,ROW_WAIT_OBJ#
from gv$session
where username is not null;
```
