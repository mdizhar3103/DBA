**Using SQL *Plus to create CDB**
Steps to create Database Creation File:
1. Create a Initialization file, for container database environement we must include the new parameter *ENABLE_PLUGGABLE_DATABASE* and set its value to true. Since, there is only a single CDB instance, so there is only one __spfile__
2. Create instance with __STARTUP NOMOUNT__ command.
3. Once instance is started, next create database with simlple command __`CREATE DATABASE`__, if this is going to be container database instance then we must include the new clause *ENABLE PLUGGABLE DATABASES* 
4. Create data dictionary CDB$ROOT, using oracle provided scripts 

***Review of SQL statement***
```SQL
create database condb user sys identified by oracle user system identified by oracle logfile group 1 ('/path') ENABLE PLUGGABLE DATABASE SEED FILE_NAME_CONVERT = ('/oradata/cdb1','/oradata/cdb1/seed') 
```

Multiple ways of specifying the seed file with order of precedence are as follows:
- FILE_NAME_CONVERT, specifying pair of path ('string1', 'string2', 'string3', 'string4')
- Oracle Managed Files
- PDB_FILE_NAME_CONVERT

The seed template may need the data file attributes on the SYSTEM ans SYSAUX tablespace to be different than they are defined on the root, you can specify by using *TABLESPACE_DATAFILE_CLAUSES*. The TABLESPACE_DATAFILE_CLAUSES can be used with any of the previous mentioned methods for specifying the names and location of the seed's datafiles.


For example if we set DB_CREATE_FILE_DEST, then our create database statement can adjust the size of the data files for these tablespaces by using a combination of tablespaces data file clauses to meet our requirements.

```SQL
SQL> DB_CREATE_FILE_DEST = /oradatacdb1
SQL> create database condb ....... ENABLE PLUGGABLE DATABASE SYSTEM DATAFILES size 100M AUTOEXTEND ON NEXT 10M MAXSIZE 2000M SYSAUX DATAFILES size 100M;
```

In above query we have specified SYSTEM DATAFILES size to 100M also given autoextend and for SYSAUX we specified the DATAFILES size to 100M but not given autoextend.

***USER_DATA TABLESPACE Clause***
```SQL
SQL> DB_CREATE_FILE_DEST = /oradatacdb1
SQL> create database condb ....... ENABLE PLUGGABLE DATABASE USER_DATA TABLESPACE usertbs DATAFILE '/path/usertbs01.dbf' size 50m;
```

Using the USER_DATA TABLESPACE clause will create a tablespace for storing user data and data options such as Oracle XML Database. In multiteant container database, the tablespace will is created as part of the seed template. Any pluggable database created using that seed template will include this tablespace and its datafile. The tablespace and datafile specified in this clause are not used by root (CDB$ROOT). 

*Demo*
```SQL
-- Before performing make sure your environment is correctly setup i.e. ORACLE_HOME

>>> echo $ORACLE_SID
    # in my case: condb

-- in my initialization file I have setup two parameters
>>> cat $ORACLE_HOME/dbs/initcondb.ora
    DB_NAME=condb
    ENABLE_PLUGGABLE_DATABASE=TRUE
    -- these are two mandatory parameters

-- create cdb creation sql script
>>> cat createcdb.sql
create database condb
user sys identified by oracle user system identified by oracle
logfile group 1 ('/oraredo1/condb/redo1_1.log', '/oraredo2/condb/redo1_2.log') size 100m,
        group 2 ('/oraredo1/condb/redo2_1.log', '/oraredo2/condb/redo2_2.log') size 100m,
character set al32utf8 national character set al16utf16
extent management local datafile '/oradata/condb/system_1.dbf' size 300m
sysaux datafile '/oradata/condb/sysaux_1.dbf' size 300m 
default temporary tablespace deftemp 
    tempfile '/oradata/condb/temp/deftemp_1.dbf' size 200m
undo tablespace undotbs
    datafile '/oradata/condb/undo01.dbf' size 200m
enable pluggable database
seed file_name_convert = ('/oradata/condb','/oradata/condb/seed')

-- The database creation will fail if the path specified are incorrect and it doesn't create on its own

>>> cd $ORACLE_HOME/dbs
>>> ls
-- We also want to setup password file, Once the password file is setup you can start the instance
>>> orapwd file=orapwcondb 
    # enter password 

>>> sqlplus / as sysdba

-- CREATE SPFILE FROM CURRENT PFILE P
SQL> create spfile from pfile;
SQL> startup nomount;
SQL> copy paste the createcdb.sql query to create database

-- once the above command is finished we need to run the catcdb.sql script, it will ask for new password for sys and system and a temporary tablespace for you to name  (temptbs_condb) and then you notice it will open in write mode container database and seed database in read only mode.

SQL> COLUMN NAME FORMAT A15;
SQL> SELECT NAME, CON_ID, OPEN_MODE, DBID FROM v$containers ORDER BY CON_ID;

SQL> SET LINESIZE 140
SQL> /

```

