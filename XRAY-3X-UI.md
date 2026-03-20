# Xray + 3x-ui на сервере

Документ описывает текущую рабочую схему 3x-ui/Xray на сервере и как ее развернуть с нуля.

Основа взята из реальной конфигурации сервера:
- панель 3x-ui запущена как systemd-сервис;
- Xray запускается 3x-ui и читает runtime-конфиг из `bin/config.json`;
- есть два inbound VLESS + Reality;
- есть routing через WARP SOCKS5 на `127.0.0.1:40000`.

В папке `files/xray` сохранены redacted-копии актуальных файлов:
- `x-ui.service`
- `config.json.redacted`
- `panel-settings.redacted.txt`

Все чувствительные данные удалены: UUID клиентов, private key Reality, WARP-пароль, base path панели и секреты.

## 1. Что развернуто сейчас

- 3x-ui service: `x-ui.service`
- Бинарник панели: `/usr/local/x-ui/x-ui`
- Runtime-конфиг Xray: `/usr/local/x-ui/bin/config.json`
- База панели: `/etc/x-ui/x-ui.db`
- Встроенный Xray binary: `/usr/local/x-ui/bin/xray-linux-amd64`

Текущие порты:
- `28500/tcp` - web-панель 3x-ui
- `7443/tcp` - inbound VLESS + Reality
- `29886/tcp` - второй inbound VLESS + Reality
- `127.0.0.1:62789` - internal API inbound
- `127.0.0.1:40000` - upstream SOCKS5 через WARP

443 не использовал т.к. на этом порту наблюдались конфликты и нестабильность соединения.

## 2. Установка 3x-ui

## 2.1. Установка панели

Обычно используется официальный install-script проекта 3x-ui.

Типовой сценарий:

```bash
apt update
apt install -y curl wget sudo
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

После установки проверить сервис:

```bash
systemctl status x-ui --no-pager
systemctl enable x-ui
```

На текущем сервере unit выглядит так:

```ini
[Unit]
Description=x-ui Service
After=network.target
Wants=network.target

[Service]
Environment="XRAY_VMESS_AEAD_FORCED=false"
Type=simple
WorkingDirectory=/usr/local/x-ui/
ExecStart=/usr/local/x-ui/x-ui
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

## 2.2. Где лежат настройки

- Общие настройки панели: `/etc/x-ui/x-ui.db`
- Runtime-конфиг Xray: `/usr/local/x-ui/bin/config.json`

Важно:
- `x-ui.db` содержит чувствительные данные панели, inbound-ов, клиентов и WARP;
- `config.json` содержит runtime-состояние Xray после применения правил из панели;
- в репозиторий сохранять только redacted-версии.

## 3. Настройка web-панели 3x-ui

На текущем сервере:
- web port: `28500`
- webBasePath задан, но в документации и файлах он скрыт

Проверка:

```bash
ss -tulpn | grep 28500
systemctl is-active x-ui
```

Рекомендации:
- не держать панель на дефолтном пути;
- ограничить доступ firewall-ом или reverse proxy;
- хранить отдельный сильный пароль панели.

## 4. Inbound VLESS + Reality

На сервере сейчас два inbound-а.

## 4.1. Inbound 1

- Protocol: `vless`
- Port: `7443`
- Transport: `tcp`
- Security: `reality`
- Flow клиентов: `xtls-rprx-vision`
- Tag: `inbound-443`
- Маскировочный destination: `www.microsoft.com:443`
- ServerName: `www.microsoft.com`

[screenshot](vless_inbound.png)

## 4.2. Inbound 2

Просто клон первого inbound-а, но на другом порту и с другим маскировочным destination:

- Protocol: `vless`
- Port: `29886`
- Transport: `tcp`
- Security: `reality`
- Flow клиентов: `xtls-rprx-vision`
- Tag: `inbound-29886`
- Маскировочный destination: `www.google.com:443`
- ServerName: `www.google.com`

## 4.3. Как создать такой inbound в 3x-ui

В панели 3x-ui при создании inbound:

- Protocol: `VLESS`
- Listen IP: пусто или `0.0.0.0`
- Port: `7443` или `29886`
- Transport: `tcp`
- Security: `reality`
- Flow: `xtls-rprx-vision`
- Sniffing: можно оставить как в текущей конфигурации

