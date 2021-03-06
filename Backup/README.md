HDP-Public-Utils/Backup
====

These scripts functions to backup:

* HDFS
* configurations in /etc
* Hive Metastore
* Oozie
* HUE database
* Ambari databases

They rely on HDFS snapshot, distcp toolsets, and the native database clients.

The scripts represents DEV and QA environments for testing backup/DR implementation. It is recommended to backup both raw and calculated data.

Backup methodology
----

The QA cluster is treated as the source and DEV as the target, with scripts run in both environments.

Oozie, Tivoli, or other enterprise schedulers can hook the scripts up and schedule them based on business rules and access patterns of Hadoop users.

### QA environment

For Backup & DR testing purposes, this environment is treated as the source cluster.

Requirements:

* All of the scripts that enables a directory to be snapshot, perform snapshots should be stored in /home/hdfs/snapshot_scripts directory.
* All the scripts need to be run by the hdfs user (commonly referred to as HDFS super user).  This is also true for the DEV environment. 

The QA scripts:

  * qa.allow_snapshot.sh:
    * This is a utility tool to enable snapshotting on multiple directory paths that needs to be backup. You run this once or as needed specially when you have growing directory paths to backup. It doesn’t hurt to run this once a week to avoid the risk of an accidental command that would disallow snapshot on the target directories.

  * qa.daily_snapshot.sh
    * This is a utility tool to capture snapshots based on the current date using the date of the form YYYY-MM-DD as the name of the snapshot. A sample snapshot looks like the one below:
      * `/datalake/raw/clickstream/.snapshot/2014-05-28/`

  * qa.backup_etc_dbs.sh
    * This script is responsible for backup of the entire /etc directory and databases for Hive, HUE, Ambari and Oozie. The entire /etc directory is backed up because it incurs very small overhead. This way both Hadoop and non-Hadoop configurations are protected.
    * Note: You have to backup each /etc directory from all nodes. You will have to update this script to uniquely identify each backup of the /etc. You can prefix the backup of /etc with the hostname plus date (yyyy-mm-dd) and load it to HDFS for snapshotting.

  * qa.etc_backup_dir_creator.sh
    * There are two new HDFS directories that only needs to be created once - /etc_backup for storing backups of the /etc directories of primary cluster, and /sql_dbs_backup for storing backups of Hive, HUE, Oozie and Ambari databases.


### DEV environment

For Backup & DR testing purposes, this environment is treated as the target cluster.

Requirements:

* All of the scripts should be stored in the local file system /home/hdfs/backup_scripts directory.
* All the scripts need to be run by the hdfs user (commonly referred to as HDFS super user).  This is also true for the QA environment. 

The DEV Scripts:

  * dev.allow_snapshot.sh & dev.daily_snapshot.sh
    * This serves the same purpose as the one from QA cluster. The reason why this is needed is to protect the data that was copied over for DR purposes. If one of the files/folders got corrupted or deleted accidentally, you have a snapshot to fallback to.

  * dev.execute_dbs_backup_dr.sh
    * This is a utility tool that will create a “pull” remote copy of the snapshot data from the QA cluster for backup and disaster recovery purposes. It uses distcp to perform the data copy. This runs the backup for the Hive, HUE, Oozie and Ambari database snapshots.

  * dev.execute_etc_backup_dr.sh
    * This is a utility tool that will create a “pull” remote copy of the snapshot data from the QA cluster for backup and disaster recovery purposes. It uses distcp to perform the data copy. This runs the backup for the local filesystem directory /etc snapshots.

  * dev.execute_master_data_backup_dr.sh
    * This is a utility tool that will create a “pull” remote copy of the snapshot data from the QA cluster for backup and disaster recovery purposes. It uses distcp to perform the data copy. This runs the backup for the target Hadoop data i.e. HDFS snapshots of /datalake/raw/clickstream.

  * dev.execute_sap_bw_backup_dr.sh
    * This is a utility tool that will create a “pull” remote copy of the snapshot data from the QA cluster for backup and disaster recovery purposes. It uses distcp to perform the data copy. This runs the backup for the HDFS snapshots of /datalake/archive/sap_bw.


Restoring Snapshot Data Into Operational HDFS
----

#### Snapshot Data & /etc configurations

Snapshot data can be restored by copying the snapshot data and providing the path to where you want that data stored.

For example if you have a file in /datalake/raw/clickstream/sample_file.tsv and somebody accidentally deleted sample_file.tsv, you can go back to the snapshot copy and put it back to hdfs by running:

      hadoop fs –cp /datalake/raw/clickstream/.snapshot/2014-05-28/sample_file.txt /datalake/raw/clickstream/

You can also use wildcards to copy multiple files/directories at once.

The same restore process can be applied for /etc.

#### HUE & Oozie Derby databases

HUE and Oozie Derby database restore is similar since you only have to copy the backup from HDFS and overwrite the ones from the local filesystem.

#### Hive database

For Hive which uses MySQL database (default), it is easier to restore to an existing database so running the command below will perform the restoration using a backup sql file.

      mysqlimport –u [username] – p [password] [database name] [backup.sql file]

#### Ambari database

For Ambari, which is Postgresql (by default) use the psql command-line to restore, for example:

      psql [databasename] < backup.sql file

If restoring from sratch (ie. the database doesn’t exist), run the command below to create the database and then load the backup file per the instructions above.

      createdb  -h [hostname of database] –U ambari  -e ambari
