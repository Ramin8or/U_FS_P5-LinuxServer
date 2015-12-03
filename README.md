####  U_FS_P5-LinuxServer

##  Introduction
This document is a step-by-step description for configuring a Linux server to host [this web application](https://github.com/Ramin8or/U_FS_P3-Catalog). 
The specifications for configuring the server are available in [this link](https://docs.google.com/document/d/1J0gpbuSlcFa2IQScrTIqI6o3dice-9T7v8EDNjJDfUI/pub?embedded=true).

##  Getting Started
[this udacity course](https://www.udacity.com/course/configuring-linux-web-servers--ud299)
Launch your Virtual Machine with your Udacity account. Follow the instructions provided to SSH into your server:

On local host machine
Go to [your development environment](https://www.udacity.com/account#!/development_environment)

* Download the private key file: Download
* Save the private key to your local .ssh directory and change access rights:
* mv ~/Downloads/udacity_key.rsa ~/.ssh/
* chmod 600 ~/.ssh/udacity_key.rsa
* Remote login to the server VM using the private key:
* ssh -i ~/.ssh/udacity_key.rsa root@52.33.77.87


##  Perform Basic Configuration

###  Create a new user named grader
Given that at this point we are logged in as root we can simply add user grader:

`adduser grader`

###  Give the grader the permission to sudo
Create a file under `/etc/sudoers.d` with the following line as its content:
```
grader ALL=(ALL) NOPASSWD:ALL
Use these commands to do this:
echo "grader ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/graderchmod 440 /etc/sudoers.d/grader
```

###  Update all currently installed packages
```
apt-get update
apt-get upgrade
```

###  Configure the local timezone to UTC
[From this link](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28unattended.29)
```
echo "Etc/UTC" | sudo tee /etc/timezone
sudo dpkg-reconfigure --frontend noninteractive tzdata
```

##  Secure your Server

### Change the SSH port from 22 to 2200
First preserve factory default file:
```
cp /etc/ssh/sshd_config /etc/ssh/sshd_config_defaults
chmod 444 /etc/ssh/sshd_config_defaults
```
I followed step 5 from [this link](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-12-04) to change these lines in file `/etc/ssh/sshd_config`:
```
nano /etc/ssh/sshd_configPort 2200
AllowUsers grader
PasswordAuthentication yes
```
Note that we're temporarily setting password authentication to yes to make sure remote login works.

Save the file, and restart ssh:

`service ssh restart`

Do not logout of root yet, use another terminal to SSH using the grader account:

`ssh -p 2200 grader@52.33.77.87`

###  SSH
Now create a public private pair of keys to use for: check out [this link](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)

```
ssh-keygen
```
Store the private key under ~/.ssh, for example pass in this file: ~/.ssh/u_p5_rsa.pub (/user/…/…)
```
cat ~/.ssh/u_p5_rsa.pub | ssh -p 2200 grader@52.33.77.87 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

Now change these back in server VM's /etc/ssh/sshd_config:
```	
sudo nano /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no
sudo service ssh restart
```	

Now you should be able to login to the server using:

```	
ssh -p 2200 -i ~/.ssh/u_p5_rsa grader@52.33.77.87
```

###  UFW
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

## Install your application

###  Install Apache, Python, and PostgreSQL
Install and configure Apache to serve a Python mod_wsgi application
Install and configure PostgreSQL:
Do not allow remote connections
Create a new user named catalog that has limited permissions to your catalog application database

###  Install Git

###  Setup the Catalog App Project
Install git, clone and setup your Catalog App project (from your GitHub repository from earlier in the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser. Remember to set this up appropriately so that your .git directory is not publicly accessible via a browser!
