
# Лабораторная работа: Аналитика в Greenplum (150 минут)




## Часть 0. Подготовка: схема и генерация текстовых данных (15 минут)

### 0.1. Создание схемы и таблицы отзывов

```sql
CREATE SCHEMA IF NOT EXISTS lab_analytics;
SET search_path TO lab_analytics, public;

CREATE TABLE reviews (
    review_id   bigint,
    product_id  integer,
    rating      integer,
    review_text text,
    review_date date
) DISTRIBUTED BY (review_id);
```

### 0.2. Генерация данных

Сгенерируем 50 000 отзывов с осмысленным текстом, чтобы полнотекстовый поиск и NLP-функции работали на реальных лексических паттернах.

```sql
INSERT INTO reviews
SELECT
    i,
    1 + (i % 1000),
    1 + (i % 5),  -- рейтинг от 1 до 5
    (ARRAY[
        'The product quality is excellent and delivery was fast',
        'Poor build quality, arrived damaged and late',
        'Great value for money, works perfectly as described',
        'Terrible experience, the item stopped working after a week',
        'Average product, nothing special but does the job',
        'Highly recommend, exceeded my expectations',
        'Not worth the price, very disappointed',
        'Good product but shipping took too long',
        'Amazing features and easy to use, very satisfied',
        'Broken on arrival, requested refund immediately'
    ])[1 + (i % 10)],
    date '2023-01-01' + (i % 730)
FROM generate_series(1, 50000) AS i;

ANALYZE reviews;
```

**Проверка:**
```sql
SELECT count(*), min(rating), max(rating) FROM reviews;
```


## Часть 1. Возможности MADlib (30 минут)

MADlib — это открытая библиотека для масштабируемой аналитики внутри базы данных. Она предоставляет **data-parallel** реализации математических, статистических и机器学习-методов для структурированных и неструктурированных данных, работающие непосредственно в движке Greenplum, без передачи данных во внешние инструменты.

### Задание 1.1. Проверка установки MADlib

MADlib устанавливается как расширение через `gppkg` на весь кластер, а затем регистрируется в каждой базе отдельно:

```bash
# На Coordinator, от gpadmin
gppkg -q --list | grep -i madlib
```

```sql
-- Проверка доступности схемы madlib
SELECT proname FROM pg_proc
WHERE pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'madlib')
LIMIT 5;
```


### Задание 1.2. Линейная регрессия: обучение модели

Создадим синтетический датасет для регрессии: предсказание `rating` по длине отзыва и количеству «позитивных» слов.

```sql
DROP TABLE IF EXISTS review_features;
CREATE TABLE review_features AS
SELECT
    review_id,
    rating AS y,
    length(review_text) AS text_len,
    (length(review_text) - length(replace(review_text, 'good', ''))) / 4 AS good_count,
    (length(review_text) - length(replace(review_text, 'bad', ''))) / 3 AS bad_count
FROM reviews
WHERE review_id <= 10000;

ANALYZE review_features;
```

Обучение модели линейной регрессии через `madlib.linregr_train()`. Функция принимает исходную таблицу, имя таблицы для модели, зависимую переменную и массив независимых переменных. **Intercept вычисляется путём включения константы 1 в массив независимых переменных**:

```sql
DROP TABLE IF EXISTS review_lm;
SELECT madlib.linregr_train(
    'review_features',   -- исходная таблица
    'review_lm',         -- таблица модели
    'y',                 -- зависимая переменная
    'ARRAY[1, text_len, good_count, bad_count]'  -- независимые переменные + intercept
);
```

Просмотр модели:
```sql
SELECT * FROM review_lm;
```

Ожидаемый вывод содержит коэффициенты `coef`, `r2`, стандартные ошибки, t-статистики, p-values и количество обработанных строк.

### Задание 1.3. Предсказание и анализ остатков

Используйте сохранённую модель и функцию `madlib.linregr_predict()` для вычисления предсказанных значений и остатков:

```sql
SELECT
    r.review_id,
    r.y AS actual_rating,
    madlib.linregr_predict(
        ARRAY[1, r.text_len, r.good_count, r.bad_count],
        m.coef
    ) AS predicted_rating,
    r.y - madlib.linregr_predict(
        ARRAY[1, r.text_len, r.good_count, r.bad_count],
        m.coef
    ) AS residual
FROM review_features r, review_lm m
WHERE r.review_id <= 20
ORDER BY r.review_id;
```


### Задание 1.4. Дополнительно: другие алгоритмы MADlib (опционально)

Ознакомьтесь с доступными алгоритмами и попробуйте один из них по выбору:

```sql
-- Список функций MADlib для регрессии
SELECT proname FROM pg_proc
WHERE pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'madlib')
  AND proname LIKE '%regr%'
ORDER BY proname;
```

Например, логистическая регрессия для бинарной классификации (rating ≥ 4 vs. rating < 4):

