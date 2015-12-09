####  U_FS_P5-LinuxServer

##  Introduction
This document describes steps required to configure a Linux Web Server, to host a complete web application, 
running on a virtual server in Amazon’s Elastic Compute Cloud (EC2). The web application, the Catalog App, is hosted 
on an Apache HTTP server running on Ubuntu.
The database server is PostgreSQL running on the same VM. 
The Catalog App is written in Python and requires Flask framework. The sources for Catalog App can be found 
[in this repository](https://github.com/Ramin8or/U_FS_P3-Catalog).

Extra effort has been put in place to make all steps repeatable, using command lines, rather than editing and cut/paste.
With a little more effort, the content of this document can turn into a shell script to stamp out the virtual Sever on a drop of a hat.

This is the last project for Full-Stack Web Development Nanodegree from Udacity. 
The assignment specifications for configuring the server are provided by Udacity in 
[this link](https://docs.google.com/document/d/1J0gpbuSlcFa2IQScrTIqI6o3dice-9T7v8EDNjJDfUI/pub?embedded=true). 
Most of the material in this document are taken from the udacity course called
[Configuring Linux Web Servers](https://www.udacity.com/course/configuring-linux-web-servers--ud299). 
Links to additional material is provided where applicable.


##  Getting Started
*  The IP_ADDRESS assigned to the Virtual Machine Server for this project is: `52.33.77.87`. 
*  The complete URL to the hosted web application: 
[http://ec2-52-33-77-87.us-west-2.compute.amazonaws.com/](http://ec2-52-33-77-87.us-west-2.compute.amazonaws.com/)
*  SSH port (additional information will be in reviewer notes)
```
ssh -p 2200 -i ~/.ssh/u_p5_rsa  grader@52.33.77.87
```

##  Perform Basic Configuration

####  Create a new user named 'grader'
At this point we are logged into the VM as root. So there is no need to preceed commands with `sudo`:
```
adduser grader
```
####  Give the 'grader' the permission to sudo
Create a file under `/etc/sudoers.d` with this content: ```grader ALL=(ALL) NOPASSWD:ALL``` and proper access rights. Use these commands:
```
echo "grader ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/grader
chmod 440 /etc/sudoers.d/grader
```
####  Update all currently installed packages
First update, then upgrade with the following two commands:
```
apt-get update
apt-get upgrade
```
**Note:** You will have to confirm “yes” when running the upgrade command.
To automatically install stable unattended upgrades follow these steps as well:
```
apt-get install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```
####  Configure the local timezone to UTC
[From Ubuntu Community:](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28unattended.29)
```
echo "Etc/UTC" | sudo tee /etc/timezone
sudo dpkg-reconfigure --frontend noninteractive tzdata
```
##  Secure your Server

#### Change the SSH port from 22 to 2200 (a non-default port)
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
ssh -p 2200 grader@IP_ADDRESS
```
####  Configure SSH key-based authentication
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
Change these lines back in `/etc/ssh/sshd_config` for the VM using `sudo nano`:
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
####  Configure the Uncomplicated Firewall (UFW) 
The goal is to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123). I used the following commands
provided in [Ubuntu Community Docs](https://help.ubuntu.com/community/UFW). 
First I enabled ufw with the default rules. Then modified them as follows:
```
sudo ufw enable
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
```
####  Monitor for repeated attempts to login
[This tutorial from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
provides the step by step guide to prevent repeated SSH using  
[fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page). 
First install the necessary packages, and copy the config file to a local copy:
```
sudo apt-get install sendmail iptables-persistent
sudo apt-get install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Open the local config file: `/etc/fail2ban/jail.local`. 

Set batime to `bantime = 1800`

Under section `[ssh]` set `port = 2200`

Change `destmail =` line to where mail should be sent

Change `action = ` line to `action = %(action_mwl)s`  
Restart fail2ban service:
```
sudo service fail2ban stop
sudo service fail2ban start
```

##  Install your application
The application is a Python Flask Web Application which will use a PostgreSQL database. First we will install Apache HTTP server. Then we will configure it to serve Python Flask Applications. Next we'll setup the PostgreSQL database server. Finally, we'll adapt the Catalog Application to run on the full stack running on our Linux server.

###  Install and configure Apache to serve a Python mod_wsgi application
Install Apache web server:
```
sudo apt-get install apache2
```
Navigate to your [IP_ADDRESS](http://52.33.77.87) on your local machine to make sure the web server is serving. 
You should see 'It works!' on top of the web page.

According to [this Udacity blog](http://blog.udacity.com/2015/03/step-by-step-guide-install-lamp-linux-apache-mysql-python-ubuntu.html)
there is an additional step to ensure that Apache and Python play well together nicely: installing a tool called **mod_wsgi**. 
This is free tool for serving Python applications from Apache server. We will also be installing Flask and other helper packages 
that will be required to setup our Catalog Application later:
```
sudo apt-get install python-setuptools libapache2-mod-wsgi python-dev
sudo apt-get install libpq-dev
sudo a2enmod wsgi
sudo apt-get install python-pip 
sudo pip install Flask
sudo service apache2 restart
```
Couple of tricks to quiet down annoying "friendly warnings." First, I used information from 
[this answer](http://askubuntu.com/questions/256013/could-not-reliably-determine-the-servers-fully-qualified-domain-name) 
to eliminate the friendly warning when restarting Apache. Here's the command I came up with to simply append a ServerName to the end of the Apache Config:
```
echo "ServerName localhost" | sudo tee -a /etc/apache2/apache2.conf
```
Also, 
based on [this answer](http://askubuntu.com/questions/59458/error-message-when-i-run-sudo-unable-to-resolve-host-none)
the top of the `/etc/hosts` file should include this information:
```
127.0.0.1    localhost.localdomain localhost
127.0.1.1    my-machine
```
where my-machine is the name found inside `/etc/hostname`.

###  Install and configure PostgreSQL
I used [this tutorial from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
to install and secure PostgreSQL. First install Postgre:
```
sudo apt-get install postgresql postgresql-contrib
```
PostgreSQL creates a new user that you can use to connect to the database server as follows:
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

####  Do not allow remote connections
By default remote connections are not allowed. You can verify that by opening this file: 
`/etc/postgresql/9.3/main/pg_hba.conf`
and verifying it using 
[this article](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).

####  Create a new user named catalog with limited permissions to database
First create the Linux new user:
```
sudo adduser catalog
```
Then switch to postgre user and connect to the database system:
```
sudo su - postgres
psql
```
Use the following commands to create catalog user with appropriate roles and create the catalog database:
```
CREATE USER catalog WITH PASSWORD 'catalog';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
```
Finally, exit and logout of the postgre account:
```
\q
exit
```
###  Install Git and clone the Catalog Web Application from GitHub
Here's how to install Git, and configure it with name and email information:
```
sudo apt-get install git
git config --global user.name "Firstname Lastname"
git config --global user.email "EmailId@somewhere.com"
```
Now you can clone any project from a repository. For example:
```
git clone https://github.com/Ramin8or/U_FS_P3-Catalog.git
```
Setup the directory structure where the Catalog Application will be served from:
```
sudo mkdir /var/www/catalog
sudo mkdir /var/www/catalog/catalog
```
According to [this answer](http://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible)
you can make the GitHub repository inaccessible by placing a `.htaccess` at the root of your web server directory. 
Here's how to do that using the command line:
```
sudo touch /var/www/catalog/.htaccess
echo "RedirectMatch 404 /\.git" | sudo tee /var/www/catalog/.htaccess
```
###  Setup the Catalog App Project
Given that the Catalog App is a Flask application, 
I followed 
[this tutorial from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
to deploy a flask application on Ubuntu. Note that we already setup Apache with mod_wsgi above. 
This section provides all the steps involved to setup the Catalog Application.

**Note:** The following image packages are required to use `python-resize-image`, and must be installed first:
```
sudo apt-get build-dep python-imaging
sudo apt-get install libjpeg8 libjpeg62-dev libfreetype6 libfreetype6-dev
sudo pip install python-resize-image
```
####  Create a Python Virtualenv Environment:
Using the following commands we will create a 
[Python virtualenv](http://docs.python-guide.org/en/latest/dev/virtualenvs/).
and install the necessary modules in our virtual evironment:

```
cd /var/www/catalog/catalog
sudo pip install virtualenv
sudo virtualenv venv
sudo chmod -R 777 venv
```
####  Install necessary modules for the Catalog Application 
These will be installed within the Python Virtual Environment venv created earlier:
```
source venv/bin/activate
pip install httplib2
pip install requests
pip install oauth2client
pip install sqlalchemy
pip install psycopg2
```
####  Create a virtual host config file:
```
sudo nano /etc/apache2/sites-available/catalog.conf
```
Insert the following lines in the file (replace with your specific information):
```
<VirtualHost *:80>
	ServerName 52.33.77.87  
	ServerAlias ec2-52-33-77-87.us-west-2.compute.amazonaws.com
	ServerAdmin ramin@outlook.com
	WSGIScriptAlias / /var/www/catalog/catalog.wsgi
	<Directory /var/www/catalog/catalog/>
		Order allow,deny
		Allow from all
 	</Directory>
	Alias /static /var/www/catalog/catalog/static
	<Directory /var/www/catalog/catalog/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Now create the wsgi file by pasting the following lines in a file called `/var/www/catalog/catalog.wsgi`:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
```
Enable Apache Virtual Host for catalog and restart: 
```
sudo a2ensite catalog
sudo service apache2 restart
```
####  Copy Catalog App over to the HTTP server directory
```
mv project.py __init__.py
sudo cp -r ~/U_FS_P3-Catalog/vagrant/catalog/* /var/www/catalog/catalog/
```
####  Setup database tables and populate them
First make sure the all *.py files use PostgreSQL in their call to `create_engine()` (see next section).
```
cd /var/www/catalog/catalog/
python database_setup.py
python populate_db.py
```
At this point the Catalog App is positioned to run. Try the app and correct any errors by looking at the error logs:
```
sudo tail -50 /var/log/apache2/error.log
```
To populate the database with 50,000 (or more) items, follow these steps:
```
cd /var/www/catalog/catalog/
sudo cp pix/*jpg static/
python more_items.py
```
The next section describes the changes I had to make to get the Catalog App fully functional.

####  Necessary changes to the Catalog Application sources
1. `create_engine()` usage should reflect the fact that PostgreSQL is being used
Change to this line in database_setup.py, project.py, and populate.py:
```
engine = create_engine('postgresql://catalog:CATALOG_PASSWORD@localhost/catalog')
```
2. Replace all relative paths with absolute paths:
```
CLIENT_ID = json.loads(
    open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']

UPLOAD_FOLDER = '/var/www/catalog/catalog/static/'
```
3. The main application at the the bottom of project.py file should look like this:
```
# Main application
app.secret_key = 'YOUR SECRET'
app.debug = True
```
4. Google authorization changes
Go Developer Console: `https://console.developers.google.com` and find the Catalog Application Project Credentials.
Under Authorized JavaScript origins, add your IP address and hostname URL:
```
http://52.33.77.87
http://ec2-52-33-77-87.us-west-2.compute.amazonaws.com
```
Under Authorized redirect URIs
```
http://ec2-52-33-77-87.us-west-2.compute.amazonaws.com/oauth2callback
```

###  Use [Glances](https://pypi.python.org/pypi/Glances) to monitor status of the server
Follow these steps. Note that other required python packages have already been installed in previous steps:
```
sudo apt-get install build-essential
sudo apt-get install lm-sensors
sudo pip install Glances
sudo pip install PySensors
```
You can type `glances` on the command line to monitor the server status at this point.
