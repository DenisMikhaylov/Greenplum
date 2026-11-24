

# Лабораторная работа: Хранение данных в Greenplum

**Продолжительность:** 120–150 минут
**Предварительные требования:** развёрнутый кластер Greenplum (минимум 2 сегментных хоста, как в лабораторной по развёртыванию), доступ `psql` от пользователя `gpadmin`.

---

## Цель работы

На сквозном примере освоить полный цикл проектирования физического хранения в Greenplum: от выбора политики распределения и модели хранения до партиционирования, сжатия и индексирования, с оценкой влияния каждого решения на планы выполнения запросов и равномерность распределения данных.

---

## Исходные данные и предметная область

Интернет-магазин имеет три основные сущности:

| Сущность | Тип | Ожидаемый объём | Паттерн нагрузки |
|---|---|---|---|
| `customers` | Справочник покупателей | ~50 000 строк | Частые точечные `SELECT`, редкие `UPDATE` |
| `products` | Справочник товаров | ~5 000 строк | Частые `SELECT`, редкие `UPDATE` |
| `orders` | Факт-таблица заказов | ~2 000 000 строк | Batch-загрузка, аналитические агрегации, `UPDATE` статуса только для «свежих» заказов |

**Схема связей:**
- `orders.customer_id` → `customers.customer_id`
- `orders.product_id` → `products.product_id`

---

## Часть 0. Подготовка: создание схемы и генерация данных

Все команды выполняются в `psql` от пользователя `gpadmin` на хосте `mdw`.

### 0.1. Создание схемы

```sql
CREATE SCHEMA IF NOT EXISTS lab_storage;
SET search_path TO lab_storage, public;
```

### 0.2. Справочник покупателей

```sql
CREATE TABLE customers (
    customer_id   integer,
    customer_name text,
    region        text,
    signup_date   date
) DISTRIBUTED REPLICATED;
```


### 0.3. Справочник товаров

```sql
CREATE TABLE products (
    product_id   integer,
    product_name text,
    category     text,
    base_price   numeric(10,2)
) DISTRIBUTED REPLICATED;
```

### 0.4. Факт-таблица заказов (исходная, heap)

```sql
CREATE TABLE orders (
    order_id    bigint,
    customer_id integer,
    product_id  integer,
    order_date  date,
    quantity    integer,
    amount      numeric(12,2),
    status      text
) DISTRIBUTED BY (order_id);
```

**Почему `DISTRIBUTED BY (order_id)`:** `order_id` уникален для каждой строки, что обеспечивает равномерное hash-распределение. При этом JOIN с `customers` и `products` не будет co-located (ключи соединения — `customer_id` и `product_id`), но эти таблицы реплицированы, поэтому broadcast не потребуется.

### 0.5. Генерация данных

```sql
-- Покупатели
INSERT INTO customers
SELECT
    i,
    'Customer_' || i,
    (ARRAY['North','South','East','West','Central'])[1 + (i % 5)],
    date '2020-01-01' + (i % 1000)
FROM generate_series(1, 50000) AS i;

-- Товары
INSERT INTO products
SELECT
    i,
    'Product_' || i,
    (ARRAY['Electronics','Clothing','Books','Home','Sports'])[1 + (i % 5)],
    (random() * 1000)::numeric(10,2)
FROM generate_series(1, 5000) AS i;

-- Заказы
INSERT INTO orders
SELECT
    i,
    1 + (random() * 49999)::int,
    1 + (random() * 4999)::int,
    date '2022-01-01' + (i % 730),
    1 + (random() * 10)::int,
    (random() * 5000)::numeric(12,2),
    (ARRAY['new','paid','shipped','delivered','cancelled'])[1 + (i % 5)]
FROM generate_series(1, 2000000) AS i;

ANALYZE customers;
ANALYZE products;
ANALYZE orders;
```

---

## Часть 1. Определение объектов базы данных и распределение данных

### Задание 1.1. Исследование метаданных

Посмотрите, как Greenplum хранит информацию о распределении и хранении таблиц.

```sql
\d+ customers
\d+ orders
```

### Задание 1.2. Диагностика перекоса

Проверьте равномерность распределения строк в `orders` по сегментам.

