# Hysteria2 на сервере

Документ описывает текущую рабочую схему Hysteria2 на сервере и как ее развернуть с нуля.

Документ основан на реальной конфигурации сервера:
- сервис `hysteria-server.service` активен;
- Hysteria2 слушает `UDP 8443`;
- используется парольная аутентификация;
- включен `obfs: salamander`;
- включен `masquerade` через `https://www.cloudflare.com`;
- TLS-сертификаты читаются из локальных файлов.

В папке `files/hysteria2` сохранены redacted-копии реальных файлов:
- `hysteria-server.service`
- `config.yaml.redacted`
- `client.example.yaml`

Секреты из них удалены: пароль auth, пароль obfs, приватные ключи и реальные TLS-данные.

## 1. Что развернуто сейчас

- Бинарник: `/usr/local/bin/hysteria`
- Версия: `v2.6.5`
- Server config: `/etc/hysteria/config.yaml`
- Systemd unit: `/etc/systemd/system/hysteria-server.service`
- Порт: `8443/udp`

Проверка:

```bash
systemctl is-active hysteria-server
ss -tulpn | grep 8443
/usr/local/bin/hysteria version
```

## 2. Текущий server config

По серверу сейчас используется такая логика:

- `listen: :8443`
- TLS:
  - `cert: /etc/hysteria/server.crt`
  - `key: /etc/hysteria/server.key`
  - `alpn: [h3, h2, http/1.1]`
- Auth:
  - `type: password`
  - один пароль для клиента
- Bandwidth:
  - `up: 100 mbps`
  - `down: 100 mbps`
- Obfuscation:
  - `type: salamander`
  - отдельный пароль obfs
- Masquerade:
  - `enabled: true`
  - `type: proxy`
  - `url: https://www.cloudflare.com`
  - `http3: true`
  - `mtu: 1200`

## 3. Установка Hysteria2

## 3.1. Подготовка

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates
sudo mkdir -p /etc/hysteria
```

## 3.2. Установка бинарника

Пример ручной установки:

```bash
sudo wget -O /usr/local/bin/hysteria https://github.com/apernet/hysteria/releases/latest/download/hysteria-linux-amd64
sudo chmod +x /usr/local/bin/hysteria
/usr/local/bin/hysteria version
```

Если ссылка на latest release изменилась, нужно взять актуальный `linux-amd64` binary со страницы релизов Hysteria2.

## 3.3. TLS-сертификаты

На текущем сервере Hysteria2 использует:

```text
/etc/hysteria/server.crt
/etc/hysteria/server.key
```

Можно использовать:
- отдельный сертификат для Hysteria2;
- или сертификат от Let's Encrypt, если удобно хранить/ссылаться на него напрямую.

Файл сертификата и ключ должны читаться пользователем/группой, под которыми запускается сервис.

## 4. Настройка сервера

Создать конфиг:

```bash
sudo nano /etc/hysteria/config.yaml
```

Шаблон на основе текущего сервера:

```yaml
listen: :8443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key
  alpn:
    - h3
    - h2
    - http/1.1

auth:
  type: password
  password: CHANGE_ME_HYSTERIA_PASSWORD

bandwidth:
  up: 100 mbps
  down: 100 mbps

obfs:
  type: salamander
  salamander:
    password: CHANGE_ME_OBFS_PASSWORD

masquerade:
  enabled: true
  type: proxy
  proxy:
    url: https://www.cloudflare.com
    http3: true
  mtu: 1200
```

Примечания:
- пароль из `auth` и пароль из `obfs` должны быть сильными и разными;
- `listen` у Hysteria2 для сервера в этом кейсе именно `UDP 8443`;
- `masquerade` помогает сделать трафик менее очевидным, но это не замена правильной сетевой настройке.

## 5. Systemd сервис

На текущем сервере используется такой unit:

```ini
[Unit]
Description=Hysteria Server Service (config.yaml)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server --config /etc/hysteria/config.yaml
WorkingDirectory=~
User=hysteria
Group=hysteria
Environment=HYSTERIA_LOG_LEVEL=info
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Если пользователя `hysteria` еще нет:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin hysteria
```

Создать unit:

```bash
sudo nano /etc/systemd/system/hysteria-server.service
sudo systemctl daemon-reload
sudo systemctl enable --now hysteria-server
```

## 6. Firewall

Для текущей схемы нужно открыть именно UDP:

```bash
sudo ufw allow 8443/udp
```

TCP 8443 для Hysteria2 не нужен, если не используется дополнительная схема поверх reverse proxy.

## 7. Проверка работы

## 7.1. Сервис и порт

```bash
systemctl status hysteria-server --no-pager
ss -tulpn | grep 8443
```

## 7.2. Логи

```bash
journalctl -u hysteria-server -n 100 --no-pager
journalctl -u hysteria-server -f
```

## 7.3. Проверка клиента

Сервер сам по себе на 8443/udp слушает, но полноценная валидация обычно делается клиентом Hysteria2.

Признаки, что сервер рабочий:
- сервис active;
- UDP 8443 слушается;
- в логах есть подключения клиентов;
- нет ошибок чтения сертификата и parsing config.

## 8. Клиентский конфиг

Пример клиентского конфига на основе серверной схемы:

```yaml
server: YOUR_DOMAIN_OR_IP:8443

auth: CHANGE_ME_HYSTERIA_PASSWORD

tls:
  sni: YOUR_DOMAIN
  insecure: false

obfs:
  type: salamander
  salamander:
    password: CHANGE_ME_OBFS_PASSWORD

transport:
  udp:
    hopInterval: 30s

socks5:
  listen: 127.0.0.1:1080
```

Что важно подставить:
- домен или IP сервера;
- auth password;
- obfs password;
- корректный `sni`, соответствующий сертификату.

Если используется self-signed сертификат, потребуется отдельная настройка доверия или временно `insecure: true` только для теста.

## 9. Обслуживание

Перезапуск после изменения конфига:

```bash
sudo systemctl restart hysteria-server
```

Бэкап перед изменениями:

```bash
cp /etc/hysteria/config.yaml /etc/hysteria/config.yaml.$(date +%F_%H-%M-%S).bak
```

## 10. Типовые проблемы

1. Клиенты не подключаются:
- проверить `8443/udp` в firewall;
- проверить соответствие `auth password`;
- проверить `obfs salamander password`;
- проверить сертификат и `sni`.

2. Сервис не стартует:
- проверить YAML синтаксис в `config.yaml`;
- проверить права на `server.crt` и `server.key`;
- проверить `journalctl -u hysteria-server`.

3. Есть таймауты или сбросы соединений:
- часть WARN в логах может быть связана уже с трафиком приложений клиентов, а не с самим handshaking Hysteria2;
- проверить MTU (`1200` сейчас задан в masquerade);
- проверить сеть и UDP-доступность у клиента.