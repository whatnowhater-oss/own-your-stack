---
title: "Self-host Invoice Ninja: free invoicing instead of FreshBooks"
description: "Send invoices, take payments and track what clients owe with Invoice Ninja on your own server. A tested Docker setup with MySQL, Redis and PDF generation."
last_checked: 2026-11-01
tested_on: "Invoice Ninja 5.13.52 (invoiceninja-debian image), MySQL 8, Redis 7 and nginx behind Caddy in Docker Compose (admin login, client, invoice, PDF, client portal link and mark-as-paid tested; data survived a full restart, Nov 1 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Invoicing apps charge by the month and by the client. FreshBooks, for example, listed plans from about $23 to $70 a month in late 2026, with a 5-client limit on the cheapest plan and extra fees per team member ([their pricing page](https://www.freshbooks.com/pricing) has today's numbers). [Invoice Ninja](https://invoiceninja.com) is open-source invoicing you can run yourself: clients, quotes, invoices, recurring invoices, online payments, expenses and a portal where clients view and pay what they owe.

**Is self-hosting it worth it?** Be honest about the alternatives:

- Invoice Ninja's own hosted service has a **free plan for up to 5 clients**, and its Pro plan was $14 a month (or $140 a year) when we checked [their pricing](https://invoiceninja.com/pricing-plans/). If that's enough for you, the hosted version is less work.
- Self-hosting makes sense when you have more clients, want no per-client or per-user limits, or want your financial records on a server you control.
- Invoices are business records. Self-hosting means **you** are responsible for backups. Set up [nightly backups](./08-backups.md) before you send your first real invoice.

**You need:** a server with Docker ([30-minute setup guide](./01-the-7-dollar-server.md)), the [one-Caddy setup](./07-caddy.md) with its `web` network, and a DNS A record such as `invoices` pointing at the server.

**Memory:** in our test, Invoice Ninja used about 540 MB and its MySQL database about 500 MB, so about 1 GB together. Use a 2 GB server if it runs alone, or 4 GB alongside other apps. See [Which VPS?](./09-which-vps.md) for prices.

Time: about 20 minutes. Extra cost: $0.

## 1. Folder and settings

```bash
mkdir -p ~/apps/invoiceninja/nginx && cd ~/apps/invoiceninja
```

Download the Invoice Ninja image first (about 1 GB, so it takes a minute or two):

```bash
docker pull invoiceninja/invoiceninja-debian:5
```

Now create the settings file. Replace the three `yourbusiness` values first. The `docker run` line inside uses the image to generate Invoice Ninja's encryption key.

```bash
cat > .env <<EOF
APP_URL=https://invoices.yourbusiness.com
APP_KEY=$(docker run --rm invoiceninja/invoiceninja-debian:5 php artisan key:generate --show)
APP_ENV=production
APP_DEBUG=false
REQUIRE_HTTPS=true
TRUSTED_PROXIES='*'
PDF_GENERATOR=snappdf
IS_DOCKER=true
SCOUT_DRIVER=null
FILESYSTEM_DISK=debian_docker

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=ninja
DB_USERNAME=ninja
DB_PASSWORD=$(openssl rand -hex 20)
DB_ROOT_PASSWORD=$(openssl rand -hex 20)

IN_USER_EMAIL=you@yourbusiness.com
IN_PASSWORD=$(openssl rand -base64 18)

MAIL_MAILER=log
MAIL_FROM_ADDRESS=invoices@yourbusiness.com
MAIL_FROM_NAME="Your Business"
EOF
chmod 600 .env
grep -E 'APP_KEY|IN_PASSWORD' .env
```

Check that `APP_KEY` starts with `base64:`. The last line shows your first admin password. **Save it in your password manager now.** `IN_USER_EMAIL` and `IN_PASSWORD` create your admin login on the first start.

## 2. The web server settings

Invoice Ninja runs PHP behind a small nginx web server. Download the two nginx files from Invoice Ninja's official Docker repository:

```bash
B=https://raw.githubusercontent.com/invoiceninja/dockerfiles/debian/debian/nginx
curl -fsSL $B/laravel.conf -o nginx/laravel.conf
curl -fsSL $B/invoiceninja.conf -o nginx/invoiceninja.conf
```

## 3. The compose file

`compose.yaml`:

