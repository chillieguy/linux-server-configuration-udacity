# Project: Linux Server Configuration

### Project Overview:

Take a baseline installation of a Linux server and prepare it to host a web application. Secure the server from a number of attack vectors, install and configure a database server, and deploy an existing web applications onto it.

### Why this Project?

A deep understanding of exactly what a web application is doing, how they are hosted, and the interactions between multiple systems are what define you as a Full Stack Web Developer. In this project, the student will be responsible for turning a brand-new, bare bones, Linux server into the secure and efficient web application host.

## Project Details
IP address: ```35.165.34.87```(Will be deactived after review)

URL: http://ec2-35-165-34-87.us-west-2.compute.amazonaws.com/

Web app is a modified version of [Flask Catalog](https://github.com/chillieguy/flask-catalog-app)

## Software Installed
* Python3
* Flask
* Postgresql
* Sqlalchemy
* Google OAuth2
* Requests

## Server Configuration Steps

- Goto [Amazon Lightsail](https://lightsail.aws.amazon.com) 
  - Select Linux/Unix for Platform
  - Select OS only and Ubuntu for Blueprint
  - Select $5 Instance Plan
  - Leave Default Name or change to customer name
  - Create Instance
  - Detailed directions can be found on [ServerPilot](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html) for setting up a Lightsail instance.
- SSH into new Server
  - Click SSH Keys from Account page
  - Download default SSH key
  - From terminal on local machine
    - Rename download file `<filename>.pem` with `mv <filename>.pem id_udacity.pub`
    - Move file to `.ssh` with `mv lightsails_key.pem ~/.ssh`
    - Change file permissions with `chmod 600 ~/.ssh/id_udacity.pub`
    - Connect with SSH by entering `ssh -i ~/.ssh/id_udacity.pub ubunutu@35.165.34.87`
- Update system files/packages
  - `sudo apt-get update`
  - `sudo apt-get upgrade`
- Add new user for Grader
  - Enter `sudo adduser grader`, set password and enter information requested or leave as default
  - Add grader as a sudo user
    - `sudo touch /etc/sudoers.d/grader`
    - `sudo nano /etc/sudoers.d/grader` add `grader ALL=(ALL:ALL) ALL` - save
    - Verify grader has sudo acces
      - `su - grader`
      - Enter password used during grader creation
      - `sudo -l`
      - Enter password
      - Output in terminal should indicate User grader may run the following commands: `(ALL:ALL) ALL`
- Create SSH key for grader access
  - `su - grader` if not still in grader account
  - `mkdir .ssh`
  - `sudo ssh-keygen`
    - Enter for file and path `.ssh/grader`
    - Enter password
  - `cat .ssh/grader.pub` copy output to clipboard
  - On local machine in terminal `cd ~/.ssh`
  - `touch grader.pub`
  - Using editor of choice pasts contents of clipboard into grader.pub
  - `chmod 600 grader.pub`
  - Test new SSH key
    - From new termainal window/tab on local machine enter `ssh -i ~/.ssh/grader.pub grader@35.165.34.87 -p 2200`
    - New SSH session should have command prompt `grader@<IP-ADDRESS>`
  - This is the key that can be provided to Udacity Grader to access grader account
- Setup configuration for Uncomplicated Firewall
  - Change sshd_config for port 2200 with `sudo nano /etc/ssh/sshd_config` edit Port 22 to Port 2200
  - `sudo ufw allow 80/tcp`
  - `sudo ufw allow 2200/tcp`
  - `sudo ufw allow 123/upd`
  - `sudo ufw enable`
  - `sudo ufw status` to check open ufw ports
  - If needed `sudo ufw delete deny 22/tcp`
  - In Lightsails admin panel add port 2200 Customer TCP
  - `sudo service ssh restart` to make changes to SSH active
- Change local timezone to UTC
  - `sudo dpkg-reconfigure tzdata` select None of the Above then UTC
- Install packages needed for Flask App with Postgresql
  - `sudo apt-get install apache2`
  - `sudo apt-get install postgresql`
  - `sudo apt-get install git`
  - `sudo apt-get install libapache2-mod-wsgi-py3`
  - `pip install flask`
  - `pip install sqlalchemy`
  - `pip install requests`
  - `pip install oauth2client`
  - `pip install psycopg2`
- Configure Apache
  - Check in browser `35.165.34.87` that default Apache2 page displays
  - sudo touch /etc/apache2/sites-available/FlaskApp.conf
  - sudo nano /etc/apache2/sites-available/App.conf paste in
```
  <VirtualHost *:80>
	ServerName 35.165.34.87
	ServerAdmin chillieguy@live.com
	WSGIScriptAlias / /var/www/Catalog/app.wsgi
	<Directory /var/www/Catalog/App/>
		Order allow,deny
		Allow from all
	</Directory>
	Alias /static /var/www/Catalog/App/static
	<Directory /var/www/Catalog/App/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Configure .wsgi
  - `cd /var/www/Catalog`
  - `sudo touch app.wsgi`
  - `sudo nano app.wsgi` and paste in
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/Catalog/")

from App import app as application
application.secret_key = '<ENTER A SECRET KEY>'
```
- Enable Virtual Host and Restart Apache
 - `sudo a2ensite App`
 - `sude service apache2 restart`
- Setup Postgresql
  - `sudo cat /etc/postgresql/9.5/main/pg_hba.conf` to check that Postgresql does not allow remote connections.
  - Login to postgres with `su - postgres`
  - Start Postgresql cli with `psql`
  - Create new database `CREATE DATABASE catalog;`
  - Create new user `CREATE USER catalog;`
  - With password `ALTER ROLE catalog WITH PASSWORD 'password';`
  - Allow user catalog to access catalog database `GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`
  - Create table(alternatively can be done in database_setup.py file)
    - `CREATE TABLE 'user'`
    - `CREATE TABLE 'item'`
    - `CREATE TABLE 'catalog'`
- Changes for Google Oauth2
  - In developer console add to Authorized redirect URIs
    - `http://ec2-35-165-34-87.us-west-2.compute.amazonaws.com/login`
    - `http://ec2-35-165-34-87.us-west-2.compute.amazonaws.com/gconnect`
  - Add to client_secrets.json
    - `http://ec2-35-165-34-87.us-west-2.compute.amazonaws.com/login`
    - `http://ec2-35-165-34-87.us-west-2.compute.amazonaws.com/gconnect`
  - In app.py change `result = json.loads(h.request(url, 'GET')[1])` to `result = json.loads(h.request(url, 'GET')[1].decode('utf-8))` to fix byte to str error
- Disable default Apache2 site
  - `sudo a2dissite 000-default.conf`
- Install Fail2Ban
  - `sudo apt-get install fail2ban`
  - `sudo apt-get install sendmail` - for mail notifications
  - `sudo apt-get install iptables-persistent`
  - Update jail.local with settings below - `sudo nano etc/fail2ban/jail.local`
  - `sudo service fail2ban stop`
  - `sudo service fail2ban start`
```
set bantime  = 600
destemail = useremail@domain
action = %(action_mwl)s
under [ssh] change port = 2220
```  
- Restart Apache2
  - `sudo service apache2 restart`











 # Resources Referenced
 * [Udacity - Deploying to Linux Servers](https://classroom.udacity.com/nanodegrees/nd004/parts/00413454014)
 * [ServerPilot](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html)
 * [Digital Ocean Deploy a Flask App](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
 * [Digital Ocean - Fail2Ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
 * [Amazon Lightsail Documentation](https://lightsail.aws.amazon.com/ls/docs/all)
 * [Stack Overflow - google authorization not working on Apache hosted on AWS lightsail
](https://stackoverflow.com/questions/47004929/google-authorization-not-working-on-apache-hosted-on-aws-lightsail)
* [Ubuntu Documentation](https://help.ubuntu.com/lts/serverguide/httpd.html)