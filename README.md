How to Access Online Catalog
-----------------------------------------
1. Make sure you have internet connection
2. Open browser and go to "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com/"
3. "Click Here to Login" to navigate to login page

How to Login to AWS Server as "grader"
-----------------------------------------
1. Open up git-bash on local machine
2. Type "ssh -i [folder where grader rsa key is located]/grader grader@52.221.179.112 -p 2200"

STEPS TO SETUP AWS SERVER
Download Default Private Key and Generate Keys
-----------------------------------------
1. Download default private key from Lightsail:
	a. Navigate to server landing page 
	b. At bottom of page should be the phrase "You can download your default private key from the Account page." Click "Account page"
	c. Select "SSH keys" tab
	d. "Download" key into ~/.ssh folder on local machine and rename to "udacitykey.pem" 
2. Generate own key:
	a. On local machine, type 'ssh-keygen -b 1024 -f grader -t rsa' and save private key in ~/.ssh on local machine (passphrase not required)

Login as ubuntu and Create User
-----------------------------------------
1. On your local machine, you should be able to login to your lightsail server as ubuntu by typing in "ssh -i ~/.ssh/udacitykey.pem ubuntu@52.221.179.112"
2. Create user in virtual machine as ubuntu user:
	a. Type "useradd -m -s /bin/bash grader"
	b. "touch /etc/sudoers.d/grader" to create grader file
	c. "nano /etc/sudoers.d/grader" to edit grader file
	d. Type in below, save and quit:
		grader ALL=(ALL) NOPASSWD:ALL
	e. "mkdir /home/grader/.ssh" to create .ssh directory for grader
	f. "chown grader:grader /home/grader/.ssh" to change owner to grader
	g. "chmod 700 /home/grader/.ssh" to give rwx access to owner
	h. "touch home/grader/.ssh/authorized_keys" to create authorized_keys file
	i. "chown grader:grader /home/grader/.ssh/authorized_keys" to change owner to grader
	j. "chmod 644 /home/grader/.ssh/authorized_keys" to give rw access to owner and r access to everyone else
	k. Copy content from grader.pub on local machine to authorized_keys file on virtual machine
	l. You should be able to login to your lightsail server as grader by typing in "ssh -i ~/.ssh/grader grader@52.221.179.112"

Update all packages
-----------------------------------------
1. On virtual machine, type "apt-get update" to update packages
2. On virtual machine, type "apt-get upgrade" to install updated packages

Disable root login
-----------------------------------------
1. Type "nano /etc/ssh/sshd_config"  to edit sshd_config file
2. Change "PermitRootLogin prohibit-password" to "PermitRootLogin no", save and quit

Change SSH port from 22 to 2200
-----------------------------------------
1. Type "nano /etc/ssh/sshd_config"  to edit sshd_config file
2. Change "Port 22" to "Port 2200", save and quit
3. Type "sudo service ssh restart" to restart ssh service
4. Go to the Lightsail server landing page, click on "Networking" tab
5. Add Custom TCP port with Port range "2200"
6. You should be able to login to your lightsail server as grader by typing in "ssh -i ~/.ssh/grader grader@52.221.179.112 -p 2200"
 
Configure Uncomplicated Firewall (UFW)
-----------------------------------------
1. Type following commands:
	a. sudo ufw default deny incoming
	b. sudo ufw default allow outgoing
	c. sudo ufw allow 2200/tcp
	d. sudo ufw allow www
	e. sudo ufw allow ntp
	f. sudo ufw show added
	g. sudo ufw enable

Configure local time zone
-----------------------------------------
1. "sudo dpkg-reconfigure tzdata" 
2. Select your timezone

Install Apache and mod-wsgi package
-----------------------------------------
1. "sudo apt-get install apache2" to install apache2
2. "sudo apt-get install libapache2-mod-wsgi" to install mod-wsgi package