Reality settings:
- Dest: например `www.microsoft.com:443` или `www.google.com:443`
- Server Names: тот же домен, что и в `dest`
- Private Key: сгенерировать в панели/через Xray
- Short IDs: сгенерировать несколько значений

Клиенты:
- для каждого клиента создается отдельный UUID;
- в документацию UUID и shortId не выгружать;
- expiry/лимиты задаются по необходимости.

Не забывайте клиенту указывать в `Flow` - `xtls-rprx-vision`, иначе не будет работать.

## 5. Routing через WARP

(В меню Xray configs)

На текущем сервере routing настроен так:

### 5.1. Outbound `warp-socks5`

Это outbound типа `socks`, который подключается к локальному WARP proxy:

- address: `127.0.0.1`
- port: `40000`
- auth: логин/пароль из `/etc/wireguard/proxy.conf`

[screenshot](warp-socks5_outbound.png)

### 5.2. Outbound `warp`

Это outbound типа `freedom`, который использует `dialerProxy = warp-socks5`:

```json
{
  "tag": "warp",
  "protocol": "freedom",
  "settings": {
    "domainStrategy": "UseIPv4v6"
  },
  "streamSettings": {
    "sockopt": {
      "dialerProxy": "warp-socks5"
    }
  }
}
```

[screenshot](warp_outbound.png)

### 5.3. Rule для routing

В текущем runtime-конфиге есть правило:

```json
{
  "type": "field",
  "ip": ["geoip:ru"],
  "outboundTag": "warp"
}
```

Это означает: трафик к IP из `geoip:ru` уходит через WARP.

Еще можно попробовать - `geosite:category-gov-ru,regexp:.*\.ru$,regexp:.*\.su$`

Важно: это именно IP-based routing, а не routing по доменам AI-сервисов.

Если нужен routing по доменам, обычно добавляют правила уровня `domain`, например:

```json
{
  "type": "field",
  "domain": [
    "domain:openai.com",
    "domain:chatgpt.com",
    "domain:oaistatic.com"
  ],
  "outboundTag": "warp"
}
```

[screenshot](warp_routing_rule.png)

## 6. Пример runtime-конфига Xray

Фактическая структура runtime-конфига сохранена в:

- `files/xray/config.json.redacted`

Там уже есть:
- API inbound;
- два inbound VLESS + Reality;
- `warp-socks5` outbound;
- `warp` outbound через `dialerProxy`;
- routing rules.

## 7. Проверка работы

## 7.1. Проверка сервисов

```bash
systemctl is-active x-ui
systemctl status x-ui --no-pager
```

## 7.2. Проверка портов

```bash
ss -tulpn | grep -E '28500|7443|29886|62789'
```

## 7.3. Проверка runtime-конфига

```bash
cat /usr/local/x-ui/bin/config.json | jq '.routing, .inbounds, .outbounds'
```

## 7.4. Проверка WARP route

Проверить отдельно, что локальный WARP proxy работает:

```bash
curl --socks5-hostname warpuser:YOUR_PASSWORD@127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace
```

Затем убедиться, что Xray использует outbound `warp` в нужных правилах.

## 8. Обслуживание

Логи панели:

```bash
journalctl -u x-ui -n 100 --no-pager
journalctl -u x-ui -f
```

Бэкап перед изменениями:

```bash
cp /etc/x-ui/x-ui.db /etc/x-ui/x-ui.db.$(date +%F_%H-%M-%S).bak
cp /usr/local/x-ui/bin/config.json /usr/local/x-ui/bin/config.json.$(date +%F_%H-%M-%S).bak
```

## 9. Типовые проблемы

1. Панель не открывается:
- проверить `webPort`;
- проверить firewall;
- проверить `systemctl status x-ui`.

2. Inbound поднят, но клиенты не подключаются:
- проверить порт;
- проверить UUID клиента;
- проверить Reality `dest`, `serverNames`, `privateKey`, `shortIds`;
- проверить, что клиент использует `xtls-rprx-vision`.

3. Routing через WARP не работает:
- проверить доступность `127.0.0.1:40000`;
- проверить логин/пароль SOCKS5;
- проверить наличие outbound `warp-socks5` и `warp`;
- проверить правила в секции `routing`.