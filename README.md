# Конфигурация приватного сервера: VPN и DNS

Важно понимать, что данная конфигурация предназначена для организации защищённого соединения и приватного DNS и может не обеспечивать достаточную безопасность или производительность для других целей. Она оптимизирована для конкретной серверной среды и может не подходить для других условий.

## Документация сервера

Сервер: `YOUR_SERVER_IP` (1 vCPU / 1 GB RAM)

Документация составлена на основе реальных конфигов сервера.

P.S. Большинство текста генерировалось ИИ, но с последующей правкой и дополнением реальными данными. Поэтому в некоторых местах может быть избыточная детализация или формулировки, характерные для ИИ. Тем не менее, вся информация проверена ( нуу +- ) и соответствует реальной конфигурации сервера.

Так же многие данные (пароли, ключи, UUID, ip сервера, собственные домены и т.д.) удалены из конфигов и заменены на заглушки. В репозиторий не сохраняются реальные секреты и готовые к использованию клиентские конфиги.

---

## Содержание

| Документ | Сервис | Порт / протокол |
|---|---|---|
| [DNS.md](DNS.md) | AdGuard Home — DoH, DoT, plain DNS | `55/tcp+udp`, `853/tls`, `443/https` |
| [Nginx.md](Nginx.md) | Nginx — reverse proxy для DoH и iOS-профиля | `80`, `443` |
| [Certificates.md](Certificates.md) | TLS-сертификаты — certbot, Let's Encrypt | — |
| [WARP.md](WARP.md) | Cloudflare WARP через wireproxy — SOCKS5 прокси | `40000/tcp` |
| [XRAY-3X-UI.md](XRAY-3X-UI.md) | 3x-ui панель + Xray VLESS+Reality | `7443/tcp`, `29886/tcp`, панель `28500` |
| [HYSTERIA2.md](HYSTERIA2.md) | Hysteria2 — UDP-прокси с obfs salamander | `8443/udp` |
| [SHADOWSOCKS.md](SHADOWSOCKS.md) | Shadowsocks (shadowsocks-rust) | `9443/tcp` |

---

## Архитектура

```
Клиент
  │
  ├─► 7443/29886 (VLESS+Reality) → Xray → geoip:ru → WARP → интернет
  │                                      └─► всё остальное → прямо в интернет
  │
  ├─► 8443/udp (Hysteria2 + salamander obfs) → прямо в интернет
  │
  ├─► 9443/tcp (Shadowsocks aes-256-gcm) → прямо в интернет
  │
  └─► dns.YOUR_DOMAIN/dns-query (DoH через Nginx → AdGuard Home)
```

WARP используется как outbound в Xray — российские IP (geoip:ru) проксируются через Cloudflare WARP, остальные — напрямую.

---

## Файлы конфигов

Папка `files/` содержит redacted-копии реальных файлов с сервера (пароли, ключи и UUID удалены):

```
files/
  dns/
    AdGuardHome.yaml          — конфиг AdGuard Home
    AdGuardHome.service       — systemd unit
    nginx.dns.conf            — nginx site для DoH
    ios_doh_profile.mobileconfig — профиль для iOS
    ai.txt                   — список хостов AI сервисов для правил в Yoga DNS (далеко не все, а те которыми я пользуюсь)
  warp/
    wireproxy.service         — systemd unit wireproxy
    proxy.conf.redacted       — WireGuard конфиг WARP (ключи удалены)
  xray/
    x-ui.service              — systemd unit 3x-ui
    config.json.redacted      — Xray config (UUID и ключи удалены)
    panel-settings.redacted.txt — настройки панели из БД
  hysteria2/
    hysteria-server.service   — systemd unit
    config.yaml.redacted      — серверный конфиг (пароль удалён)
    client.example.yaml       — пример клиентского конфига
  shadowsocks/
    shadowsocks.service       — systemd unit
    shadowsocks.json.redacted — серверный конфиг (пароль удалён)
    client.example.json       — пример клиентского конфига
```