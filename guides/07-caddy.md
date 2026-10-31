---
title: "Automatic HTTPS for every app with one Caddy"
description: "Run all your self-hosted apps behind a single Caddy web server with free, auto-renewing HTTPS. Add a new app in three lines."
last_checked: 2026-10-31
tested_on: "Caddy 2.11, Uptime Kuma 2.x and Vaultwarden 1.37.4 on a shared Docker network (both apps proxied by one Caddy, config validated and reloaded live, Oct 31 2026)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Our single-app guides ([Uptime Kuma](./02-uptime-kuma.md), [Cal.com](./03-calcom.md), [Vaultwarden](./05-vaultwarden.md)) each start their own Caddy, which is the quickest way to get one app online. The moment you add a second app, that breaks: only one program can use ports 80 and 443.

The fix is one Caddy for the whole server. Each app joins a shared Docker network called `web`, and Caddy sends each address to the right app. Adding an app afterwards is three lines in one file.

**You need:** a server with Docker ([setup guide](./01-the-7-dollar-server.md)) and a DNS A record for each app (`status`, `vault`, `cal` and so on) pointing at the server.

Time: 15 minutes. Cost: $0.

## The layout

```
~/apps/
├── caddy/          compose.yaml + Caddyfile   (the only thing on ports 80/443)
├── uptime-kuma/    compose.yaml               (joins network "web")
└── vaultwarden/    compose.yaml + .env        (joins network "web")
```

Each app keeps its own folder, data and update schedule. Only Caddy is reachable from the internet.

## 1. Create the shared network (once)

```bash
docker network create web
```

## 2. Set up Caddy on its own

```bash
mkdir -p ~/apps/caddy && cd ~/apps/caddy
```

`compose.yaml`:

```yaml
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - web

volumes:
  caddy_data:
  caddy_config:

networks:
  web:
    external: true
```

The `443/udp` line enables HTTP/3, which makes pages a little faster on phones. `caddy_data` stores your certificates; don't delete it, or Caddy has to request them all again.

`Caddyfile`, one block per app:

```
status.yourbusiness.com {
	reverse_proxy uptime-kuma:3001
}

vault.yourbusiness.com {
	reverse_proxy vaultwarden:80
}
```

The name after `reverse_proxy` is the app's `container_name`, and the number is the port the app listens on **inside** its container.

## 3. Move each app onto the network

Each app's compose file loses its Caddy service and its `ports:` section, and gains the `web` network. Uptime Kuma, for example:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    networks:
      - web

networks:
  web:
    external: true
```

With no `ports:` at all, the app can't be reached from outside except through Caddy. That's safer than our single-app setups, and it fixes the known problem of Docker-published ports slipping past the ufw firewall.

For apps with a database, like [Cal.com](./03-calcom.md), put **only the web app** on `web`. Leave the database on the app's own default network so Caddy (and anything else) can't see it:

```yaml
  calcom:
    # ...everything else unchanged...
    networks:
      - default
      - web

networks:
  web:
    external: true
```

## 4. Switch over

If you followed a single-app guide, stop its old Caddy first, then start everything:

```bash
cd ~/apps/uptime-kuma && docker compose down && docker compose up -d
cd ~/apps/vaultwarden && docker compose down && docker compose up -d
cd ~/apps/caddy && docker compose up -d
```

`docker compose down` keeps your data folders; it only removes containers. Your old Caddy's certificates live in that app folder's volume, so the new Caddy will fetch fresh ones. That takes a few seconds per address.

## Adding a new app later

1. Give the app a `container_name`, remove its `ports:`, add the `web` network.
2. Add a block to `~/apps/caddy/Caddyfile`.
3. Check and reload Caddy with no downtime:

```bash
docker exec -w /etc/caddy caddy caddy validate --config Caddyfile
docker exec -w /etc/caddy caddy caddy reload --config Caddyfile
```

If `validate` complains, nothing has changed yet; fix the typo and try again. If an edit doesn't seem to take effect, run `docker compose restart caddy` in `~/apps/caddy` (some editors save a new file, which a running container doesn't see).

## Troubleshooting

| Symptom | Usual cause |
| --- | --- |
| `502 Bad Gateway` | App not on the `web` network, wrong container name, or wrong internal port |
| Certificate error in the browser | DNS record not pointing at this server yet, or port 80/443 blocked by your provider's firewall |
| `port is already allocated` when starting Caddy | An old single-app Caddy is still running: `docker ps` and stop it |

Check what Caddy is doing with `docker logs caddy --tail 50`.

---

*Tested October 31, 2026: one Caddy served Uptime Kuma and Vaultwarden over the shared `web` network, and a changed Caddyfile was validated and reloaded without a restart. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
