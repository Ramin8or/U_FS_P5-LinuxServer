####  U_FS_P5-LinuxServer

##  Introduction
This document is a step-by-step description for configuring a Linux server, to host a complete web application whose source can be found 
[in this repository](https://github.com/Ramin8or/U_FS_P3-Catalog). 
This is the last project for Full-Stack Web Development Nanodegree from Udacity. 
The assignment specifications for configuring the server are provided by Udacity in 
[this link](https://docs.google.com/document/d/1J0gpbuSlcFa2IQScrTIqI6o3dice-9T7v8EDNjJDfUI/pub?embedded=true). 
Most of the material in this document are taken from [this udacity course](https://www.udacity.com/course/configuring-linux-web-servers--ud299). 
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
ssh -i ~/.ssh/udacity_key.rsa root@[IP_ADDRESS](http://52.33.77.87)
```

##  Perform Basic Configuration

###  Create a new user named 'grader'
Given that at this point we are logged in as root into the VM, there is no need to preceed commands with sudo. Later when we setup the grader account it is required to preceed commands with ```sudo```. Here's how to add a user called grader:

`adduser grader`

###  Give the grader the permission to sudo
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

Note that we're temporarily setting `PasswordAuthentication` to `yes` to ensure that remote login works first.
Save the file, and restart SSH:

`service ssh restart`

**Do not logout of root yet, use another terminal to SSH using the grader account:**

`ssh -p 2200 grader@`[IP_ADDRESS](52.33.77.87)

###  SSH
We're going to create a public/private pair of keys based on 
[this tutorial from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
On your local machine, generate the key pair under `~/.ssh`. Call `ssh-keygen` and pass in a filename such as: `/user/USER_NAME/.ssh/u_p5_rsa.pub`:

```
ssh-keygen
```

Keep the generated keys on your local machine. Copy the public key only to the VM. Store it in grader's home directory under 
`~/.ssh/authorized_keys`. You can do that using the following single command:

```
cat ~/.ssh/u_p5_rsa.pub | ssh -p 2200 grader@52.33.77.87 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Make sure to also set the proper access rights on the public key for the VM as follows:

```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

Now change these lines back in `/etc/ssh/sshd_config` for the VM using `sudo nano /etc/ssh/sshd_config`:

```
PasswordAuthentication no
PermitRootLogin no
```

Now restart the SSH service:

```
sudo service ssh restart
```	

Now you should be able to login to the server using: 

```	
ssh -p 2200 -i ~/.ssh/u_p5_rsa grader@IP_ADDRESS
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