**Метод 1 (прямой подсчёт):**
```sql
SELECT gp_segment_id, count(*)
FROM orders
GROUP BY gp_segment_id
ORDER BY gp_segment_id;
```

**Метод 2 (через `gp_toolkit`):**
```sql
SELECT skccoeff, skcrelname
FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'orders';

SELECT siffraction, sifrelname
FROM gp_toolkit.gp_skew_idle_fractions
WHERE sifrelname = 'orders';
```


### Задание 1.3. Сравнение политик распределения

Создайте три копии таблицы `orders` с разными политиками и сравните планы JOIN с `customers`.

```sql
-- Hash по order_id (уже есть)
-- Hash по customer_id
CREATE TABLE orders_by_customer (LIKE orders) DISTRIBUTED BY (customer_id);
INSERT INTO orders_by_customer SELECT * FROM orders;

-- Random
CREATE TABLE orders_random (LIKE orders) DISTRIBUTED RANDOMLY;
INSERT INTO orders_random SELECT * FROM orders;

ANALYZE orders_by_customer;
ANALYZE orders_random;
```

Сравните планы:
```sql
EXPLAIN SELECT c.region, count(*)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.region;

EXPLAIN SELECT c.region, count(*)
FROM orders_by_customer o JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.region;

EXPLAIN SELECT c.region, count(*)
FROM orders_random o JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.region;
```

---

## Часть 2. Полиморфное хранилище и партиционирование

### Задание 2.1. Создание партиционированной таблицы с разными storage-опциями

Создайте таблицу `orders_part`, где «свежие» партиции (2024) используют heap (потому что статус заказов ещё обновляется), а исторические (2022–2023) — AO column со сжатием (только чтение).

**Для Greenplum 6 (классический синтаксис):**
```sql
CREATE TABLE orders_part (
    order_id    bigint,
    customer_id integer,
    product_id  integer,
    order_date  date,
    quantity    integer,
    amount      numeric(12,2),
    status      text
) DISTRIBUTED BY (order_id)
PARTITION BY RANGE (order_date)
(
    START (date '2022-01-01') INCLUSIVE
    END (date '2024-01-01') EXCLUSIVE
    EVERY (INTERVAL '1 year')
    WITH (appendonly = true, orientation = column,
          compresstype = zstd, compresslevel = 5),

    START (date '2024-01-01') INCLUSIVE
    END (date '2025-01-01') EXCLUSIVE
    WITH (appendonly = false)
);
```

**Для Greenplum 7 (декларативный синтаксис):**
```sql
-- Родительская таблица
CREATE TABLE orders_part (
    order_id    bigint,
    customer_id integer,
    product_id  integer,
    order_date  date,
    quantity    integer,
    amount      numeric(12,2),
    status      text
) DISTRIBUTED BY (order_id)
PARTITION BY RANGE (order_date);

-- Исторические партиции: AOCO + ZSTD
CREATE TABLE orders_part_2022 PARTITION OF orders_part
FOR VALUES FROM ('2022-01-01') TO ('2023-01-01')
WITH (appendonly = true, orientation = column,
      compresstype = zstd, compresslevel = 5);

CREATE TABLE orders_part_2023 PARTITION OF orders_part
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01')
WITH (appendonly = true, orientation = column,
      compresstype = zstd, compresslevel = 5);

-- Свежая партиция: heap
CREATE TABLE orders_part_2024 PARTITION OF orders_part
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
WITH (appendonly = false);
```

Заполните партиционированную таблицу:
```sql
INSERT INTO orders_part SELECT * FROM orders;
ANALYZE orders_part;
```

**Вопросы:**
1. Можно ли в одной таблице `orders_part` одновременно иметь heap- и AOCO-партиции? Что покажет `\d+ orders_part_2024` и `\d+ orders_part_2022`?
2. Почему для партиции 2024 выбран heap, а для 2022–2023 — AOCO? Свяжите с паттерном нагрузки из условия.

### Задание 2.2. Проверка partition pruning

```sql
EXPLAIN SELECT count(*) FROM orders_part WHERE order_date = '2023-06-15';
EXPLAIN SELECT count(*) FROM orders_part WHERE order_date = '2024-06-15';
```


---

## Часть 3. Колоночное хранилище и сжатие

