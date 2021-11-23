### Management of Common and Local Users
- Common Users belong to Root Container Data Dictionary and are known to all pluggable databases that belong to CDB.
- Local User belong to specific PDB and can only connect to that PDB 

**Connection Supported**
- PDB/non-CDB compatibility
- SQL *Plus
- Easy connect syntax
- Service name
- OS authentication
- TWO_TASK
- ALTER SESSION SET CONTAINER


```SQL
-- Connecting 
>>> sqlplus system/oracle@localhost:1521/myplug
>>> sqlplus system/oracle               -- easy connect
>>> sqlplus system/oracle@cdb           -- service name
>>> sqlplus / as sysdba                 -- os authentication


-- where I am
SQL> select instance_name from v$instance;
SQL> select name from v$database;
SQL> show parameters db_name;
SQL> select sys_context('userenv', 'db_name') from dual;

-- The current container
SQL> select * from global_name;
SQL> select ora_database_name from dual;
SQL> select sys_context('userenv', 'service_name') from dual;
SQL> select sys_context('userenv', 'con_name') from dual;
SQL> show con_name;

```

**Creating Common User**
Note: Creating common user must prefix with ##
```SQL
SQL> create user izhar identified by Pa$$w0rd;  -- gives error 

SQL> create user c##izhar identified by Pa$$w0rd; -- user created

SQL> sqlplus / as sysdba   

>>> echo $ORACLE_SID
>>> echo $TWO_TASK
>>> export ORACLE_SID=sid_name
>>> export TWO_TASK=myplugdb

>>> sqlplus system/oracle
SQL> show con_name;
SQL> show pdbs;

SQL> alter session set container=pdb_name;
SQL> show con_name;

```

**Current Container**
- Used for data dictionary
- Only common can have CDB$ROOT
- Local and common can have PDB
- SQL with CONTAINER=ALL only in root
- Not all common can use CONTAINER=ALL

**Local User**
- Local data dictionary
- Connect to PDB
- No c## or C## username is allowed
- CONTAINER is set to CURRENT (CONTAINER=CURRENT)
- Not created in ROOT container

```SQL

SQL> create user testuser identified by Pa$$w0rd;
-- OR
SQL> create user identified by Pa$$w0rd default tablespace cdb_tbs quota unlimited on cdb_tbs container=current;

-- my demo
SQL> sqlplus / as sysdba
SQL> show con_name
SQL> show_pdbs
SQL> ALTER SESSION SET CONTAINER=pdb_name;
SQL> show con_name
SQL> create user mdizhar identified by Pa$$w0rd;
SQL> grant dba to mdizhar;
SQL> connect mdizhar/Pa$$w0rd 
-- If logon denied use this 
SQL> sqlplus mdizhar/Pa$$w0rd@localhost:1521/xepdb1

```

**Common User**
Common things to know about:
- You must be connected to root container
- And has been granted create user system privileges as a common privilege
- Prefix c## or C##
- Referenced objects in all containers
- Username should be unique
- Common user should not own objects within the pluggable databases

```SQL

SQL> show con_name
SQL> select name from v$containers;

-- creating common user
SQL> create user c##mdizhar identified by Pa$$w0rd;
SQL> grant create session to c##mdizhar;

SQL> connect c##mdizhar/Pa$$w0rd
SQL> show con_name      -- cdb$root

-- try connecting with pdb will give error 
SQL> connect c##mdizhar@pdbname

SQL> revoke create session from c##mdizhar;


-- giving permission to all containers
SQL> grant create session to c##mdizhar container=all;

-- now connecting again to pdbs
SQL> connect c##mdizhar@pdbname
SQL> show con_name

```

**Privileges**
- Commonly granted privilege is effective on the entire container database and every container within it
- For example: If we grant create session commonly to a user, then that user be able to log into every pluggable database within the container database. We can't eliminate this by revoking it locally


|Commonly Granted Privileges|Locally Granted Privileges|
|---------------------------|-------------------------|
|Every existing and future contianer| Only in container where granted |
|Granted by common to common| Granted by any user to any user/role/PUBLIC|
|Grant should be done at CDB$ROOT and wih CONTAINER=ALL clause| Limited to Current container, CONTAINER=CURRENT can be included in the grant statement, this is optional and default setting include that|
|Common privilege are never granted to public, you can include system and object privileges.| |


**Roles**
- Common Roles
    - Common belong to ROOT
    - Created by common user
    - Common user grant to any user
    - If common user contains local privileges apply only where granted
    - Lets have role C##_DEV_ROLE, GRANT CREATE SESSION TO C##_DEV_ROLE CONTAINER=ALL (now can connect to any database)
- Local Roles:
    - Local belong to specific PDB
    - Local Roles have no commonly granted privileges]
    - Granted to any user
    - Local grant is local


**Limiting Common Users**
Limiting Privileges:
```SQL
SQL> show con_name      -- cdb$root
SQL> create user c##mdizhar identified by Pa$$w0rd;
SQL> create role c##developer container=all;
SQL> grant create session to c##developer container=all;
SQL> grant c##developer to c##commonuser container=all;

SQL> show con_name      -- pdb connection
SQL> create user mdizhar identified by Pa$$w0rd;
SQL> grant dba to mdizhar;
SQL> connect c##commonuser/Pa$$w0rd@pdb_name
SQL> select count(1) from all_tables;
SQL> connect mdizhar/Pa$$w0rd@pdb_name
SQL> grant select any table to c##developer;
SQL> connect c##commonuser/Pa$$w0rd@pdb_name
SQL> select count(1) from all_tables;
    -- check whether the grant effects in other schema or not
    -- SQL> connect c##commonuser/Pa$$w0rd@schema_name
    -- SQL> select count(1) from all_tables;   

SQL> show con_name      -- connect to schema via cdb
SQL> create user mdizhar identified by Pa$$w0rd;
SQL> grant dba to mdizhar;
SQL> connect c##commonuser/Pa$$w0rd@schema_name
SQL> select count(1) from all_tables;

```