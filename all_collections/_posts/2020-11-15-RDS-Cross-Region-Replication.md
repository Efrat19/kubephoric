---
layout: post
title: RDS Cross Region Replication
lang: en
categories:
    - AWS
    - RDS
    - MYSQL
    - DR
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---
 
In this post I will walk you through the steps required to define a manual RDS Mysql 5.8 replication across 2 AWS regions, in my case - Ireland (eu-west-1) and Frankfurt (eu-central-1)

## Pre-steps
- **VPC peering:**
  make sure the source region and destination region VPCs are connected via [AWS VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- **Security:**
  make sure the source DB security group allows inbound connection from the destination DB CIDR.

## On the master instance:
By default, RDS binlog is not saved for long, so I would like to increase that parameter:
```cmd
CALL mysql.rds_set_configuration('binlog retention hours', 72);
```
next - still in the master instance, I want to create a replication user, and grant it replication permissions:
The user will be called `repl_user` and will be allowed to access the Database from the destination DB CIDR - which is `10.190.0.0/16`
```cmd
CREATE USER 'repl_user'@'10.190.%' IDENTIFIED BY 'password';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'repl_user'@'10.190.%' IDENTIFIED BY 'password';
```
once done I log out of the master and proceed to the snapshot part.

## Creating the Snapshot
1. create snapshot
```cmd
~$ aws rds create-db-snapshot \
 --db-snapshot-identifier crr-example \
 --db-instance-identifier crr-example-db
```
copying snapshots between regions requires a KMS key, lets create one:

```cmd
~$ aws kms create-key --region eu-central-1
```

now we can copy the snapshot to the destination region:
> You can copy a snapshot from one AWS Region to another. In that case, the AWS Region where you call the CopyDBSnapshot action is the destination AWS Region for the DB snapshot copy.

```cmd
~$ aws rds copy-db-snapshot \
 --source-db-snapshot-identifier arn:aws:rds:eu-west-1:123000000321:snapshot:crr-example \
 --target-db-snapshot-identifier crr-example-dr \
 --region eu-central-1 --source-region eu-west-1 \
 --kms-key-id arn:aws:kms:eu-central-1:123000000321:key/xxxxxx-xxxx-xxxxx-xxxxxxxxxx
```

and the final step is restoring the snapshot back to a living DB:
```cmd
aws rds restore-db-instance-from-db-snapshot \
 --db-instance-identifier crr-example-dr \
 --db-snapshot-identifier crr-example-dr --db-instance-class db.t3.xlarge  \
 --db-subnet-group-name xxx  --no-publicly-accessible \
 --multi-az --region eu-central-1
```

## Creating the Replication

Now we would like to search for the binlog position the snapshot was created from. In the RDS UI, go to `Logs & Events`. Go down to `Logs` and view them. search for this line:
```cmd
2020-11-12T11:40:01.214916Z 0 [Note] InnoDB: Last MySQL binlog file position 0 50567, file name mysql-bin-changelog.130453
```
the binlog file name and position will be used on the next command - setting the master, the procedure is defined as
```cmd
CALL mysql.rds_set_external_master (
  host_name, 
  host_port, 
  replication_user_name, 
  replication_user_password, 
  mysql_binary_log_file_name, 
  mysql_binary_log_file_location, 
  ssl_encryption);
``` 
so I log into the newly-created database, and run the command
```cmd
CALL mysql.rds_set_external_master (
    "master-db.fg34few.eu-west-1.rds.amazonaws.com"
  , 3306
  , "repl_user"
  , "password"
  , "mysql-bin-changelog.130453"
  , 50567
  , 1
);

```
and than I start it:
```cmd
CALL mysql.rds_start_replication;
```

now the replication should start running. to check its status, run 

```cmd
MySQL [(none)]> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master-db.fg34few.eu-west-1.rds.amazonaws.com
                  Master_User: repl_user
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-changelog.131097
          Read_Master_Log_Pos: 9694902
               Relay_Log_File: relaylog.000139
                Relay_Log_Pos: 7953529
        Relay_Master_Log_File: mysql-bin-changelog.130889
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: mysql.plugin,mysql.rds_monitor,mysql.rds_sysinfo,innodb_memcache.cache_policies,mysql.rds_history,innodb_memcache.config_options,mysql.rds_configuration,mysql.rds_replication_status
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 7953296
              Relay_Log_Space: 1888052202
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: Yes
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 62184
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 380916590
                  Master_UUID: e16fdcb8-c1b6-11e9-a1cb-063e749af6f4
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: System lock
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.002 sec)
```
Note the `Seconds_Behind_Master`, it should decrease within the upcoming hours. If `Seconds_Behind_Master = NULL` it usually means something went wrong, check for errors in the replica log.