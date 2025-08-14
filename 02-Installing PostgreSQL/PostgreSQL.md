# Домашнее задание
Установка и настройка PostgteSQL в контейнере Docker:
- Развернул докер, установим Docker Desktop на Windows
- Создал сеть внутри докера командой docker network create my-network;
- развернул контейнер с PostgreSQL 17 смонтировав в него именованный volume  с именем pgdata командой:
docker run --name pg-server -e POSTGRES_PASSWORD=postgres --network my-network -d -p 5432:5432 -v pgdata:/var/lib/postgresql/data postgres:17
![](2025-08-14_14-12-24.jpg)
- Установил развернул контейнер pgadmin на основе образа pgadmin командой 
docker run --name my-pgadmin -p 5080:80 -v pgadmin:/var/lib/pgadmin -e PGADMIN_DEFAULT_EMAIL=aleksey.kopytin@abbott.com -e PGADMIN_DEFAULT_PASSWORD=pgadmin -d --network my-network  dpage/pgadmin4
![](2025-08-14_14-25-36.jpg)

- Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
![](2025-08-14_14-54-32.jpg)
![](2025-08-14_15-14-05.jpg)

- Подключился к контейнеру с сервером с ноутбука
![](2025-08-14_15-35-59.jpg)

-Удалил контейнер с сервером создать его заново.
docker rm pg-server
![](2025-08-14_15-42-29.jpg)

- База и таблица остались целые.