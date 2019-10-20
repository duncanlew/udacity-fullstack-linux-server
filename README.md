# Udacity Fullstack Linux Server Configuration
This is a project for the Udacity Full Stack Web Developer Nanodegree program. For the purpose of this project, a baseline linux server will be configured to deploy my previous flask application. A strong understanding of the functionality of web applications, the hosting thereof and interaction between multiple systems is demonstrated in this project to illustrate the required full-stack skills. 

# Linux server details
The linux server is set up using the cloud platform of [Amazon Lightsail](https://aws.amazon.com/lightsail/). It's running Ubuntu 18.10. The details for accessing the linux server are as follows:

* IP address : [18.194.151.72](http://18.194.151.72) 
* SSH port: 2200 
* username: grader
* Using the given private ssh key during submission, you can ssh into the server with the following command
```bash
ssh grader@18.194.151.72 -p 2200 -i [path/to/private-ssh-key]
```

# Configuration details
A number of steps have been taken to set up the linux server and deploy the flask app. 

## 1.Installing and updating required packages
After the linux server has been created using Amazon Lightsail, we need to update the system packages first as follows:
```bash
sudo apt update
sudo apt upgrade
```
After performing the update, a system reboot is required which can be done as follows:
```bash
reboot
```

After having updated the baseline system packages, we can start installing the required packages for our application:
```bash
sudo apt install git
sudo apt install python3-pip
sudo apt install python3-venv
sudo apt install apache2
sudo apt install libapache2-mod-wsgi-py3

# needed for postgresql
sudo apt-get install python-psycopg2
sudo apt-get install postgresql postgresql-contrib
sudo apt-get install libpq-dev
```

## 2. Setting up security
Security wise, we're going to do two things: we're going to change the SSH port number and set up UFW.

### 2.1 SSH port number change and prevent root login
In order to change the default port number for ssh from 22 to 2200, we need to do the following:
```
sudo nano /etc/ssh/sshd_config
```

Change the following line in the file
```
Port 22
```
to 
```
Port 2200
```

And uncomment the line on `PermitRootLogin` and give it the value no to prevent root login:
```
PermitRootLogin no
```

### 2.2 Set up of UFW
UFW needs to be set up to allow our new SSH port number. In addition to that we need to also allow www and ntp and deny the default SSH port 22.
```bash
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw deny 22
```

### 2.3 Finalization
As a final step, we need to restart the ssh service so that it accepts the port number 2200, and enable the ufw so that the new rules kick into place. 
```bash
sudo service ssh restart
sudo ufw enable
```

## 3. Creating new user `grader` and setting up ssh for this user
The user can be created with the following command
```bash
adduser grader
```
In order to add the user to the sudors list, create the file `/etc/sudoers.d/grader` and add the following line:
```
grader ALL=(ALL) NOPASSWD:ALL
```

The next step is to create an ssh key pair for this user on your local machine and not on the linux server. Run the following command in your terminal:
```bash
ssh-keygen
```
It will prompt you with the following:
```
Enter file in which to save the key
```
Give it a good name and if necessary place it in a different directory than the default location of `~/.ssh/` like 
```
Users/your-user-name/Desktop/grader
```
This will generate two files `grader` and `grader.pub`. 

Create a directory `.ssh` in `/home/grader`.
```
sudo mkdir /home/grader/.ssh
```

Change your directory into this .ssh file of grader and create a file called `authorized_keys`:
```bash
sudo touch authorized_keys
```

Now add the contents of the generated file `grader.pub` into `authorized_keys` with 
```bash
sudo nano authorized_keys
``` 
Finally, you can ssh into the server from your local machine as grader with the following command:
```bash
ssh grader@18.194.151.72 -p 2200 -i [path/to/private-ssh-key]
```

For more explanation on the generation of ssh keys, check this [link](http://www.macworld.co.uk/how-to/mac-software/how-generate-ssh-keys-3521606/). 

## 4. Deploy flask app
In the final step, we're going to set up the flask app so that it can be served to the web as a wsgi app. For this flask app, we're using Python 3.6. 

### 4.1 Clone repository
First we need to clone the repository from [udacity-item-catalog](https://github.com/duncanlew/udacity-item-catalog). We will place this in the `/var/www` directory. This can be done as follows:
```bash
cd /var/www
sudo git clone https://github.com/duncanlew/udacity-fullstack-item-catalog.git
```
### 4.2 Installing required packages for the flask app
Change your directory to `/var/www/udacity-item-catalog`. Create a python virtual environment as follows:
```bash
virtualenv -p python3 venv3
```
Afterwards, activate this virtual environment as follows:
```bash
source venv3/bin/activate
```
Add the following required package into the file `requirements.txt`:
```
psycopg2
```

Finally, install the required packages for this flask app as follows:
```bash
pip3 install -r requirements.txt
```

### 4.3 Create wsgi configurations
Inside the directory `/var/www/udacity-item-catalog`, create the file `project.wsgi` with the following contents:
```python
#! /usr/bin/python3.6
activate_this = '/var/www/udacity-item-catalog/venv3/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

import logging
import sys
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/udacity-item-catalog/')
from project import app as application
```

Afterwards, change your directory to `/etc/apache2/sites-available/` and create the file `udacity-item-catalog.conf` with the following contents:
```
<VirtualHost *:80>
     # Add machine's IP address (use ifconfig command)
     ServerName 18.194.151.72
     # Give an alias to to start your website url with
     WSGIScriptAlias / /var/www/udacity-item-catalog/project.wsgi
     <Directory /var/www/udacity-item-catalog/>
     # set permissions as per apache2.conf file
            Options FollowSymLinks
            AllowOverride None
            Require all granted
     </Directory>
     ErrorLog ${APACHE_LOG_DIR}/error.log
     LogLevel warn
     CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Finally, make sure to run wsgi with the following command
```
sudo a2enmod wsgi
```
and then enable the conf file as follows:
```
sudo a2ensite udacity-item-catalog.conf
```

For more information and in-depth explanation behind this wsgi setup, check out this [link](https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft).


### 4.4 Set up PostgreSQL database
In order to set up the PostgreSQL database, we need to do a couple of things. When PostgreSQL is installed, it comes with a default user in the linux system: postgres. We are going to log into the psql terminal as the default user:
```
sudo -u postgres psql
```

Afterwads, we're going to create a user called `wsgi`, and create a database of which the privileges will be assigned to `wsgi`. Fill in anything password for `wsgi` that you would like.
```psql
CREATE USER wsgi WITH PASSWORD 'your-password';
CREATE DATABASE computer_shop
GRANT ALL PRIVILEGES ON DATABASE computer_shop TO wsgi;
```

The next step is to edit the python files `database_initializer.py`, `models.py` and `project.py` in which the sqlite connection will be replaced with a postgres connection. The new connection in the python files needs to like like this:
```python
engine = create_engine('postgresql://wsgi:your-password@localhost/computer_shop')
```

Finally, run the following two commands to initialize the postgres database:
```bash
python3 models.py
python3 database_initializer.py
```

For more information about the installation of PostgreSql, pleace check this [link](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)

### 4.5 Twitter Sign in  tokens
Inside `/var/www/udacity-item-catalog/` the Twitter consumer key and secret need to be added to 25 and 26
```python
consumer_key='YOUR_CONSUMER_KEY',
consumer_secret='YOUR_CONSUMER_SECRET'
```

### 4.6 Change ownership to www-data
The default user for web servers on Ubuntu is `www-data`. In order for the web server to be able access the files necessary for the flask app, we're going to give this user ownership of the flask app directory as follows:
```
sudo chmod -R www-data:www-data /var/www/udacity-item-catalog
```

### 4.7 Restart services
As a final step, we need to restart apache with the following command
```bash
sudo service apache2 restart 
```
