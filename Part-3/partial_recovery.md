### Partial Recovery Using Flashback Database

**Flashback Database for CDB**
| Recovery | Flashback |
|----------|-----------|
| Files Restore | No Restore |
| Roll Forward Changes | Update block from UNDO |
| Multiple block Changes | This is Quicker because we aren't restoring a datafile |
| involves restore the datafiles to previous point in time and roll forward the changes until you get to the proper SCN | |


***Flashback Technology Limitations***
- __Undamaged Datafiles:__ Should not have any damaged datafiles
- __UNDO:__ Time limitations in that the necessary information to rollback changes must be available in the UNDO tablespaces, __RECYCLE BIN__ area, or in the __FLASHBACK LOG__
- __Datafiles changes:__ Must not have performed changes in the datafiles. 
- __Control File Recreation:__ Flashback is limited to the time after restoration or recreation of control file, because their actions clear the flashback log.

***Flashback Database Limitations***
- Initialization parameters configured
    - db_recovery_file_dest
    - db_recovery_file_dest_size
    - db_flashback_retention_target
- Smallest SCN FLASHBACK log
- Flashback turned on
    - SQL> ALTER DATABASE FLASHBACK ON;
- Effects whole instance

```SQL
-- Flashback CDB (at container level)

>>> sqlplus c##commonbkup/oracle as sysbackup
SQL> select oldest_flashback_scn, oldest_flashback_time from v$flashback_database_log;
SQL> select current_scn from v$database;

-- connect to pdb
SQL> select count(1) from testtable;
SQL> delete testtable;
SQL> commit;
SQL> select count(1) from testtable;
    -- no rows
    
-- connected at CDB
SQL> select current_scn from v$database;
SQL> shutdown immediate;
SQL> startup mount;
SQL> select oldest_flashback_scn, oldest_flashback_time from v$flashback_database_log;
SQL> flashback database to scn <scn_number>;
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database open resetlogs;
SQL> alter pluggable database all open;

-- connecting to pdb
SQL> connect username/oracle@pdb_name;
SQL> select count(1) from testtable;
    -- all rows are back

```

--------------------------------

***Flashback Database After PDB DBPITR***
Flashback Multitenant Limits:
- Root container part of whole recovery
- Limited to after last restore of PDB
- To perform flashback, follow the following steps:
    - ALTER PLUGGABLE DATABASE (bring down the pdb)
    - FLASHBACK DATABASE 
    - RESTORE PLUGGABLE DATABASE
    - RECOVER PLUGGABLE DATABASE


```SQL
-- CONNECT TO PDB AND CREATE TABLE, INSERT SOME DATA 
SQL> create table newtb as (select * from all tables);
SQL> commit;
SQL> insert into newtb (select * from all tables union select * from all_tables);
SQL> commit;

-- connected at root level CDB
RMAN> create restore point after_cdb_backup;
RMAN> create restore point after_pdb_name_newtb;
RMAN> list restore point all;

-- at pdb connection
SQL> drop table newtb;

-- at cdb connection
RMAN> create restore point after_pdb_name_newtbdrop;
RMAN> list restore point all;
RMAN> alter pluggable database pdb_name close immediate;
RMAN> run {
    set until scn <scn_number>;
    restore pluggable database pdb_name;
    recover pluggable database pdb_name;
    alter pluggable database pdb_name open resetlogs;
}

-- connected to pdb to bring datafiles offline
SQL> connect sys/oracle as sysdba;
SQL> alter pluggable database datafile all offline;

-- to RMAN
RMAN> alter system switch logfile;
RMAN> alter system switch logfile;
RMAN> alter system switch logfile;
RMAN> select current_scn from v$database;
RMAN> list restore point all;
RMAN> shutdown immediate;
RMAN> startup mount;
RMAN> flashback database to scn <scn_number>;
    -- if media recovery is finished
RMAN> alter database open resetlogs;
RMAN> restore pluggable database pdb_name;

-- goto pdb
SQL> alter pluggable database datafile all offline;
SQL> connect sys/oracle as sysdba
SQL> alter pluggable database pdb_name datafile all online;

-- goto man prompt
RMAN> recover pluggable database pdb_name;
RMAN> alter pluggable database pdb_name open;

```