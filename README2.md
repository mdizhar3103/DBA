# Oracle Database Installation
Verifying System Requirements
```bash
# Windows CMD prompt
>>> systeminfo | more

# Linux based OS
>>> uname -a
>>> cat /etc/os-release
>>> df -h
>>> free -m
>>> cat /proc/meminfo
>>> cat /proc/cpuinfo
>>> /sbin/sysctl -a
>>> netstat -tlnup
>>> top
>>> ps -aux
```

**Automatic Storage Management (ASM)**
- Filesystem and logical volume manager for Oracle database files
- A disk group is a logical collection of disks that Oracle manages as a unit 
- Data is striped to accomplish load-balancing and reduced I/O latency 
- Normal redundancy: two-way mirroring 
- High redundancy: three-way mirroring
- External redundancy: can work with third-party RAID system

*ASM Installation*
- Prepare your disks (they must be in raw format)
- Choose the Grid Infrastructure media
- Run the installer and create a disk group 
- Point a new database to disk group (modify initialization parameters as necessary)

```bash
If already installed oracle db:
>>> set ORACLE_SID=+ASM       (the default name)
>>> set ORACLE_HOME=<full_path>
>>> asmcmd
    > help
    > ls
    > exit
```

---

## Installing Oracle Database in Windows

Pre-installatoin Requirements: refer to oracle documentation.
- Create the necessary user accounts
    - Admin to own the installation
    - Non-admin service accounts
- Create Oracle Directories
- Chek TCP/IP configuration (DNS, HOSTS, pre-TNS)

*Run the windows cmd as admin*
```bash
>>> whoami
>>> compmgmt.msc
    # this will open the window
    # create new user in local users and groups for service account
    # username: oraservice
    # also check user can't change password and never expires.

>>> now install the oracle database setup
    - installation dialog opens
    - select skip software updates (then click next)
    - install database software only (then click next)
    - Select Single Instance database (then click next)
    - Select Enterprise Edition (then click next)
    - Use Existing Windows User, enter the oraservice username that we created. (then click next)
    - specify the location carefully (then click next)
    - Save the response after installation is completed (then click next)
    - Wait to finish further installation 
```

Post-installatoin Tasks
- Check out the services
- Check out the environment variables (ORACLE_HOME, bin, etc)
- Check the filesystem (ORACLE_BASE)
- Check the Start Screen/Start menu
- Check the Oracle alert log

Verfiying the Oracle installation
- open resgistry editor regedit (Oracle directory -> Key...Home) Look for ORACLE_HOME
- Look for icons in windows startup menu

---

## Installing Oracle Database on Linux

[Link](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/installing-oracle-linux-with-public-yum-repository-support.html#GUID-190BAEE2-2B77-4AA2-AA6B-5D6AF73A4005)

[HelpLink](https://yum.oracle.com/getting-started.html)


```bash
>>> cat /etc/redhat-release
>>> uname -a
>>> rpm -q binutils compat-libstdc++ gcc glibc libaio libgcc libstdc++ make sysstat unixodbc
>>> ping goo.gl
>>> yum install unixODBC sysstat libstdc++ libstdc++-devel -y

>>> curl -o oracle-database-preinstall-19c-1.0-2.el8.x86_64.rpm https://yum.oracle.com/repo/OracleLinux/OL8/appstream/x86_64/getPackage/oracle-database-preinstall-19c-1.0-2.el8.x86_64.rpm
>>> yum -y localinstall oracle-database-preinstall-19c-1.0-2.el8.x86_64.rpm
>>> rm -rf oracle-database-preinstall-19c-1.0-2.el8.x86_64.rpm

>>> grep "oinstall" /etc/group
>>> id oracle
>>> passwd oracle 
    # password: oracle

>>> mkdir -p /u01/app/oracle/product/19.1.0/db_1
>>> chown -R oracle:oinstall /u01 
>>> chmod -R 775 /u01 

# install the not installed packages from above command
```
> Note: preinstall does the following: [doc](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/Chunk239081377.html#GUID-C15A642B-534D-4E4A-BDE8-6DC7772AA9C8)

```bash

>>> Now download the database zipfile from oracle website (https://www.oracle.com/database/technologies/oracle-database-software-downloads.html)

>>> wget https://download.oracle.com/otn/linux/oracle19c/190000/LINUX.X64_193000_db_home.zip?AuthParam=1635508436_d3413a6d508f7552c4eb08da99b63cbb

>>>  mv 'LINUX.X64_193000_db_home.zip?AuthParam=1635508436_d3413a6d508f7552c4eb08da99b63cbb' LINUX.X64_193000_db_home.zip

>>> cd /u01/app/oracle/product/19.3.0/db_1
>>> unzip /software_location

>>> ls -l
>>> vim ~/.bash_profile

# add the following in the file:
# oracle setting
export TMP=/tmp
export TMPDIR=$TMP

# make sure entry in hosts file with same HOSTNAME
export ORACLE_HOSTNAME=oralin.localdomain
export ORACLE_UNQNAME=orcl
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/db_1
export ORACLE_SID=orcl

export PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib

>>> hostnamectl set-hostname oralin
>>> passwd oracle
    # change the password

>>> login with oracle user
>>> cd /u01/app/oracle/product/19.3.0/db_1
>>> ./runInstaller

# incase of os error change the parameter in
>>> cd /u01/app/oracle/product/19.3.0/db_1/cv/admin
>>> vim cvg_config
    # uncomment the parameter and change value to specific os (cat /etc/os-release)
    CV_ASSUME_DISTID=OEL8
    https://oracle-base.com/misc/comments?page_id=1976
```

**Post Installation**
```bash
>>> vim /etc/oratab
    # service autostart file, set flag to Y if set to N

>>> lsnrctl status
>>> lsnrctl start
>>> sqlplus / as sysdba

SQL> startup;
SQL> select dbms_xdb_config.gethttpsport from dual;
SQL> exit;
>>> netca;
```

---

### Creating Database in Oracle

```bash
>>> dbca
    # Note: to run this command make sure your are connected via putty or Mobaxterm or related software
    # opens up the GUI for creating a database
    1. select create database 
    2. Now if you want to use the normal setup get the details filled and if you want advance setup select the advance option
    3. select advance option, then click next
    4. view details of specific options
    5. In my case, I selected General Purpose
    6. give Global database name and pdb name:
        global name: condb
        pdb name: plugdb
        then click next 
    7. use template file options, then click next 
    8. leave default for fast recovery area, click next
    9. choose the listener or leave as default
    10. data vault option, leave as default
    11. configuration options check wisely, in sample schema check the box, then click next 
    12. management options, choose database express option
    13. user credential options, I selected common for all with password: `oracle`
    14. In creation option check all initialization parameter, and modify if you want. Also check the options if you want database creation script save click that and scpecify the destination path and also if you want database template check that also.
    15. wait till process if finished

```
**About Multe-Tenant Database**
- CDBs and PDBs function the same way that non-CDBs do
- PDBs are modular and can be unplugged from one CDB and plugged into another CDB
- One CDB can have upto 253 PDBs, with limit of 512 services wintin a CDB
- Control files and Redo log file belong to the container as do the UNDO and default TEMP tablespace
- CDB dictionary views provide visibility across all PDBs 
- PDBs contain application and local temporary tablespaces, local users and roles


