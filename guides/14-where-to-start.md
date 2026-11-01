---
title: "Which apps should a small business self-host first?"
description: "A practical order for moving from paid software to self-hosted apps: what to start with, what to leave for later, and how much server you need for each step."
last_checked: 2026-11-01
tested_on: "Built from our own tested guides. Memory figures are what each app used in our Docker tests, Oct 30 to Nov 1 2026."
---

> **Disclosure:** this page has no affiliate links. Our [server setup guide](./01-the-7-dollar-server.md) and [VPS comparison](./09-which-vps.md) contain one marked affiliate link, to Vultr.

You don't have to move everything at once, and you shouldn't. Start with apps that are easy to run and cheap to get wrong, learn how your server behaves, then move the ones your business depends on. This is the order we'd use, based on setting up every app on this site ourselves.

## Step 0: the foundation (do these first, always)

| Guide | Why first |
| --- | --- |
| [Set up the server](./01-the-7-dollar-server.md) | A safe base: key-only logins, a firewall, automatic security updates and Docker. |
| [One Caddy for all your apps](./07-caddy.md) | Free, automatic HTTPS for every app you add later. |
| [Nightly backups](./08-backups.md) | Without backups, every app below is a gamble. Test a restore once. |

## Step 1: low risk, quick wins

These are light, quick to set up, and nothing breaks for your customers if one goes down for an hour.

| App | Replaces | Memory in our test | Why start here |
| --- | --- | --- | --- |
| [Uptime Kuma](./02-uptime-kuma.md) | paid uptime monitors (roughly $7 to $30/month) | very little | It watches everything else you add, so set it up first. |
| [Matomo](./04-matomo.md) | Google Analytics | about 350 MB with its database | Your website keeps working if Matomo has a bad day. |
| [Paperless-ngx](./13-paperless-ngx.md) | scattered scans, paid document tools | about 650 MB with its database | Start with copies of documents, so nothing is lost while you learn it. |

## Step 2: tools your team uses every day

Move these once Step 1 has run smoothly for a few weeks and you've seen a backup restore work.

| App | Replaces | Memory in our test | What to watch |
| --- | --- | --- | --- |
| [Vaultwarden](./05-vaultwarden.md) | team password managers ($240 to $480/year for 5 people) | very little | It holds every password you have. Backups aren't optional here. |
| [n8n](./12-n8n.md) | Zapier (from $19.99/month for 750 tasks) | about 750 MB | More technical than Zapier. Move one automation at a time. |
| [Invoice Ninja](./11-invoice-ninja.md) | FreshBooks and similar ($23 to $70/month) | about 1 GB with its database | These are financial records: back them up, and send yourself a test invoice first. |

## Step 3: the big ones

These save the most, but they're the heaviest and the ones customers or staff notice first if something goes wrong.

| App | Replaces | Memory in our test | What to watch |
| --- | --- | --- | --- |
| [Cal.com](./03-calcom.md) | Calendly ($360 to $720/year for 3 people) | about 900 MB | Customers book through it, so a broken calendar costs bookings. |
| [Nextcloud](./06-nextcloud.md) | Google Workspace storage ($420 to $840/year for 5 people) | about 400 MB idle, more with use | Needs more disk and memory as files grow. Keep Google's email if you use it. |

## How much server do you need?

Add up the memory column for the apps you plan to run, then leave room to spare:

- **Step 0 plus Step 1:** a **2 GB** server is comfortable.
- **Adding Step 2:** go to **4 GB**.
- **Everything on this page at once:** **8 GB**, with headroom for growth.

Current prices for each size are in our [VPS comparison](./09-which-vps.md). To check whether the move is worth it for your bills, use the [savings calculator](./10-savings-calculator.md). It counts your own time, too.

## When not to self-host

Be honest with yourself. Keep paying for the hosted version if:

- **Nobody on the team is comfortable with a terminal**, and you'd rather not learn. Each app needs a few minutes of updates every month.
- **The tool is mission-critical and you have no backup plan.** If you can't afford an afternoon of downtime, practice on Step 1 first.
- **The savings are small.** Paying $10 a month for a tool that works well is often the right call. Self-host where the bill grows with your team, or where owning the data matters.

---

*Memory figures come from our own Docker tests of each guide, between October 30 and November 1, 2026, and grow with use. Prices are from the guides linked above and were checked on the dates shown in each one. Found something out of date? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
