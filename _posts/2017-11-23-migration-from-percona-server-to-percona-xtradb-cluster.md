---
layout: post
title: Migration from Percona Server 5.6 to Percona Xtradb Cluster
author: Nikita Shalnov
tags:
- howto
- english version
draft: true
---

Today I want to tell you how to migrate from MySQL Percona Server 5.6 to Percona Xtradb Cluster.

My initial configuration was:
- standard mysql master-slave replication
- node1 is a master
- node2 is a slave

Now I need to setup master-master replication for failover purposes. But before all we need to add one more server in our configuration. I choose a scheme where only two mysql-servers act as primary and the third acts as an arbitrator and doesn't store any data, because percona has synchronous replication (what means a data is not written at all while it is not written on ALL nodes in cluster) and the latency of a cluster is equal to latency of the slowest node in the cluster, I don't want add one more point of possible latency and don't need to store a third replica of a data. So on the third node I will just install and configure Galera Arbitrator. I hope you understand purposes, why you should have minimum 3 members in cluster (hint: avoiding split-brain).

Our action plan consists of these parts:
1. Update a slave to xtradb node
2. Setup xtradb cluster on the slave node continuing to receive any updates from master
3. Configure Galera Arbitrator on a third node (node3) and join it in cluster with node2
4. Shutdown an application targeting on master mysql node
5. Monitor a replication lag, wait until it disappears.
6. Stop and reset slave node (node2)
7. Point the application to node2
8. Update the former master to xtradb node
9. Join node1 into cluster

Let's begin.

Before all works please shure that your replication works fine and the data really replicates. You can use command:

```bash
mysql -u root -e 'show slave status\G;'
```

1. Update a slave to xtradb node

```bash
systemctl stop mysql
apt-get remove percona-server-server-5.6 percona-server-client-5.6
apt-get install percona-xtradb-cluster-server-5.6
```

After these steps mysql database should normal start. If you have some similar output and mysql doesn't start:

```
Building dependency tree...
Reading state information...
percona-xtradb-cluster-server-5.6 is already the newest version.
The following packages were automatically installed and are no longer required:
  libhtml-template-perl libio-socket-ssl-perl libnet-libidn-perl
  libnet-ssleay-perl libperconaserverclient18.1 libterm-readkey-perl
  percona-server-common-5.6
Use 'apt-get autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 22 not upgraded.
1 not fully installed or removed.
After this operation, 0 B of additional disk space will be used.
Setting up percona-xtradb-cluster-server-5.6 (5.6.37-26.21-3.jessie) ...


 * Percona XtraDB Cluster is distributed with several useful UDF (User Defined Function) from Percona Toolkit.
 * Run the following commands to create these functions:

        mysql -e "CREATE FUNCTION fnv1a_64 RETURNS INTEGER SONAME 'libfnv1a_udf.so'"
        mysql -e "CREATE FUNCTION fnv_64 RETURNS INTEGER SONAME 'libfnv_udf.so'"
        mysql -e "CREATE FUNCTION murmur_hash RETURNS INTEGER SONAME 'libmurmur_udf.so'"

 * See http://www.percona.com/doc/percona-xtradb-cluster/5.6/management/udf_percona_toolkit.html for more details


Job for mysql.service failed. See 'systemctl status mysql.service' and 'journalctl -xn' for details.
invoke-rc.d: initscript mysql, action "start" failed.
dpkg: error processing package percona-xtradb-cluster-server-5.6 (--configure):
 subprocess installed post-installation script returned error exit status 1
Errors were encountered while processing:
 percona-xtradb-cluster-server-5.6
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

Remove newly installed Percona packages and libmysqlclient and then reinstall the packages:

```bash
root@node2:~# dpkg --get-selections | grep -i mysql
libdbd-mysql-perl    install
libmysqlclient18:amd64    install
mysql-common     install
root@node2:~# apt-get remove percona-xtradb-cluster-client-5.6 percona-xtradb-cluster-server-5.6 libmysqlclient18
```

This hint should you help. After mysql has been started, check replication status, that all works as expected. Now we will configure the node2 as a 1 node cluster. 2. Setup xtradb cluster on the slave node continuing to receive any updates from master Create and open file named `/etc/mysql/conf.d/pxc_cluster.cnf` and write the next configuration:

```bash
[mysqld]
binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
innodb_locks_unsafe_for_binlog = 1
wsrep_cluster_address = gcomm://
wsrep_cluster_name = my_unicorn_cluster
wsrep_node_address = 10.13.200.2
wsrep_node_name = node2
wsrep_provider = /usr/lib/libgalera_smm.so
wsrep_sst_auth = sst:secret
wsrep_sst_method = xtrabackup-v2
```

All important parameters are described here. Several of them I will explain:
- `wsrep_cluster_address` - cluster address. Here we defined no IP address, which means that node is allowed to build a cluster all by itself alone. Later we will change this parameter.
- `wsrep_node_address` - node2's IP.
- `wsrep_sst_auth` - the most important setting, which could save your time later. For a SST (state snapshot transfer - actually just a full backup) we need create a user on future primary (alone) node2, which will be the first node in the cluster. It's obvious, that sst is its name and secret is its password.
- `wsrep_sst_method` - to make, transfer and apply a snapshot on donor (in our case this would be node2) and joiner node (current acts as a master, would be node1) I'm using xtrabackup-v2. You can choose different methods, but then configuration can different.

Okay, now restart mysql and apply the changes.

We need to do the last 2 things: create a mysql user sst with proper privileges and check our cluster state.
To create the user execute following SQL-commands:

```sql
mysql@node2> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'secret';
mysql@node2> GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sst'@'localhost';
mysql@node2> FLUSH PRIVILEGES;
```

Our ex-slave now is fully ready.

3. Configure Galera Arbitrator on a third node (node3) and join it in cluster with node2
To decrease possibly cluster latency due to disk I/O and network latency the node3 will be set as a Galera Arbitrator.
Install percona-xtradb-cluster-garbd-5.7:

```bash
apt-get install percona-xtradb-cluster-garbd-5.7
```

Change file `/etc/default/garbd`:

```bash
# Copyright (C) 2012 Codership Oy
# This config file is to be sourced by garb service script.

# A comma-separated list of node addresses (address[:port]) in the cluster
GALERA_NODES="10.13.200.2:4567"

# Galera cluster name, should be the same as on the rest of the nodes.
GALERA_GROUP="myreports_rc"

# Optional Galera internal options string (e.g. SSL settings)
# see http://galeracluster.com/documentation-webpages/galeraparameters.html
# GALERA_OPTIONS=""

# Log file for garbd. Optional, by default logs to syslog
# Deprecated for CentOS7, use journalctl to query the log for garbd
# LOG_FILE=""
```

Don't forget to remove a line in the file, else garbd will not start:

```
# REMOVE THIS AFTER CONFIGURATION
```

Start a daemon:

```bash
service garbd start
```

`journalctl -u garbd` will show you the daemon logs and can be used for debug info.
