# Django-Postgres-Nginx-Ubuntu18.04
How To Set Up Django with Postgres, Nginx, and Gunicorn on Ubuntu 18.04
## Using Django with Python 3
```
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y autoremove
sudo apt-get -y install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```

<br><br>
## Creating the PostgreSQL Database and User
Log into an interactive Postgres session

`sudo -u postgres psql`

<br><br>
Create a database

`CREATE DATABASE myproject;`

<br><br>
Create a database user
```
CREATE USER myprojectuser WITH PASSWORD 'password';
ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myprojectuser SET timezone TO 'Asia/Bangkok';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
```

<br><br>
Exit out of the PostgreSQL prompt

`\q`

<br><br>
## Creating a Python Virtual Environment
upgrade pip and install virtualenv
```
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv
```

<br><br>
Create and move into a directory where we can keep our Python virtual environment
```
mkdir ~/envs
cd ~/envs
```

<br><br>
Create a Python virtual environment

`virtualenv myprojectenv`

<br><br>
Activate the virtual environment

`source myprojectenv/bin/activate`

<br><br>
Install Django, Gunicorn, and the psycopg2 PostgreSQL adaptor with the local instance of pip

`pip install django gunicorn psycopg2-binary`

<br><br>
## Creating and Configuring a New Django Project
Create and move into a directory where we can keep our project files
```
mkdir ~/websites
cd ~/websites
```

<br><br>
Creating the Django Project
```
django-admin startproject myproject
cd myproject
```

<br><br>
## Adjusting the Project Settings
Open the settings file in your text editor

`sudo nano ~/websites/myproject/myproject/settings.py`

<br><br>
```
...
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = ['*']
```

<br><br>
```
...

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'Asia/Bangkok'

```

<br><br>
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

<br><br>
```
...

STATIC_URL = '/myproject/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

<br><br>
Migrate the initial database schema to our PostgreSQL database
```
python manage.py makemigrations
python manage.py migrate
```

<br><br>
Create an initial user named `admin` with a password of `password123`

`python manage.py createsuperuser --email admin@example.com --username admin`

<br><br>
Edit file `~/websites/myproject/myproject/urls.py`

`sudo nano ~/websites/myproject/myproject/urls.py`

```
urlpatterns = [
    path('myproject/admin/', admin.site.urls),
]
```

<br><br>
Collect all of the static content into the directory location

`python manage.py collectstatic`

<br><br>
Test our your project by starting up the Django development server

`python manage.py runserver`

<br><br>
Open web browser with http://127.0.0.1:8000/

<br><br>
Testing Gunicorn’s Ability to Serve the Project

`gunicorn myproject.wsgi`

<br><br>
Deactivating virtualenvs

`deactivate`

<br><br>
## Creating systemd Socket and Service Files
Creating and opening a systemd socket file

`sudo nano /etc/systemd/system/myproject.socket`

<br><br>
```
[Unit]
Description=gunicorn socket for myproject

[Socket]
ListenStream=/run/myproject.sock

[Install]
WantedBy=sockets.target
```

<br><br>
Create and open a systemd service file

`sudo nano /etc/systemd/system/myproject.service`

<br><br>
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

<br><br>
We can now start and enable the socket. This will create the socket file at /run/myproject.sock now and at boot. When a connection is made to that socket, systemd will automatically start the myproject.service to handle it
```
sudo systemctl start myproject.socket
sudo systemctl enable myproject.socket
```

<br><br>
## Checking for the Gunicorn Socket File
Check the status of the process to find out whether it was able to start

`sudo systemctl status myproject.socket`

<br><br>
Next, check for the existence of the `myproject.sock` file within the `/run` directory

`file /run/myproject.sock`

> /run/myproject.sock: socket

<br><br>
Check the socket’s logs

`sudo journalctl -u myproject.socket`

<br><br>
## Testing Socket Activation
Currently, if you’ve only started the `myproject.socket` unit, the `myproject.service` will not be active yet since the socket has not yet received any connections. You can check this by typing

`sudo systemctl status myproject`

> - myproject.service - gunicorn daemon for myproject<br>Loaded: loaded (/etc/systemd/system/myproject.service; disabled; vendor prese<br>Active: inactive (dead)

<br><br>
To test the socket activation mechanism, we can send a connection to the socket through `curl`

`curl --unix-socket /run/myproject.sock localhost`

<br><br>
You should see the HTML output from your application in the terminal. This indicates that Gunicorn was started and was able to serve your Django application. You can verify that the Gunicorn service is running by typing

`sudo systemctl status myproject`

> - myproject.service - gunicorn daemon for myproject<br>   Loaded: loaded (/etc/systemd/system/myproject.service; disabled; vendor prese<br>   Active: active (running) since Tue 2019-11-19 16:16:19 +07; 1min 43s ago<br> Main PID: 14995 (gunicorn)<br>    Tasks: 4 (limit: 4673)<br>   CGroup: /system.slice/myproject.service<br>           ├─14995 /home/ubuntu/envs/myprojectenv/bin/python3 /home/ubuntu/envs/<br>           ├─14999 /home/ubuntu/envs/myprojectenv/bin/python3 /home/ubuntu/envs/<br>           ├─15001 /home/ubuntu/envs/myprojectenv/bin/python3 /home/ubuntu/envs/<br>           └─15002 /home/ubuntu/envs/myprojectenv/bin/python3 /home/ubuntu/envs/<br><br>พ.ย. 19 16:16:19 ubuntu-VirtualBox systemd[1]: Started gunicorn daemon for mypro<br>พ.ย. 19 16:16:19 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:19 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:19 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:19 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:19 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:20 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]: [2019-11-19 16:16:20 +0700]<br>พ.ย. 19 16:16:20 ubuntu-VirtualBox gunicorn[14995]:  - - [19/Nov/2019:16:16:20 +

<br><br>
If the output from `curl` or the output of `systemctl status` indicates that a problem occurred, check the logs for additional details

`sudo journalctl -u myproject`

<br><br>
Check your `/etc/systemd/system/myproject.service` file for problems. If you make changes to the `/etc/systemd/system/myproject.service` file, reload the daemon to reread the service definition and restart the Gunicorn process by typing

```
sudo systemctl daemon-reload
sudo systemctl restart myproject
```

<br><br>
## Configure Nginx to Proxy Pass to Gunicorn
Edit file `/etc/nginx/sites-available/default`

`sudo nano /etc/nginx/sites-available/default`

<br><br>
```
server {
    .
    .
    .
    
    location /myproject/static/ {
        alias /home/ubuntu/websites/myproject/static/;
    }

    location /myproject {
        include /etc/nginx/uwsgi_params;
        proxy_pass http://unix:/run/myproject.sock;
        uwsgi_param Host $host;
        uwsgi_param X-Real-IP $remote_addr;
        uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
        uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;
    }
    
    location / {
    
    .
    .
    .
}
```

<br><br>
Test your Nginx configuration for syntax errors

`sudo nginx -t`

<br><br>
If no errors are reported, go ahead and restart Nginx

`sudo systemctl restart nginx`

<br><br>
Open web browser with http://127.0.0.1/myproject/admin

<br><br>
## Summary
After edit code in Django's project, do this
```
sudo systemctl daemon-reload
sudo systemctl restart myproject
```

<br><br>
After edit systemd Socket and Service file, do this
```
sudo systemctl daemon-reload
sudo systemctl restart myproject
```
