# Udacity - Linux server configuration project

## Description


The application meant to be deployed is the **Item catalog app**, (https://github.com/trkalajian/catalog_server_files).

IP address: 35.183.85.71.

Accessible SSH port: 2200.

Application URL: (http://35.183.85.71.xip.io).

## Initial - Walkthrough

Go to AWS Lightsail and create a new account / sign in with your account.
Click Create instance and choose Linux/Unix,OS only Ubuntu 16.04LTS
Click Create button to create an instance.

1. Set up SSH key: Go to account page from your AWS account. You will find your SSH key there.
2. Download your SSH key, the file name will be like LightsailDefaultPrivateKey____.pem
3. Navigate to the directory where your file is stored in your terminal.
4. Run `chmod 600 LightsailDefaultPrivateKey____.pem` to restrict the file permission.
5. Change name to lightsail_key.rsa.
6. Login with command `ssh -i lightsail_key.rsa ubuntu@35.183.85.71` in your terminal.

### 1 - Create a new user named *grader* and grant this user sudo permissions.

1. Add a new user called *grader*: `sudo adduser grader`.
2. Create a new file under the suoders directory: `sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.
3. In order to prevent the "sudo: unable to resolve host" error, edit the hosts:
	1. `sudo nano /etc/hosts`.
	2. Add the host: `127.0.1.1 ip-10-20-37-65`.

### 2 - Update all currently installed packages

1. `sudo apt-get update`.
2. `sudo apt-get upgrade`.

### 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `sudo dpkg-reconfigure tzdata`.

### 4 - Configure the key-based authentication for *grader* user

1. run `cp /ubuntu/.ssh/authorized_keys /home/grader/.ssh/authorized_keys`
2. change some permissions:
	1. `sudo chmod 700 /home/grader/.ssh`.
	2. `sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner to *grader*: `sudo chown -R grader:grader /home/grader/.ssh`.
4. Now you are able to log into the remote VM through ssh with the following command: `ssh -i lightsail_key.rsa grader@35.183.85.71`.

### 5 - Edit sshd_config
1. Enforce key-based authentication, `sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and edit it to *no*.
2. Find the *Port* line and edit it to *2200*.
3. Disable ssh login for *root* user, find the *PermitRootLogin* line and edit it to *no*.
4. `sudo service ssh restart`.
5. Now you are able to log into the remote VM through ssh with the following command: `ssh -i lightsail_key.rsa -p 2200 grader@35.183.85.71`.

### 6 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `sudo ufw allow 2200/tcp`.
2. `sudo ufw allow 80/tcp`.
3. `sudo ufw allow 123/udp`.
4. `sudo ufw enable`.

### 7 - Install Apache, mod_wsgi

1. `sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `sudo a2enmod wsgi`.
3. `sudo service apache2 start`.

### 8 - Clone the Catalog app from Github
1. Install git, `sudo apt-get install git`
1. `cd /var/www`. Then: `sudo mkdir catalog`.
2. Change owner for the *catalog* folder: `sudo chown -R grader:grader catalog`.
3. Move inside that newly created folder: `cd /catalog` and clone the catalog repository from Github: `git clone https://github.com/trkalajian/catalog_server_files.git catalog`.
4. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
```
5. Change *app.py* to *__init__.py* with command `mv app.py __init__.py`
6. Change *client_secrets.json* path to `/var/www/catalog/catalog/client_secrets.json`
7. 


### 9 - Install virtual environment, Flask and the project's dependencies

1. Install *pip*, the tool for installing Python packages: `sudo apt-get install python-pip`.
2. If *virtualenv* is not installed, use *pip* to install it using the following command: `sudo pip install virtualenv`.
3. Move to the *catalog* folder: `cd /var/www/catalog`. Then create a new virtual environment with the following command: `sudo virtualenv venv`.
4. Activate the virtual environment: `source venv/bin/activate`.
5. Change permissions to the virtual environment folder: `sudo chmod -R 777 venv`.
6. Install Flask: `pip install Flask`.
7. Install all the other project's dependencies: `pip install bleach httplib2 request oauth2client sqlalchemy psycopg2`. 

### 10 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `sudo nano /etc/apache2/sites-available/catalog.conf`.
2. Paste in the following lines of code:
```
<VirtualHost *:80>
    ServerName 35.183.85.71
    ServerAlias ec2-35.183.85.71.ca-central-1a.compute.amazonaws.com
    ServerAdmin admin@35.183.85.71
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


3. Enable the new virtual host: `sudo a2ensite catalog`.


### 11 - Install and configure PostgreSQL

1. Install some necessary Python packages for working with PostgreSQL: `sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `sudo apt-get install postgresql postgresql-contrib`.
3. Postgres automatically creates a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. Change to that user with: `sudo su - postgres`, then connect to the database system with `psql`.
4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'password';`.
5. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
7. Connect to the database: `# \c catalog`.
8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `exit`.
11. Inside the Flask application, the database connection is now performed with: 
```python
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```
12. Setup the database with: `python /var/www/catalog/catalog/setup_database.py`.
13. Change create engine line in your *__init__.py* and *database_setup.py* to: `engine = create_engine('postgresql://catalog:password@localhost/catalog')`

### 18 - Update OAuth authorized JavaScript origins

1. Change the url's listed in the google Oauth to http://35.183.85.71.xip.io


### 19 - Restart Apache to launch the app
1. `sudo service apache2 restart`.

#### 
sources `https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps`
	`https://help.ubuntu.com/community/UbuntuTime`
	`https://askubuntu.com/questions/740976/im-getting-this-you-should-replace-this-file-located-at-var-www-html-index-h'
	'https://askubuntu.com/questions/629995/apache-not-able-to-restart'
	'https://modwsgi.readthedocs.io/en/develop/configuration-directives/WSGIDaemonProcess.html'
