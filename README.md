# Linux Server Configuration

This project is a part of Udacity Full-Stack Developer Nanodegree.

The contents are describing configuration of a baseline installation of a
Linux distribution and hosting a web application on it.
The server is secured from a number of attack vectors, its software is updated,
web and database servers are installed.

The hosted web application is written on Python framework Flask, PostgreSQL
is used as a data storage, and Apache2 with mod_wsgi is the webserver of choice.

## Server summary

Server IP: 35.167.154.83

Website address: http://item-catalog.35.167.154.83.nip.io

### Initial setup

Login as `root`:

    ssh -i ~/.ssh/udacity_key.rsa root@35.167.154.83

Create a new user named `grader` and give it the permission to sudo:

    sudo adduser grader
    cd /etc/sudoers.d/
    echo 'grader ALL=(ALL) ALL' >> grader
    chmod 0440 grader

Allow public key authentication for `grader`:

    cd ~/.ssh
    mkdir /home/grader/.ssh
    cp authorized_keys /home/grader/.ssh
    chown grader:grader /home/grader/.ssh
    chmod 700 /home/grader/.ssh
    chown grader:grader /home/grader/.ssh/authorized_keys
    chmod 664 /home/grader/.ssh/authorized_keys

Set SSH port to 2200, disable root login and password authentication:

    nano /etc/ssh/sshd_config

    Port 2200
    PermitRootLogin no
    PasswordAuthentication no

Restart SSH daemon to apply changes:

    service ssh restart

Logout from SSH session

    exit

### Additional configuration

Login as `grader`:

    ssh -i ~/.ssh/udacity_key.rsa grader@35.167.154.83 -p 2200

Configure firewall:

    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow ntp
    sudo ufw allow www
    sudo ufw enable

Set local timezone:

    sudo unlink /etc/localtime
    sudo ln -s /usr/share/zoneinfo/UTC /etc/localtime

### Install required packages

Install Flask, PostgreSQL with its PL/Python extension, SQLAlchemy, and pip:

    sudo apt-get -qqy update &&
    sudo apt-get -qqy upgrade &&
    sudo apt-get -qqy install postgresql python-psycopg2 postgresql-plpython &&
    sudo apt-get -qqy install python-flask python-sqlalchemy &&
    sudo apt-get -qqy install python-pip

Install required pip packages:

    sudo pip install bleach &&
    sudo pip install oauth2client &&
    sudo pip install requests &&
    sudo pip install httplib2 &&
    sudo pip install redis &&
    sudo pip install passlib &&
    sudo pip install itsdangerous &&
    sudo pip install flask-httpauth

Install apache, mod_wsgi and git:

    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi
    sudo apt-get install git

### Add project files

Create project folders:

    cd /var/www
    sudo mkdir item-catalog
    sudo chown grader item-catalog
    sudo mkdir item-catalog-files
    sudo chown grader item-catalog-files

Create wsgi-script:

    cd item-catalog
    nano item-catalog.wsgi

```python
#!/usr/bin/python
import sys, os

WORKING_DIRECTORY = '/var/www/item-catalog-files'

sys.path.append(WORKING_DIRECTORY)
os.chdir(WORKING_DIRECTORY)

from werkzeug.debug import DebuggedApplication
from project import app

if app.debug:
    application = DebuggedApplication(app, True)
else:
    application = app
```

Copy project files to the server:

    cd ../item-catalog-files
    git clone https://github.com/anton-kachurin/item-catalog.git
    mv item-catalog/* .
    mv item-catalog/.g* .
    rmdir item-catalog

Create config file:

    nano config.py
    DEBUG = True # or False for production
    SECRET_KEY = 'a-secret-key-of-your-choice-here'

Create `g_client_secrets.json` and `jb_client_secrets.json`:

    see [step 5](https://github.com/anton-kachurin/item-catalog/blob/master/README.md#installation)

Create VirtualHost config:

    cd /etc/apache2/sites-available/
    sudo nano item-catalog.conf

```
<virtualhost *:80>
    ServerName item-catalog.35.167.154.83.nip.io

    WSGIDaemonProcess item-catalog user=grader group=grader threads=5
    WSGIProcessGroup item-catalog
    WSGIScriptAlias / /var/www/item-catalog/item-catalog.wsgi
    WSGIScriptReloading On

    <directory /var/www/item-catalog>
        Require all granted
    </directory>
</virtualhost>
```

Enable the config:

    sudo a2ensite item-catalog.conf

### Create and populate database

Create database:

```
sudo su postgres
psql

CREATE USER catalog WITH PASSWORD 'password';
CREATE DATABASE catalog;

\c catalog

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO catalog;
CREATE EXTENSION plpythonu;
UPDATE pg_language SET lanpltrusted = true WHERE lanname = 'plpythonu';
\q
exit
```

Populate database:

    cd /var/www/item-catalog-files
    chmod 764 populate_db.py
    ./populate_db.py -i

### Run the website

Restart apache to make the website available:

    sudo service apache2 reload

### Security wrap-up

Make sure that remote connections on PostgreSQL are not allowed:

    sudo cat /etc/postgresql/9.3/main/pg_hba.conf | grep ^[^#]

Force password complexity:

    sudo apt-get install libpam-cracklib

    in the file:

    sudo nano /etc/pam.d/common-password

    replace a line:

    password  requisite   pam_cracklib.so retry=3 minlen=8 difok=3

    with this line:

    password  requisite  pam_cracklib.so try_first_pass retry=3 minlength=16lcredit=-1 ucredit=-1 dcredit=-1 ocredit=-1 difok=4 reject_username

### Further maintenance

Whenever updates are available in the repository, run:

    cd /var/www/item-catalog-files
    git pull
    touch ../item-catalog/item-catalog.wsgi

## Third-Party Resources

https://www.techonthenet.com/linux/commands/chown.php

http://unix.stackexchange.com/questions/110522/timezone-setting-in-linux

http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/

https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps#do-not-allow-remote-connections

http://www.techrepublic.com/article/how-to-force-users-to-create-secure-passwords-on-linux/
