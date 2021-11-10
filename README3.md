### Methods of Data Protection
- Backup types:
    - Physical backups are copies of physical database files
    - Logical backups are backups for logical data such as tables or stored procedures
- Backup Methods:
    - Oracle-managed Backup and Recovery (RMAN)
    - User-managed Backup and Recovery (OS commands)
- Backup-related procedures:
    - Data Archival: COpying data to long-term storage
    - Data transfer: ETL or OLAP scenarios
- Oracle Database Backup and Recovery Tools
    1. RMAN (Recovery Manager) command-line tool
    2. Oracle Enterprise Manager Cloud Control 12c
        - GUI frontend to RMAN
        - Data Recovery Advisor (DRA) - Automated corruption detection and repair utility
    3. Oracle Data Pump
        - PL/SQL packages to run from the command line
        - Perform logical imports and exports

**Preliminary Data Protection Tasks**
- **Enable ARCHIVELOG mode:** Periodically archives (backups) the online redo log files
- Back up the UNDO tablespace (despite arguments to the contrary)
- Specify the Fast Recovery Area
- Configure Flashback features
- Multiplex and back up the control file 
- Consider relocating your data files to different disks
- Back up the server parameter file

### Configuring the Oracle Recovery and Backup Environment

```
SQL> sqlplus /nolog
SQL> connect as sysdba
SQL> show con_name;
SQL> select log_mode from v$database;

# To perform archive we need to Shutdown first
SQL> shutdown immediate;
SQL> startup mount;

# changing to archivelog
SQL> alter database archivelog;
SQL> alter database open;

SQL> select log_mode from v$database;

# to clear screen from sql prompt
SQL> host clear;     # use "host cls;" in windows

SQL> show parameter recovery;
# db_recovery_file_dest is flash/fast recovery area

# if you want to change the file_dest recovery disk
SQL> alter system set db_recovery_file_dest = '/u02/app/oracle/oradata' scope=both;
    # Note: confirm your path carefully

SQL> show parameter control_files;
# control files are important as they contian lots of useful information

SQL> show parameter recovery;
SQL> alter database backup controlfile to trace;
# trace is for alert path or specify the path 

SQL> alter database backup controlfile to '/u02/app/oracle/oradata/control.bkp';

# backing up server parameter file is very easy
SQL> create pfile from spfile;

```

----

### Implementing the Oracle Flashback

**Oracle Flashback**
- Fast and easy historical querying/disaster recovery toolset originally introduced in Oracle Database 10g
- We can view past states of database objects and optionally return database objects to those previous states.
- Flashback relies upon AUM and Undo data
    - AUM the database manages undo segments in the undo tablespace
    - AUM gives Oracle the powder to determine how long data exists as an undo segment

**Oracle Flashback Feature Set**
- Flashback Database: Rewind a database to an earlier time
- Flashback Query: Run SELECTS on past data states
- Flashback Version Query: Compare past data states
- Flashback Transaction Query: Retrieve historical metadata from data transactions
- Flashback Table: Rewind a table to earlier state
- Flashback Drop: Undrop a dropped table

**Purpose of UNDO**
- To support transactions rollback
- To support read consistency
- To support Flashback operations

**Automatic UNDO Management**
- The goal is to avoid SNAPSHOT TOO OLD errors
- AUM is enabled with the UNDO_MANAGEMENT initialization parameter
- Set the minimum UNDO retention period with UNDO_RETENTION initialization parameter
- EM Cloud Control 12c includes the UNDO Advisor (PL/SQL interface as well)
- We can set undo guranttee at the UNDO tablespace level to ensure that unexpired undo data is not overwritten

**Configuring Oracle Flashback**
- Ensure you have enabled the Automatic Undo Management
    - ALTER SYSTEM SET UNDO_MANAGEMENT = AUTO;
    - Tweak retention period as necessary
- Set and configure the Flashback Recovery Area
    - DB_RECOVERY_FILE_DEST initialization parameter
- Put the database in ARCHIEVELOG mode
- Enable flashback for the database in MOUNT mode:
    - ALTER DATABASE FLASHBACK ON;

