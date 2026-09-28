# Настройка HTTPS на Nginx с reverse proxy для backend-приложения

Данная инструкция позволяет настроить Nginx как reverse proxy: Nginx будет принимать HTTPS-запросы на порту `443` и передавать их backend-приложению на локальный порт `8080`.

```mermaid
flowchart LR
    client[Клиент] -->|HTTPS :443| nginx[Nginx]
    nginx -->|HTTP :8080| backend[Backend-приложение]
```

> Self-signed сертификат подходит для тестовой среды. Для production используйте сертификат, выпущенный доверенным центром сертификации.

## Требования

Для настройки понадобятся:

- Linux-сервер и пользователь с правами `sudo`;
- DNS-имя сервера, например `app.example.local`;
- backend-приложение, доступное на `127.0.0.1:8080`;
- свободные порты `80` и `443`.

Проверьте backend-приложение:

```bash
curl http://127.0.0.1:8080/
```

## 1. Установите Nginx

В Debian или Ubuntu выполните:

```bash
sudo apt update
sudo apt install nginx openssl
```

Запустите Nginx:

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

## 2. Создайте self-signed сертификат

Создайте каталог для сертификата и закрытого ключа:

```bash
sudo install -d -m 0755 /etc/nginx/ssl
```

Сгенерируйте сертификат для `app.example.local`:

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

Ограничьте доступ к закрытому ключу:

```bash
sudo chmod 600 /etc/nginx/ssl/app.example.local.key
```

## 3. Настройте Nginx

Создайте файл `/etc/nginx/conf.d/app.conf`:

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
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 4. Проверьте конфигурацию

Проверьте синтаксис и примените изменения:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Выполните тестовый HTTPS-запрос:

```bash
curl --insecure \
  --resolve app.example.local:443:127.0.0.1 \
  https://app.example.local/
```

Параметр `--insecure` разрешает `curl` подключиться с self-signed сертификатом.
