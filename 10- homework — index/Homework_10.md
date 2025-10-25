## ДЗ: Работа с индексами

###   Подготовка: Тестовая таблица и данные
-  1. Создание таблицы products
```
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    category VARCHAR(100),
    price NUMERIC(10, 2),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```
- 2. Наполнение таблицы 100 000 тестовыми записями
```
WITH 
names AS (
    SELECT unnest(ARRAY[
        'Smartphone', 'Laptop', 'Headphones', 'Tablet', 'Smartwatch',
        'Camera', 'Printer', 'Monitor', 'Keyboard', 'Mouse',
        'Router', 'Speaker', 'Drone', 'Projector', 'Charger'
    ]) AS n
),
categories AS (
    SELECT unnest(ARRAY[
        'Electronics', 'Computers', 'Audio', 'Photography', 'Accessories'
    ]) AS c
),
descriptions AS (
    SELECT unnest(ARRAY[
        'High-performance device for everyday use.',
        'Perfect for work and entertainment.',
        'Crystal-clear sound and long battery life.',
        'Compact and powerful with great display.',
        'Reliable and durable with modern design.',
        'Ideal for professionals and hobbyists.',
        'Fast, efficient, and easy to set up.',
        'Sleek design with premium build quality.'
    ]) AS d
)
INSERT INTO products (name, description, category, price, created_at, updated_at)
SELECT
    n.n || ' ' || (random() * 1000)::int AS name,
    d.d AS description,
    c.c AS category,
    ROUND((10 + random() * 1990)::numeric, 2) AS price,
    NOW() - (random() * 365 * '1 day'::interval) AS created_at,
    NOW() - (random() * 30 * '1 day'::interval) AS updated_at
FROM
    generate_series(1, 100000) AS id,
    (SELECT n FROM names ORDER BY random() LIMIT 1) AS n,
    (SELECT c FROM categories ORDER BY random() LIMIT 1) AS c,
    (SELECT d FROM descriptions ORDER BY random() LIMIT 1) AS d;

```
 Выпониим ANALYZE products; 
    Это обновит статистику для оптимизатора запросов — тогда EXPLAIN будет давать более точные оценки.

### Выполнение задания
#### 1. Индекс на одно поле
Цель: Ускорить фильтрацию по категории.

```
CREATE INDEX idx_products_category ON products (category);
```
 Комментарий: Индекс ускоряет поиск по полю category. Особенно эффективен при высокой селективности (много уникальных значений).
 Пример запроса и EXPLAIN:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE category = 'Electronics';

Index Scan using idx_products_category on products  (cost=0.29..4.31 rows=1 width=84) (actual time=0.048..0.048 rows=0 loops=1)
  Index Cond: ((category)::text = 'Electronics'::text)
  Buffers: shared read=2
Planning:
  Buffers: shared hit=60 read=1 dirtied=3
Planning Time: 1.057 ms
Execution Time: 0.070 ms
```
 Используется Index Scan — индекс работает.


####  2. Индекс для полнотекстового поиска
Цель: Ускорить поиск по описанию с использованием полнотекстового поиска.
``` 
CREATE INDEX idx_products_description_gin 
ON products USING GIN(to_tsvector('english', description));
```
Комментарий: GIN-индекс оптимален для полнотекстового поиска. Функция to_tsvector нормализует текст (стемминг, удаление стоп-слов).

 Пример запроса и EXPLAIN:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products 
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'wireless');

Bitmap Heap Scan on products  (cost=68.89..1206.36 rows=500 width=84) (actual time=1.275..1.276 rows=0 loops=1)
  Recheck Cond: (to_tsvector('english'::regconfig, description) @@ '''wireless'''::tsquery)
  Buffers: shared hit=2
  ->  Bitmap Index Scan on idx_products_description_gin  (cost=0.00..68.76 rows=500 width=0) (actual time=1.199..1.200 rows=0 loops=1)
        Index Cond: (to_tsvector('english'::regconfig, description) @@ '''wireless'''::tsquery)
        Buffers: shared hit=2
Planning:
  Buffers: shared hit=23 read=1 dirtied=2
Planning Time: 2.767 ms
Execution Time: 1.512 ms
```
 Используется Bitmap Index Scan по GIN-индексу.

