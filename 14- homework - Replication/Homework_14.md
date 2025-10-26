## ДЗ: Репликация

Шаг 1: Подготовка всех ВМ

1.1 Установливаем  PostgreSQL на всех VM

1.2 Настрйка postgresql.conf:
```
sudo nano /etc/postgresql/17/main/postgresql.conf
```

Включаем следующие параметры:
```
listen_addresses = '*'          # или укажите конкретные IP
wal_level = logical             # обязательно для логической репликации
max_replication_slots = 10      # достаточно для 3 ВМ
max_wal_senders = 10
```

1.3 Настрока pg_hba.conf (разрешите подключения от других ВМ):
```
sudo nano /etc/postgresql/17/main/pg_hba.conf
host    replication     all             0.0.0.0/0               md5
host    all             all             0.0.0.0/0               md5
```

1.4 Перезапустим PostgreSQL:
```
sudo systemctl restart postgresql
```

Шаг 2.1: Создание пользователя репликации
- На каждой ВМ выполним от имени postgres:
```
sudo -u postgres psql

CREATE USER repuser WITH REPLICATION PASSWORD 'x500';
-- Права на нужные базы 
GRANT ALL PRIVILEGES ON DATABASE postgres TO repuser;
```
Шаг 2.2: Создание таблиц
- На каждой VM создадим таблицы
```
 CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data TEXT
);

 CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data TEXT
);
```


Шаг 3: Настройка ВМ1

3.1 Создаём публикацию для таблицы test:
```
CREATE PUBLICATION pub_test FOR TABLE test;
```
3.2 Подписываемся на test2 из ВМ2:
```
CREATE SUBSCRIPTION sub_test2
CONNECTION 'host=192.168.1.11 port=5432 user=repuser password='x500' dbname=postgres'
PUBLICATION pub_test2;

WARNING:  publication "pub_test2" does not exist on the publisher
NOTICE:  created replication slot "sub_test2" on publisher
CREATE SUBSCRIPTION
```
Шаг 4: Настройка ВМ2
4.1 Создайте публикацию:
```
CREATE PUBLICATION pub_test2 FOR TABLE test2;
```
4.2 Полписываемся на test из ВМ1:
```
CREATE SUBSCRIPTION sub_test
CONNECTION 'host=192.168.1.10 port=5432 user=repuser password=x500 dbname=postgres'
PUBLICATION pub_test;
```

Шаг 5: Настройка ВМ3 (только чтение)
Подписываемся на обе публикации:
```
-- Подписка на test из ВМ1
CREATE SUBSCRIPTION sub_test_from_vm1
CONNECTION 'host=192.168.1.10 port=5432 user=repuser password=x500 dbname=postgres'
PUBLICATION pub_test;

-- Подписка на test2 из ВМ2
CREATE SUBSCRIPTION sub_test2_from_vm2
CONNECTION 'host=192.168.1.11 port=5432 user=repuser password=x500 dbname=postgres'
PUBLICATION pub_test2;
```
Шаг 6: Проверка работы
На ВМ1 вставим данные:
```
INSERT INTO test (data) VALUES ('Hello from VM1');
```

На ВМ2 вставим данные:
```
INSERT INTO test2 (data) VALUES ('Hello from VM2');
```
Проверяем на ВМ2
```
select * from test;
```
Данные не появились, при поиске причин выяснилось, что сначала я выдал права, а потом создал таблицы и repuser не имел прав, на новые таблицы.
Выдал права на каждой VM. 
```
GRANT SELECT ON test, test2 TO repuser;
```
Проверяем повторно
```
postgres=# select * from test;
 id |      data
----+----------------
  1 | Hello from VM1
(1 rows)
```

Проверьте на ВМ3
```
postgres=# SELECT * FROM test;
 id |      data
----+----------------
  1 | Hello from VM1

(1 rows)

postgres=# SELECT * FROM test2;
 id |      data
----+----------------
  1 | Hello from VM2
(1 rows)
```


