---
title: "Vaultwarden: a password manager for your team for the cost of a coffee"
description: "Run a Bitwarden-compatible password server for your whole team on the server you already have, and use the official Bitwarden apps."
last_checked: 2026-10-31
tested_on: "Vaultwarden 1.37.4 and Caddy 2.11 in Docker Compose (started, admin login checked, served through Caddy, Oct 31 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Team password managers usually cost $4 to $8 per person per month, so a 5-person business pays $240 to $480 a year. [Vaultwarden](https://github.com/dani-garcia/vaultwarden) is a free, open-source server that works with the **official Bitwarden apps** (browser extension, phone, desktop). Your team gets a polished app; you pay only for a server you may already run.

Be honest with yourself about one thing first: **this server will hold every password your business has.** Self-hosting it means you are responsible for updates and backups. If that worries you, a paid service is a sensible choice. If you're comfortable with the [30-minute server setup](./01-the-7-dollar-server.md) and you will set up backups (guide coming soon), read on.

Vaultwarden is an independent project, not made by Bitwarden Inc. It's widely used and actively maintained, but it hasn't had the same formal security audits as Bitwarden's own server.

**You need:** a server with Docker (our [setup guide](./01-the-7-dollar-server.md)), and a DNS A record such as `vault` pointing at it. HTTPS is required: the Bitwarden apps and web vault refuse to work over plain HTTP. Caddy handles that for you below.

Time: about 15 minutes. Extra cost: $0. It uses very little memory.

## 1. Make a folder and an admin password

```bash
mkdir -p ~/apps/vaultwarden && cd ~/apps/vaultwarden
```

The admin panel (`/admin`) lets you invite users and change settings. Protect it with a hashed password. Run this and type a long password twice:

```bash
docker run --rm -it vaultwarden/server:latest /vaultwarden hash --preset owasp
```

It prints a line starting with `ADMIN_TOKEN='$argon2id$...`. Put the hash in a file called `.env`, **keeping the single quotes** (the `$` signs would break without them):

```bash
nano .env
```

```
VW_ADMIN_TOKEN='$argon2id$v=19$m=19456,t=2,p=1$...the rest of your hash...'
```

Store the password you typed (not the hash) in a safe place. That's what you'll type at `/admin`.

## 2. The compose file

Create `compose.yaml`, changing `vault.yourbusiness.com` to your address:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.yourbusiness.com"
      SIGNUPS_ALLOWED: "true"
      ADMIN_TOKEN: ${VW_ADMIN_TOKEN}
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8080:80"

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

And `Caddyfile`:

```
vault.yourbusiness.com {
	reverse_proxy vaultwarden:80
}
```

> Already running Caddy from another guide (like [Uptime Kuma](./02-uptime-kuma.md))? Two Caddys can't both use ports 80 and 443. Our [Caddy guide](./07-caddy.md) shows how to run one Caddy for all your apps.

`SIGNUPS_ALLOWED` is `true` only so you can create your own account. You'll turn it off in step 4.

## 3. Start it and create your account

```bash
docker compose up -d
docker compose ps
```

Open `https://vault.yourbusiness.com`, click **Create account**, and use a long master password. **Nobody can recover a forgotten master password, including you as the admin.** Write it down and keep it somewhere physical and safe.

## 4. Close the door to strangers

Right after creating your account, stop new sign-ups. In `compose.yaml` change:

```yaml
      SIGNUPS_ALLOWED: "false"
```

```bash
docker compose up -d
```

Now only people you invite can join. Invite your team at `https://vault.yourbusiness.com/admin` → **Users → Invite User**. Invites are emailed if you add SMTP settings (in the admin panel under **SMTP Email Settings**). Without email, people can still join: an invited address can register at the normal sign-up page.

## 5. Set up the team

1. In the web vault, create an **Organization** (for example, "Your Business").
2. Make **Collections** like "Shared: Social media" and "Shared: Banking".
3. Each person installs the Bitwarden app or browser extension, opens its settings, picks **Self-hosted**, enters `https://vault.yourbusiness.com`, and logs in.
4. Turn on two-step login for everyone (**Account settings → Security → Two-step login**).

## Backups: not optional

Everything lives in `./data`. If the server dies without a backup, every shared password is gone. At minimum, copy that folder off the server every night. A backup guide that automates this and tests the restore is coming soon.

## Updating

```bash
cd ~/apps/vaultwarden
docker compose pull
docker compose up -d
```

Update at least monthly; this is the one app where falling behind on security fixes matters most.

## Did it pay off?

| | Paid team plan (5 people) | Vaultwarden |
| --- | --- | --- |
| Yearly cost | roughly $240 to $480 | $0 on a server you already have |
| Apps | official Bitwarden apps | the same official Bitwarden apps |
| Security updates and backups | handled for you | your job, about 10 minutes a month |
| Formal audits | yes | no (independent project) |

---

*Tested October 31, 2026: this compose file was started with Vaultwarden 1.37.4, the hashed admin password was accepted (and a wrong one refused), and the vault was served through Caddy. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
