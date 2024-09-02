# Deploy_django_postgres_app
## 1. Общее описание проекта.
Проект представляет собой описание процесса деплоя приложения автоматического мониторинга цен на авиабилеты Flight_catcher с использованием Docker, Docker-compose. Предполагается локальная сборка образов каждого из компонентов проекта с загрузкой на Dockerhub и последующей сборкой многоконтейнерного приложения с использованием Docker-compose на сервере. 
Flight_catcher состоит из:
- Flight_catcher_web (контейнер1) - приложение на Django, представляющее собой веб-интерфейс для приема заявок на мониторинг цен и сохранения их в бд.
- база данных Postgres (контейнер2) - хранит данные о запросах пользвателей;
- Flight_catcher_parser + telegram agent (контейнер3) - берет данные о запросах пользователей из бд, парсит цены у авиаперевозчиков и отправляет результаты польователям в Телеграм.
- http-сервер (фронтенд, контейнер 4).
## 2. Создание образов.
Приложение будет разворачиваться в нескольких контейнерах:
* база данных;
* бэкенд;
* фронтенд.
### 2.1 Образ базы данных.

Dockerfile
```
FROM postgres:14
RUN chmod +x init_db.sh
COPY ./init_db.sh /docker-entrypoint-initdb.d/init_db.sh
```
Расширим образ посгреса с помощью скрипта init_db.sh, создающим базу данных "flight_search".  
Файлы, помещенные в папку /docker-entrypoint-initdb.d/ запускаются автоматически при создании образа.  
Команда "chmod +x" делает файл исполняемым (иногда работает и без этого).  
init_db.sh:  
```
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
SELECT "CREATE DATABASE flight_search" WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = "flight_search")\gexec
EOSQL
```

### 2.2 Образ бэкенда.
Dockerfile:  
```
FROM python:3.10
COPY requirements.txt requirements.txt
RUN python -m pip install --upgrade pip && pip install -r requirements.txt
WORKDIR /app
COPY ./flight_catcher/ ./flight_catcher/
COPY ./flight_search/ ./flight_search/
COPY ./data.py ./data.py
COPY ./manage.py ./manage.py
COPY ./entrypoint.sh ./entrypoint.sh
EXPOSE 8000
RUN chmod +x entrypoint.sh   # делаем файл entrypoint.sh исполняемым
ENTRYPOINT ["/app/entrypoint.sh"]
```
Сам файл entrypoint.sh содержит команды запуска миграций и заполнения базы данных:  
```
#! /bin/bash

python manage.py makemigrations --no-input
python manage.py migrate --no-input
python manage.py collectstatic --no-input
python manage.py load_cities

exec gunicorn flight_catcher.wsgi:application -b 0.0.0.0:8000 --reload
```
Соберем образ базы данных с именем catcher_db и тэгом 02. При этом нужно находиться в директории, где лежит наш докерфайл.
```
docker build -t catcher_db:02 .
```

### 2.3 Образ фронтенда.
Dockerfile:  
```
FROM nginx:1.27
RUN rm /etc/nginx/conf.d/default.conf   # удаляем дефолтный конфиг-файл nginx
COPY nginx.conf /etc/nginx/conf.d/   # копируем свой конфиг-файл в контейнер
```
Файл конфигурации nginx.conf:
 ```
server {
    listen 80;

    location / {
        proxy_pass http://backend:8000;
    }

    location /static/ {
        alias /static/;
    }

}
```
## 3. Поднятие конейнеров.
Контейнеры должны работать в изолированном сетевом пространстве, поэтому создадим сеть catcher_net для наших контейнеров:  
```
docker network create catcher_net
```
При падении контейнера данные, сохраненные в бд должны сохраняться. Поэтому создадим вольюм, в который будет копироваться информация из базы данных в контейнере:
```
docker volume create catcher_vol
```
Важен порядок поднятия контейнеров. Фронт сразу после поднятия обращается к бэку, поэтому последний к этому моменту должен уже работать. Бэк при поднятии запускает миграции бд и создает суперпользователя, поэтому к моменту его запуса уже должна функцинировать база данных. 
### 3.1 Поднятие базы данных.
Поднимем контейнер с именем database на основе образа catcher_db:02, реквизиты для создания базы данных передадим через переменные окружения -e:  
```
docker run -d --name database --net=catcher_net -v catcher_vol:/var/lib/postgresql/data -e POSTGRES_DB=<my_db_name> -e POSTGRES_USER=<some_user> -e POSTGRES_PASSWORD=<some_password> catcher_db:02
```

