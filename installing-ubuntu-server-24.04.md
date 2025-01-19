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
2. Create an Secondhand application owner
3. Create the application directory
4. Install Git
5. Install rbenv
6. Install node.js
7. Deploy Secondhand
8. Install MySQL
9. Install Passenger
10. Configure Apache 2
11. Install Posfix
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

## Install rbenv to manage Ruby

For the description on how to install rbenv I want to refer to [install rbenv and ruby](installing-rbenv-and-ruby.md).

## Install node.js 

The next step needs `node.js` installed for the assets to run 

    $ sudo apt install nodejs 

## Clone Secondhand into the project directory 

Change into the project directory and clone the project from Git 

    $ sudo -u secondhand -H bash -l 
    $ cd /var/www/secondhand/backup/ 
    $ git clone https://github.com/sugaryourcoffee/secondhand.git 

Copy the `database.yml` and `secret_token.rb` from the development machine to the server

    devops@development $ scp database.yml secret_token.rb secondhand@jupiter:.

Back on the server 

    $ mv database.yml /var/www/secondhand/backup/config/.

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

    $ mv secret_token.rb /var/www/secondhand/backup/config/intializers/.

## Install MySQL

We have to install MySQL and create the database. Make sure not to only install
the _mysql-server_ but also the _libmysqlclient-dev_ otherwise bundler will fail
to install the _mysql2_ gem.

    $ sudo apt-get install mysql-server libmysqlclient-dev

Next we have to create a database

    devops@uranus $ sudo mysql -uroot -p 
    Enter password
    mysql> create database secondhand_production default character set utf8mb4 collate utf8mb4_0900_ai_ci;
    Query OK, 1 row affected (0.08 sec)
    mysql> create user 'secondhand'@'localhost' identified by 'password';
    Query OK, 0 rows affected (0.07 sec)
    mysql> grant all privileges on secondhand_production.* to 'secondhand'@'localhost';
    Query OK, 0 rows affected (0.06 sec)
    mysql> exit 
 
## Install Passenger

First we install the PGP key and add HTTPS support for APT

    sudo apt-get install -y dirmngr gnupg apt-transport-https ca-certificates curl

    curl https://oss-binaries.phusionpassenger.com/auto-software-signing-gpg-key.txt | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/phusion.gpg >/dev/null

Then we add the _passenger_ APT repository

    sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger noble main > /etc/apt/sources.list.d/passenger.list'

Now we run `sudo apt-get update` and install install the _passenger_ and _Apache_ module with

    sudo apt-get install -y libapache2-mod-passenger

With installation a _.conf_ and _.load_ file is provided and stored to _Apaches_'s module directory located at `/etc/apache2/mods-available`.

Next we have to make _Apache_ aware of the _passenger_ configuration files with `sudo a2enmod passenger` and then restart _Apache_ with `sudo apache2ctl restart`.

     $ gem install passenger
     $ passenger-install-apache2-module

There might be missing some modules and the installer will tell which are
missing and have to be installed.

    $ sudo apt-get install libcurl4-openssl-dev apache2-threaded-dev \
    libapr1-dev libaprutil1-dev

Even though the installer might tell you to install `apache2-threaded-dev` in
Ubuntu 16.04 you have to use `apache2-dev`.

After `passenger-install-apache2-module` has run you will be prompted with
following message:

    Please edit your Apache configuration file, and add these lines:
    LoadModule passenger_module \
    /home/pierre/.rvm/gems/ruby-2.0.0-p648@rails4013/gems/passenger-5.0.30\
    /buildout/apache2/mod_passenger.so
    <IfModule mod_passenger.c>
      PassengerRoot /home/pierre/.rvm/gems/ruby-2.0.0-p648@rails4013/gems\
      /passenger-5.0.30
      PassengerDefaultRuby /home/pierre/.rvm/gems/ruby-2.0.0-p648@rails4013\
      /wrappers/ruby
    </IfModule>
    
    After you restart Apache, you are ready to deploy any number of web
    applications on Apache, with a minimum amount of configuration!
    
    Press ENTER when you are done editing.

Add the above code snippet to `/etc/apache2/conf-available/passenger.conf` and 
run `$ sudo a2enconf passenger` and `service apache2 reload`.

## Configure Apache 2
Create a virtual host in `/etc/apache2/sites-available/secondhand.conf`

    <VirtualHost *:8086>
      DocumentRoot /var/www/secondhand/current/public
      Servername backup.secondhand.jupiter
      PassengerRuby /home/pierre/.rvm/gems/ruby-2.0.0-p648@rails4013/wrappers/\
      ruby 
      <Directory /var/www/secondhand/public>
        AllowOverride all
        Options -MultiViews
        Require all granted
      </Directory>
      RackEnv backup
    </VirtualHost>

Then run `$ sudo a2ensite secondhand` and `service apache2 reload`.

# Adjust the development environment
In this step we conduct following ajustments

1. Create a backup environment in `config/environments/backup.rb`
2. Add the _backup_ stage to `config/deploy.rb`
3. Create a backup deployment file in `config/deploy/backup.rb`
4. Add a _backup_ group to `config/database.yml`
5. Add the server hostname _backup.secondhand.jupiter_ to `/etc/hosts`

