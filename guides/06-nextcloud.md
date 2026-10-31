---
title: "Nextcloud vs Google Workspace for a 5-person business: real costs and a tested setup"
description: "An honest cost comparison, what Nextcloud does and doesn't replace, and a tested Docker setup with Postgres, Redis and background jobs."
last_checked: 2026-10-31
tested_on: "Nextcloud 35.0.2, Postgres 16, Redis 7 and Caddy 2.11 in Docker Compose (installed itself, file upload over WebDAV worked, served through Caddy, Oct 31 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

[Nextcloud](https://nextcloud.com) is the best-known self-hosted replacement for Google Drive: file sync on every device, sharing links, calendars and contacts, all on a server you control. Before you set it up, it's worth being clear about what it saves and what it doesn't.

## The short answer

**Nextcloud doesn't replace email.** If you use Google Workspace mainly for Gmail on your own domain, keep it (or move email to a dedicated email host) and use Nextcloud only if you need more file storage.

Where Nextcloud wins is **storage per dollar** and **owning your files**. Where Google wins is **zero maintenance** and its polished document editors.

## The costs, side by side

Google Workspace prices from [Google's pricing page](https://workspace.google.com/pricing), checked October 31, 2026 (US, annual commitment):

| | Google Business Starter | Google Business Standard | Nextcloud on your own server |
| --- | --- | --- | --- |
| Price | $7 per user per month | $14 per user per month | one server, roughly $10 to $20 a month |
| 5 people, per year | $420 | $840 | roughly $120 to $240 |
| Storage | 30 GB per person (pooled) | 2 TB per person (pooled) | your server's disk; extra storage volumes typically cost about $5 to $10 per 100 GB per month |
| Email on your domain | included | included | not included |
| Document editing | Google Docs, Sheets, Slides | Google Docs, Sheets, Slides | optional add-on (Nextcloud Office); not covered here |
| Maintenance | none | none | updates and backups, about 15 minutes a month |

The server cost assumes a 4 GB server (Nextcloud is comfortable there with a few other apps). Prices vary by provider and region.

**When it's worth it:** you need lots of storage (shared photo or video libraries, design files, scanned records), you want files to stay on infrastructure you control, or you're already paying for Business Standard mainly for storage.

**When it isn't:** your team is happy on Business Starter, you rely on Google Docs collaboration every day, or nobody wants to own updates and backups.

## The tested setup

**You need:** a server with Docker ([30-minute setup guide](./01-the-7-dollar-server.md)), the [one-Caddy setup](./07-caddy.md) with its shared `web` network, at least **4 GB** of memory, and a DNS A record such as `cloud` pointing at the server.

In our test, Nextcloud used about 320 MB of memory at idle plus about 70 MB for its database; it grows with users and file activity.

### 1. Folder and secrets

```bash
mkdir -p ~/apps/nextcloud && cd ~/apps/nextcloud
cat > .env <<EOF
NC_DOMAIN=cloud.yourbusiness.com
DB_PASSWORD=$(openssl rand -hex 20)
ADMIN_PASSWORD=$(openssl rand -hex 12)
EOF
chmod 600 .env
cat .env   # note the ADMIN_PASSWORD: that's your first login
```

### 2. The compose file

`compose.yaml`:

```yaml
services:
  db:
    image: postgres:16-alpine
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./db:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  app:
    image: nextcloud:35-apache
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - db
      - redis
    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      NEXTCLOUD_ADMIN_USER: admin
      NEXTCLOUD_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NC_DOMAIN}
      TRUSTED_PROXIES: caddy
      OVERWRITEPROTOCOL: https
      PHP_UPLOAD_LIMIT: 10G
    volumes:
      - ./html:/var/www/html
    networks:
      - default
      - web

  cron:
    image: nextcloud:35-apache
    container_name: nextcloud-cron
    restart: unless-stopped
    depends_on:
      - app
    entrypoint: /cron.sh
    volumes:
      - ./html:/var/www/html

networks:
  web:
    external: true
```

What each piece does:

- **db (Postgres):** Nextcloud's database. It sits only on the app's private network, so nothing outside can reach it.
- **redis:** caching and file locking. The official image detects `REDIS_HOST` and configures it for you; it makes the web interface noticeably faster.
- **app:** Nextcloud itself. The `NEXTCLOUD_ADMIN_*` values make it install itself on first start, so there's no setup wizard to click through. `TRUSTED_PROXIES` and `OVERWRITEPROTOCOL` tell it that Caddy is in front handling HTTPS.
- **cron:** runs Nextcloud's background jobs (cleanup, notifications, previews) every 5 minutes. Without it, Nextcloud's admin page complains and housekeeping falls behind.
- `nextcloud:35-apache` pins the major version, so updates never jump a major release by surprise.

### 3. Add it to Caddy

In `~/apps/caddy/Caddyfile`:

```
cloud.yourbusiness.com {
	reverse_proxy nextcloud:80
}
```

Then reload Caddy as shown in the [Caddy guide](./07-caddy.md).

### 4. Start it

```bash
docker compose up -d
docker logs -f nextcloud
```

Wait for `Nextcloud was successfully installed` (about a minute), then Ctrl+C. Switch background jobs to the cron container:

```bash
docker exec -u www-data nextcloud php occ background:cron
docker exec -u www-data nextcloud php occ status
```

`installed: true` means you're done. Open `https://cloud.yourbusiness.com` and log in as `admin` with the password from `.env`. Change it under **Personal settings → Security**, and turn on two-factor login.

### 5. Add your team

**Administration → Users → New user** for each person. Set a storage quota per user so one person's video folder can't fill the disk. Each person installs the [Nextcloud desktop and phone apps](https://nextcloud.com/install/) and signs in with your address.

Calendars and contacts sync to phones over CalDAV and CardDAV; the Nextcloud apps or your phone's account settings set that up.

### 6. Check the admin overview

**Administration → Overview** runs Nextcloud's own security and setup checks. Work through any warnings it lists; most are one-line fixes it explains.

## Backups and updates

Everything lives in `~/apps/nextcloud` (`html` holds the app and everyone's files; `db` holds the database), so our [nightly backup guide](./08-backups.md) covers it as-is: it stops the containers, backs up, and starts them again.

To update within version 35:

```bash
cd ~/apps/nextcloud
docker compose pull
docker compose up -d
```

To move to the next major version, change `35` to `36` in both places in `compose.yaml` **after** a backup. Nextcloud only supports upgrading one major version at a time.

---

*Tested October 31, 2026: this compose file installed Nextcloud 35.0.2 with Postgres 16 and Redis automatically, Redis caching and locking were active, background jobs switched to cron, a test file uploaded over WebDAV, and the login page loaded through Caddy. Nextcloud Office, phone sync and email were not tested. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
