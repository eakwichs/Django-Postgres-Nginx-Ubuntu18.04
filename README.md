# Django-Postgres-Nginx-Ubuntu18.04
How To Set Up Django with Postgres, Nginx, and Gunicorn on Ubuntu 18.04
## Using Django with Python 3
```
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y autoremove
sudo apt-get -y install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```
## Creating the PostgreSQL Database and User
Log into an interactive Postgres session

`sudo -u postgres psql`

Create a database

`CREATE DATABASE myproject;`

Create a database user
```
CREATE USER myprojectuser WITH PASSWORD 'password';
ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myprojectuser SET timezone TO 'Asia/Bangkok';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
```
Exit out of the PostgreSQL prompt

`\q`
## Creating a Python Virtual Environment
upgrade pip and install virtualenv
```
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv
```
Create and move into a directory where we can keep our Python virtual environment
```
mkdir ~/envs
cd ~/envs
```
Create a Python virtual environment

`virtualenv myprojectenv`

Activate the virtual environment

`source myprojectenv/bin/activate`

Install Django, Gunicorn, and the psycopg2 PostgreSQL adaptor with the local instance of pip

`pip install django gunicorn psycopg2-binary`
## Creating and Configuring a New Django Project
Create and move into a directory where we can keep our project files
```
mkdir ~/websites
cd ~/websites
```
Creating the Django Project

```
django-admin startproject myproject
cd myproject
```

## Adjusting the Project Settings
Open the settings file in your text editor

`sudo nano ~/websites/myproject/myproject/settings.py`


```
...
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = ['*']
```


```
...

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'Asia/Bangkok'

```


```

...

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'myproject',
        'USER': 'myprojectuser',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```


```
...

STATIC_URL = '/myproject/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```
Migrate the initial database schema to our PostgreSQL database
```
python manage.py makemigrations
python manage.py migrate
```
Create an initial user named `admin` with a password of `password123`

`python manage.py createsuperuser --email admin@example.com --username admin`

Collect all of the static content into the directory location

`python manage.py collectstatic`

Test our your project by starting up the Django development server

`python manage.py runserver`

Open web browser with http://127.0.0.1:8000/


Testing Gunicorn’s Ability to Serve the Project

`gunicorn myproject.wsgi`

Deactivating virtualenvs

`deactivate`

## Creating systemd Socket and Service Files
Creating and opening a systemd socket file

`sudo nano /etc/systemd/system/myproject.socket`

```
[Unit]
Description=gunicorn socket for myproject

[Socket]
ListenStream=/run/myproject.sock

[Install]
WantedBy=sockets.target
```

Create and open a systemd service file

`sudo nano /etc/systemd/system/myproject.service`

```
[Unit]
Description=gunicorn daemon for myproject
Requires=myproject.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/websites/myproject
ExecStart=/home/ubuntu/envs/myprojectenv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/myproject.sock \
          myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```
We can now start and enable the socket. This will create the socket file at /run/myproject.sock now and at boot. When a connection is made to that socket, systemd will automatically start the myproject.service to handle it
```
sudo systemctl start myproject.socket
sudo systemctl enable myproject.socket
```
## Checking for the Gunicorn Socket File
Check the status of the process to find out whether it was able to start

`sudo systemctl status myproject.socket`

Next, check for the existence of the `myproject.sock` file within the `/run` directory

`file /run/myproject.sock`

> /run/myproject.sock: socket

Check the socket’s logs

`sudo journalctl -u myproject.socket`

## Testing Socket Activation
Currently, if you’ve only started the `myproject.socket` unit, the `myproject.service` will not be active yet since the socket has not yet received any connections. You can check this by typing

`sudo systemctl status myproject`

> - myproject.service - gunicorn daemon for myproject
   Loaded: loaded (/etc/systemd/system/myproject.service; disabled; vendor prese
   Active: inactive (dead)
