# Лабораторная работа: Запросы в Greenplum 

**Сценарий:** аналитическая команда исследует поведение запросов в MPP-среде: сравнивает работу GPORCA и планировщика PostgreSQL, учится управлять материализацией CTE, работает с полуструктурированными данными (JSON, XML), настраивает полнотекстовый поиск, профилирует запросы и выносит тяжёлые вычисления в изолированные контейнеры.


## Хронометраж

| Часть | Тема | Время |
|---|---|---|
| 0 | Подготовка: схема и генерация данных | 20 мин |
| 1 | Обработка запросов и GPORCA vs PostgreSQL-планировщик | 15 мин |
| 2 | Общие табличные выражения (CTE) | 15 мин |
| 3 | Функции и операторы | 10 мин |
| 4 | Работа с JSON | 20 мин |
| 5 | Работа с XML | 15 мин |
| 6 | Полнотекстовый поиск | 15 мин |
| 7 | Greenplum MapReduce | 10 мин |
| 8 | Профилирование и производительность | 15 мин |
| 9 | Расширение для языков программирования (PL/Container) | 10 мин |
| 10 | Итоговые выводы | 5 мин |


## Часть 0. Подготовка: схема и генерация данных (20 минут)

### 0.1. Создание схемы и таблиц

```sql
CREATE SCHEMA IF NOT EXISTS lab_queries;
SET search_path TO lab_queries, public;

-- Партиционированная таблица заказов (для сравнения оптимизаторов)
CREATE TABLE orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text,
    payload     jsonb
) DISTRIBUTED BY (order_id)
PARTITION BY RANGE (order_date)
(
    START (date '2023-01-01') INCLUSIVE
    END (date '2025-01-01') EXCLUSIVE
    EVERY (INTERVAL '3 months')
);

-- Справочник покупателей (реплицирован)
CREATE TABLE customers (
    customer_id integer,
    name        text,
    region      text
) DISTRIBUTED REPLICATED;
```

### 0.2. Генерация данных

```sql
-- Покупатели
INSERT INTO customers
SELECT i, 'Customer_' || i, (ARRAY['North','South','East','West'])[1 + (i % 4)]
FROM generate_series(1, 10000) AS i;

-- Заказы: 500 000 строк, ~2 года
INSERT INTO orders
SELECT
    i,
    1 + (random() * 9999)::int,
    date '2023-01-01' + (i % 730),
    (random() * 5000)::numeric(12,2),
    (ARRAY['new','paid','shipped','delivered','cancelled'])[1 + (i % 5)],
    jsonb_build_object(
        'channel', (ARRAY['web','mobile','store'])[1 + (i % 3)],
        'items', jsonb_build_array(
            jsonb_build_object('sku', 'SKU-' || (i % 1000), 'qty', 1 + (i % 5)),
            jsonb_build_object('sku', 'SKU-' || ((i + 7) % 1000), 'qty', 1 + (i % 3))
        )
    )
FROM generate_series(1, 500000) AS i;

ANALYZE orders;
ANALYZE customers;
```


## Часть 1. Обработка запросов: GPORCA vs PostgreSQL-планировщик (15 минут)

### Задание 1.1. Определение использованного оптимизатора

Выполните `EXPLAIN` и посмотрите на завершающие строки плана. Если GPORCA сгенерировал план, в конце будет указано `Settings: optimizer=on` и `Optimizer status: PQO version ...`. Если произошёл откат на планировщик PostgreSQL, будет указано `Optimizer status: legacy query optimizer`.

```sql
EXPLAIN
SELECT c.region, count(*), sum(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date BETWEEN '2024-01-01' AND '2024-06-30'
GROUP BY c.region;
```

### Задание 1.2. Принудительное переключение на планировщик PostgreSQL

```sql
SET optimizer = off;
EXPLAIN
SELECT c.region, count(*), sum(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date BETWEEN '2024-01-01' AND '2024-06-30'
GROUP BY c.region;
RESET optimizer;
```


### Задание 1.3. Проверка отката GPORCA на неподдерживаемой конструкции

Попробуйте выполнить запрос с конструкцией, которую GPORCA может не поддерживать (например, `UPDATE` с подзапросом в `SET`, ссылающимся на ту же таблицу, или запрос к партиционированной таблице, у которой листовая партиция заменена на внешнюю таблицу). Если откат произошёл, в лог-файле появится сообщение `Planner produced plan`.

```sql
-- Пример: обновление с подзапросом к той же таблице
EXPLAIN
UPDATE orders o
SET amount = amount * 1.1
WHERE order_id IN (SELECT order_id FROM orders WHERE status = 'new');
```


## Часть 2. Общие табличные выражения (CTE) (15 минут)

### Задание 2.1. CTE с материализацией по умолчанию

В Greenplum (как и в PostgreSQL до версии 12) CTE по умолчанию **материализуется**: результат вычисляется один раз и сохраняется, а внешние предикаты **не проталкиваются** внутрь CTE.

```sql
EXPLAIN ANALYZE
WITH recent_orders AS (
    SELECT customer_id, amount, order_date
    FROM orders
    WHERE order_date >= '2024-10-01'
)
SELECT * FROM recent_orders
WHERE amount > 4000;
```

