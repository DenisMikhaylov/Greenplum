# Лабораторная работа: Развёртывание standalone PostgreSQL (90 минут)


**Сценарий:** для курса нужен standalone-сервер PostgreSQL, на котором будет размещена операционная база интернет-магазина `shop_oltp`. В дальнейшем Greenplum через PXF будет читать и записывать данные в эту базу. Лабораторная проходит полный цикл: от подготовки ОС до проверки удалённого подключения с другого хоста.

---


## Часть 0. Подготовка ОС (10 минут)

Все команды выполняются от пользователя с `sudo` на хосте PostgreSQL.

### 0.1. Установка имени хоста и `/etc/hosts`

```bash
sudo hostnamectl set-hostname pgstandalone
```

Добавьте запись в `/etc/hosts`, чтобы хост разрешался по имени:

```bash
echo "192.168.100.20  pgstandalone" | sudo tee -a /etc/hosts
```

> Замените `192.168.100.20` на фактический IP-адрес вашей ВМ.

### 0.2. Отключение SELinux и firewall на учебном стенде

```bash
# SELinux: проверить статус
sestatus

# Если включён — отключить (потребуется перезагрузка)
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# Firewall: отключить на учебном стенде
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

> В продуктивной среде firewall **не отключают**, а открывают порт 5432 только для подсети Greenplum. На учебном стенде отключение упрощает отладку.

### 0.3. Обновление системы и установка утилит

```bash
sudo dnf update -y
sudo dnf install -y wget curl vim net-tools
```

---

## Часть 1. Установка PostgreSQL из PGDG (15 минут)

### 1.1. Подключение репозитория PGDG

Официальный репозиторий PostgreSQL Global Development Group (PGDG) содержит актуальные версии PostgreSQL для Rocky Linux 8. Версия 16 — актуальная стабильная на момент написания.

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### 1.2. Отключение встроенного модуля PostgreSQL

Rocky Linux 8 поставляется с модулем PostgreSQL в AppStream. Чтобы `dnf` не выбрал его вместо пакета PGDG, модуль нужно отключить:

```bash
sudo dnf -qy module disable postgresql
```

### 1.3. Установка сервера и contrib-пакетов

```bash
sudo dnf install -y postgresql16-server postgresql16-contrib
```

`postgresql16-server` — серверная часть, `postgresql16-contrib` — дополнительные расширения (`pg_stat_statements`, `dblink`, `postgres_fdw` и другие).

**Проверка:**
```bash
/usr/pgsql-16/bin/postgres --version
```

---

## Часть 2. Инициализация кластера и запуск (15 минут)

### 2.1. Инициализация кластера

Пакет PGDG не инициализирует кластер автоматически. Утилита `postgresql-16-setup` создаёт кластер в `/var/lib/pgsql/16/data` от имени системного пользователя `postgres`.

```bash
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
```

**Проверка содержимого каталога данных:**
```bash
sudo ls -la /var/lib/pgsql/16/data/
```

Вы должны увидеть `postgresql.conf`, `pg_hba.conf`, `PG_VERSION` и другие файлы.

### 2.2. Запуск и автозапуск

```bash
sudo systemctl enable --now postgresql-16
```

**Проверка статуса:**
```bash
sudo systemctl status postgresql-16
```

Служба должна быть в состоянии `active (running)`.

### 2.3. Проверка локального подключения

```bash
sudo -u postgres psql -c 'SELECT version();'
```

Вы должны увидеть версию PostgreSQL 16.x и информацию о сборке.

### 2.4. Настройка пароля пользователя `postgres`

По умолчанию пользователь `postgres` не имеет пароля и подключается только через локальный сокет. Задайте пароль для административных задач:

```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'postgres_admin_123';"
```

> Пароль `postgres_admin_123` — учебный. В продуктивной среде используйте сложный пароль и храните его в секрете.

---

## Часть 3. Настройка сетевого доступа (15 минут)

По умолчанию PostgreSQL слушает только `localhost`. Для доступа с хостов Greenplum (и для PXF) нужно разрешить прослушивание внешнего интерфейса и настроить аутентификацию.

### 3.1. Настройка `postgresql.conf`

Откройте файл конфигурации:

```bash
sudo vim /var/lib/pgsql/16/data/postgresql.conf
```

Найдите параметр `listen_addresses` и приведите его к виду:

```text
listen_addresses = '*'
```

> **Безопасность:** `'*'` означает прослушивание всех интерфейсов. В продуктивной среде укажите конкретный IP-адрес или используйте `listen_addresses = 'localhost,192.168.100.20'`.

Проверьте, что порт стандартный:

```text
port = 5432
```

### 3.2. Настройка `pg_hba.conf`

Откройте файл:

```bash
sudo vim /var/lib/pgsql/16/data/pg_hba.conf
```

Добавьте строку для доступа из подсети Greenplum. Замените `192.168.100.0/24` на фактическую подсеть вашего кластера Greenplum:

```text
# TYPE  DATABASE    USER        ADDRESS             METHOD
host    shop_oltp   pxfuser     192.168.100.0/24    scram-sha-256
host    all         postgres    192.168.100.0/24    scram-sha-256
```

> **Важно:** порядок строк в `pg_hba.conf` имеет значение — PostgreSQL использует **первую подходящую** строку. Убедитесь, что ваши строки стоят **до** общих правил, если они есть.

### 3.3. Применение изменений

Перезагрузите PostgreSQL, чтобы применить `listen_addresses`:

```bash
sudo systemctl restart postgresql-16
```

**Проверка, что PostgreSQL слушает внешний интерфейс:**

```bash
sudo ss -tlnp | grep 5432
```

Вы должны увидеть `0.0.0.0:5432` или `*:5432`, а не только `127.0.0.1:5432`.

---

## Часть 4. Создание демонстрационной БД `shop_oltp` (15 минут)

### 4.1. Создание базы данных и таблиц

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE shop_oltp;
\connect shop_oltp

CREATE TABLE customers (
    customer_id   integer PRIMARY KEY,
    customer_name text,
    region        text,
    signup_date   date
);

CREATE TABLE orders (
    order_id    bigint PRIMARY KEY,
    customer_id integer REFERENCES customers(customer_id),
    order_date  date,
    amount      numeric(12,2),
    status      text
);
```

