## Backup and Recovery using RMAN

- RMAN can backup the entire CDB including all PDBs.
- Connecting with RMAN at root level can give all capabilities to backup the entire database.

**Basic OPEN Backup (Non-CDB)**
- RMAN> BACKUP DATABASE PLUS ARCHIVELOG
    1. Performs ALTER SYSTEM ARCHIVE LOG CURRENT
    2. Backs up all current ARCHIVELOGS 
    3. Backups all datafiles (Backupset created for each PDB, Root(CDB), SEED)
    4. Create backup of Controlfile and spfile
    5. Performs ALTER SYSTEM ARCHIVE LOG CURRENT
    6. Backups all archive logs created during the backup

```SQL

>>> echo $ORACLE_SID
>>> rman target sys/oracle
RMAN> select c.con_id, c.name con_name, f.name form v$containers c, v$datafile f where c.con_id=f.con_id order by 1, 3;

-- backing up
RMAN> backup database plus archivelog;
RMAN> list backup;

```

**Backup CDB$ROOT**
Root backup commands:
```SQL
>>> rman target sys/oracle

RMAN> backup database; -- complete CDB backup
RMAN> backup database root; -- backup root only
RMAN> backup pluggable database 'CDB$ROOT'; -- same as above

```

**Backups of PDBs**
```SQL

>>> rman target sys/oracle@pdb_name
RMAN> backup database;
RMAN> list backup;

-- OR backing up multiple pluggable databases
>>> rman target sys/oracle@cdb_name
RMAN> select name from v$containers;
RMAN> backup pluggable database pdb1, pdb2;
RMAN> list backup;

```

**Backup PDB tablespaces and Datafiles**
```SQL

>>> rman target sys/oracle@pdb_name
RMAN> backup tablespace users;
RMAN> select name, files from v$datafile;   -- list datafiles
RMAN> backup datafile datafile_number;
-- OR
RMAN> backup datafile datafile_name;


-- Backing up tablespace from container database
>>> rman target sys/oracle@cdb_name
RMAN> backup tablespace pdb_name:tbs_name;
RMAN> backup datafile datafile_number;
-- OR
RMAN> backup datafile datafile_name;
-- the command for datafile backup is same because datafiles are unique
```

**Commands Not allowed from PDB**
- backup archived logs
- Delete archived log backups
- Restore archived logs
- Table recovery tables
- Running Data Recovery Advisor
- Register database
- Import catalog
- Delete archived logs
- Point-in-time recovery (PITR)
- CONFIGURE command
- Duplicate database
- Flashback operations
- Report/delete obselete
- Reset database


**Recovery in CDB environemnt:** *See cdb_recovery.md*

**Partial Recovery Using Flashback:** *See partial_recovery.md*

**Connecting RMAN using SYSBACKUP:** *See rman_sysbackup.md*

**Using Improvements of Multisection Backup:** *See multisection_backup.md*

**Database Duplication Using New RMAN Enhancements:** *See duplication.md*