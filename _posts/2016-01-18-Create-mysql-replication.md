---
layout: post
title: "Create mysql replication"
date: 2016-01-18
---

Create mysql replication is a simple procedure that usually can be done with the following steps:

1. enable bin-log on your master

```
/etc/my.cnf
```

```
[mysqld]
# Replication
server-id = 1
relay-log = mysql-relay-bin
log-bin=mysql-bin
```

2. create replication user

```
mysql
mysql> CREATE USER 'repl'@'%.mydomain.com' IDENTIFIED BY 'slavepass';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%.mydomain.com';
```

3. lock your database and write master position

```
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SHOW MASTER STATUS;
```

4. take mysql dump of the database

```
mysqldump --all-databases --master-data > fulldb.dump
```

5. unlock the database

```
mysql> UNLOCK TABLES;
```

6. prepare mysql slave server

```
/etc/my.cnf
```

```
[mysqld]
server-id=2
relay-log = mysql-relay-bin
log-bin=mysql-bin
```

7. restore mysql data

```
mysql < fulldb.dump
```

8. start replication on the slave server with the change master command

```
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='master_host_name',
    ->     MASTER_USER='replication_user_name',
    ->     MASTER_PASSWORD='replication_password',
    ->     MASTER_LOG_FILE='recorded_log_file_name',
    ->     MASTER_LOG_POS=recorded_log_position;

mysql> START SLAVE;
```

but if you have very big database let say 1TB and you can’t except downtime?

If you prepare right you storage or you are using cloud services then you can lock the database for a few seconds take a snapshot and then copy the data from the snapshot.

if you didn’t prepare right mysql storage then you need to use the right flags in mysqldump command.

These are the flags that I used (relevant for transactional DB like InnoDB):

```
mysqldump --all-databases --master-data=2 --single-transaction --quick | gzip > outputfile.sql.gz
```

--all-databases - Used to backup all the databases in mysql server
--master-data=2 - Writes binary log name and position in mysql remark to the dump file
--single-trasaction - This is an important flag that send start trasaction to the mysql server and dump the consistent state of the database at the time when start transaction started. this flag let you use the database while the dump is running. The flag is usefull only for transactional tables like InnoDB.
--quick - Used for large tables to retrieve rows from a table one raw at a time instead of retrieving the entire row set and buffer it in memory before writing it.

To me the dump took about a day and then I restore it with the following command:

```
gunzip -c outputfile.sql.gz | mysql
```

The restore took me much longer, it was about 4-5 days. If you have other methods to make the dump or restore faster please let me know.

After the restore we need to run the change master command so we need to grub it from the dump file:

```
zcat all_db.sql.gz | head -n 200 | grep "CHANGE MASTER"
```

```
mysql
mysql> CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.xxxx', MASTER_LOG_POS=1111133333;
mysql> start slave;
```

To check the slave status use the following command:

```
mysql> SHOW SLAVE STATUS\G;
```

check that Slave_IO and Slave_SQL are running and wait for the Seconds_Behind_Master to decrease to 0 (to me it took ~4 days).

On the new slave server that I created I installed LVM with enough free space for snapshots so next time I can do the following:

1. lock mysql databases
2. flush the tables
3. get master binary log file and position
4. create LVM snapshots
5. unlock mysql databases
6. rsync the data to another server

These steps should take much less time then mysqldump and restore.

During this work I got help from the following links:

1. mysql docs – <http://dev.mysql.com/doc/refman/5.7/en/replication-howto.html>
2. mysql docs – <https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_quick>
3. server fault – <http://serverfault.com/questions/220322/how-to-setup-mysql-replication-with-minimal-downtime>
