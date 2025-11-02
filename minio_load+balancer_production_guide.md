# Инструкция по установке и настройке production MinIO

## Содержание

1. [Введение](#введение)
2. [Основные концепции MinIO](#основные-концепции-minio)
3. [Примеры сетевой топологии](#примеры-сетевой-топологии)
4. [Требования к окружению](#требования-к-окружению)
5. [Установка MinIO](#установка-minio)
6. [Настройка Storage Class](#настройка-storage-class)
7. [Настройка балансировщиков нагрузки](#настройка-балансировщиков-нагрузки)
8. [Масштабирование деплоймента](#масштабирование-деплоймента)
9. [Настройка репликации](#настройка-репликации-active-active)
10. [Backup и восстановление](#backup-и-восстановление)
11. [Мониторинг и обслуживание](#мониторинг-и-обслуживание)
12. [Troubleshooting](#troubleshooting-быстрая-диагностика)
13. [Рекомендации по production](#рекомендации-по-production-окружению)

---

## Введение

MinIO — это высокопроизводительное объектное хранилище с S3-совместимым API, которое хорошо себя зарекомендовало для организации on-premise S3-хранилищ. Это решение отличается простотой первоначальной настройки, хорошей документацией и надежностью в эксплуатации.

Данная инструкция сфокусирована на production-развертывании MinIO с акцентом на отказоустойчивость и сохранность данных. Мы рассмотрим оптимальные конфигурации, способы масштабирования и настройку репликации между деплойментами.

**Важно:** Данное руководство предполагает базовое знакомство с MinIO и Linux системами.

---

## Основные концепции MinIO

### Архитектура хранения данных

MinIO использует многоуровневую архитектуру для обеспечения надежности:

**Erasure Set** — базовая единица хранения, представляющая собой набор от 4 до 16 дисков. MinIO автоматически выбирает размер erasure set, стремясь использовать максимальное количество дисков (до 16).

**Data и Parity Drives:**
- **Data drives (N)** — диски, на которых хранятся данные
- **Parity drives (M)** — диски четности для восстановления данных при отказах
- Ограничение: M ≤ N/2 (дисков четности не может быть больше половины размера erasure set)

**Server Pool** — логическое объединение одного или нескольких erasure set. При наличии более 16 дисков в кластере создается несколько erasure set, которые объединяются в server pool.

**Storage Class** — классы хранения, определяющие уровень избыточности:
- `STANDARD` — стандартный класс с максимальной избыточностью
- `REDUCED_REDUNDANCY` (RRS) — сниженная избыточность для менее критичных данных

### Принцип работы

MinIO разбивает загружаемые файлы на множество небольших частей (chunks) и распределяет их по эндпоинтам (смонтированным дискам) с учетом настроенного уровня избыточности. При отказе до M дисков в erasure set данные могут быть восстановлены благодаря дискам четности.

**Критично:** Каждый объект хранится только в пределах одного erasure set. Отказ более чем M дисков в одном erasure set приведет к недоступности всех объектов, хранящихся в этом сете.

---

## Примеры сетевой топологии

### Минимальная production конфигурация (рекомендуемая)

**Конфигурация 4×4:**
- 4 сервера (ноды)
- 4 диска на каждом сервере
- Итого: 16 дисков
- 1 erasure set размером 16 дисков
- Рекомендуемая настройка: EC:4 (4 диска четности)

**Преимущества:**
- Отказоустойчивость: до 4 дисков или 1 ноды целиком
- Один erasure set — предсказуемость отказов
- Оптимальное соотношение надежности и используемого места

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node 1    │  │   Node 2    │  │   Node 3    │  │   Node 4    │
├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤
│  Disk 1     │  │  Disk 1     │  │  Disk 1     │  │  Disk 1     │
│  Disk 2     │  │  Disk 2     │  │  Disk 2     │  │  Disk 2     │
│  Disk 3     │  │  Disk 3     │  │  Disk 3     │  │  Disk 3     │
│  Disk 4     │  │  Disk 4     │  │  Disk 4     │  │  Disk 4     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
                   Erasure Set 0 (16 дисков)
                   EC:4 (12 data + 4 parity)
```

### Расширенная конфигурация 4×8

**Не рекомендуется для production** из-за наличия двух erasure set:
- 4 сервера
- 8 дисков на каждом сервере
- Итого: 32 диска
- 2 erasure set по 16 дисков

**Проблема:** При массовом отказе дисков доступность объектов будет зависеть от того, в каком erasure set они находятся, что добавляет неопределенности.

### Geo-distributed конфигурация

**Два деплоймента с ACTIVE-ACTIVE репликацией:**

```
┌──────────────────────────────┐       ┌──────────────────────────────┐
│     Датацентр A              │       │     Датацентр B              │
│  ┌────────────────────────┐  │       │  ┌────────────────────────┐  │
│  │   MinIO Cluster A      │  │◄─────►│  │   MinIO Cluster B      │  │
│  │   (4×4 конфигурация)   │  │  Rep  │  │   (4×4 конфигурация)   │  │
│  └────────────────────────┘  │       │  └────────────────────────┘  │
└──────────────────────────────┘       └──────────────────────────────┘
```

Обеспечивает защиту от катастрофических отказов (пожар в ЦОД, полный отказ инфраструктуры).

---

## Требования к окружению

### Аппаратные требования

**Минимальные требования для production (на одну ноду):**
- CPU: 4+ ядра
- RAM: 16+ GB (рекомендуется 32 GB)
- Сеть: 10 Gbit/s между нодами (минимум 1 Gbit/s)
- Диски: 
  - Используйте одинаковые диски во всех нодах
  - Рекомендуются SSD или NVMe для высокой производительности
  - Все диски в erasure set должны быть одного размера

### Программные требования

- ОС: Linux (Ubuntu 20.04+, CentOS 7+, Debian 10+)
- Файловая система: XFS (рекомендуется) или EXT4
- Синхронизация времени: NTP/Chrony
- Открытые порты:
  - 9000 — API MinIO
  - 9001 — Console MinIO (опционально)

### Подготовка дисков

**Важно:** MinIO не рекомендует использовать overlay-решения типа LVM. Используйте прямую разметку дисков.

Пример разметки с помощью `parted`:

```bash
# Для каждого диска
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 0% 100%

# Создание файловой системы
mkfs.xfs /dev/sdb1 -L storage1

# Монтирование
mkdir -p /data/storage1
echo "LABEL=storage1 /data/storage1 xfs defaults,noatime 0 0" >> /etc/fstab
mount -a
```

Повторите для всех дисков, используя уникальные метки (storage1, storage2, и т.д.).

### Сетевая настройка

Убедитесь в наличии DNS-записей или настройте `/etc/hosts` на всех нодах:

```bash
# /etc/hosts на всех нодах
10.0.1.11  minio-node-1
10.0.1.12  minio-node-2
10.0.1.13  minio-node-3
10.0.1.14  minio-node-4
```

---

## Установка MinIO

### Установка бинарного файла

На всех нодах кластера:

```bash
# Скачивание последней версии
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# Проверка установки
minio --version
```

### Создание пользователя и директорий

```bash
# Создание пользователя
sudo useradd -r -s /sbin/nologin minio-user

# Назначение прав на директории дисков
sudo chown -R minio-user:minio-user /data/storage*
```

### Конфигурация systemd

Создайте файл `/etc/default/minio` на всех нодах:

```bash
# /etc/default/minio

# Список томов (эндпоинтов)
# Для конфигурации 4×4:
MINIO_VOLUMES="https://minio-node-{1...4}:9000/data/storage{1...4}"

# Учетные данные root
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=your-strong-password-here

# Опционально: настройки storage class
MINIO_STORAGE_CLASS_STANDARD=EC:2
# MINIO_STORAGE_CLASS_RRS=EC:1

# Адрес для API
MINIO_SERVER_URL=https://minio.example.com:9000

# Дополнительные опции
MINIO_OPTS="--certs-dir /etc/minio/certs --console-address :9001"
```

**Важные параметры:**
- `MINIO_VOLUMES` — определяет топологию кластера
- Формат `{1...4}` — это bash-expansion для диапазона
- Используйте HTTPS для production (см. раздел про TLS)

Создайте systemd unit файл `/etc/systemd/system/minio.service`:

```ini
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local

User=minio-user
Group=minio-user
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Настройки перезапуска
Restart=always
RestartSec=5s

# Лимиты
LimitNOFILE=65536
TasksMax=infinity

# Логирование
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Настройка TLS (обязательно для production)

```bash
# Создание директории для сертификатов
sudo mkdir -p /etc/minio/certs

# Размещение сертификатов
sudo cp public.crt /etc/minio/certs/public.crt
sudo cp private.key /etc/minio/certs/private.key

# Опционально: CA сертификаты
sudo mkdir -p /etc/minio/certs/CAs
sudo cp ca.crt /etc/minio/certs/CAs/

# Установка прав
sudo chown -R minio-user:minio-user /etc/minio/certs
sudo chmod 600 /etc/minio/certs/private.key
```

### Запуск кластера

**Критично:** Запускайте MinIO одновременно на всех нодах!

```bash
# На всех нодах одновременно
sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio

# Проверка статуса
sudo systemctl status minio
sudo journalctl -u minio -f
```

### Установка MinIO Client (mc)

```bash
# На управляющей машине или одной из нод
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Добавление алиаса
mc alias set myminio https://minio.example.com:9000 admin your-strong-password-here

# Проверка подключения
mc admin info myminio
```

### Проверка развертывания

```bash
# Получение информации о кластере
mc admin info myminio

# Вывод должен показать:
# - Все ноды (4 nodes)
# - Все диски (16 drives)
# - Server pool с одним erasure set
# - Storage class настройки
```

Пример вывода:

```
●  minio-node-1:9000
   Uptime: 2 hours
   Version: 2024-XX-XX
   Network: 4/4 OK
   Drives: 4/4 OK

...

Pools:
   1st, Erasure sets: 1, Drives per erasure set: 16

16 drives online, 0 drives offline, EC:2
```

---

## Настройка Storage Class

Storage class определяет уровень избыточности данных и количество дисков четности.

### Параметры конфигурации

```bash
# В файле /etc/default/minio
MINIO_STORAGE_CLASS_STANDARD=EC:2
MINIO_STORAGE_CLASS_RRS=EC:1
```

Где:
- `EC:N` — количество parity дисков
- `STANDARD` — для критичных данных
- `RRS` (Reduced Redundancy Storage) — для менее важных данных

### Выбор количества parity дисков

**Формула:** M ≤ N/2, где:
- N — размер erasure set
- M — количество parity дисков

**Для конфигурации 4×4 (16 дисков):**
- Максимум M = 8
- Рекомендуется EC:2 или EC:4

**Сравнение настроек:**

| EC | Надежность | Доступное место | Допустимые отказы |
|----|------------|-----------------|-------------------|
| EC:1 | Низкая | 93.75% | 1 диск |
| EC:2 | Средняя | 87.5% | 2 диска |
| EC:4 | Высокая | 75% | 4 диска |
| EC:6 | Очень высокая | 62.5% | 6 дисков |

**Рекомендация:** Для production используйте EC:2 как минимум, EC:4 для критичных данных.

### Применение storage class при загрузке

```bash
# Загрузка со стандартным классом (по умолчанию)
mc cp file.txt myminio/bucket/

# Загрузка с reduced redundancy
mc cp file.txt myminio/bucket/ --storage-class REDUCED_REDUNDANCY

# Установка класса по умолчанию для бакета через S3 API
aws s3api put-bucket-lifecycle-configuration \
  --bucket mybucket \
  --lifecycle-configuration file://lifecycle.json
```

### Просмотр текущих настроек

```bash
# Через mc
mc admin config get myminio storage_class

# Вывод:
# storage_class standard=EC:2 rrs=EC:1
```

### Изменение storage class

**Важно:** Изменение storage class требует перезапуска кластера и влияет только на новые объекты!

```bash
# 1. Обновите /etc/default/minio на всех нодах
MINIO_STORAGE_CLASS_STANDARD=EC:4

# 2. Перезапустите кластер (на всех нодах одновременно)
sudo systemctl restart minio

# 3. Проверьте изменения
mc admin config get myminio storage_class
```

### Особенности REDUCED_REDUNDANCY

**Внимание:** Использование RRS класса требует особой осторожности!

- При отказе более M дисков (для RRS) данные будут утеряны
- Даже при наличии репликации между деплойментами
- Рекомендуется только для временных или легко восстанавливаемых данных

**Пример потери данных RRS:**

При EC:2 для STANDARD и EC:1 для RRS:
- Отказ 2 дисков в erasure set: STANDARD данные доступны, RRS данные утеряны
- Восстановление RRS возможно только из реплики (см. раздел репликации)

---

## Настройка балансировщиков нагрузки

Балансировщик нагрузки является критически важным компонентом production-инфраструктуры MinIO. Он обеспечивает:
- Распределение нагрузки между нодами кластера
- Отказоустойчивость на уровне доступа
- SSL/TLS termination
- Health checks и автоматическое исключение неработающих нод

### Архитектура с балансировщиком

```
                           ┌──────────────┐
                           │   Internet   │
                           └──────┬───────┘
                                  │
                        ┌─────────▼──────────┐
                        │  Virtual IP (VIP)  │
                        │    10.0.1.100      │
                        └─────────┬──────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
            ┌───────▼────────┐         ┌────────▼───────┐
            │ Load Balancer 1│         │ Load Balancer 2│
            │   (MASTER)     │◄───────►│    (BACKUP)    │
            │  10.0.1.21     │ VRRP    │   10.0.1.22    │
            └───────┬────────┘         └────────┬───────┘
                    │                           │
         ┌──────────┼───────────┬───────────────┘
         │          │           │
    ┌────▼────┐ ┌──▼─────┐ ┌───▼────┐ ┌────────┐
    │ MinIO 1 │ │ MinIO 2│ │ MinIO 3│ │ MinIO 4│
    │:9000    │ │:9000   │ │:9000   │ │:9000   │
    └─────────┘ └────────┘ └────────┘ └────────┘
```

### Вариант 1: Nginx (рекомендуется)

#### Установка Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo yum install nginx

# Или установка из официального репозитория
sudo add-apt-repository ppa:nginx/stable
sudo apt update
sudo apt install nginx
```

#### Конфигурация Nginx для MinIO

Создайте файл `/etc/nginx/conf.d/minio.conf`:

```nginx
# Upstream для MinIO API серверов
upstream minio_api {
    least_conn;
    
    # Основные параметры подключения
    keepalive 32;
    keepalive_timeout 60s;
    
    # MinIO ноды с health checks
    server minio-node-1:9000 max_fails=3 fail_timeout=30s weight=1;
    server minio-node-2:9000 max_fails=3 fail_timeout=30s weight=1;
    server minio-node-3:9000 max_fails=3 fail_timeout=30s weight=1;
    server minio-node-4:9000 max_fails=3 fail_timeout=30s weight=1;
}

# Upstream для MinIO Console
upstream minio_console {
    least_conn;
    keepalive 16;
    
    server minio-node-1:9001 max_fails=2 fail_timeout=30s;
    server minio-node-2:9001 max_fails=2 fail_timeout=30s;
    server minio-node-3:9001 max_fails=2 fail_timeout=30s;
    server minio-node-4:9001 max_fails=2 fail_timeout=30s;
}

# Редирект с HTTP на HTTPS
server {
    listen 80;
    server_name minio.example.com;
    
    # Разрешить Let's Encrypt challenge
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    
    # Редирект всего остального на HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# MinIO API Server (HTTPS)
server {
    listen 443 ssl http2;
    server_name minio.example.com;
    
    # SSL сертификаты
    ssl_certificate /etc/nginx/ssl/minio.crt;
    ssl_certificate_key /etc/nginx/ssl/minio.key;
    
    # SSL настройки (современные и безопасные)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # Игнорирование некорректных заголовков
    ignore_invalid_headers off;
    
    # Поддержка больших файлов
    client_max_body_size 0;
    client_body_buffer_size 512k;
    
    # Таймауты для больших файлов и длительных операций
    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
    send_timeout 300s;
    
    # Буферизация
    proxy_buffering off;
    proxy_request_buffering off;
    
    # Отключение chunked transfer encoding для совместимости
    chunked_transfer_encoding off;
    
    # Логирование
    access_log /var/log/nginx/minio-access.log;
    error_log /var/log/nginx/minio-error.log;
    
    location / {
        proxy_pass https://minio_api;
        
        # Заголовки для проксирования
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # HTTP/1.1 для keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Отключение кеширования (важно для MinIO)
        proxy_cache off;
        
        # Retry настройки
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;
    }
    
    # Health check endpoint (опционально)
    location /minio/health/live {
        proxy_pass https://minio_api/minio/health/live;
        proxy_set_header Host $http_host;
        access_log off;
    }
}

# MinIO Console Server (HTTPS)
server {
    listen 443 ssl http2;
    server_name console.minio.example.com;
    
    # SSL сертификаты
    ssl_certificate /etc/nginx/ssl/minio.crt;
    ssl_certificate_key /etc/nginx/ssl/minio.key;
    
    # SSL настройки
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Логирование
    access_log /var/log/nginx/minio-console-access.log;
    error_log /var/log/nginx/minio-console-error.log;
    
    location / {
        proxy_pass https://minio_console;
        
        # Заголовки
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy true;
        
        # WebSocket поддержка (необходимо для Console)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Таймауты
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}
```

#### Дополнительные настройки Nginx

Отредактируйте `/etc/nginx/nginx.conf` для оптимизации:

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Логирование
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    
    # Производительность
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Таймауты
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # Размеры буферов
    client_header_buffer_size 4k;
    large_client_header_buffers 4 16k;
    
    # Gzip (только для текстовых ответов, не для объектов)
    gzip off;
    
    # Включение конфигураций
    include /etc/nginx/conf.d/*.conf;
}
```

#### Проверка и запуск Nginx

```bash
# Проверка конфигурации
sudo nginx -t

# Перезапуск Nginx
sudo systemctl restart nginx

# Включение автозапуска
sudo systemctl enable nginx

# Проверка статуса
sudo systemctl status nginx
```

### Вариант 2: HAProxy (альтернатива)

HAProxy предоставляет более продвинутые возможности балансировки и health checks.

#### Установка HAProxy

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install haproxy

# CentOS/RHEL
sudo yum install haproxy

# Или установка последней версии (2.8+)
sudo add-apt-repository ppa:vbernat/haproxy-2.8
sudo apt update
sudo apt install haproxy
```

#### Конфигурация HAProxy

Создайте файл `/etc/haproxy/haproxy.cfg`:

```haproxy
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Максимальное количество соединений
    maxconn 40000
    
    # SSL настройки
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    tune.ssl.default-dh-param 2048
    
    # Оптимизация производительности
    tune.bufsize 32768
    tune.maxrewrite 8192

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch
    
    # Таймауты
    timeout connect 10s
    timeout client 300s
    timeout server 300s
    timeout http-request 10s
    timeout http-keep-alive 10s
    timeout check 5s
    
    # Retry настройки
    retries 3
    
    # Максимальное количество соединений
    maxconn 10000
    
    # Error files
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

#---------------------------------------------------------------------
# Statistics page
#---------------------------------------------------------------------
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats show-legends
    stats show-node
    stats auth admin:your-stats-password

#---------------------------------------------------------------------
# MinIO API Frontend
#---------------------------------------------------------------------
frontend minio_api_front
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/minio.pem
    
    # Редирект HTTP на HTTPS
    http-request redirect scheme https code 301 unless { ssl_fc }
    
    # Security headers
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-Frame-Options "SAMEORIGIN"
    
    # ACL для разделения трафика
    acl is_console hdr(host) -i console.minio.example.com
    
    # Маршрутизация
    use_backend minio_console if is_console
    default_backend minio_api

#---------------------------------------------------------------------
# MinIO API Backend
#---------------------------------------------------------------------
backend minio_api
    mode http
    balance leastconn
    
    # Отключение compression (важно для MinIO)
    compression offload
    
    # Health check
    option httpchk GET /minio/health/live
    http-check expect status 200
    
    # Retry на других серверах при ошибках
    option redispatch
    option abortonclose
    
    # HTTP keep-alive
    option http-keep-alive
    
    # Заголовки
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Forwarded-Host %[req.hdr(Host)]
    
    # MinIO серверы
    server minio1 minio-node-1:9000 check ssl verify none maxconn 2000 weight 100
    server minio2 minio-node-2:9000 check ssl verify none maxconn 2000 weight 100
    server minio3 minio-node-3:9000 check ssl verify none maxconn 2000 weight 100
    server minio4 minio-node-4:9000 check ssl verify none maxconn 2000 weight 100

#---------------------------------------------------------------------
# MinIO Console Backend
#---------------------------------------------------------------------
backend minio_console
    mode http
    balance leastconn
    
    # Health check
    option httpchk GET /
    http-check expect status 200
    
    # WebSocket support
    option http-server-close
    option forwardfor
    
    # Заголовки для WebSocket
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    
    # MinIO Console серверы
    server console1 minio-node-1:9001 check ssl verify none maxconn 1000
    server console2 minio-node-2:9001 check ssl verify none maxconn 1000
    server console3 minio-node-3:9001 check ssl verify none maxconn 1000
    server console4 minio-node-4:9001 check ssl verify none maxconn 1000
```

#### Подготовка SSL сертификата для HAProxy

HAProxy требует объединенный PEM файл:

```bash
# Создание объединенного сертификата
sudo cat /etc/ssl/minio.crt /etc/ssl/minio.key > /etc/haproxy/certs/minio.pem

# Установка прав
sudo chmod 600 /etc/haproxy/certs/minio.pem
sudo chown haproxy:haproxy /etc/haproxy/certs/minio.pem
```

#### Проверка и запуск HAProxy

```bash
# Проверка конфигурации
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Перезапуск HAProxy
sudo systemctl restart haproxy

# Включение автозапуска
sudo systemctl enable haproxy

# Проверка статуса
sudo systemctl status haproxy

# Просмотр статистики
# Откройте в браузере: http://your-lb-ip:8404/stats
```

### Вариант 3: Angie (форк Nginx)

Angie — это российский форк Nginx с дополнительными возможностями.

#### Установка Angie

```bash
# Добавление репозитория
curl -o /etc/yum.repos.d/angie.repo https://angie.software/repo/angie.repo

# Или для Debian/Ubuntu
curl -o /etc/apt/sources.list.d/angie.list https://angie.software/repo/angie.list
curl https://angie.software/keys/GPG-KEY-A1D273D5 | sudo apt-key add -

# Установка
sudo apt update && sudo apt install angie
# или
sudo yum install angie
```

Конфигурация Angie аналогична Nginx (используйте конфигурацию из Варианта 1).

### Настройка Keepalived для HA балансировщиков

Для обеспечения отказоустойчивости балансировщиков используется Keepalived с VRRP.

#### Установка Keepalived

```bash
# Ubuntu/Debian
sudo apt install keepalived

# CentOS/RHEL
sudo yum install keepalived
```

#### Конфигурация на MASTER балансировщике

Создайте файл `/etc/keepalived/keepalived.conf`:

```bash
# Глобальные настройки
global_defs {
    router_id LB1
    vrrp_skip_check_adv_addr
    vrrp_strict
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    enable_script_security
    script_user root
}

# Health check script для Nginx/HAProxy
vrrp_script check_lb {
    script "/usr/bin/systemctl is-active nginx"  # или haproxy
    interval 2
    weight -20
    fall 2
    rise 2
}

# VRRP instance для API
vrrp_instance VI_API {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePassword123
    }
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0 label eth0:vip_api
    }
    
    track_script {
        check_lb
    }
    
    # Уведомления при смене состояния
    notify_master "/etc/keepalived/notify.sh MASTER VI_API"
    notify_backup "/etc/keepalived/notify.sh BACKUP VI_API"
    notify_fault "/etc/keepalived/notify.sh FAULT VI_API"
}

# VRRP instance для Console (опционально отдельный VIP)
vrrp_instance VI_CONSOLE {
    state MASTER
    interface eth0
    virtual_router_id 52
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePassword456
    }
    
    virtual_ipaddress {
        10.0.1.101/24 dev eth0 label eth0:vip_console
    }
    
    track_script {
        check_lb
    }
}
```

#### Конфигурация на BACKUP балансировщике

Файл `/etc/keepalived/keepalived.conf` на втором LB:

```bash
global_defs {
    router_id LB2
    vrrp_skip_check_adv_addr
    vrrp_strict
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    enable_script_security
    script_user root
}

vrrp_script check_lb {
    script "/usr/bin/systemctl is-active nginx"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_API {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90  # Ниже чем на MASTER
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePassword123
    }
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0 label eth0:vip_api
    }
    
    track_script {
        check_lb
    }
    
    notify_master "/etc/keepalived/notify.sh MASTER VI_API"
    notify_backup "/etc/keepalived/notify.sh BACKUP VI_API"
    notify_fault "/etc/keepalived/notify.sh FAULT VI_API"
}

vrrp_instance VI_CONSOLE {
    state BACKUP
    interface eth0
    virtual_router_id 52
    priority 90
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePassword456
    }
    
    virtual_ipaddress {
        10.0.1.101/24 dev eth0 label eth0:vip_console
    }
    
    track_script {
        check_lb
    }
}
```

#### Скрипт уведомлений

Создайте файл `/etc/keepalived/notify.sh`:

```bash
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

case $STATE in
    "MASTER")
        logger "Keepalived: $NAME transitioned to MASTER state"
        # Здесь можно добавить логику для уведомлений
        ;;
    "BACKUP")
        logger "Keepalived: $NAME transitioned to BACKUP state"
        ;;
    "FAULT")
        logger "Keepalived: $NAME is in FAULT state"
        # Отправка алертов
        ;;
    *)
        logger "Keepalived: $NAME unknown state"
        ;;
esac
```

```bash
# Установка прав
sudo chmod +x /etc/keepalived/notify.sh
```

#### Запуск Keepalived

```bash
# На обоих балансировщиках
sudo systemctl enable keepalived
sudo systemctl start keepalived

# Проверка статуса
sudo systemctl status keepalived

# Просмотр логов
sudo journalctl -u keepalived -f

# Проверка VIP
ip addr show eth0
```

### Тестирование балансировщика

#### Проверка доступности через балансировщик

```bash
# Проверка API endpoint
curl -k https://minio.example.com/minio/health/live

# Или через VIP
curl -k https://10.0.1.100/minio/health/live

# Проверка Console
curl -k https://console.minio.example.com/

# Тест загрузки файла через балансировщик
mc alias set lb-minio https://minio.example.com:443 admin password
mc mb lb-minio/test-bucket
echo "test" > test.txt
mc cp test.txt lb-minio/test-bucket/
mc ls lb-minio/test-bucket/
```

#### Тестирование отказоустойчивости

```bash
# 1. Остановка одной из MinIO нод
ssh minio-node-1 "sudo systemctl stop minio"

# 2. Проверка, что запросы продолжают работать
for i in {1..10}; do
    curl -k https://minio.example.com/minio/health/live
    sleep 1
done

# 3. Запуск ноды обратно
ssh minio-node-1 "sudo systemctl start minio"

# 4. Тест failover балансировщика (на MASTER LB)
sudo systemctl stop nginx  # или haproxy

# 5. Проверка переключения VIP
ip addr show eth0  # На BACKUP должен появиться VIP

# 6. Проверка доступности
curl -k https://10.0.1.100/minio/health/live
```

### Мониторинг балансировщиков

#### Nginx метрики

```bash
# Установка модуля stub_status
# Добавьте в конфигурацию Nginx:

server {
    listen 127.0.0.1:8080;
    
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}

# Просмотр метрик
curl http://127.0.0.1:8080/nginx_status
```

#### HAProxy статистика

```bash
# HAProxy имеет встроенную статистику
# Доступна на http://lb-ip:8404/stats

# Или через socat
echo "show stat" | socat stdio /run/haproxy/admin.sock

# Проверка состояния backend серверов
echo "show servers state" | socat stdio /run/haproxy/admin.sock
```

#### Prometheus exporters

```bash
# Nginx Prometheus Exporter
# https://github.com/nginxinc/nginx-prometheus-exporter

# HAProxy Prometheus Exporter (встроен в HAProxy 2.0+)
# Добавьте в конфигурацию:

frontend prometheus
    bind *:8405
    http-request use-service prometheus-exporter if { path /metrics }
```

### Оптимизация производительности балансировщика

#### Тюнинг ядра Linux

Создайте файл `/etc/sysctl.d/99-balancer-tuning.conf`:

```bash
# Сетевые буферы
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# Увеличение лимита соединений
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 5000

# TCP оптимизация
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_tw_reuse = 1

# Лимиты файловых дескрипторов
fs.file-max = 2097152
fs.nr_open = 2097152

# Conntrack для высоконагруженных систем
net.netfilter.nf_conntrack_max = 1048576
net.nf_conntrack_max = 1048576
```

Применение:

```bash
sudo sysctl -p /etc/sysctl.d/99-balancer-tuning.conf
```

#### Увеличение лимитов для сервиса

Для Nginx/HAProxy:

```bash
# Создайте файл /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65535
LimitNPROC=65535

# Перезагрузка
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

### DNS настройка

Настройте DNS записи для балансировщиков:

```bash
# В вашей DNS зоне
minio.example.com.           A    10.0.1.100
console.minio.example.com.   A    10.0.1.101

# Или используйте CNAME если VIP на отдельном домене
minio.example.com.           CNAME  lb-vip.example.com.
console.minio.example.com.   CNAME  lb-vip.example.com.
```

---

## Масштабирование деплоймента

MinIO поддерживает два способа масштабирования:

1. **Добавление server pool** (рекомендуется)
2. **Расширение дисков** (ограниченное применение)

### Добавление нового server pool

Это cloud-like подход к масштабированию, рекомендуемый разработчиками.

**Требования:**
- Новый pool должен соответствовать глобальным storage class настройкам
- Размер erasure set в новом pool должен удовлетворять M ≤ N/2
- Рекомендуется использовать идентичную конфигурацию серверов

**Шаги:**

1. **Подготовьте новые серверы** (например, minio-node-5 до minio-node-8)

2. **Обновите конфигурацию на всех нодах** (старых и новых):

```bash
# /etc/default/minio
# До:
MINIO_VOLUMES="https://minio-node-{1...4}:9000/data/storage{1...4}"

# После:
MINIO_VOLUMES="https://minio-node-{1...4}:9000/data/storage{1...4} https://minio-node-{5...8}:9000/data/storage{1...4}"
```

3. **Перезапустите кластер одновременно на всех нодах:**

```bash
# На ВСЕХ нодах (включая новые) одновременно
sudo systemctl restart minio
```

4. **Проверьте добавление pool:**

```bash
mc admin info myminio

# Вывод покажет:
# Pools:
#   1st, Erasure sets: 1, Drives per erasure set: 16
#   2nd, Erasure sets: 1, Drives per erasure set: 16
```

5. **Обновите конфигурацию балансировщика:**

```bash
# Добавьте новые ноды в upstream (Nginx)
upstream minio_api {
    least_conn;
    server minio-node-1:9000 max_fails=3 fail_timeout=30s;
    server minio-node-2:9000 max_fails=3 fail_timeout=30s;
    server minio-node-3:9000 max_fails=3 fail_timeout=30s;
    server minio-node-4:9000 max_fails=3 fail_timeout=30s;
    server minio-node-5:9000 max_fails=3 fail_timeout=30s;
    server minio-node-6:9000 max_fails=3 fail_timeout=30s;
    server minio-node-7:9000 max_fails=3 fail_timeout=30s;
    server minio-node-8:9000 max_fails=3 fail_timeout=30s;
}

# Перезагрузите балансировщик
sudo nginx -t && sudo systemctl reload nginx
```

### Распределение данных между pool

По умолчанию новые объекты размещаются в pool с наибольшим свободным местом. Существующие объекты остаются в своих pool.

**Ребалансировка данных:**

```bash
# Запуск ребалансировки
mc admin rebalance start myminio

# Проверка статуса
mc admin rebalance status myminio

# Остановка ребалансировки
mc admin rebalance stop myminio
```

**Важно:** 
- Ребалансировка — тяжелая операция
- Время выполнения зависит от объема данных
- Может значительно нагрузить систему
- Запускайте в периоды низкой нагрузки

### Расширение существующих дисков

**Применимость:** Когда невозможно добавить новый pool.

**Требования:**
- Все диски в erasure set должны быть расширены до одинакового размера
- MinIO автоматически определит новый размер без перезапуска

**Процедура:**

```bash
# 1. Расширьте LUN/диски на уровне гипервизора или storage

# 2. Расширьте партицию (для каждого диска)
parted /dev/sdb resizepart 1 100%

# 3. Расширьте файловую систему
xfs_growfs /data/storage1  # для XFS
resize2fs /dev/sdb1        # для EXT4

# 4. Проверьте, что MinIO увидел изменения
mc admin info myminio
```

**Важно:** Этот метод не рекомендуется как основной способ масштабирования. Используйте добавление server pool.

---

## Настройка репликации (ACTIVE-ACTIVE)

Репликация между деплойментами обеспечивает:
- Защиту от катастрофических отказов
- Географическое распределение данных
- Возможность проксирования запросов при недоступности данных

### Типы репликации

1. **ACTIVE-PASSIVE** — односторонняя репликация
2. **ACTIVE-ACTIVE** — двусторонняя репликация (рекомендуется)
3. **Multi-site ACTIVE-ACTIVE** — репликация между более чем двумя сайтами

Рассмотрим настройку ACTIVE-ACTIVE между двумя деплойментами.

### Предварительные требования

- Два работающих MinIO кластера
- Сетевая связность между кластерами
- MinIO Client (mc) установлен

**Алиасы для примера:**
- `minio-site-a` — первый деплоймент
- `minio-site-b` — второй деплоймент

### Шаг 1: Создание бакетов с версионированием

```bash
# На обоих деплойментах создаем бакет
mc mb minio-site-a/replicated-bucket
mc mb minio-site-b/replicated-bucket

# Включаем версионирование (обязательно для репликации)
mc version enable minio-site-a/replicated-bucket
mc version enable minio-site-b/replicated-bucket
```

### Шаг 2: Создание политики для репликации

Создайте файл `replication-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning",
        "s3:GetBucketObjectLockConfiguration",
        "s3:GetEncryptionConfiguration"
      ],
      "Resource": [
        "arn:aws:s3:::*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ReplicateTags",
        "s3:AbortMultipartUpload",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:GetObjectVersionTagging",
        "s3:PutObject",
        "s3:PutObjectRetention",
        "s3:PutBucketObjectLockConfiguration",
        "s3:PutObjectLegalHold",
        "s3:DeleteObject",
        "s3:ReplicateObject",
        "s3:ReplicateDelete"
      ],
      "Resource": [
        "arn:aws:s3:::*"
      ]
    }
  ]
}
```

Примените политику на обоих деплойментах:

```bash
mc admin policy create minio-site-a ReplicationPolicy replication-policy.json
mc admin policy create minio-site-b ReplicationPolicy replication-policy.json
```

### Шаг 3: Создание пользователей для репликации

```bash
# На site-a
mc admin user add minio-site-a replication-user-a StrongPassword123

# На site-b
mc admin user add minio-site-b replication-user-b StrongPassword456

# Назначение политик
mc admin policy attach minio-site-a ReplicationPolicy --user=replication-user-a
mc admin policy attach minio-site-b ReplicationPolicy --user=replication-user-b
```

### Шаг 4: Настройка репликации

```bash
# Site A -> Site B
mc replicate add minio-site-a/replicated-bucket \
  --remote-bucket 'https://replication-user-b:StrongPassword456@minio-site-b.example.com:9000/replicated-bucket' \
  --replicate "delete,delete-marker,existing-objects" \
  --priority 1

# Site B -> Site A
mc replicate add minio-site-b/replicated-bucket \
  --remote-bucket 'https://replication-user-a:StrongPassword123@minio-site-a.example.com:9000/replicated-bucket' \
  --replicate "delete,delete-marker,existing-objects" \
  --priority 1
```

**Параметры `--replicate`:**
- `delete` — реплицировать удаление объектов
- `delete-marker` — реплицировать маркеры удаления (для версионирования)
- `existing-objects` — реплицировать существующие объекты

### Шаг 5: Синхронный vs Асинхронный режим

**По умолчанию:** асинхронная репликация

Для синхронной репликации добавьте флаг `--sync enable`:

```bash
mc replicate add minio-site-a/replicated-bucket \
  --remote-bucket '...' \
  --replicate "delete,delete-marker,existing-objects" \
  --sync enable
```

**Синхронный режим:**
- Клиент получает подтверждение только после репликации
- Выше надежность, но ниже производительность
- Рекомендуется для критичных данных

### Шаг 6: Проверка репликации

```bash
# Статус репликации
mc replicate status minio-site-a/replicated-bucket
mc replicate status minio-site-b/replicated-bucket

# Детальная информация
mc replicate ls minio-site-a/replicated-bucket --json | jq
```

**Тестирование:**

```bash
# Загрузка файла в site-a
mc cp testfile.txt minio-site-a/replicated-bucket/

# Проверка появления в site-b (подождите несколько секунд)
mc ls minio-site-b/replicated-bucket/

# Удаление из site-b
mc rm minio-site-b/replicated-bucket/testfile.txt

# Проверка удаления из site-a
mc ls minio-site-a/replicated-bucket/
```

### Проксирование запросов

При ACTIVE-ACTIVE репликации MinIO автоматически проксирует запросы к недоступным объектам на реплику.

**Пример использования:**

```bash
# Допустим, в site-a отказало M+1 дисков в erasure set
# Объекты из этого set недоступны локально

# Но запрос все равно успешен благодаря проксированию:
mc cp minio-site-a/replicated-bucket/file.txt ./
# Файл получен из site-b автоматически
```

**Управление проксированием:**

```bash
# Отключить проксирование
mc replicate update minio-site-a/replicated-bucket \
  --state disable \
  --id $(mc replicate ls minio-site-a/replicated-bucket --json | jq -r '.rule.ID')

# Включить проксирование
mc replicate update minio-site-a/replicated-bucket \
  --state enable \
  --id $(mc replicate ls minio-site-a/replicated-bucket --json | jq -r '.rule.ID')
```

### Фильтрация репликации

Можно настроить репликацию только для определенных префиксов или тегов:

```bash
# Репликация только объектов с префиксом /important
mc replicate add minio-site-a/replicated-bucket \
  --remote-bucket '...' \
  --path 'important/*' \
  --replicate "delete,delete-marker"

# Репликация по тегам
mc replicate add minio-site-a/replicated-bucket \
  --remote-bucket '...' \
  --tags "priority=high&environment=production"
```

---

## Backup и восстановление

### Resync — восстановление из реплики

Если данные в одном из деплойментов были утеряны (например, RRS класс при отказе дисков), их можно восстановить из реплики:

```bash
# Получение remote bucket
REMOTE_BUCKET=$(mc replicate ls minio-site-a/replicated-bucket --json | jq -r '.rule.Destination.Bucket')

# Запуск resync
mc replicate resync start minio-site-a/replicated-bucket \
  --remote-bucket "$REMOTE_BUCKET"

# Проверка статуса
mc replicate resync status minio-site-a/replicated-bucket \
  --remote-bucket "$REMOTE_BUCKET"
```

**Процесс resync:**
- Сравнивает объекты в обоих бакетах
- Копирует недостающие объекты из реплики
- Может занять значительное время для больших объемов

### Mirror — зеркалирование бакетов

Для создания полной копии бакета:

```bash
# Однократное зеркалирование
mc mirror minio-site-a/source-bucket minio-site-b/backup-bucket

# Непрерывное зеркалирование (watch mode)
mc mirror --watch minio-site-a/source-bucket minio-site-b/backup-bucket

# С удалением файлов, отсутствующих в источнике
mc mirror --remove minio-site-a/source-bucket minio-site-b/backup-bucket
```

### Snapshot бакетов

Создание snapshot через версионирование:

```bash
# Включить версионирование
mc version enable myminio/bucket

# Все изменения будут сохраняться как версии
mc cp newfile.txt myminio/bucket/file.txt  # version 1
mc cp updated.txt myminio/bucket/file.txt  # version 2

# Просмотр версий
mc ls --versions myminio/bucket/

# Восстановление конкретной версии
mc cp --version-id "VERSION_ID" myminio/bucket/file.txt ./restored-file.txt
```

### Экспорт метаданных

```bash
# Экспорт списка объектов
mc ls --recursive myminio/bucket > bucket-inventory.txt

# Экспорт с метаданными
mc stat --recursive myminio/bucket/ > bucket-metadata.json
```

### Object Lock для защиты от удаления

```bash
# Включение Object Lock (только при создании бакета)
mc mb --with-lock myminio/protected-bucket

# Установка retention policy
mc retention set --default GOVERNANCE 30d myminio/protected-bucket

# Установка legal hold на конкретный объект
mc legalhold set myminio/protected-bucket/important-file.txt
```

---

## Мониторинг и обслуживание

### Мониторинг здоровья кластера

**Основные команды:**

```bash
# Общая информация о кластере
mc admin info myminio

# Детальная информация в JSON
mc admin info myminio --json | jq

# Проверка дисков
mc admin info myminio --json | jq '.info.servers[].drives[] | {endpoint, state, uuid}'

# Статистика использования
mc admin prometheus metrics myminio
```

**Ключевые метрики для мониторинга:**

```bash
# Статус дисков (должны быть "ok")
mc admin info myminio --json | jq '.info.servers[].drives[].state'

# Процент использования
mc admin info myminio --json | jq '.info.usage'

# Количество объектов
mc admin info myminio --json | jq '.info.objects'

# Сетевая статистика
mc admin info myminio --json | jq '.info.servers[].network'
```

### Процедура Heal

Heal восстанавливает консистентность данных после замены дисков или других сбоев:

```bash
# Запуск heal для всего кластера
mc admin heal -r myminio

# Heal конкретного бакета
mc admin heal -r myminio/bucket-name

# Проверка статуса heal
mc admin heal myminio

# Heal в фоновом режиме (по умолчанию)
# MinIO автоматически запускает heal при обнаружении проблем
```

**Когда запускать heal вручную:**
- После замены неисправного диска
- После восстановления ноды
- При обнаружении несоответствий в данных
- После сетевых проблем

### Замена неисправного диска

**Процедура:**

1. **Идентификация проблемного диска:**

```bash
mc admin info myminio --json | jq '.info.servers[].drives[] | select(.state != "ok")'
```

2. **Физическая замена диска** (при необходимости)

3. **Переразметка и монтирование:**

```bash
# На проблемной ноде
parted /dev/sdX mklabel gpt
parted /dev/sdX mkpart primary 0% 100%
mkfs.xfs /dev/sdX1 -L storageN
mount -a
chown -R minio-user:minio-user /data/storageN
```

4. **Heal запустится автоматически** или запустите вручную:

```bash
mc admin heal -r myminio
```

5. **Мониторинг восстановления:**

```bash
# Отслеживание прогресса
watch -n 5 'mc admin heal myminio'
```

**Время восстановления** зависит от:
- Объема данных в erasure set
- Производительности дисков
- Сетевой пропускной способности
- Нагрузки на кластер

### Регулярное обслуживание

**Еженедельные задачи:**

```bash
# Проверка здоровья кластера
mc admin info myminio

# Проверка статуса репликации
mc replicate status myminio/replicated-bucket

# Анализ использования
mc du myminio/
```

**Ежемесячные задачи:**

```bash
# Аудит безопасности
mc admin user list myminio
mc admin policy list myminio

# Проверка версий объектов
mc ls --versions myminio/bucket/ | wc -l

# Очистка старых версий (если нужно)
mc rm --recursive --versions --older-than 90d myminio/bucket/
```

### Prometheus мониторинг

MinIO экспортирует метрики в формате Prometheus:

```bash
# Эндпоинт метрик
curl http://minio-node-1:9000/minio/v2/metrics/cluster

# Настройка в prometheus.yml
scrape_configs:
  - job_name: 'minio'
    metrics_path: /minio/v2/metrics/cluster
    static_configs:
      - targets: ['minio-node-1:9000']
```

**Важные метрики:**

- `minio_cluster_disk_offline_total` — офлайн диски
- `minio_cluster_disk_total` — всего дисков
- `minio_cluster_nodes_offline_total` — офлайн ноды
- `minio_heal_objects_heal_total` — объекты в процессе heal
- `minio_s3_requests_total` — количество запросов
- `minio_s3_requests_errors_total` — ошибки запросов

### Логирование

**Просмотр логов:**

```bash
# Через journalctl
sudo journalctl -u minio -f

# Через mc
mc admin logs myminio

# Установка уровня логирования
mc admin config set myminio logger_webhook:mywebhook endpoint="http://logger:8080"
```

**Важные события для отслеживания:**
- Disk offline/online события
- Heal операции
- Репликация ошибки
- API ошибки

---

## Troubleshooting (Быстрая диагностика)

### Проблема: Диск показывает offline

**Диагностика:**

```bash
# Проверка статуса дисков
mc admin info myminio --json | jq '.info.servers[].drives[] | select(.state == "offline")'

# Проверка монтирования на ноде
df -h | grep /data/storage

# Проверка логов
sudo journalctl -u minio -n 100 | grep -i "drive\|disk\|error"
```

**Решение:**

```bash
# 1. Проверить монтирование
mount | grep /data/storage

# 2. Перемонтировать при необходимости
sudo umount /data/storageX
sudo mount -a

# 3. Проверить права доступа
ls -la /data/storageX
sudo chown -R minio-user:minio-user /data/storageX

# 4. Перезапустить MinIO на проблемной ноде
sudo systemctl restart minio

# 5. Запустить heal
mc admin heal -r myminio
```

### Проблема: Кластер не стартует

**Диагностика:**

```bash
# Проверка статуса на всех нодах
for i in {1..4}; do
    echo "=== Node $i ==="
    ssh minio-node-$i "systemctl status minio"
done

# Проверка логов
sudo journalctl -u minio -n 50

# Проверка сетевой связности
for i in {1..4}; do
    curl -k https://minio-node-$i:9000/minio/health/live
done
```

**Типичные причины:**

1. **Разная конфигурация MINIO_VOLUMES:**
```bash
# Проверка на всех нодах
for i in {1..4}; do
    echo "=== Node $i ==="
    ssh minio-node-$i "grep MINIO_VOLUMES /etc/default/minio"
done
```

2. **Несоответствие storage class настроек:**
```bash
# Проверка
grep MINIO_STORAGE_CLASS /etc/default/minio
```

3. **Проблемы с сертификатами:**
```bash
# Проверка сертификатов
ls -la /etc/minio/certs/
openssl x509 -in /etc/minio/certs/public.crt -text -noout
```

**Решение:**
- Синхронизировать конфигурацию на всех нодах
- Перезапустить одновременно все ноды
- Проверить firewall и SELinux

### Проблема: Медленная репликация

**Диагностика:**

```bash
# Статус репликации
mc replicate status myminio/bucket

# Метрики репликации
mc admin prometheus metrics myminio | grep replication

# Сетевая статистика
iperf3 -c minio-site-b -p 9000
```

**Оптимизация:**

```bash
# Увеличение workers для репликации
mc admin config set myminio api replication_workers=250

# Переключение на синхронную репликацию (если нужна надежность)
mc replicate update myminio/bucket --sync enable --id REPLICATION_ID

# Проверка bandwidth ограничений
mc admin config get myminio api
```

### Проблема: Heal не завершается

**Диагностика:**

```bash
# Статус heal
mc admin heal myminio

# Проверка проблемных объектов
mc admin heal myminio --json | jq '.items[] | select(.heal_status != "ok")'
```

**Решение:**

```bash
# Принудительный heal конкретного бакета
mc admin heal -r --force myminio/problematic-bucket

# При критических проблемах - остановка и повторный запуск
mc admin heal --stop myminio
# Подождать завершения
mc admin heal -r myminio
```

### Проблема: "Too many open files"

**Диагностика:**

```bash
# Проверка лимитов
ulimit -n
cat /proc/$(pgrep minio)/limits | grep "open files"
```

**Решение:**

```bash
# В /etc/systemd/system/minio.service
[Service]
LimitNOFILE=65536

# Перезагрузка конфигурации
sudo systemctl daemon-reload
sudo systemctl restart minio
```

### Проблема: Объекты недоступны после отказа

**Диагностика:**

```bash
# Определение erasure set объекта
# (поиск на эндпоинтах вручную)
for i in {1..8}; do
    echo "=== Storage $i ==="
    ssh minio-node-1 "ls /data/storage$i/bucket/"
done

# Проверка доступных дисков в erasure set
mc admin info myminio --json | jq '.info.servers[].drives[] | {endpoint, set_index, state}'
```

**Решение:**

```bash
# Если доступна реплика - проксирование активно по умолчанию
mc cp myminio/bucket/file.txt ./  # Получит из реплики

# Или запуск resync
REMOTE=$(mc replicate ls myminio/bucket --json | jq -r '.rule.Destination.Bucket')
mc replicate resync start myminio/bucket --remote-bucket "$REMOTE"
```

### Проблема: Балансировщик не распределяет нагрузку

**Диагностика:**

```bash
# Nginx - проверка upstream статуса
# Добавьте в конфигурацию zone для upstream:
upstream minio_api {
    zone minio_backend 64k;
    least_conn;
    server minio-node-1:9000;
    ...
}

# HAProxy - проверка backend
echo "show servers state minio_api" | socat stdio /run/haproxy/admin.sock

# Проверка health checks
# Nginx
curl http://minio-node-1:9000/minio/health/live

# Тестирование распределения
for i in {1..10}; do
    curl -k https://minio.example.com/minio/health/live -v 2>&1 | grep "Server:"
done
```

**Решение:**

```bash
# Nginx - проверьте health checks и fail_timeout
upstream minio_api {
    server minio-node-1:9000 max_fails=3 fail_timeout=30s;
    ...
}

# HAProxy - убедитесь что health checks работают
backend minio_api {
    option httpchk GET /minio/health/live
    http-check expect status 200
    ...
}

# Перезапуск балансировщика
sudo systemctl restart nginx  # или haproxy
```

### Проблема: VIP не переключается при отказе

**Диагностика:**

```bash
# Проверка статуса Keepalived
sudo systemctl status keepalived

# Проверка логов
sudo journalctl -u keepalived -f

# Проверка VRRP пакетов
sudo tcpdump -i eth0 vrrp

# Проверка приоритетов
sudo grep priority /etc/keepalived/keepalived.conf
```

**Решение:**

```bash
# Убедитесь что MASTER имеет выше priority
# MASTER: priority 100
# BACKUP: priority 90

# Проверьте firewall (должен пропускать VRRP - protocol 112)
sudo iptables -I INPUT -p vrrp -j ACCEPT

# Убедитесь что оба балансировщика в одной подсети
ip addr show eth0

# Перезапуск Keepalived
sudo systemctl restart keepalived
```

---

## Рекомендации по production окружению

### Архитектурные рекомендации

1. **Оптимальная конфигурация: 4×4**
   - 4 сервера по 4 диска
   - 1 erasure set (16 дисков)
   - EC:2 для баланса или EC:4 для максимальной надежности
   - Предсказуемое поведение при отказах

2. **Избегайте множественных erasure set в одном pool**
   - Создает неопределенность при отказах
   - Разные объекты имеют разную доступность
   - Усложняет capacity planning

3. **Используйте две географически разнесенные инсталляции**
   - ACTIVE-ACTIVE репликация
   - Защита от катастрофических отказов
   - Возможность обслуживания без downtime

4. **Выбор storage class:**
   - STANDARD EC:2 — минимум для production
   - STANDARD EC:4 — для критичных данных
   - RRS — только для временных/восстанавливаемых данных

5. **Обязательное использование балансировщиков:**
   - Два балансировщика с Keepalived (VRRP)
   - Health checks для автоматического исключения нод
   - SSL/TLS termination
   - Централизованная точка входа

### Сетевые рекомендации

1. **Dedicated сеть для MinIO кластера**
   - Минимум 10 Gbit/s между нодами
   - Низкая латентность (<1ms)
   - Изолированная от клиентского трафика

2. **Load Balancer обязателен**
   - Nginx/HAProxy/Angie
   - Health checks
   - SSL termination
   - Keepalived для HA

3. **DNS или Service Discovery**
   - Не использовать IP адреса напрямую
   - Упрощает замену нод
   - Совместимость с оркестраторами

4. **Оптимизация сети для балансировщиков:**
   - Jumbo frames (MTU 9000) между MinIO и LB
   - Dedicated сетевой интерфейс для backend трафика
   - Bonding/LACP для увеличения пропускной способности

### Безопасность

1. **TLS везде:**
   - Между клиентами и LB
   - Между LB и MinIO
   - Между MinIO нодами
   - Между деплойментами (репликация)

2. **Принцип наименьших привилегий:**
   - Отдельные пользователи для каждого приложения
   - Политики доступа per-bucket
   - Регулярный audit пользователей

3. **Сетевая изоляция:**
   - Firewall между клиентами и MinIO
   - Ограничение доступа к Console
   - VPN для административного доступа

4. **Версионирование и Object Lock:**
   - Защита от случайного удаления
   - Compliance требования
   - Retention policies

5. **Безопасность балансировщиков:**
   - Регулярное обновление Nginx/HAProxy
   - Rate limiting для защиты от DDoS
   - WAF (Web Application Firewall) при необходимости
   - Мониторинг подозрительной активности

### Операционные рекомендации

1. **Мониторинг:**
   - Prometheus + Grafana обязательны
   - Алерты на disk offline
   - Алерты на heal операции
   - Capacity planning метрики
   - Мониторинг балансировщиков (upstream health, response times)

2. **Backup стратегия:**
   - ACTIVE-ACTIVE репликация — первая линия
   - Периодические snapshot'ы через versioning
   - Mirror в cold storage для compliance

3. **Capacity Planning:**
   - Держите <70% использования
   - Планируйте расширение заранее
   - Учитывайте overhead parity дисков

4. **Документация:**
   - Runbook для типичных инцидентов
   - Список контактов дежурных
   - Схема сетевой топологии
   - Процедуры масштабирования
   - Документация балансировщиков и failover процедур

5. **Тестирование отказоустойчивости:**
   - Регулярно тестируйте failover сценарии
   - Симулируйте отказ дисков
   - Тренируйте процедуры восстановления
   - Проверяйте SLA на восстановление
   - Тестируйте переключение балансировщиков
   - Проверяйте работу при отказе целой ноды MinIO

### Производительность

1. **Дисковая подсистема:**
   - SSD/NVMe для высокой производительности
   - RAID не требуется (MinIO сам обеспечивает избыточность)
   - XFS файловая система (лучшая совместимость)
   - Не использовать LVM

2. **Сетевая оптимизация:**
   - Jumbo frames (MTU 9000) в dedicated сети
   - TCP tuning для high throughput
   - RSS/RPS для многоядерных систем

3. **Системные лимиты:**
   - ulimit -n 65536 (open files)
   - Достаточно RAM для caching
   - CPU зависит от количества concurrent requests

4. **Оптимизация балансировщиков:**
   - Используйте keepalive соединения
   - Настройте connection pooling
   - Отключите ненужную буферизацию
   - Тюнинг kernel parameters для high load

### Compliance и регуляторика

1. **Data Residency:**
   - Контроль расположения данных через server pools
   - Geo-репликация по требованию
   - Документирование data flow

2. **Audit Trail:**
   - Включение audit logging
   - Централизованный сбор логов
   - Retention согласно требованиям

3. **Encryption:**
   - Encryption at rest (через SSE-C или SSE-KMS)
   - Encryption in transit (TLS)
   - Key management strategy

---

## Заключение

MinIO — мощное и гибкое решение для организации S3-совместимого хранилища в собственной инфраструктуре. Правильная настройка с учетом рекомендаций данного руководства обеспечит:

- **Высокую доступность** — благодаря erasure coding, репликации и отказоустойчивым балансировщикам
- **Отказоустойчивость** — переживет отказ нескольких дисков, целых нод или балансировщиков
- **Масштабируемость** — через добавление server pools
- **Безопасность** — при правильной настройке TLS, политик доступа и сетевой изоляции

**Ключевые выводы:**

1. Используйте конфигурацию 4×4 с EC:2 или EC:4
2. Обязательно настройте балансировщики нагрузки с HA (Keepalived)
3. Настраивайте ACTIVE-ACTIVE репликацию между сайтами
4. Мониторьте здоровье кластера и балансировщиков постоянно
5. Документируйте процедуры и регулярно тестируйте failover
6. Планируйте capacity заранее
7. Используйте современные практики безопасности на всех уровнях

**Типовая production архитектура:**

```
┌─────────────────────────────────────────────────────────┐
│                      Internet/Clients                    │
└────────────────────────┬────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │   VIP   │ (Keepalived VRRP)
                    └────┬────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
     ┌────▼─────┐                 ┌─────▼────┐
     │ LB-1     │◄───Keepalived───►│  LB-2    │
     │ (MASTER) │                  │ (BACKUP) │
     └────┬─────┘                  └─────┬────┘
          │                              │
          └──────────┬───────────────────┘
                     │
       ┌─────────────┼─────────────┐
       │             │             │
  ┌────▼────┐   ┌────▼────┐   ┌───▼─────┐   ┌─────────┐
  │ MinIO-1 │   │ MinIO-2 │   │ MinIO-3 │   │ MinIO-4 │
  │ 4 disks │   │ 4 disks │   │ 4 disks │   │ 4 disks │
  └─────────┘   └─────────┘   └─────────┘   └─────────┘
       └──────────────┬──────────────┘
              Erasure Set (EC:4)
                      │
                      │ Replication
                      ▼
              ┌───────────────┐
              │   Site B      │
              │ (Mirror setup)│
              └───────────────┘
```

Удачи в эксплуатации вашего MinIO кластера!
