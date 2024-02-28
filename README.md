# Mysql 5.7 Master Slave HA with Orchestrator And ProxySQL

## VM Requirement
| Hostname       | Role         | Os               | Specs                                |
|----------------|--------------|------------------|:-------------------------------------|
| 192.168.100.18 | Master       | Ubuntu 18.04 LTS | CPU Core 2, RAM 8GB, Storage 100GB   |
| 192.168.100.19 | Slave 1      | Ubuntu 18.04 LTS | CPU Core 2, RAM 8GB, Storage 100GB   |
| 192.168.100.20 | Slave 2      | Ubuntu 18.04 LTS | CPU Core 2, RAM 8GB, Storage 100GB   |
| 192.168.100.25 | Orchestrator | Ubuntu 18.04 LTS | Cpu Core 4, RAM 4GB, Storage 60GB    |
| 192.168.100.25 | ProxySQL     | Ubuntu 18.04 LTS | Cpu Core 4, RAM 4GB, Storage 60GB    |

## Install Mysql on Ubuntu 18.04 LTS

```
$ sudo apt get update
$ sudo apt install mysql-server mysql-client -y
```

## Uninstall Mysql on Ubuntu 18.04 LTS

```
$ sudo apt purge mysql*
$ sudo apt autoremove
```

## Step 1 - Installing Mysql

Install mysql on Master Host and all Slave Host

## Step 2 - Configure Mysql Remote Access

Change default (localhost) root password and plugin :

```
$ sudo mysql
mysql> USE mysql;
mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new-password';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

Create new root user for remote access :

```
$ sudo mysql
mysql> USE mysql;
mysql> CREATE USER 'root'@'%' IDENTIFIED BY 'YOUR_PASSWD';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

Run `mysql_secure_installation` :

```
$ mysql_secure_installation
    
Enter password for user root: 

VALIDATE PASSWORD PLUGIN can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD plugin?

Press y|Y for Yes, any other key for No: n
Using existing password for root.
Change the password for root ? ((Press y|Y for Yes, any other key for No) : Y

New password: 

Re-enter new password: 
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : n

 ... skipping.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : n

 ... skipping.
By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : n

 ... skipping.
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y
Success.

All done!
```

Edit mysql config at `/etc/mysql/mysql.conf.d/mysqld.cnf` :

`sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf`

Change bind address to 0.0.0.0 :

`bind-address = 0.0.0.0`

Restart Mysql Service :

`$ sudo service mysql restart`

Do all command above in Master and Slaves Host

Verify mysql remote connection  by remoting them using your favorite mysql client management such as mysql workbench, navicat etc.

## Step 3 - Configure Master Slave Replication

Enable replication config by editing `/etc/mysql/mysql.conf.d/mysqld.cnf` :

On Master Node :

```
server-id=1
log_bin=/var/log/mysql/mysql-bin.log
log_slave_updates=1
report-host={MASTER_HOSTNAME}
report-port=3306
master-info-repository=TABLE
```
 
 change `{MASTER_HOSTNAME}` with your vm master hostname
 
 On Slave 1 Node :

 ```
 server-id=2
 log_bin=/var/log/mysql/mysql-bin.log
 log_slave_updates=1
 report-host={SLAVE1_HOSTNAME}
 report-port=3306
 master-info-repository=TABLE
 ```
 change `{SLAVE1_HOSTNAME}` with your vm slave1 hostname

 On Slave 2 Node :

 ```
 server-id=3
 log_bin=/var/log/mysql/mysql-bin.log
 log_slave_updates=1
 report-host={SLAVE2_HOSTNAME}
 report-port=3306
 master-info-repository=TABLE
 ```
 change `{SLAVE2_HOSTNAME}` with your vm slave2 hostname

 Next, restart mysql service :
 
 `$ sudo service mysql restart`
 
 Now create user for replication on master node :

 ```
 $ sudo mysql -uroot -p
 mysql> CREATE USER 'slave'@'%' IDENTIFIED BY 'SLAVE_PASSWORD';
 mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
 mysql> FLUSH PRIVILEGES
 ```

 Next, turn on the lock on your databases to prevent the change in data.
 
 `mysql> FLUSH TABLES WITH READ LOCK;`
 
 As the tables are now locked, we can record the current log position so that our Slave server can start reading data from that log position. To know the current status of master server, execute the following command in your MySQL interface.
 
 `mysql> SHOW MASTER STATUS;`
 
 You will see a table with single row just like following information.
 
 