```yaml
services:
  app:
    image: invoiceninja/invoiceninja-debian:5
    container_name: invoiceninja
    restart: unless-stopped
    env_file: .env
    volumes:
      - public:/var/www/html/public
      - ./storage:/var/www/html/storage
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy

  nginx:
    image: nginx:alpine
    container_name: invoiceninja-web
    restart: unless-stopped
    volumes:
      - ./nginx:/etc/nginx/conf.d:ro
      - public:/var/www/html/public:ro
      - ./storage:/var/www/html/storage:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - default
      - web

  mysql:
    image: mysql:8
    container_name: invoiceninja-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - ./mysql:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u$$MYSQL_USER -p$$MYSQL_PASSWORD"]
      interval: 10s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: invoiceninja-redis
    restart: unless-stopped
    volumes:
      - ./redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  public:

networks:
  web:
    external: true
```

What's going on:

- **app** is Invoice Ninja itself. It also runs the background jobs (sending emails, recurring invoices) and includes a browser for making PDFs.
- **nginx** serves the app and is the only part that joins the `web` network, so Caddy can reach it. Nothing is published straight to the internet.
- **mysql** and **redis** stay on the app's private network.
- Your data (database, uploaded files, PDFs) lives in folders inside `~/apps/invoiceninja`, so the [backup guide](./08-backups.md) covers it as-is. The `public` volume holds only the app's own web files, which are rebuilt from the image on every start; we deleted it in our test and the app came back intact.

## 4. Add it to Caddy and start it

Add this to `~/apps/caddy/Caddyfile` and reload Caddy (see the [Caddy guide](./07-caddy.md)):

```
invoices.yourbusiness.com {
	reverse_proxy invoiceninja-web:80
}
```

Then start Invoice Ninja:

```bash
cd ~/apps/invoiceninja
docker compose up -d
docker compose ps
```

The first start takes a minute or two while it sets up the database. When `invoiceninja` shows `(healthy)`, open `https://invoices.yourbusiness.com` and log in with `IN_USER_EMAIL` and the password you saved.

Once you're in, remove the first-login lines from the settings file, because they're only needed once:

```bash
sed -i '/^IN_USER_EMAIL=/d; /^IN_PASSWORD=/d' .env
```

## 5. First setup in the app

The first time you log in, a **Welcome to Invoice Ninja** box asks for your company name, language and currency. Fill it in and click **Save**. Then, from **Settings** in the left menu:

1. **Company Details:** your business details, plus your address and logo on the **Address** and **Logo** tabs. These go on every invoice.
2. **Localization:** date format and time zone.
3. **Tax Settings:** add your tax rates if you charge sales tax or VAT.

Then try it out: create a client under **Clients**, then an invoice under **Invoices**, and open the invoice's PDF to see what your client will get.

**About "Pro" and "White Label":** the settings menu marks **Advanced Settings** (invoice designs, custom fields and more) as **Pro**, and a **Purchase White Label** button sits at the top of the screen. On a self-hosted install, Invoice Ninja sells that license to unlock those extras and remove its branding. Check the current price and what it includes before you buy. Everything in this guide, including clients, invoices, PDFs, payments and the client portal, worked without it.

Each invoice also has a **client portal link**, a private web page where your client can view the invoice, download the PDF and pay online if you connect a payment provider (**Settings → Payment Settings**, for example Stripe). We tested the portal link, but not online payments.

## 6. Sending invoices by email

The settings above use `MAIL_MAILER=log`, which **writes emails to a log file instead of sending them**. That's safe for trying things out. To email invoices to clients, use your email provider's SMTP details (most business email services and senders like Postmark, Mailgun or Amazon SES provide these). Replace the mail lines in `.env`:

```
MAIL_MAILER=smtp
MAIL_HOST=smtp.yourprovider.com
MAIL_PORT=587
MAIL_USERNAME=your-smtp-username
MAIL_PASSWORD=your-smtp-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=invoices@yourbusiness.com
MAIL_FROM_NAME="Your Business"
```

Then apply it with `docker compose up -d`. Send a test invoice to yourself before you send one to a client. We didn't test real email delivery, because our test server had no mail account.

## Backups and updates

Back up with the [nightly backup guide](./08-backups.md). It stops the app, copies `~/apps/invoiceninja` (including the database folder) and starts it again, which gives a clean copy of the database. **Test a restore** before you rely on it.

To update within version 5:

```bash
cd ~/apps/invoiceninja
docker compose pull
docker compose up -d
```

Invoice Ninja recommends taking a backup before each update.

---

*Tested November 1, 2026: this setup ran Invoice Ninja 5.13.52 with MySQL 8, Redis 7 and nginx behind Caddy. We followed this guide word for word on a clean setup, logged in with the first-start admin account (it still worked after removing the first-login lines and restarting), created a client and an invoice, generated its PDF (the line items and totals were correct), opened the client portal link, and marked the invoice paid, which recorded a payment. We then stopped and restarted everything and the data was still there. Email sending and online payments were not tested. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
