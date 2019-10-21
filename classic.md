## **DBCS Configuration**

```
- sudo su - oracle
- update tnsnames and add cycpdb entry

connect sys/<password> as sysdba;
create user c##ggate identified by <password>;
grant dba to c##ggate;
alter session set container=cycpdb;
grant dba to c##ggate;

begin
dbms_goldengate_auth.grant_admin_privilege('C##GGATE',container=> 'all');
end;
/
 
connect sys/<password>@cycpdb as sysdba
create user cyc identified by <password>;
grant dba to cyc;

alter session set container=CDB$ROOT;
shutdown immediate;
startup mount
alter database archivelog;
alter database open;
ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION=TRUE SCOPE=BOTH;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE FORCE LOGGING;
ALTER SYSTEM SWITCH LOGFILE;
SELECT supplemental_log_data_min, force_logging FROM v$database;
```

## **ADW Configuration**

```
alter user ggadmin identified by <password> account unlock;

alter user ggadmin quota unlimited on data;

select * from v$parameter where name = 'enable_goldengate_replication';

alter system set enable_goldengate_replication = 'true' scope=both; -- this should not be needed

create user cyc identified by <password>;

grant dwrole to cyc;
grant unlimited tablespace to cyc;
```

## **Marketplace Image - Classic Configuration**

### **Set up connectivity on the image**
```
-- edit .bashrc and add:
export TNS_ADMIN=/u02/deployments/oracle18/network/admin
export LD_LIBRARY_PATH=/u02/deployments/oracle18/network/admin
export ORACLE_HOME=/u01/app/client/oracle18

source .bashrc

-- create network/admin directories
mkdir /u02/deployments/oracle18/network/admin

-- copy wallet to deployment directory
scp -i id_rsa /tmp/Wallet_DB201910111651.zip opc@172.16.2.2:/u02/deployments/oracle18/network/admin

unzip Wallet_DB201910111651.zip

-- update sqlnet.ora and add wallet location (ie: /u02/deployments/oracle18/network/admin)

-- update tnsnames.ora in /u02/deployments/oracle18/network/admin and add DBCS entries

db201910111651_high = (description= (address=(protocol=tcps)(port=1522)(host=adb.us-ashburn-1.oraclecloud.com))(connect_data=(service_name=kitovurrdt3i97n_db201910111651_high.adwc.oraclecloud.com))(security=(ssl_server_cert_dn=
        "CN=adwc.uscom-east-1.oraclecloud.com,OU=Oracle BMCS US,O=Oracle Corporation,L=Redwood City,ST=California,C=US"))   )

db201910111651_low = (description= (address=(protocol=tcps)(port=1522)(host=adb.us-ashburn-1.oraclecloud.com))(connect_data=(service_name=kitovurrdt3i97n_db201910111651_low.adwc.oraclecloud.com))(security=(ssl_server_cert_dn=
        "CN=adwc.uscom-east-1.oraclecloud.com,OU=Oracle BMCS US,O=Oracle Corporation,L=Redwood City,ST=California,C=US"))   )

db201910111651_medium = (description= (address=(protocol=tcps)(port=1522)(host=adb.us-ashburn-1.oraclecloud.com))(connect_data=(service_name=kitovurrdt3i97n_db201910111651_medium.adwc.oraclecloud.com))(security=(ssl_server_cert_dn=
        "CN=adwc.uscom-east-1.oraclecloud.com,OU=Oracle BMCS US,O=Oracle Corporation,L=Redwood City,ST=California,C=US"))   )
        
DB1011_IAD1HP =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = cycdbcs.cycsubnet1.cycpvtvcn.oraclevcn.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = DB1011_iad1hp.cycsubnet1.cycpvtvcn.oraclevcn.com)
    )
  )
  
cycpdb =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = cycdbcs.cycsubnet1.cycpvtvcn.oraclevcn.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cycpdb.cycsubnet1.cycpvtvcn.oraclevcn.com)
    )
  )

-- copy sqlnet.ora and tnsnames.ora to ora client directory
cp sqlnet.ora /u01/app/client/oracle18/network/admin
cp tnsnames.ora /u01/app/client/oracle18/network/admin
```