| File             | Position | Binlog_Do_DB | Bin_Log_Ignore_DB | Executed_Gtid_Set |
|------------------|----------|--------------|:------------------|:------------------|
| mysql-bin.000001 | 154      |              |                   |                   |

Next backup data on master node :

`$ sudo mysqldump -u root -p –all-databases –master-data > data.sql`

Transfer dump sql to slave nodes :

```
scp data.sql root@192.168.100.19:~
scp data.sql root@192.168.100.20:~
```

To migrate the file, you will have to enter the SSH password of slave. After migration is done, Unlock tables on master.

`mysql> UNLOCK TABLES;`

Import dump sql on slave nodes :

`$ mysql -uroot -p < data.sql`

Once the data is imported, Login to MySQL in slave and stop the slave using following command.

```
$ sudo mysql -uroot -p;
mysql> STOP SLAVE;
```

Now we can change the master so that our slave can know which server to replicate. Update master information using the following command in MySQL.

`mysql> CHANGE MASTER TO MASTER_HOST='192.168.100.18', MASTER_USER='slave', MASTER_PASSWORD='SLAVE_PASSWORD', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=154;`

Adjust the following parameter with your own :

MASTER_HOST = 192.168.100.18
MASTER_USER = slave
MASTER_PASSWORD = SLAVE_PASSWORD
MASTER_LOG_FILE = mysql-bin.000001
MASTER_LOG_POS = 154

After executing the command, Start slave by using the `START SLAVE` command on the slave server.

## Step 4 - Testing Mysql Replication

Create database on your master node :

```
$ sudo mysql -uroot -p
mysql> CREATE DATABASE test;
```

Now, Open up MySQL command line interface on your slave server and check if the new database exist. To get the list of database, execute the following command on slave.

```
$ sudo mysql -uroot -p
mysql> SHOW DATABASES;
```

In the list, you must see a database named “test”. If you can see the database, MySQL replication is working. If not, Replication is not working. In that case, you can follow the guide from start again.

## Step 5 - Installing Orchestrator

```
$ curl -s https://packagecloud.io/install/repositories/github/orchestrator/script.deb.sh | sudo bash
$ sudo apt update
$ sudo apt install orchestrator -y
```

Setup a MySQL server for backend, and invoke the following :

```
$ sudo apt update
$ sudo apt install mysql-server mysql-client -y
$ sudo mysql
mysql> CREATE DATABASE IF NOT EXISTS orchestrator;
mysql> CREATE USER 'orchestrator'@'127.0.0.1' IDENTIFIED BY 'orch_backend_password';
mysql> GRANT ALL PRIVILEGES ON 'orchestrator'.* TO 'orchestrator'@'127.0.0.1';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

Create Orchestrator config file by copying from template :

`$ sudo cp /usr/local/orchestrator/orchestrator-sample.conf.json /usr/local/orchestrator/orchestrator.conf.json`

Edit config and configure with following config :

`$ sudo nano /usr/local/orchestrator/orchestrator.conf.json`

```
...
"MySQLOrchestratorHost": "127.0.0.1",
"MySQLOrchestratorPort": 3306,
"MySQLOrchestratorDatabase": "orchestrator",
"MySQLOrchestratorUser": "orchestrator",
"MySQLOrchestratorPassword": "orch_backend_password",
...
```

Next, create user for orchestrator on master node :

```
$ sudo mysql -uroot -p
mysql> CREATE USER 'orchestrator'@'%' IDENTIFIED BY 'orch_topology_password';
mysql> GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orchestrator'@'%';
mysql> GRANT SELECT ON mysql.slave_master_info TO 'orchestrator'@'%';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

