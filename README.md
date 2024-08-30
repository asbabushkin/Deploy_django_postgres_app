# Deploy_django_postgres_app
## 1. Общее описание проекта.
Проект представляет собой описание процесса деплоя приложения автоматического мониторинга цен на авиабилеты Flight_catcher с использованием Docker, Docker-compose. Предполагается локальная сборка образов каждого из компонентов проекта с загрузкой на Dockerhub и последующей сборкой многоконтейнерного приложения с использованием Docker-compose на сервере. 
Flight_catcher состоит из:
- Flight_catcher_web (контейнер1) - приложение на Django, представляющее собой веб-интерфейс для приема заявок на мониторинг цен и сохранения их в бд.
- база данных Postgres (контейнер2) - хранит данные о запросах пользвателей;
- Flight_catcher_parser + telegram agent (контейнер3) - берет данные о запросах пользователей из бд, парсит цены у авиаперевозчиков и отправляет результаты польователям в Телеграм.
- http-сервер (фронтенд, контейнер 4).
## 2 Создание образов.
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

### 2.2 Образ фронтенд Nginx.
