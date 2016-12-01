# Linux_Server_Configuration
Udacity Full Stack Web Developer Project

## Project Description

> taking a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

The application deployed here is the **NCAA-Teams App**, previously developed for [Item-Catalog Project](https://github.com/QilinGu/NCAA-Teams).

## Server/App Info

IP address: 54.201.76.100

SSH port: 2200.

Application URL: [http://ec2-54-201-76-100.us-west-2.compute.amazonaws.com/](http://ec2-54-201-76-100.us-west-2.compute.amazonaws.com/).

Username for Udacity reviewer: `udacity`

## Configurations in Steps

### 1 - Launching an AWS EC2 Instance and connect to it via SSH

1. Launching an EC2 instance: select *Ubuntu Server 16.04 LTS (HVM), SSD Volume Type* and launch it
2. The instance's security group provides a SSH port 22 by default
3. The public IP is 54.201.76.100
4. Download the private key `udacity.pem` from AWS

### 2 - User, SSH and Security Configurations

1. Log into the remote VM as *root* user (`ubuntu`) through ssh: `$ ssh -i "udacity.pem" ubuntu@54.201.76.100`.
2. Create a new user *udacity*:  `$ sudo adduser udacity`.
3. Grant udacity the permission to sudo, by adding a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/udacity`. In the file put in: `udacity ALL=(ALL:ALL) ALL`, then save and quit.
4. Generate an encryption key pair with: `$ ssh-keygen -f /home/udacity/.ssh/udacity_key.rsa`
5. Rename the public key `udacity_key.rsa.pub` to `authorized_keys`, and change the permissions:
	1. `$ sudo chmod 700 /home/udacity/.ssh`.
	2. `$ sudo chmod 644 /home/udacity/.ssh/authorized_keys`.
	3. Change the owner from `ubuntu` to `udacity`: `$ sudo chown -R udacity:udacity /home/udacity/.ssh`
6. Copy the private key `udacity_key.rsa` to *local machine* for udacity reviewer
7. Enforce key-based authentication, change SSH port to `2200` and disable remote login of *root* user:
   1. `$ sudo nano /etc/ssh/sshd_config`  
   2. Change `PasswordAuthentication` to `no`.
   3. Change `Port` to `2200`.
   4. Change `PermitRootLogin` to `no`
   5. `$ sudo service ssh restart`.
   6. In AWS EC2 Security Group,  add `2200` as the inbound custom TCP Rule port.

Source: [Ubuntu forums](http://ubuntuforums.org/showthread.php?t=1739013),  [Askubuntu](http://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server).

### 3 - Configure the local timezone to UTC

1. Open time configuration and set it to UTC: `$ sudo dpkg-reconfigure tzdata`.
2. Install *ntp daemon ntpd* for a better synchronization of the server's time over the network connection: `$ sudo apt-get install ntp`.

Source: [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime).

### 4 - Update all currently installed packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.

### 5 - Configure cron scripts to automatically update packages

1. Install *unattended-upgrades*: `$ sudo apt-get install unattended-upgrades`.
2. Enable it by: `$ sudo dpkg-reconfigure --priority=low unattended-upgrades`.

### 6 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.
5. Add 3 rules above as Security Group inbound rules of AWS EC2 instance

### 7 - Configure firewall to monitor for repeated unsuccessful login attempts and ban attackers

Install `fail2ban` in order to mitigate brute force attacks by users and bots alike.

1. `$ sudo apt-get update`.
2. `$ sudo apt-get install fail2ban`.
3. Install the *sendmail* package to send the alerts to the admin user: `$ sudo apt-get install sendmail`.
4. Create a file to safely customize the *fail2ban* functionality: `$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local` .
5. Adjust `fail2ban` configurations:
   ```
   $ sudo nano /etc/fail2ban/jail.local
   ``` 
   
   1. Set the **destemail**: admin user's email address.
   2. Set the **bantime**: `bantime = 1800`
   3. Set the **action**: `action = %(action_mwl)s`

Sources: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04), [Reddit](https://www.reddit.com/r/linuxadmin/comments/2lravs/fail2ban_does_not_detect_my_ssh_privatekey/).


### 8 - Install Apache, mod_wsgi and Git

1. `$ sudo apt-get install apache2`.
2. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
4. `$ sudo service apache2 start`.
5. `$ sudo apt-get install git`.

### 9 - Configure Apache to serve a Python mod_wsgi application

1. Clone the item-catalog (NCAA-Teams) app from Github
   ```
   $ cd /var/www
   $ sudo mkdir catalog
   $ sudo chown -R udacity:udacity catalog
   $ cd /catalog
   $ git clone https://github.com/QilinGu/NCAA-Teams catalog
   ```
2. To make .git directory is not publicly accessible via a browser, create a .htaccess file in the .git folder and put the following in this file:  `RedirectMatch 404 /\.git`
3. Install pip , virtualenv (in /var/www/Catalog)
   ```
   $ sudo apt-get install python-pip
   $ sudo pip install virtualenv
   $ sudo virtualenv venv
   $ source venv/bin/activate
   $ sudo chmod -R 777 venv
   ```
4. Install Flask and other dependencies:
   ```
   $ sudo pip install -r catalog/requirements.txt
   ```
5. Install Python's PostgreSQL adapter *psycopg2*:
   ```
   $ sudo pip install psycopg2
   ```
6. Configure and Enable a New Virtual Host
   ```
   $ sudo nano /etc/apache2/sites-available/catalog.conf
   ```
   Add the following content:
   
   ```
   <VirtualHost *:80>
       ServerName 54.201.76.100
       ServerAlias ec2-54-201-76-100.us-west-2.compute.amazonaws.com
       ServerAdmin udacity@54.201.76.100
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
   Enable the new virtual host:
   ```
   $ sudo a2ensite catalog
   ```
7. Create and configure the .wsgi File
   ```
   $ cd /var/www/catalog/
   $ sudo nano catalog.wsgi
   ```
   Add the following content:
   ```python
   import sys
   import logging
   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0, "/var/www/catalog/")

   from catalog import app as application
   ```
8. The original NCAA-Teams app that were developed on local machine needs some tweaks in order to be deployed on AWS. The major modifications include:
   1. Rename `app.py` to `__init__.py`
   2. Update the absolute path of `client_secrets.json` in `__init__.py`
   3. Add app.secret_key for the Flask app in `__init__.py`
   4. Add the code to create a dummy user in `db_seed.py`

Sources: [DigitalOcean-1](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps), [Dabapps](http://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/), [DigitalOcean-2](https://www.digitalocean.com/community/tutorials/how-to-run-django-with-mod_wsgi-and-apache-with-a-virtualenv-python-environment-on-a-debian-vps).

### 10 - Install and configure PostgreSQL

1. Install some necessary Python packages for working with PostgreSQL: `$ sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`
3. PostgreSQL automatically creates a new user 'postgres' during its installation. So we can connect to the database by using postgres username with: `$ sudo -u postgres psql`
4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'catalog';`
5. Give *catalog* user the CREATEDB permission: `# ALTER USER catalog CREATEDB;`
6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`
7. Connect to the database: `# \c catalog`
8. Revoke all the rights: `# REVOKE ALL ON SCHEMA public FROM public;`
9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`
10. Log out from PostgreSQL: `# \q`. Then return to the *udacity* user: `$ exit`.
11. Edit the `db_seed.py` and `database_setup.py` file:   
      Change `engine = create_engine('sqlite:///category.db')`
      to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`
12. Remote connections to PostgreSQL should already be blocked. Double check by opening the config file:
    ```
    $ sudo nano /etc/postgresql/9.5/main/pg_hba.conf
    ```
Source: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).

### 11 - Final updates on OAuth to make the app run on AWS

1. Go to the project on the Developer Console, and navigate to APIs & Auth > Credentials > Edit Settings
2. Add the hostname and piblic IP address to the Authorized JavaScript origins and (host name + 'oauth2callback'), (host name + 'gconnect') to Authorized redirect URIs.
3. Populate the PostgreSQL database with `$ python db_seed.py`
4. Restart Apache to launch the app: `$ sudo service apache2 restart`
5. If an internal error shows up when you try to access the app, open Apache error log as a reference for debugging:
   ```
   $ sudo tail -20 /var/log/apache2/error.log
   ```

### 12 - Install system monitor tools

1. `$ sudo apt-get update`.
2. `$ sudo apt-get install glances`.
3. To start this system monitor program: `$ glances`.

Source: [Askubuntu](http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu).
