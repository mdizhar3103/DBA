### Managing Tablespaces in Multitenant Architecture DB
In multitenant architecture, it is stated that pluggable databases share the same resoruces such as memory and system processes of the container database instance.

**Views Hierarchy**
```
        +-------------------------------------------------------------------------------------------+
        | CDB_* Additional column CON_ID                                                            |
        |       At ROOT, can access objects across all PDBs in the CDB instance                     |
        |       Depends on assign CONTAINER_DATA privileges of user.                                |
        |       At PDB, CDB_ = DBA_                                                                 |
        |       +---------------------------------------------------------------------------+       |
        |       | DBA_* All the container/PDB objects                                       |       |
        |       |       +-----------------------------------------------------------+       |       |
        |       |       | ALL_* Objects that the current user has privileges on     |       |       |
        |       |       |       +-------------------------------------------+       |       |       |
        |       |       |       | USER_* Objects owned by the current user  |       |       |       |
        |       |       |       +-------------------------------------------+       |       |       |
        |       |       +-----------------------------------------------------------+       |       |
        |       +---------------------------------------------------------------------------+       |
        +-------------------------------------------------------------------------------------------+

```

**What is shared?**
UNDO tablespace are related to instance and not individual pluggable databases and it seems that temporary tablespaces might be shared resoruces or they may be defined for each individual pluggable database.

UNDO tablespace are integral part of recovery as it is related to control files, online redo logs, archive logs.

```SQL

SQL> select tablespace_name from dba_tablespaces;
SQL> select tablespace_name, con_id from cdb_tablespaces;
    -- con_id 1 of undo tbs represent it belong to cdb

-- switching to any pdb
SQL> show con_name;
SQL> select tablespace_name from dba_tablespaces;
    -- undo is not part of the pdb

-- looking at instance views to see information about undo tbs using gv$ views or v$ views
SQL> select name, values from gv$parameter where name like '%undo%';  2 3

-- Other views you can use would be v$tablespace view and v$datafile view and can do a join between them
SQL> select t.name, d.name, t.con_id from v$tablespace t, v$datafile d where t.ts#=d.ts# and t.con_id=d.con_id order by t.con_id, t.name;  2

SQL> select tablespace_name, con_id from cdb_tablespaces;
SQL> select name, values from gv$parameter where name like '%undo%';  2 3

```

**Temporary Tablespace**
- By default all containers point to a common temporary tablespace in the root container.
- All pdb and root uses temporary tablespace that act as the default TEMP tablespace.
- A pdb can be setup to use its own temporary tablespace for its local users.
- **Note:** Once you switched at pdb level to local tablespace, you can't switch the temporary tablespace back to the common temp tablespace. 
- Also, Once the temporary tablespace has been setup, you can't assign the user to the container database default temporary tablespace

**Setting PDB limits**
```SQL

SQL> alter pluggable database storage (MAXSIZE 5G);         -- Total size of pdb not individual datafile

-- limiting the temporary tablespace size
SQL> alter pluggable database storage (max_shared_temp_size 2G);

-- combining the clause
SQL> alter pluggable database storage (maxsize 6G max_shared_temp_size 3G);

```