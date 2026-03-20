# 1. Установить certbot

```bash
apt update
apt install snapd -y
snap install core
snap refresh core
snap install --classic certbot
ln -sf /snap/bin/certbot /usr/bin/certbot
certbot --version
```

# 2. Выпустить сертификат

```bash
certbot certonly --standalone -d YOUR_DOMAIN -d dns.YOUR_DOMAIN
```

Если есть nginx (установка описана в [Nginx.md](Nginx.md))
```bash
certbot --nginx -d YOUR_DOMAIN -d dns.YOUR_DOMAIN
```

# 3. Проверка автопродления

Должен быть таймер, который запускает `certbot renew` примерно раз в 12 часов.

```bash
systemctl list-timers | grep certbot
```
И проверить ручное продление:

```bash
certbot renew --dry-run
```

Автозапуск автопродления с перезапуском AdGuard Home после обновления сертификата:

```bash
crontab -e
0 3 * * * certbot renew --quiet --deploy-hook "systemctl restart AdGuardHome"
```