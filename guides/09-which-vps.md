---
title: "Which VPS for self-hosting? Hetzner vs DigitalOcean vs Vultr vs Hostinger"
description: "Current prices for the servers our guides need, the catches in the fine print, and which provider fits which business."
last_checked: 2026-10-31
tested_on: "Prices taken from each provider's public pricing page on Oct 31, 2026. We have not benchmarked these providers' speed or support."
---

> **Disclosure:** this page has no affiliate links today. If we add any later, they'll be marked here. Some providers pay referrers in cash, some only in account credit, and some not at all; we'll recommend the same options either way.

Every guide on this site runs on a plain Linux server (a "VPS") with Ubuntu 24.04 and Docker. All four providers below sell that. The differences are price, where their data centers are, and the fine print.

**What we haven't done:** we haven't run speed tests or contacted support at each provider, so this page compares only what they publish. When we do test them, we'll say so here.

## How much server do you need?

| What you run | Memory | Example |
| --- | --- | --- |
| 1 to 3 light apps | 2 GB | [Uptime Kuma](./02-uptime-kuma.md), [Vaultwarden](./05-vaultwarden.md), [Matomo](./04-matomo.md) |
| Cal.com or Nextcloud, plus a few light apps | 4 GB | [Cal.com](./03-calcom.md) used about 900 MB in our test, [Nextcloud](./06-nextcloud.md) about 400 MB with its database |
| Everything on this site at once | 8 GB | comfortable headroom |

Disk matters too: 50 GB is plenty for apps; file storage like Nextcloud needs more, or an add-on storage volume.

## Prices, October 31, 2026

| Provider | 2 GB server | 4 GB server | Billing | Source |
| --- | --- | --- | --- | --- |
| DigitalOcean | $12/mo (1 vCPU, 50 GB) | $24/mo (2 vCPU, 80 GB) | monthly, cancel anytime | [pricing](https://www.digitalocean.com/pricing/droplets) |
| Vultr | $10/mo (1 vCPU, 55 GB) | $20/mo (2 vCPU, 80 GB) | monthly, cancel anytime | [pricing](https://www.vultr.com/pricing/) |
| Hostinger | none smaller than 4 GB | $6.49/mo (1 vCPU, 50 GB) **with a 24-month prepay**, renews at $11.99/mo | prepaid term | [pricing](https://www.hostinger.com/vps-hosting) |
| Hetzner | plans from €5.99/mo (cost-optimized) or €6.49/mo (regular performance) | see their page for current sizes | monthly, cancel anytime | [pricing](https://www.hetzner.com/cloud) |

DigitalOcean and Vultr prices are in USD and exclude tax. Hetzner prices are in euros. We couldn't read Hetzner's full plan table on the day we checked, so check its current sizes on its page.

## The fine print that matters

- **Hostinger's low price is an intro price.** You pay for 24 months up front, then it renews at a higher rate ($11.99/mo for the 4 GB plan when we checked). Even at renewal, 4 GB for about $12 is good value; just budget for it.
- **Monthly billing is worth something.** DigitalOcean, Vultr and Hetzner bill monthly (they also show hourly prices), so you can try a server for a few days, delete it, and pay cents. That's the best way to learn.
- **Location affects speed and law.** Pick a region near your customers. Hetzner is a German company with mainly European data centers, which suits EU businesses. DigitalOcean and Vultr have many regions worldwide.
- **Backups are extra almost everywhere.** Provider snapshots cost a little more per month. They're a useful second layer, not a replacement for [your own off-server backups](./08-backups.md).

## Our picks

| If you... | Choose | Why |
| --- | --- | --- |
| are in Europe and want a low monthly price | Hetzner | low prices and EU data centers |
| want to try things with no commitment | Vultr or DigitalOcean | monthly billing, clear dashboards, many regions |
| know you'll run a 4 GB server for 2 years | Hostinger | cheapest 4 GB if you prepay; check the renewal price first |
| want the most tutorials and community answers | DigitalOcean | huge library of guides for exactly this kind of setup |

There's no wrong answer here: all four sell Ubuntu 24.04 servers, which is all our guides need. Pick on price and region, then follow the [30-minute server setup](./01-the-7-dollar-server.md).

---

*Prices checked October 31, 2026 from each provider's pricing page; they change often, so confirm before you buy. Found something out of date? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
