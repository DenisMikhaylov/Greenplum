# Лабораторная работа: PXF — доступ к внешней базе данных PostgreSQL



## Часть 0. Подготовка: создание демонстрационной БД в PostgreSQL (20 минут)

Все команды в этой части выполняются на хосте PostgreSQL (не на Greenplum) от пользователя `postgres`.

### 0.1. Создание базы данных и таблиц

```bash
# Подключение к PostgreSQL
psql -U postgres -h <postgres_host>
```

```sql
CREATE DATABASE shop_oltp;
\connect shop_oltp

-- Клиенты
CREATE TABLE customers (
    customer_id   integer PRIMARY KEY,
    customer_name text,
    region        text,
    signup_date   date
);

-- Заказы
CREATE TABLE orders (
    order_id    bigint PRIMARY KEY,
    customer_id integer REFERENCES customers(customer_id),
    order_date  date,
    amount      numeric(12,2),
    status      text
);
```

### 0.2. Заполнение демонстрационными данными

```sql
INSERT INTO customers VALUES
(1, 'Alice',   'North', '2023-01-15'),
(2, 'Bob',     'South', '2023-03-22'),
(3, 'Carol',   'East',  '2023-06-10'),
(4, 'Dave',    'West',  '2023-09-05'),
(5, 'Eve',     'North', '2024-01-20');

INSERT INTO orders VALUES
(1001, 1, '2024-01-10', 250.00, 'paid'),
(1002, 2, '2024-02-15', 130.50, 'shipped'),
(1003, 1, '2024-03-20', 99.99,  'delivered'),
(1004, 3, '2024-04-05', 500.00, 'paid'),
(1005, 4, '2024-05-12', 75.25,  'new'),
(1006, 2, '2024-06-18', 300.00, 'delivered'),
(1007, 5, '2024-07-22', 180.75, 'paid');
```

### 0.3. Создание пользователя для PXF и выдача прав

PXF подключается к PostgreSQL от имени выделенного пользователя. Этот пользователь должен иметь доступ к таблицам, которые будут читаться или записываться через внешние таблицы.

```sql
CREATE USER pxfuser WITH PASSWORD 'pxf_password_123';

GRANT CONNECT ON DATABASE shop_oltp TO pxfuser;
GRANT USAGE ON SCHEMA public TO pxfuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO pxfuser;

-- Для будущих таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO pxfuser;
```

### 0.4. Настройка `pg_hba.conf` для доступа с хостов Greenplum

PXF будет подключаться к PostgreSQL **с каждого сегментного хоста**, поэтому `pg_hba.conf` должен разрешать подключения `pxfuser` с этих адресов.

Откройте `pg_hba.conf` (обычно `/var/lib/pgsql/data/pg_hba.conf` или `/etc/postgresql/<version>/main/pg_hba.conf`) и добавьте строку для подсети Greenplum (замените `192.168.100.0/24` на фактическую подсеть вашего кластера):

```text
# TYPE  DATABASE    USER        ADDRESS             METHOD
host    shop_oltp   pxfuser     192.168.100.0/24    md5
```

Перезагрузите PostgreSQL:

```bash
sudo systemctl reload postgresql
# или
pg_ctl reload -D $PGDATA
```

**Проверка подключения** (с любого хоста Greenplum):

```bash
psql -h <postgres_host> -U pxfuser -d shop_oltp -c 'SELECT count(*) FROM customers;'
```

> **Важно:** PXF должен иметь сетевой доступ к PostgreSQL **с каждого сегментного хоста**. Если между хостами есть firewall, порт PostgreSQL (по умолчанию 5432) должен быть открыт.


## Часть 1. Архитектура PXF: как сегменты читают внешние данные (10 минут)

Прежде чем переходить к настройке, зафиксируем архитектурную картину — она объясняет, почему PXF даёт параллелизм и где искать проблемы при отладке.

**PXF-кластер** — это набор процессов **PXF Service**, по одному на каждом хосте Greenplum. Процессы на coordinator и standby coordinator **не участвуют в передаче данных**; продуктивная работа выполняется процессами на сегментных хостах.

Когда пользователь выполняет запрос к внешней таблице PXF:

1. **Coordinator** разбирает запрос и диспетчеризует его **всем сегментным экземплярам**.
2. Каждый **сегментный экземпляр** обращается к **локальному PXF Service** на том же хосте.
3. PXF Service выделяет **рабочий поток** для каждого сегментного экземпляра.
4. Внутри потока работают три компонента коннектора:
   - **Fragmenter** создаёт список фрагментов данных, которые можно читать **параллельно**.
   - **Accessor** создаёт список записей из одного фрагмента.
   - **Resolver** выполняет десериализацию и сериализацию записей — преобразование между форматом внешнего источника и типами Greenplum.

Для JDBC-коннектора **Fragmenter** определяет, как разбить удалённую таблицу на диапазоны (например, по `id`), чтобы каждый сегмент читал свою порцию строк. **Accessor** выполняет `SELECT` к PostgreSQL через JDBC. **Resolver** преобразует `ResultSet` в строки, понятные Greenplum.

**Ключевое следствие:** coordinator не является узким местом передачи данных, как в `COPY`. Пропускная способность масштабируется с числом сегментов, каждый из которых независимо подключается к PostgreSQL.


## Часть 2. Коннекторы PXF: JDBC-коннектор и профиль `jdbc` (10 минут)

**Коннектор** — это класс источников, для которых PXF умеет читать и писать данные. Встроенные коннекторы покрывают Hadoop (HDFS, Hive, HBase), объектные хранилища (S3, Azure, GCS, MinIO) и **SQL-базы через JDBC** (PostgreSQL, MySQL, Oracle, SQL Server, DB2 и другие).

Для работы с конкретным источником используются два понятия:

- **Сервер** — именованная конфигурация: куда подключаться, от чьего имени, с какими параметрами. Создаётся один раз администратором.
- **Профиль** — именованное отображение, определяющее, **как** читать/писать формат. Для JDBC-коннектора существует **единственный профиль `jdbc`**.

**Ключевая особенность JDBC-коннектора:** он **не гарантирует согласованности при записи**. Если `INSERT` завершается ошибкой, часть данных может уже попасть во внешнюю таблицу. Для критичных сценариев документация рекомендует писать в staging-таблицу и загружать в целевую только после проверки.

PXF **поставляется со встроенным JDBC-драйвером PostgreSQL** (`postgresql-42.4.3.jar` или новее). Для PostgreSQL дополнительная установка драйвера **не требуется**.


## Часть 3. Настройка сервера PXF для PostgreSQL (20 минут)

Все команды выполняются на **coordinator-хосте Greenplum** от пользователя `gpadmin`.

### 3.1. Создание каталога сервера и файла конфигурации

PXF использует файловую конфигурацию сервера: каталог с именем сервера и файл `jdbc-site.xml` внутри него.

```bash
# Перейдите в каталог серверов PXF
cd $PXF_BASE/servers

# Создайте каталог для нового сервера (имя произвольное, например pgsrvcfg)
mkdir pgsrvcfg
cd pgsrvcfg

# Скопируйте шаблон из установки PXF
cp $PXF_HOME/templates/jdbc-site.xml .
```

> **Где находится `$PXF_BASE`:** обычно `/usr/local/pxf-gp6` или `/usr/local/pxf`. Проверьте: `echo $PXF_BASE`. Если переменная не задана, найдите каталог через `find / -name "pxf-site.xml" 2>/dev/null`.

### 3.2. Редактирование `jdbc-site.xml`

Откройте файл и приведите его к следующему виду, заменив `<postgres_host>` и `<postgres_port>` на фактические значения:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>jdbc.driver</name>
        <value>org.postgresql.Driver</value>
    </property>
    <property>
        <name>jdbc.url</name>
        <value>jdbc:postgresql://&lt;postgres_host&gt;:&lt;postgres_port&gt;/shop_oltp</value>
    </property>
    <property>
        <name>jdbc.user</name>
        <value>pxfuser</value>
    </property>
    <property>
        <name>jdbc.password</name>
        <value>pxf_password_123</value>
    </property>
</configuration>
```

**Обязательные свойства**:
- `jdbc.driver` — класс JDBC-драйвера. Для PostgreSQL: `org.postgresql.Driver`.
- `jdbc.url` — URL подключения. Формат для PostgreSQL: `jdbc:postgresql://host:port/database`.
- `jdbc.user` — пользователь, от имени которого PXF подключается к PostgreSQL.
- `jdbc.password` — пароль пользователя.

> **Безопасность:** документация PXF прямо указывает, что при файловой конфигурации сервера учётные данные хранятся **в открытом виде**. Для продуктивной среды используйте внешние хранилища секретов и ограничивайте права на файл.

### 3.3. Синхронизация конфигурации на все хосты

После создания или изменения конфигурации сервера её нужно **распространить на все хосты Greenplum**:

```bash
pxf cluster sync
```

Эта команда копирует `$PXF_BASE/servers/pgsrvcfg` на все хосты кластера. Без неё сегментные хосты не увидят конфигурацию, и запросы к внешней таблице завершатся ошибкой.

### 3.4. Проверка синхронизации

```bash
pxf cluster status
```

Убедитесь, что PXF запущен на всех хостах и конфигурация доступна. Если PXF не запущен:

```bash
pxf cluster start
```


## Часть 4. Внешние таблицы PXF: чтение из PostgreSQL (25 минут)

Все SQL-команды выполняются в `psql` на coordinator-хосте Greenplum от `gpadmin`.

### 4.1. Создание читаемой внешней таблицы

Синтаксис `CREATE EXTERNAL TABLE` для PXF: в `LOCATION` указывается URI с протоколом `pxf://`, за которым следует **путь к данным в формате `<schema>.<table>`** для JDBC-коннектора.

```sql
CREATE EXTERNAL TABLE pxf_customers (
    customer_id   integer,
    customer_name text,
    region        text,
    signup_date   date
)
LOCATION ('pxf://public.customers?PROFILE=jdbc&SERVER=pgsrvcfg')
FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import');
```

**Разбор компонентов:**
- `pxf://public.customers` — `public` — схема в PostgreSQL, `customers` — имя таблицы.
- `PROFILE=jdbc` — профиль JDBC-коннектора.
- `SERVER=pgsrvcfg` — имя каталога сервера, созданного в части 3.
- `FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import')` — формат, который PXF использует для **чтения** из внешнего источника.

### 4.2. Чтение данных

```sql
SELECT * FROM pxf_customers;
```

Ожидаемый результат — 5 строк, соответствующих строкам таблицы `customers` в PostgreSQL.

**Проверка параллелизма:** выполните запрос с `EXPLAIN`, чтобы увидеть, как PXF-запрос распределяется по сегментам:

```sql
EXPLAIN SELECT count(*) FROM pxf_customers;
```

Вы увидите `External Scan` на каждом сегменте — это подтверждает, что каждый сегмент независимо читает свою порцию данных из PostgreSQL.

### 4.3. Создание внешней таблицы для заказов и JOIN с локальными данными

```sql
CREATE EXTERNAL TABLE pxf_orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
)
LOCATION ('pxf://public.orders?PROFILE=jdbc&SERVER=pgsrvcfg')
FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import');
```

Теперь можно выполнять **federated-запросы**, соединяя внешние таблицы PostgreSQL с локальными таблицами Greenplum. Создадим локальную таблицу-справочник:

```sql
CREATE TABLE local_regions (
    region text,
    manager text
) DISTRIBUTED REPLICATED;

INSERT INTO local_regions VALUES
('North', 'Ivanov'),
('South', 'Petrov'),
('East',  'Sidorov'),
('West',  'Kuznetsov');
```

```sql
SELECT r.region, lr.manager, sum(o.amount) AS total_sales
FROM pxf_orders o
JOIN pxf_customers c ON o.customer_id = c.customer_id
JOIN local_regions lr ON c.region = lr.region
GROUP BY r.region, lr.manager
ORDER BY total_sales DESC;
```

**Вопрос:** какие строки читаются через PXF, а какие — локально? Как вы думаете, где выполняется `GROUP BY`?

### 4.4. Загрузка внешних данных в локальную таблицу

Если данные из PostgreSQL нужны многократно, их выгоднее загрузить в Greenplum — это даст локальное хранение, индексы и статистику:

```sql
CREATE TABLE local_customers AS
SELECT * FROM pxf_customers
DISTRIBUTED BY (customer_id);

ANALYZE local_customers;
```

**Вопрос:** в чём разница между запросом к `pxf_customers` и `local_customers` с точки зрения плана выполнения и источника данных?


## Часть 5. Внешние таблицы PXF: запись в PostgreSQL (15 минут)

JDBC-коннектор поддерживает **два направления**: чтение и запись. Для записи создаётся **отдельная writable-таблица** — та же читаемая таблица **не может** использоваться для записи в том же сеансе.

### 5.1. Создание writable-внешней таблицы

```sql
CREATE WRITABLE EXTERNAL TABLE pxf_write_orders (
    order_id    bigint,
    customer_id integer,
    order_date  date,
    amount      numeric(12,2),
    status      text
)
LOCATION ('pxf://public.orders?PROFILE=jdbc&SERVER=pgsrvcfg')
FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export');
```

**Отличие:** `WRITABLE` и `FORMATTER='pxfwritable_export'` вместо `import`.

### 5.2. Запись данных в PostgreSQL

```sql
INSERT INTO pxf_write_orders VALUES
(2001, 1, '2024-08-01', 150.00, 'new'),
(2002, 2, '2024-08-02', 220.50, 'paid'),
(2003, 3, '2024-08-03', 75.00,  'shipped');
```

**Проверка на стороне PostgreSQL** (подключитесь к `shop_oltp`):

```sql
SELECT * FROM orders WHERE order_id >= 2001 ORDER BY order_id;
```

Строки должны присутствовать. Теперь проверьте через **читаемую** внешнюю таблицу `pxf_orders` в Greenplum:

```sql
SELECT * FROM pxf_orders WHERE order_id >= 2001 ORDER BY order_id;
```