```
SQL> startup;
SQL> show parameter undo;
    # undo_retention value is 900 seconds 
SQL> show parameter db_recovery_file_dest;

SQL> select log_mode from v$database;
SQL> select flashback_on from v$database;

# If LOG_MODE is NOARCHIEVELOG and FLASHBACK_ON is NO
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database archievelog;
SQL> alter database flashback on;
SQL> alter database open;

SQL> select log_mode from v$database;
SQL> select flashback_on from v$database;
```

**Flashback Database**
- Rewind your entire database, not individual schema to a past time/date or system change number (transaction number)
- The V$FLASHBACK_DATABASE_LOG view tells you how far into the past you can flashback your database
- We can add restore points to our transaction to flash the database back to those
    - CREATE RESTORE POINT batchload062511;

**Flashback Table**
- Restore an Oracle table to its state as of previous point in time
- Maintains all table attributes (Indexes, triggers, constraints)
- Even the flashback saves the data in the original table. We can "flash forward" because all table data is recorded in the Flash Recovery Area and Undo tablespace

```SQL
SQL> create table t1 (id NUMBER, description VARCHAR2(50));
SQL> insert into t1 (id, description) values(1, 'blah');
SQL> insert into t1 (id, description) values(2, 'blah2');
SQL> select * from t1;

SQL> select * from v$flashback_database_log;
SQL> commit;
SQL> shutdown immediate;
SQL> startup mount;
SQL> flashback database to timestamp to_date('06/26/2014 13:55:00', 'mm/dd/yyyy hh24:mi:ss');

# checking database, lets open in read-only mode
SQL> alter database open resetlogs;
SQL> select * from t1;
    # gives error since we flashbacked
```

**Flashback Drop**
- We can't unring the bell, but you can flashback that table 
- We make use of the Recycle Bin
    - SHOW RECYCLEBIN
    - SELECT * FROM RECYCLEBIN;
    - ALTER SYSTEM SET RECYCLEBIN=ON;
- The RB is a data dictionary table that records dropped objects
- We can purge recycle bin
- Restoring a dropped table
    - FLASHBACK TABLE <table_name> TO BEFORE DROP RENAME TO myrestable;
- When we restore dependent objects (indexes) you have to rename manually as well

```SQL
SQL> create table t2 (id NUMBER, description VARCHAR2(50));
SQL> insert into t2 (id, description) values (1, 'blahblah');
SQL> insert into t2 (id, description) values (2, 'sdfsdf');
SQL> commit;

SQL> create table t3 (id NUMBER, description VARCHAR2(50));
SQL> insert into t3 (id, description) values (100, 'sfdfsdfsdfsdfs');
SQL> commit;

SQL> show parameter recyclebin;
    # must be turn on

SQL> select table_name from user_tables;
SQL> show recyclebin;
SQL> drop table t3; 
SQL> show recyclebin;

Note: if you logged in as sys you will get access to recyclebin 
SQL> connect username;
SQL> show recyclebin;
SQL> FLASHBACK TABLE T3 to BEFORE DROP;
SQL> SELECT TABLE_NAME FROM USER_TABLES;
    # Must get the table t3

SQL> connect / as sysdba;
SQL> select * from t2;
SQL> select row_movement from user_tables where table_name='t2';
    # disabled
SQL> alter table t2 enable row_movement;
SQL> flashback table t2 to timestamp to_date('data format');
```

**Flashback Data Archive (FDA)**
- Previously known as Oracle Total Recall
- FDA is an extended store of undo information
    - Consist of one or more tablespaces
- We can set retention period to Maintain compliance
- We then associate tables with the FDA to create a historical record of its data

```SQL
SQL> create tablesace fda_ts
    datafile '/u01/app/oracle/oradata/fda1_01.dbf'
    size 10M autoextend on next 1M;
SQL> create flashback archive default fda_5year tablespace fda_ts quota 10G retention 5 year;

SQL> create table archieve1 (id number, description VARCHAR2(50)) flashback archive;

```

---

## Performing Database Backup

