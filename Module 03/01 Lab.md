# Лабораторная работа: Загрузка данных в Greenplum (120 минут)



## Часть 0. Подготовка: схема и тестовые данные (15 минут)

### 0.1. Создание схемы и целевой таблицы

```sql
CREATE SCHEMA IF NOT EXISTS lab_load;
SET search_path TO lab_load, public;

CREATE TABLE orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
) DISTRIBUTED BY (order_id);
```

### 0.2. Генерация CSV-файла на Coordinator

Создайте файл с 100 000 строк на хосте `mdw` (не через SQL, а средствами ОС, чтобы получить «внешний» файл):

```bash
cd /home/gpadmin
python3 - <<'PY'
import random, datetime
with open('/home/gpadmin/orders_100k.csv', 'w') as f:
    for i in range(1, 100001):
        cid = random.randint(1, 5000)
        d = datetime.date(2024, 1, 1) + datetime.timedelta(days=random.randint(0, 365))
        amt = round(random.uniform(10, 5000), 2)
        st = random.choice(['new','paid','shipped','delivered','cancelled'])
        f.write(f"{i},{cid},{d},{amt},{st}\n")
PY

wc -l /home/gpadmin/orders_100k.csv
head -3 /home/gpadmin/orders_100k.csv
```

### 0.3. Генерация файла с ошибками (для тестов изоляции ошибок)

```bash
python3 - <<'PY'
with open('/home/gpadmin/orders_bad.csv', 'w') as f:
    f.write("1,100,2024-01-15,250.00,new\n")
    f.write("2,NOT_A_NUMBER,2024-01-16,130.50,paid\n")   # ошибка типа в customer_id
    f.write("3,102,2024-13-45,99.99,shipped\n")           # невалидная дата
    f.write("4,103,2024-01-17,500.00,delivered\n")
    f.write("5,104,2024-01-18,250.00\n")                   # недостаточно колонок
    f.write("6,105,2024-01-19,300.00,new,EXTRA\n")         # лишняя колонка
PY
```


## Часть 1. Команда `INSERT` 

### Задание 1.1. Одиночный и многострочный `INSERT`

```sql
-- Одиночная вставка
INSERT INTO orders VALUES (1000001, 5001, '2024-06-01', 250.00, 'new');

-- Многострочная вставка
INSERT INTO orders (order_id, customer_id, order_date, amount, status)
VALUES
    (1000002, 5002, '2024-06-02', 130.50, 'paid'),
    (1000003, 5003, '2024-06-03', 99.99, 'shipped'),
    (1000004, 5004, '2024-06-04', 500.00, 'delivered');
```

**Замер времени** (включите тайминг в `psql`):
```sql
\timing on
-- вставьте ещё 10 одиночных строк по одной и замерьте суммарное время
\timing off
```

### Задание 1.2. `INSERT` через `SELECT` из `generate_series`

```sql
\timing on
INSERT INTO orders
SELECT
    2000000 + i,
    1 + (random() * 4999)::int,
    date '2024-01-01' + (i % 365),
    (random() * 5000)::numeric(12,2),
    (ARRAY['new','paid','shipped','delivered','cancelled'])[1 + (i % 5)]
FROM generate_series(1, 10000) AS i;
\timing off
```

### Задание 1.3. Влияние sequence-кэша (опционально, при наличии `SERIAL`)

Если бы `order_id` был `BIGSERIAL`, параллельные вставки могли бы конкурировать за последовательность. Проверьте рекомендацию из документации:

```sql
-- Создайте тестовую таблицу с BIGSERIAL
CREATE TABLE seq_test (id bigserial, val text) DISTRIBUTED RANDOMLY;
ALTER SEQUENCE seq_test_id_seq CACHE 100;
```

**Вопрос:** как увеличение `CACHE` снижает конкуренцию при параллельных вставках?


## Часть 2. Команда `COPY` 

### Задание 2.1. Базовый `COPY` из файла на Coordinator

```sql
TRUNCATE orders;

\timing on
COPY orders FROM '/home/gpadmin/orders_100k.csv'
WITH (FORMAT CSV);
\timing off

SELECT count(*) FROM orders;
```


### Задание 2.2. `COPY` с изоляцией ошибок (`SEGMENT REJECT LIMIT`)

Сначала — **без** изоляции:
```sql
TRUNCATE orders;
COPY orders FROM '/home/gpadmin/orders_bad.csv' WITH (FORMAT CSV);
-- Ожидается ошибка на первой некорректной строке
SELECT count(*) FROM orders;  -- данные не загружены
```

Теперь — **с** изоляцией:
```sql
COPY orders FROM '/home/gpadmin/orders_bad.csv'
WITH (FORMAT CSV)
LOG ERRORS SEGMENT REJECT LIMIT 10 ROWS;

SELECT count(*) FROM orders;  -- загружены только корректные строки (2 из 6)
```

### Задание 2.3. `COPY ... ON SEGMENT` (если данные уже на сегментах)

Если у вас есть доступ к сегментным хостам, скопируйте небольшой файл на `sdw1` и `sdw2` и попробуйте:

```bash
# На mdw
scp /home/gpadmin/orders_100k.csv gpadmin@sdw1:/home/gpadmin/
scp /home/gpadmin/orders_100k.csv gpadmin@sdw2:/home/gpadmin/
```

```sql
-- На mdw, в psql
COPY orders FROM '/home/gpadmin/orders_100k.csv' ON SEGMENT;
```



### Задание 2.4. Практика: удаление и пересоздание индексов

```sql
-- Создайте индекс
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Загрузите ещё 100k строк через COPY и замерьте время
\timing on
COPY orders FROM '/home/gpadmin/orders_100k.csv' WITH (FORMAT CSV);
\timing off
```

```sql
-- Теперь: удалить индекс, загрузить, пересоздать
TRUNCATE orders;
DROP INDEX idx_orders_customer;

\timing on
COPY orders FROM '/home/gpadmin/orders_100k.csv' WITH (FORMAT CSV);
CREATE INDEX idx_orders_customer ON orders (customer_id);
\timing off
```


## Часть 3. Утилита `gpfdist` 

### Задание 3.1. Запуск `gpfdist`

На **отдельном хосте** (можно использовать `mdw`, но лучше — отдельный ETL-хост, если есть):

```bash
mkdir -p /home/gpadmin/gpfdist_data
cp /home/gpadmin/orders_100k.csv /home/gpadmin/gpfdist_data/

# Запуск gpfdist в фоне
gpfdist -d /home/gpadmin/gpfdist_data -p 8081 -l /home/gpadmin/gpfdist.log &

# Проверка
ps aux | grep gpfdist
curl http://localhost:8081/orders_100k.csv | head -3
```

### Задание 3.2. Создание внешней таблицы и загрузка

```sql
TRUNCATE orders;

CREATE EXTERNAL TABLE ext_orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
)
LOCATION ('gpfdist://mdw:8081/orders_100k.csv')
FORMAT 'CSV';

\timing on
INSERT INTO orders SELECT * FROM ext_orders;
\timing off

SELECT count(*) FROM orders;
```

> **Замените `mdw`** на фактическое имя хоста, где запущен `gpfdist`. `LOCATION` должен быть доступен **со всех сегментных хостов**.

**Вопрос:** почему при загрузке через `gpfdist` Coordinator **не является узким местом**? Кто именно читает данные из `gpfdist` и распределяет строки по сегментам?

### Задание 3.3. Несколько экземпляров `gpfdist` для параллелизма

Запустите **второй** экземпляр `gpfdist` на другом порту и укажите **два URL** в `LOCATION`:

```bash
# Второй экземпляр
mkdir -p /home/gpadmin/gpfdist_data2
cp /home/gpadmin/orders_100k.csv /home/gpadmin/gpfdist_data2/orders_100k_part2.csv
gpfdist -d /home/gpadmin/gpfdist_data2 -p 8082 -l /home/gpadmin/gpfdist2.log &
```

```sql
DROP EXTERNAL TABLE ext_orders;

CREATE EXTERNAL TABLE ext_orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
)
LOCATION (
    'gpfdist://mdw:8081/orders_100k.csv',
    'gpfdist://mdw:8082/orders_100k_part2.csv'
)
FORMAT 'CSV';

TRUNCATE orders;
\timing on
INSERT INTO orders SELECT * FROM ext_orders;
\timing off
```

### Задание 3.4. `gpfdist` и изоляция ошибок

```sql
-- Загрузите файл с ошибками через внешнюю таблицу
CREATE EXTERNAL TABLE ext_orders_bad (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
)
LOCATION ('gpfdist://mdw:8081/orders_bad.csv')
FORMAT 'CSV'
LOG ERRORS SEGMENT REJECT LIMIT 5;

TRUNCATE orders;
INSERT INTO orders SELECT * FROM ext_orders_bad;
SELECT count(*) FROM orders;
```

## Часть 4. Утилита `gpload` 

### Задание 4.1. Подготовка YAML-файла управления загрузкой

Создайте файл `my_load.yml` на хосте `mdw`. Замените имя хоста в `LOCAL_HOSTNAME` на фактический хост, где запущен `gpfdist`:

```yaml
---
VERSION: 1.0.0.1
DATABASE: postgres
USER: gpadmin
HOST: mdw
PORT: 5432
GPLOAD:
  INPUT:
    - SOURCE:
        LOCAL_HOSTNAME:
          - mdw
        PORT: 8081
        FILE:
          - /home/gpadmin/gpfdist_data/orders_100k.csv
    - COLUMNS:
        - order_id: bigint
        - customer_id: integer
        - order_date: date
        - amount: numeric
        - status: text
    - FORMAT: csv
    - DELIMITER: ','
    - ERROR_LIMIT: 25
    - LOG_ERRORS: true
  OUTPUT:
    - TABLE: lab_load.orders
    - MODE: INSERT
  PRELOAD:
    - REUSE_TABLES: true
  SQL:
    - BEFORE: "INSERT INTO lab_load.load_audit VALUES('start', current_timestamp)"
    - AFTER: "INSERT INTO lab_load.load_audit VALUES('end', current_timestamp)"
```


### Задание 4.2. Таблица аудита и запуск `gpload`

```sql
CREATE TABLE IF NOT EXISTS lab_load.load_audit (
    event text,
    ts    timestamp
);
```

```bash
cd /home/gpadmin
gpload -f my_load.yml
```

Проверка:
```sql
TRUNCATE lab_load.orders;
-- После запуска gpload:
SELECT count(*) FROM lab_load.orders;
SELECT * FROM lab_load.load_audit ORDER BY ts;
```

