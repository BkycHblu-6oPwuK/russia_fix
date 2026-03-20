# 1. Установка Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

Проверка работоспособности:

```bash
systemctl status nginx
```

Если не запущен, то:

```bash
sudo systemctl start nginx
```

Автозапуск при загрузке:

```bash
sudo systemctl enable nginx
```