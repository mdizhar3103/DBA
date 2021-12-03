### Database Recovery
Instance Recovery: When the database applies redo data from the online container database redo logs all the transactions are copied to bring the container database upto date. Once you open the root container the database rollsback the uncommitted transactions in the root datafiles. It is only the open pdb that uncommitted transactions are rolled back. 

*Control file, Redo Logs and UNDO tablespaces belong to database instance not to individual pdb. If any of the datafile are lost the complete database backup recovery must be performed*

```SQL
-- Instance Recovery
SQL> select name from v$database;
SQL> select name, open_mode from v$containers;

SQL> shutdown abort;
SQL> startup;
SQL> select name, open_mode from v$containers;

-- after instance recovery we have to open PDBs because the goes into MOUNTED mode
SQL> select name, open_mode from v$containers;
SQL> alter pluggable database all open;
SQL> select name, con_id from v$datafile;

-- copy the path of any datafile and delete it
>>> rm <datafile_path>

-- after deleting shutdown the instance
SQL> shutdown immediate;
    -- gives some error related to n such file or directory

SQL> connect sys/oracle as sysdba
SQL> startup mount;

-- connecting via RMAN
>>> rman target sys/oracle
RMAN> restore database;
-- now performing database recovery
RMAN> recover database;
RMAN> alter database open;      -- open the root container not pdbs
RMAN> select name, open_mode from v$containers;
RMAN> alter pluggable database all open;
RMAN> select name, open_mode from v$containers;


-- PDB Recovery
>>> sqlplus sys/oracle@myplug as sysdba
SQL> select name from v$datafile;
SQL> !rm <path of any datafile>
SQL> shutdown abort;

RMAN> select name, open_mode from v$containers;
RMAN> restore pluggable database myplug;
RMAN> alter pluggable database myplug open; -- gives error because of missing datafile
RMAN> recover pluggable database myplug;
RMAN> alter pluggable database myplug open;
RMAN> select name, open_mode from v$containers;

```

------------

**CDB Point-in-Time Recovery**
- Sometimes we dont need to perform full recovery, this is scenario of data corruption, physical changes from data dictionary language command or bug in a software.
- Some limitations in PITR:
    - The database is unavailable during the restore process. 
    - RMAN must restore all datafile possibly REDO logs in incremental backups to recover datafiles 



**Pluggable Database PITR**
- In container database environment, the datafiles and tablespaces belong to the individual PDBs. This fits well with the ability to recover individual pluggable databases.
- RMAN can restore the pluggable database datafiles in place.
- Rolling forward to the correct point in time is the problem, this roll forward action needs the undo tablespaces and the redo logs from the same point in time that the datafiles were backed up. To this end, RMAN creates a clone auxiliary instance and restores the redo logs, the system, sysaux and undo tablespace of the CDB ROOT.
- It also restores the SEED pluggable database.
- It will then use the auxiliary database instance to roll forward the changes to the pluggable database files using undo in archive log files


```SQL

-- backing up single pdb in PITR
>>> rman target sys/oracle
RMAN> backup tag 'for_pitr' (database ROOT)(pluggable database pdb_name) plus archivelog delete input;

-- checking the everything working correctly
SQL> connect system/oracle          -- new terminal
SQL> select current_scn from v$database;
SQL> create table testtable as (select * from all_objects);
SQL> select current_scn from v$database;
SQL> insert into testtable (select * from testtable where rownum<1000);
SQL> commit;
SQL> select count(1) from testtable;
SQL> drop table testtable;
SQL> select current_scn from v$database;


>>> rman target sys/oracle
RMAN> run {
    set until SCN=<previous scn number to restore>
    restore pluggable database pdb_name;
    recover pluggable database pdb_name auxiliary destination='/oradata/AUXCDB';  -- this is include flashback path by default
    alter pluggable database pdb_name open resetlogs;
}   -- dont run till the database is shutdown

-- other terminal
>>> connect sys/oracle@pdb_name as sysdba
SQL> shutdown immediate;
-- now run the rman run block



-- Checking our PITR actually works
SQL> connect system/oracle@pdb_name
SQL> select count(1) from testtable;
    -- table exist 

```

**Summary of log File**
1. Files for PDBs recovered "in place"
2. Creating of clone AUXILIARY instance
3. Restore clone controlfile
4. Giving the clone access
5. Restore SYSTEM, SYSAUX, and UNDO tablespace of CDB$ROOT
6. Online datafiles
7. DPITR of "clone" database
8. Remove automated clone instance
9. Delete datafiles used in recovery (not all deleted)


**Table and Table Partition Recovery**
1. Determines which backup to use based on PIT
2. Creates auxiliary database
3. Recovers specified tables until the specified PIT into auxiliary database
4. Uses Data Pump to export recovered tables
5. (Optional) Imports export dump file into target instance
6. (Optional) Renames the recovered tables

*Decision to Make*
- Point-in-Time: to which the table must be recovered to recover
- Automatic import: Table to be automatically imported into the target database
- Rename table: If you want to rename the table
- Different tablespace: If you want to recover in different tablespace


```SQL

-- testtable is the table exist in pdb check your own table name
SQL> connect system/oracle@pdb_name
SQL> select current_scn from v$database;
SQL> select count(1) from testuser.testtable;
SQL> insert into testuser.testtable (select * from testuser.testtable);
SQL> commit;
SQL> select current_scn from v$database;        -- note the scn number

>>> rman target sys/oracle@cdb_name             -- to recover database we must need to connect with root database
RMAN> recover table testuser.testtable of pluggable database pdb_name 
until scn <scn_number> 
auxiliary destination '/oradata/AUXCDB';
    -- if table exist need to drop the table

SQL> drop table testuser.testtable;
RMAN> recover table testuser.testtable of pluggable database pdb_name 
until scn <scn_number> 
auxiliary destination '/oradata/AUXCDB';

SQL> select count(1) from testuser.testtable;
    -- table exists
```

*Limitations on table recovery*
- Not SYS, SYSTEM or SYSAUX
- READ-WRITE and ARCHIVELOG mode
- Not standby database
- No REMAP option when NOT NULL
- RMAN backups exist
- Will restore entire tablespace
- PDB specific requirements
    - Backup includes CDB$ROOT UNDO, SYSTEM and SYSAUX
    - Connect through CDB$ROOT
    - PDB backup includes SYSTEM AND SYSAUX


