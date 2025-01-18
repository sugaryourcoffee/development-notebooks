Fail over MySQL 8 and Rails
===========================

We have two machines with each having a MySQL server and the Rails application
running _Secondhand_. One of the servers is the _source_ with IP address 
192.168.178.164 the other is the _replica_ with the IP address 192.168.178.89. 
The _source_ database will be mirrored to the _replica_ database.
In case our _source_ is crashing we want to switch over to the _replica_ server. 

In order to implement such configuration we have to conduct following step 
assuming we have both servers running with MySQL and Secondhand

* Configure the _source_ server
  * Bind the IP address on the _source_ server that is visible to the _replica_ server
  * Set the MySQL server ID on the _source_ server
  * Set the log file on the MySQL _source_ server
  * Define the database name that shall be replicated
  * Restart the _source_ server
  * Add user and set permission on MySQL _source_ to give the MySQL _replica_ access
* Create and copy the current _source_ database dump to the _replica_ server
* Configure the _replica_ server
  * Check the connection from the _replica_ to the _source_ database
  * On the MySQL _replica_ set the server ID and bind address
  * Configure the _source_ server
  * Add same user on _replica_ with access rights for _source_ replication
* Restore the _source_ database on the MySQL _replica_
* Start replication on the MySQL _replica_

Configure the MySQL server
--------------------------
Log into the MySQL _source_ server

    development$ ssh source

To configure MySQL replication we make changes to the configuratin file `/etc/mysql/my.conf.d/mysqld.cnf`.

We have to make the database available to machines other than local host. We do that by changing the `bind-address = 172.0.0.1` to the public IP-address of the _source_ server

    bind-address = 192.168.178.164

On the MySQL server we have to set the server ID and the log file. We do that by uncommenting these two lines

    server-id = 1
    log_bin   = /var/log/mysql/mysql-bin.log

We tell which database we want to replicate 

    binlog_do_db = secondhand_production 

After changing the configuration we have to restart the server

    uranus$ sudo systemctl restart mysql

We can check up on the replication with

    source$ mysql -uroot -p
    mysql> show master status;
    +------------------+----------+-----------------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB          | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+-----------------------+------------------+-------------------+
    | mysql-bin.000001 |      157 | secondhand_production |                  |                   |
    +------------------+----------+-----------------------+------------------+-------------------+
    1 row in set (0.00 sec)

Next we create a replication user that is restricted to the rights needed for
replication.

    mysql> create user 'replicant'@'%' identified with mysql_native_password by 'replicant_pass';
    mysql> grant replication slave on *.* to 'replicant'@'%';
    mysql> flush privileges;

Copy _source_ database to the _replica_ database
------------------------------------------------

We dump the production database and then copy the dump file from the _source_ to
the _replica_ server. On the _replica_ server we restore the dump file into the _replica_
database.

    development$ ssh source
    source$ mysqldump -uroot -p --quick --single-transaction --triggers \
      --master-data secondhand_production | gzip > secondhand-repl.sql.gz
    source$ scp secondhand-repl.sql.gz user@replica:secondhand-repl.sql.gz
    source$ exit

Configure the MySQL replica
---------------------------

We first check that we can access the _source_ from the _replica_ server.

    development$ ssh replica
    replica$ mysql -ureplicant -preplicant_pass -h192.168.178.164 --port 3306 \
    -e "show databases"

This should list the databases on the server.

The configuration on the _replica_ is similar. We first edit the `/etc/mysql/mysql.conf.d/mysqld.cnf`
by setting the *server-id* and the *bind-address* of the _replica_

    uranus$ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf

We adjust mysqld.cnf to meet the settings of the _source_ database

    server-id = 2
    bind-address = 192.168.178.89
    log_bin = /var/log/mysql/mysql-bin.log
    binlog_do_db = secondhand_production
    relay_log = /var/log/mysql/mysql-relay-bin

The server-id has to be unique. We just increment for each _replica_ the server-id.
Our _source_ has the server-id 1 and our _replica_ 2. If we add additional _replica_ s we
just increment each _replica_'s server-id by 1.

The bind-address has to be set to the _replica_'s IP address.

`log_bin` and `binlog_do_db` have to be the same as on the _source_.

Additioally we make `relay_log` explicit and not using the default.

After the configuration we have to restart the server in order the changes to take
effect

    replica$ sudo systemctl restart mysql

Now in MySQL we tell the slave where to find the _source_ server and with which
user to connect as well as additional information necessary for syncing the database.

To get the information on the current `log_file` and the `log_pos` we retrieve this on the _source_ in the MySQL CLI with 

    mysql> show master status;
    +------------------+----------+-----------------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB          | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+-----------------------+------------------+-------------------+
    | mysql-bin.000001 |     1714 | secondhand_production |                  |                   |
    +------------------+----------+-----------------------+------------------+-------------------+
    1 row in set (0.00 sec)

we use this information for `master_log_file` and `master_log_pos` respectively

    uranus$ mysql -uroot -p
    mysql> change master to
        -> master_host='192.168.178.164',
        -> master_user='replicant',
        -> master_password='replicant_pass',
        -> master_log_file='mysql-bin.000001',
        -> master_log_pos=1714;

We can monitor more detailed information on what is MySQL doing by monitoring the `/var/log/mysql/error.log` with 

    replica$ tail -f /var/log/mysql/error.log 

Note: If your MySQL server or your _replica_ gets a new IP address you have to 
adjust the `bind-address` in **/etc/mysql/my.conf** and in **mysql** 
accordingly.

Otherwise you will get an error if you call the secondhand web page *something went wrong*. 
The error in log/production.log is:

`Mysql2::Error (Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2))`

And if you start MySQL from the command line with `$ mysql` you will get the same error with slightly different text:

`ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)`

Restore database on replica server
----------------------------------
We now restore the previously dumped database from the server on the _replica_. This
will include the log file name and log file position.

    development$ ssh replica

Before we restore the database we have to create the database where we want to
restore it to

    replica$ mysql -uroot -p
    mysql> create database secondhand_production default character set utf8;
    mysql> grant all privileges to 'pierre'@'localhost' identified by 'secret';
    mysql> exit

Now we restore the database into secondhand\_production

    uranus$ gunzip < secondhand-repl.sql.gz | mysql -uroot -p \
    secondhand_production

Start replication
-----------------
On the _replica_ server uranus we start the replication

    uranus$ mysql -uroot -p
    mysql> start slave;

We can check the _replica_ status with

    mysql> show slave status\G;

Adding additional _replica_ s
------------------------
If we want to have additional _replica_ s for replicating the _source_ database then we
just follow the steps above, that is

On the _source_ server

* dump the _source_ database and copy it to the new _replica_ server

On the _replica_ server

* set `server-id` to a unique positive integer and set the bind-address to the
  _replica_ server's IP address. Both is done in /etc/mysql/my.cnf 
* restart the MySQL server
* configure the _source_ server in mysql
* create the database in MySQL
* restore the dumped database into MySQL
* start the replication

Error Messages
--------------
If you get an error message saying

    Mysql2::Error (Can't connect to local MySQL server through socket 
    '/var/run/mysqld/mysqld.sock' (2)):

Then you probably have the wrong IP address in you MySQL _replica_ server. Check 
that the IP address in /etc/mysql/my.conf is the IP address of the _replica_ server
itself.

Changing IP address might happen when you retrieve your IP address from a new
DHCP server.

### Fatal Error 1236
The _source_ and _replica_ might get out of sync if the _replica_ is shut down for a
while or the connection is interrupted. As long as the `Master_Log_File` value
on the _replica_ is still available on the _source_ the _replica_ will catch up the
_source_. If the `Master_Log_File` is overridden then `show slave status\G`
will have the value `Last_IO_Errno: 1236` and the `Last_IO_Error: Got fatal
error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'`. It might help to reset the
_replica_ with 

    mysql> stop slave;
    mysql> reset slave;
    mysql> start slave;
    mysql> show slave status\G;

If the error is gone. The _replica_ might be gradually synced with the _source_. If
data is not propperly synced then dump the database on the server and restore
it on the _replica_ as shown [Copy source database to the replica database](#copy-source-database-to-the-replica-database) and in [Restore database on replica server](#restore-database-on-replica-server).

### Last_SQL_Errno: 1133
The error message
    Last_SQL_Error: Error 'Can't find any matching row in the user table' on 
    query. Detault database: ''. Query: 'SET PASSWORD FOR 'repl'@'%'='*....''

indicates that on the _replica_ machine the user 'repl'@'%' is not set. The repl 
user needs to be set on each of the machines that are part of the replication.

### Could not initialize master info structure 

If the following error message is raised while recovering the database to the backup server 

    gunzip < secondhand-repl.sql.gz | mysql -uroot -p secondhand_production
    Enter password:
    ERROR 1201 (HY000) at line 22: Could not initialize master info structure; more error messages can be found in the MySQL error log
    pierre@jupiter:~$ gunzip < secondhand-repl.sql.gz | mysql -uroot -p secondhand_production
    Enter password:
    ERROR 1201 (HY000) at line 22: Could not initialize master info structure; more error messages can be found in the MySQL error log

This can be solved by **resetting the _replica_**

    $ mysql -uroot -p
    Enter password:
    mysql> use secondhand_production;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A
    Database changed
    mysql> reset slave;
    Query OK, 0 rows affected (0.28 sec)
    mysql> exit
    Bye

The we repeat the command to recover the database to the backup server database 

    $ gunzip < secondhand-repl.sql.gz | mysql -uroot -p secondhand_production
    Enter password:
    mysql> use secondhand_production;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A
    Database changed
    mysql> start slave;
    Query OK, 0 rows affected (0.00 sec)
    mysql> exit 
    Bye

### Slave I/O: Got fatal error 1236 from master

This error might result from differnt MySQL versions, like replica runs version 5.5 and source runs 8.0 

    [ERROR] Slave I/O: Got fatal error 1236 from master when reading data from binary log: 
    'Replica can not handle replication events with the checksum that source is configured to log; the first event '' at 4, 
    the last event read from '/var/log/mysql/mysql-bin.000001' at 126, the last byte read from '/var/log/mysql/mysql-bin.000001' at 126.', 
    Error_code: 1236
    250118 15:54:03 [Note] Slave I/O thread exiting, read up to log 'mysql-bin.000001', position 4

A interims solution might be to that is setting the parameter `binlog_checksum=NONE` in the _source_'s configuration file `/etc/mysql/mysql.conf.d/mysqld.cnf`. Then the _source_ server has to be restarted with `sudo systemctl restart mysql` and the slave has to redo `mysql>change server to` with the new information about the `master_log_file` and `master_log_pos`.

Sources
-------
* [MySQL replication - YouTube](https://www.youtube.com/watch?v=JXDuVypcHNA)
* [High Performance MySQL - O'Reilly](http://shop.oreilly.com/product/0636920022343.do)
* [Deploying Rails Applications - Pragmatic Programmer](https://pragprog.com/book/cbdepra/deploying-rails)
* [How to set up replication in MySQL](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql)
