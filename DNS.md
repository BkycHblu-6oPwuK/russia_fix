# DNS сервер (AdGuard Home + DoH/DoT) и настройка клиентов

Документ фиксирует рабочую схему из текущего сервера и дает пошаговую инструкцию:
- как развернуть DNS-сервер;
- как применить конфиги;
- как подключить клиентов (iOS/Android/Windows/macOS/Linux);
- как проверять и обслуживать систему.

P.S. Замените `YOUR_DOMAIN` и `YOUR_SERVER_IP` на реальные значения в конфигурациях и инструкциях.

## 1. Что развернуто сейчас

- Сервер: 1 vCPU / 1 GB RAM.
- DNS-движок: AdGuard Home.
- Публикация DNS:
	- DoH через Nginx: `https://dns.YOUR_DOMAIN/dns-query`
	- DoT напрямую AdGuard: `dns.YOUR_DOMAIN:853`
	- Plain DNS на сервере слушается на `:55` (а не на 53).
- Upstream (в рабочем стабильном варианте):
	- `https://5u35p8m9i7.cloudflare-gateway.com/dns-query`
- iOS-профиль: `https://dns.YOUR_DOMAIN/ios-doh`

Важно: при добавлении нескольких fallback upstream ранее возникала нестабильность. Для стабильной работы использован один upstream.

## 2. Файлы из сервера (сохранены в этом каталоге)

В папке `files/dns` уже выгружены:

- `AdGuardHome.yaml` - текущий рабочий конфиг.
- `AdGuardHome.service` - systemd unit.
- `nginx.dns.conf` - актуальный nginx-конфиг для домена DNS.
- `ios_doh_profile.mobileconfig` - профиль для iOS.

## 3. Развертывание DNS-сервера с нуля

## 3.1. Подготовка сервера

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl wget ca-certificates nginx dnsutils ufw
```

Рекомендуется (для 1 GB RAM):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 3.2. Установка AdGuard Home

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | bash
systemctl start AdGuardHome
```

Открываешь в браузере: `http://YOUR_SERVER_IP:3000`

Проходишь этапы установки:
Web UI: 0.0.0.0:3005
DNS: 0.0.0.0:53 (у меня был занят, поставил 55)
Создаёшь логин/пароль

При установке я выбирал локальное размещение, чисто - lo (127.0.0.1), а далее настраивал nginx для публикации DoH. Это позволяет не открывать plain DNS наружу, а использовать только DoH/DoT.

После установки подменить конфиг на подготовленный:

```bash
sudo cp /path/to/russia_fix/files/AdGuardHome.yaml /opt/AdGuardHome/AdGuardHome.yaml
sudo cp /path/to/russia_fix/files/AdGuardHome.service /etc/systemd/system/AdGuardHome.service
sudo systemctl daemon-reload
sudo systemctl enable --now AdGuardHome
```
## 3.2.1. Сертификаты

Выпуск сертификата для `dns.YOUR_DOMAIN` (можно через certbot, как описано в [Certificates.md](Certificates.md)).

После выпуска сертификата указать пути к `fullchain.pem` и `privkey.pem` в `AdGuardHome.yaml` (параметры `certificate_path` и `private_key_path`), перезапустить сервис и проверить, что DoH работает. Проверьте `server_name` в `AdGuardHome.yaml`.

## 3.3. Nginx для DoH и iOS-профиля

Скопировать конфиг:

```bash
sudo cp /path/to/russia_fix/files/nginx.dns.conf /etc/nginx/sites-available/dns
sudo ln -sf /etc/nginx/sites-available/dns /etc/nginx/sites-enabled/dns
sudo nginx -t
sudo systemctl reload nginx
```

Скопировать iOS профиль:

```bash
sudo cp /path/to/russia_fix/files/ios_doh_profile.mobileconfig /opt/AdGuardHome/ios_doh_profile.mobileconfig
sudo chown root:root /opt/AdGuardHome/ios_doh_profile.mobileconfig
sudo chmod 644 /opt/AdGuardHome/ios_doh_profile.mobileconfig
```

## 3.4. TLS сертификат

Нужен валидный сертификат для `dns.YOUR_DOMAIN`.

Варианты:
- certbot через snap (как сейчас на сервере),
- certbot через apt.

Проверить итоговый сертификат:

```bash
openssl s_client -connect dns.YOUR_DOMAIN:443 -servername dns.YOUR_DOMAIN </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

## 3.5. Firewall

Открыть только нужное:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 55/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 853/tcp
sudo ufw allow 853/udp
sudo ufw enable
sudo ufw status verbose
```