```bash
                +------+        +------+        +------+ 
                | SMON |        | PMON |        | RECO |
                +------+        +------+        +------+
    
    +------------------------------------------------------------------+
    |                               SGA                                |
    | +-----------------------+  +-------------+  +-----------------+  |
    | | Database Buffer Cache |  | Shared Pool |  | REDO log Buffer |  |
    | +-----|-----------------+  +-------------+  +------|----------+  |
    +-------|--------------------------------------------|-------------+
            |                                  __________| 
            |                                 | 
            V                                 V
        +------+     +------+             +------+   +------+
        | DBWR | <---| CKPT |             | LGWR |   | ARCH |
        +------+     +------+             +------+   +------+
            |           |   |__             |  |__      ^
            |           |      |            |     |     |
            V           V      V            V     V     |
        +----------------+    +---------------+  +-----------+
        |   Datafiles    |    | Control files |  | Redo logs |
        +----------------+    +---------------+  +-----------+
```

Checkpoint:
- CKPT increments the SCN in the datafile headers and control file, and DBWR writes dirty buffers to datafiles
- LGWR updates SCN in the redo logs and flushes buffer to ORLFs
- ARC responsible for backing up ORLFs

**RMAN**
- It a command-line backup and recovery client
- GUI front-end Enterprise Cloud Control 12c
- New SYSBACKUP sub-administrative privilege in Oracle Database 
- Block-based backup means that you can save space in your backup sets (no backup of empty space in files)
- Recovery catalog is separate schema that supports longer backup set history than control file (Also supports storage of RMAN scripts)
- Can use the tool interactively or call rman from scripts
- Preferred backup destination: Fast Recovery Area (Can also use System Backup to Tape (SBT) devices)

**Creating Cold and Hot Backups**
- Closed (consistent or closed) backups require that you put database in MOUNT state
- Hot backups require ARCHIVELOG mode
- Image copy backups are bit-for-bit duplicates of the target files
- Backup sets can be cataloged, compressed and encrypted
- You can multiplex several data files into a single backup piece, and you can also create redundant copies (duplexes) of your backups

**Creating Incremental Backups**
- Full Backup
- Level 0 incremental: Special kind of full backup; baseline for incremental strategy 
- Level 1 incremental: Picks up changes
        - Differential incremental: changes since level 1 or level 0
        - Cumulative Incremental: changes since level 0
- In recovery, we use redo logs to restore last committed transaction

```bash
    |   |   |   |   |   |
    |   |   |   |   |<--|
    |   |<--|-------|   |
    |   |   |<--|   |   |
    |   |<--|   |   |   |   |   |
    |<--|   |   |   |   |   |   |
    |___|___|___|___|___|___|___|_____ Upto Nine levels
    0   1   2   3   2   3
        Backup level  -->                          
```

```bash
>>> which rman
>>> rman
    RMAN> exit

# to login as system user
>>> rman target /
RMAN> rman "username@servicename as sysbackup"
RMAN> show all;

# to change parameter setting
RMAN> configure backup optimization on;

RMAN> show all;
RMAN> backup database include current controlfile;

# backing up using archive log
RMAN> backup database plus archivelog;
```

