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

Install *fail2ban* in order to mitigate brute force attacks by users and bots alike.

1. `$ sudo apt-get update`.
2. `$ sudo apt-get install fail2ban`.
3. Install the *sendmail* package to send the alerts to the admin user: `$ sudo apt-get install sendmail`.
4. Create a file to safely customize the *fail2ban* functionality: `$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local` .
5. `$ sudo nano /etc/fail2ban/jail.local`. Set the *destemail* admin user's email address.

Sources: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04), [Reddit](https://www.reddit.com/r/linuxadmin/comments/2lravs/fail2ban_does_not_detect_my_ssh_privatekey/).


### 8 - Install Apache, mod_wsgi and Git

1. `$ sudo apt-get install apache2`.
2. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
4. `$ sudo service apache2 start`.
5. `$ sudo apt-get install git`.
