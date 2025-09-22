## Настройка autovacuum с учетом особеностей производительности
### Развернул VM и установил Postgres 17:


### Подготавливаем pgbench:  
```
kopytax@UB25:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) of pgbench_accounts done (elapsed 0.11 s, remaini                                                                                vacuuming...
creating primary keys...
done in 0.88 s (drop tables 0.18 s, create tables 0.08 s, client-side generate 0.25 s, vacuum 0.16 s, primary keys 0.21 s).
```
### Запускаем тест:  
```   
kopytax@UB25:~$ sudo -u postgres  pgbench -c8 -P 6 -T 60
pgbench (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 257.6 tps, lat 30.672 ms stddev 23.264, 0 failed
progress: 12.0 s, 269.7 tps, lat 29.617 ms stddev 20.528, 0 failed
progress: 18.0 s, 273.4 tps, lat 29.290 ms stddev 21.124, 0 failed
progress: 24.0 s, 264.0 tps, lat 30.172 ms stddev 21.500, 0 failed
progress: 30.0 s, 268.5 tps, lat 29.878 ms stddev 21.460, 0 failed
progress: 36.0 s, 201.4 tps, lat 39.447 ms stddev 32.793, 0 failed
progress: 42.0 s, 170.3 tps, lat 47.252 ms stddev 42.169, 0 failed
progress: 48.0 s, 231.7 tps, lat 34.445 ms stddev 26.681, 0 failed
progress: 54.0 s, 200.3 tps, lat 39.912 ms stddev 33.754, 0 failed
progress: 60.0 s, 206.5 tps, lat 38.600 ms stddev 29.967, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 14068
number of failed transactions: 0 (0.000%)
latency average = 34.099 ms
latency stddev = 27.697 ms
initial connection time = 50.993 ms
tps = 234.336871 (without initial connection time)
kopytax@UB25:~$

```
### Применим параметры настройки PostgreSQL из прикрепленного к материалам занятия файла: 
```
 sudo nano /etc/postgresql/17/main/postgresql.conf

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

kopytax@UB25:~$ sudo systemctl restart postgresql@17-main
```

### Проводим тест с новыми параметрами:  
```
kopytax@UB25:~$ sudo -u postgres  pgbench -c8 -P 6 -T 60
pgbench (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 249.3 tps, lat 31.691 ms stddev 23.791, 0 failed
progress: 12.0 s, 247.8 tps, lat 32.177 ms stddev 23.299, 0 failed
progress: 18.0 s, 269.8 tps, lat 29.735 ms stddev 20.726, 0 failed
progress: 24.0 s, 256.2 tps, lat 30.983 ms stddev 23.317, 0 failed
progress: 30.0 s, 235.5 tps, lat 33.881 ms stddev 27.500, 0 failed
progress: 36.0 s, 236.1 tps, lat 33.954 ms stddev 28.558, 0 failed
progress: 42.0 s, 266.8 tps, lat 30.167 ms stddev 22.133, 0 failed
progress: 48.0 s, 257.3 tps, lat 30.960 ms stddev 23.726, 0 failed
progress: 54.0 s, 253.8 tps, lat 31.596 ms stddev 24.115, 0 failed
progress: 60.0 s, 259.0 tps, lat 30.827 ms stddev 24.106, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 15198
number of failed transactions: 0 (0.000%)
latency average = 31.563 ms
latency stddev = 24.198 ms
initial connection time = 49.999 ms
tps = 253.089164 (without initial connection time)

```
Количество tps увеличилось при повторном тесте. 
На производительность скорей всего повлияло:
- Резко сократила обращения к диску за счет увеличения кэша БД (shared_buffers).
- Позволили операциям в памяти работать быстрее за счет увеличения лимита памяти на операцию (work_mem).
 Оптимизация автовакуума во второй конфигурации произошла в основном за счёт увеличения параметра maintenance_work_mem с 128MB до 512MB, для его работы стало больше памяти.


### Создаем и заполняем таблицу:
```
kopytax@UB25:~$ sudo -u postgres psql
[sudo] password for kopytax:
psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
Type "help" for help.

postgres=# CREATE TABLE test(id INT, val char(50) );
ERROR:  relation "test" already exists
postgres=# CREATE TABLE test1(id INT, val char(50) );
CREATE TABLE
postgres=# INSERT INTO test1(val) SELECT 'testTest' FROM generate_series(1,1000000);
INSERT 0 1000000
```
### Hазмер таблицы:
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test1'));
 pg_size_pretty
----------------
 81 MB
(1 row)
```
### Делаем update и смотрим количество метрвых строк:
```
postgres=# update test1 set val=val||'a';
UPDATE 1000000
postgres=# update test1 set val=val||'b';
UPDATE 1000000
postgres=# update test1 set val=val||'c';
UPDATE 1000000
postgres=# update test1 set val=val||'d';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=#


postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test1';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test1   |    1000000 |    1999697 |    199 | 2025-09-22 16:13:52.653959+00
(1 row)
```

### Проверяем через некоторое время и видим, что количество мертвых строк стало 0:  
```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test1';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test1   |    1000000 |          0 |      0 | 2025-09-22 16:14:52.440887+00
(1 row)

```
### Проводим update повторно и смотрим размер таблицы:   
```

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test1'));
 pg_size_pretty
----------------
 403 MB
(1 row)


postgres=#
```
Размер таблицы увеличился.

### Отключаем autovacuum и делаем новые обновления :  
```
p
postgres=# ALTER TABLE test1 SET (autovacuum_enabled = off);
ALTER TABLE
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# update test1 set val=val||'f';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test1'));
 pg_size_pretty
----------------
 805 MB
(1 row)

```
Объем таблицы увеличился, это происходит из-за метрвых строк, которые как бы неочищаются.

### Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.:

```

postgres=# DO $$
DECLARE
    i INT;  -- счетчик цикла
BEGIN
    FOR i IN 1..10 LOOP  -- цикл от 1 до 10
        RAISE NOTICE 'Шаг цикла: %', i;  -- вывод номера шага

        UPDATE test1
        SET val = 'aaa';  -- операция обновления

    END LOOP;
END $$;
NOTICE:  Шаг цикла: 1
NOTICE:  Шаг цикла: 2
NOTICE:  Шаг цикла: 3
NOTICE:  Шаг цикла: 4
NOTICE:  Шаг цикла: 5
NOTICE:  Шаг цикла: 6
NOTICE:  Шаг цикла: 7
NOTICE:  Шаг цикла: 8
NOTICE:  Шаг цикла: 9
NOTICE:  Шаг цикла: 10
DO






```