**Automating Database Backup**
- Create a recovery catalog (can't be SYS)
- Register your database with the recovery catalog
- Compose your RMAN backup scripts
    - Save them to the recovery catalog as stored scripts
- Use Cloud Control or OS-level scheduler to run your backup scripts

**Managing Your Backup Library**
Query the RMAN repository 
- LIST gives historical metadata
- REPORT lets your know which backups are expired and other management information
- SHOW reveals RMAN configuration 
- PRINT SCRIPT shows stored scripts

*Working with RMAN*
```SQL

RMAN> alter system switch logfile;
RMAN> backup spfile;
RMAN> backup incremental level 0 tablespace users;

# this differential backup by default (level 1)
RMAN> backup incremental level 1 tablespace users;

# performing cumulative backup
RMAN> backup incremental level 1 cumulative tablespace users;

RMAN> list backup;
RMAN> list backup by file;
RMAN> spool log to '/home/oracle/backuplog.f';

# select * from v$datafile;
RMAN> backup datafile 1;
>>> cat '/home/oracle/backuplog.f'
>>> rman target /

RMAN> shutdown immediate;
RMAN> startup mount;
RMAN> backup database;
RMAN> backup database tag 'full backup';

```
---

## Automating Database Backup

- *Recovery catalog:* Database schema that serves as a centralized source of backup metadata
- Oracle backup information is stored in the control file and that has limited capacity for storage
- Use the CONTROL_FILE_RECORD_KEEP_TIME initialization parameter to manage the RMAN repository in the control file.
- Best practice is to create a separate database, because if the recovery catalog is created within the same schema or database and if anything goes wrong with production database then recovery catalog will also be lost, so it is single point of failure. Therefore use best practice.
- Can also store RMAN script files, stored scripts has two scope global and local

**Creating a Recovery Catalog**
- Create RC owner user with RC tablespace
- Grant permission to the RC owner
- Create RC
- Register database(s) with RC

```SQL
>>> sqlplus / as sysdba
SQL> create tablespace cattbs;

SQL> create user rman identified by Password (in my case I have given `rman`) temporary tablespace temp default tablespace cattbs quota unlimited on cattbs;

SQL> grant recovery_catalog_owner to rman;

SQL> exit;

>>> rman target /
RMAN> connect catalog rman/rman
    # in case recovery database
    # rman@recovery_database_name this is best practice

RMAN> create catalog;
RMAN> register database;    
    # registering local database
    # in case of multiple database, do this procedure on each of those targets

RMAN> report schema;
    # to see whats in the recovery catalog

RMAN> exit;

# using RMAN credentials to connect to catalog
>>> rman target / catalog rman/rman@psight
    # psight is the service name, in this case we created from default file so using the service name as psight

RMAN> 
```

**Options for Scheduling Oracle Backups**
- OS shell scripts that call RMAN, generally called bash scripts in unix, batch file in windows. In those files we simply have to calls to RMAN and call the RMAN executable and then in turn we can call RMAN scripts.
- OS-level scheduling utilities (using crontab in linux)
- Recovery Catalog stored scripts
- EM Cloud Control

```SQL
>>> rman target / catlog rman/rman@psight
RMAN> exit;

>>> vim rmanbackup.rman
    # add the following lines of code
    run {
        allocate channel ch1 device type disk format '/u01/app/oracle/oradata/%U';
        backup database;
        release channel ch1;
    }
    # use of %U is for autoformatting

>>> vim shellbackup.sh
    # add the following code:
    
    #!/bin/bash 
    rman target / catalog rman/rman@psight @rmanbackup.rman

>>> chmod +x shellbackup.sh
>>> ./shellbackup.sh

# storing scripts in to recovery catalog
>>> rman target / catalog rman/rman@psight
RMAN> create global script global_full_backup {
           backup database;
       }

RMAN> list global script names;
RMAN> print script global_full_backup;

# to delete the script 
RMAN> delete script <script_name>;

# to run or execute scripts internally
RMAN> run {execute script global_full_backup;}
```

---

## Performing Database Recovery

- Determine the need for performing recovery 
- Use Recovery Manager (RMAN) and the Data Recovery Advisor (DRA) to perform recovery of:
- The database
- The control file 
- The redo log file 
- The data file

**Media Failure**
- Oracle defines "media failure" as the loss or corruption of an oracle physical database element
    - Data file, control file, redo log file, undo file, server parameter file
- Types of Database failure:
    - Statement failure
    - User process failure (with or without user error)
    - Network error
    - Instance failure (malware, malicious intrusion, environmental)

**Phase of Instance Startup**
- SHUTDOWN: A crash is something like issuing a SHUTDOWN ABORT
- STARTUP
    - NOMOUNT: Access the SPFILE; no datafiles checked
    - MOUNT: Accesses the control file; datafiles verified

**Complete VS Incomplete Recovery**
- ***Complete Recovery:*** Using backups and redo log stream to restore to the last committed transaction. Incase of database crash (Oracle uses UNDO data to rollback uncommitted transaction to the previous data state)
- ***Incomplete Recovery:*** Using the same, but to restore to a particular in time (normally used when a complete recovery fails)
- ***Phases of Oracle Recovery:***
    - Restoration of physical files
    - Recovery, which is synchronization of system change numbers (SCNs) in the control file and the data files

Understanding Oracle Recovery command is important which performs media recovery, and alter database reset log statement. Reset log is important, actually mandatory when we do recovery because if we bring back data files or if we bring back a control file from the backup, we are going to have system change numbers that are different between the control file and your data files. So in order to make sure that those are synchronized once recovery takes place we can essentially put a special record in our control file and in our data file headers. This is what it means start over from here, start a new instantiation of this database.
This is what reset log does, it resets the SCN, such that the database knows that it's been recovered and now everything is in sync.

**Data Recovery Advisor (DRA)**
- Feature introduced in Oracle Database 11g
- Automatically gathers detailed data failure information
- Proactively checks for failures
- Provides detailed advice on resolving the problem
- Available in Enterprise Manager Cloud Control 
- RMAN interface to the DRA as well:
    - LIST FAILURE DETAIL;
    - ADVISE FAILURE;

**Recovering From the Loss of a Control File**
- A missing or damaged control file makes the instance abort
- We should have an FRA with control file auto backup enabled in RMAN

```SQL
>>> rman target /
RMAN> startup nomount;
RMAN> restore controlfile from autobackup;

# to check configured auto backup
RMAN> show all;

# use the show parameter options to change the default
RMAN> configure ......

RMAN> 

# using SQL plus
SQL> show user;
SQL> show paramter recovery_file_dest;
SQL> show paramter control_files;
SQL> host
>>> rm -rf /remove_one_ctl_file_if_have_many

>>> sqlplus / as sysdba
SQL> show paramter control;
    # if still shows the control files(all) restart the instance
SQL> shutdown abort;
SQL> startup;
    # now will give ora error regarding control file

SQL> startup nomount;
    # gives error again because of control file

>>> rman target /
RMAN> restore controlfile from autobackup;
RMAN> startup mount;
RMAN> alter database open;
    # gives error related to reset logs
RMAN> alter database open resetlogs;
    # another error if (related to consistency)
RMAN> recover database;
    # (recovery is the process of synchronization)
RMAN> alter database open resetlogs;
    # no error then fine

```

**Recovering From the Loss of a Redo Log File**
Process: 
- Use the DB Express to study redo log file group members
- Inspect the alert log to see which redo log members have a failure 
- Query v$log and v$logfile to determine same
- Force a log switch to move away from troubled file
- Drop the damaged member
- Add a new member to the group

**Restoring the Entire Database**
This is worst case scenario.
- Instance should be running in ARCHIEVELOG mode and have good baseline backup of your database
- Procedure:
    1. Determine which files need to be restored. (select file#, status, error, recovery from v$datafile_header;)
    2. Set database in proper startup mode
    3. Use RESTORE command to restore files from backup 
    4. Use RECOVER to synchronize SCNs
    5. Open the database 
- We can restore tablespaces the same was as the entire database
    - RESTORE TABLESPACE users;
    - RECOVER TABLESPACE users;

```SQL
>>> sqlplus / as sysdba
SQL> select file#, name from v$datafile;
SQL> host

# for demo removing the system01 file
>>> rm -f /system_file_from_above_query
>>> sqlplus / as sysdba
SQL> shutdown immediate;
    # gives error
SQL> startup mount;
SQL> shutdown abort;
SQL> startup mount;

>>> rman target /
RMAN> restore datafile 1;
    # 1 is datafile number
RMAN> recover datafile 1;

RMAN> alter database open;
    # no error then OK
```

**Using DRA Dashboard**
- Refer to doc

---

## Moving Data

**Options for moving data**
- Copy table data between schemas
    - CREATE TABLE newtable AS SELECT * FROM schema.oldtable
- Extract, Transform, Load (ETL)
    - MySQL > Intermediate > Oracle
    - SQL Server > Intermediate > Oracle
- Exporting data out of Oracle
    - Tables
    - Tablespaces

**SQL *Loader**
```SQL

        +-----------+   +------+    +---------+ 
        | Parameter |   | Data |    | Control |
        |   File    |   | File |    |   File  |
        +-----------+   +------+    +---------+
              |             |            |
              |             |            |
              |             V            |
              |     +-------------+      |          +----------+
              +-->  | SQL *Loader |  <---+   +--->  | Log File |
                    +-------------+          |      +----------+
                        |        |           |
                        V        +-----------+     +--------------+
                    +----------+             |---> | Discord File |
                    | Database |             |     +--------------+
                    +----------+             |
                                             |      +----------+
                                             +--->  | Bad File |
                                                    +----------+

```

*Using SQL\*Loader*
- Using command line __sqlldr__
- Oracle Cloud Conrtol 

> Parameter File includes (parameter_filename.par)
> userid/password
> control=control_filename.ctl
> log=log_filename.log
> bad=bad_filename.bad
> data=data_filename.dat
> direct=true

```
>>> vim control_filename.ctl
    LOAD DATA into table table_name
    INSERT FIELDS TERMINATED BY ","
    (COLUMN_NAME1, COLUMN_NAME2, ...)

# TABLE MUST BE CREATED
>>> sqlldr parfile=/paramter_file_path
>>> check the data is loaded in table or not
```

**Data Pump**
- High Speed data and metadata load/unload utility
- Access using command-line (expdp and impdp) and Cloud Control, underlying engines are PL/SQL packages DBMS_DATAPUMP and DBMS_METADATA
- Cool Features:
    - Compression, encryption
    - Direct network based import and export  (between two linked oracle DBs)
    - Ability to rename tablespaces, schemas and tables
    - Can create DP scheduled jobs

```bash
# Export

>>> vim table_name_export.par
    # must contain the following parameters
    userid=username/password
    directory=dept_directory
    dumpfile=table_name%U.dmp
    logfile=table_name_export.log
    filesisze=50M
    tables=table_name

>>> sqlplus / as sysdba
SQL> show user;
SQL> create directory table_name_directory as '/directory_path_where_to_create/dump';

>>> expdp parfile=table_name_export.par

# Import

>>> vim table_name_import.par
    userid=username/password
    directory=table_name_directory
    dumpfile=table_name_export__.dmp
    logfile=table_name_import.log
    remap_table=schema_name.table_name:table_name_copytable

>>> impdp parfile=table_name_import.par

SQL> select * from table_name_copytable;
```

---

### Managing PDBs and Backing up

**Managing Pluggable Database**
```SQL
>>> sqlplus / as sysdba

SQL> show con_name;
SQL> show parameter db_name;
SQL> exit

>>> sqlplus system@pdb1
    #pdb1 is pluggable database in my case

SQL> show con_name;

# switching to cdb 
SQL> alter session set container = cdb$root;
SQL> show con_name;

SQL> show pdbs;
SQL> alter session set container = pdb1;

SQL> create user pluguser indentified by Pa$$w0rd;
SQL> grant dba to pluguser;

SQL> column username format a10;
SQL> column common format a10;
SQL> select username, common from all_users;

SQL> alter session set container = cdb$root;
SQL> column name format a10;
SQL> select name, con_id from v$container order by con_id;

SQL> column pdb_name format a10;
SQL> column status format a10;
SQL> select pdb_name, status from cdb_pdbs;

# to check the current copntext
SQL> select cdb from v$database;
```

**Unplugging the PDBs**
```SQL

SQL> show con_name;
# get out from pdb
SQL> alter pluggable database pdb1 close immediate;
SQL> alter pluggable database pdb1 unplug into '/path_where_to_store/pdb_name.xml';
    # xml file format is must

SQL> drop pluggable database pdb1 keep datafiles;
SQL> 
```

**Plugging PDB**
```SQL
SQL> show user;

Refer to doc to test the pluggable_database.xml compatibility, A small script to test.

SQL> create pluggable database pdb1_plug using 'xml_file_path' nocopy tempfile reuse;

Note: if you get some error check the xml file path reference must match

```

**Backing up PDBs**
```SQL
>>> rman target /
RMAN> backup pluggable database pdb_name;
RMAN> 
```