### Задание 2.2. `NOT MATERIALIZED` и инлайнинг

Если версия Greenplum поддерживает модификаторы CTE (Greenplum 7 / PostgreSQL 12+), выполните:

```sql
EXPLAIN ANALYZE
WITH recent_orders AS NOT MATERIALIZED (
    SELECT customer_id, amount, order_date
    FROM orders
    WHERE order_date >= '2024-10-01'
)
SELECT * FROM recent_orders
WHERE amount > 4000;
```


### Задание 2.3. Рекурсивный CTE

Greenplum по умолчанию включает `RECURSIVE` для `WITH`; рекурсию можно отключить параметром `gp_recursive = false`.

```sql
WITH RECURSIVE nums(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10
)
SELECT sum(n) FROM nums;
```



## Часть 3. Функции и операторы (10 минут)

### Задание 3.1. Встроенные функции для работы с JSON и строками

```sql
SELECT
    order_id,
    payload ->> 'channel' AS channel,
    jsonb_array_length(payload -> 'items') AS item_count,
    upper(status) AS status_upper,
    amount * 1.2 AS amount_with_tax
FROM orders
WHERE order_id <= 5;
```

### Задание 3.2. Пользовательская функция на SQL

```sql
CREATE OR REPLACE FUNCTION order_category(amount numeric)
RETURNS text
AS $$
    SELECT CASE
        WHEN $1 < 100 THEN 'small'
        WHEN $1 < 1000 THEN 'medium'
        ELSE 'large'
    END;
$$ LANGUAGE sql IMMUTABLE;

SELECT order_id, amount, order_category(amount)
FROM orders
WHERE order_id <= 10;
```


## Часть 4. Работа с JSON (20 минут)

### Задание 4.1. Операторы доступа и извлечение данных

```sql
-- Извлечение поля верхнего уровня
SELECT order_id, payload ->> 'channel' AS channel
FROM orders
WHERE order_id <= 5;

-- Доступ к вложенному массиву
SELECT
    order_id,
    jsonb_array_elements(payload -> 'items') ->> 'sku' AS sku
FROM orders
WHERE order_id <= 3;
```

### Задание 4.2. Фильтрация по JSON и GIN-индекс

Без индекса фильтрация по `payload ->> 'channel'` выполняет полное сканирование. Создайте GIN-индекс на `jsonb`-колонку. Документация Greenplum указывает, что GIN-индексы на `jsonb` поддерживают операторы `@>` (containment), `?` (existence) и `@?` (jsonpath).

```sql
-- Запрос без индекса
EXPLAIN ANALYZE
SELECT count(*) FROM orders
WHERE payload @> '{"channel": "mobile"}';

-- Создание GIN-индекса
CREATE INDEX idx_orders_payload_gin ON orders USING gin (payload);

ANALYZE orders;

-- Повторный запрос
EXPLAIN ANALYZE
SELECT count(*) FROM orders
WHERE payload @> '{"channel": "mobile"}';
```

### Задание 4.3. `jsonb_path_ops` vs `jsonb_ops`

Создайте альтернативный GIN-индекс с операторным классом `jsonb_path_ops`:

```sql
CREATE INDEX idx_orders_payload_pathops ON orders USING gin (payload jsonb_path_ops);

EXPLAIN ANALYZE
SELECT count(*) FROM orders
WHERE payload @> '{"channel": "mobile"}';
```


## Часть 5. Работа с XML (15 минут)

### Задание 5.1. Создание таблицы с XML-документами

```sql
CREATE TABLE xml_docs (
    doc_id  bigint,
    content xml
) DISTRIBUTED BY (doc_id);

INSERT INTO xml_docs VALUES
(1, '<order><customer>Alice</customer><amount>100</amount><items><item>Book</item></items></order>'),
(2, '<order><customer>Bob</customer><amount>250</amount><items><item>Pen</item><item>Paper</item></items></order>'),
(3, '<order><customer>Carol</customer><amount>75</amount><items><item>Notebook</item></items></order>');
```

### Задание 5.2. `xpath` и `xpath_exists`

Функция `xpath` вычисляет XPath 1.0 и возвращает массив XML-значений; `xpath_exists` возвращает булево значение.

```sql
-- Извлечение значения
SELECT
    doc_id,
    (xpath('/order/customer/text()', content))[1]::text AS customer,
    (xpath('/order/amount/text()', content))[1]::text::numeric AS amount
FROM xml_docs;

-- Проверка существования узла
SELECT doc_id
FROM xml_docs
WHERE xpath_exists('/order/items/item[2]', content);
```

### Задание 5.3. Ограничения индексирования XML

Документация Greenplum прямо указывает: тип `xml` **не имеет операторов сравнения**, поэтому создать индекс непосредственно на колонке `xml` **невозможно**.

```sql
-- Эта команда завершится ошибкой
CREATE INDEX idx_xml_content ON xml_docs (content);
```

**Обходной путь:** создайте индекс на выражении, приведённом к тексту:

