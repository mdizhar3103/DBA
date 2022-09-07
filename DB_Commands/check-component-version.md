```sql
col COMP_ID for a12
col COMP_NAME for a20
col SCHEMA for a30
col STATUS for a20
set lines 190 pages 1000
select COMP_ID,COMP_NAME,VERSION,VERSION_FULL,STATUS,SCHEMA from dba_registry;
```
