<p align="center">
  <h1 align="center">Docker_django_PostgreSQL_Gunicorn_Nginx</h1>
  <p align="center">
  <h3 align="center"> Django + PostgreSQL + Gunicorn + Nginx Docker containers</h3>
  </p>
</p>
<details open="open">
  <summary><h2 style="display: inline-block">Table of Contents</h2></summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#tech-stack">Tech stack</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#requirement">Requirement</a></li>

      </ul>
    </li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

This project tells how to configure Django and Postgres to run on Docker.


### Tech stack

* []()Django v3.1.5
* []()Python v3.8.3
* []()Docker v19.03.8

<!-- GETTING STARTED -->
## Getting Started
**Project Setup**
* Create a new project directory:
  ```sh
  mkdir django_docker && cd django_docker
  mkdir app && cd app
  python -m venv env
  source env/bin/activate
  pip install django==3.1.5
  django-admin.py startproject hello_django .
  python manage.py migrate
  python manage.py runserver
  ```
* Create a **requirements.txt** file in the **"app"** directory
  ```sh
  Django==3.1.5
  psycopg2-binary==2.8.5
  ```
* The Project directory should look like:
  ```sh
  └── app
    ├── hello_django
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── requirements.txt
  ```
**Docker**
* Add a Dockerfile to the "app" directory:
  ```sh
  FROM python:3.8.3-alpine

  WORKDIR /usr/src/app

  ENV PYTHONDONTWRITEBYTECODE 1
  ENV PYTHONUNBUFFERED 1

  # install psycopg2 dependencies
  RUN apk update \
      && apk add postgresql-dev gcc python3-dev musl-dev


  RUN pip install --upgrade pip
  COPY ./requirements.txt .
  RUN pip install -r requirements.txt

  # copy project
  COPY . .

  ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
    ```
* Create a **/entrypoint.sh** file in the **"app"** directory
  ```sh
  #!/bin/sh

  if [ "$DATABASE" = "postgres" ]
  then
      echo "Waiting for postgres..."

      while ! nc -z $SQL_HOST $SQL_PORT; do
        sleep 0.1
      done

      echo "PostgreSQL started"
  fi

  python manage.py flush --no-input
  python manage.py migrate

  exec "$@"
  ```
* Update the file permissions:
  ```sh
$ chmod +x app/entrypoint.sh
  ```
* Add a docker-compose.yml file to the project root:
  ```sh

  version: '3.7'

  services:
    web:
      build: ./app
      command: python manage.py runserver 0.0.0.0:8000
      volumes:
        - ./app/:/usr/src/app/
      ports:
        - 8000:8000
      env_file:
        - ./.env.dev
      depends_on:
        - db
    db:
      image: postgres:12.0-alpine
      volumes:
        - postgres_data:/var/lib/postgresql/data/
      environment:
        - POSTGRES_USER=hello_django
        - POSTGRES_PASSWORD=hello_django
        - POSTGRES_DB=hello_django_dev

  volumes:
    postgres_data:

  ```
