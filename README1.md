## Oracle Database Fundamentals
- Relation = Table 
- Attribute = Column
- Tuple = Row

**Database and Instance**
- __Instance:__ Shared Memory Area (System Global Area, or SGA) and background process
- __Database:__ Set of files, located on disk that store the data. Data files, ORL files, control files, etc
- __Single Instance vs Clustered (grid) Instance:__ Cluster involves several instances that access shared storage and a single database in an actve configuration
- **Multitenant Architecture:**
    - Container Database (CDB) with multiple pluggable databases (PDBs) 
    - Provides hardware cost reduction, easier Management/ monitoring.
    - Updates and maintenance can be performed just once on the CDB.

**Oracle Database Memory Structure**
- __SGA (System Global Area):__ Group of shared memory structures that store data and Manage information for single database instance.
- __PGA (Pluggable Global Area):__ Stores data and manages information for a server process or background process.

**SGA Components**
- __Database buffer cache:__ Stores copies of data blocks read from data files and waiting to be written to data files
- __Redo log buffer:__ Circular buffer that stores all changes made to the DB (can reconstruct DDL or DML operations)
- __Shared Pool:__ Catch all bucket, Shared SQL, Data Dictionary cache, etc.
- __Large Pool:__ Stores memory allocations that are larger than the shared pool can handle (overflow).
- __Java Pool:__ Stores session-specific java code within the JVM 
- __Streams Pool:__ Used in Oracle Streams database replication
- __Fixed SGA:__ Internal housekeeping area (locks, processes, etc)

