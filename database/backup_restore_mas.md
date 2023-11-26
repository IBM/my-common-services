# Informal steps on how to backup and restore DB2 database 
## For Maximo Application Suite (Maximo Manage) using publicly accessible Ansible Roles (MASDevOps).

The following write-up is informally written and includes example keys and information. The steps have been validated by multiple individuals; however, I cannot take responsibility for any changes or issues that you may encounter and not be able to resolve independently.

## Source System:
### Manage Base installed with Demo Data & Demo Users + Spatial add-on.

Step 1. Locate and identify the default, out-of-the-box content of the secret in Manage project: `internal-maximoproperty-secret`. For example:

```
mxe.name=MXServer
mxe.db.url=jdbc:db2://c-db2w-manage-db2u-engn-svc.db2u.svc:50001/BLUDB:sslConnection=true;sslVersion=TLSv1.2;
mxe.db.driver=com.ibm.db2.jcc.DB2Driver
mxe.db.user=maximo
mxe.db.password=passw0rd
mxe.db.schemaowner=maximo
mxe.security.crypto.key=zLnnKvBMkXOnmWiZhMLTwwjS
mxe.security.cryptox.key=MPYvKyOcYRmqfQqelrnfTcSW
mxe.security.old.crypto.key=zLnnKvBMkXOnmWiZhMLTwwjS
mxe.security.old.cryptox.key=MPYvKyOcYRmqfQqelrnfTcSW
```

Step 2. Backup database using Ansible Role. 

```
export DB2_BACKUP_DIR=/scripts/newdbbackup
export DB2_BACKUP_INSTANCE_NAME=db2w-manage
export DB2_NAMESPACE=db2u
ROLE_NAME=db2_backup ansible-playbook ibm.mas_devops.run_role
```

Note 1:  Your DB2_BACKUP_DIR directory is whatever local computer directory you mapped to run Ansible via the mascli docker/podman container. In above mentioned case, the directory which is mapped via docker is called scripts; however, at the local computer level, it could be any directory that contains the directory newdbbackup. It is at this directory level, we will backup the database files.

Note 2: Your DB2_BACKUP_INSTANCE_NAME is whatever your DB2 database instance name. If you used a shared database, it is likely called db2w-shared. If you used separate database, it is likely called db2w-manage. Check your DB2 pods to locate the name.

Note 3: This Ansible Role, if run for the first time, will take few minutes to download the backed up database file.  It will produce the following files. The second run of this Role may require you to delete the *.kdb file from the container. 

```
BLUDB.0.db2inst1.DBPART000.20231030183902.001
keystore.p12
keystore.sth
master_key_label.kdb
```

Note 4: Now, your database of Manage is backed up. Please note that this backed up database does not contain your pre-installed Spatial add-on application. It also does not contain your users.

## Destination system: 
### Manage Base with NO demo data or users, and no add-on

In my testing, I installed Manage Base without demo data. This setup only provides maxadmin user. Later I learned that I should have installed Spatial add-on, to match my Source System because restoring the database does not install any add-on. The restoration of the database only restores the data.

Step 1. Log in to your destination cluster by mapping the source folder (the folder which contains your earlier backed up database files) and run the Ansible Role to restore.

```
export DB2_RESTORE_DIR=/scripts/newdbbackup
export DB2_RESTORE_INSTANCE_NAME=db2w-manage
export DB2_NAMESPACE=db2u
ROLE_NAME=db2_restore ansible-playbook ibm.mas_devops.run_role
```

Note 1: After you run the Ansible role to restore, you will encounter the white-screen problem. 

### Solution for white-screen problem:

Step 2: In Manage UI (under Database), create key/value pairs

(example keys from the destination system)

```
MXE_SECURITY_OLD_CRYPTOX_KEY
XMqzxFaonECrFFslTzOjAVQb

MXE_SECURITY_OLD_CRYPTO_KEY
JNcQXPqILpOkbAmlLRtuLkwt

(keys from the source system)
MXE_SECURITY_CRYPTO_KEY
abCDefGHijKLmnOPqrSTuvWX

MXE_SECURITY_CRYPTOX_KEY
xwVUtsRQpoNMlkJIhgFEdcAB
```

Note 2: It will take several minutes but you will eventually see errors. This still does not solve the problem.

Step 3. Reset the keys by following the script from this document. https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=encryption-resetting-crypto-cryptox-fields-in-database

This will solve the problem and your database from the source system will be restored to the destination system. It will take several hours for this entire process to complete successfully.

References:

MASDevOps Backup Role: https://github.com/ibm-mas/ansible-devops/tree/master/ibm/mas_devops/roles/db2_backup

MASDevOps Restore Role: https://github.com/ibm-mas/ansible-devops/tree/master/ibm/mas_devops/roles/db2_restore
