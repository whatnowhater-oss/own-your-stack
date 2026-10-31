---
title: "Replace Calendly with self-hosted Cal.com"
description: "Run your own booking pages, with no per-seat fees, on a small server. Tested setup with Docker Compose, Postgres and automatic HTTPS."
last_checked: 2026-10-31
tested_on: "Cal.com 6.2.0, Postgres 16 and Caddy 2.11 in Docker Compose (started, database migrated, setup wizard served through Caddy, Oct 31 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Paid scheduling tools charge roughly $10 to $20 per person per month once you want more than one event type or remove their branding. For a 3-person team that's $360 to $720 a year. [Cal.com](https://github.com/calcom/cal.com) is open source and you can run it yourself: booking pages, multiple event types, buffers between meetings, and reminders, with your own domain and no per-seat bill.

**A note on names:** Cal.com now calls its self-hosted open-source edition **Cal.diy**, with docs at [cal.diy](https://www.cal.diy/docker). When we tested (October 31, 2026), the newest published Docker image was still `calcom/cal.com:v6.2.0` from March 2026, so that's what this guide uses and pins. We'll update this guide when a newer self-hosting image ships.

**Is it right for you?** Cal.com is the heaviest app we cover so far. It used about 900 MB of memory in our test, so give it a server with at least **2 GB** (4 GB if it shares the server with other apps). Some advanced team features are part of Cal.com's paid enterprise edition, and connecting Google Calendar takes an extra 15 minutes of setup in Google's console (step 5). If you just need one booking link and don't want to maintain anything, a free tier of a paid tool may be enough.

**You need:** a server with Docker (our [30-minute setup guide](./01-the-7-dollar-server.md)) and a DNS A record such as `cal` pointing at it.

Time: about 20 minutes. Extra cost: $0 on a server with enough memory.

## 1. Make a folder and secrets

```bash
mkdir -p ~/apps/calcom && cd ~/apps/calcom
```

Cal.com needs a few random secrets. This creates a `.env` file with all of them; change the address on the first line to yours:

```bash
cat > .env <<EOF
CAL_URL=https://cal.yourbusiness.com
POSTGRES_PASSWORD=$(openssl rand -hex 24)
NEXTAUTH_SECRET=$(openssl rand -base64 32)
CALENDSO_ENCRYPTION_KEY=$(openssl rand -base64 24)
EOF
chmod 600 .env
```

Keep a copy of `.env` with your backups. **Losing `CALENDSO_ENCRYPTION_KEY` means losing access to connected calendar accounts.**

## 2. The compose file

Create `compose.yaml`:

```yaml
services:
  database:
    image: postgres:16-alpine
    container_name: calcom-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: calcom
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: calcom
    volumes:
      - ./db:/var/lib/postgresql/data

  calcom:
    image: calcom/cal.com:v6.2.0
    container_name: calcom
    restart: unless-stopped
    depends_on:
      - database
    environment:
      NEXT_PUBLIC_WEBAPP_URL: ${CAL_URL}
      NEXTAUTH_URL: ${CAL_URL}/api/auth
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      CALENDSO_ENCRYPTION_KEY: ${CALENDSO_ENCRYPTION_KEY}
      DATABASE_HOST: database:5432
      DATABASE_URL: postgresql://calcom:${POSTGRES_PASSWORD}@database:5432/calcom
      DATABASE_DIRECT_URL: postgresql://calcom:${POSTGRES_PASSWORD}@database:5432/calcom
      NEXT_PUBLIC_LICENSE_CONSENT: agree
      CALCOM_TELEMETRY_DISABLED: "1"
    ports:
      - "127.0.0.1:3000:3000"

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

What the less obvious lines do:

- `DATABASE_HOST` lets Cal.com wait for Postgres before it starts.
- `NEXT_PUBLIC_LICENSE_CONSENT: agree` accepts Cal.com's open-source license (AGPLv3) for self-hosting.
- `CALCOM_TELEMETRY_DISABLED` stops usage reporting back to Cal.com.
- Port 3000 is bound to `127.0.0.1`, so only Caddy faces the internet.

Create `Caddyfile`:

```
cal.yourbusiness.com {
	reverse_proxy calcom:3000
}
```

> Already running Caddy from another guide? Two Caddys can't share ports 80 and 443. Our [Caddy guide](./07-caddy.md) shows how to run one Caddy for all your apps.

## 3. Start it

```bash
docker compose up -d
docker compose logs -f calcom
```

The first start takes a minute or two: it sets up the database (you'll see `All migrations have been successfully applied.`) and then prints `Ready`. Press Ctrl+C to stop watching the logs; the app keeps running.

## 4. Create the admin account

Open `https://cal.yourbusiness.com`. You'll land on a **Setup** wizard. Create the admin user with a strong password, then set your timezone and working hours.

Do this right after starting, before anyone else finds the address.

## 5. Connect your calendar

Cal.com checks your calendar so people can't book over existing meetings.

- **Apple iCloud, Fastmail, Nextcloud and other CalDAV calendars:** go to **Apps → CalDAV** and enter your calendar address and an app password. No extra setup.
- **Google Calendar:** you create your own Google OAuth credentials once. Follow the [Cal.diy Google setup steps](https://www.cal.diy/apps/google): enable the Google Calendar API, set up the consent screen, and create a "Web application" OAuth client with these redirect addresses:

  ```
  https://cal.yourbusiness.com/api/integrations/googlecalendar/callback
  https://cal.yourbusiness.com/api/auth/callback/google
  ```

  Download the client JSON, add it to `.env` as one line, `GOOGLE_API_CREDENTIALS='{"web":{...}}'`, add `GOOGLE_API_CREDENTIALS: ${GOOGLE_API_CREDENTIALS}` under `environment:` in `compose.yaml`, and run `docker compose up -d`. The container refreshes its app store on every start, so Google Calendar then appears under **Apps**. (We haven't tested this step ourselves yet.)

## 6. Turn on email

Without email settings, Cal.com can't send booking confirmations. Add your mail provider's SMTP details to `.env`:

```
EMAIL_FROM=bookings@yourbusiness.com
EMAIL_SERVER_HOST=smtp.yourprovider.com
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=your-smtp-username
EMAIL_SERVER_PASSWORD=your-smtp-password
```

Then add the same five names under `environment:` in `compose.yaml` (for example `EMAIL_FROM: ${EMAIL_FROM}`) and run `docker compose up -d`.

## 7. Make your first booking page

**Event Types → New**: name it ("30-minute intro call"), set the length, add a buffer, and copy the link. Put it in your email signature and on your website.

## Updating

```bash
cd ~/apps/calcom
docker compose pull
docker compose up -d
```

Back up first: the `db` folder plus your `.env`. Because the version is pinned, `pull` only fetches fixes you choose: to upgrade, change `v6.2.0` to a newer tag from [Docker Hub](https://hub.docker.com/r/calcom/cal.com/tags), then run the two commands above.

## Did it pay off?

| | Paid scheduling tool (3 people) | Self-hosted Cal.com |
| --- | --- | --- |
| Yearly cost | roughly $360 to $720 | $0 on a 2 to 4 GB server you already have |
| Your domain and branding | often paid extra | included |
| Google Calendar | one click | about 15 minutes of setup, once |
| Updates and backups | handled for you | your job, about 10 minutes a month |

---

*Tested October 31, 2026: this compose file started Cal.com 6.2.0 with Postgres 16, applied all database migrations, and served the setup wizard through Caddy. Calendar and email connections were not tested. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
