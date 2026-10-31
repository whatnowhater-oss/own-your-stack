---
title: "Back up your self-hosted apps every night (and test the restore)"
description: "One script, one cron line: encrypted nightly backups of every app on your server, kept for 6 months, with a restore you've actually tried."
last_checked: 2026-10-31
tested_on: "restic 0.16.4 (Ubuntu 24.04 package) backing up Uptime Kuma 2.x and Vaultwarden 1.37.4; both apps wiped and restored from the backup, Oct 31 2026"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to storage providers), they'll be marked here, and we recommend the same options whether or not they pay us.

A self-hosted app without backups is a bet that your server never dies, never gets hacked and never gets a bad update. Paid apps handle this for you; here it's your job. The good news is it takes 20 minutes to set up once, and then it runs every night on its own.

This guide uses [restic](https://restic.net), a free, widely used backup tool. Your backups are **encrypted** before they leave the server, **deduplicated** (each night only stores what changed, so 6 months of history costs little space), and easy to restore.

**You need:** a server set up like our [30-minute guide](./01-the-7-dollar-server.md), with each app in its own folder under `~/apps` (as in all our app guides).

Time: about 20 minutes. Cost: storage for the backups, typically a few dollars a month for a small business, or $0 if you use space you already have.

## How it works

Every night at 3 a.m. a script:

1. briefly **stops** each app (usually for under a minute), so its database files are copied in a consistent state
2. backs up all of `~/apps` with restic
3. **starts** every app again, even if the backup failed
4. deletes old snapshots, keeping the last 7 days, 4 weeks and 6 months

Caddy keeps running the whole time, so visitors briefly see an error page from Caddy rather than nothing at all.

## 1. Install restic

```bash
sudo apt update
sudo apt install -y restic
restic version
```

Ubuntu 24.04 ships restic 0.16. That's what we tested; `sudo restic self-update` upgrades to the newest release if you want it.

## 2. Choose where backups go

Backups on the same server don't protect you when the server dies. Send them somewhere else. restic supports many destinations; two easy ones:

| Destination | Repository address | Good for |
| --- | --- | --- |
| S3-compatible storage (Backblaze B2, Wasabi, Cloudflare R2, AWS S3) | `s3:https://<endpoint>/<bucket>` | Cheapest off-site option; pay per GB |
| Another server or storage box over SSH | `sftp:user@host:/backups/myserver` | You already have a second machine or a provider storage box |

Create the bucket or folder at your provider first. For S3 storage, also create an access key that can only use that one bucket.

We tested the steps below with a local repository; restic uses the same commands for every destination, and only the address and keys change.

## 3. Save the settings and an encryption password

```bash
mkdir -p ~/.config ~/bin
openssl rand -base64 32 > ~/.config/restic-password
chmod 600 ~/.config/restic-password
```

**Copy that password somewhere off the server now** (your password manager is ideal). Without it, your backups can't be opened by anyone, including you.

Create `~/.config/backup.env` (example for S3 storage; change the four values):

```bash
export RESTIC_REPOSITORY="s3:https://s3.your-region.example.com/your-bucket"
export RESTIC_PASSWORD_FILE="$HOME/.config/restic-password"
export AWS_ACCESS_KEY_ID="your-key-id"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
```

```bash
chmod 600 ~/.config/backup.env
```

For SSH storage, use the `sftp:` address and leave out the two `AWS_` lines.

Now create the repository (once):

```bash
source ~/.config/backup.env
restic init
```

## 4. The backup script

Save this as `~/bin/backup.sh`:

```bash
#!/usr/bin/env bash
# Nightly backup of every app in ~/apps with restic.
set -euo pipefail
source "$HOME/.config/backup.env"   # sets RESTIC_REPOSITORY, RESTIC_PASSWORD_FILE (+ provider keys)
APPS="$HOME/apps"

# 1. Stop apps that hold databases, so files are copied in a consistent state.
for app in "$APPS"/*/; do
  [ "$(basename "$app")" = caddy ] && continue
  (cd "$app" && docker compose stop)
done

# 2. Back up everything, then always restart the apps, even if the backup failed.
status=0
restic backup "$APPS" --tag nightly || status=$?
for app in "$APPS"/*/; do
  (cd "$app" && docker compose start)
done

# 3. Keep 7 daily, 4 weekly and 6 monthly snapshots; delete the rest.
restic forget --tag nightly --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
exit $status
```

```bash
chmod +x ~/bin/backup.sh
~/bin/backup.sh
```

The first run uploads everything and can take a while; later runs only send changes. Check it worked:

```bash
source ~/.config/backup.env
restic snapshots
```

Your app's `.env` files (with secrets like Cal.com's encryption key) are inside `~/apps`, so they're included. That's what you want for a restore, and it's why the backup is encrypted.

## 5. Run it every night

```bash
crontab -e
```

Add this line at the bottom:

```
0 3 * * * $HOME/bin/backup.sh >> $HOME/backup.log 2>&1
```

That runs at 3 a.m. server time (check with `date`). The `deploy` user can run Docker because you added it to the `docker` group in our server guide.

**Get told when it fails.** Add an Uptime Kuma **Push** monitor (see our [Uptime Kuma guide](./02-uptime-kuma.md)), set its heartbeat interval to 25 hours, and add this as the script's last line, before `exit $status`:

```bash
[ "$status" -eq 0 ] && curl -fsS "https://status.yourbusiness.com/api/push/YOUR_PUSH_TOKEN" > /dev/null
```

If a night's backup doesn't check in, you get an alert.

## 6. Test the restore (don't skip this)

A backup you've never restored is a hope, not a backup. Do this once now, and again every few months. For example, to bring back Vaultwarden:

```bash
cd ~/apps/vaultwarden
docker compose down
mv data data.old                    # keep the current copy, just in case

source ~/.config/backup.env
restic restore latest --target / --include "$HOME/apps/vaultwarden"

docker compose up -d
```

`--target /` puts files back at their original paths, and `--include` limits the restore to one app. Log in and check your data is there, then delete `data.old`.

In our test we deleted the data folders of both Uptime Kuma and Vaultwarden completely, restored them with exactly these commands, and both apps came back with their settings and accounts intact.

To check the backups themselves aren't damaged, run `restic check` every month or so.

## Restoring onto a new server

If the whole server is gone: set up a new one with our [30-minute guide](./01-the-7-dollar-server.md), install restic, put back `~/.config/backup.env` and the password file (this is why you saved the password elsewhere), then:

```bash
source ~/.config/backup.env
restic restore latest --target /
docker network create web   # if you use one Caddy for all apps
cd ~/apps/caddy && docker compose up -d   # then start each app the same way
```

Point your DNS records at the new server's address, and you're back. Restoring to `/` assumes the same username (`deploy`) as before.

---

*Tested October 31, 2026 with restic 0.16.4 from Ubuntu 24.04: the script stopped and restarted Uptime Kuma and Vaultwarden, the snapshot contained both apps' databases, and both apps were wiped and restored successfully. The off-site destinations (S3 and SFTP) were not tested by us; restic's commands are the same for them. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
