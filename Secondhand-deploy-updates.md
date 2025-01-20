Secondhand deploy updates 
=========================

## Environment 

Ruby 2.7.8
Rails 4.2.11 
Bundler 1.17.3 
Passenger 

## Deployment 

`ssh` into the server 

    $ ssh uranus 
    $ sudo -u secondhand -H bash -l

`cd` into the web directory of Secondhand production

    $ cd /var/www/secondhand/production

`pull` the new version from _Github_ 

    $ git pull 

check Ruby version to be 2.7.8 otherwise

    $ rbenv 2.7.8 

`bundle` install the required `gem`s without test and development

    $ bundle install --without test development

`precompile` the assets 

    $ bundle exec rake assets:precompile RAILS_ENV=production

`migrate` the database if you don't have made changes to the database, this won't do nothing 

    $ bundle exec rake db:migrate RAILS_ENV=production

`restart` _Secondhand_

    $ touch tmp/restart.txt 
