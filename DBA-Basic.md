## DBA 

```bash
>>> sqlplus / as sysdba
SQL> startup 

# get control files and spfiles
SQL> select name, value from v$parameter where name like '%spf%' or name like 'co%es';

# get datafiles
SQL> select name from v$datafile;

# get redo logs
SQL> select member from v$logfile;

```

|Database| Instance |
|--------|----------|
|Resides on disk | Memory structures and the background processes |
| comprises of datafiles and redo logs | Requried to mount and open the database |

### Stages of Database Startup 
|Stage | Files | Functions |
|------|-------|-----------|
|nomount| spfile | builds the instance |
| mount | control files | mounts the database |
|open | data files + redo logs | opens the database |

```bash
# moving the ora file, control file, data file to test the database
>>> mv $ORACLE_HOME/....../*.ora <new_path>
>>> mv $ORACLE_HOME/....../control01.ctl <new_path>
>>> mv $ORACLE_HOME/....../system01.dbf <new_path>

SQL> shutdown immediate
SQL> conn / as sysdba       -- connect to an idle instance
SQL> startup
    # if fails due to couldn't open parameter file *.ora
    # restore the ora file to previous location
SQL> startup
    # gives error since control file is missing but instance is started
    # restore control file back to original location
SQL> startup force
# OR
SQL> alter database mount;
SQL> alter database open;
    # gives error because datafile is missing
    # restore the datafile back
SQL> alter database open; -- startup force
```

### Automatic Diagnostic Repository
```bash
SQL> ! ls $ORACLE_BASE/diag/rdms/../..
SQL> select name, value from v$diag_info;
SQL> select name, value from v$diag_info where name like 'ADR%';

# bash prompt
>>> ardci
ardci> help
ardci> set base <path>
ardci> help show alert

```

