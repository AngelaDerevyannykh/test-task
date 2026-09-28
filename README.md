# Настройка HTTPS на Nginx с reverse proxy для backend-приложения

Данная инструкция позволяет настроить Nginx как reverse proxy: Nginx будет принимать HTTPS-запросы на порту `443` и передавать их backend-приложению на локальный порт `8080`.

```mermaid
flowchart LR
    client[Клиент] -->|HTTPS :443| nginx[Nginx]
    nginx -->|HTTP :8080| backend[Backend-приложение]
```

> Self-signed сертификат подходит для тестовой среды и внутренней проверки. Браузеры и HTTP-клиенты не доверяют ему по умолчанию. Для production используйте сертификат, выпущенный доверенным центром сертификации.

## Предварительные требования

Для настройки понадобятся:

- Linux-сервер и пользователь с правами `sudo`;
- DNS-имя сервера известно, например `app.example.local`, и разрешается в его IP-адрес;
- backend-приложение запущено на том же сервере и слушает `127.0.0.1:8080`;
- порты `80` и `443` не заняты другим веб-сервером;
- установлена утилита `openssl`.

Проверьте доступность backend-приложения:

```bash
curl --fail --verbose http://127.0.0.1:8080/
```

Команда должна получить HTTP-ответ приложения. Если соединение не устанавливается, сначала запустите или исправьте backend.

## 1. Установите Nginx

В Debian или Ubuntu выполните:

```bash
sudo apt update
sudo apt install nginx openssl
```

Запустите Nginx и включите его автоматический запуск после перезагрузки:

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

В выводе должна быть строка `Active: active (running)`.

[Screenshot: вывод systemctl status nginx с состоянием active (running)]

Если на сервере включён сетевой экран, разрешите HTTP- и HTTPS-трафик. Для UFW выполните:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

Для firewalld выполните:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Используйте команды только для того сетевого экрана, который установлен и включён в вашей системе.

## 2. Создайте self-signed сертификат

В примерах используется имя `app.example.local`. Замените его на DNS-имя своего сервера во всех командах и в конфигурации Nginx. Имя должно совпадать в `subjectAltName`, директиве `server_name`, именах файлов сертификата и URL проверки.

Создайте каталог для сертификата и закрытого ключа:

```bash
sudo install -d -m 0755 /etc/nginx/ssl
```

Сгенерируйте закрытый ключ и сертификат сроком на 365 дней:

```bash
sudo openssl req \
  -x509 \
  -nodes \
  -newkey rsa:2048 \
  -days 365 \
  -keyout /etc/nginx/ssl/app.example.local.key \
  -out /etc/nginx/ssl/app.example.local.crt \
  -subj "/CN=app.example.local" \
  -addext "subjectAltName=DNS:app.example.local"
```

Если вы обращаетесь к серверу по IP-адресу, добавьте его в расширение, например `-addext "subjectAltName=DNS:app.example.local,IP:192.0.2.10"`.

Ограничьте доступ к закрытому ключу и проверьте сертификат:

```bash
sudo chmod 600 /etc/nginx/ssl/app.example.local.key
sudo openssl x509 \
  -in /etc/nginx/ssl/app.example.local.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

[Screenshot: вывод openssl с CN, SAN и сроком действия сертификата]

## 3. Настройте server block с SSL

В стандартной конфигурации Nginx файл `/etc/nginx/nginx.conf` подключает конфигурационные фрагменты из `/etc/nginx/conf.d/`. Поэтому server block приложения удобно хранить отдельно от основного файла.

Создайте файл `/etc/nginx/conf.d/app.conf` со следующим содержимым:

```nginx
server {
    listen 80;
    server_name app.example.local;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.example.local;

    ssl_certificate     /etc/nginx/ssl/app.example.local.crt;
    ssl_certificate_key /etc/nginx/ssl/app.example.local.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> В `proxy_pass` нет завершающего `/`. Поэтому Nginx передаёт исходный URI без преобразования. Например, запрос `/api/health` уйдёт на `http://127.0.0.1:8080/api/health`.

Убедитесь, что Nginx подключает созданный файл:

```bash
sudo nginx -T | grep -F '/etc/nginx/conf.d/app.conf'
```

Команда должна вывести строку с именем файла.

### Параметры nginx.conf и server block

| Параметр | Значение в примере | Назначение |
|---|---|---|
| `listen` | `443 ssl` | Принимает HTTPS-соединения на TCP-порту 443 |
| `server_name` | `app.example.local` | Выбирает server block по имени из запроса |
| `ssl_certificate` | `/etc/nginx/ssl/app.example.local.crt` | Задаёт путь к сертификату сервера |
| `ssl_certificate_key` | `/etc/nginx/ssl/app.example.local.key` | Задаёт путь к закрытому ключу сертификата |
| `ssl_protocols` | `TLSv1.2 TLSv1.3` | Разрешает современные версии протокола TLS |
| `location` | `/` | Применяет proxy-настройки ко всем URI |
| `proxy_pass` | `http://127.0.0.1:8080` | Передаёт запросы локальному backend-приложению |
| `proxy_http_version` | `1.1` | Использует HTTP/1.1 при обращении к backend |
| `proxy_set_header Host` | `$host` | Передаёт backend-приложению исходное имя хоста |
| `proxy_set_header X-Real-IP` | `$remote_addr` | Передаёт IP-адрес клиента |
| `proxy_set_header X-Forwarded-For` | `$proxy_add_x_forwarded_for` | Добавляет IP-адрес клиента в цепочку proxy |
| `proxy_set_header X-Forwarded-Proto` | `$scheme` | Сообщает backend-приложению исходную схему `https` |

## 4. Проверьте и примените конфигурацию

Проверьте синтаксис и пути к сертификату:

```bash
sudo nginx -t
```

При успешной проверке Nginx выведет сообщения `syntax is ok` и `test is successful`.

[Screenshot: успешный результат команды nginx -t]

Примените изменения без остановки сервера:

```bash
sudo systemctl reload nginx
```

Убедитесь, что Nginx слушает порт `443`:

```bash
sudo ss -ltnp | grep ':443'
```

## 5. Проверьте HTTPS и reverse proxy

С самого сервера выполните запрос с именем хоста:

```bash
curl --verbose \
  --insecure \
  --resolve app.example.local:443:127.0.0.1 \
  https://app.example.local/
```

Параметр `--resolve` позволяет провести проверку до создания DNS-записи. Параметр `--insecure` нужен только потому, что сертификат self-signed.

Проверьте перенаправление с HTTP на HTTPS:

```bash
curl --head \
  --resolve app.example.local:80:127.0.0.1 \
  http://app.example.local/
```

Ожидаемый результат — статус `301` и заголовок `Location: https://app.example.local/`.

Затем откройте `https://app.example.local/` в браузере с клиентского компьютера. Браузер покажет предупреждение о недоверенном сертификате: для self-signed сертификата это ожидаемо. После подтверждения исключения должна открыться страница backend-приложения.

[Screenshot: backend-приложение, открытое по HTTPS через Nginx]