```sql
CREATE INDEX idx_xml_customer ON xml_docs ((xpath('/order/customer/text()', content)::text));

EXPLAIN SELECT * FROM xml_docs
WHERE xpath('/order/customer/text()', content)::text = '{Alice}';
```



## Часть 6. Полнотекстовый поиск (15 минут)

### Задание 6.1. `tsvector` и `tsquery`

Greenplum предоставляет типы `tsvector` (предобработанный документ) и `tsquery` (обработанный запрос).

```sql
CREATE TABLE documents (
    doc_id bigint,
    title  text,
    body   text
) DISTRIBUTED BY (doc_id);

INSERT INTO documents VALUES
(1, 'Greenplum MPP', 'Greenplum is a massively parallel processing database for analytics'),
(2, 'PostgreSQL', 'PostgreSQL is an open source relational database'),
(3, 'Greenplum Architecture', 'Greenplum uses a shared-nothing architecture with segments');

-- Предобработка документа
SELECT doc_id, to_tsvector('english', title || ' ' || body) AS tsv
FROM documents;
```

### Задание 6.2. GIN-индекс для полнотекстового поиска

Документация рекомендует создавать GIN-индекс на `tsvector` для ускорения поиска.

```sql
-- Индекс на выражении (простой в настройке)
CREATE INDEX idx_docs_fts_expr ON documents
USING gin (to_tsvector('english', title || ' ' || body));

EXPLAIN ANALYZE
SELECT doc_id, title
FROM documents
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'greenplum & database');
```

**Альтернативный подход** — отдельная колонка `tsvector`:

```sql
ALTER TABLE documents ADD COLUMN tsv tsvector;
UPDATE documents SET tsv = to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''));

CREATE INDEX idx_docs_fts_col ON documents USING gin (tsv);

EXPLAIN ANALYZE
SELECT doc_id, title FROM documents WHERE tsv @@ to_tsquery('english', 'greenplum & architecture');
```


## Часть 7. Greenplum MapReduce (10 минут)


### Задание 7.1. Создание YAML-файла MapReduce

```yaml
# /home/gpadmin/wordcount.yml
VERSION: 1.0.0.1
DATABASE: postgres
USER: gpadmin
HOST: mdw
PORT: 5432
DEFINE:
  - INPUT:
      NAME: docs
      TABLE: lab_queries.documents
      COLUMNS:
        - title
        - body
  - MAP:
      NAME: word_map
      FUNCTION: |
        import sys
        for line in sys.stdin:
            for word in line.strip().split():
                print(f"{word.lower()}\t1")
  - REDUCE:
      NAME: word_reduce
      FUNCTION: |
        import sys
        current = None
        count = 0
        for line in sys.stdin:
            word, c = line.strip().split('\t')
            c = int(c)
            if word == current:
                count += c
            else:
                if current:
                    print(f"{current}\t{count}")
                current, count = word, c
        if current:
            print(f"{current}\t{count}")
EXECUTE:
  - RUN:
      SOURCE: docs
      MAP: word_map
      REDUCE: word_reduce
      TARGET: word_counts
```

```bash
gpmapreduce -f /home/gpadmin/wordcount.yml
```



## Часть 8. Профилирование и производительность (15 минут)

### Задание 8.1. `EXPLAIN` vs `EXPLAIN ANALYZE`

```sql
-- Только план и оценки
EXPLAIN
SELECT c.region, count(*), avg(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.region;

-- Фактическое выполнение
EXPLAIN ANALYZE
SELECT c.region, count(*), avg(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.region;
```

### Задание 8.2. Поиск motion-узлов

Планы Greenplum читаются **снизу вверх**; верхние узлы — это motion-операции (`Redistribute`, `Broadcast`, `Gather`), которые перемещают строки между сегментами.

```sql
EXPLAIN ANALYZE
SELECT o.order_id, c.name
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date = '2024-06-15';
```


### Задание 8.3. Проверка перекоса данных

```sql
SELECT gp_segment_id, count(*)
FROM orders
GROUP BY gp_segment_id
ORDER BY gp_segment_id;

SELECT * FROM gp_toolkit.gp_skew_coefficients
WHERE skcrelname = 'lab_queries.orders';
```



## Часть 9. Расширение для языков программирования: PL/Container (10 минут)

> **Опционально:** если расширение `plcontainer` не установлено, выполните `CREATE EXTENSION plcontainer;` от суперпользователя. PL/Container требует Docker на сегментных хостах.

### Задание 9.1. Регистрация и простая UDF

```sql
CREATE EXTENSION IF NOT EXISTS plcontainer;

-- UDF на PL/Python, выполняемая в контейнере
CREATE OR REPLACE FUNCTION py_log10(x double precision)
RETURNS double precision
AS $$
    import math
    return math.log10(x)
$$ LANGUAGE plpython3u;
```

```sql
SELECT py_log10(1000), py_log10(100);
```


### Задание 9.2. Сравнение производительности: встроенная функция vs PL/Python

```sql
-- Встроенная функция
\timing on
SELECT sum(log(amount)) FROM orders;
\timing off

-- PL/Python UDF (если доступна)
\timing on
SELECT sum(py_log10(amount)) FROM orders WHERE amount > 0;
\timing off
```

