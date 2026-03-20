# WARP на сервере (wireproxy, SOCKS5 :40000)

Документ описывает текущую рабочую схему WARP на сервере и как ее поднять с нуля.

## 1. Текущее состояние на сервере

- Сервис: `wireproxy.service`
- Бинарник: `/usr/bin/wireproxy`
- Конфиг: `/etc/wireguard/proxy.conf`
- Порт прокси: `40000/tcp`
- Тип прокси: SOCKS5 (логин/пароль в секции `[Socks5]`)
- Unit запускает:

```ini
ExecStart=/usr/bin/wireproxy -c /etc/wireguard/proxy.conf
```

Проверка:

```bash
systemctl is-active wireproxy
ss -tulpn | grep 40000
```

## 2. Установка wireproxy

## 2.1. Пакеты

```bash
sudo apt update
sudo apt -y install curl wget ca-certificates
```

## 2.2. Установка бинарника

Вариант A (через готовый скрипт warp-sh):

```bash
bash <(curl -fsSL https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh)
```

Вариант B (вручную, если нужен полный контроль):

```bash
sudo wget -O /usr/bin/wireproxy https://github.com/pufferffish/wireproxy/releases/latest/download/wireproxy_linux_amd64
sudo chmod +x /usr/bin/wireproxy
```

## 2.3. Регистрация WARP и получение рабочего профиля

Это обязательный этап.

Без регистрации WARP конфиг `proxy.conf` не заработает, даже если вручную сгенерировать обычные WireGuard-ключи.

Для рабочего профиля нужно получить:
- `PrivateKey` клиента;
- `PublicKey` peer Cloudflare;
- `Address` (IPv4 и/или IPv6);
- `Endpoint`;
- при необходимости дополнительные параметры профиля.

Практически это можно сделать тремя путями.

### Вариант A. Через warp-sh

Самый простой путь для сервера.

```bash
bash <(curl -fsSL https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh)
```

Что делает скрипт:
- регистрирует WARP-устройство;
- получает рабочие ключи и адреса;
- может сразу установить `wireproxy`;
- может подготовить WARP-конфиг/параметры профиля, которые затем используются для сборки `proxy.conf` под SOCKS5 на `:40000`.

### Вариант B. Через wgcf

Альтернатива для более ручной настройки.

Примерный поток:

```bash
wget https://github.com/ViRb3/wgcf/releases
# дальше скачать актуальный linux_amd64 binary со страницы releases
# и переименовать его в /usr/local/bin/wgcf
chmod +x wgcf
sudo mv wgcf /usr/local/bin/wgcf
wgcf register
wgcf generate
```

После этого в `wgcf-profile.conf` будут нужные параметры для секций `[Interface]` и `[Peer]`, которые можно перенести в `/etc/wireguard/proxy.conf`.

### Вариант C. Вручную через API

Теоретически возможно, но для документации и обычной эксплуатации не рекомендуется.

Причина: нужно самостоятельно повторять процедуру регистрации WARP-клиента через API Cloudflare и корректно собирать итоговый профиль. Для сервера это обычно избыточно.

### Важное уточнение

Не описывайте этот этап как обычную генерацию WireGuard-ключей через `wg genkey`.

Для WARP этого недостаточно: нужны именно параметры из зарегистрированного WARP-профиля, а не произвольная пара ключей.

## 2.4. Подготовка конфига

Создать директорию и конфиг:

```bash
sudo mkdir -p /etc/wireguard
sudo nano /etc/wireguard/proxy.conf
```

Шаблон `proxy.conf`:

```ini
[Interface]
Address = <WARP_IPV4_FROM_REGISTERED_PROFILE>
Address = <WARP_IPV6_FROM_REGISTERED_PROFILE>
MTU = 1280
PrivateKey = <YOUR_WARP_PRIVATE_KEY>
DNS = 1.1.1.1,8.8.8.8

[Peer]
PublicKey = <CLOUDFLARE_WARP_PUBLIC_KEY>
Endpoint = engage.cloudflareclient.com:2408

[Socks5]
BindAddress = 0.0.0.0:40000
Username = warpuser
Password = CHANGE_ME_STRONG_PASSWORD
```

Примечания:
- Ключи и адреса берутся из этапа регистрации WARP, описанного выше.
- Строки `Address`, `PrivateKey` и `PublicKey` нельзя копировать из примера как есть: нужно подставить значения из зарегистрированного WARP-профиля.
- Не используйте пароль из примеров, задайте свой.
- Для повышения безопасности можно ограничить bind: `127.0.0.1:40000` и использовать прокси только локально/через туннели.

## 3. Systemd сервис

Создать unit:

```bash
sudo nano /etc/systemd/system/wireproxy.service
```

Содержимое:

```ini
[Unit]
Description=WireProxy for WARP
After=network.target
Documentation=https://github.com/fscarmen/warp-sh
Documentation=https://github.com/pufferffish/wireproxy

[Service]
ExecStart=/usr/bin/wireproxy -c /etc/wireguard/proxy.conf
RemainAfterExit=yes
Restart=always

[Install]
WantedBy=multi-user.target
```

Применить:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wireproxy
```

## 4. Firewall

Если прокси нужен снаружи:

```bash
sudo ufw allow 40000/tcp
```

Если прокси нужен только локально на сервере, порт 40000 в firewall не открывать.

## 5. Проверка работы

## 5.1. Сервис и порт

```bash
systemctl status wireproxy --no-pager
ss -tulpn | grep 40000
```

## 5.2. Проверка выхода через WARP SOCKS5

```bash
curl --socks5-hostname warpuser:YOUR_PASSWORD@127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace
```

Обычно в ответе должен быть `warp=on`.

Примечание: не храните реальный пароль в shell history. Для регулярных проверок лучше использовать временную переменную окружения или отдельный клиентский конфиг.

## 5.3. Проверка публичного IP через прокси

```bash
curl --socks5-hostname warpuser:YOUR_PASSWORD@127.0.0.1:40000 https://api.ipify.org
```

IP должен отличаться от обычного IP сервера (без прокси).

## 6. Интеграция с Xray/3x-ui (если нужно)

WARP можно использовать как upstream proxy:
- в outbound типа SOCKS;
- адрес `127.0.0.1`, порт `40000`;
- с логином/паролем из `[Socks5]`.

Так можно отправлять часть трафика через WARP по правилам маршрутизации.

## 7. Обслуживание

Логи:

```bash
journalctl -u wireproxy -n 100 --no-pager
journalctl -u wireproxy -f
```

Перезапуск после изменения конфига:

```bash
sudo systemctl restart wireproxy
```

## 8. Типовые проблемы

1. Порт 40000 не слушается:
- проверить `systemctl status wireproxy`;
- проверить путь к конфигу в unit;
- проверить синтаксис `/etc/wireguard/proxy.conf`.

2. Прокси слушает, но интернета через него нет:
- проверить ключи `PrivateKey/PublicKey`;
- проверить `Endpoint = engage.cloudflareclient.com:2408`;
- проверить исходящий UDP у провайдера/фаервола.

3. Частые переподключения в логах:
- для WARP это может быть нормальным поведением keepalive/handshake;
- если есть потери, уменьшить нагрузку и проверить сеть/MTU.

## 9. Быстрый чек-лист

```bash
systemctl is-active wireproxy
ss -tulpn | grep 40000
curl --socks5-hostname warpuser:YOUR_PASSWORD@127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace | grep warp
```