### Задание 3.1. Сравнение row vs column для аналитического запроса

Создайте две таблицы с одинаковыми данными: heap row и AO column.

```sql
CREATE TABLE orders_row (LIKE orders) DISTRIBUTED BY (order_id);
INSERT INTO orders_row SELECT * FROM orders;

CREATE TABLE orders_col (
    LIKE orders
) WITH (appendonly = true, orientation = column)
DISTRIBUTED BY (order_id);
INSERT INTO orders_col SELECT * FROM orders;

ANALYZE orders_row;
ANALYZE orders_col;
```

Выполните запрос, который читает только 2 колонки из 7:

```sql
-- Замер времени
\timing on

SELECT product_id, sum(amount)
FROM orders_row
GROUP BY product_id;

SELECT product_id, sum(amount)
FROM orders_col
GROUP BY product_id;

\timing off
```


### Задание 3.2. Сжатие и его эффективность

```sql
-- Таблица с сжатием ZSTD
CREATE TABLE orders_col_zstd (
    LIKE orders
) WITH (appendonly = true, orientation = column,
        compresstype = zstd, compresslevel = 5)
DISTRIBUTED BY (order_id);
INSERT INTO orders_col_zstd SELECT * FROM orders;
ANALYZE orders_col_zstd;

-- Проверка коэффициента сжатия
SELECT get_ao_compression_ratio('orders_col_zstd');
```

**Сравнение размеров на диске:**
```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relname IN ('orders_row', 'orders_col', 'orders_col_zstd')
ORDER BY relname;
```

**Column-level сжатие:**
```sql
CREATE TABLE orders_col_mixed (
    order_id    bigint,
    customer_id integer,
    product_id  integer,
    order_date  date,
    quantity    integer ENCODING (compresstype = zstd, compresslevel = 9),
    amount      numeric(12,2) ENCODING (compresstype = zlib, compresslevel = 6),
    status      text ENCODING (compresstype = rle_type)
) WITH (appendonly = true, orientation = column)
DISTRIBUTED BY (order_id);
```


---

## Часть 4. Append-Optimized таблицы: ограничения и поведение

### Задание 4.1. Проверка ограничений AO

Попробуйте выполнить `UPDATE` и `DELETE` на AO-таблице в транзакции с уровнем изоляции `SERIALIZABLE`.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE orders_col SET amount = amount + 1 WHERE order_id = 1;
-- Ожидается ошибка
ROLLBACK;
```

```sql
DELETE FROM orders_col WHERE order_id = 1;
-- Проверьте, выполнилась ли команда и что произошло с физическим файлом
SELECT count(*) FROM orders_col WHERE order_id = 1;
```


### Задание 4.2. Сравнение поведения heap и AO при удалении

```sql
-- Heap
DELETE FROM orders_row WHERE order_id <= 1000;
VACUUM orders_row;

-- AO
DELETE FROM orders_col WHERE order_id <= 1000;
VACUUM orders_col;
```

Сравните размеры после `VACUUM`:
```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(oid))
FROM pg_class
WHERE relname IN ('orders_row', 'orders_col');
```


---

## Часть 5. Индексирование

### Задание 5.1. Индекс на heap vs AO

```sql
-- Heap: обычный B-tree
CREATE INDEX idx_orders_row_customer ON orders_row (customer_id);
ANALYZE orders_row;

EXPLAIN SELECT * FROM orders_row WHERE customer_id = 100;
```

```sql
-- AO column: попытка B-tree
CREATE INDEX idx_orders_col_customer ON orders_col (customer_id);
-- Проверьте, создался ли индекс и какой тип сканирования используется
EXPLAIN SELECT * FROM orders_col WHERE customer_id = 100;
```


### Задание 5.2. Bitmap-индекс на AO и его пределы

```sql
-- Bitmap-индекс на AO-таблице
CREATE INDEX idx_orders_col_status_bitmap ON orders_col USING bitmap (status);
ANALYZE orders_col;

EXPLAIN SELECT * FROM orders_col WHERE status = 'delivered';
```

```sql
-- Проверка при отключённом bitmap-скане
SET enable_bitmapscan = off;
EXPLAIN SELECT * FROM orders_col WHERE status = 'delivered';
RESET enable_bitmapscan;
```


---
