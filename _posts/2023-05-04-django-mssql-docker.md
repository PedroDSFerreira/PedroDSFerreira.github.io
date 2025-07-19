---
title:  "Django + MSSQL with Docker"
date:   2023-05-04
# cover: "/posts/django-mssql-docker/cover.png"
tags: [django, mssql, docker, docker-compose]
---
In this post, I will explain how to set up Django with MSSQL using Docker Compose.


### Pre-requisites

Before we start, make sure you have the following tools installed on your system:
- Docker
- Docker Compose

### Step 1: Create a Django project

The first step is to create a new Django project. Open a terminal and run the following command:
```bash
python -m django startproject myproject
```

This will create a new Django project called "myproject". Navigate into the project directory:
```bash
cd myproject
```

### Step 2: Create dependencies
Next, we need to create the required dependencies for Django to work with MSSQL. We will be using the `mssql-django` library to connect to the database. 

Add the following lines to the `requirements.txt` file in your project directory:

```
Django==4.1.9
mssql-django==1.2
pyodbc==4.0.39
```

### Step 3: Create a Dockerfile
Create a new file `Dockerfile` in your project directory with the following contents:

```Dockerfile
# syntax=docker/dockerfile:1
FROM python:3

# set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
WORKDIR /app
COPY . .

# mssql dependency (Debian 11)
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -
RUN curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list
RUN apt-get update
RUN ACCEPT_EULA=Y apt-get install msodbcsql17 -y



# python dependencies
RUN pip install --no-cache-dir --upgrade -r requirements.txt
```

This Dockerfile will create a container with Python 3 installed and copy the contents of the current directory into `/app` the container, install the required Python dependencies listed above, using `pip`, and the Microsoft ODBC driver 17 (see [this](https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15#ubuntu17), using Debian 11 image).

### Step 4: Create a docker-compose.yml file
Create a new file `docker-compose.yml` in your project directory with the following contents:

```yml
version: '3'

services:
  bd:
    image: mcr.microsoft.com/mssql/server:2022-latest 
    environment:
      - MSSQL_SA_PASSWORD=MyPass@word
      - ACCEPT_EULA=Y
    ports:
      - "1433:1433"

    
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    environment:
      - DB_HOST=bd
      - DB_NAME=master
      - DB_USER=sa
      - DB_PASS=MyPass@word
    ports:
      - "8000:8000"
    depends_on:
      - bd
```

This docker-compose.yml file defines two services: `db` and `web`. The `db` service uses the official MSSQL Server 2022 image, sets the SA password and accepts the end-user license agreement. The port 1433 is exposed to allow connections from other containers. 

The `web` service builds the Dockerfile we created earlier and runs the Django app, exposing port 8000. It also sets the database connection settings using environment variables.


At this point, the folder structure should look like this:

```
myproject
  ├── Dockerfile
  ├── docker-compose.yml
  ├── myproject/
  │     ├── asgi.py
  │     ├── __init__.py
  │     ├── settings.py
  │     ├── urls.py
  │     └── wsgi.py
  ├── manage.py
  └── requirements.txt
```


### Step 5: Update the Django Settings
Once you have installed and configured the ODBC driver, you can update the Django settings to use MSSQL as the database backend. In your Django settings file (`myproject/myproject/settings.py`), add the following lines:

```python
import os


DATABASES = {
    'default': {
        'ENGINE': 'mssql',
        'HOST': os.environ.get('DB_HOST'),
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASS'),
        'PORT': '1433',

        'OPTIONS': {
            'driver': 'ODBC Driver 17 for SQL Server',
        },
    },
}

DATABASE_CONNECTION_POOLING = False
```

Make sure to update the HOST, NAME, USER, PASSWORD, and OPTIONS settings based on your configuration.

### Step 6: Run Docker Compose
Now that we have everything set up, we can run the following command to start the containers:

```bash
docker compose up
```


That's it! You should now be able to open [http://localhost:8000](http://localhost:8000) to see the Django app running.