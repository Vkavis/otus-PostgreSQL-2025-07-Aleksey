## ДЗ: ДЗ: Бэкапы

###  Шаг 1: Создание БД, схемы и таблицы
```
sudo -u postgres psql
-- Создаем базу данных
CREATE DATABASE homework_backup;

-- Подключаемся к ней
\c homework_backup

-- Создаем схему
CREATE SCHEMA hw_schema;

-- Создаем таблицу в схеме
CREATE TABLE hw_schema.table1 (
    id SERIAL PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

###  Шаг 2: Заполнение таблицы 100 автосгенерированными записями
```
INSERT INTO hw_schema.table1 (name)
SELECT 'User ' || generate_series(1, 100);
```
Проверим:
```
homework_backup=# SELECT COUNT(*) FROM hw_schema.table1;
 count
-------
   100
(1 row)
```
###  Шаг 3: Создание каталога для бэкапов 
```
sudo mkdir -p /var/backups/postgres
sudo chown postgres:postgres /var/backups/postgres
sudo chmod 700 /var/backups/postgres
```
### Шаг 4: Логический бэкап с помощью COPY

Создадим резервную копию таблицы table1 в формате CSV 
```
kopytax@UB25:~$ sudo -u postgres psql -d homework_backup -c "\COPY hw_schema.table1 TO '/var/backups/postgres/table1_copy_backup.csv' WITH CSV HEADER"
COPY 100
```
### Шаг 5: Восстановление данных из бэкапа в новую таблицу
Cначала создадим вторую таблицу с такой же структурой:
```
sudo -u postgres psql
\c homework_backup

CREATE TABLE hw_schema.table2 (
    id SERIAL PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

Теперь восстановим данные:
```
kopytax@UB25:~$ sudo -u postgres psql -d homework_backup -c "\COPY hw_schema.table2 FROM '/var/backups/postgres/table1_copy_backup.csv' WITH CSV HEADER"
COPY 100
```
Проверим:
```
postgres=# \c homework_backup
You are now connected to database "homework_backup" as user "postgres".
homework_backup=# SELECT COUNT(*) FROM hw_schema.table2;
 count
-------
   100
(1 row)
```

### Шаг 6: Бэкап двух таблиц с помощью pg_dump в кастомном сжатом формате
Создадим бэкап обеих таблиц (table1 и table2) в кастомном формате:
```
sudo -u postgres pg_dump -U postgres -d homework_backup -n hw_schema -F c -f /var/backups/postgres/homework_backup_custom.dump
```
### Шаг 7: Создание новой БД для восстановления
```
sudo -u postgres createdb homework_restore
создадим схему
CREATE SCHEMA hw_schema;
```
 ### Шаг 8: Восстановление только второй таблицы (table2) в новую БД

Теперь восстановим только table2:
```
sudo -u postgres pg_restore -U postgres -d homework_restore --table=table2 -n hw_schema /var/backups/postgres/homework_backup_custom.dump
```

Проверим восстановление:
```
kopytax@UB25:~$ sudo -u postgres psql
psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c homework_backup
You are now connected to database "homework_backup" as user "postgres".
homework_backup=# SELECT COUNT(*) FROM hw_schema.table2;
 count
-------
   100
(1 row)
```

