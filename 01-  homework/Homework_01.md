# Домашнее задание
# Работа с уровнями изоляции транзакции в PostgreSQL
1. Установил Ubuntu в Oracle Virual Box
2. Далее провел установку PostgreSQL 17

3. Выключаем auto commit:
```
postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```
4. В первой сессии создаем новую таблицу и наполняем ее данными:
``` 
postgres=#  create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```
5. Посмотрим текущий уровень изоляции: show transaction isolation level
```
postgres=# SHOW transaction_isolation;
 transaction_isolation
-----------------------
 read committed
(1 row)

```
6. Начинаем новую транзакции:
```
postgres=# begin;
BEGIN

```
7. В первой сессии добавить новую запись:
```
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1

```
8. Cделать select from persons во второй сессии:
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```  
Новой записи нет, так как  нет коммита в первой сессии на вставку новой строки, а уровень изоляции стоит  read committed.

9. Завершаем первую транзакцию:  
```
postgres=*# commit;
COMMIT

```
10. Делаем select * from persons во второй сессии:
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
 ```
  Видим новую запись, так как  мы сделали commit  и запись теперь доступна для чтения (уровень изоляции  read committed).

11. Начинаем новые, но уже repeatable read транзации - set transaction isolation level repeatable read;
```
BEGIN;
postgres=*# set transaction isolation level repeatable read;
SET

```
12. Добавляем в первой сессии новую запись insert into persons(first_name, second_name) values('svaeta', 'svetova');

```
insert into persons(first_name, second_name) values('svaeta', 'svetova');
INSERT 0 1

```
13. Выполним запрос во второй сессии:
```

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

```
Новой строки нет, так как уровень изоляции repeatable read при котором  не видны изменения данных из первой сессии.  

14. Завершаем первую транзакцию:
```
postgres=*# commit;
COMMIT
```
15. Выполняем запрос во второй сессии:
``` 
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev

(3 rows)

``` 
Нам до сих пор не доступны новые данные из первой сессии, поскольку мы находимся внутри своей сессии с уровенем изоляции repeatable read.  

16. Завершаем вторую транзацию:
``` 
postgres=*# commit;
COMMIT
```
17. Выполняем запрос во второй сессии:
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
 11 | svaeta     | svetova
(4 rows)
```
Мы видим 4 строки поскольку мы закончили  транзакцию и  теперь видны все изменения из первой сессии.