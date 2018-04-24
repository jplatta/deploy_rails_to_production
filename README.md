# Deploying Ruby on Rails Application to Production

## Description
This document is a synthesis (and not an explanation) of the steps for deploying a Ruby on Rails application with a PostgreSQL database to a Linux hosting environment using Capistrano.

## Assumptions
* Linux hosting env, e.g. Digital Ocean Droplet: Ubuntu 16.04.4 x64 
* Ruby on Rails application versioned using git
* Application uses a PostgreSQL database in production

## References 
* [DO - Deploy Rails](https://www.digitalocean.com/community/tutorials/deploying-a-rails-app-on-ubuntu-14-04-with-capistrano-nginx-and-puma)
* [DO - Setup PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04)
* [DO - Setup Rails with PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-use-postgresql-with-your-ruby-on-rails-application-on-ubuntu-14-04)
* [Medium - Deploy Rails](https://medium.com/ruby-on-rails-web-application-development/how-to-deploy-ruby-on-rails-apps-to-the-internet-production-staging-49efc503c91d)
* [Go Rails - Deploy Rails](https://gorails.com/deploy/ubuntu/16.04)
* [Ralf Ebert - Rails Deployment](https://www.ralfebert.de/tutorials/rails-deployment/)
* [Missing Secret Key Base](https://stackoverflow.com/questions/23180650/how-to-solve-error-missing-secret-key-base-for-production-environment-rai)
* [Env Variables in Rails](https://railsguides.net/how-to-define-environment-variables-in-rails/)
* [Capistrano and DatabaseYML](https://simonecarletti.com/blog/2009/06/capistrano-and-database-yml/)

## Hosting Env Setup: Create deploy user
Connect to your hosting env from your local machine:
```
~local$ ssh root@SERVER_IP
```
Accept the warning concerning authenticity and update root password when prompted (assuming this is your first time connecting with the root user).

Create a new user named "deploy":
```
root@server:~# adduser deploy
```
Add deploy to the sudo group to give deploy root privileges:
```
root@server:~# gpasswd -a deploy sudo
```

## Hosting Env Setup: Setup SSH for deploy user
Generate keys on your local machine: 
```
~local$ ssh-keygen
```
Enter a file name for the new key pair when prompted. Then either choose a passphrase or leave blank when prompted. Do not share your private key.

New keys stored in ssh directory:
```
~local$ cd ~/.ssh
```
Copy the public key to the hosting environment server using ssh-copy or manually copying the public key to the hosting env server:

Using ssh-copy: 
```
~local$ ssh-copy deploy@SERVER_IP
```
To manually copying the key, first print the public key on your local machine:
```
~local$ cat ~/.ssh/test_rsa.pub
```
The command should print something like:
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC2MyO5V4yE1WRIbGLJ4rBuFbs3Jp2SPGwkf7wQ6GbphB+rWdSrEDh9R0EEYOUlaa9gpwDSF+UlAtqPRJiqWGRoxPuR6JXy0n0yyYpMZjnjXxeCzO3YqvW+evWi99LsKySYdqS0jOWjGrD06fk9MV17FutXS123/VXZ/FYwApJ3k48safqKFvEpKj2ZiBCrF6SN6rAEHnoKlHQcib+S1e6OitCvxJYZLcjW5l+FewKAuukFKJrchRfq/JvD3XPpdbu2xaVbcOa6eyiKi2AMlvVqxrfjjM4WJgFtLVS4AQBa2CxN0E0HOZkoUfPNIZyAxC/5INBJr4cLkIj3U7EoiceT user@MacBook.local
```
Copy the printed statement to your clipboard. 

Back on the hosting env server, switch from the root user to the deploy user: 
```
root@server:~# su - deploy
```
Create an .ssh directory:
```
deploy@server:~$ mkdir ~/.ssh
```
Set the permission on the .ssh directory so only the owner can read, write, and execute: 
```
deploy@server:~$ chmod 700 ~/.ssh
```
Create an authorized keys file and open the file with a text editor:
```
deploy@server:~$ mk ~/.ssh/authorized_keys
deploy@server:~$ vi ~/.ssh/authorized_keys
```
Paste the public key into the authorized_keys file and save.

Set the permissions on the authorized_keys file so that only owner can read and write: 
```
deploy@server:~$ chmod 600 ~/.ssh/authorized_keys
```
Exit the deploy user: 
```
deploy@server:~$ exit
```
From your local machine test the ssh setup for the dpeloy user: 
```
~local$ ssh deploy@SERVER_IP
```
If the setup does not initially work, try starting the ssh-agent:
```
~local$ eval "$(ssh-agent -s)"
```
Then add the private key to the ssh agent:
```
~local$ ssh-add ~/.ssh/test_rsa
```
Try connecting to deploy again.

## Hosting Env Setup: Turn off SSH to root
Turn off remote SSH access to the root user.

First open the ssh config file:
```
root@server:~# vi etc/ssh/sshd_config
```
Find the line that looks like:
```
PermitRootLogin yes
```
Change that line to:
```
PermitRootLogin no
```
Save the file and restart the ssh service:
```
root@server:~# service ssh restart
```
Before logging off root, ssh using deploy to confirm that ssh with deploy is working:
```
~local$ ssh deploy@SERVER_IP
```
Log off the root user:
```
root@server:~# exit
```

## Hosting Env Setup: Setup Firewall

Ubuntu ships with Uncomplicated Firewall(ufw).

Log onto the deploy user:
```
~local$ ssh deploy@SERVER_IP
```

Create acceptions to the firewall policy for: 

SSH - port 22
```
deploy@server:~$ sudo ufw allow ssh
```
HTTP - port 80
```
deploy@server:~$ sudo ufw allow 80/tcp
```
SSL/TLS - port 443
```
deploy@server: $ sudo ufw allow 443/tcp
```
SMTP - port 25
```
deploy@server:~$ sudo ufw allow 25/tcp
```
Confirm acceptions:
```
deploy@server:~$ sudo ufw show added
```
Enable firewall and confirm:
```
deploy@server:~$ sudo ufw enable
```
If the firewall needs to be disabled, then execute:
```
deploy@server:~$ sudo ufw disable
```

## Hosting Env Setup: Setup Timezone and NTP Sync

Setup the server's timezone:
```
deploy@server:~$ sudo dpkg-reconfigure tzdata
```
Select appropriate values and confirm. 

Setup NTP Sync:
```
deploy@server:~$ sudo apt-get update
deploy@server:~$ sudo apt-get install ntp
```

## Hosting Env Setup: Setup Swap File
Skipping for now.

## Hosting Env Setup: Install Nginx

Update the package index file:
```
deploy@server:~$ sudo apt-get update
```
Install Nginx:
```
deploy@server:~$ sudo apt-get install curl git-core nginx -y
```

## Hosting Env Setup: Intall Database
Make sure the Postgresql gem is included in the application's Gemfile:
```ruby
gem 'pg', '0.18.4'
```
Install Postgresql
```
deploy@server:~$ sudo apt-get install postgresql postgresql-contrib
```
Installation creates the linux user account postgres.

Change to postgres user:
```
deploy@server:~$ sudo -i -u postgres
```
Create app_name database role:
```
postgres@server:~$ createuser -s app_name
```
Create a password for the new database role.

Get a postgresql command prompt:
```
postgres@server:~$ psql
```
The prompt should look like this:
```
postgres=#
```
Set the password for db role app_name:
```
postgres=# \password app_name
```
Follow the prompts to set the password.

Quit the postgresql command prompt:
```
postgres=# \q
```
Create database with the same name as the new db role:
```
postgres@server:~$ createdb app_name
```
To connect to the new database as the postgres user:
```
postgres@server:~$ psql -d app_name
```

## Host Env Setup: Ruby Version Manager

First import the RVM GPG Key:
```
deploy@server:~$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
```
Install RVM:
```
deploy@server:~$ curl -sSL https://get.rvm.io | bash -s stable
```
Load the RVM script as a function to start using it: 
```
deploy@server:~$ source ~/.rvm/scripts/rvm
```
Run the requirements command to install dependencies and files for RVM and Ruby:
```
deploy@server:~$ rvm requirements
```
To install a Ruby version:
```
deploy@server:~$ rvm install 2.4.0
```
To see which Ruby versions are installed:
```
deploy@server:~$ rvm list
```
To switch to a Ruby version:
```
deploy@server:~$ rvm use 2.4.0
```
To set a version of Ruby as the default:
```
deploy@server:~$ rvm use 2.4.0 --default
```

## Host Env Setup: Rails and Bundler

Install rails:
```
deploy@server:~$ gem install rails -v '5.0.7' -V --no-ri --no-rdoc
```
Install bundler:
```
deploy@server:~$ gem install bundler -V --no-ri --no-rdoc
```

## Host Env Setup: Setup SSH with Remote Git Repo

Shake hands with remote Git repo:
```
deploy@server:~$ ssh -T git@github.com
```
Create a new ssh key pair:
```
deploy@server:~$ ssh-keygen -t rsa
```
Save the the public key to the repository's deployment keys.

Test the ssh setup by cloning the repo to the hosting env:
```
deploy@server:~$ git clone git@github.com:jplatta/svn_explorer.git
```
If the clone succeeds, then ssh is setup correctly. Delete the cloned repo. 

## Setup Rails Application and Capistrano for Deployment

Include the following the application's Gemfile for intalling Capistrano and Puma:
```ruby
group :development do
    gem 'capistrano',         require: false
    gem 'capistrano-rvm',     require: false
    gem 'capistrano-rails',   require: false
    gem 'capistrano-bundler', require: false
    gem 'capistrano3-puma',   require: false
end

gem 'puma'
```
Run bundle install:
```
~local$ bundle install
```
Install Capistrano:
```
~local$ cap install
```
Update the Capfile in the root directory of the Rails app with:
```ruby
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/rails'
require 'capistrano/bundler'
require 'capistrano/rvm'
require 'capistrano/puma'
install_plugin Capistrano::Puma

# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```
Update the config/deploy.rb file with:
```ruby
server 'your_server_ip', port: 'your_port_number', roles: [:web, :app, :db], primary: true

set :repo_url,        'git@github.com:jplatta/svn_explorer.git'
set :application,     'svn_explorer'
set :user,            'deploy'
set :puma_threads,    [4, 16]
set :puma_workers,    0

# Don't change these unless you know what you're doing
set :pty,             true
set :use_sudo,        false
set :stage,           :production
set :deploy_via,      :remote_cache
set :deploy_to,       "/home/#{fetch(:user)}/apps/#{fetch(:application)}"
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.error.log"
set :puma_error_log,  "#{release_path}/log/puma.access.log"
set :ssh_options,     { forward_agent: true, user: fetch(:user), keys: %w(~/.ssh/id_rsa.pub) }
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true  # Change to false when not using ActiveRecord

## Defaults:
# set :scm,           :git
# set :branch,        :master
# set :format,        :pretty
# set :log_level,     :debug
# set :keep_releases, 5

## Linked Files & Directories (Default None):
# set :linked_files, %w{config/database.yml}
# set :linked_dirs,  %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}

namespace :puma do
  desc 'Create Directories for Puma Pids and Socket'
  task :make_dirs do
    on roles(:app) do
      execute "mkdir #{shared_path}/tmp/sockets -p"
      execute "mkdir #{shared_path}/tmp/pids -p"
    end
  end

  before :start, :make_dirs
end

namespace :deploy do
  desc "Make sure local git is in sync with remote."
  task :check_revision do
    on roles(:app) do
      unless `git rev-parse HEAD` == `git rev-parse origin/master`
        puts "WARNING: HEAD is not the same as origin/master"
        puts "Run `git push` to sync changes."
        exit
      end
    end
  end

  desc 'Initial Deploy'
  task :initial do
    on roles(:app) do
      before 'deploy:restart', 'puma:start'
      invoke 'deploy'
    end
  end

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      invoke 'puma:restart'
    end
  end
  before 'deploy:assets:precompile', :symlink_config_files

  desc "Link shared files"

  task :symlink_config_files do
    symlinks = {"#{shared_path}/config/database.yml" => "#{release_path}/config/database.yml"}
    on roles(:app) do
      execute symlinks.map{|from, to| "ln -nfs #{from} #{to}"}.join(" && ")
    end
  end

  before :starting,     :check_revision
  after  :finishing,    :compile_assets
  after  :finishing,    :cleanup
  after  :finishing,    :restart
end

# ps aux | grep puma    # Get puma pid
# kill -s SIGUSR2 pid   # Restart puma
# kill -s SIGTERM pid   # Stop puma
```
Note the following code in the config/deploy.rb file:
```ruby
desc "Link shared files"

  task :symlink_config_files do
    symlinks = {"#{shared_path}/config/database.yml" => "#{release_path}/config/database.yml"}
    on roles(:app) do
      execute symlinks.map{|from, to| "ln -nfs #{from} #{to}"}.join(" && ")
    end
  end
```
This code creates symlinks for storing database credentials in a shared folder on the hosting environment. 

In order for this to work, the config/database.yml needs to be added to the .gitignore file on your local machine.

Next configure Nginx in config/nginx.conf:
```conf
upstream puma {
  server unix:///home/deploy/apps/svn_explorer/shared/tmp/sockets/svn_explorer-puma.sock;
}

server {
  listen 80 default_server deferred;
  # server_name example.com;

  root /home/deploy/apps/svn_explorer/current/public;
  access_log /home/deploy/apps/svn_explorer/current/log/nginx.access.log;
  error_log /home/deploy/apps/svn_explorer/current/log/nginx.error.log info;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @puma;
  location @puma {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;

    proxy_pass http://puma;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 10M;
  keepalive_timeout 10;
}
```
Update the application's config/secrets.yml to have production get the secret key base from the hosting environment's variables.
```ruby
# Do not keep production secrets in the repository,
# instead read values from the environment.
production:
  secret_key_base: ENV["SECRET_KEY_BASE"]
```

## First Deploy of Application

Commit changes to git repository:
```
~local$ git add -A
~local$ git commit -m "setup for nginx, puma, and capistrano"
~local$ git push
```
Run Capistrano deployment. If first deploy, then run:
```
~local$ cap production deploy:initial
```
If the deploy succeeds, ssh to the hosting enviornment. 

Get the secret key base on the production by navigating to the root folder of the current deployed version of the application:
```
deploy@server:~$ cd apps/svn_explorer/current
```
Run the following command to get a secret key base:
```
deploy@server:~/apps/svn_explorer/current$ RAILS_ENV=production rake secret
```
The command should print something like the following:
```
61a9f1d9e11bd835f00473a776f9ffa9afc8c8e3ab127a70fb618b964ec36f1ffa9e53c76d7d46e012508d0160d368fa95ef993d53c53e027d61b4086abb29bf
```
Copy the string and then open up the profile file:
```
deploy@server:/$ sudo vi /etc/profile
```
Add the SECREY_KEY_BASE variable to the end of the file and save:
```
export SECRET_KEY_BASE=61a9f1d9e11bd835f00473a776f9ffa9afc8c8e3ab127a70fb618b964ec36f1ffa9e53c76d7d46e012508d0160d368fa95ef993d53c53e027d61b4086abb29bf
```
Check that the SECRET_KEY_BASE is successfully set:
```
deploy@server:/$ printenv | grep SECRET_KEY_BASE
```
Next update the production database credentials in the shared/config/database.yml. Navigate to:
```
deploy@server:~$ sudo vi /apps/svn_explorer/shared/config/database.yml
```
Add your production database credentials:
```ruby
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem 'sqlite3'
#
default: &default
  adapter: postgresql
  pool: 5
  timeout: 5000

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  adapter: postgresql
  host: localhost
  port: 5432
  encoding: unicode
  database: svn_ruby_test
  pool: 5
  username:
  password:

production:
  adapter: postgresql
  host: localhost
  database: svn_explorer
  pool: 5
  username: svn_explorer
  password: password
```
Restart Nginx:
```
eploy@server:~$ sudo service nginx restart
```

## Subsequent Deploys

Push changes to git repo:
```
~local$ git add -A
~local$ git commit -m "pushing recent changes"
~local$ git push
```
Run Capistrano deployment from the application's root directory on your local machine:
```
~local$ cap production deploy
```
If deploy fails, then check deployment logs:
If application is inaccessible after deploy, then check the puma logs:
