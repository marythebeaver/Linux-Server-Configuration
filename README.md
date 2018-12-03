# Linux-Server-Configuration
Using Amazon Lightsail to configure and deploy a Flask app to an Ubuntu server

# INFO
- IP Address: 35.182.95.210
- SSH port: 2200
- URL: http://35.182.95.210.xip.io

# Amazon Lightsail Set Up
1. Register an AWS account and go to the Amazon Lightsail website
2. Create an instance with Linux/Unix, OS only, Ubuntu 16.04 LTS
3. Wait for the instance to set up, and click "Account Page" at the bottom to download your private SSH key, so that you can ssh into the server through your local terminal.
4. Click Networking and add TCP 123 and TCP 2200 into firewall to enable this two ports.

# Linux Configuration
## login and update 
1. Store the private SSH Key downloaded from Lighsail into the .ssh folder, ex. Macintosh HD/Users/[Your username]/.ssh/
2. Use `$ ssh -i ~/.ssh/filename.pem ubuntu@ip_address` to ssh to the Lightsail server you just created, wherein up_address of your instance will be shown on the Amazon Lightsail page.
3. Using the following commands to update the package:
```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
$ sudo apt-get autoremove
```
4. Use `$ sudo apt-get install finger` to install Finger, so that you can use it to see the users on this server.

## create new user
1. Use `$ sudo su -` to switch to the root user
2. Create a new user grader for Udacity by using `$ sudo adduser grader`, and enter the password indicated by Udacity.
3. Use `$ sudo nano /etc/sudoers.d/grader` to create and open file grader and put `grader ALL=(ALL) NOPASSWD:ALL` into the file, so that grader can use sudo comments.
4. Open a new terminal on local, use`$ ssh-keygen -f ~/.ssh/udacity.rsa` to create public key and private kay
5. On the same local terminal, use `$ cat ~/.ssh/udacity.rsa.pub` to see the created public key and copy it.
6. Back to the lightsail server terminal, use `$ mkdir /home/grader/.ssh` to create a .ssh folder.
7. Use `$ cd /home/grader/.ssh` to cd into the .ssh file and use `$ touch .ssh/authorized_keys` to create a file to store the created public key.
8. Use `$ nano .ssh/authorized_keys` to paste the copied public key into the file.
9. Use the following commands to change the permissions:
``` 
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys 
$ sudo chown -R grader:grader /home/grader/.ssh
$ sudo service ssh restart
```

## login as grader and configure firewall/ports
1. Disconnect from the server
2. Use `$ ssh -i ~/.ssh/udacity.rsa grader@ip_address` to login the lightsail server as grader
3. Use `$ sudo nano /etc/ssh/sshd_config` to change sshd configuration, wherein change Port 22 and to Port 2200 and use `$ sudo service ssh restart` to restart and enable the setting.
4. Now you have to login the server by `$ ssh -i ~/.ssh/udacity.rsa grader@ip_address -p 2200`
5. Use the following commands to set the firewall, wherein you can see the firewall status by `$ sudo ufw status`:
```
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw allow 123/tcp
$ sudo ufw enable
```

# Application Deployment
Please see https://github.com/marythebeaver/item_catalog for the details of the Web application.

## Install the required softwares
```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo apt-get install git
$ sudo apt-get install python-pip
$ sudo pip install --upgrade pip==9.0.3
```
## Set the APP envirment
1. Use `$ sudo a2enmod wsgi` and `$ sudo service apache2 restart` to enable mod_wsgi, wherein after enabling the wsgi, you can see the Apache2 Ubuntu Default Page on browser through your IP address.
2. Use followed comments to create a directory for our catalog application and make the user grader the owner.
```
$ cd /var/www
$ sudo mkdir catalog
$ sudo chown -R grader:grader catalog
$ cd catalog
```
3. Use `$ git clone https://github.com/marythebeaver/item_catalog catalog` to clone the web app into catalog
4. Anywhere in the file where Python tries to open client_secrets.json or fb_client_secrets.json must be changed to its complete path, in our case it's `/var/www/catalog/catalog/client_secrets.json`
5. Go to https://console.developers.google.com/apis and/or https://developers.facebook.com/ to add your ip adreess for Authorization, and redownload the client_secrets.json, and make `/var/www/catalog/catalog/client_secrets.json` to be the same, wherein if you cannot add your ip address, use ip_address.xip.io. 
6. Use `$ sudo nano catalog.wsgi` to create the .wsgi file and paste the followed code into the file, wherein make sure your secret key matches with your project secret key:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```
7. Use `$ mv project.py __init__.py` to rename the category.py file by ` __init__.py`
8. Use the followed comments to create a virtual environment, make sure you are in /var/www/catalog:
```
$ sudo pip install virtualenv
$ sudo virtualenv venv
$ source venv/bin/activate
$ sudo chmod -R 777 venv
```
9. Use the followed comments to install all packages required for our Flask application:
```
$ sudo pip install flask
$ sudo pip install httplib2 
$ sudo pip install oauth2client 
$ sudo pip install sqlalchemy 
$ sudo pip install psycopg2 
$ sudo pip install psycopg2-binary
```
10. Use `$ sudo nano /etc/apache2/sites-available/catalog.conf` and paste the followed codes in it to configure virtual host, wherein the host name can lookup from https://whatismyipaddress.com/ip-hostname:
```
<VirtualHost *:80>
    ServerName [Public IP]
    ServerAlias [Hostname]
    ServerAdmin admin@[Public IP]
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
```
11. Use `$ sudo a2ensite catalog.conf` and `$ a2dissite 000-default.conf` to enable virtual host.
12. Use the followed comments to set up the database:
```
$ sudo apt-get install libpq-dev python-dev
$ sudo apt-get install postgresql postgresql-contrib
$ sudo su - postgres -i
$ psql
postgres=# CREATE USER catalog WITH PASSWORD 'password';
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog with OWNER catalog;
postgres=# \c catalog
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
catalog=# \q
$ exit
```

10. Use `$ sudo nano  __init__.py` and `$ sudo nano  database_setup.py` to change the database engine from `sqlite://category.db` to `postgresql://username:password@localhost/catalog` 
11. Use `$ sudo service apache2 restart` to restart your apache server
12. Enter http://ip_address.xip.io in the browser, and now you can see the web app through it.

# References:
https://github.com/ddavignon/linux-server-configuration
https://github.com/mulligan121/Udacity-Linux-Configuration
