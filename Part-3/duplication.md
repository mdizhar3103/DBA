### Duplicating Database

**Duplicating CDB and PDB**
Setup for duplication:
- Backup of TARGET available
- Configuration files updated/created
    - listerner.ora
    - tnsnames.ora
    - init<oracle_sid>.ora
- Connection to TARGET and AUXILIARY root via RMAN

CDB duplication: It does not matter you are duplicating the entire database or just pluggable database. By default, RMAN also duplicates the ROOT container and the seed database as part of the process. Because, we need container database instance to plug databases into.

```SQL
-- duplicating the entire container database
>>> rman target sys/oracle 
RMAN> report schema;        -- to see schema details
RMAN> backup tag base_for_clone database plus archivelog;

-- creating init file 
>>> sqlplus sys/oracle as sysdba
SQL> create pfile from spfile;
SQL> exit;
>>> cd $ORACLE_HOME/dbs
    -- check for init file
>>> mv createdinit_file init_newdb.ora
-- once file is created perform the changes according to your need say dbname change, path change,etc.
>>> export ORACLE_SID=new_db_name
>>> sqlplus / as sysdba
SQL> startup nomount pfile='/u01/app/oracle/product/19.1.0/dbs/init_newname.ora';
    -- incase of memory target not supported change the init_newname.ora file parameter memory_target to sga_target
SQL> startup nomount pfile='/u01/app/oracle/product/19.1.0/dbs/init_newname.ora';

-- checking the backup
RMAN> list backup; 
    -- check the backup is completed or not
RMAN> exit;

-- Connecting locally
>>> export ORACLE_SID=new_db_name
>>> rman target sys/oracle@pdb_name auxiliary sys/oracle
    -- auxiliary is new_db_name
RMAN> duplicate database to auxcdb;
RMAN> exit; 
    -- if completed then exit RMAN

>>> echo $ORACLE_SID            -- new_db_name
>>> sqlplus sys/oracle as sysdba
SQL> select name, open_mode from v$pdbs;

-- duplicating pdbs
>>> sqlplus /nolog
SQL> connect / as sysdba
SQL> startup nomount;
SQL> exit;

>>> rman target sys/oracle auxiliary sys/oracle
RMAN> duplicate database to auxcdb pluggable database pdb_name;
-- watch the logs carefully

```

**Active Database Duplication**
- Using backup sets, when sufficient channels are allocated the auxiliary instance connnects to the target instance and then retries the backup set over the network thus reducing the load on target instance. Unused block compression can be used during the duplication process, thus reducing the size of the backups transported over the network. We can specify the binary compression level to be used, we can also encrypt the backups, as well as use the multisection backup features while performing active database duplication. 
- One varaition on active database duplication is referred to as the push-based method. This method was available in previous version of oracle. This method involves RMAN copying the database over the network using image copies.  RMAN connects to the source database as the target, and the destination database as the auxiliary, and the target then copies the mounted or online database files  over the network to the auxiliary instance.
- Another variation of active database duplication is referred to as the pull-ased method. In this method, instead of using image copies, backup sets are used. Similar to the push-based method, RMAN connects using the source database as the target and destination database as an auxiliary. RMAN creates the backup set, and the auxiliary instance connects to the source database using Oracle Net Services, and retrieves any required database files from the soruce database. 

*Active duplication either uses image copies or backup set*

| Image Copies | Backup Set |
|--------------|------------|
| No AUX channels allocated | Used when the connection is through a Net Service name and the SET ENCRYPTION command has been run |
| AUX channels allocated are less than TARGET channels | Or options with COMPRESSED BACKUPSET or SECTION SIZE are utilized. |
| Image copies are transmitted via Oracle Net to the auxiliary database instance |  Optional - compression, multisection |


```SQL
-- Image Copies: Push Method
-- Backup Set: Pull Method 

>>> cd $ORACLE_HOME/dbs
>>> orapwd file=orapwdauxcdb force=y
>>> cat ../network/admin/tnsnames.ora
>>> lsnrctrl reload
>>> lsnrctrl status
>>> rman target sys/oracle@pdb_name auxiliary sys/oracle
RMAN> run {
    allocate auxiliary channel dupdb1 type disk;
    allocate auxiliary channel dupdb2 type disk;
    allocate auxiliary channel sourcedb1 type disk;
    allocate auxiliary channel sourcedb2 type disk;
    allocate auxiliary channel sourcedb3 type disk;
    duplicate target database to auxcdb from active database;
}


-- Pull method using backup set
>>> rman target sys/oracle@svc_name_or_pdb_name auxiliary sys/oracle
RMAN> duplicate target database to auxcdb from active database section size 200M;
-- Read the logs and see what RMAN does behind the scene

```

----

### New General RMAN Enhancements
**Using SQL in RMAN**
```SQL

>>> rman target '"c##commonbkup/oracle@pdb_name as sysbackup"'
-- in previous version we use `sql` predicate to run the query
RMAN> select * from v$database;
```

**Restoring and Recovering Files Over the Network**
```SQL
>>> rman target '"c##commonbkup/oracle@pdb_name as sysbackup"'
RMAN> RESTORE DATAFILE '/u01/.../filename.dbf' FROM SERVICE svc_name SECTION SIZE 120M; -- Backup sets are used, uses incremental backup and only used with data blocks

```

