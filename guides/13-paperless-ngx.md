---
title: "Self-host Paperless-ngx: search every receipt, invoice and contract you've scanned"
description: "Turn scans and PDFs into a searchable archive on your own server. A tested Docker setup with automatic text recognition (OCR), Postgres and HTTPS."
last_checked: 2026-11-01
tested_on: "Paperless-ngx 3.4.1 with Postgres 17 and Redis 7 in Docker Compose behind Caddy (a scanned, image-only invoice was read, dated and found by search; data survived a full restart, Nov 1 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Most small businesses keep receipts, supplier invoices and contracts as a mix of paper, email attachments and phone photos. Finding "that plumbing invoice from October" means digging. [Paperless-ngx](https://docs.paperless-ngx.com) fixes that. You give it a scan or PDF, it reads the text (OCR), works out the date, and makes every word searchable. You can tag documents by supplier, type or tax year.

**What it replaces:** paid document-management tools, and the habit of dumping scans into Google Drive or Dropbox where the text in scanned images can't always be searched. It's free and open source.

**Honest trade-offs:**

- **It's an archive, not an accounting app.** It finds documents; it doesn't do your books. Pair it with your accounting software, or with [Invoice Ninja](./11-invoice-ninja.md) for invoicing.
- **These are important papers.** If they only live on your server, your server's [backups](./08-backups.md) are your safety net. Set them up before you shred anything.
- **Disk space:** Paperless keeps your original file plus a searchable PDF copy, so plan for roughly twice the size of what you upload.

**You need:** a server with Docker ([30-minute setup guide](./01-the-7-dollar-server.md)), the [one-Caddy setup](./07-caddy.md) with its `web` network, and a DNS A record such as `docs` pointing at the server.

**Memory:** Paperless used about 600 MB in our test, plus about 50 MB for Postgres. Use a 2 GB server if it runs alone, or 4 GB alongside other apps ([Which VPS?](./09-which-vps.md)).

Time: about 15 minutes. Extra cost: $0.

## 1. Folder and settings

```bash
mkdir -p ~/apps/paperless/consume ~/apps/paperless/export && cd ~/apps/paperless
cat > .env <<EOF
PAPERLESS_URL=https://docs.yourbusiness.com
PAPERLESS_SECRET_KEY=$(openssl rand -hex 32)
PAPERLESS_ADMIN_USER=admin
PAPERLESS_ADMIN_PASSWORD=$(openssl rand -hex 16)
PAPERLESS_TIME_ZONE=America/New_York
PAPERLESS_OCR_LANGUAGE=eng
DB_PASSWORD=$(openssl rand -hex 20)
EOF
chmod 600 .env
grep ADMIN_PASSWORD .env
```

Replace `docs.yourbusiness.com` with your address and `PAPERLESS_TIME_ZONE` with [your time zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). `PAPERLESS_OCR_LANGUAGE` is the language of your documents: `eng` for English, `deu` for German, `fra` for French, or several joined with `+`, such as `eng+fra`.

The last line shows the password for the `admin` account. **Save it in your password manager now.**

## 2. The compose file

`compose.yaml`:

```yaml
services:
  broker:
    image: redis:7-alpine
    container_name: paperless-redis
    restart: unless-stopped
    volumes:
      - ./redis:/data

  db:
    image: postgres:17-alpine
    container_name: paperless-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./db:/var/lib/postgresql/data

  webserver:
    image: paperlessngx/paperless-ngx:latest
    container_name: paperless
    restart: unless-stopped
    depends_on:
      - db
      - broker
    env_file: .env
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_DBENGINE: postgresql
      PAPERLESS_DBPASS: ${DB_PASSWORD}
      PAPERLESS_BIND_ADDR: 0.0.0.0
      USERMAP_UID: 1000
      USERMAP_GID: 1000
    volumes:
      - ./data:/usr/src/paperless/data
      - ./media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    networks:
      - default
      - web

networks:
  web:
    external: true
```

Notes:

- **`PAPERLESS_BIND_ADDR: 0.0.0.0`** fixed a crash loop in our test ("Address family not supported by protocol") on a server without IPv6. It's harmless where IPv6 works.
- **`USERMAP_UID: 1000`** makes Paperless's files belong to the first normal user on the server (`deploy` in our [setup guide](./01-the-7-dollar-server.md)), so you can copy files in and out without `sudo`.
- **`consume`** is the inbox folder. Anything you put there is imported and then removed from the folder. **`media`** holds your documents, and **`data`** holds the search index.
- Everything lives in `~/apps/paperless`, so the [backup guide](./08-backups.md) covers it as-is.

## 3. Add it to Caddy and start it

Add this to `~/apps/caddy/Caddyfile` and reload Caddy (see the [Caddy guide](./07-caddy.md)):

```
docs.yourbusiness.com {
	reverse_proxy paperless:8000
}
```

Then start Paperless:

```bash
docker compose up -d
docker compose ps
```

The first start takes a minute or two while it sets up the database. Then open `https://docs.yourbusiness.com` and log in as `admin` with the password you saved. The `PAPERLESS_ADMIN_*` lines only create the account on the first start; you can change the password later under your account menu.

## 4. Add documents

There are two easy ways to get documents in:

- **Upload in the browser:** click **Upload** at the top, or drag files onto the dashboard. Phone photos and PDFs both work.
- **Drop them in the inbox folder:** copy files into `~/apps/paperless/consume`. From your own computer, for example:

```bash
scp receipt.pdf deploy@YOUR_SERVER_IP:apps/paperless/consume/
```

Within a minute, the document shows up under **Documents**. Paperless reads the text, guesses the date from the content, and makes it searchable from the **Search** box. In our test, we fed it a scanned supplier invoice that was only a picture, with no text layer. Paperless found it by searching for "copper", "plumbing" and the invoice number, and set its date to the one printed on the invoice.

Next, organise as you go, using the menu on the left:

- **Correspondents:** who sent it (your suppliers, the tax office).
- **Document types:** invoice, receipt, contract.
- **Tags:** anything else, like `tax-2026` or `to-pay`.

Paperless learns from what you assign and starts suggesting these for new documents.

**Scanner tip:** many office scanners can "scan to network folder". Pointing one at the `consume` folder needs file sharing (SMB) set up on the server, which this guide doesn't cover. Uploading in the browser or with `scp` is the simplest way to start.

## Backups and updates

The [nightly backup guide](./08-backups.md) covers `~/apps/paperless`. Paperless also has its own exporter, which writes every document and its tags to the `export` folder in a format that's easy to move to another server:

```bash
docker compose exec -T webserver document_exporter ../export
```

To update:

```bash
cd ~/apps/paperless
docker compose pull
docker compose up -d
```

Read the [changelog](https://docs.paperless-ngx.com/changelog/) before a major version jump, and take a backup first.

---

*Tested November 1, 2026: this setup ran Paperless-ngx 3.4.1 with Postgres 17 and Redis 7 behind Caddy. A scanned, image-only invoice dropped into the `consume` folder was imported in about 6 seconds, read by OCR, dated correctly, and found by searching for words in it. After stopping and restarting everything, the document and the search still worked. We then followed this guide word for word on a clean setup, including the exporter command. Email import and scan-to-folder over the network were not tested. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
