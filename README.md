# Udacity - Linux server configuration project

## Description

Goals of the project given by our instructors from Udacity:

> You will take a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

The application meant to be deployed is the **Item catalog app**.

## Server Info

IP address: 18.216.86.137

Accessible SSH port: 2200.

Application URL: http://ec2-18-216-86-137.us-east-2.compute.amazonaws.com

## Steps taken

### Step 1 - Create a user named grader and give grader sudo permissions.

1. Log into remote VM as the *root* user: '$ ssh root@18.216.86.137'.
2. Add a user *grader*: `$ sudo adduser grader`.
3. Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.

### Step 2 - Update and upgrade the installed packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.
3. Also, install finger to keep track of your users' status: `$ apt-get install finger`.

### Step 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.

### Step 4 - Configure the key-based authentication for *grader* user

(I struggled a lot with this part personally, so I have some notes that I think will help!)

1. Since we're using Lightsail, download the default key from the Lightsail server and save it somewhere easy to find. ( I saved mine in the .ssh folder)
2. Log into the remote VM as *root* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
3. Copy the content of the default key from Lightsail to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
4. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa grader@18.216.86.137'.

### Step 5 - Enforce key-based authentication
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 6 - Add the SSH port 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and put *2200* under 22. (I decdied to do this because my machine kept locking me out after I got rid of port 22)
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@18.216.86.137`.

### 7 - Disable ssh login for *root* user
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PermitRootLogin* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 8 - Configure the Uncomplicated Firewall (UFW)

1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.

## Setting up Flask

> For this, I used the digital ocean tutorial found here: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
> I found it extremely helpful, but it does leave some bits out, so I recommend checking out my thread here too: https://discussions.udacity.com/t/issues-deploying-application-to-lightsail/403522

### Step 9 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
3. `$ sudo service apache2 start`.

### Step 10 - Install Git

1. `$ sudo apt-get install git`.

### Step 11 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir FlaskApp`.
2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader FlaskApp`.
3. Move inside that newly created folder: `$ cd /FlaskApp` and clone the item catalog repository from Github: `$ git clone https://github.com/adinsxx/fullstack-nanodegree-vm/tree/master/vagrant.git FlaskApp`.
4. The structure should look like this: 
|----FlaskApp
|---------FlaskApp
|--------------fullstack-nanodegree-vm

5. Add a __init__.py file that will contain the flask app logic. I used mine to take the place of my project.py file. 
(NOTE: CHECK YOUR INDENTATION, THIS WILL SAVE YOU A LOT OF HEARTACHE)

6. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
```

### Step 12 - Install Flask and the project's dependencies

1. Install *pip*, the tool for installing Python packages: `$ sudo apt-get install python-pip`.
2. Install Flask: `$ pip install Flask`.
3. Install all the other project's dependencies: `$ pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2`. 

### Step 13 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/catalog.conf`.
2. Paste in the following lines of code:
```
<VirtualHost *:80>
    ServerName 52.34.208.247
    ServerAlias ec2-52-34-208-247.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.34.208.247
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
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

3. Enable the new virtual host: `$ sudo a2ensite catalog`.

### Step 14 - Install and configure PostgreSQL
I followed this tutorial to the line in order to get my database set up. Make sure that you're calling the proper database in your
database_setup.py, project.py, and items.py files!!
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04

### 15 - Update OAuth authorized JavaScript origins

1. To let users correctly log-in change the authorized URI to http://ec2-18-216-86-137.us-east-2.compute.amazonaws.com on both Google and Facebook developer dashboards.

### 16 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.

## Sources

For flask deploying help, check out this link: 
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
http://flask.pocoo.org/docs/0.12/config/

For PostregreSQL help, check out these links:
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04
http://suite.opengeo.org/docs/latest/dataadmin/pgGettingStarted/firstconnect.html

To follow the adventure of this project for me, check out these links:
https://discussions.udacity.com/t/issues-deploying-application-to-lightsail/403522
https://discussions.udacity.com/t/remote-login-trouble/398675/5

Special thanks to:
Swooding, Greg Berger, Trish, Teraisa, and Johan Soetens for all the help you gave me. 