### **config credentials**

```
cd /usr/local/bin
./ggsci oracle18

-- add credentialstore
alter credentialstore add user c##ggate@DB1011_IAD1HP password <password> alias dbcs domain OracleGoldenGate -- CDB connection
alter credentialstore add user ggadmin@db201910111651_low password <password> alias adw domain OracleGoldenGate

-- confirm connectivity
dblogin useridalias dbcs
dblogin useridalias adw

-- note you can also connect directly without using the alias, and will need to use this when connecting to the pdb instead of CDB above (dblogin userid ggadmin@db201910111651_low password <password>)
```

### **config source**

```
-- use integrated extract

dblogin useridalias dbcs

register extract ext1 database container cycpdb
Extract EXT1 successfully registered with database at SCN 2809527
add extract ext1, integrated tranlog, begin now
add exttrail ./dirdat/aa, extract ext1, megabytes 50

dblogin userid cyc@cycpdb, password <password> -- connect directly to PDB instead of CDB
Successfully logged into database CYCPDB.

GGSCI (ogg19cora as cyc@DB1011/CYCPDB) 5> add schematrandata cycpdb.cyc preparecsn allcols

2019-10-14 21:28:37  INFO    OGG-01788  SCHEMATRANDATA has been added on schema "cyc".

2019-10-14 21:28:37  INFO    OGG-01976  SCHEMATRANDATA for scheduling columns has been added on schema "cyc".

2019-10-14 21:28:37  INFO    OGG-01977  SCHEMATRANDATA for all columns has been added on schema "cyc".

2019-10-14 21:28:37  INFO    OGG-10154  Schema level PREPARECSN set to mode NOWAIT on schema "cyc".

2019-10-14 21:28:40  INFO    OGG-10471  ***** Oracle Goldengate support information on table CYC.TEST ***** 
Oracle Goldengate support native capture on table CYC.TEST.
Oracle Goldengate marked following column as key columns on table CYC.TEST: COLA
No unique key is defined for table CYC.TEST.
```
Note this blog on logging:  https://www.oracle-scn.com/oracle-goldengate-integration-with-datapump-dboptions-enable_instantiation_filtering/

### **config target**

Note documentation: Currently, only non-integrated Replicats are supported with Oracle Autonomous Data Warehouse Cloud. Integrated Replicat, parallel Replicat, and coordindated Replicat are not supported.

```
-- create GLOBALS config file
vi /u02/deployments/oracle18/GLOBALS
ggschema ggadmin
checkpointtable ggadmin.checkpointtable

-- config 
dblogin useridalias adw
add checkpointtable ggadmin.checkpointtable
add replicat REP1, exttrail ./dirdat/aa, checkpointtable ggadmin.checkpointtable
```

### **create parameter files and start processes**

```
-- ext1.prm - create in /home/opc/oracle18/dirprm 
extract ext1
setenv (ORACLE_HOME='/u01/app/client/oracle18')
useridalias dbcs
exttrail ./dirdat/aa
ddl include mapped
table cycpdb.cyc.*

-- rep1.prm
replicat rep1
setenv (ORACLE_HOME='/u01/app/client/oracle18')
useridalias adw
dboptions ENABLE_INSTANTIATION_FILTERING
ddl include mapped
map cycpdb.cyc.*, target cyc.*;

start ext1
start rep1
info all
stats ext1
stats rep1

```

**Post configuration steps:**

1. Start ext1

2  Export data from source using expdp (or use RMAN).  Ensure rep1 is not running.  GG should internally note the SCN, but you can also issue:
select current_scn from V$database;

3. Import data into target.

4.  Start rep1.  GG should pick up changes from the point of the export.  You can also issue:
start replicat , aftercsn