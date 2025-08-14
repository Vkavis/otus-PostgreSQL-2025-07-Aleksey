# Домашнее задание
Установка и настройка PostgteSQL в контейнере Docker
- Развернул докер, установим Docker Desktop на Windows
- -развернул контейнер с PostgreSQL 17 смонтировав в него именованный volume  с именем pgdata командой:
docker run --name pg-server -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v pgdata:/var/lib/postgresql/data postgres:17
![](2025-08-14_14-12-24.jpg)
- Установил развернул контейнер pgadmin на основе образа pgadmin командой 
docker run --name my-pgadmin -p 80:80 -v pgadmin:/var/lib/pgadmin -e PGADMIN_DEFAULT_EMAIL=aleksey.kopytin@abbott.com -e PGADMIN_DEFAULT_PASSWORD=pgadmin -d dpage/pgadmin4
![](2025-08-14_14-25-36.jpg)
