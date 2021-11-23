### More on Oracle Database

- **CDB**: Container Database, is the primary database that contains multiple plugged-in databases.
    - Oracle System metadata in CDB$ROOT
- **PDB**: Pluggable Database, a set of schemas, objects, and non-schema objects that can be plugged and unplugged from a container database.

***Demo: Physical Architecture***
```bash

>>> ps -ef | grep pmon

# number of process for upgrade database
>>> ps -ef | grep UPGR | wc -l

# number of process for container database
>>> ps -ef | grep CDB1 | wc -l          

# to list all processes related to oracle database
>>> ps -ef | grep ora_
```

Exploring From SQL query:
```SQL

>>> sqlplus /nolog

SQL> connect system/oracle

SQL> @containers;

SQL> show con_name;
SQL> show pdbs;
SQL> desc v$containers;
SQL> desc v$pdbs;           -- similar to v$containers

SQL> select con_id, name, open_mode from v$pdbs;

SQL> desc v$system_parameter;
/*
Look from ISPDB_MODIFIABLE
*/

SQL> @parameters.sql        -- control files gives instance information

SQL> select name, display_value, ispdb_modifiable, con_id from v$system_parameter where upper(name) in ('CONTROL_FILES', 'CORE_DUMP_DEST', 'SESSIONS', 'UNDO_TABLESPACE', 'UNDO_RETENTION','SORT_AREA_SIZE') order by name;

SQL> desc v$tablespace;
SQL> select ts#, name, con_id from v$tablespace;

SQL> @filename.sql
SQL> select con_id, file_name, tablespace_name from cdb_data_files;

SQL> select con_id, file_name, tablespace_name from dba_data_files;         -- gives error because dba_data_files doesn't have con_id

SQL> select file_name, tablespace_name from cdb_data_files;

SQL> @detailfile.sql
SQL> select p.PDB_ID, p.PDB_NAME, d.FILE_ID, d.TABLESPACE_NAME, d.FILE_NAME from DBA_PDBS p, CDB_DATA_FILES d WHERE p.PDB_ID = d.CON_ID;

```

These parameters are stored in __SPFILE__. SPFILE only relates to container instance. It is the information needed for the container instant to set its values. 

The other values for the pluggable database are kept within their own system tables.

```SQL
SQL> show parameter spfile;

-- copy the spfile path and open in bash shell to see details info
>>> cat spfile_path
```

**Creating CDBs and PDBs:** *See create_cdb_pdb.md*

**Local and Common Users Management:** *See user_management.md*