Back to orchestrator node and edit config file :

`$ sudo nano /usr/local/orchestrator/orchestrator.conf.json`

```
...
"MySQLTopologyUser": "orchestrator",
"MySQLTopologyPassword": "orch_topology_password",
...
```

Now start orchestrator service :

```
$ sudo service orchestrator start
$ sudo systemctl enable orchestrator
```

Then access web interface in the browser with following url :

http://192.168.100.25:3000

![alt text](https://github.com/indragto/mysql57-ha-orchestrator-proxysql/blob/main/Screenshot%202024-02-28%20at%2000.08.53.png?raw=true)

If you get warning `NoLoggingReplicasStructureWarning` or `NoFailoverSupportStructureWarning` then edit mysql config and add following params on master and slave nodes :

```
$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
gtid_mode=ON
enforce-gtid-consistency=ON
```

Restart mysql

`$ sudo service mysql restart`

then if you get warning `ErrantGTIDStructureWarning` just click icon on topology interface then find and click button `fix`. Select `inject with empty transaction` to fix error.

## Step 6 - Installing ProxySQL

```
$ sudo apt-get install -y --no-install-recommends lsb-release wget apt-transport-https ca-certificates gnupg
$ sudo wget -O - 'https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/repo_pub_key' | apt-key add -'
# echo deb https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/$(lsb_release -sc)/ ./ | tee /etc/apt/sources.list.d/proxysql.list
$ wget -nv -O /etc/apt/trusted.gpg.d/proxysql-2.5.x-keyring.gpg 'https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/repo_pub_key.gpg'
$ sudo apt get update
$ sudo apt-get install proxysql
```

Check current installation proxysql version :

`$ proxysql --version`

Starting service :

`$ service proxysql start`

## Step 7 - Configure ProxySQL

Login to proxysql admin cli :

`$ mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt 'ProxySQL Admin> '`

Verify that the configuration is empty by checking that there are no entries in the mysql_servers, mysql_replication_hostgroups and mysql_query_rules tables.

```
ProxySQL Admin> SELECT * FROM mysql_servers;
Empty set (0.00 sec)
```

```
ProxySQL Admin> SELECT * from mysql_replication_hostgroups;
Empty set (0.00 sec)
```
```
ProxySQL Admin> SELECT * from mysql_query_rules;
Empty set (0.00 sec)
```

Add Backends :

```
ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'192.168.100.18',3306);
Query OK, 1 row affected (0.01 sec)

ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'192.168.100.19',3306);
Query OK, 1 row affected (0.01 sec)

ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'192.168.100.20',3306);
Query OK, 1 row affected (0.00 sec)

ProxySQL Admin> SELECT * FROM mysql_servers;
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
| hostgroup_id | hostname  | port | status | weight | compression | max_connections | max_replication_lag |
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
| 1            | 192.168.100.18 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
| 1            | 192.168.100.19 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
| 1            | 192.168.100.20 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
3 rows in set (0.00 sec)
```

Create user for proxysql monitoring on master node :

```
mysql> CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';
Query OK, 1 row affected (0.00 sec)
```
```
mysql> GRANT USAGE, REPLICATION CLIENT ON *.* TO 'monitor'@'%';
Query OK, 0 rows affected (0.00 sec)
```
```
mysql> FLUSH PRIVILEGES;
```

Then add the credentials of the monitor user to ProxySQL :

```
ProxySQL Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
Query OK, 1 row affected (0.00 sec)

ProxySQL Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';
Query OK, 1 row affected (0.00 sec)
```

Then configure the various monitoring intervals :

```
ProxySQL Admin> UPDATE global_variables SET variable_value='2000' WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');
Query OK, 3 rows affected (0.00 sec)

ProxySQL Admin> SELECT * FROM global_variables WHERE variable_name LIKE 'mysql-monitor_%';
+----------------------------------------+---------------------------------------------------+
| variable_name                          | variable_value                                    |
+----------------------------------------+---------------------------------------------------+
| mysql-monitor_history                  | 600000                                            |
| mysql-monitor_connect_interval         | 2000                                              |
| mysql-monitor_connect_timeout          | 200                                               |
| mysql-monitor_ping_interval            | 2000                                              |
| mysql-monitor_ping_timeout             | 100                                               |
| mysql-monitor_read_only_interval       | 2000                                              |
| mysql-monitor_read_only_timeout        | 100                                               |
| mysql-monitor_replication_lag_interval | 10000                                             |
| mysql-monitor_replication_lag_timeout  | 1000                                              |
| mysql-monitor_username                 | monitor                                           |
| mysql-monitor_password                 | monitor                                           |
| mysql-monitor_query_variables          | SELECT * FROM INFORMATION_SCHEMA.GLOBAL_VARIABLES |
| mysql-monitor_query_status             | SELECT * FROM INFORMATION_SCHEMA.GLOBAL_STATUS    |
| mysql-monitor_query_interval           | 60000                                             |
| mysql-monitor_query_timeout            | 100                                               |
| mysql-monitor_timer_cached             | true                                              |
| mysql-monitor_writer_is_also_reader    | true                                              |
+----------------------------------------+---------------------------------------------------+
17 rows in set (0.00 sec)
```

Changes made to the MySQL Monitor in table global_variables will be applied after executing the LOAD MYSQL VARIABLES TO RUNTIME statement. To persist the configuration changes across restarts the SAVE MYSQL VARIABLES TO DISK must also be executed.

```
ProxySQL Admin> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)

ProxySQL Admin> SAVE MYSQL VARIABLES TO DISK;
Query OK, 54 rows affected (0.02 sec)
```

Backend’s health check

Once the configuration is active verify the status of the MySQL backends in the monitor database tables in ProxySQL Admin:

```
ProxySQL Admin> SHOW TABLES FROM monitor;
+----------------------------------+
| tables                           |
+----------------------------------+
| mysql_server_connect             |
| mysql_server_connect_log         |
| mysql_server_ping                |
| mysql_server_ping_log            |
| mysql_server_read_only_log       |
| mysql_server_replication_lag_log |
+----------------------------------+
6 rows in set (0.00 sec)
```

Each check type has a dedicated logging table, each should be checked individually:

```
ProxySQL Admin> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 3;
+-----------+------+------------------+----------------------+---------------+
| hostname  | port | time_start_us    | connect_success_time | connect_error |
+-----------+------+------------------+----------------------+---------------+
| 192.168.100.18 | 3306 | 1456968814253432 | 562                  | NULL          |
| 192.168.100.19 | 3306 | 1456968814253432 | 309                  | NULL          |
| 192.168.100.20 | 3306 | 1456968814253432 | 154                  | NULL          |
+-----------+------+------------------+----------------------+---------------+
3 rows in set (0.00 sec)

ProxySQL Admin> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 3;
+-----------+------+------------------+-------------------+------------+
| hostname  | port | time_start_us    | ping_success_time | ping_error |
+-----------+------+------------------+-------------------+------------+
| 192.168.100.18 | 3306 | 1456968828686787 | 124               | NULL       |
| 192.168.100.19 | 3306 | 1456968828686787 | 62                | NULL       |
| 192.168.100.20 | 3306 | 1456968828686787 | 57                | NULL       |
+-----------+------+------------------+-------------------+------------+
3 rows in set (0.01 sec)
```

One important thing to note here is that connect and ping monitoring is done based on the configured mysql_servers even before this is loaded to RUNTIME. This approach is intentional: in this way it is possible to perform basic health checks before adding the nodes in production.

After verifying that the servers are being monitored correctly and are healthy activate the configuration:

```
ProxySQL Admin> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)

ProxySQL Admin> SELECT * FROM mysql_servers;
+--------------+-----------+-------+--------+--------+-------------+-----------------+---------------------+
| hostgroup_id | hostname  | port  | status | weight | compression | max_connections | max_replication_lag |
+--------------+-----------+-------+--------+--------+-------------+-----------------+---------------------+
| 1            | 192.168.100.18 | 3306  | ONLINE | 1      | 0           | 1000            | 0                   |
| 1            | 192.168.100.19 | 3306  | ONLINE | 1      | 0           | 1000            | 0                   |
| 1            | 192.168.100.20 | 3306  | ONLINE | 1      | 0           | 1000            | 0                   |
+--------------+-----------+-------+--------+--------+-------------+-----------------+---------------------+
3 rows in set (0.00 sec)
```

MySQL replication hostgroups

Cluster topology changes are monitored based on MySQL replication hostgroups configured in ProxySQL. ProxySQL understands the replication topology by monitoring the value of read_only on servers configured in hostgroups that are configured in mysql_replication_hostgroups.

This table is empty by default and should be configured by specifying a pair of READER and WRITER hostgroups, although the MySQL backends might all be right now in a single hostgroup. For example:

```
ProxySQL Admin> SHOW CREATE TABLE mysql_replication_hostgroups\G
*************************** 1. row ***************************
       table: mysql_replication_hostgroups
Create Table: CREATE TABLE mysql_replication_hostgroups (
writer_hostgroup INT CHECK (writer_hostgroup>=0) NOT NULL PRIMARY KEY,
reader_hostgroup INT NOT NULL CHECK (reader_hostgroup<>writer_hostgroup AND reader_hostgroup>0),
check_type VARCHAR CHECK (LOWER(check_type) IN ('read_only','innodb_read_only','super_read_only','read_only|innodb_read_only','read_only&innodb_read_only')) NOT NULL DEFAULT 'read_only',
comment VARCHAR,
UNIQUE (reader_hostgroup))
1 row in set (0.00 sec)

ProxySQL Admin> INSERT INTO mysql_replication_hostgroups (writer_hostgroup,reader_hostgroup,comment) VALUES (1,2,'cluster1');
Query OK, 1 row affected (0.00 sec)
```

Now, all the MySQL backend servers that are either configured in hostgroup 1 or 2 will be placed into their respective hostgroup based on their read_only value:

If they have read_only=0 , they will be moved to hostgroup 1
If they have read_only=1 , they will be moved to hostgroup 2
To enable the replication hostgroup load mysql_replication_hostgroups to runtime using the same LOAD command used for MySQL servers since LOAD MYSQL SERVERS TO RUNTIME processes both mysql_servers and mysql_replication_hostgroups tables.

```
ProxySQL Admin> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)
```

The read_only check results are logged to the mysql_servers_read_only_log table in the monitor database:

```
ProxySQL Admin> SELECT * FROM monitor.mysql_server_read_only_log ORDER BY time_start_us DESC LIMIT 3;
+-----------+-------+------------------+--------------+-----------+-------+
| hostname  | port  | time_start_us    | success_time | read_only | error |
+-----------+-------+------------------+--------------+-----------+-------+
| 192.168.100.18 | 3306  | 1456969634783579 | 762          | 0         | NULL  |
| 192.168.100.19 | 3306  | 1456969634783579 | 378          | 1         | NULL  |
| 192.168.100.20 | 3306  | 1456969634783579 | 317          | 1         | NULL  |
+-----------+-------+------------------+--------------+-----------+-------+
3 rows in set (0.01 sec)
```

Allright, ProxySQL is monitoring the read_only value for the servers.
And it also created hostgroup2 to where it has moved servers with read_only=1 (readers) from hostgroup1.

```
ProxySQL Admin> SELECT * FROM mysql_servers;
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
| hostgroup_id | hostname  | port | status | weight | compression | max_connections | max_replication_lag |
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
| 1            | 192.168.100.18 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
| 2            | 192.168.100.19 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
| 2            | 192.168.100.20 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   |
+--------------+-----------+------+--------+--------+-------------+-----------------+---------------------+
3 rows in set (0.00 sec)
```

As a final step, persist the configuration to disk.

```
ProxySQL Admin> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.01 sec)

ProxySQL Admin> SAVE MYSQL VARIABLES TO DISK;
Query OK, 54 rows affected (0.00 sec)
```

## Step 8 - Create ProxySQL User for Application Connection

After configuring the MySQL server backends in mysql_servers the next step is to configure mysql users.

For the purpose of this example, we are creating a MySQL user with no particular restrictions: this is not a good practice and the user should be configured with proper connection restrictions and privileges according to the setup and the application needs.
To create the user in MySQL connect to the PRIMARY and execute:

```
mysql> CREATE USER 'stnduser'@'%' IDENTIFIED BY 'stnduser';
Query OK, 1 row affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'stnduser'@'%';
Query OK, 0 rows affected (0.00 sec)
```

Time to configure the user into ProxySQL: this is performed by adding entries to the mysql_users table:

```
ProxySQL Admin> SHOW CREATE TABLE mysql_users\G
*************************** 1. row ***************************
       table: mysql_users
Create Table: CREATE TABLE mysql_users (
username VARCHAR NOT NULL,
password VARCHAR,
active INT CHECK (active IN (0,1)) NOT NULL DEFAULT 1,
use_ssl INT CHECK (use_ssl IN (0,1)) NOT NULL DEFAULT 0,
default_hostgroup INT NOT NULL DEFAULT 0,
default_schema VARCHAR,
schema_locked INT CHECK (schema_locked IN (0,1)) NOT NULL DEFAULT 0,
transaction_persistent INT CHECK (transaction_persistent IN (0,1)) NOT NULL DEFAULT 0,
fast_forward INT CHECK (fast_forward IN (0,1)) NOT NULL DEFAULT 0,
backend INT CHECK (backend IN (0,1)) NOT NULL DEFAULT 1,
frontend INT CHECK (frontend IN (0,1)) NOT NULL DEFAULT 1,
max_connections INT CHECK (max_connections >=0) NOT NULL DEFAULT 10000,
PRIMARY KEY (username, backend),
UNIQUE (username, frontend))
1 row in set (0.00 sec)
```

The table is initially empty, to add users specify the username, password and default_hostgroup for basic configuration:

```
ProxySQL Admin> INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('stnduser','stnduser',1);
Query OK, 1 row affected (0.00 sec)

ProxySQL Admin> SELECT * FROM mysql_users;                                                                                                                      
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
| username | password | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
| stnduser | stnduser | 1      | 0       | 1                 | NULL           | 0             | 0                      | 0            | 1       | 1        | 10000           |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
1 row in set (0.00 sec)
```

By defining the default_hostgroup we are specifying which backend servers a user should connect to BY DEFAULT (i.e. this will be the default route for traffic coming from the specific user, additional rules can be configured to re-route however in their absence all queries will go to the specific hostgroup).

```
ProxySQL Admin> LOAD MYSQL USERS TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)
ProxySQL Admin> SAVE MYSQL USERS TO DISK;
Query OK, 0 rows affected (0.01 sec)
```

ProxySQL is now ready to serve traffic on port 6033 (by default):

```
$ mysql -u stnduser -pstnduser -h 127.0.0.1 -P6033 -e"SELECT @@port"
Warning: Using a password on the command line interface can be insecure.
+--------+
| @@port |
+--------+
|  3306  |
+--------+
```

This query was sent to the server listening on port 3306 , the primary, as this is the server configured on hostgroup1 and is the default for user `stnduser`.

## References Links

https://www.interserver.net/tips/kb/create-master-slave-replication-mysql-server/

https://github.com/openark/orchestrator/blob/master/docs/install.md

https://github.com/openark/orchestrator/blob/master/docs/using-the-web-interface.md

https://proxysql.com/documentation/installing-proxysql/

https://proxysql.com/documentation/getting-started/

https://proxysql.com/documentation/ProxySQL-Configuration/

https://www.percona.com/blog/fixing-errant-gtid-with-orchestrator/

https://github.com/openark/orchestrator/issues/1396

https://github.com/openark/orchestrator/issues/837

https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost
