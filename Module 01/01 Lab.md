# Лабораторная работа: Развертывание кластера Greenplum на стенде из 3 хостов Debian 12

## Важное предварительное замечание

---

## Цель работы

Развернуть минимальный MPP-кластер Greenplum из трёх хостов под управлением Debian 12, пройдя полный цикл: подготовка ОС → установка пакета → настройка хранилища → инициализация → проверка работоспособности.

---

## Состав стенда и распределение ролей

| Хост | Роль | Функции |
|---|---|---|
| `mdw` | **Coordinator (Master)** | Точка входа для клиентов, планирование и диспетчеризация запросов |
| `sdw1` | **Segment host 1** | Хранение и обработка данных (primary + mirror сегменты) |
| `sdw2` | **Segment host 2** | Хранение и обработка данных (primary + mirror сегменты) |

**Конфигурация сегментов:** на каждом сегментном хосте создаётся **по 2 primary-сегмента** и **по 2 mirror-сегмента** (зеркала распределяются на противоположный хост). Итого кластер содержит 4 primary-сегмента и 4 mirror-сегмента.

> **Примечание:** Standby Coordinator в данной лабораторной **не разворачивается** в силу ограниченности стенда тремя хостами. В продуктивной среде Standby Master обязателен.

**Схема размещения сегментов:**

```text
sdw1: gpseg0 (primary), gpseg1 (primary),  gpseg2 (mirror), gpseg3 (mirror)
sdw2: gpseg2 (primary), gpseg3 (primary),  gpseg0 (mirror), gpseg1 (mirror)
```

---

## Требования к виртуальным машинам

- **ОС:** Debian 12 (x86_64), минимальная установка.
- **RAM:** не менее **4 GB** на хост (для учебного стенда; продуктивный минимум — 16 GB).
- **Диск:** не менее **20 GB** на хост, файловая система для данных — **XFS** (рекомендуется) или ext4.
- **Сеть:** статическая IP-адресация, все хосты в одной подсети, беспарольный SSH между всеми хостами от имени `gpadmin`.
- **Пакет Greenplum:** DEB-пакет `greenplum-db-6.x.x-ubuntu18.04-amd64.deb` (или актуальная версия для Ubuntu/Debian), скачанный с greenplum.org или из репозитория проекта.

---

## Часть 1. Подготовка операционной системы (на всех трёх хостах)

Все команды в этом разделе выполняются **от root** или через `sudo` на **каждом** хосте: `mdw`, `sdw1`, `sdw2`.

### 1.1. Установка базовых зависимостей

DEB-пакет Greenplum для Ubuntu/Debian автоматически подтягивает большинство зависимостей, но для Debian 12 часть пакетов может потребоваться установить вручную.

```bash
apt update
apt install -y \
  bash bzip2 curl iproute2 less net-tools openssh-server openssh-client \
  openssl libapr1 libaprutil1 libcurl4 libevent-2.1-7 libxml2 libyaml-0-2 \
  libopenldap2-2.6 libreadline8 libzstd1 zlib1g perl python3 rsync sed tar which zip \
  locales xfsprogs
wget https://github.com/greenplum-db/gpdb/releases/download/6.24.0/greenplum-db-6.24.0-ubuntu18.04-amd64.deb
```

> **Если `libopenldap2-2.6` недоступен в Debian 12**, проверьте доступные версии: `apt-cache search libopenldap`. Имя пакета может отличаться (например, `libldap-2.5-0`); DEB-пакет Greenplum может требовать конкретную версию, и при несовпадении потребуется либо симлинк, либо сборка из исходников.

### 1.2. Создание пользователя `gpadmin`

UID/GID **должны совпадать** на всех хостах.

```bash
groupadd -g 1001 gpadmin
useradd -g gpadmin -u 1001 -m -s /bin/bash gpadmin
passwd gpadmin   # задайте пароль
```

Добавьте `gpadmin` в `sudoers` (удобно для выполнения административных команд):

```bash
echo 'gpadmin ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/gpadmin
chmod 440 /etc/sudoers.d/gpadmin
```

### 1.3. Настройка `/etc/hosts` (на всех хостах, одинаково)

Замените IP-адреса на фактические адреса ваших ВМ:

```bash
cat >> /etc/hosts <<'EOF'
192.168.100.10  mdw
192.168.100.11  sdw1
192.168.100.12  sdw2
EOF
```

Проверка разрешения имён:

```bash
for h in mdw sdw1 sdw2; do ping -c1 -W1 $h && echo "$h OK"; done
```

### 1.4. Параметры ядра (`/etc/sysctl.conf`)

Создайте файл с настройками, адаптированными под учебный стенд (значения `shmmax`/`shmall` рассчитаны на 4 GB RAM; для продуктивной среды рассчитывайте от фактического объёма памяти):

```bash
cat > /etc/sysctl.d/90-greenplum.conf <<'EOF'
kernel.shmmax = 2147483648
kernel.shmall = 524288
kernel.shmmni = 4096
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 10000 65535
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
vm.swappiness = 10
EOF

sysctl -p /etc/sysctl.d/90-greenplum.conf
```

> **Важно:** `net.ipv4.ip_local_port_range = 10000 65535` означает, что порты сегментов (начиная с `PORT_BASE = 6000`) **не должны** попадать в этот диапазон. В конфигурации ниже используется `PORT_BASE=6000` и `MIRROR_PORT_BASE=7000` — оба значения ниже 10000, конфликта не будет.

### 1.5. Лимиты ресурсов (`/etc/security/limits.conf`)

```bash
cat >> /etc/security/limits.conf <<'EOF'
gpadmin soft nofile 65536
gpadmin hard nofile 65536
gpadmin soft nproc 131072
gpadmin hard nproc 131072
EOF
```

Проверка (от `gpadmin`, после повторного входа):

```bash
su - gpadmin -c 'ulimit -n; ulimit -u'
```

### 1.6. Отключение SELinux и firewall

В Debian 12 SELinux по умолчанию не активен, но проверьте:

```bash
getenforce 2>/dev/null || echo "SELinux not active"
```

Firewall на учебном стенде отключается, чтобы избежать блокировки interconnect и случайных портов:

```bash
systemctl stop nftables 2>/dev/null || true
systemctl disable nftables 2>/dev/null || true
# Если используется ufw:
ufw disable 2>/dev/null || true
```

> В продуктивной среде firewall **не отключают**, а настраивают правила для портов interconnect (см. лекцию 2, блок 9.3).

### 1.7. Настройка беспарольного SSH для `gpadmin`

Выполняется **от имени `gpadmin` на хосте `mdw`** (ключ генерируется один раз и распространяется на все хосты):

```bash
su - gpadmin
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

# Распространение ключа на все хосты, включая сам mdw
for host in mdw sdw1 sdw2; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub gpadmin@$host
done

# Проверка беспарольного доступа
for host in mdw sdw1 sdw2; do
  ssh -o BatchMode=yes gpadmin@$host hostname && echo "$host SSH OK"
done
```

### 1.8. Создание области хранения

**На Master (`mdw`):**

```bash
sudo mkdir -p /data/coordinator
sudo chown gpadmin:gpadmin /data/coordinator
```

**На сегментных хостах (`sdw1`, `sdw2`) — через `gpssh` с `mdw`:**

```bash
# На mdw, от gpadmin
cat > /home/gpadmin/hostfile_segonly <<'EOF'
sdw1
sdw2
EOF

# Требуется установленный Greenplum для gpssh; если gpssh ещё нет — используйте обычный ssh в цикле:
for host in sdw1 sdw2; do
  ssh gpadmin@$host "sudo mkdir -p /data/primary /data/mirror && sudo chown -R gpadmin:gpadmin /data/primary /data/mirror"
done
```

Проверка файловой системы (должна быть XFS или ext4):

```bash
df -hT /data/primary /data/mirror
```

---

## Часть 2. Установка пакета Greenplum (на всех трёх хостах)

### 2.1. Копирование DEB-пакета

Скопируйте пакет в домашний каталог `gpadmin` на **каждом** хосте:

```bash
# С локальной машины, где лежит пакет:
for host in mdw sdw1 sdw2; do
  scp greenplum-db-*.deb gpadmin@$host:/home/gpadmin/
done
```

### 2.2. Установка пакета

На каждом хосте **от root** (или через `sudo`):

```bash
sudo apt install -y /home/gpadmin/greenplum-db-*.deb
```

`apt` автоматически установит зависимости и создаст:
- каталог `/usr/local/greenplum-db-<version>`;
- символическую ссылку `/usr/local/greenplum-db` на этот каталог.

> **Возможная проблема на Debian 12:** если `apt` сообщает о неудовлетворённых зависимостях (например, требуется `libldap-2.4-2`, которого нет в Debian 12), установите пакет с игнорированием зависимостей и вручную добавьте недостающие библиотеки:
> ```bash
> sudo dpkg -i --force-depends /home/gpadmin/greenplum-db-*.deb
> sudo apt install -f   # попытаться доустановить зависимости
> ```
> Если конкретная версия библиотеки отсутствует, проверьте, доступен ли совместимый soname, и при необходимости создайте симлинк **с осторожностью** (это учебный стенд, не продуктив).

### 2.3. Смена владельца

```bash
sudo chown -R gpadmin:gpadmin /usr/local/greenplum-db-*
sudo chown -h gpadmin:gpadmin /usr/local/greenplum-db
```

### 2.4. Первичная проверка

На `mdw`, от `gpadmin`:

```bash
su - gpadmin
source /usr/local/greenplum-db/greenplum_path.sh
which gpinitsystem gpcheckperf gpstate gpssh
```

Все четыре утилиты должны быть найдены в `PATH`.

---

## Часть 3. Проверка производительности системы (`gpcheckperf`)

Выполняется **с `mdw`**, после того как пакет установлен на всех хостах и SSH настроен.

### 3.1. Подготовка hostfile

```bash
cat > /home/gpadmin/hostfile_gpcheckperf <<'EOF'
sdw1
sdw2
EOF
```

### 3.2. Дисковый I/O и память

```bash
source /usr/local/greenplum-db/greenplum_path.sh

gpcheckperf -f /home/gpadmin/hostfile_gpcheckperf \
  -d /data/primary -d /data/mirror -r ds
```

- `-r ds` — тесты **d**isk и **s**tream (пропускная способность памяти).
- Результат покажет MB/s для каждого хоста. В учебном стенде значения могут быть невысокими (виртуальные диски), но **разброс между `sdw1` и `sdw2` не должен быть значительным**.

### 3.3. Сетевой тест (interconnect)

```bash
gpcheckperf -f /home/gpadmin/hostfile_gpcheckperf -r N -d /tmp
```

- `-r N` — параллельный парный тест сети.
- Если пропускная способность ниже ~100 MB/s, запустите последовательный тест `-r n` для локализации проблемы.

> **Типичная проблема на Debian:** `gpcheckperf` использует `dd` и стандартные сетевые утилиты; убедитесь, что пакет `net-tools` установлен (он включён в зависимости в разделе 1.1).

---

## Часть 4. Инициализация кластера Greenplum

Все команды выполняются **только на `mdw`**, от имени `gpadmin`.

### 4.1. Конфигурационный файл `gpinitsystem_config`

```bash
cat > /home/gpadmin/gpinitsystem_config <<'EOF'
ARRAY_NAME="GP_LAB"
MACHINE_LIST_FILE=/home/gpadmin/hostfile_gpinitsystem
SEG_PREFIX=gpseg
PORT_BASE=6000

declare -a DATA_DIRECTORY=(/data/primary /data/primary)

MASTER_HOSTNAME=mdw
MASTER_DIRECTORY=/data/coordinator
MASTER_PORT=5432

TRUSTED_SHELL=ssh
ENCODING=UTF8

# Зеркала
MIRROR_PORT_BASE=7000
declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror)
EOF
```

**Пояснения к конфигурации:**
- `DATA_DIRECTORY` содержит **два пути** — на каждом сегментном хосте будет создано **2 primary-сегмента**.
- `MIRROR_DATA_DIRECTORY` также содержит два пути — на каждом хосте будет создано **2 mirror-сегмента**.
- `PORT_BASE=6000` и `MIRROR_PORT_BASE=7000` — оба диапазона ниже `ip_local_port_range` (10000–65535), поэтому конфликта с эфемерными портами ОС не будет.

### 4.2. Hostfile для инициализации

```bash
cat > /home/gpadmin/hostfile_gpinitsystem <<'EOF'
sdw1
sdw2
EOF
```

### 4.3. Запуск `gpinitsystem`

```bash
source /usr/local/greenplum-db/greenplum_path.sh

gpinitsystem -c /home/gpadmin/gpinitsystem_config \
  -h /home/gpadmin/hostfile_gpinitsystem \
  -S
```

- `-c` — конфигурационный файл.
- `-h` — hostfile сегментных хостов.
- `-S` — развернуть **зеркальные сегменты** (spread mirror). Флаг `-s <standby>` **не используется**, поскольку Standby Master в данной лабораторной отсутствует.

Утилита выполнит предварительные проверки и запросит подтверждение:

```text
=> Continue with Greenplum creation? Yy/Nn
```

Введите `Y`. При успехе в выводе появится:

```text
=> Greenplum Database instance successfully created.
```

> **Если `gpinitsystem` завершается с ошибкой на этапе проверки SSH**, убедитесь, что беспарольный доступ работает **в обе стороны** (не только с `mdw` на сегменты, но и с сегментов на `mdw`, и между сегментами). `gpinitsystem` проверяет SSH-связность в том числе между сегментными хостами.

### 4.4. Настройка переменных окружения

Добавьте в `~/.bashrc` пользователя `gpadmin` на `mdw` (и на сегментных хостах, если планируете подключаться к ним напрямую):

```bash
cat >> /home/gpadmin/.bashrc <<'EOF'
source /usr/local/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/data/coordinator/gpseg-1
export PGPORT=5432
export PGUSER=gpadmin
export PGDATABASE=postgres
EOF

source /home/gpadmin/.bashrc
```

Проверка:

```bash
echo $MASTER_DATA_DIRECTORY
gpstate
```

---

## Часть 5. Проверка кластера

### 5.1. Статус компонентов

```bash
gpstate
```

Ожидаемый вывод (сокращённо):

```text
Greenplum instance status: Up
Master host: mdw
Master port: 5432
...
Primary segments: 4 (4 primary, 4 mirror)
```

### 5.2. Детальная конфигурация

```bash
gpstate -s
```

Убедитесь, что:
- **все 4 primary-сегмента** в статусе `Up`;
- **все 4 mirror-сегмента** в статусе `Up` (синхронизированы);
- Standby Master отсутствует (это ожидаемо для данной лабораторной).

### 5.3. Статус зеркал

```bash
gpstate -m
```

Все mirror-сегменты должны быть в состоянии `Synchronized`.

### 5.4. Проверка ошибок

```bash
gpstate -e
```

Не должно быть сегментов в статусе `Down` или `Not Synchronized`.

### 5.5. Тестовое SQL-подключение

```bash
psql -d postgres -c "SELECT version();"
psql -d postgres -c "SELECT gp_segment_id, count(*) FROM gp_dist_random('gp_id') GROUP BY 1 ORDER BY 1;"
```

Второй запрос вернёт по одной строке на каждый primary-сегмент — это подтверждает, что данные распределены и все сегменты доступны для выполнения запросов.

---
