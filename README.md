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
ALTER ROLE myprojectuser SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
```
Exit out of the PostgreSQL prompt

`\q`
## Creating a Python Virtual Environment
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