Порт 53 можно не открывать, если используете только DoH/DoT. Я поставил 55 т.к. 53 был занят.

## 4. Проверка работоспособности

## 4.1. На сервере

```bash
systemctl is-active AdGuardHome nginx
dig @127.0.0.1 -p 55 chatgpt.com +short
dig +https @dns.YOUR_DOMAIN chatgpt.com +short
```

Ожидаемо для рабочего smart-DNS сценария: ответы вида `45.95.233.23` / `185.246.223.127`.

## 4.2. Снаружи

```bash
dig +https @dns.YOUR_DOMAIN chatgpt.com +short
```

Если plain DNS на 53 закрыт, `dig @dns.YOUR_DOMAIN ...` без `+https` будет выдавать ошибку подключения - это нормально.

## 5. Настройка клиентов

## 5.1. iOS/iPadOS

Вариант 1 (самый удобный):
- открыть в Safari: `https://dns.YOUR_DOMAIN/ios-doh`
- скачать профиль и установить его в Settings.

Вариант 2 (вручную):
- установить DNS-профиль/приложение, поддерживающее DoH;
- URL DoH: `https://dns.YOUR_DOMAIN/dns-query`.

## 5.2. Android 9+

В системных настройках Private DNS:
- выбрать режим "Private DNS provider hostname";
- указать: `dns.YOUR_DOMAIN`.

Это DoT (порт 853).

## 5.3. Windows

Вариант 1 (предпочтительно): использовать AdGuard/Firefox/Chrome с DoH:
- сервер DoH: `https://dns.YOUR_DOMAIN/dns-query`.

Вариант 2: Windows 11 DNS over HTTPS в настройках адаптера:
- указать DoH URL вручную, если редакция/политики это позволяют.

## 5.4. macOS

Варианты:
- через профиль/клиент с DoH: `https://dns.YOUR_DOMAIN/dns-query`;
- через apps (AdGuard/NextDNS profile tools и т.п.).

## 5.5. Linux

Пример через `cloudflared`/`dnscrypt-proxy` как локальный DoH-клиент,
или напрямую в приложениях, поддерживающих DoH.

Тест:

```bash
dig +https @dns.YOUR_DOMAIN chatgpt.com +short
```

## 6. Обслуживание и мониторинг

Проверка сервисов:

```bash
systemctl status AdGuardHome nginx --no-pager
journalctl -u AdGuardHome -n 100 --no-pager
```

Контроль аптайма DNS:

```bash
for i in $(seq 1 20); do dig @127.0.0.1 -p 55 chatgpt.com +short | tr '\n' ' '; echo; sleep 1; done
```

Бэкап рабочего конфига:

```bash
cp /opt/AdGuardHome/AdGuardHome.yaml /opt/AdGuardHome/AdGuardHome.yaml.$(date +%F_%H-%M-%S).bak
```

## 7. Типовые проблемы

1. `dig @dns.YOUR_DOMAIN` не отвечает:
- если порт 53 закрыт, это ожидаемо;
- проверять DoH (`dig +https @dns.YOUR_DOMAIN ...`).

2. В ответе снова "обычные" IP вместо proxy-IP:
- проверить `upstream_dns` в `AdGuardHome.yaml`;
- убедиться, что не добавлены лишние fallback upstream;
- перезапустить AdGuardHome.

3. iOS профиль не скачивается:
- проверить location `/ios-doh` в nginx;
- проверить наличие файла `/opt/AdGuardHome/ios_doh_profile.mobileconfig`;
- проверить сертификат на домене.

## 8. Быстрый чек-лист после изменений

```bash
nginx -t && systemctl reload nginx
systemctl restart AdGuardHome
systemctl is-active AdGuardHome nginx
dig @127.0.0.1 -p 55 chatgpt.com +short
dig +https @dns.YOUR_DOMAIN chatgpt.com +short
```

Если все команды успешны и приходят proxy-IP, конфигурация применена корректно.

## 9. Как добавить DNS в windows

Я использую программу Yoga DNS, которая позволяет легко переключаться между разными DNS-серверами и поддерживает DoH. В ней нужно добавить новый профиль с URL `https://dns.YOUR_DOMAIN/dns-query` и выбрать его в качестве активного.

Можно использовать правила - hostnames, для каких адресов будет действовать этот DNS. Например, для всех адресов или только для определенных доменов.

Пример списка с хостами AI сервисов (chat.gpt, claude, copilot) [ai.txt](files/dns/ai.txt), который можно использовать в правилах.