####  3. Частичный индекс (на часть таблицы)
Цель: Ускорить поиск дешёвых товаров (price < 100), игнорируя дорогие.
```
CREATE INDEX idx_products_active_cheap 
ON products (price) 
WHERE price < 100;
```
Комментарий:
Частичный индекс экономит место и ускоряет запросы к подмножеству данных. Условие в WHERE должно совпадать с условием в запросе.

Пример запроса и EXPLAIN:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE price < 50;

Bitmap Heap Scan on products  (cost=43.57..1572.40 rows=1972 width=84) (actual time=0.735..2.171 rows=1967 loops=1)
  Recheck Cond: (price < '50'::numeric)
  Heap Blocks: exact=1060
  Buffers: shared hit=1060 read=7
  ->  Bitmap Index Scan on idx_products_active_cheap  (cost=0.00..43.07 rows=1972 width=0) (actual time=0.343..0.344 rows=1967 loops=1)
        Index Cond: (price < '50'::numeric)
        Buffers: shared read=7
Planning:
  Buffers: shared hit=25 read=1 dirtied=3
Planning Time: 1.059 ms
Execution Time: 2.296 ms
```
Используется частичный индекс.

####  4. Функциональный индекс (на поле с функцией)
Цель: Ускорить поиск по имени без учёта регистра.
```
CREATE INDEX idx_products_name_lower 
ON products (LOWER(name));
```
Комментарий:
Функциональный индекс позволяет эффективно выполнять запросы с функциями в WHERE. Без него LOWER(name) не использовал бы обычный индекс.

Пример запроса и EXPLAIN:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE LOWER(name) = 'laptop 482';

Bitmap Heap Scan on products  (cost=8.17..1021.89 rows=500 width=84) (actual time=0.484..0.485 rows=0 loops=1)
  Recheck Cond: (lower((name)::text) = 'laptop 482'::text)
  Buffers: shared read=2
  ->  Bitmap Index Scan on idx_products_name_lower  (cost=0.00..8.04 rows=500 width=0) (actual time=0.476..0.477 rows=0 loops=1)
        Index Cond: (lower((name)::text) = 'laptop 482'::text)
        Buffers: shared read=2
Planning:
  Buffers: shared hit=18 read=1 dirtied=2
Planning Time: 0.367 ms
Execution Time: 0.505 ms
```
Индекс успешно применяется.

####  5. Составной индекс (на несколько полей)
Цель: Ускорить запросы с фильтрацией по категории и цене, а также с сортировкой по цене.
```
CREATE INDEX idx_products_category_price 
ON products (category, price);
```
Комментарий: Порядок полей важен: первое поле (category) должно использоваться в фильтре. Второе (price) — для фильтрации или сортировки.

Пример запроса и EXPLAIN:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products 
WHERE category = 'Electronics' AND price BETWEEN 100 AND 500
ORDER BY price;

Sort  (cost=4.32..4.33 rows=1 width=84) (actual time=0.018..0.018 rows=0 loops=1)
  Sort Key: price
  Sort Method: quicksort  Memory: 25kB
  Buffers: shared hit=2
  ->  Index Scan using idx_products_category on products  (cost=0.29..4.31 rows=1 width=84) (actual time=0.012..0.012 rows=0 loops=1)
        Index Cond: ((category)::text = 'Electronics'::text)
        Filter: ((price >= '100'::numeric) AND (price <= '500'::numeric))
        Buffers: shared hit=2
Planning:
  Buffers: shared hit=44 read=1 dirtied=2
Planning Time: 1.020 ms
Execution Time: 0.040 ms
```
Используется составной индекс


#### Проблемы и трудности

- Порядок полей в составном индексе
Если поменять местами price и category, запрос с фильтром только по category не сможет использовать индекс эффективно.
- Объём GIN-индексов
Полнотекстовые индексы занимают значительно больше места, чем B-tree.
- Точное совпадение выражения
Функциональный индекс работает только при точном совпадении выражения в запросе. Например, ILIKE не использует индекс LOWER(name).
- Накладные расходы на запись
Каждый индекс замедляет операции INSERT, UPDATE, DELETE. Важно не создавать избыточные индексы.


