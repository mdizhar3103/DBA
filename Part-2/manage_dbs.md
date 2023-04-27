### Managing CDBs and PDBs
__CDB Administrator:__
- Instance Performance
- Instance startup/shutdown
- Common Users
- Common objects
- Instance Security
- Instance backup/recovery

__PDB Administrator:__
- Management of data
- Management of local user
- Management of local security
- PDB startup/shutdown
- PDB backup/recovery
- Application objects and issues

***Connecting to Container***
```SQL

SQL> connect / as sysdba
SQL> conn c##user/pwd@localhost:1521/cdb1
SQL> connect user/pwd@inv
SQL> select name, pdb from dba_services;

-- switching between containers
SQL> alter session set container;

-- connection to database root is called database service

```

**Connections and Views**
```SQL
>>> sqlplus / as sysdba         -- default username connect to ROOT
SQL> show con_name;             -- CDB is instance running
SQL> !ps -ef | grep pmon

/*
Connecting instance without service name results in
Oracle not available.

>>> sqlplus / as sysdba                 --
    Connected to an idle instance

SQL> show con_name;
ERROR:
ORA-01034: ORACLE not available

*/

SQL> select name, network_name, pdb from cdb_services;

-- connecting to pluggable database using service name
SQL> connect sys/oracle@myplug as sysdba
SQL> show con_name;

SQL> connect / as sysdba
SQL> get cdb_services;      -- select name, network_name, pdb from cdb_services;
SQL> /
SQL> get vservices;         -- select name, network_name, pdb from v$services;
SQL> /

SQL> get cdb_pdb;       -- select pdb_name, status from cdb_pdbs;
SQL> /

-- listing open pdbs
SQL> @pdbopen.sql
SQL> select name, open_mode, restricted, open_time from v$pdbs;
SQL> @cdb_services;
SQL> /
SQL> @vservices;
SQL> /
SQL> alter pluggable database myplug unplug into '/oradata/myplug.xml';
-- lets see what  happened now
SQL> @vservices;
SQL> /

SQL> select name, open_mode, restricted, open_time from v$pdbs;
SQL> select pdb_name, status from cdb_pdbs;

-- Although the database is unplugged its status is still listed
--  because data dictionary knows about it.
-- To get rid of it is to drop the database

SQL> drop pluggable database myplug keep datafiles;
SQL> select pdb_name, status from cdb_pdbs;
SQL> select name, open_mode, restricted, open_time from v$pdbs;
SQL> select name, network_name, pdb from v$services;

```

**Current Container**
- Current container is where session is connected 
- Used for data dictionary
- User with correct privileges can *alter session set container* statement

**Administrative Tasks**
- Administrative tasks at ROOT:
    - Starting and shutting down CDB
    - Modifying root or CDB instance
    - Managing instance-level items, processes, memory, alerts, redo
    - PDB creation, unplugging, and droppping

*Startup CDB Instance Stages*
- SHUTDOWN
- NOMOUNT
    - All steps are at the instance level
    - Server parameter file read
    - SGA memory allocated
    - Background processes started
- MOUNT
    - Obtain file names from control file
    - Mount files (all containers should be mounted)
- OPEN
    - Opens online data files
    - Opens online redo log files
    - PDB$SEED read only
    - User PDBs mounted

```SQL

SQL> alter pluggable database pdb_name open;
SQL> alter pluggable database all open;
SQL> create trigger open_all_pdbs 
     after startup on database
     begin
     execute immediate 'alter pluggable database all open;'
     end open_all_pdbs;
     /

```

**PDB Modes**
- MOUNTED
    - Data can't be viewed or modified
    - Allow access to DBA
- OPEN READ ONLY
    - Allows only query against the pdb but no data manipulation statements 
- OPEN MIGRATE
    - Allows only database ungrade script ran within the database
- OPEN READ WRITE
    - Normal state, that allows users to run queries and transactions against the database

```SQL
-- Running the following commands within PDB
SQL> startup force;
SQL> startup open read write [restricted];
SQL> startup open read only [restricted];
SQL> startup upgrade;
SQL> shutdown [immediate];

--- You can start up a database in restricted mode.
SQL> shutdown immediate;
SQL> startup restrict;
SQL> select logins from v$instance;
    LOGINS
    ----------
    RESTRICTED
    
The good thing about this method is, that all users connected to the database will get disconnected when you shutdown the database. 
These users will not be able to connect unless they have the RESTRICTED SESSION privilege.


SQL> alter database open [read write] [read only] [upgrade];
SQL> alter database open read only;

-- Running the commands from CDB
SQL> alter pluggable database pdb_name open read write [restricted] [force];
SQL> alter pluggable database pdb_name open read only [restricted] [force];
SQL> alter pluggable database pdb_name open upgrade [restricted];
SQL> alter pluggable database pdb_name close [immediate];

-- limiting the pdbs
SQL> alter pluggable database pdb_name1, pdb_name2 close immediate;
SQL> alter pluggable database all except pdb_name open;

```

**Instance Parameters for CDB/PDB**
There is single spfile or initialization file for container instance. All pluggable databases inherit their values from that container unless they are specifically set within the pluggable database itself.

We can view the parameter by querying the v$parameter view. From within the pluggable database we can execute the alter system command to set the value. These values are not saved within the initialization parameter file but within the system tables of the pluggable database. When we make the change, if you can view the v$system_parameter view, we can see the change along with the container ID that pertains to the database that you changed in.

**PDB Administrative Tasks**
- Management of tablespaces
- Management of PDB datafiles and temp file
- Management of local schemas 
- Changing open mode of current PDB
- Specificaion of PDB storage parameters
- ALTER PLUGGABLE DATABASE
    - UNPLUG/OPEN/CLOSE
    - SET DEFAULT [temporary] tablespace
    - RENAME global_name
    - BACKUP/RECOVERY
    - DATAFILE clauses
    - PLUGGABLE storage clause

