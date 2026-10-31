---
title: "Self-host Uptime Kuma: free status monitoring instead of a paid uptime service"
description: "Get a text or email the minute your website goes down, plus a public status page, on the server you already have."
last_checked: 2026-10-30
tested_on: "Uptime Kuma 2.x and Caddy 2.11 in Docker Compose (setup run end to end, Oct 30 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

If your website, booking page or online shop goes down at 2 a.m., you want to know before your customers tell you. Paid uptime services charge roughly $7 to $30 a month once you want more than a few checks or a branded status page. [Uptime Kuma](https://github.com/louislam/uptime-kuma) does the same job for free on a server you already pay for.

You'll get:

- checks every 60 seconds (or as often as you like) on websites, APIs, ports and SSL certificate expiry
- alerts by email, Slack, Telegram, Discord, SMS services and dozens more
- a public status page like `status.yourbusiness.com`

**You need:** a server set up with [our 30-minute guide](./01-the-7-dollar-server.md) (Docker installed, ports 80 and 443 open), and a DNS A record such as `status` pointing at it.

Time: about 10 minutes. Extra cost: $0. Uptime Kuma uses very little memory, so it fits beside other apps on a 2 GB server.

> **Monitor from somewhere else.** If Uptime Kuma runs on the same server as your website and that server dies, it can't warn you. Run it on a separate small server, or at least monitor your main site from a second location.

## 1. Make a folder and two files

On your server:

```bash
mkdir -p ~/apps/uptime-kuma && cd ~/apps/uptime-kuma
```

Create `compose.yaml`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    ports:
      - "127.0.0.1:3001:3001"

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

Why it looks like this:

- `louislam/uptime-kuma:2` follows the 2.x line, so you get fixes without surprise major upgrades.
- `127.0.0.1:3001` keeps the app off the public internet. Only Caddy faces the world, which also avoids Docker slipping past your firewall.
- `./data` holds everything (monitors, settings, history). Back up this folder and you've backed up the app.

Create `Caddyfile`, replacing the domain with yours:

```
status.yourbusiness.com {
	reverse_proxy uptime-kuma:3001
}
```

That's the whole web server config. Caddy fetches and renews a free HTTPS certificate on its own.

> Already running Caddy for another app? Don't start a second one. Add the `status.yourbusiness.com { … }` block to your existing Caddyfile, put both apps on the same Docker network, and reload Caddy.

## 2. Start it

```bash
docker compose up -d
docker compose ps
```

Both containers should show `Up`, and Uptime Kuma should show `(healthy)` after about 30 seconds. If the first `up -d` stops during the download with an error like `failed to extract layer ... EOF`, the download broke, not your setup. Run the same command again.

## 3. First-run setup

Open `https://status.yourbusiness.com`.

1. **Database:** Uptime Kuma 2 first asks which database to use. Choose **SQLite**. It's a single file inside `./data`, needs no extra setup and is plenty for a small business. (Embedded MariaDB is also offered; you only need it for very large setups.)
2. **Admin account:** pick a username and a long password. Store it in your password manager.

Do step 3 right after starting. Until you create the admin account, anyone who finds the address can create it first.

## 4. Add your first monitors

Click **Add New Monitor** and create these:

| Monitor type | What to enter | Why |
| --- | --- | --- |
| HTTP(s) | `https://yourbusiness.com` | Is the website up? |
| HTTP(s) - Keyword | your booking page URL + a word that's always on it, like "Book" | Catches "up but broken" pages |
| HTTP(s) | any app you self-host | Your own stack |

On HTTP monitors, turn on **Certificate Expiry Notification** so you hear about an expiring certificate before visitors see a warning.

## 5. Get alerted

Go to **Settings → Notifications → Setup Notification**. Email (SMTP) works everywhere; Telegram and Discord are the quickest to set up for free phone push alerts. Click **Test**, then tick **Default enabled** so new monitors use it.

## 6. A public status page (optional)

**Status Pages → New Status Page**. Pick the monitors customers care about, add your logo, and share the link. A visible status page cuts "is it down?" emails during an outage.

## Updating

```bash
cd ~/apps/uptime-kuma
docker compose pull
docker compose up -d
```

Copy the `data` folder somewhere safe first. Our [backup guide](./08-backups.md) automates that.

## Did it pay off?

| | Paid uptime service | Uptime Kuma |
| --- | --- | --- |
| Monthly cost | roughly $7 to $30 | $0 on a server you already have |
| Check interval | often 1 to 5 min depending on plan | down to 20 seconds |
| Status pages | often paid extra | included |
| You maintain it | no | about 5 minutes a month |

If you'd rather not maintain anything, a paid service is a fair choice. If you already run a server, this is one of the easiest wins in self-hosting.

---

*Tested October 30, 2026: this exact compose file was started, the SQLite setup completed and the dashboard served through Caddy. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