```sql
-- Подготовка бинарной метки
ALTER TABLE review_features ADD COLUMN y_binary int;
UPDATE review_features SET y_binary = CASE WHEN y >= 4 THEN 1 ELSE 0 END;

-- Обучение логистической регрессии
DROP TABLE IF EXISTS review_logreg;
SELECT madlib.logregr_train(
    'review_features',
    'review_logreg',
    'y_binary',
    'ARRAY[1, text_len, good_count, bad_count]'
);
```


## Часть 2. Интерфейс R (PL/R) (30 минут)

PL/R — это процедурный язык, позволяющий писать функции базы данных на языке R. Функции выполняются на сегментах, что даёт возможность реализовывать параллельные алгоритмы в MPP-среде.

### Задание 2.1. Проверка установки R и регистрация PL/R

R должен быть установлен на **всех хостах** кластера (координатор и сегменты). PL/R — **недоверенный язык**, поэтому регистрировать его может только суперпользователь.

```bash
# Проверка R на Coordinator
R --version

# Проверка R на сегментах (пример для двух сегментов)
gpssh -h sdw1 -h sdw2 -e 'R --version'
```

```sql
-- Регистрация PL/R в текущей базе (от суперпользователя)
CREATE EXTENSION IF NOT EXISTS plr;
```

### Задание 2.2. Простая функция PL/R

Пример из документации: функция генерирует массив чисел с нормальным распределением, используя встроенную R-функцию `rnorm()`:

```sql
CREATE OR REPLACE FUNCTION r_norm(n integer, mean float8, std_dev float8)
RETURNS float8[]
AS $$
    x <- rnorm(n, mean, std_dev)
    return(x)
$$ LANGUAGE plr;
```

Использование:
```sql
CREATE TABLE test_norm_var AS
SELECT id, r_norm(10, 0, 1) AS x
FROM generate_series(1, 30) AS id
DISTRIBUTED BY (id);

SELECT * FROM test_norm_var LIMIT 5;
```

### Задание 2.3. Возврат `data.frame` как таблицы

Если PL/R-функция возвращает `data.frame`, необходимо создать композитный тип, соответствующий структуре `data.frame`, и объявить функцию как `RETURNS SETOF` этого типа:

```sql
-- Создание типа, соответствующего data.frame
DROP TYPE IF EXISTS review_stats CASCADE;
CREATE TYPE review_stats AS (
    rating_bucket text,
    avg_length    float8,
    cnt           bigint
);

-- Функция, возвращающая data.frame
CREATE OR REPLACE FUNCTION get_review_stats()
RETURNS SETOF review_stats
AS $$
    ratings <- c("low", "medium", "high")
    avgs <- c(150.5, 200.3, 250.7)
    cnts <- c(1000, 2000, 1500)
    return(data.frame(ratings, avgs, cnts))
$$ LANGUAGE plr;

-- Использование
SELECT * FROM get_review_stats();
```

### Задание 2.4. Практическая задача: расчёт доверительных интервалов

Реализуем PL/R-функцию, которая вычисляет средний рейтинг и 95% доверительный интервал для каждого продукта. Это типовая задача, которую неудобно выражать на чистом SQL.

```sql
DROP TYPE IF EXISTS product_ci CASCADE;
CREATE TYPE product_ci AS (
    product_id  integer,
    mean_rating float8,
    ci_lower    float8,
    ci_upper    float8
);

CREATE OR REPLACE FUNCTION product_rating_ci()
RETURNS SETOF product_ci
AS $$
    # Предполагается, что данные уже агрегированы по продуктам
    # В реальной PL/R-функции можно использовать SPI для запроса к таблице
    means <- c(3.2, 4.1, 2.8)
    sds <- c(1.1, 0.9, 1.3)
    ns <- c(100, 150, 80)
    se <- sds / sqrt(ns)
    ci_lower <- means - 1.96 * se
    ci_upper <- means + 1.96 * se
    product_ids <- c(1, 2, 3)
    return(data.frame(product_ids, means, ci_lower, ci_upper))
$$ LANGUAGE plr;
```



## Часть 3. Анализ текстовых данных: встроенный полнотекстовый поиск (25 минут)

Greenplum наследует от PostgreSQL типы `tsvector` (предобработанный документ) и `tsquery` (обработанный запрос), а также индексы GIN и GiST для ускорения поиска.

### Задание 3.1. Базовый полнотекстовый поиск

```sql
-- Предобработка документа
SELECT
    review_id,
    to_tsvector('english', review_text) AS tsv
FROM reviews
WHERE review_id <= 3;
```

```sql
-- Поиск по лексемам
SELECT review_id, review_text
FROM reviews
WHERE to_tsvector('english', review_text) @@ to_tsquery('english', 'excellent & fast');
```

### Задание 3.2. Индекс GIN на выражении

Создайте GIN-индекс на выражении `to_tsvector()` для ускорения поиска:

```sql
CREATE INDEX idx_reviews_fts ON reviews
USING gin(to_tsvector('english', review_text));

-- Проверка плана
EXPLAIN ANALYZE
SELECT review_id FROM reviews
WHERE to_tsvector('english', review_text) @@ to_tsquery('english', 'poor & quality');
```

### Задание 3.3. Альтернатива: отдельная колонка `tsvector`

Документация отмечает, что при индексе на выражении `to_tsvector` вычисляется заново для проверки совпадения. Альтернатива — хранить `tsvector` в отдельной колонке:

```sql
ALTER TABLE reviews ADD COLUMN tsv tsvector;
UPDATE reviews SET tsv = to_tsvector('english', review_text);

CREATE INDEX idx_reviews_tsv ON reviews USING gin(tsv);

EXPLAIN ANALYZE
SELECT review_id FROM reviews WHERE tsv @@ to_tsquery('english', 'poor & quality');
```


### Задание 3.4. Статистика по лексемам

Функция `ts_stat()` собирает статистику по лексемам, что полезно для настройки поиска и выявления стоп-слов:

```sql
SELECT word, ndoc, nentry
FROM ts_stat('SELECT to_tsvector(''english'', review_text) FROM reviews')
ORDER BY ndoc DESC
LIMIT 10;
```



## Часть 4. Архитектура Solr / GPText (40 минут)

> **Опционально:** если GPText не установлен, выполните раздел 4.1 (теоретический) и напишите в отчёте, что практические задания 4.2–4.4 пропущены из-за отсутствия GPText в стенде.

### 4.1. Архитектура: как GPText связывает Greenplum и Solr

GPText объединяет кластер Greenplum с кластером **Apache SolrCloud**. Управление кластером Solr осуществляет **Apache ZooKeeper**. Ключевая архитектурная особенность: **число шардов Solr соответствует числу primary-сегментов Greenplum** — по умолчанию один Solr-шард на каждый primary-сегмент.

Процесс поиска выглядит так:
1. Пользователь отправляет SQL-запрос с вызовом `gptext.search()`.
2. Coordinator диспетчеризует запрос на сегменты.
3. **Каждый сегмент** выполняет запрос, обращаясь через Solr API к **своему** шарду индекса.
4. Результаты возвращаются на Coordinator.
5. Coordinator агрегирует результаты и возвращает клиенту.

### Задание 4.2. Создание индекса GPText

Типичный workflow создания индекса:
1. Создать и заполнить исходную таблицу в Greenplum.
2. Создать пустой индекс через `gptext.create_index()`.
3. (Опционально) настроить конфигурацию индекса через `gptext-config`.
4. Заполнить индекс через `gptext.index()`.
5. Зафиксировать изменения через `gptext.commit_index()`.

```sql
-- 1. Исходная таблица уже есть (reviews)

-- 2. Создание пустого индекса
SELECT * FROM gptext.create_index('lab_analytics', 'reviews', 'review_id', 'review_text');

-- 3. Заполнение индекса
SELECT * FROM gptext.index(TABLE(SELECT * FROM reviews), 'lab_analytics.public.reviews');

-- 4. Фиксация изменений (до коммита данные не видны для поиска)
SELECT * FROM gptext.commit_index('lab_analytics.public.reviews');
```

### Задание 4.3. Поиск через `gptext.search()`

Синтаксис функции:
```text
gptext.search(<src_table>, <index_name>, <search_query>, <filter_queries>[, <options>])
```

Простой поиск по термину «quality»:
```sql
SELECT *
FROM gptext.search(
    TABLE(SELECT 1 SCATTER BY 1),
    'lab_analytics.public.reviews',
    'quality',
    null,
    'rows=5'
);
```

Поиск с булевым оператором `AND` и возвратом поля `review_text`:
```sql
SELECT
    r.review_id,
    r.review_text,
    q.score
FROM reviews r,
gptext.search(
    TABLE(SELECT 1 SCATTER BY 1),
    'lab_analytics.public.reviews',
    'poor AND quality',
    null,
    'rows=10&fl=review_text'
) q
WHERE q.id::int8 = r.review_id
ORDER BY q.score DESC;
```

> **Обратите внимание:** `gptext.search()` по умолчанию возвращает только колонки `id`, `score`, `hs` и `rf`. Чтобы получить дополнительные поля, используйте опцию `fl` (field list) или функцию `gptext.gptext_retrieve_field()` для извлечения полей из JSON.

### Задание 4.4. Фасетный поиск

GPText предоставляет функции фасетного поиска, разбивающие результаты по категориям:

```sql
-- Пример фасетного поиска (синтаксис может отличаться в зависимости от версии GPText)
SELECT * FROM gptext.faceted_query_search(
    TABLE(SELECT 1 SCATTER BY 1),
    'lab_analytics.public.reviews',
    '*:*',                          -- все документы
    'rating'                        -- фасет по полю rating
);
```
