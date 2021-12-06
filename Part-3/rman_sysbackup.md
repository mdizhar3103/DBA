### Connecting RMAN Using SYSBACKUP 

Administrative User Accounts:
- SYSBACKUP: RMAN operations
- SYSDG: Data guard operations
- SYSKM: Transparent data encryption key store operations

**SYSBACKUP Privileges/Administrator**
- 13 system privileges
- 1 role privilege
- Initially locked
- Password file: FORMAT=12:SYSBACKUP=Y
- Connection string "AS SYSBACKUP"

For SYSDBA:
- 219 system privileges
- 60 role privileges
- Initially locked
- Password file: FORMAT=legacy or 12
- Connection string as "AS SYSDBA"


**Setting Up SYSBACKUP**
```SQL
-- Unlocking User
-- check for orapw password file for user in ORACLE_HOME path, if file not present we cant connect remotely
SQL> select username, account_status from dba_users where username like 'SYS%';

SQL> select * from v$pwfile_users;

-- connecting locally to database using sysbackup
SQL> connect sysbackup/oracle as sysbackup
    -- connected
SQL> connect sysbackup/oracle 
    -- connecting like this will give error account is locked 

SQL> connect sysbackup/oracle@pdb_name
    -- error account is locked

-- connecting remotely to database using sysbackup will also give error account is locked
SQL> connect sysbackup/oracle@pdb_name as sysbackup

SQL> alter user sysbackup account unlock;

SQL> connect sysbackup/oracle
    -- gives error as should use AS SYSBACKUP
SQL> connect sysbackup/oracle@pdb_name
    -- connecting remotely will also gives error as should use AS SYSBACKUP

SQL> connect sysbackup/oracle@pdb_name as sysbackup
    -- still get error as logon denied
    -- Because that user needs to be in password file

-- Creating password file using bash command
>>> cd $ORACLE_HOME
>>> orapwd file=orapw_username sysbackup=y
    -- will ask to enter password for sys and sysbackup user

SQL> select * from v$pwfile_users;

SQL> connect system/oracle
SQL> select * from v$pwfile_users;

SQL> connect sysbackup/oracle@pdb_name as sysbackup;

-- comparing privileges between the two users
-- connecting  using sysdba
>>> sqlplus sys/oracle as sysdba
SQL> select * from session_privs;
    -- notice the number of rows (privileges)

-- connected at sysbackup
SQL> select * from session_privs;
    -- notice the number of rows (privileges)
SQL> select count(1), privilege from user_tab_privs group by privilege;
    -- check the same query at sysdba session connection

```

**Connection Using SYSBACKUP**
Authentication Method of DBAs
- Data Dictionary (account password)
- Operating System (OS) authentication
- Password Files
- Strong Authentication (OID - Oracle Internet Directory)

RMAN Connections:
- SYSDBA - default
- SYSBACKUP 
    - Using network connection password (FORMAT=12)
    - Using local OS authentication new OS group (ie. OSBACKUP)
    - Connection string must include "AS SYSBACKUP"
    - Command line RMAN command both single and double quotes 
        - rman target '"c##commonbkup@pdb_name as sysbackup"'


```SQL
>>> useradd -G OSBACKUP commonbkup
>>> login with same user
>>> check the group it belong to
>>> id

>>> sqlplus system/oracle@pdb_name
SQL> select * from v$pwfile_users;

>>> rman target c##commonbkup/oracle as sysbackup
    -- gives error
>>> rman target /
    -- gives error
>>> rman target '" / as sysbackup "'
    -- gives error
>>> rman target '"c##commonbkup/oracle as sysbackup"'
    -- connection succeeded

-- connecting using service name
>>> rman target '"c##commonbkup/oracle@svc_name as sysbackup"'
SQL> select upper(sys_context('userenv', 'curent_user'))||'@'||lower(sys_context('userenv', 'con_name')) from dual;

```

### Utilizing Storage Snapshot Optimization

**Backup with Third-party Snapshot Technologies**
Taking snapshot in earlier version requires database to be in __hot backup mode__ (ALTER DATABASE BEGIN BACKUP;). Using third party GUI or CLI to create snapshot, then perform the redologs (ALTER DATABASE END BACKUP; ALTER SYSTEM ARCHIVE LOG CURRENT;). Then writes the files to tape or virtual library.

**Recovery with Storage Snapshot Optimization**
```SQL
RMAN> RECOVER DATABASE UNTIL TIME '06/12/2021 18:00:00' SNAPSHOTTIME '06/12/2021 16:00:10';

```