## Adjust `config/deploy/backup.rb`
Set the `rvm_ruby_string` to the gemset where we have installed rails

    set :rvm_ruby_string, '2.0.0@rails4013'

# Deploy to the backup server
During deployment the application is downloaded from Github. To download from
Github a rsa key is required that is known to Github. We use the key from our
development machine by copying it to the backup server

    $ scp ~/.ssh/id_rsa me@jupiter:key
    $ ssh jupiter
    $ mv key ~/.ssh/id_rsa

Next we start the ssh agent with `$ eval $(ssh-agent)` and issue `$ ssh-add`.
After entering the passphrase we should be good to deploy.

One final tweak is to ommit entering a passphrase when we deploy. To do that we
copy the public key from our development machine to our backup server

    $ scp ~/.ssh/id_rsa.pub me@jupiter:key
    $ ssh jupiter
    $ cat key >> ~/.ssh/authorized_keys

Back on the development resp. deployment machine we initially issue following 
commands to prepare deployment

    $ cap backup deploy:setup
    $ cap backup deploy:check
    $ cap backup deploy:cold

## Errors during deployment
Here are listed some errors that might come up during deployment. Some specific
to Ubunter 16.04 Server LTS.

### No connection to Github
If the deployment is canceled because of an credential issue try 
`ssh-add -l`. If it doesn't return a fingerprint but instead saying 
`The agent has no identities` then redo `ssh-add` but this time with the path to
you key `$ ssh-add .ssh/id_rsa`. Then check on the server wheather you can
connect to github with `$ ssh -T git@github.com`.

### MySQL gem cannot be installed
When capistrano is executing bunlder on the backup server an error might occur
saying

    * 2016-08-18 14:06:30 executing `bundle:install'                            
    * executing "cd /var/www/secondhand/releases/20160818120630 && bundle 
    install --gemfile /var/www/secondhand/releases/20160818120630/Gemfile 
    --path /var/www/secondhand/shared/bundle --deployment --quiet 
    --without development test"         
      servers: ["backup.secondhand.jupiter"]                                    
      [backup.secondhand.jupiter] executing command                             
    ** [out :: backup.secondhand.jupiter] An error occurred while installing 
    mysql2 (0.3.20), and Bundler cannot continue.                               
    ** [out :: backup.secondhand.jupiter] Make sure that 
   `gem install mysql2 -v '0.3.20'` succeeds before bundling.                   
      command finished in 12435ms                                               
    *** [deploy:update_code] rolling back

This most likely is because you are missing `libmysqlclient_dev` on the backup
server. To resolve this issue

    $ sudo apt-get install libmysqlclient-dev

### MySQL access denied
If the deployment is aborted during _deploy:migrate_ with the error 
`MySQL2::Error, Access denied for user` then you probably don't have created
the database.

### Mysql2::Error: All parts of a PRIMARY KEY must be NOT NULL
Ubuntu Server 16.04 LTS ships with MySQL 5.7.13. Since version 5.7 it is not 
allowed to have PRIMARY KEYs to be NULL. Here we are using Rails 4.0.13 and we
have to monkey patch the _Mysql2Adapter_ to fix this. In later versions 4.1 and
above this is fixed.

The error during deployment is showing

    * 2016-08-18 15:22:36 executing `deploy:migrate'                          
    * executing "cd /var/www/secondhand/releases/20160818132225 && bundle exec
    rake RAILS_ENV=backup  db:migrate"                                        
      servers: ["backup.secondhand.jupiter"]
      [backup.secondhand.jupiter] executing command
    *** [err :: backup.secondhand.jupiter] rake aborted!
    *** [err :: backup.secondhand.jupiter] StandardError: An error has occurred,
    all later migrations canceled:
    *** [err :: backup.secondhand.jupiter]
    *** [err :: backup.secondhand.jupiter] Mysql2::Error: All parts of a PRIMARY
    KEY must be NOT NULL; if you need NULL in a key, use UNIQUE 
    instead: CREATE TABLE `events` (`id` int(11) DEFAULT NULL auto_increment 
    PRIMARY KEY, `title` varchar(255), `event_date` datetime, `location` 
    varchar(255), `fee` decimal(2,2), `deduction` decimal(2,2), `provision` 
    decimal(2,2), `max_lists` int(11), `max_items_per_list` int(11), 
    `created_at` datetime, `updated_at` datetime) ENGINE=InnoDB 

The monkey patch is to be saved to `config/initializers/mysql_adapter`

    class ActiveRecord::ConnectionAdapters::Mysql2Adapter
      NATIVE_DATABASE_TYPES[:primary_key] = "int(11) auto_increment PRIMARY KEY"
    end

But this is not reliably working in all cases. If it doesn't consider upgrading 
to Rais 4.1 or using Ubuntu 14.04 Server.

# Configuring Failover
In case our secondhand master server mercury is crashing we want to point to our
secondhand backup server jupiter. We already installed secondhand on jupiter.
Next we need to start replicating the database from mercury. How this can be
accomplished is described in [MySQL-failover](https://github.com/sugaryourcoffee/secondhand/blob/master/doc/MySQL-failover.md).

