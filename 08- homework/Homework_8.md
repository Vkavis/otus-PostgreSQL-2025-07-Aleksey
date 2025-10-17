## Механизм блокировок

### Настраиваем сервер:  
```
Настройки sudo nano /etc/postgresql/17/main/postgresql.conf
log_lock_waits = on
deadlock_timeout = 200ms
```
Рестарт кластера:  
```
kopytax@UB25:/etc/postgresql/17/main$ sudo pg_ctlcluster 17 main restart
```

Создаем таблицу и заполняем данными:  
```
sudo -u postgres psql

CREATE TABLE users(id serial, name char(50));
CREATE TABLE
INSERT INTO users VALUES (1, 'Valera'), (2,'Petr'), (3,'Aleks');
INSERT 0 3
```

### Делаем симуляцию блокировки:

В певрой сессии делаем апдейт без коммита:
```
postgres=# \set AUTOCOMMIT off
postgres=# begin;
BEGIN
postgres=*# update users set name='Fedor' where id=2;
UPDATE 1

```
Во второй сессии делаем такой же апдейт:  
```
postgres=# update users set name='Fedor' where id=2;
^CCancel request sent
ERROR:  canceling statement due to user request
CONTEXT:  while updating tuple (0,2) in relation "users"
postgres=# /q
```
Сессия повисает, через некоторое время отменяем действие:  
```
^CCancel request sent
ERROR:  canceling statement due to user request
CONTEXT:  while updating tuple (0,2) in relation "accounts"
```

Провераем логи:
```
kopytax@UB25:~$ sudo cat /var/log/postgresql/postgresql-17-main.log
```
Здесь видим строки, которые говорят о наличии блокировок:
```
2025-10-17 17:24:17.658 UTC [1175] postgres@postgres LOG:  process 1175 still waiting for ShareLock on transaction 331438 after 201.239 ms
2025-10-17 17:24:17.658 UTC [1175] postgres@postgres DETAIL:  Process holding the lock: 1099. Wait queue: 1175.
2025-10-17 17:24:17.658 UTC [1175] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "users"
2025-10-17 17:24:17.658 UTC [1175] postgres@postgres STATEMENT:  update users set name='Fedor' where id=2;
2025-10-17 17:24:34.239 UTC [1175] postgres@postgres ERROR:  canceling statement due to user request
2025-10-17 17:24:34.239 UTC [1175] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "users"
2025-10-17 17:24:34.239 UTC [1175] postgres@postgres STATEMENT:  update users set name='Fedor' where id=2;
2025-10-17 17:24:51.963 UTC [840] LOG:  checkpoint starting: time
2025-10-17 17:24:52.410 UTC [840] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.007 s, sync=0.003 s, total=0.447 s; sync files=1, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=0 kB; lsn=2/4B4A2330, redo lsn=2/4B4A22D8

### Update в 3-х сессиях:

Запускаем update в первой сессии:
```

postgres=# \set AUTOCOMMIT off
postgres=# begin;
BEGIN
postgres=*# update users set name='Fedor' where id=1;
UPDATE 1

```
Затем такую команду еще в 2-x сессиях:
```
postgres=*# update users set name='Fedor' where id=1;
```
Смотрим select * FROM pg_locks;

   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+------+------------------+---------+----------+-------------------------------
 relation      |        5 |    24913 |      |       |            |               |         |       |          | 5/2                | 1223 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/2        |               |         |       |          | 5/2                | 1223 | ExclusiveLock    | t       | t        |
 relation      |        5 |    24913 |      |       |            |               |         |       |          | 6/2                | 1234 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 6/2        |               |         |       |          | 6/2                | 1234 | ExclusiveLock    | t       | t        |
 relation      |        5 |    24913 |      |       |            |               |         |       |          | 7/2                | 1305 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 7/2        |               |         |       |          | 7/2                | 1305 | ExclusiveLock    | t       | t        |
 relation      |        5 |    12073 |      |       |            |               |         |       |          | 8/3                | 1396 | AccessShareLock  | t       | t        |
 virtualxid    |          |          |      |       | 8/3        |               |         |       |          | 8/3                | 1396 | ExclusiveLock    | t       | t        |
 tuple         |        5 |    24913 |    0 |     1 |            |               |         |       |          | 6/2                | 1234 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        331441 |         |       |          | 6/2                | 1234 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        331440 |         |       |          | 6/2                | 1234 | ShareLock        | f       | f        | 2025-10-17 17:33:24.880448+00
 tuple         |        5 |    24913 |    0 |     1 |            |               |         |       |          | 7/2                | 1305 | ExclusiveLock    | f       | f        | 2025-10-17 17:34:06.041183+00
 transactionid |          |          |      |       |            |        331442 |         |       |          | 7/2                | 1305 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        331440 |         |       |          | 5/2                | 1223 | ExclusiveLock    | t       | f        |
  

Посмотрим с учетом зависимостей: 
SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'users'::regclass;

```postgres=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'users'::regclass;
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1223 | {}
 relation | RowExclusiveLock | t       | 1234 | {1223}
 relation | RowExclusiveLock | t       | 1305 | {1234}
 tuple    | ExclusiveLock    | t       | 1234 | {1223}
 tuple    | ExclusiveLock    | f       | 1305 | {1234}
(5 rows)
```

Видим, что PID 1223 начал транзакцию (331440) и заблокировал какие-то данные в таблице users
PID 1234 пытается изменить те же данные:
Получил блокировку на строку (tuple)
Но не может завершить операцию, т.к. ждет завершения транзакции 331440
PID 1305 также пытается работать с той же строкой:
Не может даже получить блокировку на строку, т.к. она уже занята PID 1234

### Анализ блокровок по логам.

Смортим логи:  
```
2025-10-17 17:33:25.082 UTC [1234] postgres@postgres LOG:  process 1234 still waiting for ShareLock on transaction 331440 after 201.744 ms
2025-10-17 17:33:25.082 UTC [1234] postgres@postgres DETAIL:  Process holding the lock: 1223. Wait queue: 1234.
2025-10-17 17:33:25.082 UTC [1234] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "users"
2025-10-17 17:33:25.082 UTC [1234] postgres@postgres STATEMENT:  update users set name='Fedor' where id=1;
2025-10-17 17:34:06.241 UTC [1305] postgres@postgres LOG:  process 1305 still waiting for ExclusiveLock on tuple (0,1) of relation 24913 of database 5 after 200.273 ms
2025-10-17 17:34:06.241 UTC [1305] postgres@postgres DETAIL:  Process holding the lock: 1234. Wait queue: 1305.
2025-10-17 17:34:06.241 UTC [1305] postgres@postgres STATEMENT:  update users set name='Fedor' where id=1;
2025-10-17 17:36:32.340 UTC [1396] postgres@postgres ERROR:  syntax error at or near "SELECT" at character 86
2025-10-17 17:36:32.340 UTC [1396] postgres@postgres STATEMENT:  SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for

```
Из логов видно, Процесс 1223 (держит ShareLock на транзакцию 331440)
    ↓ ждет процесс 1234
Процесс 1234 (держит ExclusiveLock на кортеж (0,1))
    ↓ ждет процесс 1305  
Процесс 1305 (заблокирован в очереди)

Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Ответ - Да.

### Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Ответ нет, одна транзакция будет ждать завершения другой, на крайний случай сработает deadlock detected.
