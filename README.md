# Налаштування динамічного HTTPS-середовища

# Гіпотеза: Можливо викорситовувати один ssl сертифікат для всіх саб саб доменів *.dev1.mordach.com 

## Інфраструктура

- **Сервер:** DigitalOcean Droplet (Ubuntu 24.04)
- **DNS:** Cloudflare
- **Вебсервер:** Nginx
- **SSL:** Let’s Encrypt (Wildcard, DNS-01 через Cloudflare)
- **Домен:** `dev1.mordach.com`
- **Ціль:** Автоматичний доступ до `https://[feature].dev1.mordach.com`

---

## Крок 1 — DNS в Cloudflare

Увійдіть у Cloudflare → Зона `mordach.com` → вкладка **DNS**.

Додайте запис:

| Type | Name    | Content (IP)        | Proxy status |
|------|---------|---------------------|--------------|
| A    | `*.dev1`| `<ваш Droplet IP>`  | DNS only 🔵   |

> 🔹 **Не ставте Proxied** — має бути **DNS only** для Let's Encrypt DNS-01.

---

## Крок 2 — Сертифікат Let's Encrypt wildcard

Встановіть Certbot через Snap:

```bash
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo snap set certbot trust-plugin-with-root=ok
sudo snap install certbot-dns-cloudflare
```

Створіть токен в Cloudflare:

- **Permissions:** Zone > DNS > Edit
- **Zone:** `mordach.com`

Збережіть його в:

```bash
sudo nano /etc/letsencrypt/cloudflare.ini
```

```ini
dns_cloudflare_api_token = your_token_here
```

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

Отримайте сертифікат:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.dev1.mordach.com" \
  -d dev1.mordach.com \
  --agree-tos \
  --email your-email@example.com \
  --non-interactive
```

Перевірте сертифікати:

```bash
sudo certbot certificates
```

---

## Крок 3 — Налаштування Nginx

Створіть конфіг:

```bash
sudo nano /etc/nginx/sites-available/dev1.mordach.com
```

Вставте:

```nginx
map $host $app_dir {
    default "";
    ~^(?<branch>.+)\.dev1\.mordach\.com$ /var/www/html/$branch;
}

server {
    listen 443 ssl;
    server_name ~^(?<branch>.+)\.dev1\.mordach\.com$;

    ssl_certificate     /etc/letsencrypt/live/dev1.mordach.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dev1.mordach.com/privkey.pem;

    root $app_dir/public;
    index index.html index.php;

    access_log /var/log/nginx/dev1_access.log;
    error_log  /var/log/nginx/dev1_error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $app_dir/public$fastcgi_script_name;
        include fastcgi_params;
    }
}

server {
    listen 80;
    server_name ~^(?<branch>.+)\.dev1\.mordach\.com$;
    return 301 https://$host$request_uri;
}
```

Активуйте:

```bash
sudo ln -s /etc/nginx/sites-available/dev1.mordach.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## Крок 4 — Створення тестових фолдерів

```bash
cd /var/www/html

for i in $(seq -w 1 100); do
    folder="uiv4-$i"
    mkdir -p "$folder/public"
    echo "<h1>$folder</h1>" > "$folder/public/index.html"
done
```

---

## Крок 5 — Автоматичне оновлення сертифіката

```bash
sudo crontab -e
```

```cron
0 3 * * * /snap/bin/certbot renew --quiet
```

---

## Готово!

Тепер можна відкривати:

- https://uiv4-001.dev1.mordach.com
- https://uiv4-002.dev1.mordach.com
- ...
- https://uiv4-100.dev1.mordach.com
