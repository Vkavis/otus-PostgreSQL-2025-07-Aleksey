## Логический уровень PostgreSQL
1. Установил  PostgreSQL 17 и проверил, что кластер работает.
```
kopytax@UB25:~$ sudo systemctl status postgresql@17-main
[sudo] password for kopytax:
● postgresql@17-main.service - PostgreSQL Cluster 17-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; >
     Active: active (running) since Thu 2025-09-04 11:21:14 UTC; 4h 37min ago
    Process: 668 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 17-main>
   Main PID: 803 (postgres)
      Tasks: 6 (limit: 2268)
     Memory: 46.8M (peak: 52.0M)
        CPU: 5.758s
     CGroup: /system.slice/system-postgresql.slice/postgresql@17-main.service
             ├─803 /usr/lib/postgresql/17/bin/postgres -D /mnt/postgres/17/main -c >
             ├─859 "postgres: 17/main: checkpointer "
             ├─860 "postgres: 17/main: background writer "
             ├─888 "postgres: 17/main: walwriter "
             ├─889 "postgres: 17/main: autovacuum launcher "
             └─890 "postgres: 17/main: logical replication launcher "
```

2. Создаем новую БД данных testdb: 
```
postgres=# create database testdb;
CREATE DATABASE
```
3. Заходим в созданную базу данных под пользователем postgres
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
3. Создаем новую схему testnm
```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```
4. Создаём новую таблицу t1 с одной колонкой c1 типа integer и вставляем строку со значением c1=1

```
testdb=# CREATE TABLE testnm.test_tbl(col1 integer);
CREATE TABLE
testdb=#  INSERT INTO testnm.test_tbl values(1);
INSERT 0 1
testdb=#
```
5. Создаём новую роль readonly
```
testdb=# CREATE role readonly;
CREATE ROLE
```
6. Даю новой роли право на подключение к базе данных testdb
```
testdb=# grant connect on DATABASE testdb to readonly;
GRANT
```
7. Даю новой роли право на использование схемы testnm

```
testdb=# grant usage on SCHEMA testnm to readonly;
GRANT
```
8. Даю  новой роли право на select для всех таблиц схемы testnm
```
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
```
9. Создаём пользователя и назначаем ему роль:  
```
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
testdb=# grant readonly TO testread;
GRANT ROLE
testdb=#
```
10. При попытке зайти под новым пользователем получаем ошибку
```
postgres=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```
11. Чтобы исправить ошибку правим файл pg_hba.conf и перезапускаем кластер:  
```
sudo nano /etc/postgresql/17/main/pg_hba.conf 

# "local" is for Unix domain socket connections only
local   all             all                                     md5
```
- Перезапускаем кластер:
```
kopytax@UB25:~$ sudo systemctl restart postgresql@17-main
```
12.  Заходим под пользователем в БД и делаем select:  
```
postgres=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>
testdb=> select * from testnm.test_tbl;
 col1
------
    1
(1 row)
```


13. Пробуем выполнить команду create table t2(c1 integer); insert into t2 values (2);
```
testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
```
- Нет прав на схему public:
Хотя по судя по тексту задания, должны были быть,но в моем случае пользователь их не получил

-  Проверим более точно права
```
testdb=> SELECT
    has_schema_privilege('public', 'USAGE') AS can_usage,
    has_schema_privilege('public', 'CREATE') AS can_create;
 can_usage | can_create
-----------+------------
 t         | f
(1 row)
```
- Видно, что на создание прав нет...

