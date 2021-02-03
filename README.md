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
* Create a **entrypoint.sh** file in the **"app"** directory
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
  chmod +x app/entrypoint.sh
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
   You may comment out the **database flush** and **migrate** commands in the entrypoint.sh script so they don't run on every container start or re-start:
  ```sh     
if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

# python manage.py flush --no-input
# python manage.py migrate

exec "$@"
  ```
Instead, you can run them manually, after the containers spin up, like so:
  ```sh
  docker-compose exec web python manage.py flush --no-input
  docker-compose exec web python manage.py migrate
  ```
## Production
In production uses gunicorn + nginx.
Add Gunicorn in **requirements.txt** file:
```sh
Django==3.0.7
gunicorn==20.0.4
psycopg2-binary==2.8.5
```
**Nginx config:**
```sh
upstream hello_django {
  server web:8000;
}

server {

  listen 80;

  location / {
      proxy_pass http://hello_django;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $host;
      proxy_redirect off;
      client_max_body_size 100M;
  }

  location /staticfiles/ {
      alias /home/app/web/staticfiles/;
  }

  location /mediafiles/ {
      alias /home/app/web/mediafiles/;
  }

}
```
**docker-compose.prod.yml**
  ```sh
  version: '3.7'

  services:
    web:
      build: ./app
      command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
      ports:
        - 8000:8000
      env_file:
        - ./.env.prod
      depends_on:
        - db
    db:
      image: postgres:12.0-alpine
      volumes:
        - postgres_data:/var/lib/postgresql/data/
      env_file:
        - ./.env.prod.db

  volumes:
    postgres_data:
    ```
**.env.prod:**
  ```sh
  DEBUG=0
  SECRET_KEY=change_me
  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
  SQL_ENGINE=django.db.backends.postgresql
  SQL_DATABASE=hello_django_prod
  SQL_USER=hello_django
  SQL_PASSWORD=hello_django
  SQL_HOST=db
  SQL_PORT=5432
  DATABASE=postgres
  ```
**.env.prod.db:**
  ```sh
  POSTGRES_USER=hello_django
  POSTGRES_PASSWORD=hello_django
  POSTGRES_DB=hello_django_prod
  ```

**Dockerfile**
```sh
  FROM python:3.8.3-alpine as builder

  WORKDIR /usr/src/app

  ENV PYTHONDONTWRITEBYTECODE 1
  ENV PYTHONUNBUFFERED 1

  RUN apk update \
      && apk add postgresql-dev gcc python3-dev musl-dev

  RUN pip install --upgrade pip
  RUN pip install flake8
  COPY . .
  RUN flake8 --ignore=E501,F401 .

  COPY ./requirements.txt .
  RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt


  FROM python:3.8.3-alpine

  RUN mkdir -p /home/app

  RUN addgroup -S app && adduser -S app -G app

  ENV HOME=/home/app
  ENV APP_HOME=/home/app/web
  RUN mkdir $APP_HOME
  WORKDIR $APP_HOME

  RUN apk update && apk add libpq
  COPY --from=builder /usr/src/app/wheels /wheels
  COPY --from=builder /usr/src/app/requirements.txt .
  RUN pip install --no-cache /wheels/*

  COPY ./entrypoint.prod.sh $APP_HOME

  COPY . $APP_HOME

  RUN chown -R app:app $APP_HOME

  USER app

  ENV HOME=/home/app
  ENV APP_HOME=/home/app/web
  RUN mkdir $APP_HOME
  RUN mkdir $APP_HOME/staticfiles
  RUN mkdir $APP_HOME/mediafiles
  WORKDIR $APP_HOME

  ENTRYPOINT ["/home/app/web/entrypoint.prod.sh"]
```
**entrypoint.prod.sh:**
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

exec "$@"
```
**docker-compose.prod.yml**
```sh
  version: '3.7'

  services:
    web:
      build:
        context: ./app
        dockerfile: Dockerfile.prod
      command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
      volumes:
        - static_volume:/home/app/web/staticfiles
        - media_volume:/home/app/web/mediafiles
      expose:
        - 8000
      env_file:
        - ./.env.prod
      depends_on:
        - db
    db:
      image: postgres:12.0-alpine
      volumes:
        - postgres_data:/var/lib/postgresql/data/
      env_file:
        - ./.env.prod.db
    nginx:
      build: ./nginx
      volumes:
        - static_volume:/home/app/web/staticfiles
        - media_volume:/home/app/web/mediafiles
      ports:
        - 1337:80
      depends_on:
        - web

  volumes:
    postgres_data:
    static_volume:
    media_volume:
```
Project structure should look like:
```sh
  ├── .env.prod
  ├── .env.prod.db
  ├── .gitignore
  ├── app
  │   ├── Dockerfile
  │   ├── Dockerfile.prod
  │   ├── entrypoint.prod.sh
  │   ├── entrypoint.sh
  │   ├── hello_django
  │   │   ├── __init__.py
  │   │   ├── asgi.py
  │   │   ├── settings.py
  │   │   ├── urls.py
  │   │   └── wsgi.py
  │   ├── manage.py
  │   └── requirements.txt
  ├── docker-compose.prod.yml
  ├── docker-compose.yml
  └── nginx
      ├── Dockerfile
      └── nginx.conf
```
### Static Files
**settings.py:**
```sh
  STATIC_URL = "/staticfiles/"
  STATIC_ROOT = os.path.join(BASE_DIR, "staticfiles")
```
### Media Files
**settings.py:**
```sh
  INSTALLED_APPS = [
      "django.contrib.admin",
      "django.contrib.auth",
      "django.contrib.contenttypes",
      "django.contrib.sessions",
      "django.contrib.messages",
      "django.contrib.staticfiles",

      "upload",
  ]
```
**app/upload/views.py:**
```sh
  from django.shortcuts import render
  from django.core.files.storage import FileSystemStorage


  def image_upload(request):
      if request.method == "POST" and request.FILES["image_file"]:
          image_file = request.FILES["image_file"]
          fs = FileSystemStorage()
          filename = fs.save(image_file.name, image_file)
          image_url = fs.url(filename)
          print(image_url)
          return render(request, "upload.html", {
              "image_url": image_url
          })
      return render(request, "upload.html")
```
Add a new template **upload.html**:
```sh
  from django.shortcuts import render
  from django.core.files.storage import FileSystemStorage


  def image_upload(request):
      if request.method == "POST" and request.FILES["image_file"]:
          image_file = request.FILES["image_file"]
          fs = FileSystemStorage()
          filename = fs.save(image_file.name, image_file)
          image_url = fs.url(filename)
          print(image_url)
          return render(request, "upload.html", {
              "image_url": image_url
          })
      return render(request, "upload.html")
```
**app/hello_django/urls.py:**
```sh
  from django.contrib import admin
  from django.urls import path
  from django.conf import settings
  from django.conf.urls.static import static

  from upload.views import image_upload

  urlpatterns = [
      path("", image_upload, name="upload"),
      path("admin/", admin.site.urls),
  ]

  if bool(settings.DEBUG):
      urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

**app/hello_django/settings.py:**
```sh
  MEDIA_URL = "/mediafiles/"
  MEDIA_ROOT = os.path.join(BASE_DIR, "mediafiles")
```
  ### Development
