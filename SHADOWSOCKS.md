# Shadowsocks на сервере

Документ описывает текущую рабочую схему Shadowsocks на сервере и как ее развернуть с нуля.

Документ основан на реальной конфигурации сервера:
- сервис `shadowsocks.service` активен;
- сервер слушает `9443/tcp`;
- используется `ssserver`;
- конфиг лежит в `/etc/shadowsocks.json`;
- шифрование: `aes-256-gcm`.

В папке `files/shadowsocks` сохранены redacted-копии реальных файлов:
- `shadowsocks.service`
- `shadowsocks.json.redacted`
- `client.example.json`

Пароль из них удален намеренно.

## 1. Что развернуто сейчас

- Бинарник: `/usr/local/bin/ssserver`
- Версия: `shadowsocks 1.23.5`
- Systemd unit: `/etc/systemd/system/shadowsocks.service`
- Конфиг: `/etc/shadowsocks.json`
- Порт: `9443/tcp`

Текущий server config:

```json
{
  "server": "0.0.0.0",
  "server_port": 9443,
  "password": "<REDACTED>",
  "timeout": 300,
  "method": "aes-256-gcm"
}
```

Проверка:

```bash
systemctl is-active shadowsocks
ss -tulpn | grep 9443
/usr/local/bin/ssserver --version
```

## 2. Установка Shadowsocks

На текущем сервере используется standalone binary `ssserver`, а не пакет `shadowsocks-libev`.

## 2.1. Подготовка

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates
```

## 2.2. Установка бинарника

Релизы shadowsocks-rust включают версию в имени архива, поэтому скачивать нужно через GitHub API или вручную.

Автоматическая установка последней версии:

```bash
LATEST=$(curl -s https://api.github.com/repos/shadowsocks/shadowsocks-rust/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
ARCHIVE="shadowsocks-${LATEST}.x86_64-unknown-linux-gnu.tar.gz"
curl -L "https://github.com/shadowsocks/shadowsocks-rust/releases/download/${LATEST}/${ARCHIVE}" -o /tmp/ss.tar.gz
tar -xzf /tmp/ss.tar.gz -C /tmp ssserver
sudo mv /tmp/ssserver /usr/local/bin/ssserver
sudo chmod +x /usr/local/bin/ssserver
/usr/local/bin/ssserver --version
```

Или вручную: зайти на https://github.com/shadowsocks/shadowsocks-rust/releases/latest, скачать архив `shadowsocks-vX.X.X.x86_64-unknown-linux-gnu.tar.gz`, извлечь `ssserver` и положить в `/usr/local/bin/`.

## 3. Настройка сервера

Создать конфиг:

```bash
sudo nano /etc/shadowsocks.json
```

Шаблон на основе текущего сервера:

```json
{
  "server": "0.0.0.0",
  "server_port": 9443,
  "password": "CHANGE_ME_STRONG_PASSWORD",
  "timeout": 300,
  "method": "aes-256-gcm"
}
```

Примечания:
- пароль должен быть длинным и случайным;
- `aes-256-gcm` сейчас совпадает с реальной конфигурацией сервера;
- в текущей схеме не используется ни plugin, ни TLS-обертка, ни obfs.

## 4. Systemd сервис

На сервере используется такой unit:

```ini
[Unit]
Description=Shadowsocks Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks.json
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Создать unit:

```bash
sudo nano /etc/systemd/system/shadowsocks.service
sudo systemctl daemon-reload
sudo systemctl enable --now shadowsocks
```

## 5. Firewall

Для текущей схемы открыт TCP-порт:

```bash
sudo ufw allow 9443/tcp
```

В этой конфигурации UDP для Shadowsocks не используется.

## 6. Клиентский конфиг

Пример client config:

```json
{
  "server": "YOUR_SERVER_IP_OR_DOMAIN",
  "server_port": 9443,
  "password": "CHANGE_ME_STRONG_PASSWORD",
  "method": "aes-256-gcm",
  "timeout": 300,
  "local_address": "127.0.0.1",
  "local_port": 1080
}
```

Что должно совпадать у клиента и сервера:
- `server_port`
- `password`
- `method`

## 7. Проверка работы

## 7.1. Сервис и порт

```bash
systemctl status shadowsocks --no-pager
ss -tulpn | grep 9443
```

## 7.2. Логи

```bash
journalctl -u shadowsocks -n 100 --no-pager
journalctl -u shadowsocks -f
```

## 7.3. Признаки работы

- сервис active;
- `9443/tcp` слушается;
- клиент подключается без ошибок `decrypt length failed`;
- в логах нет массовых ошибок handshake для вашего клиента.

## 8. Типовые проблемы

1. `decrypt length failed` в логах:
- обычно это неверный пароль или неверный `method` у клиента;
- также это могут быть случайные сканеры интернета на открытом порту.

2. Подключение есть, но часть сайтов не открывается:
- на сервере в логах уже видны попытки выхода на IPv6 с ошибкой `No route to host`;
- если у сервера нет рабочего IPv6, часть запросов может упираться в это;
- в таком случае нужно либо настроить IPv6, либо заставить клиентов/маршрутизацию использовать IPv4.

3. Порт не слушается:
- проверить `/etc/shadowsocks.json`;
- проверить `systemctl status shadowsocks`;
- проверить наличие `/usr/local/bin/ssserver`.

## 9. Обслуживание

Перезапуск после изменения конфига:

```bash
sudo systemctl restart shadowsocks
```

Бэкап:

```bash
cp /etc/shadowsocks.json /etc/shadowsocks.json.$(date +%F_%H-%M-%S).bak
```