####  U_FS_P5-LinuxServer

##  Introduction
This document is a step-by-step description for configuring a Linux Web Server, to host a complete web application whose source can be found 
[in this repository](https://github.com/Ramin8or/U_FS_P3-Catalog). 
This is the last project for Full-Stack Web Development Nanodegree from Udacity. 
The assignment specifications for configuring the server are provided by Udacity in 
[this link](https://docs.google.com/document/d/1J0gpbuSlcFa2IQScrTIqI6o3dice-9T7v8EDNjJDfUI/pub?embedded=true). 
Most of the material in this document are taken from the udacity course called
[Configuring Linux Web Servers](https://www.udacity.com/course/configuring-linux-web-servers--ud299). 
Links to additional material is provided where applicable.


##  Getting Started
The IP_ADDRESS assigned to the Virtual Machine Server for this project is: `52.33.77.87`. 
Steps to SSH to the Virtual Machine with a Udacity account were provided as follows:

*  On local host machine go to [your development environment](https://www.udacity.com/account#!/development_environment)
*  Download the private key file
*  Save the private key to your local .ssh directory and change access rights:
```
mv ~/Downloads/udacity_key.rsa ~/.ssh/
chmod 600 ~/.ssh/udacity_key.rsa
```
*  Remote login to the server VM using the private key:
```
ssh -i ~/.ssh/udacity_key.rsa root@IP_ADDRESS
```

##  Perform Basic Configuration

###  Create a new user named 'grader'
Given that at this point we are logged in as root into the VM, there is no need to preceed commands with `sudo`. Later when we setup the grader account
all commands will have `sudo` in front of them. Here's how to add a user called grader when logged in as the root account:
```
adduser grader
```

###  Give the 'grader' the permission to sudo
Create a file under `/etc/sudoers.d` with this content: ```grader ALL=(ALL) NOPASSWD:ALL``` and proper access rights. Use these commands:
```
echo "grader ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/grader
chmod 440 /etc/sudoers.d/grader
```

###  Update all currently installed packages
First update, then upgrade with the following two commands:
```
apt-get update
apt-get upgrade
```
**Note:** You will have to confirm “yes” when running the upgrade command.

###  Configure the local timezone to UTC
[From Ubuntu Community:](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28unattended.29)
```
echo "Etc/UTC" | sudo tee /etc/timezone
sudo dpkg-reconfigure --frontend noninteractive tzdata
```

##  Secure your Server

### Change the SSH port from 22 to 2200
First preserve factory defaults in a file as follows:
```
cp /etc/ssh/sshd_config /etc/ssh/sshd_config_defaults
chmod 444 /etc/ssh/sshd_config_defaults
```
I followed step 5 from [Digital Ocean Tutorials](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-12-04) 
to change the following lines in `/etc/ssh/sshd_config`:
```
nano /etc/ssh/sshd_config
```
Make the following changes:
```
Port 2200
AllowUsers grader
PasswordAuthentication yes
```
**Important note:** we're temporarily setting `PasswordAuthentication yes` to ensure that remote login works first.


Save the file, and restart SSH:
```
service ssh restart
```
**Do not logout of root yet!** Use another terminal to SSH using the grader account:
```
ssh -p 2200 grader@`[IP_ADDRESS](52.33.77.87)
```

###  Configure SSH key-based authentication
We're going to create a public/private pair of keys based on this tutorial from 
[DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server).


**On your local machine**, generate the keys under `~/.ssh` directory. Use `ssh-keygen`, and pass in a filename such as: 
`/user/USER_NAME/.ssh/u_p5_rsa.pub`.
```
ssh-keygen
```
Keep the generated keys on your local machine. Only the public key will be copied over to the VM, in grader's home directory under 
`~/.ssh/authorized_keys`. You can copy the public key from you local machine to the VM location specified above using the following single command:
```
cat ~/.ssh/u_p5_rsa.pub | ssh -p 2200 grader@IP_ADDRESS "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
Make sure to also set the proper access rights on the public key for the VM as follows:
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```
Change these lines back in `/etc/ssh/sshd_config` for the VM using `sudo nano /etc/ssh/sshd_config`:
```
PasswordAuthentication no
PermitRootLogin no
```
Finally restart the SSH service:
```
sudo service ssh restart
```	
Now you should be able to login to the server using: 
```	
ssh -p 2200 -i ~/.ssh/u_p5_rsa grader@IP_ADDRESS
```

###  Configure the Uncomplicated Firewall (UFW) 
The goal is to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123). I used the following commands
provided in [Ubuntu Community Docs](https://help.ubuntu.com/community/UFW). 
First I enabled ufw with the default rules. Then modified them as follows:
```
sudo ufw enable
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
```

## Install your application

###  Install Apache, Python, and PostgreSQL

####  Install and configure Apache to serve a Python mod_wsgi application
Install Apache web server:
```
sudo apt-get install apache2
```
Navigate to your [IP_ADDRESS](http://52.33.77.87) on your local machine to make sure the web server is serving. 
You should see 'It works!' on top of the web page.

According to [this Udacity blog](http://blog.udacity.com/2015/03/step-by-step-guide-install-lamp-linux-apache-mysql-python-ubuntu.html)
there is an additional step to ensure that Apache and Python play well together nicely: installing a tool called **mod_wsgi**. 
This is free tool for serving Python applications from Apache server. We will also be installing a helper package called **python-setuptools**.
```
sudo apt-get install python-setuptools libapache2-mod-wsgi
```
Now is the time to restart the Apache web server in order to load mod_wsgi:
```
sudo service apache2 restart
```
Use the information from [this answer](http://askubuntu.com/questions/256013/could-not-reliably-determine-the-servers-fully-qualified-domain-name) 
to eliminate the friendly warning when restarting Apache: 
```
sudo nano /etc/apache2/apache2.conf
```
And add the following at the end of this file:
```
# Introduce a servername to eliminate friendly warning
ServerName localhost
```

####  Install and configure PostgreSQL
I used [this tutorial from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
to install and secure PostgreSQL. First install Postgre:
```
sudo apt-get install postgresql postgresql-contrib
```
At this point, PostgreSQL has created a new user that you can use to connect to the database server as follows:
```
sudo su - postgres
psql
```
To exit:
```
\q
exit
```
Since we are installing the web server and database server on the same machine, there is no need to modify any firewall settings. 
The web server will communicate with the database via an internal mechanism that does not cross the boundaries of the firewall. 

##### Do not allow remote connections
By default remote connections are not allowed. You can verify that by opening this file:
```
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
```
and verifying that using [this article](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).

##### Create a new user named catalog that has limited permissions to your catalog application database
First create the new user:
```
sudo adduser catalog
```
Then switch to postgre user and connect to the database system:
```
sudo su - postgres
psql
```
Use the following commands to create catalog user and give it roles in the database system:
```
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
```
To see the affects of these commands type `\du` then exit and logout of the postgre account:
```
\q
exit
```

###  Install Git and clone the Catalog Web Application from GitHub
Here's how to install Git, and configure it with name and email information:
```
sudo apt-get install git
git config --global user.name "Ramin Halviatti"
git config --global user.email "ramin@outlook.com"
```
Now you can clone any project from a repository. For example:
```
git clone https://github.com/Ramin8or/U_FS_P3-Catalog.git
```
According to [this answer](http://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible)
you can make the GitHub repository inaccessible by placing a `.htaccess` at the root of your web server directory, e.g., 
`/var/www/catalog/.htaccess` and placing the following line in it: 
```
RedirectMatch 404 /\.git
```


###  Setup the Catalog App Project
Install git, clone and setup your Catalog App project (from your GitHub repository from earlier in the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser. Remember to set this up appropriately so that your .git directory is not publicly accessible via a browser!
