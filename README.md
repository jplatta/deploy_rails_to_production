# deploy_rails_to_production

## Description
This document is a synthesis (and not an explanation) of the steps for deploying a Ruby on Rails application using Capistrano.

## Assumptions
* Linux hosting env, e.g. Digital Ocean Droplet: Ubuntu 16.04.4 x64 
* Ruby on Rails application versioned on Github
* Application uses a PostgreSQL database in production

## References 
* Ref 1
  * Ref 1 - sub 
  * Ref 1 - sub
* Ref 2
* Ref 3

## Hosting Env Setup: Create deploy user
Connect to your hosting env from your local machine:
```
local$ ssh root@SERVER_IP
```
Accept the warning concerning authenticity and update root password when prompted (assuming this is your first time connecting with the root user).

Create a new user named "deploy":
```
root: # adduser deploy
```
Add deploy to the sudo group to give deploy root privileges:
```
root: # gpasswd -a deploy sudo
```
## Hosting Env Setup: Setup SSH for deploy user
Generate keys on your local machine: 
```
local$ ssh-keygen
```
Enter a file name for the new key pair when prompted. Then either choose a passphrase or leave blank when prompted. Do not share your private key.

New keys stored in ssh folder:
```
local$ cd ~/.ssh
```
Copy the public key to the hosting env server using ssh-copy or manually copying the public key to the hosting env server:

Using ssh-copy: 
```
local$ ssh-copy deploy@SERVER_IP
```
To manually copying the key, first print the public key on your local machine:
```
local$ cat ~/.ssh/test_rsa.pub
```
The command should print something like:
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC2MyO5V4yE1WRIbGLJ4rBuFbs3Jp2SPGwkf7wQ6GbphB+rWdSrEDh9R0EEYOUlaa9gpwDSF+UlAtqPRJiqWGRoxPuR6JXy0n0yyYpMZjnjXxeCzO3YqvW+evWi99LsKySYdqS0jOWjGrD06fk9MV17FutXS123/VXZ/FYwApJ3k48safqKFvEpKj2ZiBCrF6SN6rAEHnoKlHQcib+S1e6OitCvxJYZLcjW5l+FewKAuukFKJrchRfq/JvD3XPpdbu2xaVbcOa6eyiKi2AMlvVqxrfjjM4WJgFtLVS4AQBa2CxN0E0HOZkoUfPNIZyAxC/5INBJr4cLkIj3U7EoiceT user@MacBook.local
```
Copy the printed statement to your clipboard. 

Back on the hosting env server, switch from the root user to the deploy user: 
```
root: # su - deploy
```
Create an .ssh directory:
```
deploy: $ mkdir ~/.ssh
```
Set the permission on teh .ssh directory so only the owner can read, write, and execute: 
```
deploy: $ chmod 700 ~/.ssh
```
Create an authorized keys file and open the file with a text editor:
```
deploy: $ mk ~/.ssh/authorized_keys
deploy: $ vi ~/.ssh/authorized_keys
```
Paste the public key into the authorized_keys file and save.

Set the permissions on the authorized_keys file so that only owner can read and write: 
```
deploy: $ chmod 600 ~/.ssh/authorized_keys
```
Exit the deploy user: 
```
deploy: $ exit
```
From your local machine test the ssh setup for the dpeloy user: 
```
local$ ssh deploy@SERVER_IP
```
If the setup does not initially work, try starting the ssh-agent:
```
local$ eval "$(ssh-agent -s)"
```
Then add the private key to the ssh agent:
```
local$ ssh-add ~/.ssh/test_rsa
```
Try connecting to deploy again.

## Hosting Env Setup: Turn off SSH to root
Turn off remote SSH access to the root user.

First open the ssh config file:
```
root: # vi etc/ssh/sshd_config
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
root: # service ssh restart
```
Before logging off root, ssh using deploy to confirm that ssh with deploy is working:
```
local$ ssh deploy@SERVER_IP
```
Log off the root user:
```
root: # exit
```
## Hosting Env Setup: Add Acceptions to Firewall

## Hosting Env Setup: Enable Firewall 

## Hosting Env Setup: 

## Database Setup: Install PostgreSQL

## Database Setup: Create app_name role on db

## Database Setup:

## Hosting Env Setup: Set Env variables for db

## Capistrano Setup:

## Run Deploy Initial Process:

## Future Deploys