### 4.2. Заполнение демонстрационными данными

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

\q
```

**Проверка:**
```bash
sudo -u postgres psql -d shop_oltp -c "SELECT count(*) FROM customers; SELECT count(*) FROM orders;"
```

Ожидаемые результаты: `5` и `7`.

---

## Часть 5. Создание пользователя для PXF (10 минут)

PXF подключается к PostgreSQL от имени выделенного пользователя. Этот пользователь должен иметь доступ к таблицам, которые будут читаться или записываться через внешние таблицы.

### 5.1. Создание пользователя `pxfuser`

```bash
sudo -u postgres psql -d shop_oltp
```

```sql
CREATE USER pxfuser WITH PASSWORD 'pxf_password_123';

GRANT CONNECT ON DATABASE shop_oltp TO pxfuser;
GRANT USAGE ON SCHEMA public TO pxfuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO pxfuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO pxfuser;

\q
```

> **Пароль `pxf_password_123`** — учебный. В продуктивной среде используйте сложный пароль и передавайте его в PXF через защищённое хранилище секретов.

### 5.2. Проверка прав

```bash
sudo -u postgres psql -d shop_oltp -c "\dp customers"
```

Убедитесь, что `pxfuser` имеет права `arwd` (append, read, write, delete) на таблицу `customers`.

---

## Часть 6. Проверка удалённого подключения (10 минут)

### 6.1. Подключение с самого хоста PostgreSQL (проверка `pg_hba.conf`)

```bash
psql -h 192.168.100.20 -U pxfuser -d shop_oltp -c "SELECT count(*) FROM customers;"
```

При запросе пароля введите `pxf_password_123`. Если подключение успешно — `pg_hba.conf` настроен корректно.

> Если подключение не удаётся, проверьте:
> - `listen_addresses` в `postgresql.conf`;
> - строку в `pg_hba.conf` для `192.168.100.0/24`;
> - что PostgreSQL перезапущен после изменения `listen_addresses`;
> - что firewall отключён или порт 5432 открыт.

### 6.2. Подключение с хоста Greenplum

На **любом** хосте Greenplum (например, `mdw`):

```bash
psql -h 192.168.100.20 -U pxfuser -d shop_oltp -c "SELECT count(*) FROM orders;"
```

Это ключевая проверка: PXF будет подключаться к PostgreSQL **с каждого сегментного хоста**, поэтому доступ должен работать со всех хостов кластера, а не только с coordinator.

### 6.3. Проверка с сегментного хоста

На **сегментном** хосте Greenplum (например, `sdw1`):

```bash
psql -h 192.168.100.20 -U pxfuser -d shop_oltp -c "SELECT count(*) FROM orders;"
```

Если подключение с сегментного хоста не работает — PXF не сможет читать данные, даже если coordinator подключается успешно. Убедитесь, что подсеть в `pg_hba.conf` покрывает **все** хосты Greenplum.