> [Ref](https://www.oracle.com/webfolder/technetwork/tutorials/obe/db/12c/r1/poster/OUTPUT_poster/poster.html#)

**PGA Components**
- File clerk/countertop analogy for understanding the server process and the PGA
- Session memory: Logon information and session related data 
- Private SQL area: Query execution work data
- Restricting the PGA size (PGA_AGGREGATE_LIMIT initialization parameter)

**Memory Management in Oracle Database**
- Automatic Memory Management - MEMORY_TARGET initialization parameter
- Automatic or Manual Shared Memory Management - Pertains to SGA
- Automatic or Manual PGA Memory Management

---

### Oracle Database Architecture
Reference [Link](https://www.oracle.com/webfolder/technetwork/tutorials/obe/db/12c/r1/poster/OUTPUT_poster/poster.html#)

**Oracle Database Background Process**
- Client process (Oracle client), background process, server process.
- Multithreaded Oracle Database model
- Mandatory and optional background process
    
**Mandatory Background Process**
- __PMON:__ (Process Monitor) Cleans up caches and fixes user process sesssions, cleaning up the session in SGA.
- __LREG:__ (Listener Registration) Orchestrates the net listener
- __SMON:__ (System Monitor) Performs instance recovery, If database crashes due to power failure or otherwise shutdown in a non-graceful way. Oracle defines system change number (SCN) as the logical point in time at which changes are made to a database.
- __DBWn:__ (Database writer) Writes modified buffer in the buffer cache to disk. We can use DB_WRITER_PROCESS initialization parameter to specify upto 100 DBWn processes. It make sure changes made in database are permanently stored.
- __LGWR:__ (Log Writer) Writes redo log buffer to an online redo log file every 3 seconds
- __CKPT:__ (Checkpoint Process) Records checkpoint information in control file and data file headers. Note log writer just instructs the checkpoint to write a checkpoint and there are many scenarios where checkpoint happens.
- __MMON:__ (Manageability Monitor) Performs tasks on behalf of the AWR (Automatic Workload Repository). AWR is subsystem in oracle that is involved in reporting, stats, performance monitoring.
- __RECO:__ Manages distributed transactions 

**Optional Background Processes**
- The IT security principle of least service 
- __ARCn:__ Copies ORLFs to another storage device during a log switch 
- __CJQ:__ (Job Queue Coordinator Process) Manages the job queue
- __FBDA:__ Archives rows in the Flashback Data Archive
- __SMCO:__ (Space Management Coordinator Process) Performs various space management tasks, this is for internal process.


```SQL
SQL> select pname from v$process where pname is not null order by pname;
    # this will pull data not on disk which is basically from virtual table
```

**Oracle Physical Storage**
- File system with Optimal Flexible Architecture (OFA) naming 
- __Contorl File:__ Contains metadata concerning the physical structure of the database (Multiplexing)
- __Online redo log:__ Contains changes made to the database data (DDL, DML)
- __Server parameter file:__ Binary file that stores initialization parameter

**Oracle File Storage Options**
- File system with Optimal Flexible Architecture (OFA) naming
- Automatic Storage Management (ASM)
    - File system and logical volume manager for Oracle Database
    - Uses disk groups to store files
- ASM requires a separate installation
    - Oracle Enterprise Manager Cloud Control 12c
    - Oracle Grid Infrastructure 

**Oracle Database Logical Storage Structures**
- __OS blocks:__ Allocation units or clusters
- __Oracle data blocks:__ Logical collection of OS blocks
- __Extents:__ Specific number of contiguous data blocks
- __Segments:__ Sets of extents allocated to single object(table, index)
- __Tablespaces:__ Logical container for segment

**Oracle Management Tools**
- __SQL *PLUS:__ CLI for the Oracle Database
- __SQL Developer:__ Graphical utility for DBA and development work
- __EM Express:__ Lightweight Web Interface
- __EM Cloud Control:__ Robust, multi-platform management tool
- __DBCA:__ Create and manage traditional and pluggable database
- __NetCA:__ Wizard-based net services configuration tool
- __Netcfg:__ Java-based graphical front-end for Oracle Net Services
- __OUI:__ Java-based software installation and maintenance tool for Oracle products

----

### Oracle Instance Configuration
- When an Oracle instance is started one of the first things that happened is that the instance reads its **initialization parameter file** in order to determine the operating characteristics of the database. 
- If oracle instance doesn't have a valid PFILE or SPFILE as they're called, the database will not be going to open. 
- An initialization parameter file must contain, at minimum , the ***DB_NAME parameter***
- Server paramter file (SPFILE): binary format 
    - Linux: ORACLE_HOME/dbs/initSID.ora
    - Windows: ORACLE_HOME\database\initSID.ora

> Note: PFILE is text representation file

**Some Key Initialization Parameter**
- __Memory settings__
    - MEMORY_TARGET
    - MEMORY_MAX_TARGET
- __Fast Recovery Area__
    - DB_RECOVERY_FILE_DEST
    - DB_RECOVERY_FILE_DEST_SIZE
- __Control Files__
    - CONTROL_FILES
- __Oracle block size__
    - DB_BLOCK_SIZE

**Working with initialization parameter**
```SQL
>>>  sqlplus /nolog
    # this will not give access to anything
SQL> help;
SQL> connect system as sysdba;
    # requires password
SQL> exit

# Easy connect with sqlplus
>>> sqlplus system as sysdba
SQL> show parameters;
SQL> show parameter spfile;

SQL> show parameter memory;

# to change the memory target
SQL> alter session set memory_target = 1G scope=memory;
    # Note: specifying the memory target above MEMORY_MAX_TARGET will cause problems so do it carefully. Note: to make permanent change use the `both` scope or spfile

SQL> show parameter memory;

SQL> startup force;
    -- restarting instance forcelly, dont do this in production level server. Doesn't save any transaction. Specifically, the STARTUP FORCE command runs the SHUTDOWN ABORT and STARTUP OPEN commands. Use it in development environment.

SQL> show parameter memory;
    # this will again set the memory target to default in parameter file

SQL> alter session set memory_target = 1G scope=both;
    # changing the scope and `both` will make the changes persistent

SQL> help index;
SQL> help shutdown;
```

**Phases of Oracle Database Startup**
```
 _
/|\ STARTUP                                                  OPEN               |
 |                                                      +-----------------      |
 |                                                      | Database opened       |
 |                                       MOUNT          | for this instance     |
 |                                     +----------------+                       |
 |                                     | Control file                           |
 |                                     | opened for this                        |
 |                      NOMOUNT        | instance                               |
 |                   +-----------------+                                        |
 |                   | Instance Started                                         |
 |                   |                                                          |
 |     ____SHUTDOWN__|                                                         \|/ SHUTDOWN
                                                                                
Refernce: Pluralsight
```

- At MOUNT stage the control file has located the data files, the online redo logs files has ready for business as long as there are no errors.
- OPEN stage allows user connection to come in.
- STARTUP NOMOUNT or STARTUP MOUNT is used when we want to restore the database and dont want to allow user to connect to the database.

**Shutdown Modes**
[Link](https://www.oracletutorial.com/oracle-administration/oracle-shutdown/)

```SQL
SQL> SHUTDOWN ABORT
SQL> SHUTDOWN IMMEDIATE
SQL> SHUTDOWN TRANSACTIONAL
SQL> SHUTDOWN NORMAL
```

```SQL
SQL> startup nomount;
```
We want to stop at the nomount stage. This is because if we have a corrupted or missing control files. It would be at this point that we would use the statement restore control file in order to bring the backed up copy file and then you want a mount and eventually open the database.

Now we dont want to again restart the instance then how to do that, we will use DDL for that.
```SQL
SQL> alter database mount;
```
Now we are at mount stage and now we are able to run recover database, perform different kinds of recovery, and so on.

Now we will open the database using the DDL.
```SQL
SQL> alter database open;
```
Let's check the database is open, if sql gives no error it is open then.
```SQL
SQL> select user from dual;
SQL> select sysdate from dual;
```

**Dynamic Performance Views**
- These are v$ views
- These are virtual tables that record current database activity
    - System and Session parameters
    - Memory usage and allocation
    - File states (backup, diagnostic, etc)
    - Job/tasks execution stats
    - SQL execution stats
- DPVs are available for both standard and pluggable databases

**Viewing the Alert Logs**
```SQL
SQL> SELECT * FROM v$diag_info;
    # Look for diag trace path

SQL> exit

>>> adrci
    # you will get adrci prompt 
adrci> help

```

---

## Oracle Database Network Environment
- Configuring the Oracle Net Services
- Using tools to manage the Oracle network
- Installing the Oracle Client software

**Oracle Net Services**
- Called the Oracle Net and sometime SQL *Net
- TNS = Transparent Network Substrate
- Provides vendor and version neutral connectivity to Oracle databases
- TCP/IP is the de facto network communications protocol 

**Oracle Net Listener**
- Server-side process that accepts incoming client connection requests and forwards them to the database
- LREG background process manages the listener's activity
- Service name: Logical name for an Oracle database service
- Service ID (SID): Unique name if specific Oracle Database
- One SID > multiple service names

**Configuring LISTENER.ORA**
- The listener file, LISTENER.ORA is located by default in:
    - %ORACLE_HOME%\Network\Admin
    - Set %TNS_ADMIN% environment variable 
- The listener file contains:
    - Name of the listener
    - Protocol address(es) that the listener is accepting requests on
    - Database services

```
LISTENER = 
    (DESCRIPTION_LIST = 
        (DESCRIPTION =
            (ADDRESS = (PROTOCOL = TCP)(HOST = oraserver)(PORT = 1521))    
        )
    )
```

**Managing the listener**
- The default listener name is LISTENER, basic syntax is lsnrctrl command listenername
- Most common commands: stop, start, reload
- Security enhancements ideas:
    - Rename the listener
    - Password protect the listener
    - Configure alternate port for Oracle (default is TCP 1521)

```
>>> lsnrctrl status
>>> lsnrctrl
    # this will give the lsnrctrl prompt
    LSNRCTRL> help
```

**Oracle Naming Methods**
- TNSNAMES.ORA static configuration file
    - Located in %TNS_ADMIN% path
    - LISTENER.ORA is a server-side file; TNSNAMES.ORA is a client-side file
    - Easy Connect
        - Connect user/password@oracleserver/orcl
        - Don't put your password on the command line
    - Oracle Internet Directory (OID)

```
PSIGHT = 
    (DESCRIPTION =
        (ADDRESS_LIST =
            (ADDRESS = (PROTOCOL = TCP)(HOST = oraserver)(PORT = 1521))
        )
        (CONNECT_DATA = 
            (SERVICE_NAME = psight)
        )
    )
```

---

## Oracle Database Storage Structures
Create and manage tablespaces
- Single instance database
- Pluggable database

**What is Tablespace?**
- A tablespace is a logical storage container for segments 
    - Segments are database objects such as tables or indexes
    - Tablespace contains one or more datafiles that are physical db objects
    - A segment can't span multiple tablespaces
- SYSTEM and SYSAUX are crucial to the operation of the database
- For user tablespaces, take in to account availability, quotas, read/write vs read-only, security, etc.

```
                                   Permanent Tablespace | Temporary Tablespace
                                                        |
    +--------+ +--------+ +------+ +---------------+    |   +------+
    | SYSTEM | | SYSAUX | | UNDO | | Optional User |    |   | TEMP |
    +--------+ +--------+ +------+ | Tablespace    |    |   +------+
         |          |         |    +---------------+    |       |
Logical  |          |         |             |           |       |
---------|----------|---------|-------------|-----------|-------|------
Physical |          |         |             |           |       |
         V          V         V             V           |       V
        |__________________________________________|    |    Temp Files
                            |
                        Data files
```

- **SYSTEM:** It is where your data dictionary objects exits. All the objects in the SYSTEM are owned by the sys super user.
- **SYSAUX:** It is used as overflow space or auxiliary space for system objects.
- **UNDO:** When user deletes or modifies the data and didn't commit the changes that data are temporarily stored in undo tablespace in undo data files, so the other users querying that data will see the before image. So thats's giving us Read consistency. UNDO is also used by Oracle Flashback to be able to restore the data that has been deleted or modified.

> Note: when we are using container database we have one SYSTEM and one SYSAux that serves the entire CDB/PDB implementation.

```SQL
>>> sqlplus /nolog
SQL> connect system as sysdba
SQL> startup
SQL> show con_name
SQL> show pdbs;
SQL> alter session set container=<plugabble database name>;
SQL> startup
SQL> show con_name

# changing the port of database express
SQL> exec dbms_xdb_config.sethttpsport(5501);

# checking the port from dual table
SQL> select dbms_xdb_config.gethttpsport from dual;

SQL> select tablespace_name from dba_tablespaces;
```

**Initialization Parameter**
- ALTER SYSTEM to configure initialization parameters for an Oracle Database. SCOPE=SPLFILE, MEMORY, BOTH
- The CONTAINER clause is important for those who deploy container database (CDBs) with pluggable databases (PDBs). 
    - ALTER SYSTEM SET parameter=value CONTAINER=current/all
- We can use DPV's to quickly look up which initialization parameters are modifiable in a PDB.
    - SELECT name,value FROM v$system_parameter WHERE ispdb_modifiable='TRUE' ORDER BY name;

***Creating Tablespaces***
Using Express url: https://localhost:5500/em/login
1. Goto __Configuration__ -> Initialization parameter
2. Filter (search) file_dest, look for parameter db_create_file_dest
3. Select that line and click on Set (edit icon) to change the value of the parameter
4. Now got to __Storage__ -> tablespaces
5. To give some spaces to tablespaces select the tablespace and click on add __datafile__
6. If we want to create a new tablespace click on the __action__ (within the storage -> tablespaces) click on create specify the details.

> This is when want to do using dashboard

---

## Creating Common Database Objects in Oracle Database

- A schema is a collection of logical structures (tables, views, triggers, stored procedures, functions, etc) that are owned by particular Oracle user.
- Therefore in general schema = user
- Schemas have no direct relationship to tablespaces
- Schema objects use a dotted indentifier syntax. (schema.object)
- Schemas make it easier to manage related objects (Can give permission to entire schema with one SQL statement)

**Tables in Relational Database Model**
- Tables are stored in the data dictionary in UPPERCASE
- We can view tables using static DD views:
    - USER_TABLES
    - ALL_TABLES
    - DBA_TABLES

```SQL
SQL> SELECT COUNT(*) FROM DBA_TABLES;
SQL> CREATE TABLESPACE userdata2;
SQL> DESCRIBE v$datafile;
SQL> SELECT tablespace_name, file_name, bytes FROM dba_data_files;
```

---

### Oracle Database User Administration

Predefined roles and Accounts:
- SYS, SYSTEM, SYSMAN (root users)
- CREATE SESSION, CONNECT, DBA roles
- SYSDBA (special privilege)

```SQL
>>> sqlplus system as sysdba
SQL> show user;

SQL> connect system 
SQL> show user;

SQL> exit;

>>> sqlplus / as sysdba  -- logged in as current OS user
SQL> show user;
SQL> show con_name;
SQL> show pdbs;
SQL> alter session set container=pdbname;

-- create user
SQL> create user mdizhar identified by Password
    default tablespace tbs_name
    password expire;

-- new login 
>>> sqlplus mdizhar@pdbname
-- logon denied

SQL> grant create session to mdizhar;
SQL> alter session identified by Password;
>>> sqlplus /nolog
SQL> connect mdizhar@pdbname;

SQL> select count(*) from all_tables;
SQL> create table t1 (col1 number);

-- granting other privileges from sys user
SQL> grant resource to mdizhar;
SQL> grant unlimited tablespace to mdizhar;

>>> sqlplus /nolog
SQL> connect mdizhar@pdbname;
SQL> create table t1 (col1 number);
SQL> insert into t1(1);
SQL> insert into t1(2);
SQL> select * from t1;

```

---

## Managing Space and UNDO Data

What is UNDO data?
- Orace Database stores the before image of data in an undo segment in the designated undo tablespaces
    - Initialization parameters 
- UNDO serves many crucial purposes for us:
    - Supports ROLLBACK to undo changes that were made by uncommitted transactions
    - Data read consistency: ACID model
    - Flashback Operations

**ACID Model**
Transaction is a unit of work (single SQL statement, for instance).
- __*Atomic*__: A transaction should succeed or fail 100%
- __*Consistent*__: Data needs to stay in reliable state
- __*Isolated*__: Transaction should not be disturbed by the other session or transactions
- __*Durability*__: Changes should survive a system restart

**Elements of UNDO Space Management**
- AUM is enabled by default
- Verify AUTOEXTEND on UNDO tablespace
- Set UNDO Retention Period (the number of seconds the Oracle will keep data in the tablespace after that data is flushed)
- Use UNDO Advisor in DB Express

```SQL
SQL> show parameter undo;
```