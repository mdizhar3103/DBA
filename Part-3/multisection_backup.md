### Multisection Backup
Datafile are split in multiple backup piece set with single session and mulitple process, each process managing single backup piece.
- Speeds up backup and recovery
- Reduces image copy creation time
- Reduces time for performing transportable tablespace
- Reduces completion time of active duplication
- Automatic comperssion and block change tracking

*Basic Command*
```SQL
RMAN> backup section size 300M database;
RMAN> backup section size 200m tablespace inventory;
RMAN> run {
    backup section size 2000m datafile 1,3,5;
    backup section size 3000m datafile 2,4;
}
RMAN> backup incremental level 1 section size 2000m database;
RMAN> backup as copy section size 1024m database;

```
> Notes: if section size > file size, then multisection is not used
> If section size < file size/256, then it will automatic increase
> Can vary section size
> Compatible 12.0 minimum for incremental level 1++
> Only used for datafiles
> FILESPERSET=1 (always) for multi-section incremental backups
> Need to watch parallelism

```SQL
-- checking how to utilize multisection backup
SQL> select multi_section, set_stamp, backup_type from v$backup_set;
SQL> select multi_section, set_stamp, backup_type from RC_BACKUP_SET;

```

**Setting up for Multisection backup**

```SQL
>>> rman target sys/oracle
RMAN> select user||':'||upper(sys_context('userenv','con_name') || '@' || sys_context('userenv', 'db_name')) global_name from dual;
RMAN> show all;
RMAN> configure device type disk parallelism 3 backupset;
RMAN> configure channel 1 device type disk format '/oradata/backup1/%U';
RMAN> configure channel 2 device type disk format '/oradata/backup2/%U';
RMAN> configure channel 3 device type disk format '/oradata/backup3/%U';
RMAN> show all;


-- Mulitsection Incremental Backup

SQL> select bytes/1026/1026/1026 GB, d.name, file_name, t.name tbs_name, bigfile from v$datafile d, v$tablespace t where t.name like '%BIG%' and d.TS#=t.TS#;
SQL> select owner, table_name, tablespace_name from dba_tables where tablespace_name like '%BIG%';

SQL> select segment_name, bytes/1024/1024/1024 MB from dba_segments where ownner='BFUSER';

>>> rman target sys/oracle
RMAN> select user||':'||upper(sys_context('userenv','con_name') || '@' || sys_context('userenv', 'db_name')) global_name from dual;
RMAN> run {
    backup tag pdb_namedb1_level0 incremental level 0
    section size 300M
    pluggable database pdb_name;
} 

-- on pdb connection to check the backups
SQL> select tag, s.set_count, p.piece#, multi_section, bytes/1024/1024 MB, p.elapsed_seconds from v$backup_set s, v$backup_piece p where s.set_count = p.set_count order by tag, set_count, piece#, handle 2 3 4;


-- performing incremental level 1 backup
RMAN> run {
    backup tag pdb_namedb1_level1 incremental level 1
    section size 300M
    pluggable database pdb_name;
} 2> 3> 4> 5> 6>

-- lets see what happened to incremental level 1 backup
-- on pdb connection to check the backups
SQL> select tag, s.set_count, p.piece#, multi_section, bytes/1024/1024 MB, p.elapsed_seconds from v$backup_set s, v$backup_piece p where s.set_count = p.set_count order by tag, set_count, piece#, handle 2 3 4;

```

**Multiscetion Image Copies**
```SQL
>>> rman target '"c##commonbackup/oracle as sysbackup"'
RMAN> select user||':'||upper(sys_context('userenv','con_name') || '@' || sys_context('userenv', 'db_name')) global_name from dual;

RMAN> backup as copy section size 500M pluggable database pdb_name;
RMAN> list copy;

```