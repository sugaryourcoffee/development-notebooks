# Install Ubuntu Server on a Dell Poweredge T130 server hardware
To install Ubuntu Server there are some easy steps to follow. We are 
using a Dell PowerEdge T130. The manuals can be found at [dell.com/poweredgemanuals](http://www.dell.com/support/home/de/de/debsdt1/product-support/product/poweredge-t130/research)

We set up the server following these steps:

1. Download ISO file from [ubuntu.com](https://ubuntu.com/download/server)
2. Create a bootable USB with the downloaded ISO file
3. Start the server hardware from USB
4. Follow the instructions prompted by the installation guide

## Create bootable USB 

After downloading the ISO image fire up a console and issue the `dd` command. Assuming the ISO image has been downloaded to the `~/Downloads` directory.

    devops@development sudo dd bs=4M if=Downloads/ubuntu-24.04.1-live-server-amd64.iso of=/dev/sda status=progress oflag=sync
    [sudo] password for devops: 
    2768240640 bytes (2,8 GB, 2,6 GiB) copied, 247 s, 11,2 MB/s
    661+1 records in
    661+1 records out
    2773874688 bytes (2,8 GB, 2,6 GiB) copied, 247,671 s, 11,2 MB/s

Next we plug in the USB drive to the server hardware DELL PowerEdge T130 and reboot the machine. _F11_ will bring us into boot selection, where we can select to boot from USB.

When prompted for _Software selection_ select 

* OpenSSH server

Then after the installation is finished and logged in run 

    $ sudo apt update 
    $ sudo apt upgrade 

# Install Ruby on Rails server environment

1. Install Apache 2
2. Create a Secondhand application user
3. Create the application directory
4. Install Git
6. Install node.js
5. Install rbenv to manage Ruby
9. Install MySQL
7. Deploy Secondhand
8. Install Passenger
10. Configure Apache 2
11. Install Postfix
12. Setup printer

## Install Apache 2 

To install Apache 2 we invoke the command 

    $ sudo apt install apache2

This will install configuration files and web directories. 

## Create a Secondhand application user 

We create a user that is managing the application. Good practice is to name the application user after the application. So we create a user _secondhand_.

    $ sudo adduser secondhand
    info: Adding user `secondhand' ...
    info: Selecting UID/GID from range 1000 to 59999 ...
    info: Adding new group `secondhand' (1001) ...
    info: Adding new user `secondhand' (1001) with group `secondhand (1001)' ...
    info: Creating home directory `/home/secondhand' ...
    info: Copying files from `/etc/skel' ...
    New password:
    Retype new password:
    passwd: password updated successfully
    Changing the user information for secondhand
    Enter the new value, or press ENTER for the default
            Full Name []:
            Room Number []:
            Work Phone []:
            Home Phone []:
            Other []:
    Is the information correct? [Y/n] Y
    info: Adding new user `secondhand' to supplemental / extra groups `users' ...
    info: Adding user `secondhand' to group `users' ...

## Create a Secondhand application directory
Apache 2's default application directory is at `/var/www`. There we add the 
directory for our secondhand backup application. 

    $ sudo mkdir /var/www/secondhand

This directory is owned by _root_. When we deploy our application we need to 
have it owned by the user deploying which is in our case the secondhand user.
In order to enable the secondhand user to deploy we change the owner and give it
write access

    $ sudo chown secondhand: /var/www/secondhand/

Now we can create the backup directory which will be owned by the user secondhand

    $ sudo -u secondhand -H bash -l
    $ sudo mkdir /var/www/secondhand/backup

## Install Git 

We can install git with 

    $ sudo install git-all 

## Install node.js 

The next step needs `node.js` installed for the assets to run 

    $ sudo apt install nodejs 

## Install rbenv to manage Ruby

For the description on how to install rbenv I want to refer to [install rbenv and ruby](installing-rbenv-and-ruby.md).

## Install MySQL

We have to install MySQL and create the database. Make sure not to only install
the _mysql-server_ but also the _libmysqlclient-dev_ otherwise bundler will fail
to install the _mysql2_ gem.

    $ sudo apt-get install mysql-server libmysqlclient-dev

## Deploy Secondhand

Change into the project directory and clone the project from Git 

    $ sudo -u secondhand -H bash -l 
    $ cd /var/www/secondhand/backup/ 
    $ git clone https://github.com/sugaryourcoffee/secondhand.git .

Copy the `database.yml` and `secret_token.rb` from the development machine to the server

    devops@development $ scp database.yml secret_token.rb secondhand@jupiter:.

Back on the server 

    $ mv database.yml /var/www/secondhand/backup/config/.
    $ mv secret_token.rb /var/www/secondhand/backup/config/intializers/.

Then check the required Ruby version 

    $ cat .ruby-version
    2.7.8

Install Ruby with 

    $ rbenv install 2.7.8 
    $ rbenv local 2.7.8 

Then install bundler that is latest regarding the Gemfile.lock 

    $ cat Gemfile.lock | grep bundler 

will reveal `bundler (>= 1.3.0, < 2.0) and we install v 1.17.3 as this is the latest before 2.0 

    $ gem install bundler -v 1.17.3

With bundler installed we install the rest of the gems 

    $ bundle install --without test development

Precompile the assets with 

    $ bundle exec rake assets:precompile RAILS_ENV=backup

### Database management

We have three stages towards production: beta, staging and production. The production version has an acompanying backup version, which is replicating the production database, and can step in in case of a breakdown of the production server.

This breaks down to three scenarios:

1. We have a pristine state and start with a blank database 
2. We are upgrading to a new Secondhand version and already have an existing database 
3. We want to setup a backup server and use the existing production database 

We describe the implementation of each of the scenarios 

What is common for scenario 1. and 3., we need to create the database and a database user that can access the database from Secondhand.

    $ sudo mysql -uroot -p 
    Enter password
    mysql> create database secondhand_production default character set utf8mb4 collate utf8mb4_0900_ai_ci;
    Query OK, 1 row affected (0.08 sec)
    mysql> create user 'secondhand'@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.07 sec)
    mysql> grant all privileges on secondhand_production.* to 'secondhand'@'localhost';
    Query OK, 0 rows affected (0.06 sec)
    mysql> exit 

Important: we have to adjust the database-name, the user and the password in `/var/www/secondhand/backup/config/database.yml` to the settings we have created the database and user in MySQL. The section for back should look like this 

    backup:
      adapter: mysql2
      encoding: utf8
      reconnect: false
      database: secondhand_production # the database we've created in MySQL
      pool: 5
      timeout: 5000
      username: secondhand            # the user we have created with all rights on the database
      password: password              # the password of secondhand user
      host: localhost

1. Start from a pristine state 

If we start from scratch then we run the rake command that creates the tables 

    $ bundle exec rake db:migrate RAILS_ENV="production"

The `RAILS_ENV` we set accordingly to the environment we want install the database for.

2. Uprgrading Secondhand with existing database 

If we have Secondhand running with an existing database, then we just run 

    $ bundle exec rake db:migrate RAILS_ENV="production" 

This only takes effect if we have changed our database tables.

3. We want to setup a backup server and replicate the existing production database 

In this scenario we copy the production database to the backup server and restore the database to MySQL. We configure MySQL to replicate the database from the production to the backup server's database. How replication is configured is described in [MySQL 8 failover](MySQL-8-failover.md).

Note: Scenario 3. is also valid for a staging and a beta server if we just want to have a database with data we can test with.

## Install Passenger

First we install the PGP key and add HTTPS support for APT

    sudo apt-get install -y dirmngr gnupg apt-transport-https ca-certificates curl

    curl https://oss-binaries.phusionpassenger.com/auto-software-signing-gpg-key.txt | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/phusion.gpg >/dev/null

Then we add the _passenger_ APT repository

    sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger noble main > /etc/apt/sources.list.d/passenger.list'

Now we run `sudo apt-get update` and install install the _passenger_ and _Apache_ module with

    sudo apt-get install -y libapache2-mod-passenger

### Configure Passenger for Apache 2

With the Passenger installation a _.conf_ and _.load_ file is provided and stored to _Apaches_'s module directory located at `/etc/apache2/mods-available`.

Next we have to make _Apache_ aware of the _passenger_ configuration files with `sudo a2enmod passenger` and then restart _Apache_ with `sudo apache2ctl restart`.

## Configure Apache 2

Even though we describe how to setup a backup server, it can be applied to a production, staging and beta server accordingly. A lengthy deployment walkthrough for the different scenarios can be found in [Deployment with rbenv](deployment-with-rbenv.md).

For the backup server we create a virtual host in `/etc/apache2/sites-available/secondhand-backup.conf`. In the virtual host we need the information 

* The application directory we have created in a previous step, which is in `/var/www/secondhand/backup/`. We have to point the document root to the `public` directory.
* The Ruby version that is running the application. This can be determined with following command. 

The `Command:` part is the one we are interested in

    $ cd /var/www/secondhand/backup 
    $ passenger-config about ruby-command
    passenger-config was invoked through the following Ruby interpreter:
    Command: /home/secondhand/.rbenv/versions/2.7.8/bin/ruby

* The environment we want to run the application in. In our case it is backup which is defined with `RackEnv backup`.

Our VirtualHost now looks like this 

    <VirtualHost *:8081>
      ServerName backup.secondhand.jupiter
    
      DocumentRoot /var/www/secondhand/backup/public
    
      PassengerRuby /home/secondhand/.rbenv/versions/2.7.8/bin/ruby
    
      <Directory /var/www/secondhand/backup/public/>
        AllowOverride all
        Options -MultiViews
        Order allow,deny
        Allow from all
        Require all granted
      </Directory>
      RackEnv backup
    </VirtualHost>

We need to enable the newly created virtual host with `a2ensite secondhand-backup.conf`. 

We also have to tell Apache 2 to listen on port 8081. We add `Listen 8081` to `/etc/apache2/ports.conf`that it looks like so

    Listen 8081
    Listen 80 

And restart _Apache_ with `sudo systemctl reload apache2.service` and `sudo apache2ctl restart`. 

# Checkup Secondhand 

Now we should be all set to run Secondhand backup with `http://backup.secondhand.jupiter`. If we have made any changes to Secondhand, we need to restart it with `touch /var/www/secondhand/backup/tmp/restart.txt`.

If the message _something went wrong_ is displayed. We can check the log files at 

* /var/www/secondhand/backup/log/backup.log 
* /var/log/mysql/error.log 

