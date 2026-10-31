---
title: "Self-host Matomo analytics instead of Google Analytics"
description: "See who visits your website without sending your visitors' data to Google. A tested Docker setup with MariaDB, plus the privacy settings worth turning on."
last_checked: 2026-10-31
tested_on: "Matomo 5.14.2 and MariaDB 11 in Docker Compose behind Caddy (install wizard completed, page views tracked and processed into reports, Oct 31 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Google Analytics is free because your visitors' data is the product. [Matomo](https://matomo.org) is an open-source alternative you run yourself: page views, where visitors come from, which pages work, and goals like "booked a call". The data stays on your server.

**Why a small business might switch:**

- No data shared with Google, which makes privacy policies and cookie banners simpler (see "Privacy settings" below).
- Reports you can actually read: Matomo's dashboard is closer to the older, simpler Google Analytics many people preferred.
- No sampling or data limits on your own server.

**Why you might not:** it's one more app to update and back up, and Google Analytics connects directly to Google Ads. If you run ads heavily, you may want both.

We'd have liked to cover Plausible here too (a lighter, cookie-free option), but we couldn't test its current version in our setup. We only publish setups we've run, so Matomo it is for now.

**You need:** a server with Docker ([30-minute setup guide](./01-the-7-dollar-server.md)), the [one-Caddy setup](./07-caddy.md) with its `web` network, and a DNS A record such as `stats` pointing at the server. Matomo used about 230 MB of memory in our test, plus about 115 MB for its database.

Time: about 15 minutes. Extra cost: $0.

## 1. Folder and passwords

```bash
mkdir -p ~/apps/matomo && cd ~/apps/matomo
cat > .env <<EOF
DB_ROOT_PASSWORD=$(openssl rand -hex 20)
DB_PASSWORD=$(openssl rand -hex 20)
EOF
chmod 600 .env
```

## 2. The compose file

`compose.yaml`:

```yaml
services:
  db:
    image: mariadb:11
    container_name: matomo-db
    restart: unless-stopped
    command: --max-allowed-packet=64MB
    environment:
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MARIADB_DATABASE: matomo
      MARIADB_USER: matomo
      MARIADB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./db:/var/lib/mysql

  app:
    image: matomo:5-apache
    container_name: matomo
    restart: unless-stopped
    depends_on:
      - db
    environment:
      MATOMO_DATABASE_HOST: db
      MATOMO_DATABASE_ADAPTER: mysql
      MATOMO_DATABASE_TABLES_PREFIX: matomo_
      MATOMO_DATABASE_USERNAME: matomo
      MATOMO_DATABASE_PASSWORD: ${DB_PASSWORD}
      MATOMO_DATABASE_DBNAME: matomo
    volumes:
      - ./matomo:/var/www/html
    networks:
      - default
      - web

networks:
  web:
    external: true
```

The `MATOMO_DATABASE_*` values pre-fill the database step of the setup wizard, so you don't have to type them. The database stays on the app's private network.

Add Matomo to `~/apps/caddy/Caddyfile` and reload Caddy (see the [Caddy guide](./07-caddy.md)):

```
stats.yourbusiness.com {
	reverse_proxy matomo:80
}
```

## 3. Start it and run the setup wizard

```bash
docker compose up -d
```

Wait about 30 seconds, open `https://stats.yourbusiness.com`, and click through:

1. **System check:** should be all green. Click Next.
2. **Database setup:** already filled in from the compose file. Click Next; Matomo creates its tables.
3. **Superuser:** your admin login. Use a long password and a real email address (for password resets).
4. **Set up a website:** your site's name and address. **Pick a timezone**: the wizard won't continue until you do, and it doesn't say why very clearly. Choose your business's local city, or UTC.
5. **Tracking code:** copy it now (you can find it again later under **Administration → Websites → Tracking code**).
6. **Congratulations:** click **Continue to Matomo** to finish. Setup isn't complete until you do.

Do this right after starting: until the wizard is finished, anyone who finds the address could complete it.

## 4. Add the tracking code to your website

Paste the code from step 5 just before `</head>` on every page. Most website builders have a "custom code" or "header scripts" box for this. It looks like this, with your own address and site ID:

```html
<!-- Matomo -->
<script>
  var _paq = window._paq = window._paq || [];
  _paq.push(['trackPageView']);
  _paq.push(['enableLinkTracking']);
  (function() {
    var u="https://stats.yourbusiness.com/";
    _paq.push(['setTrackerUrl', u+'matomo.php']);
    _paq.push(['setSiteId', '1']);
    var d=document, g=d.createElement('script'), s=d.getElementsByTagName('script')[0];
    g.async=true; g.src=u+'matomo.js'; s.parentNode.insertBefore(g,s);
  })();
</script>
<!-- End Matomo Code -->
```

Visit your site in another browser, then open Matomo's **Visitors → Visits Log**. Your visit should be there within a minute.

## 5. Process reports on a schedule

By default Matomo builds its reports when you open the dashboard, which gets slow as traffic grows. Run it hourly in the background instead:

```bash
crontab -e
```

```
5 * * * * docker exec -u www-data matomo ./console core:archive --url=https://stats.yourbusiness.com > /dev/null 2>&1
```

Then in Matomo, under **Administration → System → General settings**, set **Archive reports when viewed from the browser** to **No**.

## Privacy settings worth turning on

Under **Administration → Privacy**:

- **Anonymize visitors' IP addresses** (2 bytes is a common choice).
- **Delete old raw data** after, for example, 6 or 12 months; your reports are kept.

To track without cookies, add this line to the tracking code, just before `_paq.push(['trackPageView']);`:

```js
_paq.push(['disableCookies']);
```

Whether you still need a cookie banner depends on your country and setup. Matomo's [privacy guides](https://matomo.org/privacy/) explain the options; check with a professional if you're unsure.

## Backups and updates

Everything lives in `~/apps/matomo`, so our [nightly backup guide](./08-backups.md) covers it as-is. To update within version 5:

```bash
cd ~/apps/matomo
docker compose pull
docker compose up -d
```

If Matomo shows a "database upgrade required" screen afterwards, click through it; it takes a few seconds on a small site.

---

*Tested October 31, 2026: this compose file ran Matomo 5.14.2 with MariaDB 11 behind Caddy. We completed the setup wizard, sent page views to the tracker (each recorded as a visit), and processed them into reports with the archive command above. The cookie and privacy settings were not tested. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