Install PostgreSQL and configure user
-----------------------------------------
1. "sudo apt-get install postgresql postgresql-contrib" to install postgresql
2. Type "sudo nano /etc/postgresql/9.3/main/pg_hba.conf" to check that only the local host addresses 127.0.0.1 for IPv4 and ::1 for IPv6 are allowed
3. "sudo -u postgres createuser -P catalog" to create user called catalog
4. Enter password (e.g. password)
5. "sudo -u postgres createdb -O catalog catalog" to create database called catalog

Install Flask and Other Packages
-----------------------------------------
1. Type following commands to install flask and other packages:
	a. sudo apt-get install python-psycopg2 python-flask
	b. sudo apt-get install python-sqlalchemy python-pip
	c. sudo pip install oauth2client
	d. sudo pip install requests
	e. sudo pip install httplib2
	f. sudo pip install flask-seasurf
	g. sudo apt-get install git

Clone Catalog Repository and Make Changes to Catalog Files
-----------------------------------------
1. Type "cd /var/www" to move to directory
2. Create new directory by typing "sudo mkdir FlaskApp"
3. Move inside directory using "cd FlaskApp"
4. Clone catalog repository by typing "git clone https://github.com/darrylma/Udacity_Catalog.git"
5. Move repository to a inner FlaskApp directory "sudo mv ./Udacity_Catalog ./FlaskApp"
6. Move to the inner FlaskApp directory using "cd FlaskApp"
7. Rename main.py to __init__.py by typing "sudo mv main.py __init__.py"
8. Edit database_setup.py, populate_database.py, __init__.py 
	a. Change engine = create_engine('sqlite:///nba_database.db') to engine = create_engine('postgresql://catalog:password@localhost/catalog')
9. Create database schema "sudo python database_setup.py"
10. Populate database using "sudo python populate_database.py"
11. Comment out line "app.run(host='0.0.0.0', port=8000)" in __init.py__
12. Changed all code for: 
	'client_secrets.json'
    to:
	'/var/www/FlaskApp/FlaskApp/client_secrets.json'

Configure New Virtual Host
-----------------------------------------
1. Create FlaskApp.conf by typing "sudo nano /etc/apache2/sites-available/FlaskApp.conf"
2. Add following code:

<VirtualHost *:80>
	ServerName http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com/
	ServerAdmin darrylma85@gmail.com
	WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
	<Directory /var/www/FlaskApp/FlaskApp/>
		Order allow,deny
		Allow from all
	</Directory>
	Alias /static /var/www/FlaskApp/FlaskApp/static
	<Directory /var/www/FlaskApp/FlaskApp/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

3. Type "sudo a2ensite FlaskApp" to enable virtual host

Create .wsgi File
-----------------------------------------
1. Create the .wsgi file under /var/www/FlaskApp using "sudo nano flaskapp.wsgi"
2. Add the following code

	#!/usr/bin/python
	import sys
	import logging
	logging.basicConfig(stream=sys.stderr)
	sys.path.insert(0,"/var/www/FlaskApp/")

	from FlaskApp import app as application
	application.secret_key = 'super_secret_key'

3. File structure should now be as such:
	|--------FlaskApp
	|----------------FlaskApp
	|-----------------------static
	|-----------------------templates
	|-----------------------__init__.py
	|----------------flaskapp.wsgi

Reconfigure Google Login 
-----------------------------------------
1. Go to Google Developers Console -> API Manager -> Credentials
2. In the web client under "Authorized JavaScript origins", add: "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com"
3. In the web client under "Authorized redirect URIs", add: "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com/login" and "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com/gconnect"
4. Save and Download JSON
5. Copy and overwrite content in new client_secrets file into the current client_secrets.json file on your virtual machine (i.e. /var/www/FlaskApp/FlaskApp/client_secrets.json)

Reconfigure Facebook Login 
-----------------------------------------
1. Go to Facebook Developers Console
2. Navigate to your apps landing page
3. Under Products, click Facebook Login -> Settings
4. Under "Valid OAuth redirect URIs", add: "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com/login" and "http://ec2-52-221-179-112.ap-southeast-1.compute.amazonaws.com"
5. Save Changes

Restart Apache Server
-----------------------------------------
1. Restart Apache using "sudo service apache2 restart"