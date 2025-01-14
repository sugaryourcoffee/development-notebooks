Run MySQL without sudo
======================

If it happens that a login attempt like this `$ mysql -uroot -p` and you type in the correct password, and still get the error `ERROR 1698 (28000): Access denied for user 'root'@'localhost'` you can try to run `$ sudo mysql -uroot -p`. If this works you adminstrate your databases but in some cases it doesn't work, at least to my experience. When I tried to recover a database with `$ gunzip < secondhand-repl.sql.gz | mysql -uadmin -p secondhand_production`, I got the above mentioned error. Also when invoking the command with `$ sudo gunzip < secondhand-repl.sql.gz | mysql -uadmin -p secondhand_production`, I got the exact same error. After searching the Internet there was the easiest solution provided at [askubuntu.com](https://askubuntu.com/questions/1312977/logging-in-mysql-as-root-works-only-on-sudo).

The relevant part is following, but a more detailed explantation and additional use cases are at following the link mentioned above.

    $ sudo mysql -uroot -p
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 21
    Server version: 8.0.40-0ubuntu0.24.04.1 (Ubuntu)
    
    Copyright (c) 2000, 2024, Oracle and/or its affiliates.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

Now the relevant parts follow 

    mysql> create user 'admin'@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.10 sec)
    
    mysql> grant all on *.* to 'admin'@'localhost' with grant option;
    Query OK, 0 rows affected (0.12 sec)
    
    mysql> flush privileges;
    Query OK, 0 rows affected (0.04 sec)
    
    mysql> exit
    Bye

When I login with the new admin account, I don't need `sudo`

    $ mysql -uadmin -p
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 21
    Server version: 8.0.40-0ubuntu0.24.04.1 (Ubuntu)
    
    Copyright (c) 2000, 2024, Oracle and/or its affiliates.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    
    mysql> exit
    Bye

Now also the recovery of the database is possible 

    $ gunzip < secondhand-repl.sql.gz | mysql -uadmin -p secondhand_production
    Enter password:

And if I check up for the database, everything is in place 

    mysql -uadmin -p
    Enter password:
    ...

    mysql> use secondhand_production;
    Database changed

    mysql> show tables;
    +---------------------------------+
    | Tables_in_secondhand_production |
    +---------------------------------+
    | carts                           |
    | conditions                      |
    | events                          |
    | items                           |
    | line_items                      |
    | lists                           |
    | news                            |
    | news_translations               |
    | pages                           |
    | reversals                       |
    | schema_migrations               |
    | sellings                        |
    | terms_of_uses                   |
    | users                           |
    +---------------------------------+
    14 rows in set (0.00 sec)
    