**Recommended Way of Creating CDB using DBCA**
    **DBCA (Database Configuring Assistant)**
Some recommendations:
- Create new default permanent tablespace
- Set default temporary tablespace for entire CDB
- If created PDB create new temporary tablespaces in PDBs (can't return)
- Create event triggers for opening PDBs

> Refer to README2.md for DBCA command installation

------------------------------------------------

**Sources to Create PDB**
- Using SEED template
- Clone existing PDB
- Plug in a PDB
- Move non-CDB in as PDB

Syntax for creating pluggable database is very similar to the creation of container database. The use of FILE_NAME_CONVERT syntax is the same in the pluggable database as in the container database. But it has another keyword PLUGGABLE

```SQL
create pluggable database finpdb
admin user finadm identified by oracle
default tablespace fintbs 
datafile '/oradata/condb/fin/find01.dbf' size 250M
file_name_convert = ('/oradata/condb/seed','/oradata/condb/fin/');
```

> PDB from SEED, copies seed files and is fastest and best strategy.

Things performed by PDB create statement are:
- Copies datafiles to target directory
- Creates SYSTEM and SYSAUX tablespaces
- Creates SYSTEM and SYS users
- Creates local user myplugadm
- Grants local PDB_DBA role to myplugadm
- Creates new default service for the PDB

*Demo*
```SQL
-- creating from seed database
SQL> column name format a60
SQL> select con_id, name from v$datafile order by con_id;
    -- copy pdseed path only directory path 

SQL> create pluggable database finpdb admin user finadm identified by oracle file_name_convert = ('/oradata/condb/pdseed/','/oradata/fin/');

-- once its created, lets see the mode and restriction
SQL> select name, open_mode, restricted, open_time from v$pdbs;

-- changing to read-write mode
SQL> alter pluggable database finpdb open read write;


-- another script to create pdb database with more options
SQL> create pluggable database findup 
     admin user findup identified by oracle1234
     roles=(dba)
     default tablespace finance
     datafile '/oradata/findup/finance.dbf' size 25M autoextend on
     file_name_convert=('/oradata/cdb1/pdbseed/', '/oradata/findup/')
     storage (maxsize 2g)
     path_prefix='/oradata/findup/';

```

**Clone PDB from PDB same CDB**
- Source plugged or unplugged 
- If plugged mode must be read-only mode
- User w/create pluggable database privilege
- Good for patch test

```SQL
SQL> select name from v$datafile order by con_id;

SQL> create pluggable database findup from finpdb file_name_convert = ('/oradata/fin','/oradata/findup/');

```

**Moving PDB (unplug or plug)**
This is done by creating a xml file.
- create xml file
- unplug from cdb
- move data and xml file
- plug into new cdb

```SQL
SQL> select name, open_mode, cdb, con_id, con_dbid from v$database;

SQL> select pdb_id, pdb_name, dbid, con_uid, con_id, status from cdb_pdbs;

SQL> @pdbopen

-- unplugging database, first close the pdb
SQL> alter pluggable database pdbname close immediate;

-- creating xml file, that describes database in datafiles
SQL> alter pluggable database pdbname unplug into '/oradata/pdb_name.xml';

SQL> @pdbopen

-- droppping pluggable database and keeping datafiles
SQL> drop pluggable database pdbname keep datafiles;

SQL> @pdbopen

-- Before moving to other cdb you should check the compatibility on other cdb
SQL> set serverouput on
SQL> declare
     compatible boolean := false;
     begin
     compatible ;= dbms_pdb.check_plug_compatibility(
         pdb_descr_file=>'/oradata/pdb_name.xml');
     if compatible then
     dbms_output.put_line('yes');
     else
     dbms_output.put_line('no');
     end if;
     end;
     /

-- if output is yes then perform the plugging
SQL> create pluggable database moved_pdbname using '/oradata/pdb_name.xml' nocopy tempfile reuse;

SQL> select name, open_mode, cdb, con_id, con_dbid from v$database;

-- checking the location of datafiles, it will be same as before unplugging
SQL> select name from v$datafile where con_id = 3;  -- CHECK YOU CON_ID

-- opening the database
SQL> alter pluggable database moved_pdbname open;

SQL> select pdb_id, pdb_name, dbid, con_uid, con_id, status from cdb_pdbs; -- check the status of moved_pdbname is normal
```

**Migrations Strategy**
Non-CDB to PDB:
- Convert to unpluggable PDB using DBMS_PDB package
- Move the files and plug into other cdb

